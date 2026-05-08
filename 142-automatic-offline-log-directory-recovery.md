# Automatic Detection and Recovery of Offline Log Directories

Automatically detect brokers with offline log directories during reconciliation and trigger a rolling restart to recover them, with ISR-aware safety checks, a weaker availability check for self-caused URPs, and a configurable cooldown to prevent infinite restart loops on permanent hardware failures.

## Current situation

When a Kafka broker's log directory goes offline (due to disk failure, filesystem corruption, or permission issues), the broker marks the directory as offline and stops serving partitions on it. The affected partitions become under-replicated or unavailable.

Currently, Strimzi has no mechanism to detect or recover from this state. The operator continues reconciling normally, unaware that a broker is operating in a degraded state. Recovery requires manual intervention — typically an operator noticing the issue through monitoring/alerts and manually triggering a broker restart (either by annotating the pod with `strimzi.io/manual-rolling-update: true` or by deleting the pod).

The `KafkaRoller` already has infrastructure for detecting various broker issues and triggering rolling restarts (certificate changes, configuration drift, stuck pods, unresponsive pods), but offline log directories are not among the conditions it checks.

## Motivation

Offline log directories are a common operational issue, particularly in clusters using JBOD storage with multiple volumes per broker. Single-volume brokers are also affected — when the only log directory goes offline, the entire broker becomes non-functional. Common causes include:

- Transient disk I/O errors
- Filesystem corruption after an unclean shutdown
- Storage subsystem issues (SAN timeouts, NFS mount failures, cloud disk detach/reattach)
- Permission changes after node maintenance

In most cases, a broker restart recovers the offline directory — the broker reinitializes the log directory on startup and begins replicating data from other brokers. The current manual recovery process has several problems:

1. **Delayed detection**: Operators may not notice the issue until users report missing data or consumer lag increases
2. **Mean time to repair**: Even after detection, coordinating a safe manual restart takes time
3. **Operational toil**: Repetitive manual work that should be automated
4. **After-hours incidents**: Offline dirs during off-hours require on-call intervention for a routine fix

Automating this recovery aligns with the Strimzi operator's core purpose — managing the Kafka cluster lifecycle so users don't have to.

## Proposal

### Overview

The proposal adds four interconnected capabilities to the Strimzi cluster operator:

1. **Detection**: Query each broker's log directory status via `describeLogDirs()` during reconciliation
2. **Safe restart**: Trigger ISR-aware rolling restarts for brokers with offline directories
3. **Weaker availability check**: Exclude partitions on offline directories from the ISR safety gate to break the catch-22 where offline dirs cause URPs that block recovery
4. **Cooldown**: Prevent infinite restart loops when offline dirs are caused by permanent hardware failure

### Detection mechanism

During each reconciliation cycle, after the existing `checkIfRestartOrReconfigureRequired()` checks, a new `checkForOfflineLogDirs()` method queries each broker's log directory status via the Kafka Admin API's `describeLogDirs()` method. This API returns a `LogDirDescription` for each configured log directory on the broker. A directory is considered offline when `LogDirDescription.error()` returns a non-null value (typically a `KafkaStorageException`).

The detection uses a single `describeLogDirs()` call per broker per reconciliation, returning a `LogDirStatus` record that captures both:
- Whether any directories are offline (`hasOfflineDirs`)
- The set of topic-partitions on healthy directories (`partitionsOnHealthyDirs`)

This cached result is reused by the weaker availability check, avoiding redundant Admin API round trips.

When offline directories are detected, the broker is added to the rolling restart queue with a new `OFFLINE_LOG_DIRS` restart reason.

The detection is best-effort: all exceptions from the Admin API are caught (`catch (Exception e)`) so that a `describeLogDirs` failure never blocks normal rolling restart flow. `InterruptedException` is correctly re-thrown to preserve the interrupt propagation chain.

The `describeLogDirs()` API has been available since Kafka 1.0 (KIP-113), covering all Kafka versions currently supported by Strimzi.

### Restart safety

The offline-dir restart integrates into the existing `KafkaRoller` safety mechanisms:

**ISR-aware rolling**: The restart uses `needsRestart` (not `forceRestart`), so the existing `KafkaAvailability` check gates every restart. No broker is restarted if doing so would drop any topic partition below `min.insync.replicas`.

**One-at-a-time with readiness wait**: The `KafkaRoller` restarts brokers sequentially and waits for each to become ready before proceeding, allowing ISR to recover between restarts.

**Controller-last ordering**: The active KRaft controller is always restarted last, avoiding unnecessary controller elections.

**Restart precedence over dynamic reconfiguration**: When both `needsRestart` (from offline dirs) and `needsReconfig` (from a pending config diff) are set simultaneously, the broker is fully restarted rather than dynamically reconfigured. Offline log directories require a restart to recover — a dynamic config update cannot fix them. The pending config change is applied on the next reconciliation cycle after the restart.

### Weaker availability check for offline-dir URPs

**The problem**: When a broker has an offline log directory, partitions on that directory are already offline or under-replicated. The standard `KafkaAvailability.canRoll()` check sees these URPs and blocks the restart — even though the restart is the fix. This creates a catch-22.

**The solution**: A weaker availability check that excludes partitions on offline directories from the ISR evaluation. This check applies only when:

- `OFFLINE_LOG_DIRS` is the **sole** restart reason (no config changes, cert rotations, etc. combined with it)
- The node is a **broker-only** node (`isBroker && !isController`)

The restriction to broker-only nodes is critical: the standard `canRoll()` for combined controller+broker nodes verifies both broker partition availability AND KRaft quorum safety. The weaker check only evaluates broker partition availability, so allowing it on combined nodes would bypass the quorum safety check. This could allow a restart that breaks the controller quorum.

**How it works**:

1. The `describeLogDirs()` response identifies which topic-partitions reside on healthy (non-offline) directories
2. `KafkaAvailability.canRollExcludingOfflineDirPartitions()` runs the same ISR availability check but skips partitions not in the healthy set
3. If all healthy-dir partitions can tolerate the restart, the broker is rolled

This ensures:
- Partitions on healthy directories are still fully protected by the ISR check
- Partitions on offline directories (already degraded) do not block their own recovery
- KRaft quorum safety is never bypassed

### Cooldown to prevent infinite restart loops

If offline directories are caused by permanent hardware failure (dead disk, unrecoverable filesystem corruption), the broker would be restarted every reconciliation cycle (default 120 seconds) indefinitely, creating a restart storm that degrades cluster availability.

To prevent this, the proposal includes a configurable cooldown period. After a broker is restarted, the operator checks the pod's `metadata.creationTimestamp` — if the pod was created within the cooldown window, the offline-dir check is skipped entirely with a debug log message.

**Configuration**:

| Layer | Config | Default |
|---|---|---|
| Helm | `offlineLogDirRestartCooldownMs` | `1800000` (30 minutes) |
| Env var | `STRIMZI_OFFLINE_LOG_DIR_RESTART_COOLDOWN_MS` | `1800000` |
| Java | `ClusterOperatorConfig.OFFLINE_LOG_DIR_RESTART_COOLDOWN_MS` | `1800000` |

Setting the cooldown to `0` disables it entirely, allowing restart on every reconciliation cycle. Operators with reliable storage who want the fastest recovery can use this setting.

The 30-minute default was chosen to balance recovery speed against restart storm risk. For most transient failures, the first restart succeeds and the directory is recovered. For permanent failures, 30 minutes provides ample time for operators or monitoring systems to notice and take corrective action.

### Scope boundaries

The offline-dir detection and restart logic is scoped to `KafkaReconciler.rollingUpdate()` — the main reconciliation path for Kafka broker pods. It is **not** involved in:

- CA reconciliation (`CaReconciler`) — certificate-triggered rolling restarts do not need offline-dir detection
- Connect, MirrorMaker2, or Bridge reconciliation
- Cruise Control rebalancing — see "Relationship to Cruise Control" below

### Relationship to Cruise Control

Cruise Control can detect offline disks and rebalance partitions away from affected brokers (using `fixOfflineReplicas` goals). This is a complementary approach:

| Approach | Best for | Speed | Data movement |
|---|---|---|---|
| **This proposal** (restart-based) | Transient disk failures where restart recovers the directory | Fast (seconds) | None |
| **Cruise Control** (rebalance-based) | Permanent failures where the disk will not recover | Slow (minutes to hours) | Yes |

The two approaches are not mutually exclusive. The cooldown mechanism naturally defers to manual or CC-based intervention for permanent failures: after one failed restart attempt, the broker is left alone for the cooldown period, during which an operator or Cruise Control can take corrective action (replace the disk, reassign partitions).

If there are existing plans or roadmap items for CC-based disk failure handling in Strimzi, this proposal is designed to coordinate rather than conflict. The `OFFLINE_LOG_DIRS` restart reason is distinct from any CC-triggered actions, and the cooldown prevents this feature from interfering with CC rebalancing operations.

Future integration could coordinate the two approaches: if a broker has been restarted for offline dirs and the dirs are still offline after the cooldown, the operator could trigger a CC rebalance instead of another restart. This coordination is explicitly out of scope for this proposal and would be addressed separately if needed.

### Observability

#### Kubernetes Events

A Kubernetes Event is published for each broker restarted due to offline log directories:

| Field | Value |
|---|---|
| `action` | `StrimziInitiatedPodRestart` |
| `reason` | `OfflineLogDirs` |
| `note` | `Rolling Pod <name> due to Broker has offline log directories` |
| `regarding` | The `Kafka` custom resource |
| `related` | The restarted `Pod` |

This follows the existing pattern used by all other restart reasons (e.g., `CaCertHasOldGeneration`, `ConfigChangeRequiresRestart`).

#### Logging

The feature produces log messages at the following levels:

| Level | Message | When |
|---|---|---|
| WARN | `Pod {name} has {N} offline log director(y/ies): [paths]` | Offline dirs detected on a broker |
| DEBUG | `Skipping offline log directory check for pod {name} (created recently, within cooldown period)` | Cooldown prevents check |
| DEBUG | `Pod {name} has {N} partitions on healthy log dirs; checking availability excluding offline-dir partitions` | Weaker availability check attempted |
| INFO | `Rolling Pod {name} for offline log dir recovery (URPs on offline dirs excluded from availability check)` | Broker restarted via weaker check |
| INFO | `Rolling Pod {name} due to [Broker has offline log directories]` | Broker restarted via standard check |
| WARN | `Failed to check offline log directories for pod {name}: {error}` | `describeLogDirs` API call failed (best-effort, non-blocking) |

#### Operator startup configuration logging

The cooldown value is included in `ClusterOperatorConfig.toString()`, which is logged at operator startup:

```
offlineLogDirRestartCooldownMs=1800000
```

#### Future observability enhancements (out of scope)

The following could be added in follow-up work:

- **Metrics**: A gauge metric for the number of brokers with offline log directories, exposed via the cluster operator's metrics endpoint
- **Kafka CR status condition**: A condition on the `Kafka` custom resource indicating that one or more brokers have offline log directories (similar to how other degraded states are reported)

## Affected/not affected projects

### Affected

- **strimzi-kafka-operator** (cluster-operator module):
  - `KafkaRoller` — offline-dir detection (`describeLogDirStatus`, `checkForOfflineLogDirs`), cooldown (`isPodWithinOfflineLogDirCooldown`), weaker availability check (`canRollExcludingOfflineDirPartitions`), `LogDirStatus` record, timeout constants (`ADMIN_API_TIMEOUT_MS`, `AVAILABILITY_CHECK_TIMEOUT_MS`), restart precedence over dynamic reconfiguration
  - `KafkaAvailability` — `canRollExcludingOfflineDirPartitions()` method with partition exclusion in `wouldAffectAvailability()`
  - `KafkaReconciler` — wire cooldown configuration to `KafkaRoller`
  - `RestartReason` — new `OFFLINE_LOG_DIRS` enum value
  - `ClusterOperatorConfig` — new `OFFLINE_LOG_DIR_RESTART_COOLDOWN_MS` config parameter with getter and `toString()` inclusion
  - Helm charts — new `offlineLogDirRestartCooldownMs` value in `values.yaml` and deployment template
  - `KubernetesRestartEventPublisher` — no changes needed; existing infrastructure publishes events for the new restart reason automatically

### Not affected

- **strimzi-kafka-operator** (other modules): topic-operator, user-operator, drain-cleaner
- **strimzi-kafka-operator** (`CaReconciler`): CA-triggered rolling restarts do not use offline-dir detection
- **strimzi-kafka-bridge**
- **strimzi-kafka-oauth**
- **strimzi CRD/API**: No changes to the `Kafka`, `KafkaNodePool`, or any other custom resource definitions

## Compatibility

### Backward compatibility

- **No breaking changes**: The feature is purely additive. Existing clusters upgrading to a version with this feature will automatically benefit from offline-dir detection without any configuration changes.
- **Default behavior is safe**: The default 30-minute cooldown prevents aggressive restart behavior. The ISR-aware availability check ensures no partition drops below `min.insync.replicas`. The weaker check is restricted to broker-only nodes with `OFFLINE_LOG_DIRS` as the sole reason.
- **Opt-out**: Operators who do not want automatic offline-dir recovery can set the cooldown to a very large value (effectively disabling the feature) or use a feature gate if one is added in the future.
- **No CRD changes**: The feature is entirely operator-internal and configured via environment variables / Helm values, not through the Kafka CR. This means no CRD migration is needed.

### Forward compatibility

- **Cruise Control integration**: The design does not preclude future CC integration. The `OFFLINE_LOG_DIRS` restart reason is distinct from CC-triggered restarts, so the two can coexist. The cooldown period creates a natural handoff point where CC can take over for permanent failures.
- **KRaft quorum safety**: The weaker availability check is gated to broker-only nodes (`isBroker && !isController`). Combined controller+broker nodes always use the standard availability check, preserving quorum safety as Strimzi evolves its KRaft support.
- **New restart reasons**: The `RestartReason` enum is additive. Adding `OFFLINE_LOG_DIRS` does not affect existing reasons or their behavior.
- **Feature gate migration**: If the Strimzi community later decides this should be behind a feature gate, the configuration is already isolated in `ClusterOperatorConfig` and can be wrapped in a gate without architectural changes.

## Rejected alternatives

### 1. Annotation-based cooldown tracking

Instead of using `pod.metadata.creationTimestamp` for cooldown, track the last offline-dir restart time in a pod annotation (e.g., `strimzi.io/offline-logdir-last-restart`).

**Rejected because**: Pod annotations are lost when the pod is deleted and recreated (which is exactly what a restart does in StrimziPodSet-managed pods). Tracking the annotation would require writing to the StrimziPodSet template, adding complexity. The creation timestamp approach is simpler and achieves the same practical result — a recently restarted pod is always recently created.

### 2. Cruise Control-only approach

Defer entirely to Cruise Control for offline-dir recovery, using CC's `fixOfflineReplicas` goal to rebalance partitions away from affected brokers.

**Rejected because**: Not all Strimzi deployments use Cruise Control. CC-based rebalancing is slower (requires data movement across the network) and more complex to configure. For the common case of transient disk failures, a simple restart is faster, cheaper, and more appropriate. The two approaches are complementary — this proposal handles transient failures quickly, while CC handles permanent failures that require data redistribution.

### 3. Always use the weaker availability check for offline-dir restarts on all node types

Apply the weaker availability check (excluding offline-dir partitions) to all node types, including combined controller+broker nodes.

**Rejected because**: The weaker check only evaluates broker partition availability — it does not verify KRaft quorum safety. For combined controller+broker nodes, the standard `canRoll()` checks both broker availability AND controller quorum. Bypassing the quorum check could allow a restart that breaks the controller quorum. Restricting the weaker check to broker-only nodes preserves quorum safety while still providing automated recovery for the most common deployment topologies (separate controller and broker pools).

### 4. Force-restart (bypass all availability checks)

Use `forceRestart` instead of `needsRestart` to skip the availability check entirely, on the assumption that the broker is already degraded and restarting it quickly is more important than availability preservation.

**Rejected because**: Even a degraded broker with one offline directory may have multiple healthy directories serving many partitions. Force-restarting could disrupt those healthy partitions unnecessarily. The `needsRestart` path with the optional weaker check provides targeted recovery while preserving availability for healthy workloads.

### 5. Feature gate for opt-in

Add a Strimzi feature gate (e.g., `+OfflineLogDirRecovery`) to control whether the feature is active, defaulting to disabled.

**Rejected for initial proposal because**: The configurable cooldown already provides sufficient operational control. Setting cooldown to a very large value effectively disables the feature. Adding a separate feature gate increases configuration surface area without proportional benefit. However, if the Strimzi community prefers explicit opt-in for new operator behaviors, a feature gate can be added without architectural changes — the detection logic is already cleanly isolated behind `checkForOfflineLogDirs()`.
