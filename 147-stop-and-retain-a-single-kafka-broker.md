# Stop and retain a single Kafka broker

This proposal adds a way to **stop** a single Kafka broker — take it fully offline (no `Pod`, no Kafka process) while **retaining** its node ID, storage, and configuration — and keep it stopped until the operator is explicitly told to start it again. The rest of the cluster keeps reconciling normally. This is for planned maintenance and investigation where you want the *same* broker back afterwards, not a permanent removal.

Today the only ways to take a broker down are destructive or transient: scaling down a `KafkaNodePool` permanently removes the node (and, post-[108](https://github.com/strimzi/proposals/blob/main/108-list-unregister-nodes-admin-api.md), unregisters it), while deleting the pod just gets it recreated. There is no "off, but kept" state.

### Terminology

A **node** is a Kafka node in the Strimzi/KRaft sense: one `NodeRef(podName, nodeId, poolName, controller, broker)` — exactly one `Pod`, one integer node ID, with role `broker`, `controller`, or both. **Stopped** means the node's `Pod` does not exist and no Kafka process runs for it, but its node ID, `PersistentVolumeClaim`(s), and pool membership are retained so the identical broker can be started again later. v1 targets **broker-only** nodes (see Scope).

## Current situation

Strimzi's pod lifecycle has no concept of a node that exists but is intentionally not running. Every node is binary — running, or removed entirely:

- `KafkaPool.nodes()` turns each desired node ID from `NodeIdAssignor` into a `NodeRef`, and `KafkaCluster.generatePodSets()` materializes **one `Pod` entry per `NodeRef`** into `StrimziPodSet.spec.pods`. There is no path that yields a desired node without a pod.
- `StrimziPodSetController` treats `spec.pods` as absolute: it creates any listed pod that is missing (`maybeCreateOrPatchPod`) and **deletes any matching pod not listed** (`removeDeletedPods`). A terminal-state pod is deleted and recreated. So `kubectl delete pod` always bounces back, and hand-editing the `StrimziPodSet` is overwritten on the next `KafkaReconciler` run.
- The Kafka container command is hardcoded to `/opt/kafka/kafka_run.sh`, which unconditionally execs `kafka-server-start.sh`. There is no idle/sleep mode and no per-node command override.
- `KafkaRoller` assumes every input node should be running. (It does not throw on a *missing* pod — `restartIfNecessary` no-ops when `podOperations.get(...)` returns `null`; the `FatalProblem("Pod is unschedulable or is not starting")` fires only for a pod that *exists* and is stuck/Pending. The hazard for a stopped node is therefore not that path but the transient window described under "Readiness and reconciliation.")
- No annotation or CRD field expresses stop/park/offline; no `.status` field tracks per-node running state.

The adjacent controls all do something else: `strimzi.io/pause-reconciliation` freezes the whole CR but does not stop Kafka; `strimzi.io/manual-rolling-update` restarts a pod; scale-down + `strimzi.io/skip-broker-scaledown-check` permanently removes a node; `strimzi.io/delete-pod-and-pvc` deletes then recreates.

## Motivation

The driving use case is **planned host maintenance** where the same broker must return with its data intact: take the broker offline so the underlying Kubernetes host (or its disks) can be serviced, then bring the identical broker back. Because the broker's storage and identity are retained, on restart it rejoins, runs log recovery, and catches up — no partition reassignment, no re-replication of the full dataset, no node ID churn.

This earns its keep specifically when the broker's pod **cannot** simply be rescheduled elsewhere:

- **Local / host-attached storage** (`PersistentVolume`s bound `WaitForFirstConsumer` to a specific host). Draining the host cannot move the broker — its pod just goes `Pending` on the gone node — so there is no graceful way to take it down and bring it back on the same disks today.
- **The host or its disks are themselves the maintenance target** (firmware, disk replacement, kernel work) — the broker must be down on *that* host, not relocated.
- **Deliberate isolation** of one broker for investigation/forensics without the operator restarting it underneath you.

For networked storage where the pod can freely reschedule, draining the Kubernetes host already moves the broker (the `StrimziPodSet` recreates the pod elsewhere and the volume reattaches); "stop" is not needed there. The feature targets the cases above, where today there is no first-class answer.

Doing this today forces a bad choice:

- **Scale down** to drop the broker, then scale back up — but that permanently removes the node, reassigns/loses its replicas, unregisters it (108), and the broker that comes back is a *different* node ID with empty storage that must re-replicate everything.
- **Delete the pod** — the `StrimziPodSet` controller recreates it within seconds, so the broker never stays down.
- **Pause reconciliation** on the whole `Kafka` CR and manually delete the pod — this freezes config, cert rotation, scaling, and rolling for *every* node in the cluster, and is fragile.

What is missing is a first-class "stop this one broker, keep everything about it, start it when I say" control.

## Proposal

Introduce an opt-in, per-node **stop** control. A stopped broker's `Pod` is removed while its node ID, storage, and pool membership are retained; the operator keeps reconciling every other node. Starting it again is a single declarative change.

This sits on a broader "how much of a node's lifecycle does the operator manage" axis alongside the existing whole-CR `strimzi.io/pause-reconciliation` and (if accepted) per-node skip-rolling controls. This proposal is deliberately scoped to the **stopped** point on that axis and stands alone; a future unified per-node lifecycle-state field could subsume these, but that consolidation is out of scope here.

### Scope (v1)

- **Brokers only.** A stop targeting a controller node (controller-only or combined broker+controller) is **refused** — the entry is ignored, a `.status` condition and warning event explain why, and the operator keeps managing that node. Stopping a controller removes it from the KRaft metadata quorum, which needs dedicated quorum-safety machinery; that is future work.
  - **Known limitation — combined-role deployments.** Because a combined broker+controller node *is* a controller, every stop targeting it is refused. In a fully **combined-mode** cluster (every node holds both roles, common on small KRaft deployments — and exactly the clusters most likely to run the host-attached storage this feature targets) v1 can stop *nothing*. The feature is only useful where dedicated broker-only nodes exist; combined-node stops arrive with the controller work.
- **At least one broker must remain running (enforced).** Despite the title's "a single broker," `stoppedNodes` is a per-pool `array<integer>` and nothing structurally bounds it. The operator must refuse a stop that would leave **zero** running brokers cluster-wide (ignored entry + warning + `.status`), because the bootstrap-dependent reconcile steps below would otherwise wedge regardless of `runningNodes()`. v1's safe stance is "one broker at a time"; stopping enough brokers to break bootstrap/min-ISR is the operator's responsibility, surfaced loudly (see Safety).
- **Stop = no pod, identity retained.** The node stays in the pool's desired set (its ID is not freed, not unregistered, its PVC is not deleted), but no `Pod` is generated for it.
- **Behind a feature gate**, default off.

### API: trigger surface

The trigger is a durable field on `KafkaNodePool.spec`:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: brokers
spec:
  replicas: 6
  roles:
    - broker
  stoppedNodes: [2]    # node IDs in this pool to stop and retain (offline)
```

A node is stopped iff its ID appears in its pool's `spec.stoppedNodes`. To start it again, remove the ID from the list. The field is schema-validated as `array<integer>`; entries that parse but are not meaningful (an ID not in this pool, a duplicate, a node that resolves to a controller) are **ignored** with a warning event and `.status` condition, while valid entries still apply — ignoring is the fail-safe direction.

**Why a `spec` field and not a pod annotation.** Unlike controls that leave the node running, stopping deletes the pod — so there is no pod to carry the signal, and nothing to annotate to keep it stopped or to start it. The signal must live on a retained, declarative resource. `KafkaNodePool` is user-authored and reconciled every loop, so it survives pod deletion and is the natural source of truth (also durable under any declarative re-apply). `replicas` is deliberately **not** decremented when stopping — the node remains a member of the pool; `stoppedNodes` only controls whether its pod runs.

### How a node is taken offline

The key design point is the distinction between two node sets. `KafkaCluster.nodes()` stays the **desired-and-retained** set (it still includes a stopped node), so everything that should keep working for a stopped node keeps working unchanged: PVC generation, certificate Secrets, per-broker `ConfigMap`s, listener/Service generation, `KafkaNodePool.status.nodeIds`, the 049 scale-down check (sees no removal), and 108 unregistration suppression (sees the node as still present). A new accessor, **`runningNodes()` = `nodes()` minus stopped nodes**, is introduced for every place that requires a node to be *live right now*.

The mechanism is **pod omission with identity retention**, injected as:

1. **Keep the node desired, drop only its pod.** The stopped node ID stays in `NodeIdAssignor`'s `desired` set and in `KafkaNodePool.status.nodeIds`, so it is *not* seen as a scale-down or removal. `KafkaCluster.generatePodSets()` omits the pod entry for a stopped node when building `StrimziPodSet.spec.pods`. Its PVC, certs, and config are still generated (it is still in `nodes()`).
2. **Let `StrimziPodSetController` delete the now-absent pod, retaining the PVC.** A previously-running pod is deleted by `removeDeletedPods` (background propagation; PVCs are untouched because they are reconciled from the desired set, not from `spec.pods`, and the node is still a pool member). The deletion is a normal graceful shutdown (see Safety). The controller must not treat the intentional absence as an error.
3. **Exclude stopped nodes from "must be live now" waits via `runningNodes()`.** The cluster-level readiness wait (`KafkaReconciler.podsReady()`) must use `runningNodes()`, not `nodes()`, or it waits the full operation timeout on the never-ready absent pod every reconcile (see "Readiness and reconciliation"). The node set handed to `KafkaRoller` (`KafkaReconciler.maybeRollKafka`, called inline from `rollingUpdate()`) must also use `runningNodes()` — not because the roller throws on a missing pod (it no-ops on `null`), but because of an **async-deletion race**: `removeDeletedPods` runs in the *separate* `StrimziPodSetController` (background propagation), while `rollingUpdate()` runs inline in the *same* `KafkaReconciler` pass, so on the first reconcile that stops a node the pod is often still present (terminating). Without `runningNodes()` the roller would `await(isReady(pod), operationTimeoutMs, …)` on a dying pod and/or evaluate restart reasons against a `StrimziPodSet` that no longer lists it. This is the load-bearing change.
4. **Do not unregister.** The KRaft node-unregistration logic ([081](https://github.com/strimzi/proposals/blob/main/081-unregistration-of-KRaft-nodes.md)/108) already treats a node still in `brokerNodes()` as retained, so a stopped node is not unregistered — no change needed beyond keeping it in `nodes()`.
5. **Refuse controller stops at parse time, before the pod is omitted.** The controller-refusal (see Scope) is **not** "no change needed": there is no existing pre-podset gate keyed on per-node roles. Role is known at the pool level (`KafkaPool.isController()`), so the refusal must be computed where `stoppedNodes` is parsed into the pool model (around `KafkaPool.fromCrd`) and threaded through so `generatePodSets()` only omits pods for *validated broker-only* stops. Evaluating it there — before `generatePodSets()` would drop the pod — guarantees the operator never drops a voter and only then refuses, leaving a half-stopped cluster.

Cruise Control needs **no** special handling: Strimzi forbids CC self-healing and the broker-failure detector (`self.healing.`, `kafka.broker.failure.detection.enable` are in `FORBIDDEN_PREFIXES`), and the 078 auto-rebalance fires only on `replicas` deltas (none here). A stopped broker stays in CC's capacity config as a normal broker and its replicas are not moved off — automatically, not by anything this proposal imposes.

### Readiness and reconciliation

This is the change that makes the feature actually work. Today `KafkaReconciler.podsReady()` awaits readiness of **every** `kafka.nodes()` pod; for an absent (stopped) pod, readiness never becomes true, the wait times out after `STRIMZI_OPERATION_TIMEOUT_MS`, the whole reconcile **fails**, and the `Kafka` CR is marked `Ready=False` (`reason=TimeoutException`) on every loop. Left unaddressed, "stop one broker" would wedge reconciliation for the entire cluster.

- `podsReady()` (and any other cluster-level pod readiness wait) operates on `runningNodes()`, so a stopped node does not block readiness.
- With all *running* nodes ready, the `Kafka` CR is `Ready=True`; the separate `NodeStopped` condition (below) carries the "one node intentionally down" signal and is **not** folded into `Ready`.
- On **start** (ID removed from `stoppedNodes`), the node re-enters `runningNodes()`; the next reconcile regenerates its pod and `podsReady()` now waits for it, so the reconcile does not report success until the restarted broker is actually ready.

**`runningNodes()` is not sufficient on its own** — several pipeline steps that run right after `podsReady()` are *cluster-wide*, not per-node, and only need ≥1 reachable broker rather than a per-node filter: `serviceEndpointsReady()` / `headlessServiceEndpointsReady()` wait for at least one endpoint on the bootstrap/brokers `Service`s, and `clusterId()`, `defaultKafkaQuotas()`, `metadataVersion()`, and `nodeUnregistration()` each open an Admin client to the bootstrap. With at least one running broker these all pass and a single stop in a multi-broker cluster is fine. But if a stop drains the **last** running broker, these steps time out and wedge the reconcile `NotReady` no matter what `runningNodes()` does — which is precisely why the "at least one broker must remain running" invariant (Scope) is enforced rather than advisory.

### Safety

Stopping a broker takes its replicas offline (they are retained on disk but not served), so partitions it leads/follows lose a replica until it returns.

- **Availability check — stance and cost (open question for review).** If stopping a broker would push any partition below `min.insync.replicas`, the operator raises a loud warning event and a `.status` condition naming the affected partitions; partitions whose only in-sync replica is on the stopped node become unavailable for `acks=all` writes (surfaced via metrics). The proposal's default is **warn-and-allow** (honor the explicit operator action), deliberately laxer than [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md)'s refuse-by-default — because the impacts differ in kind: scale-down **permanently removes** the broker and risks data loss, whereas a stop **retains data** and the unavailability is temporary and self-healing on restart. Two caveats reviewers should weigh: (1) given the maintenance motivation, **refuse-by-default with an explicit `force` override** may be the safer v1 stance — this is genuinely a judgement call, noted as the alternative. (2) The per-partition ISR check has **no existing home** in a stop path; the only comparable machinery is 049's conditional `describeTopics` scan in `KafkaClusterCreator`. Running a full ISR scan **every reconcile** is Admin-heavy and costly on large clusters, so it must be gated (only when `stoppedNodes` is non-empty) and ideally reuse 049's scan rather than add a second one. The `..._sole_isr_on_stopped_node` / `..._under_min_isr` metrics are produced from that same scan, not an independent one.
- **Data is retained, not lost.** PVCs are kept; on start the broker recovers its logs and rejoins the ISR. This is the key difference from scale-down.
- **Scale-down of a pool with a stopped member is forbidden.** `stoppedNodes` governs whether a pod runs, *not* pool membership — so the two must not conflict. If a user lowers `replicas` (or uses `remove-node-ids`) such that a stopped node's ID would leave the desired set, that ID enters `removedNodes()` and is treated as a real removal: the 049 check, 108 unregistration, and PVC deletion all fire, destroying exactly what "retain" promised. v1 refuses a scale-down that would remove a currently-stopped node (warning + `.status`); the operator must start the node first, then scale down. (The existing 049 replica-holding check already blocks the common case, but membership removal of a stopped node must be refused explicitly, not left to 049.)
- **Graceful shutdown.** Stopping deletes the pod via the normal background propagation, so the broker receives SIGTERM (forwarded by `tini`) and runs a controlled shutdown (leadership handoff, fence, flush). For partition-heavy brokers the default `terminationGracePeriodSeconds` (30s) may be too short for a clean handoff; operators should set `template.pod.terminationGracePeriodSeconds` accordingly (a longer maintenance grace may be applied on the stop path).
- **PodDisruptionBudget interaction.** A stopped broker has no pod, so it counts against the pool/cluster PDB's available replicas: with `maxUnavailable: 1` a single stop consumes the whole budget and blocks *voluntary* disruptions (host drains, manual rolls) of other brokers — directly relevant since the host-maintenance flow drains nodes. The PDB is therefore recomputed excluding stopped nodes, so the budget reflects only running brokers.
- **CA rotation while stopped.** Certificate Secrets keep rotating for a stopped node (it stays in `nodes()`), but the broker is not running to pick them up. A stop that outlives the cluster CA cert overlap window may require an extra roll on restart, or for the CA renewal window to be widened; long stops crossing a CA rotation should be avoided or planned around.
- **No auto-start.** The operator never starts a stopped node on its own — that would defeat the purpose. Start is always an explicit removal of the ID from `stoppedNodes`.

### Restart flow

Removing the node ID from `stoppedNodes` makes the next reconcile regenerate its pod (same name/ID, same PVC). The container starts Kafka, performs log recovery, and rejoins the quorum/ISR. The node passes through the normal readiness path; the operator resumes managing it (config, certs, rolling).

### Feature gate

A `StopKafkaNode` feature gate, default **disabled** in the introducing release, graduating alpha → beta → GA. The CRD always accepts `spec.stoppedNodes` (Kubernetes admission does not know about feature gates), but when the gate is disabled the operator ignores it entirely — no pod omission and no `StopRefused` conditions (so users do not see refusals for entries that would be valid). The gate is warranted because this lets a human take a broker offline indefinitely — a new, availability-relevant behavior.

### Status and metrics

- A `Kafka` CR `.status` condition (`type: NodeStopped`) listing stopped node IDs and, per node, the time it was stopped ("stopped since `<timestamp>`" via `lastTransitionTime`). Mirror per-node detail in the owning `KafkaNodePool.status`; a node that resolves to a controller shows `StopRefused` instead. This condition is informational and is **not** folded into the `Ready` condition.
- Kubernetes events when a node is stopped, started, or a controller stop is refused.
- Metrics (`strimzi_` prefix): `strimzi_nodes_stopped` (gauge, per cluster); `strimzi_node_stopped_start_timestamp_seconds` (gauge, per node, backing "stopped since" and long-lived-stop alerts); `strimzi_partitions_under_min_isr` and `strimzi_partitions_sole_isr_on_stopped_node` (gauges) so the availability impact is alertable.
- **Observability caveats** to document so they are not mistaken for faults:
  - A stopped broker's per-broker `Service` (still generated, since the node is in `nodes()`) has empty `EndpointSlice`s, so `kafka-exporter` probes and external clients see connection-refused for it.
  - For `nodePort` listeners, per-broker addresses are computed from live pods, so the stopped node is omitted from `Kafka.status.listeners[].addresses` gracefully. For the other listener types only the *bootstrap* address appears in `ListenerStatus`, so there is no per-broker address to omit — the earlier blanket "omitted from `status.listeners[].addresses`" was imprecise. The per-broker external resources (LB/route/ingress) for the stopped node still exist and resolve, so the listener reconciler does **not** wedge.
  - `KafkaNodePool.status.replicas` and `status.nodeIds` continue to **include** the stopped node (they reflect desired membership, not running pods), so `status.replicas` will exceed the count of running pods — monitoring that compares the two may flag it; this is expected.
  - The `NodeStopped` condition and the metrics above make the intentional cause discoverable.

## Testing strategy

- **Unit tests.** A node listed in `stoppedNodes` produces no pod entry in the generated `StrimziPodSet`, but stays in `nodes()`/`desired`/`status.nodeIds` and is absent from `runningNodes()`. The PVC, cert Secret, and per-broker `ConfigMap` are still generated. `podsReady()` operates on `runningNodes()`, so with one node stopped and the rest ready the reconcile **succeeds** and the `Kafka` CR is `Ready=True` (the core regression test — without `runningNodes()` it would time out and wedge `NotReady`). PDB is computed from `runningNodes()` (a stopped node does not consume the disruption budget). The 049 check sees no removal and 108 does not unregister the node. A controller-targeting stop (controller-only **and** combined broker+controller) is refused with `StopRefused` at parse time, *before* the pod is omitted. A stop that would leave **zero** running brokers cluster-wide is refused. A scale-down that would remove a currently-stopped node is refused. Parse-failure: invalid `stoppedNodes` entries ignored, valid ones applied, warning surfaced. Availability: stopping a broker that would breach min-ISR is honored (warn-and-allow) with a warning condition, and the ISR scan runs only when `stoppedNodes` is non-empty.
- **System tests.** Stopping a broker removes its pod and keeps it down across reconciles while the `Kafka` CR stays `Ready=True`; the operator keeps rolling/reconciling other brokers; voluntary disruption of another broker is not blocked by the stopped one (PDB). Starting it (removing the ID) recreates the pod with its PVC, the reconcile waits for it, and the broker rejoins the ISR. Controller stop is refused. With the gate off, `stoppedNodes` is accepted by the CRD but inert (no `StopRefused`, no pod omission).

## Affected/not affected projects

Affected (`strimzi/strimzi-kafka-operator`):

- `KafkaCluster.generatePodSets` — omit stopped nodes' pods; new `runningNodes()` accessor.
- `KafkaReconciler.podsReady()` — use `runningNodes()` (the load-bearing fix).
- `KafkaReconciler.maybeRollKafka` / `rollingUpdate()` — feed the roller `runningNodes()` (avoids the async-deletion race; the roller itself no-ops on a missing pod).
- `KafkaPool.fromCrd` / `NodeIdAssignor` — parse `stoppedNodes`, retain stopped IDs as desired, and **compute the controller-refusal here** so only validated broker-only stops reach `generatePodSets` (a genuinely new gate, not "no change").
- `KafkaCluster.generatePodDisruptionBudget` — today sizes the PDB from `nodes().size()`; must use `runningNodes()` or stopped nodes starve the voluntary-disruption budget.
- `KafkaReconciler.scaleDown()` / `KafkaClusterCreator` removal path — refuse removing a currently-stopped node.
- The availability/ISR check — gated on non-empty `stoppedNodes`, reusing 049's `describeTopics` scan rather than adding a per-reconcile one.
- `KafkaNodePool` CRD (`spec.stoppedNodes`), `.status` conditions, metrics, and the `StopKafkaNode` feature gate.

No change needed (tolerate a retained-but-pod-less node by construction, because they key off the desired set or off live pods): PVC retention (`deletePersistentClaims` diffs `generatePersistentVolumeClaims()`), certificate Secrets / per-broker `ConfigMap`s, 108 unregistration (`nodeUnregistration` uses `brokerNodes()`), Cruise Control capacity + 078 auto-rebalance (no replica delta), `CaReconciler` (its roller set is derived from the `StrimziPodSet`, so the stopped node is already absent), `manualRollingUpdate()` (derived from `spec.pods`/pod annotations), and `nodePortExternalListenerStatus` (pod-driven). `KafkaListenersReconciler` intentionally stays on `nodes()` so per-broker Services/external resources for the stopped node keep existing and resolving (it must **not** be "fixed" to `runningNodes()`).

Not affected: bridge, oauth, test-clients, drain-cleaner. No new Kafka Agent API is required.

## Compatibility

Fully opt-in and gated; with the gate disabled the new field is inert and behavior is unchanged. The `KafkaNodePool` CRD gains an optional `spec.stoppedNodes` field — additive and backward compatible. A downgrade to an operator without the feature, while a node is stopped, would regenerate the pod and start the broker (it would no longer understand `stoppedNodes`); document that operators should start all stopped nodes before downgrading.

## Rejected alternatives

- **Idle/sleep pod (keep the pod, don't run Kafka).** A new entrypoint mode that sleeps instead of starting Kafka would keep the pod object around. Rejected for the maintenance use case: it wastes the pod's CPU/RAM, still occupies a Kubernetes node (so it does not free the host for servicing), and is a semantically confusing "running pod, dead broker." Pod omission is the cleaner "offline." (Could be revisited as an option if a use case needs the pod to stay schedulable.)
- **Scale down then scale up.** Permanently removes the node: replicas reassigned, node unregistered (108), and the returning broker is a new ID with empty storage that must re-replicate everything. Wrong semantics for "same broker back."
- **Delete the pod / `delete-pod-and-pvc`.** The `StrimziPodSet` controller recreates the pod immediately; `delete-pod-and-pvc` also wipes storage.
- **Whole-CR `pause-reconciliation` + manual pod delete.** Cluster-wide scope (freezes config/cert/scaling/rolling for every node) and fragile; not a per-node control.
- **A pod annotation as the trigger.** Impossible to keep durable here: stopping deletes the pod, so there is no object to carry the signal or to start the node from. The retained `spec` field is required.

## Out of scope / Future work

- **Controller stop support**, with KRaft quorum-safety guardrails (refuse-by-default when stopping would risk quorum, an explicit force override, per-reconcile quorum re-evaluation), mirroring the care needed for controllers.
- **Refuse-by-default availability check** (like the scale-down check in [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md)) as an opt-in stricter mode, if warn-and-allow proves too permissive.
- **Auto-expiry of a stop** after a configurable duration, so a forgotten stop does not leave a broker offline indefinitely (metrics make it alertable in the meantime).
- **First-class Drain Cleaner / maintenance integration** to stop and start a broker as part of a host-maintenance workflow.
