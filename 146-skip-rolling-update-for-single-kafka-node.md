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
Sometimes one node must be taken out of automatic rolling while the operator keeps managing the rest — for example a known-bad host awaiting a hardware swap, a node held for live investigation, or the tail of a human-gated staged rollout.

The feature began with a sharper incident: a broker in a long on-disk log recovery was repeatedly force-restarted by the roller, discarding the in-progress recovery each time.
That turned out to be an agent bug ([#12513](https://github.com/strimzi/strimzi-kafka-operator/issues/12513)) that defeated the recovery check from proposal [048](https://github.com/strimzi/proposals/blob/main/048-avoid-broker-restarts-when-in-recovery.md), and it has been fixed ([#12675](https://github.com/strimzi/strimzi-kafka-operator/pull/12675)).

A human lever is still needed for the cases 048 cannot see, where the operator cannot auto-distinguish "stuck, restart me" from "intentionally not-ready, leave me" — only a human can, and the skip annotation is that deterministic override.

The concrete scenario, from production experience operating these clusters: a broker sits on a host with a failing-but-alive disk, and the replacement depends on datacenter operations — a wait measured in days, not hours.
During that window the broker is NotReady, so every reconciliation ends in the roller's unresponsive-node handling force-restarting it, and on `ReadWriteOnce` storage the recreated pod can land in a `Multi-Attach` crash loop against the dying volume — strictly worse than leaving the pod alone.
The only lever today is `pause-reconciliation`, and holding it for days means no certificate rotation, no config changes, and no scaling for every healthy node in the cluster; it is safe only for short windows, which this intervention is not.
A skip is scoped to one node and can be held for the days the intervention takes: all other reconciliation continues cluster-wide and rolls of unaffected nodes proceed (the bounds on how long a skip may be held are events, not time — see "What a skip does not stop").

The disk swap is one member of a class: situations where the right action for one node is decided by information the operator cannot see.
Others from the same operations experience:

- A planned on-host storage operation on a healthy node (an LVM resize, a volume migration), where a restart landing mid-operation can abort or corrupt the work.
- Live forensics on a misbehaving broker (heap dumps, JFR recordings, inspection of on-disk state), where a roll triggered by an unrelated certificate renewal or config change destroys the evidence being captured.
- A human-gated staged rollout: skip some nodes, let the others take a risky config change, verify broker health, then unskip. `pause-reconciliation` cannot express this — pausing stops the rollout for every node at once, while the point here is to let it proceed partially.

This kind of per-node hold is an established control for stateful workloads on Kubernetes: StatefulSets expose an ordinal-based version of it as `spec.updateStrategy.rollingUpdate.partition`, bounding which pods a rollout may touch.
StrimziPodSets have no equivalent; this annotation supplies one.

The skip does **not** promise that the rest of the cluster keeps rolling freely: when a skipped broker is actually down, a roll of another broker sharing an at-risk partition is deferred by the existing min-ISR check, exactly as any unsafe roll is today.

## Proposal

Add a single annotation on the `KafkaNodePool` resource listing the node IDs in that pool to exclude from automatic rolling:

```
strimzi.io/skip-rolling-update="[2,5]"
```

The value reuses the node-ID format of the existing `strimzi.io/next-node-ids` / `strimzi.io/remove-node-ids` annotations (IDs and ranges, e.g. `[2,5]` or `[2,4-6]`).
It lives on the `KafkaNodePool` rather than the `Kafka` CR because node IDs are owned by the pool; it is set and removed by the user, so it survives pod recreation and GitOps re-apply.

A node ID that does not belong to the pool or is invalid (negative or malformed) is ignored: the operator applies the remaining valid IDs, logs a warning, and surfaces the rejected value in the status condition.
An empty or absent annotation means nothing is skipped.

Skipping several nodes at once is allowed but carries the same risk as taking those nodes down manually: if the skipped nodes share partition replicas, their combined absence can degrade or disable those partitions.
The operator does not validate partition placement at annotation-admission time; the existing availability check still defers any other roll that would breach `min.insync.replicas`, and Kafka's own metrics (`UnderMinIsrPartitionCount`, `OfflinePartitionsCount`) remain the authoritative signal for partitions put at risk.

### Mechanism

The whole change lives in `KafkaReconciler`; `KafkaRoller` itself is not modified.
At the start of each reconciliation, `KafkaReconciler` resolves the `skip-rolling-update` annotations across the cluster's `KafkaNodePool` resources into a set of skipped node IDs, validating pool membership and node roles at that point.
The set is applied at two places in the `reconcile()` pipeline:

- `rollingUpdate()` — today it passes the unfiltered `kafka.nodes()` to `maybeRollKafka()` and on to `KafkaRoller`; the skipped IDs are removed from that set, so the roller never considers the node and none of its internal not-ready / force-restart paths can fire for it.
  (A `RestartReasons` filter would not work: the roller's force-restart decisions are made internally and produce no predicate-visible reason.)
- `podsReady()` — it runs right after `rollingUpdate()` and waits up to the operation timeout for every pod in `kafka.nodes()` to become Ready; the skipped node is removed from that list, otherwise a NotReady skipped pod would time this stage out on every reconciliation even though the roller ignored it.

The later `serviceEndpointsReady()` and `headlessServiceEndpointsReady()` stages need no filtering: `Endpoints` readiness requires only one ready address, and the headless brokers service publishes not-ready addresses, so a single NotReady node does not block them.
The `manualRollingUpdate()` step is deliberately **not** filtered (see "Interaction with manual rolling update").

With those exclusions the reconciliation completes and the `Kafka` `Ready` condition reflects only the non-skipped nodes; the skipped node's real state stays visible via kubelet probes (still `0/1`), with the skip itself recorded in the status condition.

Two things the skip must **not** do:

- It must not make the node look removed: membership stages (KRaft register/unregister, scale-down) keep deriving membership from `kafka.nodes()`, which still includes the skipped node.
- It must not advance cluster-wide state past the held node: a cluster-wide Kafka version / `metadata.version` change is deferred while any node is skipped, since the skipped node stays on its old version and finalizing the upgrade without it could leave it unable to rejoin.
  Only the upgrade rollout is held: all other reconciliation, including rolls of other nodes for other reasons, continues, and the deferral is logged and surfaced in the status condition.

### Controller nodes

Skipping applies to broker-only nodes; controller support is a non-goal, because skipping a controller thins the KRaft metadata quorum majority and controller log recovery is fast enough that the motivating scenarios barely apply.
A node ID that resolves to a controller — controller-only or combined, so in a fully combined-mode cluster the skip is inert — is ignored and kept managed, surfaced in status and logs.

The role check happens at annotation-resolution time in `KafkaReconciler`, keyed on the `NodeRef` desired roles, because that is where the skip set is built and it works even when the pod does not exist.
Desired and actual roles can diverge during a role transition (`KafkaRoller` reads the pod's role labels as the actual roles), so the check is conservative: the skip is honored only if the node is broker-only by desired role *and*, when the pod exists, its role labels do not claim the controller role; otherwise the ID is ignored and the reason logged.
Rolling one node too many during a transition is recoverable; silently thinning the quorum is not.

### Interaction with manual rolling update

Skip suppresses only *automatic* rolls; an explicit `manual-rolling-update` is a direct human instruction and still rolls the node — a fresh human action outranks a standing skip — so no precedence logic is added.

Drain Cleaner is the awkward case: it is automated (so by that rule it *should* honor the skip) but it triggers rolls through the manual-rolling-update path, which the initial implementation of this feature cannot cleanly intercept.
So a drain can still move a skipped node; while a node is skipped, exclude it from Drain Cleaner and cordon its host.
A first-class Drain Cleaner integration that honors the skip can be added as a follow-up.

### What a skip does not stop

A skip suppresses automatic rolling restarts only; non-rolling reconciliation (Services, PVCs, ConfigMaps/Secrets, the `StrimziPodSet` pod template) continues, just not applied to the running pod until it next restarts.
A skip also does not stop:

- the `StrimziPodSet` controller recreating a deleted/evicted pod (skipped again next reconcile);
- kubelet probes, or host-level eviction/drain/scheduling;
- an explicit `manual-rolling-update` (above);
- scale-down — `KafkaReconciler.scaleDown()` derives the pod set from `kafka.nodes()` outside `KafkaRoller`, so lowering `replicas` or `strimzi.io/remove-node-ids` still removes the pod (the [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md) safety check still refuses removing a broker that holds replicas).

Because rolling is suppressed, a skip holds the node back from config, version, and certificate changes — and across a cluster CA replacement the node keeps its old trust set and may fail to rejoin when later restarted.
A skip is human-supervised and lasts as long as the manual intervention it protects — typically hours to days (for example, waiting on a datacenter hardware swap); there is no hard time limit.
The safety boundary is not wall-clock time but events: a skip must not be held across a Kafka version upgrade or a CA renewal.
The design enforces the first — a version rollout is not initiated while any node is skipped (see "Mechanism") — and warns about the rest: while a skip is active the operator logs a warning on every reconciliation that the node is not receiving config, version, or certificate changes.
No per-cause conflict detection is needed, because a skipped node is by definition behind once any change lands; the standing warning carries the same information.
If that visibility proves insufficient, automatic expiry of a skip after a configurable duration can be added as a follow-up.

### Status, security, and observability

A `.status` condition of type `RollingUpdateSkipped` on the `Kafka` CR (paralleling `ReconciliationPaused`) lists the skipped node IDs, plus ignored controller IDs and rejected invalid IDs; its `lastTransitionTime` reflects the most recent change to that set.
The condition deliberately does not report partition health: Kafka already exports the authoritative signals (`UnderMinIsrPartitionCount`, `UnderReplicatedPartitions`, `OfflinePartitionsCount`), and recomputing them in the operator would require cluster-wide topic describes correlated with per-topic `min.insync.replicas`, re-evaluated while the skip is held, with ISR fluctuation churning the `Kafka` CR status — significant cost for information existing alerting already covers.
Entering and leaving the skipped state is logged, and a Kubernetes `Event` is emitted through the operator's existing event publisher (the same mechanism used for restart events), with reasons `RollingUpdateSkipEnabled` and `RollingUpdateSkipDisabled`.

A skip disables part of a node's self-healing, so who can set it matters.
With the rejected pod-annotation design, anyone with pod-edit permission could do it silently; with this design it requires edit rights on the `KafkaNodePool` custom resource, which in most deployments is GitOps-managed and reviewed, so the action is gated by the same access controls as any other cluster-shape change and leaves an audit trail.

No dedicated Prometheus metrics are added: the status condition plus log warnings is the observability surface for this feature, and both the condition and the annotation are visible to Kube State Metrics for alerting.

## Testing strategy

The core regression to guard is the pair of exclusions working together: a unit/integration test at `KafkaReconciler` scope runs a reconciliation with one pod NotReady and skipped, and asserts that (a) no pod deletion or restart is issued for it, (b) the reconciliation completes instead of timing out in `podsReady()`, and (c) the `RollingUpdateSkipped` condition is present and the skipped node's NotReady state does not degrade `Ready`.

Further tests cover:

- No global deadlock: a broker sharing no at-risk partition still rolls, while a roll that would breach min-ISR is deferred by the existing availability check and proceeds once the skip is lifted.
- A cluster-wide version / `metadata.version` change is deferred while a node is skipped; a controller-targeting skip is ignored and still managed; an explicit `manual-rolling-update` on a skipped node still rolls it.
- A system test with the annotation on a `KafkaNodePool` and a genuinely NotReady broker, verifying end-to-end that the node is left alone while the rest of the cluster reconciles.
- Downgrade to an operator without the feature reverts to legacy behavior; the documented data-loss risk on a skipped node is exercised and surfaced.

## Affected/not affected projects

Affected: the Cluster Operator in `strimzi-kafka-operator` (the `KafkaReconciler` changes described above) and its documentation.
Not affected: `KafkaRoller` itself (unchanged), the Topic and User Operators, the Kafka Agent, Kafka Bridge, and all other Strimzi projects.
Drain Cleaner has no code change; its interaction with a skipped node is documented in "Interaction with manual rolling update".

## Compatibility

The feature is opt-in: an absent annotation means the operator behaves exactly as today, including the 048 recovery logic, and the annotation is ignored by older tooling.

Downgrading while a node is skipped is a data-loss risk, not cosmetic: an operator without the feature treats the skipped, recovering node as an ordinary not-ready pod and force-restarts it, discarding in-progress recovery.
Operators must lift all skips and confirm the nodes are healthy before downgrading, and the downgrade docs must call this out.

## Rejected alternatives

- **Whole-`Kafka`-CR pause (`strimzi.io/pause-reconciliation`).** Too coarse: to protect one node it freezes config, cert rotation, scaling, and rolling for the entire cluster, which makes it safe only for short windows — it cannot cover the days-long interventions in the motivation.
- **Stop force-restarting not-ready pods that have no pending `RestartReasons`.** Tempting (it would stop the recovery loop too), but the operator cannot tell a genuinely stuck pod that needs a restart from one that is intentionally not-ready; removing the self-healing for everyone is worse than an opt-in per-node lever.
- **A pod annotation.** Lost when the pod is recreated from the `StrimziPodSet` template; the durable surface must be the user-managed `KafkaNodePool`.
- **A new `KafkaNodePool.spec` field.** Heavier than needed for an operational override that is removed when the intervention ends; an annotation in the existing node-ID family is sufficient and consistent.
- **A feature gate.** Opt-in by annotation and no change to default behavior, so a gate is not warranted.
- **Relying solely on the [#12513](https://github.com/strimzi/strimzi-kafka-operator/issues/12513) / 048 fix.** That fixes the recovery heuristic but not the non-recovery, agent-unreachable, or deliberate-hold cases; the two compose.
- **Skipping controller nodes.** Thins the KRaft quorum majority for little benefit (controller recovery is fast); a controller-resolving ID is ignored and kept managed. This is a non-goal, not deferred work.
