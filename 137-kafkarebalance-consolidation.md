# Consolidating the KafkaRebalance API for extensibility

 This proposal addresses the increasing complexity and limited extensibility of the `KafkaRebalance` custom resource API. 
 It introduces a `spec.config` Map<String, String> that replaces mode-specific primitive fields with their upstream Cruise Control key-value equivalents, allowing users to consult Cruise Control documentation directly and enabling support for new parameters without Strimzi API changes.
 It also replaces the separate `brokers` and `moveReplicasOffVolumes` fields with a unified `nodes` field that identifies target brokers and, optionally, specific volumes on those brokers, reusable by any mode.
 These changes establish a clear separation between rebalance mode selection, targeting, and tuning configuration, preventing API sprawl as new modes are introduced.

## Current situation

The `KafkaRebalance` resource currently supports four distinct modes:
- `full` - Rebalances across all brokers in the cluster.
- `add-brokers` - Moves replicas to newly added brokers.
- `remove-brokers` - Moves replicas out of brokers to be removed.
- `remove-disks` - Moves replicas off specified JBOD volumes of specified brokers so those volumes can be removed.

Over time, as new modes have been added, the API has accumulated an increasing number of top-level fields, many of which are only applicable to specific modes. 
The following table shows which `KafkaRebalanceSpec` parameters are supported by each mode based on the current operator implementation:

| Parameter                                 | `full`   | `add-brokers`   | `remove-brokers` | `remove-disks` | Notes                                        |
|-------------------------------------------|----------|-----------------|------------------|----------------|----------------------------------------------|
| `mode`                                    | --       | --              | --               | --             | Rebalance mode, defaults to `full`           |
| `brokers`                                 | ignored  | **required**    | **required**     | ignored        | List of broker IDs for the operation         |
| `moveReplicasOffVolumes`                  | ignored  | ignored         | ignored          | **required**   | List of broker/volume ID mappings            |
| `rebalanceDisk`                           | supported| ignored         | ignored          | ignored        | Enables intra-broker disk rebalancing        |
| `concurrentIntraBrokerPartitionMovements` | supported| ignored         | ignored          | ignored        | Concurrent intra-broker partition movements  |
| `goals`                                   | supported| supported       | supported        | ignored        | Optimization goal list                       |
| `skipHardGoalCheck`                       | supported| supported       | supported        | ignored        | Whether to skip hard goal validation         |
| `excludedTopics`                          | supported| supported       | supported        | ignored        | Regex pattern for topics to exclude          |
| `concurrentPartitionMovementsPerBroker`   | supported| supported       | supported        | ignored        | Concurrent inter-broker partition movements  |
| `concurrentLeaderMovements`               | supported| supported       | supported        | ignored        | Concurrent leader movements                  |
| `replicationThrottle`                     | supported| supported       | supported        | ignored        | Replication bandwidth throttle (bytes/sec)   |
| `replicaMovementStrategies`               | supported| supported       | supported        | ignored        | Replica movement strategy class list         |

All parameters are currently defined as top-level fields in `KafkaRebalanceSpec`, making it unclear from the API schema alone which fields apply to which mode. 
This information is only enforced at runtime in `KafkaRebalanceAssemblyOperator`, which selects a mode-specific options builder (`RebalanceOptions`, `AddBrokerOptions`, `RemoveBrokerOptions`, or `RemoveDisksOptions`) that inherits common parameters from `AbstractRebalanceOptions`.

### Problems with the current approach

1. **API Sprawl**: Each new mode requires adding new top-level fields, making the API increasingly difficult to understand and maintain.

2. **Poor Field Organization**: Mode-specific and common fields are mixed at the same level, making it unclear which fields apply to which mode without consulting the upstream Cruise Control documentation or implementation code.

3. **Documentation Burden**: As more modes are added, the documentation must explain increasingly complex field interdependencies and conditional requirements.

4. **Future Mode Constraints**: Adding new modes becomes increasingly difficult as the top-level namespace becomes crowded with mode-specific fields.

5. **Action-Specific Naming**: The `moveReplicasOffVolumes` field encodes a specific action ("move replicas off") in its name, making it unsuitable for reuse by future modes that target volumes for different purposes (e.g. [broker demotion](https://github.com/strimzi/strimzi-kafka-operator/issues/11907)).
This forces each new volume-related mode to introduce its own field.

## Motivation

The motivation for this proposal stems from several factors:

1. **Prevent API Debt**: As discussed in planning for future modes (like [broker demotion support](https://github.com/strimzi/strimzi-kafka-operator/issues/11907)), we should not continue adding primitive fields to the top-level spec for every new mode.

2. **Improve User Experience**: Users should be able to distinguish at a glance what an operation targets (`mode`, `nodes`) and refer to upstream [Cruise Control documentation](https://github.com/linkedin/cruise-control/wiki/REST-APIs) on how different endpoints are tuned.

3. **Maintain Long-term API Sustainability**: The current trajectory will lead to a bloated and confusing API that becomes increasingly difficult to maintain and extend.

4. **Eliminate Operand Ambiguity**: With separate `brokers` and `volumes` fields, modes that can target either level of granularity (e.g. [broker demotion](https://github.com/strimzi/strimzi-kafka-operator/issues/11907)) force users to choose between two fields for the same mode.
A unified `nodes` field with optional `volumeIds` resolves this by providing a single targeting mechanism that works at both broker and volume granularity.

## Proposal

### API Structure Redesign

Consolidate auxiliary configuration fields into a new `spec.config` field, replace the separate `brokers` and `moveReplicasOffVolumes` fields with a unified `nodes` field, and keep `mode` at the top level of `spec`:

- `mode`: A string representing the rebalancing operation (e.g. `full`, `add-brokers`, `remove-brokers`, `remove-disks`)
- `nodes`: A list of `BrokerAndVolumeIds` objects, each containing a `brokerId` (integer) and an optional `volumeIds` (list of integers), identifying the brokers and optionally the specific volumes the operation targets.
  This field replaces the separate `brokers` and `moveReplicasOffVolumes` fields with a single targeting mechanism.
  Accepted in 3 out of 4 of the current rebalancing modes.
  For broker-targeting modes (`add-brokers`, `remove-brokers`), only `brokerId` is specified and `volumeIds` must not be present.
  For volume-targeting modes (`remove-disks`), both `brokerId` and `volumeIds` are required.
  Future modes (e.g. `demote-brokers`) can use `nodes` with or without `volumeIds` to target either whole brokers or specific volumes, without ambiguity about which field to use.
  When `nodes` is provided for modes where it is not relevant (e.g. `full`), the resource transitions to `NotReady` with a message explaining that the `nodes` field is not supported for the selected mode.
- `config`: a field of type `Map<String, String>` replacing the existing primitive fields with their upstream [Cruise Control REST API](https://github.com/linkedin/cruise-control/wiki/REST-APIs) key-value equivalents:
  - `skipHardGoalCheck` (new config key: `skip_hard_goal_check`, e.g. `"true"`):
    Boolean value whether to skip hard goal checks.
    Accepted in 3 out of 4 of the current rebalancing modes.
    Although a top-level primitive boolean offers convenient CRD validation, placing it under `.spec.config` alongside the other configuration options is more consistent, intuitive, and scalable.
    The field is also expected to be used infrequently so the convenience tradeoff affects few users in practice.
  - `rebalanceDisk` (new config key: `rebalance_disk`, e.g. `"true"`):
    Enable intra-broker disk rebalancing.
    Accepted in 1 out of 4 of the current rebalancing modes and expected to be used frequently.
    However, when enabled, fields such as `replicationThrottle`, `goals`, and `concurrentPartitionMovementsPerBroker` are ignored. 
    A top-level field that invalidates other top-level fields creates a confusing user experience.
    This is a composability issue in the upstream Cruise Control API.
    Placing the field under `.spec.config` keeps the problem in the Cruise Control API's domain while providing flexibility if the Cruise Control API evolves to fix it.
  - `excludedTopics` (new config key: `excluded_topics`, e.g. `"__consumer_offsets|__transaction_state"`):
    Regex pattern for topics to exclude.
    Accepted in 3 out of 4 of the current rebalancing modes and is expected to see moderate to frequent use.
    However, CRD validation can only verify the value is a string, not whether the referenced topics exist, so a top-level field provides little practical benefit over `.spec.config`. 
    Moving it to `.spec.config` alongside other configuration options keeps the API surface consistent.
  - `concurrentPartitionMovementsPerBroker` (new config key: `concurrent_partition_movements_per_broker`, e.g. `"10"`):
    Concurrent inter-broker movements.
    Accepted in 3 out of 4 of the current rebalancing modes.
    We expect moderate usage of this field, but representing it as a primitive provides limited benefit.
    While CRD validation can ensure the value is non-negative, this alone does not justify maintaining it as a top-level field. 
    Moving this field under `.spec.config` alongside the other configuration options provides a more consistent and intuitive user experience.
  - `concurrentIntraBrokerPartitionMovements` (new config key: `concurrent_intra_broker_partition_movements`, e.g. `"2"`):
    Concurrent intra-broker movements per broker.
    Accepted in 1 out of 4 of the current rebalancing modes and expected to be used infrequently.
    Like the `rebalanceDisk` field, it cannot be combined with other rebalancing options in the current Cruise Control REST API. 
    It is only applicable when `rebalanceDisk` is set to `true`.
    This is a composability issue in the upstream Cruise Control API.
    Placing the field under `.spec.config` keeps the problem in the upstream Cruise Control API's domain while providing flexibility if the upstream Cruise Control API evolves to fix it.
  - `concurrentLeaderMovements` (new config key: `concurrent_leader_movements`, e.g. `"500"`):
    Concurrent leader movements.
    Accepted in 3 out of 4 of the current rebalancing modes.
    We expect low to moderate usage of this field, but representing it as a primitive provides limited benefit.
    While CRD validation can ensure the value is non-negative, this alone does not justify maintaining it as a top-level field. 
    Moving this field under `.spec.config` alongside the other configuration options provides a more consistent and intuitive user experience.
  - `replicationThrottle` (new config key: `replication_throttle`, e.g. `"20971520"`):
    Replication bandwidth throttle in bytes/sec.
    Accepted in 3 out of 4 of the current rebalancing modes.
    We expect moderate usage of this field but representing it as a primitive provides limited benefit.
    While CRD validation can ensure the value is non-negative, this alone does not justify maintaining it as a top-level field. 
    Moving this field under `.spec.config` alongside the other configuration options provides a more consistent and intuitive user experience.
  - `goals` (new config key: `goals`, e.g. `"RackAwareGoal,ReplicaCapacityGoal"`):
    Optimization goals (comma-separated string).
    Accepted in 3 out of 4 of the current rebalancing modes.
    We expect high usage of this field but representing it as a List<String> provides limited benefit other than formatting.
    The CRD validation can verify the value is a List<String> but cannot check whether the referenced goals are valid.
    Moving this field under `.spec.config` alongside the other configuration options provides a more consistent user experience and its frequent use does not depend on top-level visibility.
    However, users who currently specify a YAML list will need to join them into a single comma-separated string value.
  - `replicaMovementStrategies` (new config key: `replica_movement_strategies`, e.g. `"PrioritizeSmallReplicaMovementStrategy"`):
    Replica movement strategies (comma-separated string).
    Accepted in 3 out of 4 of the current rebalancing modes.
    We expect infrequent usage of this field and representing it as a primitive provides limited benefit.
    While CRD validation can verify that the value is a string, it does not check whether the referenced movement strategies are valid so this alone does not justify maintaining it as a top-level field.
    Moving this field under `.spec.config` alongside the other configuration options provides a more consistent and intuitive user experience.
    However, users who currently specify a YAML list will need to join them into a single comma-separated string value.


  These primitive fields are the primary source of API sprawl.
  Instead of maintaining Strimzi-specific field names, `config` entries use the keys and values defined by the [Cruise Control REST API](https://github.com/linkedin/cruise-control/wiki/REST-APIs) directly.
  This removes the translation layer between Strimzi field names and Cruise Control parameters, allowing users to consult Cruise Control documentation directly and new Cruise Control parameters to be supported without changes to the Strimzi API.
  See [Error Examples](#error-examples) for how invalid or unsupported config values are surfaced to users.

#### Type Safety Tradeoff

Moving parameters from typed primitive fields to a `Map<String, String>` field means CRD-level type validation is no longer enforced at the schema level.
This is the same tradeoff that other Strimzi components have made — `Kafka.spec.kafka.config` and `Kafka.spec.cruiseControl.config` both use `Map<String, String>` configuration fields to gain full extensibility.
In practice, the loss is limited: CRD validation for the existing fields only checks basic type constraints (non-negative integers, non-null strings) and cannot validate semantic correctness (e.g. whether a goal name is valid or an excluded topic regex matches any topics).
Invalid values passed through `spec.config` will be caught by Cruise Control at request time and the operator will surface the error by transitioning the `KafkaRebalance` resource to `NotReady` with a descriptive status condition.

#### Proposed API Structure

The selected fields mentioned in the [API Structure Redesign](#api-structure-redesign) move into `.spec.config` field using upstream Cruise Control keys.

**Example of a `full` inter-broker rebalance with all the fields moved to `.spec.config`:**
```yaml
# Before (deprecated but supported)
spec:
  mode: full
  goals:
    - RackAwareGoal
  skipHardGoalCheck: true
  excludedTopics: "internal-.*"
  concurrentPartitionMovementsPerBroker: 10
  concurrentLeaderMovements: 500
  replicationThrottle: 10485760
  replicaMovementStrategies:
     - com.linkedin.kafka.cruisecontrol.executor.strategy.PrioritizeSmallReplicaMovementStrategy

# After
spec:
  mode: full
  config:
    goals: "RackAwareGoal"
    skip_hard_goal_check: "true"
    excluded_topics: "internal-.*"
    concurrent_partition_movements_per_broker: "10"
    concurrent_leader_movements: "500"
    replication_throttle: "10485760"
    replica_movement_strategies: "com.linkedin.kafka.cruisecontrol.executor.strategy.PrioritizeSmallReplicaMovementStrategy"
```

**Example of a `full` intra-broker rebalance with all the fields moved to `.spec.config`:**
```yaml
# Before (deprecated but supported)
spec:
  mode: full
  rebalanceDisk: true
  skipHardGoalCheck: true
  excludedTopics: "internal-.*"
  concurrentIntraBrokerPartitionMovements: 5
  replicaMovementStrategies:
     - com.linkedin.kafka.cruisecontrol.executor.strategy.PrioritizeSmallReplicaMovementStrategy

# After
spec:
  mode: full
  config:
    rebalance_disk: "true"
    skip_hard_goal_check: "true"
    excluded_topics: "internal-.*"
    concurrent_intra_broker_partition_movements: "5"
    replica_movement_strategies: "com.linkedin.kafka.cruisecontrol.executor.strategy.PrioritizeSmallReplicaMovementStrategy"
```

**Example of an `add-brokers` rebalance with all the fields moved to `.spec.config` and `brokers` to `nodes`:**
```yaml
# Before (deprecated but supported)
spec:
  mode: add-brokers
  brokers: [3, 4]
  goals:
    - RackAwareGoal
  skipHardGoalCheck: true
  excludedTopics: "internal-.*"
  concurrentPartitionMovementsPerBroker: 10
  concurrentLeaderMovements: 500
  replicationThrottle: 10485760
  replicaMovementStrategies:
     - com.linkedin.kafka.cruisecontrol.executor.strategy.PrioritizeSmallReplicaMovementStrategy

# After
spec:
  mode: add-brokers
  nodes:
    - brokerId: 3
    - brokerId: 4
  config:
    goals: "RackAwareGoal"
    skip_hard_goal_check: "true"
    excluded_topics: "internal-.*"
    concurrent_partition_movements_per_broker: "10"
    concurrent_leader_movements: "500"
    replication_throttle: "10485760"
    replica_movement_strategies: "com.linkedin.kafka.cruisecontrol.executor.strategy.PrioritizeSmallReplicaMovementStrategy"
```

**Example of a `remove-disks` rebalance with `moveReplicasOffVolumes` moved to `nodes`:**
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
  nodes:
    - brokerId: 0
      volumeIds: [1, 2]
```

### Implementation Strategy

1. **Introduce the new `config` and `nodes` field** while maintaining backward compatibility:
   - Add a `config` field of type Map<String, String> and a `nodes` field of type `List<BrokerAndVolumeIds>` to the `KafkaRebalanceSpec` alongside the existing fields.
     The `nodes` field reuses the existing `BrokerAndVolumeIds` type with one modification:
     - Update `@Description` annotations to be mode-neutral (e.g. "ID of the broker to target" instead of "ID of the broker that contains the disk from which you want to move the partition replicas").
   - Mark the following legacy fields in the `KafkaRebalanceSpec` with the `@Deprecated` and `@DeprecatedProperty` annotations: `brokers`, `goals`, `skipHardGoalCheck`, `rebalanceDisk`, `excludedTopics`, `concurrentPartitionMovementsPerBroker`, `concurrentIntraBrokerPartitionMovements`, `concurrentLeaderMovements`, `replicationThrottle`, `replicaMovementStrategies`, and `moveReplicasOffVolumes`
   - These fields will be removed in the next major API version, Strimzi 2.0.
     A conversion webhook or migration tool will be provided to automatically convert existing `KafkaRebalance` resources from the legacy fields to the new `spec.config` and `spec.nodes` structure, following the same approach used for the 1.0 migration.

   No existing fields are removed so the legacy fields (for example, `brokers`, `moveReplicasOffVolumes`, and `goals`) will continue to be supported when specified in the `KafkaRebalance` resource.

2. **Convert legacy fields to new fields at the beginning of reconciliation**.
   At the start of each reconciliation, before any other processing, the operator converts deprecated fields to their new equivalents one by one.
   This follows the same process used for the [MirrorMaker 2](https://github.com/strimzi/strimzi-kafka-operator/blob/0.49.0/v1-api-conversion/src/main/java/io/strimzi/kafka/api/conversion/v1/converter/conversions/MirrorMaker2Conversions.java) and [Connect API](https://github.com/strimzi/strimzi-kafka-operator/blob/0.49.0/v1-api-conversion/src/main/java/io/strimzi/kafka/api/conversion/v1/converter/conversions/ConnectAndConnectorConversions.java) changes and other v1 API related changes:
     - Take the `KafkaRebalance` resource at the beginning of reconciliation.
     - For each deprecated field, check if it is set.
     - If the deprecated field is set and the corresponding new field is empty, move the value from the deprecated field to the new field.
     - If both the deprecated field and the corresponding new field are set, the new field takes priority.
       The operator issues a warning that the fields conflict and the deprecated value is being ignored.
     - Mixing of old and new fields is allowed.
       This way, users can migrate fields incrementally without needing to convert all fields at once.
     - Deprecation warnings are issued automatically when any deprecated field is used.
     - After conversion, the rest of the reconciliation code uses only the new `spec.config` and `spec.nodes` fields.
     - In the future, a gatekeeper plugin could be used to perform this conversion at admission time.

3. **Validation**:

    - **Mode-specific operand validation**:
      - `nodes` is required and non-empty for `add-brokers`, `remove-brokers`, and `remove-disks` modes.
      - For `add-brokers` and `remove-brokers` modes, `volumeIds` must not be present on any entry in `nodes`.
      - For `remove-disks` mode, `volumeIds` is required and non-empty on every entry in `nodes`.
      - When `nodes` is provided but irrelevant to the selected mode (e.g. `full`), the `nodes` field is not accepted.
      - If any of the above constraints are violated, the resource transitions to `NotReady` with a message identifying the validation error.

    - **Parameter field validation**
      - Following the same pattern used for [`kafka.config`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/api/src/main/java/io/strimzi/api/kafka/model/kafka/KafkaClusterSpec.java#L56-L72) and [`cruiseControl.config`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/api/src/main/java/io/strimzi/api/kafka/model/kafka/cruisecontrol/CruiseControlSpec.java#L46-L50) sections of the `Kafka` resource configuration, we will maintain `FORBIDDEN_PREFIXES` and `FORBIDDEN_PREFIX_EXCEPTIONS` constants in the KafkaRebalanceSpec.
       These constants will help filter keys in `spec.config` that conflict with operator-managed behavior or top-level fields, for example:
      - `dryrun`: managed by the operator as part of the rebalance proposal lifecycle.
      - `json`, `verbose`: response format parameters managed internally by the operator.
      - `brokerid`: broker targeting is handled by the top-level `nodes` field.
      - `brokerid_and_logdirs`: volume targeting is handled by the top-level `nodes` field (via `volumeIds`).

        If any forbidden key is present in `spec.config`, the operator will log a warning and ignore the key before passing the configuration to Cruise Control.
        Supported config parameters will be passed as-is to the Cruise Control REST API.
      - If Cruise Control returns an error for a config parameter whether due to an invalid value or an unknown key, Strimzi will transition the `KafkaRebalance` resource to the `NotReady` state and surface the error in a warning condition on the resource's status.

4. **Update examples** to encourage use of new API structure
   - Ensure the packaged `KafkaRebalance` resource examples are updated to use the new API structure (replacing `brokers` and `moveReplicasOffVolumes` with `nodes`).

5. **Update documentation** to promote the new structure while documenting the old structure as deprecated and point to the upstream [Cruise Control REST API Wiki](https://github.com/linkedin/cruise-control/wiki/REST-APIs) where needed.
   - Add a table to the documentation mapping new fields to the corresponding legacy fields that they are replacing.
   - Add examples to show how to migrate from the legacy to the new API structure.
   - Using the `FORBIDDEN_PREFIXES` and `FORBIDDEN_PREFIX_EXCEPTIONS` constants maintained in the KafkaRebalanceSpec we will generate API documentation listing which upstream Cruise Control fields are unsupported by Strimzi in the same way we do for [`cruiseControl.config`](https://strimzi.io/docs/operators/latest/configuring#type-CruiseControlSpec-schema-reference) in the Kafka resource.

#### Filtered Parameters

| Parameter               | Why it is filtered                                                                                                                              |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| `dryrun`                | Strimzi controls this via the rebalance state machine. Proposal generation vs. execution are separate states (`ProposalReady` -> `Rebalancing`) |
| `json`                  | Hardcoded to `true` by Strimzi. Changing this would break response parsing.                                                                     |
| `verbose`               | Changing verbosity could break status reporting.                                                                                                |
| `super_verbose`         | Same as `verbose`                                                                                                                               |
| `brokerid`              | Managed via `spec.nodes` top-level field (`brokerId`).                                                                                          |
| `brokerid_and_logdirs`  | Managed via `spec.nodes` top-level field (`brokerId` + `volumeIds`).                                                                            |

#### Validation Examples

  1. **Mixing old and new fields (Strimzi converts with warnings)**:
    - **KafkaRebalance status**: The resource proceeds normally.
      A warning condition is added if both a deprecated field and its corresponding new field are set, indicating that the deprecated value is being ignored.
    - **Cluster Operator log**: WARN for each deprecated field used (deprecation notice), and an additional WARN if both old and new field are set (conflict notice, e.g. "Both `concurrentPartitionMovementsPerBroker` and `spec.config[concurrent_partition_movements_per_broker]` are set.
     Using the value from `spec.config` and ignoring the deprecated field.")
    - **Cruise Control log**: N/A

  2. **Forbidden config key (Strimzi filters)**:
    - **KafkaRebalance status**: The resource proceeds normally. The forbidden key is ignored.
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

  6. **`volumeIds` present on wrong mode (Strimzi validates)**:
    - **KafkaRebalance status**: `NotReady` with message: "The `volumeIds` field in `nodes` entries is not supported in `add-brokers` mode.
      Remove `volumeIds` from all `nodes` entries to proceed."
    - **Cluster Operator log**: WARN with same message
    - **Cruise Control log**: N/A

### Example Configurations

The following examples use the new API structure.
Note that `spec.config` accepts any key supported by and passed to the Cruise Control REST API provided it is not in the Strimzi forbidden parameter list (see [Filtered Parameters](#filtered-parameters)).
Keys like `max_partition_movements_in_cluster` and `stop_ongoing_execution` shown below are not migrated from existing fields but are newly accessible without any Strimzi API changes.

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
  nodes:
    - brokerId: 3
    - brokerId: 4
  config:
    goals: "RackAwareGoal, ReplicaCapacityGoal"
    concurrent_partition_movements_per_broker: "10"
    replication_throttle: "20971520"
```

#### Example of a `remove-brokers` rebalance

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: remove-brokers-rebalance
spec:
  mode: remove-brokers
  nodes:
    - brokerId: 3
    - brokerId: 4
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
  nodes:
    - brokerId: 0
      volumeIds: [1, 2]
    - brokerId: 2
      volumeIds: [1]
  config:
    stop_ongoing_execution: "false"
```

### Future Extensibility

#### Add Broker Demotion Support

This structure enables cleaner additions for future modes.
The unified `nodes` field provides a stable, reusable targeting mechanism that works at both broker and volume granularity, and `spec.config` allows supporting new optimization parameters without the need to update the Strimzi `KafkaRebalance` API.
One example of this would be to add [broker demotion](https://github.com/strimzi/strimzi-kafka-operator/issues/11907) support.

With the proposed API, such a feature could look like this:

##### Example of `demote-brokers` rebalance, demoting brokers

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: demote-brokers-example-1
spec:
  mode: demote-brokers
  nodes:
    - brokerId: 3
    - brokerId: 4
  config:
    exclude_recently_demoted_brokers: "true"
```

##### Example of `demote-brokers` rebalance, demoting disks of brokers

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: demote-brokers-example-2
spec:
  mode: demote-brokers
  nodes:
    - brokerId: 3
      volumeIds: [1, 2]
    - brokerId: 4
      volumeIds: [1]
  config:
    exclude_recently_demoted_brokers: "true"
```

Note how `demote-brokers` uses the same `nodes` field regardless of whether it targets whole brokers or specific volumes.

#### Cruise Control Parameter Schema Validation

If Cruise Control were to expose a parameter schema per endpoint, the operator could cache it and validate `spec.config` entries locally before making API calls.
This would provide a faster feedback loop without coupling the Strimzi API to specific Cruise Control parameters.
However, this is not possible with the current state of Cruise Control and is outside the scope of this proposal.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator.

## Compatibility

### Backward Compatibility Strategy

The proposal maintains strict backward compatibility.
Both old and new fields are supported simultaneously.
At the beginning of each reconciliation, the operator converts deprecated fields to their new equivalents one by one.
If the deprecated field is set and the new field is empty, the value is copied to the new field.
If both are set, the new field takes priority and a warning is issued.
After conversion, the rest of the reconciliation code uses only the new fields.
See [Implementation Strategy, step 2](#implementation-strategy) for the conversion process.
All deprecated fields will be removed in Strimzi 2.0.

## Rejected alternatives

### Alternative 1: Keep Current Structure

**Rejected because**:
- Continues the problematic pattern
- Makes the API increasingly difficult to understand and maintain
- Does not scale well as more modes are added

### Alternative 2: Separate `brokers` and `volumes` Operand Fields

Instead of a unified `nodes` field, use two separate top-level operand fields: `brokers` of type `List<Integer>` for broker-targeting modes and `volumes` of type `BrokerAndVolumeIds` for volume-targeting modes.

**Rejected because**:
- Creates ambiguity for modes that can target either brokers or volumes (e.g. `demote-brokers`).
- Users must decide which of two fields to use for the same mode depending on whether they are targeting whole brokers or specific volumes.
- Does not clearly signal whether future modes like broker-level demotion should use `brokers` or `volumes`, leading to a confusing user experience.
- The unified `nodes` field with optional `volumeIds` naturally expresses both levels of granularity in a single, consistent structure.
