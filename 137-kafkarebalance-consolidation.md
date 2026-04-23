# Consolidating the KafkaRebalance API for extensibility

 This proposal addresses the increasing complexity and limited extensibility of the `KafkaRebalance` custom resource API. 
 It introduces a `spec.config` map that replaces mode-specific primitive fields with their upstream Cruise Control key-value equivalents, allowing users to consult Cruise Control documentation directly and enabling support for new parameters without Strimzi API changes.
 It also replaces the action-specific `moveReplicasOffVolumes` field with a generic `volumes` field reusable by any mode that targets specific volumes on specific brokers. 
 These changes establish a clear separation between rebalance mode selection and tuning configuration, preventing API sprawl as new modes are introduced.

## Current situation

The `KafkaRebalance` resource currently supports four distinct modes:
- `full` - Rebalances across all brokers in the cluster.
- `add-brokers` - Moves replicas to newly added brokers.
- `remove-brokers` - Moves replicas out of brokers to be removed.
- `remove-disks` - Moves replicas off specified JBOD volumes of specified brokers so those volumes can be removed.

Over time, as new modes have been added, the API has accumulated an increasing number of top-level fields, many of which are only applicable to specific modes. 
The following table shows which `KafkaRebalanceSpec` parameters are supported by each mode based on the current operator implementation:

| Parameter | `full` | `add-brokers` | `remove-brokers` | `remove-disks` | Notes                                       |
|---|---|---|---|----------------|---------------------------------------------|
| `mode` | -- | -- | -- | --             | Rebalance mode, defaults to `full`          |
| `brokers` | ignored | **required** | **required** | ignored        | List of broker IDs for the operation        |
| `moveReplicasOffVolumes` | ignored | ignored | ignored | **required**   | List of broker/volume ID mappings           |
| `rebalanceDisk` | supported | ignored | ignored | ignored        | Enables intra-broker disk rebalancing       |
| `concurrentIntraBrokerPartitionMovements` | supported | ignored | ignored | ignored        | Concurrent intra-broker partition movements |
| `goals` | supported | supported | supported | ignored        | Optimization goal list                      |
| `skipHardGoalCheck` | supported | supported | supported | ignored        | Whether to skip hard goal validation        |
| `excludedTopics` | supported | supported | supported | ignored        | Regex pattern for topics to exclude         |
| `concurrentPartitionMovementsPerBroker` | supported | supported | supported | ignored        | Concurrent inter-broker partition movements |
| `concurrentLeaderMovements` | supported | supported | supported | ignored        | Concurrent leader movements                 |
| `replicationThrottle` | supported | supported | supported | ignored        | Replication bandwidth throttle (bytes/sec)  |
| `replicaMovementStrategies` | supported | supported | supported | ignored        | Replica movement strategy class list        |

All parameters are currently defined as top-level fields in `KafkaRebalanceSpec`, making it unclear from the API schema alone which fields apply to which mode. 
This information is only enforced at runtime in `KafkaRebalanceAssemblyOperator`, which selects a mode-specific options builder (`RebalanceOptions`, `AddBrokerOptions`, `RemoveBrokerOptions`, or `RemoveDisksOptions`) that inherits common parameters from `AbstractRebalanceOptions`.

### Problems with the current approach

1. **API Sprawl**: Each new mode requires adding new top-level fields, making the API increasingly difficult to understand and maintain.

2. **Poor Field Organization**: Mode-specific and common fields are mixed at the same level, making it unclear which fields apply to which mode without consulting documentation or implementation code.

3. **Documentation Burden**: As more modes are added, the documentation must explain increasingly complex field interdependencies and conditional requirements.

4. **Future Mode Constraints**: Adding new modes becomes increasingly difficult as the top-level namespace becomes crowded with mode-specific fields.

5. **Action-Specific Naming**: The `moveReplicasOffVolumes` field encodes a specific action ("move replicas off") in its name, making it unsuitable for reuse by future modes that target volumes for different purposes (e.g., broker demotion). 
This forces each new volume-related mode to introduce its own field.

## Motivation

The motivation for this proposal stems from several factors:

1. **Prevent API Debt**: As discussed in planning for future modes (like broker demotion support), we should not continue adding primitive fields to the top-level spec for every new mode.

2. **Improve User Experience**: Users should be able to distinguish at a glance what an operation targets (`mode`, `brokers`, `volumes`) and refer to upstream [Cruise Control documentation](https://github.com/linkedin/cruise-control/wiki/REST-APIs) on how different endpoints are tuned.

3. **Maintain Long-term API Sustainability**: The current trajectory will lead to a bloated and confusing API that becomes increasingly difficult to maintain and extend.

4. **Prepare for Volume-Targeting Modes**: Future modes like broker demotion will need to target specific volumes on specific brokers. 
A generic `volumes` field allows these modes to reuse the existing targeting mechanism.

## Proposal

### API Structure Redesign

Consolidate auxiliary configuration fields into a new `spec.config` map, keeping `mode`, `brokers`, and adding a new `volumes` field at the top level of `spec`:

- `mode`: A string representing the rebalancing operation (e.g. `full`, `add-brokers`, `remove-brokers`, `remove-disks`)
- `brokers`: A list of integers identifying the brokers the operation targets.
- `volumes`: A list of objects, each containing a `brokerId` (integer) and `volumeIds` (list of integers), identifying the volumes the operation targets. 
Replaces the current `moveReplicasOffVolumes` field with a generic name that can be reused by any mode that needs to target specific volumes on specific brokers (e.g., `remove-disks` today, `demote-brokers` in the future).
- `config`: a map replacing the existing primitive fields (`skipHardGoalCheck`, `rebalanceDisk`, `excludedTopics`, `concurrentPartitionMovementsPerBroker`, `concurrentIntraBrokerPartitionMovements`, `concurrentLeaderMovements`, `replicationThrottle`, `goals`, `replicaMovementStrategies`) with their upstream Cruise Control key-value equivalents.
These primitive fields are the primary source of API sprawl.
Instead of maintaining Strimzi-specific field names, `config` entries use the keys and values defined by the [Cruise Control REST API](https://github.com/linkedin/cruise-control/wiki/REST-APIs) directly.
This removes the translation layer between Strimzi field names and Cruise Control parameters, allowing users to consult Cruise Control documentation directly and new Cruise Control parameters to be supported without changes to the Strimzi API.

#### Proposed API Structure

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: my-rebalance
spec:
  mode: remove-disks  # full, add-brokers, remove-brokers, remove-disks
  volumes:
    - brokerId: 0
      volumeIds: [1, 2]
    - brokerId: 2
      volumeIds: [1]

  # All auxiliary configurations consolidated under config
  config:
    stop_ongoing_execution: "false"
```

### Field Organization Principles

**Top-level `spec` fields** (mode selector, operands, and optimization configuration):
- `mode` - The rebalancing mode (unchanged)
- `brokers` - List of broker IDs for `add-brokers` and `remove-brokers` modes (unchanged).
When this operand is used for modes where it is not relevant (e.g. `full` or `remove-disks`), an error is written in the status of the `KafkaRebalance` resource and a warning is logged.
When this operand is used with the `remove-disks` mode, where a brokers list could be relevant, the operator will note to use `volumes` instead in the error and log message.
This distinction will be documented as well.
- `volumes` - List of broker/volume ID mappings for `remove-disks` and future volume-targeting modes (replaces `moveReplicasOffVolumes`)
When this operand is used for modes where it is not relevant (e.g. `full`, `add-brokers`, or `remove-brokers`), an error is written in the status of the `KafkaRebalance` resource and warning is logged.
- `config` - A map using upstream parameter names as specified by the [Cruise Control REST API](https://github.com/linkedin/cruise-control/wiki/REST-APIs).
The following config keys will replace the primitive fields that are currently used in the `KafkaRebalance` API:
  - `goals` - Optimization goals (comma-separated string)
  - `skip_hard_goal_check` - Whether to skip hard goal checks
  - `excluded_topics` - Regex pattern for topics to exclude
  - `replica_movement_strategies` - Replica movement strategies (comma-separated string)
  - `rebalance_disk` - Enable intra-broker disk rebalancing
  - `concurrent_partition_movements_per_broker` - Concurrent inter-broker movements per broker
  - `concurrent_intra_broker_partition_movements` - Concurrent intra-broker movements
  - `concurrent_leader_movements` - Concurrent leader movements
  - `replication_throttle` - Replication bandwidth throttle in bytes/sec

### Implementation Strategy

1. **Introduce the new `config` map and `volumes` field** while maintaining backward compatibility:
   - Accept both old top-level primitive fields and new `config` map.
   - Accept both `moveReplicasOffVolumes` and `volumes` (mapped to the same internal representation)
   - If both old and new forms are provided, the new form takes precedence
   - Log deprecation warnings when old top-level primitive fields or `moveReplicasOffVolumes` are used

2. **Update documentation** to promote the new structure while documenting the old structure as deprecated.

3. **Update examples** to show the usage of the deprecated and the new structure.

4. **Add validation** that warns about mixing old and new structures.

5. **Deprecate old fields and plan removal** Mark the old top-level primitive fields: `goals`, `skipHardGoalCheck`, `rebalanceDisk`, `excludedTopics`, `concurrentPartitionMovementsPerBroker`, `concurrentIntraBrokerPartitionMovements`, `concurrentLeaderMovements`,
  `replicationThrottle`, `replicaMovementStrategies` and `moveReplicasOffVolumes` as deprecated in the current API version.
  These fields will be removed in the next API version.

### Validation Improvements

With the new structure, validation is split across two layers:

1. **Mode-specific operand validation**:
   - `brokers` is required and non-empty for `add-brokers` and `remove-brokers` modes
   - `volumes` is required and non-empty for `remove-disks` mode.
   - Strimzi will log a warning and write an error in the `KafkaRebalance` status when `brokers` or `volumes` are provided but irrelevant to the selected mode.

2. **Parameter field validation**:
   - Strimzi won't pre-validate new config parameter entries.
   Strimzi will pass the config parameters as-is to the Cruise Control REST API.
   If Cruise Control returns an error for an invalid or unsupported parameter, Strimzi will transition the `KafkaRebalance` resource to the `NotReady` state and write a warning condition containing the error returned by Cruise Control.
   - Strimzi will write the error returned from Cruise Control REST API to the `KafkaRebalance` status when an invalid configuration parameter is used in conjunction with a specific rebalance "mode".

### Example Configurations

#### Example of a `full` inter-broker rebalance
```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: full-rebalance
spec:
  mode: full
  config:
    goals: "CpuCapacityGoal, NetworkInboundCapacityGoal, DiskCapacityGoal"
    max_partition_movements_in_cluster: "100"
    concurrent_partition_movements_per_broker: "10"
```
#### Example of a `full` intra-broker (Disk) rebalance
```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: full-rebalance
spec:
  mode: full
  config:
    rebalance_disk: "true"
    goals: "IntraBrokerDiskCapacityGoal, IntraBrokerDiskUsageDistributionGoal"
    concurrent_intra_broker_partition_movements: "2"
```

#### Example of an `add-brokers` rebalance
```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: add-brokers-rebalance
spec:
  mode: add-brokers
  brokers: [3, 4]
  config:
    goals: "RackAwareGoal, ReplicaCapacityGoal"
    concurrent_partition_movements_per_broker: "10"
    replication_throttle: "20971520"
```

#### Example of a `remove-disks` rebalance
```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: remove-disks-rebalance
spec:
  mode: remove-disks
  volumes:
    - brokerId: 0
      volumeIds: [1, 2]
    - brokerId: 2
      volumeIds: [1]
  config:
    stop_ongoing_execution: "false"
```

### Future Extensibility

This structure enables cleaner additions for future modes. 
The top-level operand fields (`brokers`, `volumes`) provide a stable, reusable targeting mechanism and allow supporting new optimization parameters without the need to update the Strimzi `KafkaRebalance` API. One example of this would be to add broker demotion support.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator.

## Compatibility

### Backward Compatibility Strategy

The proposal maintains strict backward compatibility. 
Both old and new structures are supported:
- Old top-level primitive fields read as before, with deprecation warnings
- `moveReplicasOffVolumes` is mapped internally to `volumes`
- The new `config` map and `volumes` field are used when present.
- The `config` field takes precedence over deprecated top-level fields if both are present.

### Migration Examples

**Tuning parameters move into `config` using upstream Cruise Control keys:**
```yaml
# Before (deprecated but supported)
spec:
  mode: add-brokers
  brokers: [3, 4]
  goals:
    - RackAwareGoal
  concurrentPartitionMovementsPerBroker: 10

# After 
spec:
  mode: add-brokers
  brokers: [3, 4]
  config:
    goals: "RackAwareGoal"
    concurrent_partition_movements_per_broker: "10"
```

**Volume targeting uses the new field name:**
```yaml
# Before (deprecated but supported)
spec:
  mode: remove-disks
  moveReplicasOffVolumes:
    - brokerId: 0
      volumeIds: [1, 2]

# After
spec:
  mode: remove-disks
  volumes:
    - brokerId: 0
      volumeIds: [1, 2]
```

## Rejected alternatives

### Alternative 1: Keep Current Structure

**Rejected because**:
- Continues the problematic pattern
- Makes the API increasingly difficult to understand and maintain
- Does not scale well as more modes are added