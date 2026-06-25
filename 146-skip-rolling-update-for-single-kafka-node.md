# Skip operator-driven rolling for a Kafka node

This proposal adds a way to exclude one or more broker nodes from the operator's *automatic* rolling updates, while the rest of the cluster keeps reconciling normally.
It is opt-in, set by a single annotation on the `KafkaNodePool` resource.

Throughout this proposal, "automatic" rolling means a roll the operator decides on its own (config change, certificate renewal, version upgrade, or the not-ready / force-restart path).
It is distinct from an *explicit* roll a human triggers with `strimzi.io/manual-rolling-update`, which is an intentional instruction and is deliberately **not** suppressed by this feature (see "Interaction with manual rolling update").

## Current situation

Strimzi can pause reconciliation at exactly one granularity: the whole `Kafka` custom resource.
Setting `strimzi.io/pause-reconciliation="true"` on the `Kafka` CR freezes everything the operator does — config changes, certificate rotation, scaling, and rolling — for every node in the cluster.

There is no built-in way to say "leave this one node alone, keep managing the others".
The existing per-node controls do the opposite: `strimzi.io/manual-rolling-update` on a pod or `StrimziPodSet` *forces* a roll, and pausing a `KafkaNodePool` does not stop rolling because `KafkaRoller` is driven from the `Kafka` CR reconcile, not the node-pool reconcile.

## Motivation

The cluster-wide pause is the wrong tool for the common case, which is almost always a single misbehaving node.
An operator sometimes needs to take one node out of automatic rolling — while the rest of the cluster keeps being managed — and today the only way to express that is the cluster-wide pause, which also freezes config, cert rotation, scaling, and rolling for every healthy node.

The motivating incident was a broker stuck in a long on-disk log recovery, repeatedly force-restarted by the roller, which discarded in-progress recovery each time.
That specific recovery case should already be handled by proposal [048](https://github.com/strimzi/proposals/blob/main/048-avoid-broker-restarts-when-in-recovery.md): the roller queries the Kafka Agent broker-state endpoint and, on `RECOVERY`, fails the reconcile instead of force-restarting.
In the incident it still looped because the agent hung and returned `BrokerState(-1, null)` instead of `RECOVERY` ([strimzi-kafka-operator#12513](https://github.com/strimzi/strimzi-kafka-operator/issues/12513), fix in flight in [#12675](https://github.com/strimzi/strimzi-kafka-operator/pull/12675)), so `-1 != 2` and the roller fell through to force-restart.
That part is a bug and the #12513 timeout fix should land independently of this proposal.

This proposal does not replace that fix; it covers the cases the recovery heuristic cannot.
The heuristic only acts when the agent reports a clean `RECOVERY` state, so it does nothing when a node is not-ready for another reason.
A concrete example that survives the #12513 fix: a broker that fails its readiness probe because of a degraded-but-alive disk or a slow/hung mount, with the agent reporting `RUNNING` (or nothing actionable).
The roller still force-restarts it, and on `ReadWriteOnce` storage the restart reschedules the pod to another host and hits `Multi-Attach`, turning a degraded broker into a crash loop.
There is no automatic signal that tells the operator to stop; only a human knows "this disk is dying, do not roll this node until I swap it".
The skip annotation is that deterministic human override.

What the skip does **not** promise is "the rest of the cluster keeps rolling freely".
When the skipped broker is genuinely down it has dropped out of the ISR of the partitions it hosted, so a roll of another broker that shares one of those partitions is deferred by the existing min-ISR safety check — exactly as any unsafe roll is deferred today.
The durable, honest win over `pause-reconciliation` is therefore narrower but real: all *non-rolling* reconciliation (scaling, PVCs, config and certificate generation, status) continues across the whole cluster, and rolls of nodes that share no at-risk partition still proceed; only the flagged node, and rolls that would breach availability, are held back.

## Proposal

Add a single annotation on the `KafkaNodePool` resource listing the node IDs in that pool to exclude from automatic rolling:

```
strimzi.io/skip-rolling-update="[2,5]"
```

The value reuses the same node-ID format as the existing `strimzi.io/next-node-ids` and `strimzi.io/remove-node-ids` annotations (individual IDs and ranges, e.g. `[2,5]` or `[2,4-6]`).
A node is skipped if its ID appears in the list on its owning `KafkaNodePool`.
The annotation lives on the `KafkaNodePool` rather than the `Kafka` CR because node IDs are owned by the pool, so the skip sits on the resource that owns the node; it is set and removed by the user, so it survives pod recreation and GitOps re-apply.

A node ID that does not belong to the pool, is a duplicate, or is negative is ignored; the operator applies the remaining valid IDs, logs a warning, and surfaces the rejected value in the status condition below so it is visible without reading operator logs.
An empty or absent annotation means nothing is skipped.

### Mechanism

The skip removes the node from the set `KafkaRoller` iterates, so it is never a roll candidate and the roller never waits on its readiness.
This is working-set exclusion at set-construction time, not a filter on `RestartReasons`: the destructive not-ready / force-restart path is decided inside `KafkaRoller` (after `POD_UNRESPONSIVE`, or in the `catch (ForceableProblem)` branch) and produces no restart reasons, so a reason filter would never see it.

Excluding the node from the roller is necessary but not sufficient.
`KafkaReconciler.reconcile()` runs a separate `podsReady()` stage after rolling that waits on the unfiltered `kafka.nodes()` set, so a skipped-but-NotReady node would still time out the reconcile every cycle.
The skip therefore also excludes the node from `podsReady()` (and the endpoint-readiness stages), so the reconcile completes and the post-readiness steps still run.

The `Kafka` CR `Ready` condition then reflects only the non-skipped nodes; the skipped node's real state stays visible on the pod (kubelet probes are untouched) and in the status condition below.

Completing the reconcile with a node deliberately down must not let the operator treat that node as removed: the skip suppresses rolling only, so the post-readiness stages that act on cluster membership (KRaft node registration/unregistration, scale-down) must continue to derive membership from `kafka.nodes()`, which still includes the skipped node.
The implementation must verify that the post-readiness stages which talk to the cluster (cluster ID, metadata version, quotas/ACLs) tolerate a listed-but-down broker — they already operate against the cluster as a whole rather than the skipped pod, but this is called out as a thing to confirm in testing.

### Controller nodes

The first version handles broker nodes only.
A node ID that resolves to a controller node (controller-only or combined broker+controller) is ignored, the operator keeps managing it normally, and the situation is surfaced in the status and logs.
Quorum-safe skipping of controllers is future work, because a skipped controller can thin the KRaft metadata quorum.
In a fully combined-mode cluster every node is a controller, so the skip is inert there until the controller work lands.

### Interaction with manual rolling update

`strimzi.io/manual-rolling-update` and `strimzi.io/skip-rolling-update` are independent: skip suppresses *automatic* rolls, while a manual rolling update is an explicit human instruction and still rolls the node.
This is intentional — a human forcing a roll is a stronger, more recent signal than a standing skip — and it avoids adding precedence logic for a combination that should not normally be set at once.
The consequence to document clearly: a manual rolling update overrides a skip on the same node.

### Drain Cleaner interaction

Strimzi Drain Cleaner drives graceful eviction through the manual-rolling-update path, so by the rule above a drain can still roll/evict a skipped node and trigger a fresh recovery.
This is the one interaction that can silently undo a skip, so it must be handled explicitly: while a node is skipped, exclude it from Drain Cleaner and cordon its host so eviction tooling does not move it.
A first-class integration where Drain Cleaner itself honors the skip and refuses to evict a skipped node is future work.

### What a skip does and does not affect

A skip suppresses the operator's automatic rolling restarts of the node, and only that.
Non-rolling reconciliation continues as before: Services, PVCs, ConfigMaps and Secrets, and the pod template in the `StrimziPodSet` are still updated; the change is simply not applied to the running pod until it is next restarted by something else or unskipped.
Reconciliation of all other nodes is unaffected.

A skip does not stop:
- the `StrimziPodSet` controller recreating the pod if it is deleted or evicted (the recreated pod is skipped again on the next reconcile);
- kubelet liveness/readiness probes;
- Kubernetes host-level eviction, drain, or scheduling;
- an explicit `manual-rolling-update` (see above);
- operator-driven scale-down — `KafkaReconciler.scaleDown()` derives the pod set from `kafka.nodes()` outside `KafkaRoller`, so lowering `replicas` or using `strimzi.io/remove-node-ids` still removes the pod (the existing scale-down safety check in [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md) already refuses removing a broker that holds replicas).

Because rolling is suppressed, a long-lived skip can hold a node back from a config change, a version upgrade, or certificate renewal — and across a cluster CA replacement the skipped node keeps its old trust set and may fail to rejoin when later restarted.
A skip is therefore meant to be short-lived and human-supervised, and must not be held across an upgrade or CA renewal.

### Security

The trigger is a `KafkaNodePool` annotation, so the privilege required to disable a node's self-healing is edit access to that CR — typically far more restricted, and more auditable, than pod-edit.
This is deliberately a tighter surface than a pod annotation would be.
Every enter/leave of the skipped state is recorded via the status condition and an event for audit.

### Status reporting

The skipped state is surfaced on the `Kafka` CR via a `.status` condition listing the skipped node IDs and the time each entered the state, paralleling the existing `ReconciliationPaused` condition; ignored controller IDs and rejected invalid IDs are noted there too.
Per-node detail is mirrored on the owning `KafkaNodePool` `.status`, since that is where the annotation is set.
The operator logs and emits an event when a node enters or leaves the skipped state.
No dedicated Prometheus metrics are added in the first version — the annotation and the status condition are already visible to Kube State Metrics if alerting is needed.

## Testing strategy

- **Unit (`KafkaRoller` / `KafkaReconciler`).** A skipped NotReady broker is excluded from both the roller working set and the `podsReady()` stage, so it is never force-restarted *and* the whole reconcile completes with `Kafka` `Ready=True` and a `NodeRollingUpdateSkipped` condition that is not folded into `Ready` (this is the core regression — without the `podsReady()` exclusion the reconcile times out every cycle). A skipped broker with a pending config/cert reason is not rolled.
- **Availability.** A roll of another broker that would breach min-ISR given a skipped (down) broker is deferred via the existing availability check and proceeds once the skip is lifted; a broker sharing no at-risk partition still rolls and the reconcile completes the safe subset (confirming no global deadlock and no availability-math change needed).
- **Controllers.** A controller-targeting skip (controller-only and combined) is ignored, surfaced in status, and still managed.
- **Manual rolling update.** An explicit `manual-rolling-update` on a skipped node still rolls it.
- **System tests.** A NotReady broker is not force-restarted while skipped and the operator keeps rolling unaffected brokers; post-readiness stages complete with a listed-but-down broker.
- **Downgrade.** The documented data-loss risk on a skipped node when downgrading to an operator without this feature is exercised and surfaced.

## Compatibility

The feature is opt-in: an absent annotation means the operator behaves exactly as today, including the proposal-048 recovery logic.
The annotation is additive and is ignored by older tooling.

Downgrading the operator while a node is skipped is a data-loss risk, not a cosmetic one.
An operator without this feature ignores the annotation and treats a skipped, recovering node as an ordinary not-ready pod, force-restarting it and discarding in-progress recovery.
Operators must lift all skips and confirm the nodes are healthy before downgrading, and the downgrade docs must call this out.

## Rejected alternatives

- **Whole-`Kafka`-CR pause (`strimzi.io/pause-reconciliation`).** Too coarse: to protect one node it freezes config, cert rotation, scaling, and rolling for the entire cluster.
- **A pod annotation.** Pods are recreated from a `StrimziPodSet` template, so a pod annotation is lost on recreation; the annotation belongs on the user-managed `KafkaNodePool`.
- **A new `KafkaNodePool.spec` field.** Heavier than needed for a short-lived operational override; an annotation in the existing node-ID family is sufficient and consistent.
- **A feature gate.** The behavior is opt-in by annotation and does not change default behavior, so a gate is not warranted.
- **Raising `STRIMZI_OPERATION_TIMEOUT_MS`.** Operator-wide and blunt; slows every legitimate roll and still loops on any not-ready state longer than the new threshold.
- **Making the readiness probe pass during recovery.** Lies to clients and load balancers and masks genuine not-ready states; the fix belongs in what the operator does with the state, not in falsifying it.
- **Relying solely on the #12513 / 048 fix.** That fixes the recovery heuristic but not the non-recovery, agent-unreachable, or deliberate-hold cases; the two compose.

## Future work

- **Controller skip support**, with a quorum-safe admission check (refuse a skip that would drop the KRaft quorum below majority), an explicit override flag, per-reconcile re-evaluation, and deciding other-controller rolls from real `DescribeQuorum` state.
- **First-class Drain Cleaner integration** so a skipped node is automatically excluded from drain-driven evictions.
- **Auto-expiry of a skip** after a configurable duration, so a forgotten skip does not block cert rotation, config, or upgrades indefinitely.
- **Availability-math refinement** to subtract skipped brokers from the effective ISR when judging whether other nodes can roll.
- **Dedicated metrics** if the status condition proves insufficient for alerting.
