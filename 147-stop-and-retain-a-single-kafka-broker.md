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
- `KafkaRoller` assumes every input node should be running: a non-starting pod raises `FatalProblem("Pod is unschedulable or is not starting")`.
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
3. **Exclude stopped nodes from "must be live now" waits via `runningNodes()`.** The cluster-level readiness wait (`KafkaReconciler.podsReady()`) and the node set handed to `KafkaRoller` (`KafkaReconciler.maybeRollKafka`) must use `runningNodes()`, not `nodes()` — otherwise the operator waits forever on the absent pod (see "Readiness and reconciliation"). This is the load-bearing change.
4. **Do not unregister.** The KRaft node-unregistration logic ([081](https://github.com/strimzi/proposals/blob/main/081-unregistration-of-KRaft-nodes.md)/108) already treats a node still in `brokerNodes()` as retained, so a stopped node is not unregistered — no change needed beyond keeping it in `nodes()`.
5. **Refuse controller stops before omitting the pod.** For a combined broker+controller node, the controller-refusal (see Scope) must be evaluated *before* `generatePodSets()` would omit the pod — otherwise the operator could drop a voter and only then refuse, leaving a half-stopped cluster.

Cruise Control needs **no** special handling: Strimzi forbids CC self-healing and the broker-failure detector (`self.healing.`, `kafka.broker.failure.detection.enable` are in `FORBIDDEN_PREFIXES`), and the 078 auto-rebalance fires only on `replicas` deltas (none here). A stopped broker stays in CC's capacity config as a normal broker and its replicas are not moved off — automatically, not by anything this proposal imposes.

### Readiness and reconciliation

This is the change that makes the feature actually work. Today `KafkaReconciler.podsReady()` awaits readiness of **every** `kafka.nodes()` pod; for an absent (stopped) pod, readiness never becomes true, the wait times out after `STRIMZI_OPERATION_TIMEOUT_MS`, the whole reconcile **fails**, and the `Kafka` CR is marked `Ready=False` (`reason=TimeoutException`) on every loop. Left unaddressed, "stop one broker" would wedge reconciliation for the entire cluster.

- `podsReady()` (and any other cluster-level pod readiness wait) operates on `runningNodes()`, so a stopped node does not block readiness.
- With all *running* nodes ready, the `Kafka` CR is `Ready=True`; the separate `NodeStopped` condition (below) carries the "one node intentionally down" signal and is **not** folded into `Ready`.
- On **start** (ID removed from `stoppedNodes`), the node re-enters `runningNodes()`; the next reconcile regenerates its pod and `podsReady()` now waits for it, so the reconcile does not report success until the restarted broker is actually ready.

### Safety

Stopping a broker takes its replicas offline (they are retained on disk but not served), so partitions it leads/follows lose a replica until it returns.

- **Warn-and-allow on availability (and why this differs from the 049 scale-down check).** If stopping a broker would push any partition below `min.insync.replicas`, the operator still honors the stop (it is an explicit operator action) but raises a loud warning event and a `.status` condition naming the affected partitions; partitions whose only in-sync replica is on the stopped node become unavailable for `acks=all` writes (surfaced via metrics). This is deliberately laxer than proposal [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md), which *refuses* scale-down by default — because the impacts differ in kind: scale-down **permanently removes** the broker and risks data loss, whereas a stop **retains the data** and the unavailability is temporary and self-healing on restart. The bar to block is therefore lower. A refuse-by-default mode is offered as opt-in future work for operators who want scale-down-equivalent strictness.
- **Data is retained, not lost.** PVCs are kept; on start the broker recovers its logs and rejoins the ISR. This is the key difference from scale-down.
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
- **Observability caveats** to document so they are not mistaken for faults: a stopped broker's per-broker `Service` has empty `EndpointSlice`s (so `kafka-exporter` probes and external clients see connection-refused for it), and the stopped node is omitted from `Kafka.status.listeners[].addresses`. The `NodeStopped` condition and the metrics above make the intentional cause discoverable.

## Testing strategy

- **Unit tests.** A node listed in `stoppedNodes` produces no pod entry in the generated `StrimziPodSet`, but stays in `nodes()`/`desired`/`status.nodeIds` and is absent from `runningNodes()`. The PVC, cert Secret, and per-broker `ConfigMap` are still generated. `podsReady()` operates on `runningNodes()`, so with one node stopped and the rest ready the reconcile **succeeds** and the `Kafka` CR is `Ready=True` (the core regression test — without `runningNodes()` it would time out and wedge `NotReady`). PDB is computed excluding stopped nodes. The 049 check sees no removal and 108 does not unregister the node. A controller-targeting stop is refused with `StopRefused` *before* the pod is omitted. Parse-failure: invalid `stoppedNodes` entries ignored, valid ones applied, warning surfaced. Availability: stopping a broker that would breach min-ISR is honored with a warning condition.
- **System tests.** Stopping a broker removes its pod and keeps it down across reconciles while the `Kafka` CR stays `Ready=True`; the operator keeps rolling/reconciling other brokers; voluntary disruption of another broker is not blocked by the stopped one (PDB). Starting it (removing the ID) recreates the pod with its PVC, the reconcile waits for it, and the broker rejoins the ISR. Controller stop is refused. With the gate off, `stoppedNodes` is accepted by the CRD but inert (no `StopRefused`, no pod omission).

## Affected/not affected projects

Affected: `strimzi/strimzi-kafka-operator` — `KafkaCluster.generatePodSets` (omit stopped nodes' pods) plus a new `runningNodes()` accessor; `KafkaReconciler.podsReady()` and the `KafkaRoller` node set (use `runningNodes()`); `KafkaPool`/`NodeIdAssignor` (retain stopped IDs as desired); `StrimziPodSetController` (tolerate the intentional pod absence); PDB generation (exclude stopped nodes); the `KafkaNodePool` CRD (new `spec.stoppedNodes` field); `.status` conditions; metrics; and the `StopKafkaNode` feature gate. No change needed for PVC retention, the 049 scale-down check, 108 unregistration, or Cruise Control (all tolerate a retained-but-pod-less node by construction).

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
