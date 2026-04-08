# Restructuring the KafkaRebalance API for extensibility

This proposal addresses the growing complexity and fragmentation of the `KafkaRebalance` custom resource API by consolidating auxiliary tuning and optimization parameters into a `spec.config` map. 
This would establish a clean separation between the `KafkaRebalance` modes and their configuration in order to prevent further API sprawl as new modes are added.

## Current situation

The `KafkaRebalance` resource currently supports four distinct modes:
- `full` - Rebalances across all brokers in the cluster.
- `add-brokers` - Moves replicas to newly added brokers.
- `remove-brokers` - Moves replicas out of brokers to be removed.
- `remove-disks` - Moves data between JBOD volumes within the same broker

Over time as new modes have been added the API has accumulated an increasing number of top-level fields, many of which are only applicable to specific modes. 
The following table shows which `KafkaRebalanceSpec` parameters are supported by each mode, based on the current operator implementation:

| Parameter | `full` | `add-brokers` | `remove-brokers` | `remove-disks` | Notes |
|---|---|---|---|----------------|---|
| `mode` | -- | -- | -- | --             | Mode selector; defaults to `full` |
| `brokers` | ignored | **required** | **required** | ignored        | List of broker IDs for the operation |
| `moveReplicasOffVolumes` | ignored | ignored | ignored | **required**   | List of broker/volume ID mappings |
| `rebalanceDisk` | supported | ignored | ignored | ignored        | Enables intra-broker disk rebalancing |
| `concurrentIntraBrokerPartitionMovements` | supported | ignored | ignored | ignored        | Concurrent intra-broker partition movements |
| `goals` | supported | supported | supported | ignored        | Optimization goal list |
| `skipHardGoalCheck` | supported | supported | supported | ignored        | Whether to skip hard goal validation |
| `excludedTopics` | supported | supported | supported | ignored        | Regex pattern for topics to exclude |
| `concurrentPartitionMovementsPerBroker` | supported | supported | supported | ignored        | Concurrent inter-broker partition movements |
| `concurrentLeaderMovements` | supported | supported | supported | ignored        | Concurrent leader movements |
| `replicationThrottle` | supported | supported | supported | ignored        | Replication bandwidth throttle (bytes/sec) |
| `replicaMovementStrategies` | supported | supported | supported | ignored        | Replica movement strategy class list |

All parameters are currently defined as top-level fields in `KafkaRebalanceSpec`, making it unclear from the API schema alone which fields apply to which mode. 
This information is only enforced at runtime in `KafkaRebalanceAssemblyOperator`, which selects a mode-specific options builder (`RebalanceOptions`, `AddBrokerOptions`, `RemoveBrokerOptions`, or `RemoveDisksOptions`) that inherits common parameters from `AbstractRebalanceOptions`.

### Problems with the current approach

1. **API Sprawl**: Each new mode requires adding new top-level fields making the API increasingly difficult to understand and maintain.

2. **Poor Field Organization**: Mode-specific and common fields are intermixed at the same level, making it unclear which fields apply to which mode without consulting documentation or implementation code.

3. **Validation Complexity**: The API schema cannot enforce mode-specific field requirements at the CRD level. 
Validation only occurs at runtime, providing poor user experience.

4. **Documentation Burden**: As more modes are added, the documentation must explain increasingly complex field interdependencies and conditional requirements.

5. **Future Mode Constraints**: Adding new modes becomes increasingly difficult as the top-level namespace becomes crowded with mode-specific fields.

6. **Action-Specific Naming**: The `moveReplicasOffVolumes` field encodes a specific action ("move replicas off") in its name, making it unsuitable for reuse by future modes that target volumes for different purposes (e.g., broker demotion). 
This forces each new volume-related mode to introduce its own field.

## Motivation

The motivation for this proposal stems from several factors:

1. **Prevent API Debt**: As discussed in planning for future modes (like broker demotion support), we should not continue adding primitive fields to the top-level spec for every new mode.

2. **Improve User Experience**: Users should be able to distinguish at a glance between what an operation targets (`mode`, `brokers`, `volumes`) and refer to upstream [Cruise Control documentation](https://github.com/linkedin/cruise-control/wiki/REST-APIs) on how different endpoint or "modes" are tuned.

3. **Maintain Long-term API Sustainability**: The current trajectory will lead to a bloated, confusing API that becomes increasingly difficult to maintain and extend.

4. **Prepare for Volume-Targeting Modes**: Future modes like broker demotion will need to target specific volumes on specific brokers. 
A generic, action-neutral `volumes` field allows these modes to reuse the existing targeting mechanism.

## Proposal

### API Structure Redesign

Consolidate auxiliary configuration fields into a new `spec.config` map, keeping `mode`, `brokers`, and `volumes` at the top level of `spec`:

- `mode`: A string representing the rebalancing operation (e.g. `full`, `add-brokers`, `remove-brokers`, `remove-disks`)
- `brokers`: a list of integers identifying the brokers the operation targets.
- `volumes`: a list of objects, each containing a `brokerId` (integer) and `volumeIds` (list of integers), identifying the volumes the operation targets. 
Replaces the current `moveReplicasOffVolumes` with an action-neutral name that can be reused by any mode that needs to target specific volumes on specific brokers (e.g., `remove-disks` today, `demote-brokers` in the future).
- `config`: a map replacing the existing primitive fields (`skipHardGoalCheck`, `rebalanceDisk`, `excludedTopics`, `concurrentPartitionMovementsPerBroker`, `concurrentIntraBrokerPartitionMovements`, `concurrentLeaderMovements`, `replicationThrottle`, `goals`, `replicaMovementStrategies`) with their upstream Cruise Control key-value equivalents. 
These primitive fields are the primary source of API sprawl. 
Instead of maintaining Strimzi-specific field names, `config` entries use the keys and values defined by the [Cruise Control REST API](https://github.com/linkedin/cruise-control/wiki/REST-APIs) directly. 
This eliminates the translation layer between Strimzi field names and Cruise Control parameters allowing users to consult Cruise Control documentation directly and allows new Cruise Control parameters to be supported without changes to the Strimzi API.

#### Proposed API Structure

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
spec:
  mode: remove-disks  # full, add-brokers, remove-brokers, remove-disks
  brokers: [3, 4]
  volumes:
    - brokerId: 0
      volumeIds: [1, 2]
    - brokerId: 2
      volumeIds: [1]

  # All auxiliary configuration consolidated under config
  config:
    rebalance_disk: "true"
    goals: "RackAwareGoal, ReplicaCapacityGoal"
    skip_hard_goal_check: "false"
    excluded_topics: "test-.*"
    replica_movement_strategies: "BaseReplicaMovementStrategy"
    concurrent_partition_movements_per_broker: "5"
    concurrent_intra_broker_partition_movements: "2"
    concurrent_leader_movements: "1000"
    replication_throttle: "0"
```

### Field Organization Principles

**Top-level `spec` fields** (mode selector, operands, and optimization parameters):
- `mode` - The rebalancing mode (unchanged)
- `brokers` - List of broker IDs for `add-brokers` and `remove-brokers` modes (unchanged).
When this operand is used for modes where it is not relevant (e.g. `full` or `remove-disks`) an error is written in the status of the `KafkaRebalance` resource and warning is logged.
For the case where this operand is used in conjunction with the `remove-disks` mode, where a brokers list could be relevant, the operator will note to use `volumes` instead in the error and log message.
This distinction will be documented as well.
- `volumes` - List of broker/volume ID mappings for `remove-disks` and future volume-targeting modes (replaces `moveReplicasOffVolumes`)
When this operand is used for modes where it is not relevant (e.g. `full`, `add-brokers`, or `remove-brokers`) an error is written in the status of the `KafkaRebalance` resource and warning is logged.
- `config` - A map using upstream parameter names as specified by the [Cruise Control REST API](https://github.com/linkedin/cruise-control/wiki/REST-APIs)). 
Here is a list of the parameters names that will replace the existing primitive fields that are used in the `KafkaRebalance` API today:
  - `goals` - Optimization goals (comma-separated string)
  - `skip_hard_goal_check` - Whether to skip hard goal checks
  - `excluded_topics` - Regex pattern for topics to exclude
  - `replica_movement_strategies` - Replica movement strategies (comma-separated string)
  - `rebalance_disk` - Enable intra-broker disk rebalancing (for `full`)
  - `concurrent_partition_movements_per_broker` - Concurrent inter-broker movements per broker
  - `concurrent_intra_broker_partition_movements` - Concurrent intra-broker movements (for `full`)
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

### Validation Improvements

With the new structure, validation is split across two layers:

1. **Mode-specific operand validation**:
   - `brokers` is required and non-empty for `add-brokers` and `remove-brokers` modes
   - `volumes` is required and non-empty for `remove-disks` mode
   - Strimzi will log a warning and write an error in the `KafkaRebalance` status when `brokers` or `volumes` are provided but irrelevant to the selected mode

2. **Config field validation**:
   - Strimzi will write the error returned from Cruise Control REST API to the `KafkaRebalance` status when an invalid configuration parameter is used in conjunction with a specific rebalance "mode".

### Example Configurations

#### Full Rebalance with Disk Rebalancing
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: full-rebalance
spec:
  mode: full
  config:
    rebalance_disk: "true"
    goals: "RackAwareGoal, ReplicaCapacityGoal"
    concurrent_partition_movements_per_broker: "5"
    concurrent_intra_broker_partition_movements: "2"
    replication_throttle: "10485760" 
```

#### Add Brokers Rebalance
```yaml
apiVersion: kafka.strimzi.io/v1beta2
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

#### Remove Disks Rebalance
```yaml
apiVersion: kafka.strimzi.io/v1beta2
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
```

### Future Extensibility

This structure enables cleaner additions for future modes. 
The top-level operand fields (`brokers`, `volumes`) provide a stable, reusable targeting mechanism and allow supporting new optimization parameters without the need to update the Strimzi `KafkaRebalance` API. One example of this would be to add broker demotion support.

## Affected/not affected projects

- strimzi-kafka-operator

## Compatibility

### Backward Compatibility Strategy

The proposal maintains strict backward compatibility. 
Both old and new structures are supported
- Old top-level primitive fields reads as before, with deprecation warnings
- `moveReplicasOffVolumes` is mapped internally to `volumes`
- New `config` map and `volumes` field is used when present.
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

