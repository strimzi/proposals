# Broker demotion support via KafkaRebalance resource

This proposal extends the `KafkaRebalance` custom resource to support broker and disk-level demotion by integrating with Cruise Control's `/demote_broker` endpoint. 
This would allow users to demote brokers or disks, removing them from partition leadership eligibility, in preparation for maintenance, decommissioning, or other operational needs.

## Current situation

The `KafkaRebalance` resource currently supports four distinct modes:
* `full`: Rebalances across all brokers in the cluster.
* `add-brokers`:  Moves replicas to newly added brokers.
* `remove-brokers`: Moves replicas out of brokers to be removed.
* `remove-disks`: Moves replicas off specified JBOD volumes of specified brokers so those volumes can be removed.

However, there is no built-in way to demote brokers or disks, meaning to remove them from partition leadership without moving replicas.
When preparing brokers for removal or maintenance, users must rely on the `remove-brokers` mode which moves all partition replicas off the target brokers.
Similarly, when preparing disks for removal, users must rely on the `remove-disks` mode which moves all partition replicas off the target disks.
These operations are more disruptive than necessary if the goal is simply to ensure the brokers or disks are not serving as partition leaders.

Cruise Control provides a dedicated [`/demote_broker`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#demote-a-list-of-brokers-from-the-kafka-cluster) endpoint specifically for this use case, but Strimzi does not currently expose it via the `KafkaRebalance` resource.

## Motivation

There are a few scenarios where demoting brokers or disks without moving replicas is beneficial:

1. **Broker or disk maintenance**: Before performing maintenance on a broker such as upgrading, patching, or restarting, operators may want to transfer leadership away to minimize impact on client traffic, while maintaining replication factor and availability.

2. **Staged decommissioning**: In a multi-step decommissioning process, operators may first demote brokers to observe the impact on leadership distribution and client performance before committing to fully removing replicas with the `remove-brokers` mode.

3. **Performance isolation**: Operators may want to reduce load on specific brokers or disks experiencing performance issues by removing their leadership responsibilities while keeping them as followers until the issue is diagnosed and resolved.

The `remove-brokers` and `remove-disks` modes are too aggressive for these scenarios because they move all replicas off the target brokers or disks resulting in:
- Significant network bandwidth consumption from replica movement (for broker-level operations)
- Increased CPU and disk I/O on source and destination brokers or disks
- Extended time to complete the operation
- Unnecessary disruption when the goal is only to remove leadership

Broker demotion addresses these concerns by only transferring leadership, which is a lightweight operation compared to transferring replicas.

## Proposal

### API Design

The proposal is to add a new `demote-brokers` mode to the existing `KafkaRebalance` custom resource, following the same pattern established by the `add-brokers` and `remove-brokers` modes introduced in  [proposal 035](https://github.com/strimzi/proposals/blob/main/035-rebalance-types-scaling-brokers.md).
When `spec.mode` is set to `demote-brokers` in the `KafkaRebalance` resource, partition leadership is moved off the specified brokers or disks while replicas remain in place.

#### Broker-level demotion

For broker-level demotion, users can specify a list of broker IDs to demote via the `spec.nodes` field, following the unified targeting mechanism introduced in  [proposal 154](154-kafkarebalance-custom-resource-consolidation.md).

A `KafkaRebalance` custom resource for broker-level demotion would look like this:
```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: demote-brokers-example
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: demote-brokers
  nodes:
    - brokerId: 3
    - brokerId: 4
  config:
    concurrent_leader_movements: 10
```

This example would demote brokers 3 and 4, transferring all partition leadership away from them while keeping the replicas in place.

#### Disk-level demotion

For disk-level demotion, users can specify a list of volume (disk) IDs to demote via the `spec.nodes` entries following the unified targeting mechanism introduced in  [proposal 154](154-kafkarebalance-custom-resource-consolidation.md).

A `KafkaRebalance` custom resource for disk-level demotion would look like this:
```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: demote-broker-disks-example
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: demote-brokers
  nodes:
    - brokerId: 3
      volumeIds:
        - 0
        - 1
    - brokerId: 4
      volumeIds:
        - 2
  config:
    concurrent_leader_movements: 10
```

This example would demote leadership for partitions on volumes 0 and 1 of broker 3 and volume 2 of broker 4, while leaving leadership for partitions on other volumes unchanged.

#### Supported fields for `demote-brokers` mode

The supported fields for `demote-brokers` mode in the `KafkaRebalance` resource spec are as follows.

##### Top-level fields

| Field   | Type                     | Description                                                                                                     | Required     |
|---------|--------------------------|-----------------------------------------------------------------------------------------------------------------|--------------|
| `mode`  | string                   | Must be set to `demote-brokers`.                                                                                | **required** |
| `nodes` | list of `RebalanceNode`  | List of brokers to be demoted, each containing a `brokerId` and optionally `volumeIds` for disk-level demotion. | **required** |

**NOTE**: Deprecated top-level fields that map to supported parameters (e.g. `concurrentLeaderMovements` -> `concurrent_leader_movements`, `replicationThrottle` -> `replication_throttle`, `replicaMovementStrategies` -> `replica_movement_strategies`) are converted to their `spec.config` equivalents at the start of reconciliation, following the conversion process defined in [proposal 154](154-kafkarebalance-custom-resource-consolidation.md).

##### Supported parameters

Following the `spec.config` approach from [proposal 154](154-kafkarebalance-custom-resource-consolidation.md), the parameters supported by Cruise Control's [`/demote_broker`](https://github.com/cruise-control-for-kafka/cruise-control/wiki/REST-APIs#demote-a-list-of-brokers-from-the-kafka-cluster) endpoint are passed as key-value pairs in `spec.config`:

| Config Key                          | Description                                                                         | Default   |
|-------------------------------------|-------------------------------------------------------------------------------------|-----------|
| `concurrent_leader_movements`       | Upper bound of ongoing leadership movements.                                        | N/A       |
| `skip_urp_demotion`                 | Whether to skip demoting leader replicas for under-replicated partitions.           | `"true"`  |
| `exclude_follower_demotion`         | Whether to skip demoting follower replicas on the broker to be demoted.             | `"true"`  |
| `exclude_recently_demoted_brokers`  | Whether to allow leader replicas to be moved to recently demoted brokers.           | `"false"` |
| `replica_movement_strategies`       | Replica movement strategy to use.                                                   | N/A       |
| `replication_throttle`              | Upper bound on bandwidth used to move replicas (bytes/sec).                         | N/A       |

**NOTE**: The `exclude_recently_demoted_brokers` config key is also supported for the `full`, `add-brokers`, and `remove-brokers` KafkaRebalance modes to give users the ability to prevent leader replicas from being moved to recently demoted brokers.
When `exclude_recently_demoted_brokers` is set to `"true"`, a broker is considered demoted for the duration specified by the Cruise Control `demotion.history.retention.time.ms` server configuration.
By default, this value is 1209600000 milliseconds (14 days) but is configurable in the `spec.cruiseControl.config` section of the `Kafka` custom resource.

##### Filtered parameters

The following Cruise Control `/demote_broker` parameters are managed by the operator or by `spec.nodes` and are forbidden in `spec.config`, following the same pattern as the filtered parameters in  [proposal 154](154-kafkarebalance-custom-resource-consolidation.md).
These are enforced using `FORBIDDEN_PREFIXES` and `FORBIDDEN_PREFIX_EXCEPTIONS` constants in `KafkaRebalanceSpec`, following the same pattern used for [`kafka.config`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/api/src/main/java/io/strimzi/api/kafka/model/kafka/KafkaClusterSpec.java#L56-L72) and [`cruiseControl.config`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/api/src/main/java/io/strimzi/api/kafka/model/kafka/cruisecontrol/CruiseControlSpec.java#L46-L50) sections of the `Kafka` resource configuration.
If any of these parameters are specified in `spec.config`, they are silently removed and a warning is logged (see [Validation Example 2](#validation-examples)).

| Parameter                   | Why it is filtered                                                                                                 |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------|
| `brokerid`                  | Managed via `spec.nodes` top-level field (`brokerId`).                                                             |
| `brokerid_and_logdirs`      | Managed via `spec.nodes` top-level field (`brokerId` + `volumeIds`).                                               |
| `dryrun`                    | Strimzi controls this via the rebalance state machine. Proposal generation vs. execution are separate states.      |
| `json`                      | Hardcoded to `true` by Strimzi. Changing this would break response parsing.                                        |
| `verbose`                   | Changing verbosity could break status reporting.                                                                   |
| `allow_capacity_estimation` | Managed by the operator.                                                                                           |
| `reason`                    | Managed by the operator.                                                                                           |
| `doAs`                      | Not applicable in Strimzi's deployment model.                                                                      |

### User workflow

The workflow for using broker demotion follows the same pattern as other rebalance modes:

1. User creates a `KafkaRebalance` custom resource with `spec.mode: demote-brokers` and specifies the target brokers and optionally volumes in `spec.nodes`.

2. The `KafkaRebalanceAssemblyOperator` requests an optimization proposal from Cruise Control via the `/demote_broker` endpoint with `dryrun=true`.

3. The operator transitions the `KafkaRebalance` resource to the `ProposalReady` state. 
The proposal is stored in `status.optimizationResult` and shows which partition leadership transfers will occur.

4. If [auto-approval](https://strimzi.io/docs/operators/latest/deploying#automatically_approving_an_optimization_proposal) is not enabled, the user reviews the proposal and approves it by annotating the resource with `strimzi.io/rebalance=approve`.

5. The operator executes the broker demotion via the `/demote_broker` endpoint with `dryrun=false`.

6. When complete, the operator transitions the `KafkaRebalance` resource to `Ready` state.

### Implementation Strategy

1. **Validation**

    - **Mode-specific operand validation**:
      - `nodes` is required and non-empty for `demote-brokers` mode.
      - For `demote-brokers` mode, `volumeIds` on `nodes` entries is optional.
        When provided, only partitions on the specified volumes are demoted.
        When omitted, all partitions on the broker are demoted.
      - Broker-level and disk-level demotion cannot be mixed in a single `KafkaRebalance` resource.
        All `nodes` entries must either specify `volumeIds` (disk-level) or none must (broker-level).
      - The specified broker IDs in the `nodes` list must exist in the cluster.
      - Impossible demotion operations are rejected, for example demoting all brokers or transferring leadership from the only in-sync replica when `skip_urp_demotion` is set to `"false"` in `spec.config`.
      - If a target broker fails during leadership transfer, demotion operations involving that broker are aborted and the remaining operations continue on a best-effort basis.
      - The deprecated `brokers` and `moveReplicasOffVolumes` fields are not supported for `demote-brokers` mode.
        Since `demote-brokers` is a new mode introduced after [proposal 154](154-kafkarebalance-custom-resource-consolidation.md), there are no existing resources to maintain backward compatibility with.
        Users must use the `nodes` field instead.

    - **Parameter field validation**:
      - Deprecated top-level fields (e.g. `concurrentLeaderMovements`, `replicationThrottle`, `replicaMovementStrategies`) are converted to their `spec.config` equivalents following the conversion process defined in [proposal 154](154-kafkarebalance-custom-resource-consolidation.md).
      - Deprecated top-level fields that are incompatible with, or no-ops for, broker demotion will be rejected by Cruise Control.
      The following fields are not supported in `demote-brokers` mode as they are not parameters of the Cruise Control [`/demote_broker`](https://github.com/cruise-control-for-kafka/cruise-control/wiki/REST-APIs#demote-a-list-of-brokers-from-the-kafka-cluster) endpoint:
        - `skipHardGoalCheck`
        - `rebalanceDisk`
        - `excludedTopics`
        - `concurrentPartitionMovementsPerBroker`
        - `concurrentIntraBrokerPartitionMovements`
      - Forbidden parameters in `spec.config` are filtered as described in [Filtered parameters](#filtered-parameters).

    See [Validation Examples](#validation-examples) for how errors are surfaced to users.

2. **Update examples** to encourage use of new API structure
   - Ensure the packaged `KafkaRebalance` resource examples are updated to include `demote-brokers` mode examples using `spec.nodes` and `spec.config`.

3. **Update documentation** to document the new `demote-brokers` mode and point to the upstream [Cruise Control REST API Wiki](https://github.com/cruise-control-for-kafka/cruise-control/wiki/REST-APIs) where needed.
   - Add a table to the documentation mapping supported and unsupported `spec.config` keys for `demote-brokers` mode.
   - Add examples showing broker-level and disk-level demotion.
   - Using the `FORBIDDEN_PREFIXES` and `FORBIDDEN_PREFIX_EXCEPTIONS` constants maintained in the `KafkaRebalanceSpec`, generate API documentation listing which upstream Cruise Control fields are unsupported by Strimzi in the same way it is done for [`cruiseControl.config`](https://strimzi.io/docs/operators/latest/configuring#type-CruiseControlSpec-schema-reference) in the `Kafka` resource.
   
#### Validation Examples

  1. **Mixing old and new parameter fields (Strimzi converts with warnings)**:
    - **KafkaRebalance status**: The resource proceeds normally.
      A warning condition is added if both a deprecated field and its corresponding new field are set, indicating that the deprecated value is being ignored.
    - **Cluster Operator log**: WARN for each deprecated field used (deprecation notice), and an additional WARN if both old and new field are set (conflict notice, e.g. "Both `concurrentPartitionMovementsPerBroker` and `spec.config[concurrent_partition_movements_per_broker]` are set.
     Using the value from `spec.config` and ignoring the deprecated field.")
    - **Cruise Control log**: N/A

  2. **Forbidden config key (Strimzi filters)**:
    - **KafkaRebalance status**: The resource proceeds normally. 
      The forbidden key is ignored.
    - **Cluster Operator log**: WARN "The config key `dryrun` is forbidden because it is managed by the operator.
      The key has been ignored."
    - **Cruise Control log**: N/A

  3. **Invalid config value (Cruise Control rejects)**:
    - **KafkaRebalance status**: `NotReady` condition with CC error message surfaced
    - **Cluster Operator log**: WARN with CC error response
    - **Cruise Control log**: Full error / stack trace

  4. **Unknown config key (Cruise Control rejects)**:
    - **KafkaRebalance status**: `NotReady` with CC error message surfaced.
    - **Cluster Operator log**: WARN with CC error response
    - **Cruise Control log**: Full error / stack trace

  5. **Irrelevant operand for mode (Strimzi rejects)**:
    - **KafkaRebalance status**: `NotReady` with message: "The `nodes` field is not supported in `full` mode.
      Remove the `nodes` field to proceed."
    - **Cluster Operator log**: WARN with same message
    - **Cruise Control log**: N/A

  6. **Missing `nodes` field (Strimzi rejects)**:
    - **KafkaRebalance status**: `NotReady` with message: "The `nodes` field is required in `demote-brokers` mode."
    - **Cluster Operator log**: WARN with same message
    - **Cruise Control log**: N/A

  7. **Invalid broker ID (Strimzi rejects)**:
    - **KafkaRebalance status**: `NotReady` with message identifying the invalid broker ID
    - **Cluster Operator log**: WARN with same message
    - **Cruise Control log**: N/A

  8. **Deprecated targeting field (e.g. "brokers" or "moveReplicasOffVolumes") used with `demote-brokers` (Strimzi rejects)**:
    - **KafkaRebalance status**: `NotReady` with message: "The `brokers` field is not supported in `demote-brokers` mode. Use the `nodes` field instead."
    - **Cluster Operator log**: WARN with same message
    - **Cruise Control log**: N/A

  9. **Mixed broker-level and disk-level demotion in same resource (Strimzi rejects)**:
    - **KafkaRebalance status**: `NotReady` with message: "Cannot mix broker-level and disk-level demotion. 
      All `nodes` entries must either specify `volumeIds` or none must."
    - **Cluster Operator log**: WARN with same message
    - **Cruise Control log**: N/A

### Interaction with other rebalance modes

Broker demotion is independent of other rebalance modes but can be used before or after them manually:

* **add-brokers**: After new brokers are added to the cluster, broker demotion could be used to explicitly transfer partition leadership away from existing brokers to accelerate leadership adoption on newly added brokers. 

* **remove-brokers**: Before decommissioning or scaling down brokers, broker demotion could be performed as a preparatory step to minimize disruption.

* **remove-disks**: Before removing disks from a broker, disk-level demotion could be performed as a preparatory step to transfer leadership away from
    partitions on the targeted disks, minimizing disruption before replica movement begins.

* **full**: After demoting brokers, users could run a `full` mode rebalance to further redistribute leadership across the remaining leader-eligible brokers.

To reduce the complexity of this proposal and its implementation, broker demotion will remain as a manual operation independent of the other rebalance modes and cluster scaling as described in [proposal 078](https://github.com/strimzi/proposals/blob/main/078-auto-rebalancing-cluster-scaling.md).

## Affected/not affected projects

This proposal impacts the Strimzi Cluster Operator in places related to the `KafkaRebalanceAssemblyOperator` and the `KafkaRebalance` API.

## Compatibility

The proposed changes are fully backward compatible:

* **API compatibility**: Adding a new enum value to `KafkaRebalanceMode` does not break existing resources. 
Existing `KafkaRebalance` resources using `full`, `add-brokers`, `remove-brokers`, and `remove-disks` modes continue to work unchanged.

* **CRD compatibility**: The `KafkaRebalance` CRD already includes the `mode`, `nodes`, and `config` fields introduced by [proposal 154](154-kafkarebalance-custom-resource-consolidation.md).
No structural changes to the CRD schema are needed beyond allowing the new `demote-brokers` enum value.

* **Behavioral compatibility**: Existing rebalancing workflows are unaffected. 
The new mode is opt-in and requires explicit user action.

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