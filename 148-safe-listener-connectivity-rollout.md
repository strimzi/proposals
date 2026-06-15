# Safe rollout of listener connectivity changes

Make connectivity-affecting listener changes (listener type changes, port changes, adding/removing a listener) safe to roll out and roll back, for **all listener types that provision per-broker Kubernetes resources** — `nodeport`, `loadbalancer`, `cluster-ip`, `route`, and `ingress`.

The proposal reconciles each broker's listener resources (its `Service`, and `Route`/`Ingress` where applicable) **in lockstep with that broker's own restart**, instead of replacing all per-broker resources for the whole cluster up front while the brokers restart one by one. This removes the availability window — present for every per-broker listener type — where a not-yet-rolled broker's resources already point at a listener the broker is not yet serving.

> **Scope: KRaft only.** This proposal assumes KRaft-based Kafka clusters and does not consider ZooKeeper-based clusters (ZooKeeper support has been removed from current Strimzi). The partial-roll correctness argument relies on KRaft broker-registration semantics (see [KRaft assumption](#kraft-assumption)).

## Current situation

The Kafka reconciliation pipeline reconciles listeners and rolls the pods in two separate, sequential phases:

```java
// KafkaReconciler.reconcile()
.compose(i -> listeners())                       // (1) reconcile ALL Services + compute advertised.listeners
...
.compose(i -> brokerConfigurationConfigMaps())   // (2) write broker configs (incl. advertised.listeners)
...
.compose(i -> podSet())
.compose(podSetDiffs -> rollingUpdate(podSetDiffs)) // (3) restart pods ONE BY ONE
```

In phase (1), `listeners()` reconciles the shared bootstrap resources and **every** per-broker listener resource at once. Each external listener provisions per-broker Kubernetes objects — a `Service` for `nodeport`/`loadbalancer`/`cluster-ip`, plus a `Route` (`route`) or `Ingress` (`ingress`) — and they are all reconciled cluster-wide before any pod rolls.

The brokers, however, only pick up the new listener configuration when they restart, which happens **one pod at a time** in phase (3). This gap produces two failure modes.

#### Failure mode A — resource/broker mismatch window (all per-broker listener types)

When the listener **type** changes (for example `nodeport` → `cluster-ip` or a rollback back to `nodeport`), a `port` changes, or a listener is added/removed, all per-broker resources are switched/replaced immediately, but each broker only starts serving the new listener when it restarts. Brokers that have **not yet** restarted still serve the **old** listener, but their resources have **already** been switched. Clients connecting to those brokers fail (connection refused / unreachable / wrong endpoint) for the whole duration of the roll, not just briefly during a restart. This applies to **every** per-broker listener type, regardless of how addresses are assigned.

#### Failure mode B — address churn on resource recreate (`nodeport` and `loadbalancer`)

Some listener types do not have a stable externally-visible address across a resource recreate: a `nodeport` `Service`'s node port is preserved only when the old and new `Service` are both already `NodePort`, so across a **type change** Kubernetes allocates fresh random ports on the way back to `NodePort`; a `loadbalancer` `Service` may get a new external IP/hostname on recreate. Combined with Failure mode A, the consequence can be severe: a port that previously fronted one listener may be reallocated to a *different* listener, so a client with cached metadata that cannot refresh (because the rest of the cluster is unreachable mid-roll) sends, for example, a SCRAM handshake to a port now serving the Kerberos listener — an authentication-protocol mismatch.

`cluster-ip`, `route`, and `ingress` addresses are derived deterministically from resource names, so they suffer Failure mode A but not B.

## Motivation

The root cause is an **atomicity mismatch**: listener resources are switched atomically for the whole cluster, but the brokers that back those resources converge gradually as they restart one by one. During that window every not-yet-rolled broker advertises a listener it is not yet serving, so clients fail to connect; and because most of the cluster is unreachable, clients cannot refresh their metadata to recover. On `nodeport`/`loadbalancer` the same window is what makes a port/address reshuffle dangerous (Failure mode B): without a reachable cluster to refresh from, cached clients keep hitting stale — and possibly cross-wired — endpoints (for example a SASL client reaching a port now serving a different mechanism), which on a real cluster has produced cluster-wide produce/consume failures for the entire duration of a listener type change and rollback.

Reconciling each broker's listener resources in lockstep with that broker's own restart removes the window: at every instant each broker is internally consistent (its resources match the listener it is running), so the change rolls through the cluster the same safe way a normal restart does, and clients can always refresh metadata against the parts of the cluster that are still up.

## Proposal

When a reconciliation includes a **connectivity-affecting listener change**, reconcile the per-broker listener resources (the broker's `Service`, and its `Route`/`Ingress` where applicable) as part of the rolling update, broker by broker, instead of all at once before the roll.

A change is "connectivity-affecting" when it changes what a client must connect to:

- a listener's `type` changes;
- a listener's `port` changes;
- a listener is added or removed.

All other reconciliations (config-only changes, certificate rotations, image upgrades, scaling, etc.) keep the current behaviour unchanged.

### Precondition: the internal listeners are never part of the change

The rolling update's safety checks (quorum / in-sync-replica `canRoll`) depend on the operator's admin client, which connects over the **internal** replication and control-plane listeners (the headless `brokers` service on `REPLICATION_PORT` / `CONTROLPLANE_PORT`), not the external listener being changed:

```java
// KafkaRoller.adminClient(...)
bootstrapHostnames = nodes.stream().filter(NodeRef::broker)
    .map(node -> DnsNameGenerator.podDnsNameWithoutClusterDomain(namespace, KafkaResources.brokersServiceName(cluster), node.podName()) + ":" + KafkaCluster.REPLICATION_PORT)
    .collect(Collectors.joining(","));
```

This is exactly what makes the approach safe: the external listener can be in flux while the operator still reaches the cluster for availability checks. The coupled path is therefore valid **only for external/client listeners**; a change to the internal replication/control listener cannot use this mechanism and must fall back to the current behaviour (or be disallowed). The detector must guard for this.

### Per-broker convergence guard (idempotency)

Reconciliations can be interrupted and re-run, leaving the cluster half-migrated (some brokers new, some old). The "connectivity change" decision must therefore be evaluated **per broker** — "do this broker's observed listener resources already match the desired listener config?" — not as a single cluster-global flag. Brokers already at the desired state are skipped, so a re-run resumes the migration instead of re-rolling the whole cluster. This makes the operation convergent and resumable.

### Per-broker rollout loop

For each broker that is not yet converged, in the order and under the safety gates the existing roll already enforces:

1. Reconcile **that broker's** listener resource(s) — `Service`, and `Route`/`Ingress` where applicable — to the desired state (create/update/delete). For types whose externally-visible address is assigned asynchronously, wait until it is available for that broker: the assigned `nodePort` (`nodeport`), the load-balancer ingress IP/hostname (`loadbalancer`), or the admitted host (`route`). `cluster-ip`/`ingress` addresses are deterministic and need no wait.
2. Render **that broker's** configuration with the resulting `advertised.listeners` and apply its `ConfigMap`.
3. Restart the broker so it starts serving the new listener and re-registers its advertised endpoint.
4. Wait until the broker is ready before moving to the next broker.

### Bootstrap resource

The bootstrap resource is shared (one `Service`/`Route`/`Ingress` for the whole listener) and is reconciled **last**, after all brokers have rolled.
This avoids switching the shared entry point until the per-broker endpoints behind it are all consistent, and makes the bootstrap change a single, fast operation at the end.
Note this does **not** guarantee a *reachable* bootstrap throughout the roll: if the bootstrap's current state is itself part of the change (e.g. mid-rollback it is still the wrong type), clients doing a cold start or metadata refresh *through the bootstrap* recover only once it flips at the end. Clients holding cached per-broker metadata are unaffected (see below).

### Availability during a partial roll

A half-rolled cluster does **not** cause widespread failure for clients with cached metadata; it degrades to an ordinary rolling restart.

For any partition, such a client connects to the leader's currently-advertised address:

- leader on an already-rolled broker → metadata gives the new address, the new resource exists → works;
- leader on a not-yet-rolled broker → metadata gives the old address, the old resource still exists → works;
- leader on the broker currently restarting → brief unavailability while leadership fails over to an in-sync replica, covered by normal client retries (RF ≥ 2).

This also bounds the impact of address churn (Failure mode B). Because each broker's resource is switched in lockstep with its restart and the rest of the cluster stays reachable, a client targeting a stale `nodeport`/`loadbalancer` address simply refreshes its metadata and connects to that broker's current address. A reshuffled or cross-wired port affects at most one broker momentarily and is self-healed by a metadata refresh — instead of the sustained, cluster-wide outage that occurs today when the whole cluster is switched at once and clients cannot refresh.

Remaining failure modes are the ordinary ones, not new to this change:

- **RF = 1 topics**: partitions led by the broker currently restarting are unavailable until it returns (true for any restart).
- **Cold-start / bootstrap-dependent clients**: see the bootstrap note above — they recover when the bootstrap flips at the end.
- **Momentary stale metadata**: a client that cached a broker's address immediately before that broker rolls gets a single failed connection and recovers on the next metadata refresh.

This is fundamentally different from the current behaviour, where a not-yet-rolled broker's mismatched resources fail *all* connections to it for the *entire* roll.

### KRaft assumption

The partial-roll correctness argument depends on KRaft semantics and applies to **KRaft-based clusters only**:

- Each broker registers its **own** `advertised.listeners` with the KRaft controller on start-up, and the controller propagates the full set of broker endpoints to all brokers. A broker re-registering a new endpoint when it rolls is therefore enough for the rest of the cluster (and client metadata) to see the change — Strimzi does **not** need to rewrite every broker's configuration up front for the new addresses to become visible. This is what makes incremental per-broker convergence safe.
- Each broker's advertised address depends only on **that broker's own** resources, so its config can be rendered from its own resources alone, with no cross-broker dependency: for `nodeport` the advertised host is resolved by the pod from its scheduled node (via an env var) and only the port comes from the broker's `Service`; for `loadbalancer` the address comes from that broker's LB `Service` ingress; for `cluster-ip` from its per-broker DNS name; for `route`/`ingress` from that broker's `Route`/`Ingress` host.
- Controller-only KRaft nodes have no client listeners and are skipped by the per-broker resource/config steps.

ZooKeeper-based clusters are explicitly out of scope.

### Implementation sketch

The change is an ordering/wiring change in the Cluster Operator; the building blocks already exist.

**1. Detect connectivity change, per broker.**
Before the roll, for each broker compare the desired listener config against the observed per-broker resources to compute whether that broker needs a coupled reconcile. If no broker needs one, the reconciliation keeps the **current** behaviour exactly (`listeners()` → `perBrokerKafkaConfiguration()` → `podSet()` → `rollingUpdate()`). The new path is taken only for connectivity changes on external listeners (per the precondition above).

**2. Split `KafkaListenersReconciler.reconcile()` into reusable pieces.**
Today it reconciles bootstrap + all per-broker resources and returns one `ReconciliationResult` (with `advertisedHostnames` / `advertisedPorts`, both `Map<Integer, Map<String, String>>`). Factor out:

- `reconcileSharedPrerequisites()` — listener `Secret`s/certificates and other non-connectivity, non-bootstrap resources.
- `reconcileBrokerListener(NodeRef)` — reconcile **one** broker's per-broker resources (`Service`, and `Route`/`Ingress` where applicable), wait for that broker's asynchronously-assigned address where relevant (`nodePort` / LB ingress / route host), and return that broker's `advertisedHostnames` / `advertisedPorts` entries. This generalizes the existing per-listener-type readiness steps (`nodePortServicesReady`, `loadBalancerServicesReady`, `routesReady`, …) to operate on a single broker.
- `reconcileBootstrap()` — the shared bootstrap resource (`Service`/`Route`/`Ingress`) and final listener status.

**3. Drive the per-broker resource reconcile from the exact pre-restart moment.**
The roll iterates node by node on a single-threaded executor via `maybeRollKafka(...)` → `KafkaRoller`. A node can be *considered* many times and deferred (`UnforceableProblem` + backoff) when `canRoll` fails, so the only correct place to switch a broker's resources is **immediately before the pod is actually deleted** — i.e. right before each `restartAndAwaitReadiness(...)` call, *after* the availability gate has passed. There are three such call sites in `restartIfNecessary(...)`:

- the `forceRestart` branch (used for stuck/unresponsive/old-revision pods — *not* the connectivity-change path, which is `needsRestart`);
- the normal `needsRestart`/`needsReconfig` branch, reached only after `canRoll(...)` returns true;
- the force-roll-on-error fallback, reached only after `canRoll(..., true, ...)` returns true.

The hook (`reconcileBrokerListener(node)` + that broker's `ConfigMap` update) must:

- fire only at those points, never when a node is merely "considered" (otherwise the broker's resources are switched while the still-running broker serves the old listener — the exact bug, localized to one broker for the duration of a deferral);
- be **idempotent**, because a node may pass `canRoll` and still be retried;
- run before **every** restart of a not-yet-converged broker, including the `forceRestart` and force-roll-on-error branches (see below).

A connectivity change is signalled by `podNeedsRestart` returning a new reason (e.g. `LISTENER_CONFIGURATION_CHANGE`) for brokers that are not yet converged. This yields `needsRestart = true` (so it is gated by `canRoll`) and short-circuits the dynamic-reconfiguration path (`advertised.listeners` is not dynamically updatable anyway).

**Handling the `forceRestart` branch.** Although a connectivity change itself maps to `needsRestart` (not `forceRestart`), a not-yet-converged broker can still hit the `forceRestart`/force-roll-on-error path for an *unrelated* reason — e.g. the pod is stuck, unresponsive, or on an old revision — during the migration. If the listener hook only ran on the `needsRestart` branch, such a broker would be restarted with its resources/`ConfigMap` still on the old listener and would come back mismatched, converging only on a later reconcile. The hook must therefore key off "is this broker not yet converged?" and run before the restart on **all** branches. Because the hook is idempotent and only reconciles that broker's own resources, doing so on the force paths is safe.

**4. Avoid the PodSet "double-roll".**
The advertised host/port hash is folded into the broker config hash and surfaced as a **pod annotation** (`perBrokerKafkaConfiguration()` → `podSetPodAnnotations()`), and `podSet()` runs *before* the roll and can itself restart pods when annotations change. If the new advertised config were applied up front, `podSet()` would roll pods prematurely — with the new annotation but before the per-broker resource hook runs. The implementation must therefore keep the up-front `podSet()` carrying the **old** advertised hash and apply each broker's new config/annotation during the roll (step 3), so a broker is rolled exactly once, by `KafkaRoller`, with its resources already switched. This is the most delicate part of the implementation and must be covered by tests.

**5. Reconcile the bootstrap last**, then populate listener status.

Resulting high-level flow in `KafkaReconciler.reconcile()` for a connectivity-affecting change:

```text
reconcileSharedPrerequisites()        // certs/secrets, no connectivity change
  -> podSet()                         // non-connectivity pod changes only; OLD advertised hash
  -> coupledListenerRollingUpdate()   // maybeRollKafka; for each not-yet-converged broker,
                                      // at the pre-restartAndAwaitReadiness point (post canRoll):
       reconcileBrokerListener(broker)    // that broker's Service/Route/Ingress + wait for assigned address
       update that broker's ConfigMap     // advertised.listeners + annotation for this broker
       restart broker                     // serves new listener, re-registers endpoint
       wait until ready
  -> reconcileBootstrap()             // shared bootstrap resource switched once, at the end
  -> listener status
```

**Edge cases / notes.**

- If a broker fails to roll, the bootstrap is never switched; the cluster is left in a still-functional mixed state (each broker self-consistent) and the next reconcile resumes via the per-broker convergence guard.
- `KafkaRoller`'s existing controller/quorum and in-sync-replica safety checks are unchanged and continue to gate each broker's restart.

### Decision: in-roller hook (and the rejected alternative)

**This proposal recommends the in-roller hook** described above: a single `KafkaRoller` instance for the whole roll, with the strictly-placed, idempotent pre-restart hook reconciling each broker's resources.

The decision is driven by **safety over boundary-cleanliness**. A single `KafkaRoller` instance holds the cluster-wide view needed to roll safely — controller-first ordering, unready-first ordering, and quorum/ISR `canRoll` checks evaluated across all nodes. Preserving that global view is the property that actually protects availability during the roll, and it is the harder thing to get right.

The rejected alternative is to drive the roll from the listener reconciler by calling `maybeRollKafka(...)` with a **single-node set per broker**, reconciling that broker's resources between calls. This keeps resource/`ConfigMap` I/O out of `KafkaRoller` and preserves its clean "pods + admin client only" boundary — attractive, but it spins up a fresh roller per broker, which loses the cross-node ordering and quorum/ISR batching, re-creates admin clients on every call, and re-derives ordering the roller otherwise does for free. Trading away the roller's global safety reasoning to keep a code boundary tidy is the wrong trade for the operator's most safety-critical path, so this variant is rejected. The cost of the chosen approach — widening `KafkaRoller`'s responsibility — is contained by the strict hook placement and idempotency requirements above and must be covered by tests.

### Feature gate and rollout

Because this changes the behaviour of the reconcile/roll path — the operator's most safety-critical area — it is introduced behind a **feature gate** (e.g. `CoupledListenerRollout`), following Strimzi's usual gate lifecycle:

- **Alpha (disabled by default):** opt-in for early adopters and CI; the current all-at-once behaviour remains the default.
- **Beta (enabled by default):** after the test matrix below is green across supported listener types, with the gate still available to disable.
- **GA / gate removal:** once proven in the field.

When the gate is disabled, the code path is exactly today's behaviour, so the change is risk-free to ship dark.

### Testing

The proposal is only credible with explicit coverage of the partial-roll states it claims to make safe:

- **Unit tests** for the per-broker convergence detector (mixed/half-migrated state, internal-listener guard) and for the `KafkaRoller` hook placement (fires only pre-restart after `canRoll`, fires on the `forceRestart`/error branches for not-yet-converged brokers, is idempotent under retries, does not fire for "considered-but-deferred" nodes).
- **Integration tests** asserting no PodSet "double-roll" occurs (each broker rolls exactly once for a connectivity change).
- **System tests** that perform a listener `type` change and a rollback for each per-broker listener type (`nodeport`, `loadbalancer`, `cluster-ip`, `route`, `ingress`) on a multi-broker cluster, asserting clients with cached metadata keep producing/consuming throughout the roll, and that an interrupted reconcile resumes rather than re-rolling.

## Affected/not affected projects

This proposal affects the **Strimzi Cluster Operator** only: `KafkaReconciler` (reconcile ordering), `KafkaListenersReconciler` (split into shared / per-broker / bootstrap reconciliation), and `KafkaRoller` (a strictly-placed, idempotent pre-restart hook and just-in-time advertised host/port supply).

No CRD/API changes are required. No other Strimzi projects are affected. Only KRaft-based clusters are in scope (see [KRaft assumption](#kraft-assumption)).

## Compatibility

- **Gated** behind a feature gate that is disabled by default initially (see [Feature gate and rollout](#feature-gate-and-rollout)), so it ships with zero behaviour change until explicitly enabled.
- **Behaviour is unchanged for non-connectivity-affecting reconciliations**, and for connectivity changes on internal listeners (which fall back to current behaviour).
- **No API/CRD change.**
- **Reconciliation takes longer** for connectivity-affecting changes, because per-broker resource reconciliation (and waiting for an asynchronously-assigned address) is interleaved with the roll. This is most pronounced for `loadbalancer`, where each per-broker LB is provisioned by the cloud (often minutes) and is now serialized across the roll; for large clusters (200+ brokers) the total can be very large. Per-broker waits are bounded by `operationTimeoutMs`, but the overall reconcile may approach or exceed the reconciliation interval, so overlap/lock behaviour and progress logging must be considered. The migration is resumable (per-broker convergence guard), so an aborted reconcile continues on the next pass rather than restarting from scratch.
- **Listener status** is fully populated only after the roll completes; intermediate reconcile passes may report a mix of old and new per-broker addresses, all valid at the time.

## Rejected alternatives

### Roll all resources, then delete all pods at once

Forcing a full, simultaneous pod restart (as was done manually to recover the incident) makes the cluster consistent quickly but causes a hard, total outage during the restart and defeats the purpose of rolling updates. Rejected.

### Make clients tolerate the mismatch

Relying on client-side retry/metadata-refresh tuning to ride out the window does not work when the resource actively fronts the *wrong* listener (auth-mechanism mismatch) or the address no longer exists for a sustained period. The operator must keep server-side resources consistent with the running brokers. Rejected.

### Pin node ports deterministically

Giving `nodeport` listeners stable, formula-derived node ports (so a type change/rollback never reshuffles ports between listeners) was considered. It only narrows Failure mode B for one listener type (`nodeport`) and does nothing for Failure mode A or for `loadbalancer`/`cluster-ip`/`route`/`ingress`, so it is not a general fix. It is also inflexible: formula-derived ports must fit within the cluster's `service-node-port-range`, can collide with other `NodePort` services, and pin the operator to a fixed allocation scheme. Per-broker reconciliation addresses the root cause for all listener types instead. Rejected.
