# Skip operator-driven rolling for a single Kafka node

This proposal adds a way to exclude a single Kafka node from operator-driven rolling, while the rest of the cluster continues to reconcile normally. The design covers both brokers and controllers, but the first cut (v1) is **brokers only** — a skip targeting a controller is refused in v1, and controller support lands as a follow-up (see MVP scope). Today the only pause control is `strimzi.io/pause-reconciliation` on the whole `Kafka` custom resource, which is cluster-wide and far too coarse for the situations where you actually need it.

The mechanism is deliberately framed as "skip the rolling update for this node," not "pause everything about this node." It lives in the same family as the existing `strimzi.io/manual-rolling-update` lever — one forces a roll, the new one suppresses one.

### Terminology: "node" means a Kafka node, not a Kubernetes node

Throughout this proposal, **node** means a *Kafka node* in the Strimzi/KRaft sense: one `NodeRef(podName, nodeId, poolName, controller, broker)` — exactly **one `Pod`, one integer node ID**, with role `broker`, `controller`, or both (combined/dual-role). It is **not** a Kubernetes worker node (a host), which can schedule several Kafka `Pod`s. The skip is keyed at this granularity — the pod annotation sits on one Kafka node's `Pod`, the `spec` field lists Kafka **node IDs** — so skipping one Kafka node leaves any other pods on the same host fully managed.

## Current situation

Strimzi can pause reconciliation at exactly one granularity: the entire `Kafka` custom resource. Setting the annotation `strimzi.io/pause-reconciliation="true"` on the `Kafka` CR freezes everything the operator would otherwise do — config changes, certificate rotation, scaling, and rolling — for every broker and every controller in the cluster.

There is no built-in way to tell the operator "leave this one node alone, keep managing the others". The closest existing per-node controls do the opposite of what is needed here:

- `strimzi.io/manual-rolling-update` on a pod or a `StrimziPodSet` *forces* a roll of that node. There is no symmetric "do not roll this node" annotation.
- Pausing the `KafkaNodePool` does not stop rolling. `KafkaRoller` is invoked from the `Kafka` CR reconcile loop, not from the node pool reconcile, so a paused node pool does not prevent the operator from restarting that pool's pods.

`KafkaRoller`'s rolling decision is driven largely by pod readiness, so the problem applies to *any* node that stays not-ready longer than the operation timeout — log recovery is just the most acute example. The Kafka Agent readiness endpoint (`/v1/ready/`) returns `204` only when the broker state is `RUNNING` (`3`); during on-disk log recovery the state is `RECOVERY` (`2`), the readiness check fails, and the pod stays `0/1` NotReady. `KafkaRoller` waits up to `STRIMZI_OPERATION_TIMEOUT_MS` (default `300000` ms / 5 min) for the pod to become Ready (`KafkaRoller.restartIfNecessary` → `await(isReady(pod), operationTimeoutMs, ...)`); if it does not, the roller can take its not-ready-pod path and restart the pod — regardless of *why* the pod is not ready.

A detail that matters for the design: the force-restart of a not-ready pod is **not** produced by the `podNeedsRestart` predicate. For a node with no config/cert/revision change that predicate returns an **empty** `RestartReasons` set; the restart is decided *inside* `KafkaRoller` (via `markRestartContextWithForceRestart()` after `POD_UNRESPONSIVE`, or `POD_FORCE_RESTART_ON_ERROR` in the `catch (ForceableProblem)` branch). This is why the lever must remove the node from the roller's working set rather than filter restart reasons (see "KafkaRoller behavior for a skipped node").

Proposal [048](https://github.com/strimzi/proposals/blob/main/048-avoid-broker-restarts-when-in-recovery.md) ("Avoid broker restart when in log recovery state") made `KafkaRoller` query the agent broker-state endpoint and, on `RECOVERY`, fail the reconcile rather than force-restart. That helps but does not close the gap: it only fires when the roller reaches the agent and reads `brokerState == 2`. If the agent is unreachable/non-200, or the not-ready reason is not `RECOVERY`, there is still no operator-facing lever to take one node out of the loop — leaving the cluster-wide pause as the only off switch.

## Motivation

The cluster-wide pause is the wrong tool for the most common operational emergencies, which are almost always single-node problems. An operator needs to take one node out of operator-driven rolling — while the rest of the cluster keeps being managed — for many reasons, for example:

- A broker doing a long on-disk log recovery (`RECOVERY`) that exceeds the operation timeout, where a restart throws the in-progress recovery away.
- A broker stuck not-ready for a *non*-`RECOVERY` reason (slow startup, a hung mount, a degraded-but-not-failed disk) where a restart will not help and may make things worse.
- A node deliberately held out for investigation — you want to inspect it live, not have the operator restart it underneath you.
- A known-bad host awaiting a hardware/disk swap, where rolling the pod just reschedules the same problem.
- A node mid-rebalance or in some other in-flight state you do not want a roll to interrupt.
- (Future) a controller you need to hold steady — out of v1 scope; see "Controller handling".

These share one shape: a human knows a specific node should be left alone, and wants the operator to keep doing everything else. Today there is no way to express that short of the cluster-wide pause.

One verified incident makes the failure mode concrete. The affected cluster ran in KRaft mode with `KafkaNodePools` and `StrimziPodSet`. After a storage outage, a single broker had to perform a long on-disk log recovery — thousands of partitions, broker state `RECOVERY`, recovering at roughly 470 segments/min, taking well over the 5-minute operation timeout.

The failure sequence:

1. The affected broker is in `RECOVERY`, so the Kafka Agent readiness endpoint fails and the pod is `0/1` NotReady.
2. `KafkaRoller` waits up to `STRIMZI_OPERATION_TIMEOUT_MS` (default `300000` ms) for readiness, does not get it, and takes its not-ready-pod path. It rolls (deletes/recreates) the still-recovering pod. The empty reasons list seen in the incident logs (`due to []`) reflects that the `podNeedsRestart` predicate produced no reasons — the restart was driven by the roller's internal not-ready / force-restart path, not by the predicate.
3. The reconcile times out again, surfaces as a `FatalProblem`, and the next periodic reconcile (`STRIMZI_FULL_RECONCILIATION_INTERVAL_MS`, default `120000` ms) repeats the whole thing.

This is an infinite restart loop. The intrinsic harm — independent of storage setup — is that every forced restart discards in-progress recovery: `LogLoader` starts over from scratch, so a long recovery can never finish and the cluster stays permanently under-replicated. In this particular incident it was made worse by storage topology: the restart rescheduled the pod onto a different Kubernetes node (host), and because the volumes were `ReadWriteOnce` the pod then hit `Multi-Attach` errors (the PersistentVolume was still attached to the prior host). That is an aggravating factor specific to RWO storage, not a precondition for the loop — the recovery-restart loop happens regardless.

Notably, the deployed operator already shipped proposal-048's recovery detection and it still did not prevent the loop. With the data volume stuck, the Kafka Agent stayed reachable on TCP but never returned a broker-state response, so `KafkaAgentClient.getBrokerState()` — which has no request timeout, [strimzi-kafka-operator#12513](https://github.com/strimzi/strimzi-kafka-operator/issues/12513) — hung and resolved to `BrokerState(-1, null)`. Because `-1 != 2` (`RECOVERY`), `KafkaRoller` fell through to force-restart as if 048 were absent. A timeout fix is tracked in [#12675](https://github.com/strimzi/strimzi-kafka-operator/pull/12675), but in the operator source reviewed here `KafkaAgentClient` still sets no request timeout, so the hang is reproducible — check its merge state before relying on it.

The only mitigation available without this proposal is `strimzi.io/pause-reconciliation="true"` on the whole `Kafka` CR. That does stop the loop, but at the cost of freezing config, certificate rotation, scaling, and rolling for *every other* broker and controller in the cluster — exactly when you most want the operator to keep managing the healthy nodes. What is missing is a scalpel: stop the operator from rolling one node, and only that node.

### Why an explicit lever, not just a #12513/048 fix

The natural objection is "just fix #12513 and harden 048 — then the roller detects recovery and stops on its own." That passive fix (give `getBrokerState()` a request timeout so 048 more reliably observes `RECOVERY`) is worth having, and this proposal does not replace it — but a heuristic and a lever solve different problems:

| | Passive #12513/048 heuristic | Explicit skip lever (this proposal) |
|---|---|---|
| Trigger | Operator infers `brokerState == RECOVERY` | Human sets an annotation / `spec` field |
| Coverage | Only states the agent reports as `RECOVERY` | Any reason: stuck non-recovery state, investigation, known-bad hardware, controller issues |
| Failure mode on indeterminate state | `BrokerState(-1)` → falls through to force-restart | Node is skipped regardless of agent reachability |
| Determinism | Best-effort; depends on agent reachability | Deterministic; independent of agent |

Even a fully-patched 048 only changes the *automatic* decision when it can read a clean `RECOVERY` state; on a stuck or unreachable agent it returns `BrokerState(-1)` and force-restarts. The two compose: the heuristic shrinks how often a human must intervene; the lever is the deterministic override for when they must. Ship both.

## Proposal

Introduce an opt-in, per-node control that takes a single node out of operator-driven rolling while the operator keeps reconciling every other node. The full design spans brokers and controllers; v1 implements brokers and refuses controllers.

### MVP / v1 scope

Throughout this proposal, **"v1"** refers to the first implementation increment of this feature — not a Strimzi release version or a CRD API version (e.g. `v1beta2`). To give reviewers a smaller, clearly bounded first cut, v1 is intentionally narrow. Everything outside this list is either deferred (see Out of scope) or a later increment of the same design:

- **Brokers only; controllers refused in v1.** Broker skip is the motivating case and is quorum-safe by construction. In v1 a skip that targets a controller node (controller-only or combined broker+controller) is **refused** — the entry is ignored, a `.status` condition and warning event explain why, and the operator keeps managing that node normally. The whole controller machinery (refuse-by-default quorum math, the `force` exception, per-reconcile quorum re-evaluation) is **future work**, not v1. This keeps the most contentious surface out of the first review.
- **Two surfaces, one meaning.** The ephemeral pod-annotation fast path (`strimzi.io/skip-rolling-update="true"`) and the durable `KafkaNodePool.spec` field. The pod annotation is what an operator reaches for mid-incident; the `spec` field is what survives pod recreation (and any declarative re-apply, GitOps included).
- **Remove the skipped node from the roller's working set.** This is the mechanism (see "KafkaRoller behavior"): the node is excluded from the set `KafkaRoller` iterates, so it is never a roll candidate **and** the reconcile never waits on its readiness. As a consequence *every* operator-driven roll of the node is suppressed at once — the not-ready/force-restart loop, config, cert-renewal, and version-upgrade rolls — and it keeps its current config/cert/version until unskipped. This is a deeper change than filtering restart reasons, and the part v1 strictly requires.
- **Behind a feature gate**, default off (see below).

The genuinely separable later increments are (1) the availability-math refinement for the *other* nodes and (2) controller support; both are out of v1. A reviewer uneasy about controllers can still accept v1 — broker-only, working-set exclusion, feature-gated.

### API: trigger surfaces

Two surfaces with two distinct, non-overlapping schemas — never the same key carrying two value grammars.

**1. Ephemeral fast path — pod annotation (boolean only).**

```
strimzi.io/skip-rolling-update="true"
```

Set on the node's `Pod`. Boolean grammar only: `"true"` enables the skip, anything else (including absence) means not skipped. This is precise, immediately actionable during an incident (one `kubectl annotate pod` on the exact misbehaving broker), and mirrors the granularity of `strimzi.io/manual-rolling-update`. A sibling key `strimzi.io/skip-rolling-update-force="true"` gates the controller force exception; it is reserved for the **future** controller work (inert in v1) so the grammar is stable.

**2. Durable path — `KafkaNodePool.spec` field (list of node IDs).**

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: brokers
spec:
  replicas: 6
  roles:
    - broker
  skipRollingUpdate:
    nodeIds: [2]       # node IDs in this pool to exclude from operator rolling
    force: false       # future: required to skip a controller that would risk KRaft quorum (inert in v1)
```

The durable surface is a real `spec` field on `KafkaNodePool`, **not** an annotation. This is a deliberate consequence of the durability analysis below. `nodeIds` is a list of integer node IDs that belong to this pool; `force` is the controller exception flag and is **inert in v1** (controller skips are refused regardless of `force`; the flag becomes meaningful only when the future controller work lands).

**Why two surfaces.** The annotation is the fast path (zero CRD round-trips, ideal for "stop hurting this broker right now"); the `spec` field is the durable path (see below). A node is skipped if **either** its pod carries `strimzi.io/skip-rolling-update="true"` **or** the owning pool's `spec.skipRollingUpdate.nodeIds` lists that node's ID.

**Parse-failure behavior.** The `spec` field is schema-validated by the CRD (`nodeIds` is `array<integer>`), so non-integer entries are rejected at admission by the API server. For values that parse but are not meaningful — a node ID that does not belong to this pool, a duplicate, or a negative ID — the operator **ignores the invalid entry**, raises a Kubernetes warning event and a `.status` condition naming the bad value, and **still applies the valid entries in the same list**. Ignoring (rather than skipping) is the fail-safe direction: a typo must never silently disable operator self-healing for a node the operator should be managing. A malformed pod annotation value (anything other than `"true"`) is treated as "not skipped," same fail-safe direction.

### Why the durable path is a `spec` field, not an annotation

This is **not** a GitOps-specific concern; it follows from how Strimzi owns pods. A pod annotation is inherently ephemeral:

- Pods are created by the operator from a `StrimziPodSet`-owned template. A human-applied annotation on a running `Pod` is **not** part of that template, so the moment the operator recreates or reschedules the pod the annotation is gone. This is true on any cluster, with or without GitOps.
- Desired state that must persist belongs in a declared, reconciled `spec` — not in an out-of-band imperative annotation. `KafkaNodePool.spec.skipRollingUpdate` survives pod recreation because the operator re-reads it from the pool on every reconcile.

Continuous-reconciliation tooling makes the ephemerality more acute but is only one instance of it: ArgoCD/Flux (especially with auto-sync + self-heal) will actively *revert* a `kubectl annotate` that is not in the manifest, often within minutes. Plain `kubectl apply` of a manifest that omits the annotation does the same on the next apply. The general rule holds regardless: a long-lived skip must live in a CR `spec` the user authors; the pod annotation is the **ephemeral fast path** ("right now, before I commit the change"), explicitly documented as non-durable.

### Naming and precedence

- **Name.** `strimzi.io/skip-rolling-update` (and the `spec.skipRollingUpdate` field) deliberately sit in the `manual-rolling-update` family and read as "exclude this node from rolling." The earlier `pause-node-reconciliation` name is rejected: it implies freezing all reconciliation for the node, which is broader than what this does and collides conceptually with the whole-CR `pause-reconciliation`.
- **Precedence vs `strimzi.io/manual-rolling-update`.** The two are opposites (force vs suppress a roll). If both apply to a node, **skip wins** (conservative — the point is to keep the operator off a flagged node), with a warning event + `.status` condition flagging the contradiction. Crucially this must be enforced at **two call sites**: manual rolling update runs in its own early phase (`KafkaReconciler.manualRollingUpdate()`, built from pod/`StrimziPodSet` annotations) *before* the main `rollingUpdate()`, so skipped nodes must be filtered out of **both** the early manual set and the main working set — enforcing it only in the main roll would still let the early phase restart a skipped node.

### Feature gate

The gate is warranted because this lever lets a human **switch off the operator's self-healing** for a node — a new, safety-relevant behavior. Letting a node sit NotReady indefinitely without the operator acting should ship opt-in and graduate gradually. (Justified by the self-healing change alone; the deferred availability-math refinement, when it lands, sits behind the same gate.)

- Gate name: `SkipNodeRollingUpdate`, default **disabled** in the introducing release, following Strimzi's standard alpha → beta → GA graduation. (Mechanically this is a new `FeatureGate` in `FeatureGates`, constructed with default `false`, matching how `UseBackgroundPodDeletion` is wired.)
- When the gate is disabled, the annotation and `spec` field are parsed but inert, and the roller behaves exactly as today (including 048 logic). This lets the field bake without changing anyone's safety semantics until they opt in.

### Security and RBAC

This lever switches off the operator's self-healing for a node, so it is privileged: anyone who can `patch`/`annotate` pods can set `strimzi.io/skip-rolling-update="true"` on every broker and stop the operator healing the cluster — an availability/DoS vector where pod-edit is broadly granted. No new RBAC object is introduced; the mitigation is that the durable surface is a `KafkaNodePool.spec` field (behind CR-level RBAC, typically far more restricted than pod-edit, and auditable), plus guidance to restrict pod `patch` in Strimzi namespaces and treat the annotation as break-glass. Every enter/leave of the skipped state emits an event for audit.

### KafkaRoller behavior for a skipped node

The skip state of a node is evaluated via a single predicate `isRollingUpdateSkipped(nodeId)` (true if the pod annotation or the pool `spec` lists the node). The important design decision is **where** this predicate is applied — and it is *not* a filter on the assembled `RestartReasons`. See "Why working-set exclusion, not a reason filter" below for the code-level reason.

For a skipped (broker) node, the predicate gates **membership in the working set**: the node is excluded when the roller builds the list it iterates (the `controllerPods`/`brokerPods` ordering in `KafkaRoller.rollingRestart`, fed from `kafka.nodes()` via `KafkaReconciler.maybeRollKafka`). That single exclusion removes it from **both** the roll-candidate set **and** the readiness-wait set, so the node is never force-restarted *and* the reconcile never times out on its readiness. Because it is out of the working set it also cannot trip the `FatalProblem` from the readiness-wait (`await(isReady(...), operationTimeoutMs, FatalProblem::new)`) that would otherwise abort the *entire* roll — the roller still evaluates and, where safe, rolls the *other* nodes (kept safe by the existing availability check; see "Availability math") and completes the reconcile. **This is the v1 behavior.**

#### Why working-set exclusion, not a reason filter

Filtering the assembled `RestartReasons` for a skipped node does **not** work for the case this proposal exists to fix:

- The not-ready / force-restart loop is **not** driven by the predicate's `RestartReasons`. For a node with no config/cert/revision delta, `podNeedsRestart.apply(pod)` returns an **empty** set (the `due to []` in the incident); the restart is decided inside `KafkaRoller` by `markRestartContextWithForceRestart()` (after `POD_UNRESPONSIVE`) or `POD_FORCE_RESTART_ON_ERROR` (in the `catch (ForceableProblem)` branch). A reason filter never sees these.
- Even with zero reasons, an in-scope node is still subject to `await(isReady(...), operationTimeoutMs, ...)`, whose timeout maps to a `FatalProblem` that aborts the whole reconcile. A reason filter does nothing about this.

So the predicate gates **membership in the working set**, applied where the roller orders nodes and where `manualRollingUpdate()` builds its early node set (per precedence) — one exclusion at set-construction time, not a per-reason guard.

### Availability math for a skipped node

**v1 needs no change to the availability math.** Before rolling a broker, `KafkaAvailability.canRoll(podId)` checks, per partition replicated on the candidate, that taking it down would not push the partition below `min.insync.replicas` — evaluated against the **live ISR** from `describeTopics` (`TopicPartitionInfo.isr()`). A broker that is genuinely down or mid-recovery has, in the common case, already dropped out of the reported ISR, so it is already excluded with no code change. v1 therefore ships working-set exclusion alone and still rolls the rest of the cluster safely: the roller uses real ISR for the other nodes and already defers any roll that would breach min-ISR.

**No-deadlock guarantee (holds in v1 with exclusion alone).** The skipped node is removed from the candidate set and the readiness-wait, so it can never hang or time out the reconcile. A roll of some *other* node that would breach min-ISR is deferred, exactly as the roller already defers any unsafe roll today; nodes that share no at-risk partition with the skipped node continue to roll; the reconcile completes the safe subset and held-back rolls retry next reconcile. Worst case is "an unsafe roll waits until it is safe" — never "all rolling is frozen".

**Optional follow-up refinement (not v1).** One residual race: a broker skipped but still *listed* in ISR (very early in recovery) is counted as in-sync when judging some *other* roll. The refinement closes it by subtracting skipped brokers — `effectiveISR(P) = ISR(P) \ ({C} ∪ skippedBrokers)` — threaded into `KafkaAvailability.wouldAffectAvailability`. Small, but bounded to that window, so deferred.

**Last-in-sync-replica honesty.** Skipping a node cannot *restore* availability for a partition whose only ISR member lives on the skipped node — that partition is already under min-ISR for acks-`all` writes. The skip does not change that; it only stops the operator making it worse by rolling another replica. The metrics below surface under-min-ISR partitions and those whose sole ISR member is on a skipped node, so the risk is visible.

### Controller handling: refused in v1

Skipping a controller is more dangerous than skipping a broker, because controllers form the KRaft metadata quorum. Rather than ship a half-built quorum-safety story, **v1 refuses controller skips outright** and defers the full machinery:

- A skip targeting a controller node (controller-only or combined broker+controller) is **ignored**; the operator raises a `.status` condition (`type: SkipRollingUpdateRefused`) and a warning event, and **keeps managing that controller normally** (`KafkaQuorumCheck` behavior unchanged). Fail-safe: a controller the quorum may not be able to lose is never silently dropped from self-healing.
- **Pure-broker skip is always quorum-safe** and is honored without any quorum check (normal broker availability checks still apply).

This keeps v1 out of the controller roll path entirely — the most failure-prone surface. The future `force`-gated controller path (admission-time `DescribeQuorum` check, per-reconcile re-evaluation, `ControllerQuorumAtRisk`, and deciding other-controller rolls from real quorum state rather than a blanket "skipped = down") is specified in "Out of scope / Future work".

### Interaction with readiness checks

Skipping changes only what the *operator* does; it does not touch kubelet probes. The pod's `readinessProbe`/`livenessProbe` keep running, so it still reports `0/1` NotReady in `RECOVERY` and `1/1` at `RUNNING` — readiness keeps reflecting the genuine broker state, and the skip only stops the operator from *acting* on that signal. The trade-off: a node failing liveness for a genuinely fatal reason will not be helped while skipped, so a skip is an explicit "I am supervising this node" statement to be cleared once healthy.

The `POD_STUCK` / unschedulable path is also suppressed, deliberately: `KafkaRoller`'s early `isPodStuck` check (→ `FatalProblem`, firing for `CrashLoopBackOff`, `ImagePullBackOff`, `ContainerCreating`, `InvalidImageName`, or Pending+Unschedulable, *before* any restart-reason logic) is not reached for a node out of the working set. This matches the lever's meaning ("I am handling this") and is the safe direction — a recovering pod that flaps into a stuck-looking state must not be force-restarted. A genuinely stuck pod left alone is acceptable for a short-lived, surfaced, human-supervised skip.

### What a skip blocks: config, certs, and version upgrades

Because the node is out of the working set, a skip blocks *all* operator-driven rolls of it, not just the not-ready loop: a restart-requiring **config change** does not reach it; a **version upgrade/downgrade** leaves it on its current version (do not leave a node skipped across an intended upgrade); and **certificate renewal** does not roll it, so across a CA renewal its cert can age out or even expire while skipped.

Handling: surface a `.status` condition when a skipped node nears its cert-renewal deadline (reusing the window the operator already tracks) or has a pending config/version change blocked by the skip; **do not auto-unskip** (that reintroduces the suppressed restart — the human decides, prompted by the condition); and document that a skip is meant to be short-lived, not a permanent exclusion.

### Drain Cleaner interaction

A skip does **not** make a node un-evictable. Strimzi Drain Cleaner intercepts pod eviction (e.g. from a node drain) and asks the operator to roll the pod gracefully; a skipped node is still a normal pod to the eviction path. So a drain can evict a skipped, recovering broker → the `StrimziPodSet` recreates it → recovery restarts from scratch — the exact harm the skip was meant to prevent, via a different door.

v1 guidance, at minimum:

- **Operationally exclude the skipped node from Drain Cleaner.** While a node is skipped, the operator documentation instructs disabling Drain Cleaner handling for that node (and cordoning the Kubernetes node so the cluster autoscaler / drain tooling does not evict it), so an eviction does not silently undo the skip.
- A first-class integration where Drain Cleaner itself honors the skip and refuses to evict a skipped node is noted as future work, not v1.

### Status reporting

The skipped state must be observable without reading annotations off individual pods:

- A `Kafka` CR `.status` condition (`type: NodeRollingUpdateSkipped`) listing skipped node IDs and, per node, the time it entered the state ("skipped since `<timestamp>`", via `lastTransitionTime`) — paralleling the existing `ReconciliationPaused` condition. Mirror per-node detail (with reason source: annotation vs `spec`) in the owning `KafkaNodePool` `.status`; a node that resolves to a controller shows `SkipRollingUpdateRefused` instead.
- Kubernetes events when a node enters/leaves the skipped state and when a controller skip is refused. (Force-override and quorum-at-risk conditions/events arrive with the future controller work.)

### What a skip does not stop

A skip suppresses operator-*initiated* rolls only. It does **not** stop: the `StrimziPodSet` controller recreating the pod if it is deleted/evicted (which is why the durable `spec` path exists and Drain Cleaner must be handled — a recreated pod is skipped again on the next reconcile); kubelet liveness/readiness probes; Kubernetes host-level eviction/drain/scheduler actions (see Drain Cleaner); or reconciliation of *other* nodes.

## Metrics

New operator metrics (Prometheus, `strimzi_` prefix, labeled by cluster / node pool / node ID where applicable), to make the dangerous states (forgotten skip, quorum thinning, sole-replica-on-skipped-node) alertable rather than implicit.

v1:
- `strimzi_nodes_skip_rolling_update` (gauge) — nodes currently excluded from rolling, per cluster.
- `strimzi_node_skip_rolling_update_start_timestamp_seconds` (gauge, per node) — when the node entered the skipped state, backing "skipped since" and long-lived-skip alerts.
- `strimzi_skip_rolling_update_refused_total` (counter) — skips refused (in v1, controller-skip refusals).
- `strimzi_partitions_under_min_isr` and `strimzi_partitions_sole_isr_on_skipped_node` (gauges) — read-only observability backing the last-in-sync-replica honesty (independent of the deferred availability-math change).

Future work (with controller support): `strimzi_controller_quorum_at_risk` (gauge) — `1` when a skipped controller's quorum drops to/below sub-majority.

## Testing strategy

- **`KafkaRoller` unit tests.** A skipped NotReady broker (for any reason, not only `RECOVERY`) is not rolled and excluded from the working set, so the reconcile does not time out on its readiness (`FatalProblem` never reached) — the core regression test; likewise a skipped broker with a pending config/cert reason is not rolled. No global deadlock: a broker sharing no at-risk partition still rolls and the reconcile completes the safe subset, while a roll that would breach min-ISR is deferred via the *existing* availability check (confirming v1 needs no math change). Precedence: skip wins at **both** call sites (early `manualRollingUpdate()` set and main roll) with the contradiction warning. Parse-failure: invalid `nodeIds` ignored, valid ones applied, warning surfaced. Controller refusal: a controller-targeting skip (controller-only and combined) is refused with `SkipRollingUpdateRefused` and still managed.
- **System tests (`systemtest`).** A NotReady broker is not force-restarted while skipped and the operator keeps rolling other brokers; rolling another node is blocked when it would breach min-ISR given a skipped node and proceeds once lifted; controller skip is refused and surfaced; with the gate off, annotation and `spec` field are inert.
- **Upgrade/downgrade tests.** A downgrade to an operator without the gate reverts to legacy behavior; assert the documented data-loss risk on a skipped node is exercised and surfaced.

## Documentation

- Operator docs: a "Skipping rolling update for a single node" section covering the pod-annotation fast path, the `KafkaNodePool.spec.skipRollingUpdate` durable field, the ephemeral-vs-durable distinction, and the v1 controller-refusal limitation (`force` as future work). Plus operational guidance (skips are short-lived; disable Drain Cleaner and cordon the host while skipped; never skip across an intended upgrade/cert renewal; restrict pod-edit RBAC) and a troubleshooting entry tying the `Rolling Pod ... due to []` symptom to this lever and 048 / #12513.
- API reference updates for the new `spec` field, annotations, `.status` conditions, and metrics.

## Affected/not affected projects

Affected:
- `strimzi/strimzi-kafka-operator` — the cluster operator. In v1: `KafkaRoller`/`KafkaReconciler` working-set construction (excluding skipped broker nodes from both the main roll and the early `manualRollingUpdate()` set), the reconcile pipeline that reads the pod annotation and `KafkaNodePool.spec.skipRollingUpdate` field, the `KafkaNodePool` CRD (new `spec` field), `Kafka` / `KafkaNodePool` `.status` conditions (including controller `SkipRollingUpdateRefused`), new metrics, and the `SkipNodeRollingUpdate` feature gate. **Not** changed in v1: `KafkaAvailability` (the ISR-subtraction refinement is a follow-up) and `KafkaQuorumCheck` / controller quorum math (controller skips are refused).

Not affected (by v1): `strimzi/strimzi-kafka-bridge`, `strimzi/strimzi-kafka-oauth`, `strimzi/test-clients` and CI repos. The Kafka Agent's existing endpoints are reused as-is; no new agent API is required. `strimzi/drain-cleaner` is unchanged in v1 (the interaction is handled operationally), but a future first-class integration would affect it.

## Compatibility

The feature is fully opt-in and gated. With the `SkipNodeRollingUpdate` feature gate disabled (the introducing-release default), the new annotation and `spec` field are inert and the operator behaves exactly as it does today, including the proposal-048 recovery-aware logic. Older clients/tooling that do not understand the new `.status` conditions simply ignore them (conditions are additive).

The `KafkaNodePool` CRD gains an optional `spec.skipRollingUpdate` field. Adding an optional field is backward compatible: existing `KafkaNodePool` resources that omit it are unaffected.

**Downgrade is a data-loss risk, not a cosmetic one.** A downgraded operator that does not understand the skip will treat a skipped, recovering node as an ordinary not-ready pod and **force-restart it** — which is exactly the recovery-destroying loop this proposal exists to stop. Restarting a node mid log-recovery discards in-progress recovery (`LogLoader` starts over), and on `ReadWriteOnce` storage can additionally trigger `Multi-Attach` volume errors when the pod reschedules to another host. So downgrading the operator while a node is skipped can directly cause recovery loss and prolonged under-replication. Operators must lift skips and confirm all nodes healthy before downgrading to a version without this feature; the downgrade docs must call this out as a destructive-action warning, not a footnote.

## Rejected alternatives

- **Whole-`Kafka`-CR pause (`strimzi.io/pause-reconciliation`).** This exists today and is exactly the pain this proposal addresses. It is too coarse: to protect one node you must freeze config, cert rotation, scaling, and rolling for the entire cluster, including all the healthy nodes you still want the operator to manage. It is an emergency sledgehammer, not a per-node control.

- **A durable node-pool *annotation* instead of a `spec` field.** Rejected on durability grounds (above): a pod annotation does not survive pod recreation, and any declarative re-apply (plain `kubectl apply`, or ArgoCD/Flux with self-heal) reverts an annotation not present in the manifest. The durable surface must be a `spec` field the user authors; the annotation survives only as the documented ephemeral fast path.

- **Overloading one annotation key with two value grammars** (`="true"` on pods, `="2,3"` on node pools). Rejected: one key with two meanings is a parsing and documentation hazard. v1 uses a boolean pod annotation and a typed `spec` list — distinct keys, distinct schemas, schema-validated.

- **Raising `STRIMZI_OPERATION_TIMEOUT_MS`.** Operator-wide and a blunt instrument. It would have to exceed the *longest* plausible not-ready duration for *any* node, slowing every legitimate roll across every cluster, requires an operator restart to change, and still loops on any not-ready state longer than the new threshold. It trades one fragile threshold for another.

- **Making the readiness probe pass during recovery.** Reporting a recovering broker as Ready would stop `KafkaRoller` from restarting it, but it lies to everything else that depends on readiness. Clients and load balancers would route to a broker that cannot serve its partitions, and genuine not-ready states would be masked. The fix belongs in what the operator *does* with the state, not in falsifying the state.

- **Pausing the `KafkaNodePool` (as a means to stop rolling).** Does not work: `KafkaRoller` is driven from the `Kafka` CR reconcile, not the node-pool reconcile, so a paused node pool does not prevent rolling its pods. This proposal adds a `spec` field on `KafkaNodePool` as a durable trigger; it does not rely on node-pool pause semantics.

- **Relying solely on proposal 048 / a #12513 fix.** Covered in "Why an explicit lever, not just a #12513/048 fix": a passive heuristic and an explicit lever cover different problems; ship both.

## Out of scope / Future work

- **Controller skip support.** v1 refuses all controller skips. The follow-up adds a `force`-gated warn-and-honor path, built so it cannot deadlock controller rolling: a refuse-by-default `DescribeQuorum` check at skip admission (refuse if honoring the skip would drop the quorum below majority); `force: true` to override with a loud `.status` condition and event; per-reconcile re-evaluation raising `type: ControllerQuorumAtRisk` and `strimzi_controller_quorum_at_risk` if a skipped controller's quorum thins; and deciding whether *other* controllers can roll from real `DescribeQuorum` state (`isQuorumHealthyWithoutNode`) rather than treating the skipped one as a blanket "down".
- **Availability-math refinement for other nodes.** Subtracting skipped brokers from the effective ISR when judging *other* rolls (beyond the live ISR the roller already reads) can land as a separate increment; v1 can ship with working-set exclusion alone, since the live ISR already excludes a genuinely-down skipped broker in the common case.
- **First-class Drain Cleaner integration** so a skipped node is automatically excluded from drain-driven evictions, rather than handled operationally.
- **Auto-expiry of a skip** after a configurable duration, so a forgotten skip does not silently block cert rotation / config / upgrades forever (the metrics make a forgotten skip alertable in the meantime).
- **Per-node skip expressed as a `Pod`-owned CRD or sub-resource** rather than an annotation fast path, if the annotation's RBAC surface proves too permissive in practice.
