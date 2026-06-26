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

Extend the auto-rebalancing mechanism to automatically cordon brokers during the scale down process.
The operator will:

1. Detect brokers scheduled for removal and block the scale down
2. Cordon **all log directories** on those brokers by setting `cordoned.log.dirs=*` via the Kafka Admin API (using wildcard `*` to cordon all directories)
3. Wait for the cordoning to take effect (across reconciliations)
4. Proceed with the existing auto-rebalance workflow (create `KafkaRebalance` resource, etc.)
5. Remove brokers after successful rebalancing to complete the scale down
6. Handle cancelled scale down scenarios (uncordoning all log directories)

The `cordoned.log.dirs` broker configuration property will be considered as **forbidden**.
The users can't set it within the `Kafka` custom resource `spec.kafka.config` to prevent conflicts with operator-managed cordoning.
Since broker configuration in Strimzi is common across all Kafka node pools, users cannot target specific brokers or log directories through the `Kafka` custom resource.
The operator needs exclusive control over this property to manage the scale-down lifecycle correctly (cordoning before rebalancing, uncordoning on cancelled scale down).

### Detailed workflow

The cordoning logic is implemented in a new `KafkaClusterCreator.brokerCordoningCheck()` method, following the same pattern as the existing `brokerRemovalCheck()` method.
The method sets a `cordoningCheckFailed` instance variable that indicates whether cordoning/uncordoning operations were performed and propagation is needed, similar to how `brokerRemovalCheck()` sets `scaleDownCheckFailed`.
It also takes into account the fact that cordoning (or uncordoning) brokers needs some time for the configuration change to propagate across the cluster.
This means that the auto-rebalancing, together with the actual scale-down, could be delayed across reconciliations, until the cordoning of brokers being removed is confirmed.
The cordoning check runs on every reconciliation to ensure brokers are in the correct cordoning state, for scaling down or cleaning up (in case of cancelled scale down):

- **Always check metadata version first**: Determines if cordoning is supported (Kafka 4.3+ with IBP_4_3_IV0+)
- **Query all broker cordoning states**: Uses `describeLogDirs()` to check which brokers are currently cordoned
- **Analyze and reconcile**: Compares current state against desired state (brokers being removed should be cordoned, others should not)
- **Apply operations if needed**: Cordons brokers being removed, uncordons brokers no longer being removed (cleanup)
- **Wait for cordoning propagation**: If operations were performed, revert scale down, and retry on next reconciliation

Following a more detailed flow of what happens within the `KafkaClusterCreator` class as cordoning check.

```
├─ Check metadata version via describeFeatures()
│   ├─ Fails → Log warning, skip cordoning, proceed with normal reconciliation
│   └─ Succeeds → Parse metadata.version
│       ├─ < IBP_4_3_IV0 → Skip cordoning, proceed with normal reconciliation
│       └─ >= IBP_4_3_IV0 → Cordoning is supported, continue
│
└─ Query current cordoning state for all brokers
    ├─ describeLogDirs() fails → Log warning, revert scale down (if any), retry next reconciliation
    └─ describeLogDirs() succeeds → Analyze broker states
        │
        ├─ For each broker, determine state:
        │   ├─ If cordoned=true AND removed=false → Collect in "to-uncordon" set
        │   ├─ If cordoned=false AND removed=true → Collect in "to-cordon" set
        │   └─ Otherwise (correct state) → Skip
        │
        ├─ Are both collections empty?
        │   ├─ Yes → All brokers in correct state, proceed with normal reconciliation (scale down if any)
        │   │
        │   └─ No → Operations needed
        │       ├─ Cordon brokers in "to-cordon" set (if not empty)
        │       ├─ Uncordon brokers in "to-uncordon" set (if not empty)
        │       ├─ Operations succeed or fail → Mark check as failed
        │       └─ Revert scale down (if any), wait for propagation, retry next reconciliation
```

#### Detailed step-by-step workflow

**Step 1: Check if cordoning is supported (metadata version check)**

When `KafkaClusterCreator.prepareKafkaCluster()` is called, always check if cordoning is supported, even when no scale down is happening.
This is necessary to handle cleanup when users cancel a scale down operation.
If a scale down was initiated, brokers were cordoned, but then the user reverted the `KafkaNodePool` replicas back before the scale down happened, those brokers remain cordoned.
The check must run on every reconciliation to detect and uncordon such orphaned brokers.
Without this cleanup, cordoned brokers would remain excluded from partition placement indefinitely.

Call `AdminClient.describeFeatures()` to get the finalized `metadata.version`:

**If `describeFeatures()` fails (exception thrown):**
- Unable to check metadata version, skipping cordoning
- Return success and proceed with auto-rebalancing without cordoning
- Metadata version check failure is treated as "cordoning unavailable" rather than blocking scale down

Proceeding the reconciliation when `describeFeatures()` fails also helps on a new Kafka cluster creation.
In such a case, on the first reconciliation, the cluster doesn't exist yet and getting the `metadata.version` will fail without blocking the next steps.

**If `describeFeatures()` succeeds:**
- Parse `metadata.version` from the response
- **If `metadata.version` < IBP_4_3_IV0**:
  - Cordoning is not supported
  - Return success and proceed with standard auto-rebalancing without cordoning
- **If `metadata.version` >= IBP_4_3_IV0**:
  - Cordoning is supported, checking broker states at step 2

Checking the metadata version is necessary regardless of which Apache Kafka version the operator supports.
Even if a future operator release requires Kafka 4.3+, users may still run with `metadata.version < IBP_4_3_IV0` (e.g., during phased upgrades or for compatibility reasons).
In such cases, cordoning remains unavailable despite running Kafka 4.3+, so the runtime metadata version check is essential.

**Step 2: Query current cordoning state for all brokers**

Call `AdminClient.describeLogDirs()` for all brokers in the cluster (not just brokers being removed, if any).
The API returns information about each log directory on each broker, including whether each directory is cordoned or not.

**If `describeLogDirs()` fails (exception thrown):**
- Failed to check cordoning state
- Set `cordoningCheckFailed = true`
- Revert scale down if any (keep pods running)
- Return (will retry on next reconciliation)

**If `describeLogDirs()` succeeds:**
- Parse the response to map each broker ID → is cordoned
- For each broker:
  - Get all its log directories and check if **ALL** log directories have `LogDirDescription.isCordoned() = true`
  - A broker is considered cordoned **only if ALL** its log directories are cordoned
  - Store the result in the map
- Continue to Step 3

**Step 3: Analyze broker states and collect cordoning/uncordoning operations needed**

Initialize two collections:
- `Set<Integer> brokersToCordon`: Brokers that need to be cordoned
- `Set<Integer> brokersToUncordon`: Brokers that need to be uncordoned

For each broker in the cluster, determine its current and desired state:
- Get whether the broker is currently cordoned (from Step 2 results)
- Get whether the broker is being removed (from `kafka.removedNodes()`)
- Compare the two states:
  - **If cordoned AND not being removed**: Add to `brokersToUncordon` (orphaned cordoning, needs cleanup)
  - **If not cordoned AND being removed**: Add to `brokersToCordon` (scale down in progress, needs cordoning)
  - **Otherwise**: Broker is in correct state, skip (either being removed and already cordoned, or not being removed and not cordoned)

This logic handles all scenarios:
- **Normal scale down**: Removed brokers are uncordoned → added to `brokersToCordon`
- **Cordoning propagated**: Removed brokers are cordoned → both collections empty → proceed with auto-rebalancing
- **User cancelled scale down**: Previously cordoned brokers are no longer being removed → added to `brokersToUncordon`
- **Partial failure recovery**: Some removed brokers cordoned, others not → uncordoned ones added to `brokersToCordon`
- **No scale down happening**: Any cordoned brokers from previous operations → added to `brokersToUncordon` for cleanup

Continue to Step 4.

**Step 4: Execute cordoning/uncordoning operations**

**If both `brokersToCordon` and `brokersToUncordon` collections are empty:**
- All brokers are in the correct state
- All brokers in correct cordoning state, set `cordoningCheckFailed = false`
- Return success and proceed with auto-rebalancing and scale down

**If at least one collection is not empty:**

Cordoning and/or uncordoning operations are needed, execute them:

- **Cordon brokers** (if `brokersToCordon` not empty): Set `cordoned.log.dirs=*` via `incrementalAlterConfigs()` for each broker
- **Un-cordon brokers** (if `brokersToUncordon` not empty): Delete `cordoned.log.dirs` config via `incrementalAlterConfigs()` for each broker

In case of any failure to cordone/uncordone a broker, continue anyway (will retry on the next reconciliation).

After operations (success or failure):
- Set `cordoningCheckFailed = true` (operations were performed, need to wait for propagation)
- If scale down is happening: revert scale down (keep pods running)
- Save blocked nodes (if any): `scalingDownBlockedNodes.addAll(kafka.removedNodes())`
- Update status: `status.autoRebalance.modes[remove-brokers].brokers` will be set if brokers are being removed (auto-rebalancing starts in same reconciliation)
- Add warning condition if scale down reverted: "Reverting scale-down of KafkaNodePool X due to pending broker cordoning"
- Return (next reconciliation will verify propagation)

Reverting the scale down in case of failures helps with:
- If cordoning fails due to transient error (network, broker unavailable), we'll retry on next reconciliation
- If it's a permanent failure (permissions, unsupported despite version check), repeated warnings will alert operators
- Reverting ensures pods aren't removed before cordoning is active

#### Reconciliation scenarios

Following some examples showing how all the above steps fit within a single reconciliation loop but also together across multiple reconciliations. 

**Reconciliation N (first detection of scale down):**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. Metadata version check: >= IBP_4_3_IV0 → cordoning supported
3. describeLogDirs: All brokers uncordoned
4. Collections: `brokersToCordon = [3, 4]`, `brokersToUncordon = []`
5. Execute: Cordon brokers [3, 4]
6. Result: `cordoningCheckFailed = true`
7. Revert scale down (pods stay at 5)
8. Update status with blocked nodes [3, 4]
9. Auto-rebalancing reconciler sees blocked nodes, updates status, but doesn't create KafkaRebalance yet (pods still exist)

**Reconciliation N+1 (cordoning propagated):**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. Metadata version check: >= IBP_4_3_IV0 → cordoning supported
3. describeLogDirs: Brokers [0, 1, 2] uncordoned, brokers [3, 4] cordoned
4. Collections: `brokersToCordon = []`, `brokersToUncordon = []` (all in correct state)
5. Result: `cordoningCheckFailed = false`
6. Auto-rebalancing starts (KafkaRebalance resource created)
7. After rebalancing completes, proceed with scale down and pods [3, 4] removed

**Reconciliation N+1 (alternate: cordoning not yet propagated):**
1. Scale down detected: 5 → 3
2. Metadata version check: >= IBP_4_3_IV0
3. describeLogDirs: Broker [3] cordoned, broker [4] not yet cordoned
4. Collections: `brokersToCordon = [4]`, `brokersToUncordon = []`
5. Execute: Cordon broker [4] again (idempotent)
6. Result: `cordoningCheckFailed = true`
7. Revert scale down (pods stay at 5), wait for next reconciliation

**Reconciliation N+2 (cordoning fully propagated):**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. Metadata version check: >= IBP_4_3_IV0 → cordoning supported
3. describeLogDirs: Both [3, 4] cordoned
4. Collections: `brokersToCordon = []`, `brokersToUncordon = []` (all in correct state)
5. Result: `cordoningCheckFailed = false`
6. Auto-rebalancing starts (KafkaRebalance resource created)
7. After rebalancing completes, proceed with scale down and pods [3, 4] removed

#### Rollback scenario (user cancels scale down)

The user may initiate a scale down operation, causing the operator to cordon brokers. 
Since cordoning requires propagation across reconciliations before the auto-rebalancing and actual scale down proceed, the user has a window of time to change their mind and revert the `KafkaNodePool` replicas back to the original count.
In this scenario, the previously cordoned brokers must be uncordoned since they are no longer scheduled for removal.
The cordoning check handles this cleanup automatically by detecting brokers that are cordoned but not in the removal list.
Without such cleanup, the brokers would remain cordoned indefinitely and the controller would not assign any new partitions to them.

Following a sequence of reconciliations describing such scenario.

**Reconciliation N:**
1. Scale down detected: 5 → 3 (brokers [3, 4] to be removed)
2. Metadata version check: >= IBP_4_3_IV0 → cordoning supported
3. describeLogDirs: All brokers uncordoned
4. Collections: `brokersToCordon = [3, 4]`, `brokersToUncordon = []`
5. Execute: Cordon brokers [3, 4]
6. Result: `cordoningCheckFailed = true`
7. Revert scale down (pods stay at 5)
8. Update status with blocked nodes [3, 4]
9. Auto-rebalancing reconciler sees blocked nodes, updates status, but doesn't create KafkaRebalance yet (pods still exist)

**User reverts replicas back to 5 before next reconciliation**

**Reconciliation N+1:**
1. Metadata version check: >= IBP_4_3_IV0 → cordoning supported
2. describeLogDirs: Brokers [0, 1, 2] uncordoned, brokers [3, 4] cordoned
3. `removedNodes()` is empty (no scale down anymore)
4. Collections analysis:
   - For brokers [0, 1, 2]: `isCordoned=false, isRemoved=false` → correct state, skip
   - For brokers [3, 4]: `isCordoned=true, isRemoved=false` → cleanup needed
   - Result: `brokersToUncordon = [3, 4]`, `brokersToCordon = []`
5. Execute: Uncordon brokers [3, 4]
6. Result: `cordoningCheckFailed = true`
7. No scale down to revert (no pods removed)
8. Return (next reconciliation will verify uncordoning completed)

**Reconciliation N+2 (uncordoning propagated):**
1. No scale down detected (replicas back to 5)
2. Metadata version check: >= IBP_4_3_IV0 → cordoning supported
3. describeLogDirs: All brokers uncordoned
4. Collections: `brokersToCordon = []`, `brokersToUncordon = []` (all in correct state)
5. Result: `cordoningCheckFailed = false`
6. Proceed with normal reconciliation

The workflow always runs (even when `removedNodes()` is empty), so the collection logic automatically detects and cleans up orphaned cordoning from cancelled scale downs.

## Affected/not affected projects

This proposal affects only the `strimzi-kafka-operator` by adding the logic for cordoning (and uncordoning when needed) brokers within the `KafkaClusterCreator` class.

## Compatibility

The KIP-1066 is available in Kafka 4.3.0+ (metadata version IBP_4_3_IV0).
Older Kafka versions or metadata versions do not support cordoning.
With this proposal, the Strimzi Cluster Operator gracefully handles both scenarios:
- **Kafka >= 4.3.0 with metadata.version >= IBP_4_3_IV0**: Enable automatic cordoning during scale down
- **Kafka < 4.3.0 or metadata.version < IBP_4_3_IV0**: Use existing auto-rebalancing without cordoning (no behavior change)

Upgrading the operator doesn't need any migration steps:
- Upgrading the operator with existing Kafka < 4.3.0 clusters: No behavior change
- Upgrading the operator with Kafka >= 4.3.0 clusters: Cordoning automatically enabled for subsequent scale down operations

When upgrading an Apache Kafka cluster from 4.2 to 4.3, cordoning behavior depends on the `metadata.version`:
- If `metadata.version` is also upgraded to IBP_4_3_IV0 or higher: Cordoning is automatically enabled
- If `metadata.version` remains at < IBP_4_3_IV0 (e.g., for phased upgrades or compatibility): Cordoning remains disabled until the metadata version is upgraded

This runtime check ensures the operator never attempts to use cordoning features when the cluster's metadata version doesn't support them, regardless of the Kafka binary version.

### Impact on existing auto-rebalancing users

**Users with auto-rebalancing enabled (Kafka < 4.3.0 or old metadata version):**
- No changes to behavior
- Scale down continues to work as before
- No action required

**Users with auto-rebalancing enabled (Kafka >= 4.3.0 with metadata.version >= IBP_4_3_IV0):**
- Automatic cordoning is enabled transparently
- No configuration changes required
- Scale down may take 1-2 additional reconciliation cycles for cordoning propagation

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