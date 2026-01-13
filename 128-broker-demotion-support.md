# Broker demotion support via KafkaRebalance resource

This proposal extends the `KafkaRebalance` custom resource to support broker demotion by integrating with Cruise Control's `/demote_broker` endpoint. 
This would allow users to demote brokers, removing them from partition leadership eligibility, in preparation for maintenance, decommissioning, or other operational needs.

## Current situation

Currently, Strimzi's Cruise Control integration provides several [rebalancing modes](https://strimzi.io/docs/operators/latest/deploying#con-optimization-proposals-modes-str) through the `KafkaRebalance` custom resource:
* `full`: performs a complete cluster rebalance across all brokers
* `add-brokers`: moves replicas to newly added brokers after scaling up
* `remove-brokers`: moves replicas off brokers before scaling down
* `remove-disks`: performs intra-broker disk rebalancing for JBOD configurations

However, there is no built-in way to demote brokers, which would remove them from partition leadership without moving replicas. 
When preparing brokers for removal or maintenance, users must rely on the `remove-brokers` mode which moves all partition replicas off the target brokers. 
This is more disruptive than necessary if the goal is simply to ensure the brokers are not serving as partition leaders.

Cruise Control provides a dedicated [`/demote_broker`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#demote-a-list-of-brokers-from-the-kafka-cluster) endpoint specifically for this use case, but Strimzi does not currently expose it.

## Motivation

There are a few scenarios where demoting brokers without moving replicas is beneficial:

1. **Broker maintenance**: Before performing maintenance on a broker such as upgrading, patching, or restarting, operators may want to transfer leadership away to minimize impact on client traffic, while keeping the broker as a follower to maintain replication factor and availability.

2. **Staged decommissioning**: In a multi-step decommissioning process, operators may first demote brokers to observe the impact on leadership distribution and client performance before committing to fully removing replicas with the `remove-brokers` mode.

3. **Performance isolation**: Operators may want to reduce load on specific brokers experiencing performance issues by removing their leadership responsibilities while keeping them as followers until the issue is diagnosed and resolved.

The `remove-brokers` mode is too aggressive for these scenarios because it moves all replicas off the target brokers resulting in:
- Significant network bandwidth consumption from replica movement
- Increased CPU and disk I/O on source and destination brokers
- Extended time to complete the operation
- Temporary reduction in replication factor during replica migration
- Unnecessary disruption when the goal is only to remove leadership

Broker demotion addresses these concerns by only transferring leadership, which is a lightweight operation compared to replica movement.

## Proposal

The proposal is to add a new `demote-brokers` mode to the existing `KafkaRebalance` custom resource, following the same pattern established by the `add-brokers` and `remove-brokers` modes introduced in [proposal 035](https://github.com/strimzi/proposals/blob/main/035-rebalance-types-scaling-brokers.md).

When `spec.mode` is set to `demote-brokers` in the `KafkaRebalance` resource, paritition leadership is moved off the specified brokers while replicas remain in place.

Users must provide a list of broker IDs to demote for broker-level demotion via the `spec.brokers` field.

A `KafkaRebalance` custom resource for broker demotion would look like this:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: demote-brokers-example
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: demote-brokers
  brokers:
    - 3
    - 4
```

This example would demote brokers 3 and 4, transferring all partition leadership away from them while keeping the replicas in place.

### Supported parameters for `demote-broker` mode

The supported parameters, new and old, for `demote-brokers` mode in the `KafkaRebalance` resource spec are as follows:

| Field                            | Type    | Description                                                                
|----------------------------------|---------|-----------------------------------------------------------------------------|
| brokers                          | integer array    | List of ids of broker to be demoted in the cluster.                |
| concurrentLeaderMovements        | integer | Upper bound of ongoing leadership movements. Default is 1000.               |
| skipUrpDemotion                  | boolean | Whether to skip demoting leader replicas for under-replicated partitions.   |
| excludeFollowerDemotion          | boolean | Whether to skip demoting follower replicas on the broker to be demoted.     |
| excludeRecentlyDemotedBrokers    | boolean | Whether to allow leader replicas to be moved to recently demoted brokers.   |
| replicaMovementStrategies        | string array | A list of strategy class names used to determine the execution order for replica position swaps within partitions (metadata changes only, no data movement) in the generated optimization proposal. By default BaseReplicaMovementStrategy is used, which will execute the replica position swaps in the order in which they are generated. |
| replicationThrottle             | integer | The upper bound, in bytes per second, on the bandwidth used to move replicas. There is no limit by default. |

**NOTE**: As part of this proposal, we will also add a `excludeRecentlyDemotedBrokers` parameter for the `full`, `add-brokers`, and `remove-brokers` KafkaRebalance modes to give users the ability to prevent to leader replicas to be moved to recently demoted brokers.
When `excludeRecentlyDemotedBrokers` is set to `true`, a broker is considered demoted for the duration specified by the Cruise Control `demotion.history.retention.time.ms` server configuration.
By default, this value is 1209600000 milleseconds (14 days) but is configurable in the `spec.cruiseControl.config` section of the `Kafka` custom resource.

### User workflow

The workflow for using broker demotion follows the same pattern as other rebalance modes:

1. User creates a `KafkaRebalance` custom resource with `spec.mode: demote-brokers` and specifies the target broker IDs in `spec.brokers`.

2. The `KafkaRebalanceAssemblyOperator` requests an optimization proposal from Cruise Control via the `/demote_broker` endpoint with `dryrun=true`.

3. The operator transitions the `KafkaRebalance` resource to the `ProposalReady` state. 
The proposal is stored in `status.optimizationResult` and shows which partition leadership transfers will occur.

4. User reviews the proposal and approves it by annotating the resource with `strimzi.io/rebalance=approve`.

5. The operator executes the broker demotion via the `/demote_broker` endpoint with `dryrun=false`.

6. When complete, the operator transitions the `KafkaRebalance` resource to `Ready` state.

### Validation and constraints

The implementation includes the following validation:

* When `demote-brokers` mode is specified, the `brokers` parameter must be provided.
If the field is missing or empty, the operator will reject the rebalance request and report an error in the `KafkaRebalance` status.

* The specified broker IDs in the `brokers` list must exist in the cluster. 
If any of the broker IDs are invalid, the demotion request will be rejected and the error will be reported in the `KafkaRebalance` status.

* When an impossible demotion operation is requested for example demoting all brokers or transfering leadship from the only in-sync replica when the KafkaRebalance `spec.skipUrpDemotion` configuration is set to `false`, the demotion request will be rejected and the error will be reported in the `KafkaRebalance` status.

* The following `KafkaRebalance` resource configuration fields are incompatible with broker demotion and if specified, will cause the rebalance to be rejected, an error to logged by the Strimzi Operator, and an errorto be added to the `KafkaRebalance` status:
  * `goals`
  * `skipHardGoalCheck`
  * `rebalanceDisk`
  * `excludedTopics`
  * `concurrentPartitionMovementsPerBroker`
  * `concurrentIntraBrokerPartitionMovements`
  * `moveReplicasOffVolumes`

### Interaction with other rebalance modes

Broker demotion is independent of other rebalance modes:

* **After demotion**: If decommissioning is the end goal, users can subsequently create a `remove-brokers` rebalance to move replicas off of targeted brokers.

* **Before scaling down**: Demotion can be performed before a `remove-brokers` operation as a preparatory step to minimize disruption.

* **With auto-rebalancing upon scaling**: To reduce the complexity of this proposal, broker demotion will remain as a manual operation independent of cluster scaling as described in [proposal 078](https://github.com/strimzi/proposals/blob/main/078-auto-rebalancing-cluster-scaling.md).
 
* **With standard rebalancing**: After demoting brokers, users can run a `full` mode rebalance to redistribute leadership across the remaining leader-eligible brokers.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator.

**Affected components:**
* `KafkaRebalanceAssemblyOperator`
* CRD schema for `KafkaRebalance`

**Not affected:**
* Topic Operator
* User Operator
* Entity Operator
* Kafka Bridge
* Kafka Connect
* Kafka MirrorMaker 2

## Compatibility

The proposed changes are fully backward compatible:

* **API compatibility**: Adding a new enum value to `KafkaRebalanceMode` does not break existing resources. 
Existing `KafkaRebalance` resources using `full`, `add-brokers`, `remove-brokers`, and `remove-disks` modes continue to work unchanged.

* **CRD compatibility**: The `KafkaRebalance` CRD already includes the `mode` and `brokers` fields required for this feature. 
No structural changes to the CRD schema are needed beyond allowing the new enum value and adding new parameters.

* **Behavioral compatibility**: Existing rebalancing workflows are unaffected. 
The new mode is opt-in and requires explicit user action.

## Future Improvements

### Add support for disk-level demotion

Since Cruise Control's [`/demote_broker`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#demote-a-list-of-brokers-from-the-kafka-cluster) endpoint includes a parameter for demoting individual disks, `brokerid_and_logdirs`, a logical follow up feature would be to add support for disk-level demotion.

For disk-level demotion, partition leadership is moved off specific disk volumes on selected brokers, while replicas remain on those volumes.

A `KafkaRebalance` custom resource for disk demotion would look like this:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: demote-disks-example
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: demote-disks
  moveLeadersOffVolumes:
    - brokerId: 0
      volumeIds: [1, 2]
    - brokerId: 1
      volumeIds: [0, 1]
```

This example would demote the disk volumes 1 and 2 of brokers 0 and volumes 0 and 1 of broker 1, transferring all partition leadership away from those volumes while keeping the replicas in place.

#### Supported parameters for `demote-disks` mode

In addition to the supported parameters of the `demote-brokers` mode, the `demote-disks` mode in the `KafkaRebalance` resource spec would introduce the following:

| Field                            | Type    | Description
|----------------------------------|---------|-----------------------------------------------------------------------------|
| moveLeadersOffVolumes            | [BrokerAndVolumeIds](https://strimzi.io/docs/operators/latest/configuring.html#type-BrokerAndVolumeIds-reference) array | List of broker ID and volume pairs to be demoted in the cluster. |

#### Validation and constraints

The implementation would include the following validation:

* The specified broker and volume IDs in the `moveLeadersOffVolumes` list must exist in the cluster. 
Cruise Control will validate this and return an error for the `KafkaRebalance` status if invalid broker IDs or volume IDs are provided.

### Add support for broker-level demotion before automatic cluster scaling

A future enhancement could integrate broker-level demotion with automatic cluster scaling as described in [proposal 078](https://github.com/strimzi/proposals/blob/main/078-auto-rebalancing-cluster-scaling.md).
In such an implementation, brokers scheduled for removal could first be automatically demoted, ensuring that partition leadership is transferred away from those brokers before scaling operations proceed. 
This would reduce operational overhead and improve safety by minimizing leadership movement during broker removal.

This enhancement would build on the existing auto-rebalancing mechanisms and could be implemented as an optional, configurable step in the cluster scaling process.

## Rejected alternatives

### Alternative 1: Make demotion part of `remove-brokers` mode

Instead of adding a separate mode, enhance the `remove-brokers` mode to support a two-phase operation: first demote to transfer leadership only and then optionally move replicas.

This could be controlled via a new field like `spec.demoteOnly: true`.

**Reasons for rejection:**
* Overloads the semantics of `remove-brokers`, which are intended for replica removal
* Makes the `remove-brokers` mode more complex with conditional behavior
* Reduces clarity for users about what operation is being performed
* Inconsistent with the design philosophy of having distinct modes for distinct operations
* A separate mode provides better visibility in status, metrics, and logs about which operation is in progress.
