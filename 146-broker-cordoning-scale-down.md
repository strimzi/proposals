# Support broker cordoning in auto-rebalancing on scale down

## Summary

This proposal extends the existing auto-rebalancing feature to leverage Apache Kafka's [KIP-1066](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1066%3A+Mechanism+to+cordon+brokers+and+log+directories) broker cordoning mechanism during scale down operations.
By automatically cordoning brokers scheduled for removal before initiating partition reassignment, we prevent new partition assignments to those brokers during such operation, ensuring a clean and efficient scale down process.

## Current situation

Strimzi's auto-rebalancing feature automates the process of rebalancing Kafka clusters during scale up and scale down operations.
In particular, when scaling down:

1. The operator detects brokers to be removed, but it holds on with the scale down process
2. A `KafkaRebalance` resource is automatically created with `spec.mode: remove-brokers`
3. Cruise Control moves partitions off the brokers being removed
4. After successful rebalancing, the brokers are removed from the cluster and the scale down completes

However, a critical limitation exists: during the rebalancing phase, Kafka's partition placement logic can still assign new partitions (from newly created topics or partition additions) to the brokers being removed.
This creates several problems:

- Cruise Control must perform additional rebalancing work to move newly placed partitions
- In clusters with frequent topic creation, the scale down process may never complete as new partitions continuously get assigned to brokers being removed
- The overall scale down operation takes longer due to repeated rebalancing cycles

Currently, there is no mechanism in Strimzi to prevent this behavior, mainly because there is nothing within Apache Kafka to help with.

## Motivation

Apache Kafka 4.3 introduces a native mechanism to "cordon" individual log directories on brokers, through [KIP-1066](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1066%3A+Mechanism+to+cordon+brokers+and+log+directories).
When a log directory is cordoned:

- It remains fully functional for existing partitions
- New partition assignments to that specific log directory are blocked
- The controller can still place partitions on the broker's **other non-cordoned log directories**
- When **ALL** log directories on a broker are cordoned, the controller excludes the entire broker from partition placement decisions

Cordoning log directories is done by setting the folders list within the `cordoned.log.dirs` property.
This is a dynamic per-broker configuration property, so it can be applied via the Kafka Admin API without requiring a broker restart.
For the scale-down use case, the Strimzi Cluster Operator will cordon **all log directories** on brokers being removed (by setting `cordoned.log.dirs=*`), ensuring those brokers are completely excluded from new partition assignments.

This capability directly addresses the auto-rebalancing scale down limitation.
By automatically cordoning the brokers being removed, before initiating the rebalance, Strimzi gets the following:

- Once cordoned, no new partitions can be assigned to brokers being removed
- Cruise Control only needs to move existing partitions, not chase newly assigned ones
- The scale down process would have an end when all existing partitions are moved

This enhancement is particularly valuable for:

- Large clusters with high topic creation rates
- Production environments requiring predictable maintenance windows
- Automated scaling scenarios where reliability is critical

## Proposal

### Overview

Extend the auto-rebalancing mechanism to automatically cordon brokers during the scale down process by integrating `cordoned.log.dirs` into the existing per-broker configuration pipeline.
The operator will:

1. Detect brokers scheduled for removal and block the scale down
2. Include `cordoned.log.dirs=*` in the desired per-broker configuration for those brokers, so that the existing configuration reconciliation applies it automatically
3. Proceed with the existing auto-rebalance workflow (create `KafkaRebalance` resource, etc.)
4. Remove brokers after successful rebalancing to complete the scale down
5. Handle cancelled scale down scenarios automatically: when a broker is no longer scheduled for removal, its desired configuration omits `cordoned.log.dirs`, and the existing configuration diff uncordons it

Even if cordoning brokers needs some time to propagate across the cluster, no explicit wait is considered to avoid over complexity.
All the brokers to be removed will be cordoned eventually.
The scale down will not complete until Cruise Control has moved all partitions off the brokers being removed, so any new partitions that sneak onto a broker during the brief propagation window will be moved as well.
This time window can be considered negligible compared to the duration of a cluster rebalancing.

The `cordoned.log.dirs` broker configuration property will be considered as **forbidden**.
The users can't set it within the `Kafka` custom resource `spec.kafka.config` to prevent conflicts with operator-managed cordoning.
Since broker configuration in Strimzi is common across all Kafka node pools, users cannot target specific brokers or log directories through the `Kafka` custom resource.
The operator needs exclusive control over this property to manage the scale-down lifecycle correctly.

Despite the cordoning feature is available from Apache Kafka 4.3, there is no need for explicit Kafka version gating.
The existing `KafkaConfigurationDiff` relies on the Kafka config model, which is selected based on the Kafka binary version.
On Kafka < 4.3, `cordoned.log.dirs` is not in the config model, so the diff treats it as a "custom config" and silently skips it so the property appears in the desired configuration but is never applied to the broker.
On Kafka >= 4.3, `cordoned.log.dirs` is in the config model, so the diff generates the appropriate `SET`/`DELETE` operations.

It is worth noting that the Kafka binary version and the metadata version can differ during phased upgrades.
KIP-1066 requires metadata version `IBP_4_3_IV0` for the controller to enforce cordoning during partition placement.
If a cluster runs Kafka 4.3 but the metadata version has not been upgraded yet, the `cordoned.log.dirs` configuration will be accepted by the broker (since the binary supports it) but the controller will not enforce it.
Cordoning is treated as best-effort in this scenario: the scale down still works correctly because Cruise Control moves all partitions off the brokers before they are removed, regardless of whether cordoning is active.
Once the metadata version is upgraded, cordoning becomes effective automatically.

### Detailed workflow

The cordoning logic is integrated into the existing per-broker configuration pipeline.
The existing `KafkaRoller` already reads broker configuration via `describeConfigs()` and applies changes via Kafka Admin API `incrementalAlterConfigs()`.

When the operator detects a scale down, the existing `KafkaClusterCreator.brokerRemovalCheck()` saves the brokers scheduled for removal into `scalingDownBlockedNodes` before reverting the scale down.
After the revert, the `KafkaCluster` model has `removedNodes()` empty (the scale down is reverted so that brokers are not removed yet), but the `scalingDownBlockedNodes` set preserves which brokers were originally targeted.
This set already flows to the `KafkaAutoRebalancingReconciler` to trigger the `KafkaRebalance` resource creation with the removed brokers.
With this proposal, it also flows to the `KafkaReconciler` to drive cordoning through the configuration pipeline.

The per-broker configuration is generated by the `KafkaBrokerConfigurationBuilder` class.
A new `withCordonedLogDirs()` builder method writes `cordoned.log.dirs=*` for brokers in `scalingDownBlockedNodes`, and nothing for others.
The existing configuration reconciliation then handles everything:

- **Cordoning**: when `cordoned.log.dirs=*` is in the desired configuration but not on the broker, `KafkaConfigurationDiff` generates a `SET` operation applied via `incrementalAlterConfigs()` and no broker restart is needed since this is a dynamic per-broker config
- **Uncordoning** (cancelled scale down): when `cordoned.log.dirs` is on the broker but not in the desired configuration, `KafkaConfigurationDiff` generates a `DELETE` operation to allow automatic cleanup
- **No-op** (already cordoned): when the desired and live configurations match, no operation is generated

The reconciliation ordering guarantees that cordoning is applied (during `KafkaReconciler.rollingUpdate()`) before auto-rebalancing starts (the `KafkaAutoRebalancingReconciler` runs after the full `KafkaReconciler` pipeline completes).

```
KafkaClusterCreator.prepareKafkaCluster()
  ├─ Detects scale down, brokerRemovalCheck() finds brokers still in use
  ├─ Saves scalingDownBlockedNodes (before revert)
  └─ Reverts scale down → KafkaCluster model with removedNodes() = []

KafkaAssemblyOperator
  ├─ Passes scalingDownBlockedNodes to KafkaReconciler (new)
  └─ Passes scalingDownBlockedNodes to KafkaAutoRebalancingReconciler (existing)

KafkaReconciler.reconcile()
  ├─ brokerConfigurationConfigMaps(): generates per-broker ConfigMaps
  │   └─ For brokers in scalingDownBlockedNodes, the desired config includes cordoned.log.dirs=*
  └─ rollingUpdate(): KafkaRoller diffs current vs desired config
      └─ Applies cordoned.log.dirs=* via incrementalAlterConfigs()

KafkaAutoRebalancingReconciler.reconcile()   (runs after KafkaReconciler completes)
  └─ Sees scalingDownBlockedNodes, creates KafkaRebalance with remove-brokers mode
```

#### Detailed step-by-step workflow

**Step 1: Scale down detected and blocked**

When `KafkaClusterCreator.prepareKafkaCluster()` is called and a scale down is requested, the existing `brokerRemovalCheck()` determines that the brokers scheduled for removal still have partition-replicas assigned.
Before reverting the scale down, it saves the set of broker IDs into `scalingDownBlockedNodes`.
The scale down is then reverted and the `KafkaCluster` model is rebuilt with the original replica count, so `removedNodes()` returns an empty set.
The `scalingDownBlockedNodes` set is the only place that remembers which brokers were originally targeted for removal.

**Step 2: `scalingDownBlockedNodes` flows to `KafkaReconciler`**

The `KafkaAssemblyOperator` passes `scalingDownBlockedNodes` to the `KafkaReconciler` constructor (in addition to the existing flow to `KafkaAutoRebalancingReconciler`).
The `KafkaReconciler` stores it and uses it during per-broker configuration generation.

**Step 3: Per-broker configuration includes cordoning**

The `KafkaReconciler` reconciliation pipeline generates per-broker configuration at two points, both of which need to include `cordoned.log.dirs=*` for brokers in `scalingDownBlockedNodes`:

1. During the `brokerConfigurationConfigMaps()` step, per-broker ConfigMaps are generated and stored in Kubernetes. For each broker, `KafkaBrokerConfigurationBuilder` is called with a cordoning flag based on whether the broker's node ID is in `scalingDownBlockedNodes`. If the flag is `true`, `cordoned.log.dirs=*` is included in the desired configuration for that broker via `withCordonedLogDirs()` builder method.

2. During the later `rollingUpdate()` step, the `KafkaRoller` gets the desired configuration for each broker to diff it against the live configuration. This configuration must also include `cordoned.log.dirs=*` for the same brokers, to be consistent with the ConfigMaps generated in the previous step.

**Step 4: `KafkaRoller` applies or removes the cordoning configuration**

During the `KafkaReconciler.rollingUpdate()` step of the reconciliation, the `KafkaRoller` processes each broker:

- It reads the current broker configuration via `describeConfigs()`
- It gets the desired configuration generated via the `KafkaCluster.generatePerBrokerConfiguration()` method and passed to the constructor
- By using a `KafkaConfigurationDiff` instance, it compares the two and determines the operation needed:
  - **Cordoning** (broker in `scalingDownBlockedNodes`): `cordoned.log.dirs=*` is in the desired configuration but not on the broker, so the diff generates a `SET` operation
  - **Uncordoning** (cancelled scale down): `cordoned.log.dirs` is on the broker but not in the desired configuration (because `scalingDownBlockedNodes` is now empty), so the diff generates a `DELETE` operation
  - **No-op** (already in correct state): the current and desired configurations match, so no operation is generated
- The `KafkaRoller` applies the operation via `incrementalAlterConfigs()` as a dynamic per-broker config change, without restarting the broker

**Step 5: Auto-rebalancing starts**

After `KafkaReconciler.reconcile()` completes, `KafkaAutoRebalancingReconciler` runs.
It sees `scalingDownBlockedNodes` is not empty and creates a `KafkaRebalance` resource with `spec.mode: remove-brokers`.
At this point, the brokers are already cordoned (or cordoning is propagating), so new partitions will not be assigned to them.
Cruise Control starts moving existing partitions off the brokers being removed.

**Step 6: Subsequent reconciliations during rebalancing**

On each subsequent reconciliation while rebalancing is in progress:
- `brokerRemovalCheck()` still finds the brokers in use (partitions being moved), so the scale down remains reverted and `scalingDownBlockedNodes` is repopulated
- The desired configuration for the cordoned brokers still includes `cordoned.log.dirs=*`, so the `KafkaRoller` sees no diff and takes no action
- The `KafkaAutoRebalancingReconciler` monitors the `KafkaRebalance` resource status and waits for rebalancing to complete

**Step 7: Scale down completes**

Once Cruise Control has moved all partitions off the brokers being removed, the next reconciliation finds:
- `brokerRemovalCheck()` sees the brokers are empty, so the scale down is not reverted this time and `scalingDownBlockedNodes` is empty
- The `KafkaCluster` model has `removedNodes()` containing the brokers to remove
- `scaleDown()` removes the pods for those brokers
- No ConfigMaps are generated for the removed brokers, and the `KafkaRoller` does not process them (pods are gone)
- The cordoning configuration is gone with the brokers, so no cleanup needed

#### Reconciliation scenarios

Following some examples showing how all the above steps fit within a single reconciliation loop but also together across multiple reconciliations.

**Reconciliation N (first detection of scale down):**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. `brokerRemovalCheck()` finds brokers [3, 4] still have partitions → `scaleDownCheckFailed = true`
3. `scalingDownBlockedNodes = [3, 4]` saved before revert
4. Scale down reverted (pods stay at 5), `KafkaCluster` model rebuilt with `removedNodes() = []`
5. `KafkaReconciler` runs with `scalingDownBlockedNodes = [3, 4]`
6. `brokerConfigurationConfigMaps()`: ConfigMaps for brokers [3, 4] include `cordoned.log.dirs=*`
7. `rollingUpdate()`: `KafkaRoller` diffs current vs desired for brokers [3, 4], applies `SET cordoned.log.dirs=*` via `incrementalAlterConfigs()`
8. `KafkaAutoRebalancingReconciler` sees `scalingDownBlockedNodes = [3, 4]`, creates `KafkaRebalance` with `mode: remove-brokers`

**Reconciliation N+1 (rebalancing in progress):**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. `brokerRemovalCheck()` finds brokers [3, 4] still have partitions (being moved) → scale down reverted again
3. `scalingDownBlockedNodes = [3, 4]`
4. `KafkaReconciler` runs: desired config for [3, 4] still includes `cordoned.log.dirs=*`
5. `rollingUpdate()`: `KafkaRoller` diffs current vs desired for brokers [3, 4] → no change (already cordoned), no action
6. `KafkaAutoRebalancingReconciler` monitors `KafkaRebalance` status, rebalancing continues

**Reconciliation N+K (rebalancing complete, scale down proceeds):**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. `brokerRemovalCheck()` finds brokers [3, 4] are empty → `scaleDownCheckFailed = false`
3. `scalingDownBlockedNodes` is empty (no blocked nodes)
4. Scale down NOT reverted: `KafkaCluster` model has `removedNodes() = [3, 4]`
5. `scaleDown()` removes pods for brokers [3, 4]
6. No ConfigMaps generated for removed brokers, `KafkaRoller` does not process them
7. Cordoning configuration gone with the brokers, so no cleanup needed

#### Rollback scenario (user cancels scale down)

The user may initiate a scale down operation, causing the operator to cordon brokers.
If the user reverts the `KafkaNodePool` replicas back to the original count before the scale down completes, the previously cordoned brokers must be uncordoned since they are no longer scheduled for removal.
The configuration pipeline handles this cleanup automatically.
When `scalingDownBlockedNodes` is empty, the desired configuration for those brokers no longer includes `cordoned.log.dirs`, and the existing `KafkaConfigurationDiff` generates a `DELETE` operation to uncordon them.
Without this cleanup, the brokers would remain cordoned indefinitely and the controller would not assign any new partitions to them.

Following a sequence of reconciliations describing such scenario.

**Reconciliation N:**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. `brokerRemovalCheck()` finds brokers [3, 4] still have partitions → `scaleDownCheckFailed = true`
3. `scalingDownBlockedNodes = [3, 4]` saved before revert, scale down reverted
4. `KafkaReconciler` runs: `rollingUpdate()` applies `SET cordoned.log.dirs=*` on brokers [3, 4]
5. `KafkaAutoRebalancingReconciler` creates `KafkaRebalance` with `mode: remove-brokers`

**User reverts replicas back to 5 before next reconciliation**

**Reconciliation N+1:**
1. No scale down detected (replicas are back to 5)
2. `brokerRemovalCheck()` finds no removed nodes → `scaleDownCheckFailed = false`
3. `scalingDownBlockedNodes` is empty
4. `KafkaReconciler` runs: desired config for brokers [3, 4] does NOT include `cordoned.log.dirs`
5. `rollingUpdate()`: `KafkaRoller` diffs current vs desired for brokers [3, 4], generates `DELETE` remove `cordoned.log.dirs` → brokers [3, 4] uncordoned
6. Normal reconciliation proceeds

## Affected/not affected projects

This proposal affects only the `strimzi-kafka-operator` by adding the cordoning logic into the existing per-broker configuration pipeline within `KafkaBrokerConfigurationBuilder`, `KafkaCluster`, `KafkaReconciler`, and `KafkaAssemblyOperator` classes.

## Compatibility

The KIP-1066 is available in Kafka 4.3.0+ and requires metadata version `IBP_4_3_IV0` or higher for the controller to enforce cordoning during partition placement.
It is important to distinguish between the Kafka binary version and the metadata version, as they can differ during phased upgrades.

This proposal does not require explicit version checks or runtime metadata version detection.
Instead, it relies on the existing `KafkaConfigurationDiff` behavior with the Kafka config model.
The config model is selected based on the Kafka binary version, not the metadata version.
This leads to three possible scenarios:

**Kafka binary < 4.3.0:**
`cordoned.log.dirs` is not in the config model, so `KafkaConfigurationDiff` treats it as a "custom config" and silently skips it.
The property appears in the desired configuration but is never applied to the broker.
Scale down works as before without cordoning.

**Kafka binary >= 4.3.0 with metadata version >= `IBP_4_3_IV0`:**
`cordoned.log.dirs` is in the config model with `PER_BROKER` scope, so the diff generates `SET`/`DELETE` operations and the `KafkaRoller` applies them via `incrementalAlterConfigs()`.
The controller enforces cordoning during partition placement.
This is the fully effective scenario.

**Kafka binary >= 4.3.0 with metadata version < `IBP_4_3_IV0` (phased upgrade):**
`cordoned.log.dirs` is in the config model (based on the binary version), so the diff generates a `SET` operation and the broker accepts the configuration.
However, the controller running at the older metadata version does not enforce cordoning during partition placement and the configuration is set but has no effect.
This is a best-effort scenario: cordoning is a no-op, but the scale down still works correctly because Cruise Control moves all partitions off the brokers before they are removed, regardless of whether cordoning is active.
Once the metadata version is upgraded to `IBP_4_3_IV0` or higher, cordoning becomes effective automatically on subsequent scale down operations.

Upgrading the operator doesn't need any migration steps:
- Upgrading the operator with existing Kafka < 4.3.0 clusters: No behavior change (cordoning is silently skipped by the config model)
- Upgrading the operator with Kafka >= 4.3.0 clusters: Cordoning automatically enabled for subsequent scale down operations (effective only if metadata version also supports it)

### Impact on existing auto-rebalancing users

**Users with auto-rebalancing enabled (Kafka < 4.3.0):**
- No changes to behavior
- Scale down continues to work as before
- No action required

**Users with auto-rebalancing enabled (Kafka >= 4.3.0):**
- Automatic cordoning is enabled transparently
- No configuration changes required
- Cordoning is fully effective when metadata version >= `IBP_4_3_IV0`, best-effort otherwise

**Users without auto-rebalancing:**
- No impact (feature is opt-in via `spec.cruiseControl.autoRebalance`)

## Rejected alternatives

### Alternative 1: Make cordoning opt-in via feature gate

**Description:**  
Gate the cordoning feature behind a Strimzi feature gate, requiring users to explicitly enable it even when the Kafka version supports it.

**Rejection reasons:**
- Adds unnecessary user-facing configuration burden for a feature that has benefits
- No clear use case for disabling cordoning when Kafka supports it
- Automatic enablement based on Kafka version is simpler and provides better default behavior
- Feature gates are typically used for experimental or potentially disruptive features
- Users who want to avoid the feature can use Kafka < 4.3.0 or disable auto-rebalancing entirely

### Alternative 2: Cruise Control-managed cordoning

**Description:**  
Have Cruise Control automatically configure the `cordoned.log.dirs` parameter on brokers when it starts a rebalancing operation due to a call on the `/remove_broker` REST API endpoint (`KafkaRebalance` resource with `mode: remove-brokers`).

**Rejection reasons:**
- Operator should be the single source of truth for broker configuration
- Requires modifications to Cruise Control (external project)
- Operator loses visibility into when cordoning takes effect
- Unclear ownership for uncordoning when scale-down is cancelled
- Incompatible with required flow (cordoning must happen before `KafkaRebalance` resource exists)

### Alternative 3: Split cordoning and uncordoning into different reconciliation phases

**Description:**  
Handle cordoning at the beginning of reconciliation (in `KafkaClusterCreator.prepareKafkaCluster()`) only when scale down is detected, but move uncordoning cleanup to the end of the Kafka reconciliation (similar to broker unregistration pattern).

**Rejection reasons:**
- Same complexity: Still requires Admin API calls (`describeFeatures()`, `describeLogDirs()`) and state checking in both locations
- Worse failure handling: If reconciliation fails mid-way after cordoning but before reaching the uncordoning phase, brokers remain incorrectly cordoned until the next successful reconciliation
- Still needs multiple reconciliations: Uncordoning requires propagation time, so verification still happens on the next reconciliation regardless of where uncordoning runs
- Separates related logic: Splits cordoning and uncordoning into different code locations, reducing maintainability
- Different semantics from broker unregistration: Broker unregistration cleans up metadata for deleted brokers (fire-and-forget), while uncordoning fixes state of running brokers (requires verification)
- Must check `removedNodes()` anyway: End-of-reconciliation uncordoning still needs to check if scale down is in progress to avoid uncordoning legitimately cordoned brokers
- No clear benefits: Doesn't reduce complexity, improve reliability, or better match existing patterns

### Alternative 4: Standalone cordoning check in `KafkaClusterCreator`

**Description:**
Implement cordoning as a separate `brokerCordoningCheck()` method in `KafkaClusterCreator`, following the same pattern as the existing `brokerRemovalCheck()`.
This method would use `describeFeatures()` to check the runtime metadata version, `describeLogDirs()` to query each broker's cordoning state, and direct `incrementalAlterConfigs()` calls to cordon or uncordon brokers.
It would track state via a `cordoningCheckFailed` flag and wait for cordoning propagation across reconciliations before allowing the auto-rebalancing to proceed.

**Rejection reasons:**
- The regular configuration reconciliation in `KafkaRoller` would overwrite the cordoning set by the standalone check: `KafkaRoller` calls `describeConfigs()` and diffs against the desired configuration which does not include `cordoned.log.dirs`, generating a `DELETE` operation that unsets the cordoning. The operator would fight itself.
- Requires additional Admin API calls (`describeFeatures()`, `describeLogDirs()`) that the integrated approach avoids entirely
- Adds unnecessary reconciliation cycles by waiting for cordoning propagation, while the scale down already cannot complete until Cruise Control has moved all partitions
- Introduces new state tracking (`cordoningCheckFailed`) and explicit uncordoning logic that the configuration pipeline handles automatically