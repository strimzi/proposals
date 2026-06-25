# Skip automatic rolling updates for a Kafka node

This proposal adds a way to exclude one or more broker nodes from the operator's *automatic* rolling updates, while the rest of the cluster keeps reconciling normally.
It is opt-in, set by a single annotation on the `KafkaNodePool` resource.

"Automatic" rolling means a roll the operator decides on its own: config change, certificate renewal, version upgrade, or the not-ready / force-restart path.
It is distinct from an *explicit* roll a human triggers with `strimzi.io/manual-rolling-update`, which is deliberately **not** suppressed (see "Interaction with manual rolling update").

## Current situation

Strimzi can pause reconciliation at exactly one granularity: the whole `Kafka` custom resource.
Setting `strimzi.io/pause-reconciliation="true"` on the `Kafka` CR freezes everything — config, certificate rotation, scaling, and rolling — for every node in the cluster.

There is no built-in way to say "leave this one node alone, keep managing the others".
The existing per-node controls do the opposite: `strimzi.io/manual-rolling-update` *forces* a roll, and pausing a `KafkaNodePool` does not stop rolling because `KafkaRoller` runs from the `Kafka` CR reconcile, not the node-pool reconcile.

## Motivation

The cluster-wide pause is the wrong tool for a single-node problem.
Sometimes one node must be taken out of automatic rolling while the operator keeps managing the rest — for example a healthy node held for live investigation, a known-bad host awaiting a disk/hardware swap before it fails, or a node mid-rebalance a roll would interrupt.
In these cases the node is still up and in-sync, so every other node keeps rolling normally; the cluster-wide pause is far too coarse, freezing config, cert rotation, scaling, and rolling for every healthy node too.

The feature began with a sharper incident: a broker stuck in a long on-disk log recovery, repeatedly force-restarted by the roller (`due to []` — no pending change), discarding in-progress recovery each time.
That recovery case should already be handled by proposal [048](https://github.com/strimzi/proposals/blob/main/048-avoid-broker-restarts-when-in-recovery.md): the roller reads the Kafka Agent broker state and, on `RECOVERY`, stops instead of force-restarting.
It looped only because the agent hung and returned `BrokerState(-1)` instead of `RECOVERY` ([#12513](https://github.com/strimzi/strimzi-kafka-operator/issues/12513), fix in [#12675](https://github.com/strimzi/strimzi-kafka-operator/pull/12675)), so 048's check fell through.
That part is a bug and the #12513 fix should land independently of this proposal.

A human lever is still needed for the cases 048 cannot see: a node not-ready for a non-`RECOVERY` reason (a degraded-but-alive disk, a hung mount) where the agent reports nothing actionable and a force-restart only reschedules the pod into a crash loop (`Multi-Attach` on `ReadWriteOnce` storage).
The operator cannot auto-distinguish "stuck, restart me" from "intentionally not-ready, leave me" — only a human can — so the skip annotation is that deterministic override.

The skip does **not** promise that the rest of the cluster keeps rolling freely.
When a skipped broker is actually down it has already dropped out of its partitions' ISR, so a roll of another broker sharing an at-risk partition is deferred by the existing min-ISR check, exactly as any unsafe roll is today.
The honest win over `pause-reconciliation`: all *non-rolling* reconciliation (scaling, PVCs, config/cert generation, status) continues cluster-wide and rolls of unaffected nodes proceed; only the flagged node and genuinely-unsafe rolls are held back.

## Proposal

Add a single annotation on the `KafkaNodePool` resource listing the node IDs in that pool to exclude from automatic rolling:

```
strimzi.io/skip-rolling-update="[2,5]"
```

The value reuses the node-ID format of the existing `strimzi.io/next-node-ids` / `strimzi.io/remove-node-ids` annotations (IDs and ranges, e.g. `[2,5]` or `[2,4-6]`).
It lives on the `KafkaNodePool` rather than the `Kafka` CR because node IDs are owned by the pool; it is set and removed by the user, so it survives pod recreation and GitOps re-apply.

A node ID that does not belong to the pool, is a duplicate, or is negative is ignored: the operator applies the remaining valid IDs, logs a warning, and surfaces the rejected value in the status condition.
An empty or absent annotation means nothing is skipped.

### Mechanism

A single predicate `isSkipped(nodeId)` is consulted at every gate that waits on node readiness; missing one would hang the reconcile, so the set is explicit:

- the `KafkaRoller` working set — the node is never a roll candidate, so it never reaches the force-restart paths (decided inside the roller after `POD_UNRESPONSIVE` or in the `catch (ForceableProblem)` branch, which produce no `RestartReasons` — a reason filter would never catch them);
- `KafkaReconciler.podsReady()` and the endpoint-readiness stages, which otherwise wait on the unfiltered `kafka.nodes()` and would time out every cycle.

With those excluded the reconcile completes and the `Kafka` `Ready` condition reflects only the non-skipped nodes; the skipped node's real state stays visible via kubelet probes (still `0/1`) and the status condition.

Two things the skip must **not** do:

- It must not make the node look removed: membership stages (KRaft register/unregister, scale-down) keep deriving membership from `kafka.nodes()`, which still includes the skipped node.
- It must not advance cluster-wide state past the held node: a cluster-wide Kafka version / `metadata.version` change is deferred while any node is skipped, since the skipped node stays on its old version and finalizing the upgrade without it could leave it unable to rejoin.

### Controller nodes

The first version handles broker nodes only.
A node ID that resolves to a controller (controller-only or combined broker+controller) is ignored and kept managed, surfaced in status and logs.
Quorum-safe controller skipping is future work, because a skipped controller can thin the KRaft metadata quorum; in a fully combined-mode cluster every node is a controller, so the skip is inert there until then.

### Interaction with manual rolling update

Skip suppresses only *automatic* rolls; an explicit `manual-rolling-update` is a direct human instruction and still rolls the node — a fresh human action outranks a standing skip — so no precedence logic is added.

Drain Cleaner is the awkward case: it is automated (so by that rule it *should* honor the skip) but it triggers rolls through the manual-rolling-update path, which the first version cannot cleanly intercept.
So a drain can still move a skipped node; while a node is skipped, exclude it from Drain Cleaner and cordon its host.
A first-class Drain Cleaner integration that honors the skip is the named follow-up.

### What a skip does not stop

A skip suppresses automatic rolling restarts only; non-rolling reconciliation (Services, PVCs, ConfigMaps/Secrets, the `StrimziPodSet` pod template) continues, just not applied to the running pod until it next restarts. It also does not stop:

- the `StrimziPodSet` controller recreating a deleted/evicted pod (skipped again next reconcile);
- kubelet probes, or host-level eviction/drain/scheduling;
- an explicit `manual-rolling-update` (above);
- scale-down — `KafkaReconciler.scaleDown()` derives the pod set from `kafka.nodes()` outside `KafkaRoller`, so lowering `replicas` or `strimzi.io/remove-node-ids` still removes the pod (the [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md) safety check still refuses removing a broker that holds replicas).

Because rolling is suppressed, a long-lived skip holds the node back from config, version, and certificate changes — and across a cluster CA replacement the node keeps its old trust set and may fail to rejoin when later restarted.
A skip is therefore meant to be short-lived and human-supervised, and must not be held across an upgrade or CA renewal.

### Status, security, and observability

A `.status` condition on the `Kafka` CR (paralleling `ReconciliationPaused`) lists skipped node IDs with their start time, plus ignored controller IDs and rejected invalid IDs; per-node detail is mirrored on the owning `KafkaNodePool`.
Because `Ready` excludes skipped nodes, the condition also flags when a skipped node holds the sole in-sync replica of any partition, so an intentionally-skipped node cannot make the cluster look healthy while a partition is actually offline.
Entering/leaving the skipped state is logged and emitted as an event.

Disabling a node's self-healing requires edit access to its `KafkaNodePool` — a tighter, more auditable surface than pod-edit.
No dedicated Prometheus metrics are added in the first version; the condition and annotation are visible to Kube State Metrics if alerting is needed.

## Testing strategy

- A skipped NotReady broker is excluded from both the roller working set and `podsReady()`, so it is never force-restarted *and* the reconcile completes with `Ready=True` plus a `NodeRollingUpdateSkipped` condition not folded into `Ready` (the core regression — without the `podsReady()` exclusion the reconcile times out every cycle).
- No global deadlock: a broker sharing no at-risk partition still rolls, while a roll that would breach min-ISR is deferred by the existing availability check and proceeds once the skip is lifted.
- A cluster-wide version / `metadata.version` change is deferred while a node is skipped; a controller-targeting skip is ignored and still managed; an explicit `manual-rolling-update` on a skipped node still rolls it.
- Downgrade to an operator without the feature reverts to legacy behavior; the documented data-loss risk on a skipped node is exercised and surfaced.

## Compatibility

The feature is opt-in: an absent annotation means the operator behaves exactly as today, including the 048 recovery logic, and the annotation is ignored by older tooling.

Downgrading while a node is skipped is a data-loss risk, not cosmetic: an operator without the feature treats the skipped, recovering node as an ordinary not-ready pod and force-restarts it, discarding in-progress recovery.
Operators must lift all skips and confirm the nodes are healthy before downgrading, and the downgrade docs must call this out.

## Rejected alternatives

- **Whole-`Kafka`-CR pause (`strimzi.io/pause-reconciliation`).** Too coarse: to protect one node it freezes config, cert rotation, scaling, and rolling for the entire cluster.
- **Stop force-restarting not-ready pods that have no pending `RestartReasons`.** Tempting (it would stop the recovery loop too), but the operator cannot tell a genuinely stuck pod that needs a restart from one that is intentionally not-ready; removing the self-healing for everyone is worse than an opt-in per-node lever.
- **A pod annotation.** Lost when the pod is recreated from the `StrimziPodSet` template; the durable surface must be the user-managed `KafkaNodePool`.
- **A new `KafkaNodePool.spec` field.** Heavier than needed for a short-lived operational override; an annotation in the existing node-ID family is sufficient and consistent.
- **A feature gate.** Opt-in by annotation and no change to default behavior, so a gate is not warranted.
- **Raising `STRIMZI_OPERATION_TIMEOUT_MS`.** Operator-wide and blunt; slows every legitimate roll and still loops on any not-ready state longer than the new threshold.
- **Making the readiness probe pass during recovery.** Lies to clients and load balancers and masks genuine not-ready states; the fix belongs in what the operator does with the state, not in falsifying it.
- **Relying solely on the #12513 / 048 fix.** That fixes the recovery heuristic but not the non-recovery, agent-unreachable, or deliberate-hold cases; the two compose.

## Future work

- **Controller skip support**: a quorum-safe admission check (refuse a skip that would drop the KRaft quorum below majority), an explicit override flag, per-reconcile re-evaluation, and deciding other-controller rolls from real `DescribeQuorum` state.
- **First-class Drain Cleaner integration** so a skipped node is automatically excluded from drain-driven evictions.
- **Auto-expiry of a skip** after a configurable duration, so a forgotten skip does not block cert rotation, config, or upgrades indefinitely.
- **Availability-math refinement** to subtract skipped brokers from the effective ISR when judging whether other nodes can roll.
- **Dedicated metrics** if the status condition proves insufficient for alerting.
