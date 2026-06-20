# Safe rollout of listener connectivity changes

Make connectivity-affecting listener changes safe to roll out and roll back.
These changes are listener type changes, port changes, and adding or removing a listener.
This covers all listener types that provision per-broker Kubernetes resources: `nodeport`, `loadbalancer`, `cluster-ip`, `route`, and `ingress`.

The proposal reconciles each broker's listener resources in lockstep with that broker's own restart.
Those resources are the broker's `Service`, plus its `Route` or `Ingress` where applicable.
Today the operator replaces every per-broker resource for the whole cluster up front and then restarts the brokers one by one.
Reconciling per broker shrinks the mismatch window from the whole cluster for the duration of the roll down to one broker at a time, the same exposure as a normal rolling restart.
This applies to every per-broker listener type.

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

In phase (1), `listeners()` reconciles the shared bootstrap resources and every per-broker listener resource at once.
Each external listener provisions per-broker Kubernetes objects: a `Service` for `nodeport`, `loadbalancer`, and `cluster-ip`, plus a `Route` for `route` or an `Ingress` for `ingress`.
All of these are reconciled cluster-wide before any pod rolls.

The brokers only pick up the new listener configuration when they restart, and that happens one pod at a time in phase (3).
This gap produces two failure modes.

#### Failure mode A — resource/broker mismatch window (all per-broker listener types)

This covers a listener `type` change (for example `nodeport` to `cluster-ip`, or a rollback to `nodeport`), a `port` change, or adding or removing a listener.
All per-broker resources are switched immediately, but each broker only starts serving the new listener when it restarts.
Brokers that have not yet restarted still serve the old listener while their resources already point at the new one.
Clients connecting to those brokers fail for the whole duration of the roll, not just briefly during a restart.
The failures look like connection refused, unreachable, or wrong endpoint.
This applies to every per-broker listener type, regardless of how addresses are assigned.

#### Failure mode B — address churn on resource recreate (`nodeport` and `loadbalancer`)

Some listener types do not keep a stable externally-visible address across a resource recreate.
A `nodeport` `Service` keeps its node port only when the old and new `Service` are both already `NodePort`, so a type change makes Kubernetes allocate fresh random ports on the way back to `NodePort`.
A `loadbalancer` `Service` may get a new external IP or hostname on recreate.
Combined with Failure mode A, this can be severe: a port that previously fronted one listener can be reallocated to a different listener.
A client with cached metadata that cannot refresh, because the rest of the cluster is unreachable mid-roll, then sends (for example) a SCRAM handshake to a port now serving the Kerberos listener, which is an authentication-protocol mismatch.

`cluster-ip`, `route`, and `ingress` addresses are derived deterministically from resource names, so they hit Failure mode A but not B.

## Motivation

The root cause is an atomicity mismatch.
Listener resources are switched atomically for the whole cluster, but the brokers behind them converge gradually as they restart one by one.
During that window every not-yet-rolled broker advertises a listener it is not yet serving, so clients fail to connect.
Because most of the cluster is unreachable, clients cannot refresh their metadata to recover.
On `nodeport` and `loadbalancer` the same window is what makes a port or address reshuffle dangerous (Failure mode B): with no reachable cluster to refresh from, cached clients keep hitting stale, and possibly cross-wired, endpoints (for example a SASL client reaching a port now serving a different mechanism).
On a real cluster this has caused cluster-wide produce and consume failures for the entire duration of a listener type change and rollback.

Reconciling each broker's listener resources in lockstep with that broker's own restart shrinks the window to one broker at a time.
At every instant each broker is internally consistent, because its resources match the listener it is running.
The change then rolls through the cluster the same safe way a normal restart does, and clients can always refresh metadata against the parts of the cluster that are still up.
This does not remove the window entirely: a client can still hit the single broker that is currently restarting, and bootstrap-only clients depend on the final bootstrap switch (see below).

## Existing workaround

A connectivity change can already be rolled out safely today, with no operator change, by never mutating an in-use listener:

1. Add the new listener on a new port, alongside the existing one.
2. Once the roll is complete, move clients to the new listener.
3. Once clients have moved, remove the old listener.

This avoids both failure modes, because the existing listener and its per-broker resources are never switched while in use.
It is the recommended approach whenever the operator team controls, or can coordinate with, the clients.

This proposal only helps the case where that discipline is not followed and the listener `type` or `port` is changed in place.
Today such an in-place edit turns into a cluster-wide outage for the duration of the roll; the value here is turning that into an ordinary rolling restart, not enabling anything otherwise impossible.

## Proposal

When a reconciliation includes a connectivity-affecting listener change, reconcile the per-broker listener resources as part of the rolling update, broker by broker, instead of all at once before the roll.
The per-broker resources are the broker's `Service`, plus its `Route` or `Ingress` where applicable.

A change is connectivity-affecting when it changes what a client must connect to:

- a listener's `type` changes;
- a listener's `port` changes;
- a listener is added or removed.

All other reconciliations keep the current behaviour unchanged: config-only changes, certificate rotations, image upgrades, scaling, and so on.

### Precondition: the internal listeners are never part of the change

The rolling update's safety checks (the quorum and in-sync-replica `canRoll` checks) depend on the operator's admin client.
That client connects over the internal replication and control-plane listeners (the headless `brokers` service on `REPLICATION_PORT` and `CONTROLPLANE_PORT`), not the external listener being changed:

```java
// KafkaRoller.adminClient(...)
bootstrapHostnames = nodes.stream().filter(NodeRef::broker)
    .map(node -> DnsNameGenerator.podDnsNameWithoutClusterDomain(namespace, KafkaResources.brokersServiceName(cluster), node.podName()) + ":" + KafkaCluster.REPLICATION_PORT)
    .collect(Collectors.joining(","));
```

This is what makes the approach safe: the external listener can be in flux while the operator still reaches the cluster for availability checks.
The coupled path is therefore valid only for external client listeners.
A change to the internal replication or control listener cannot use this mechanism and must fall back to the current behaviour, or be disallowed.
The detector must guard for this.

### Per-broker convergence guard (idempotency)

Reconciliations can be interrupted and re-run, which can leave the cluster half-migrated with some brokers new and some old.
The connectivity-change decision must therefore be evaluated per broker, by asking whether this broker's observed listener resources already match the desired listener config.
It cannot be a single cluster-global flag.
Brokers already at the desired state are skipped, so a re-run resumes the migration instead of re-rolling the whole cluster.
This makes the operation convergent and resumable.

### Per-broker rollout loop

For each broker that is not yet converged, in the order and under the safety gates the existing roll already enforces:

1. Reconcile that broker's listener resources (`Service`, plus `Route`/`Ingress` where applicable) to the desired state, creating, updating, or deleting as needed.
   For types whose externally-visible address is assigned asynchronously, wait until it is available for that broker: the assigned `nodePort` for `nodeport`, the load-balancer ingress IP or hostname for `loadbalancer`, or the admitted host for `route`.
   `cluster-ip` and `ingress` addresses are deterministic and need no wait.
2. Render that broker's configuration with the resulting `advertised.listeners` and apply its `ConfigMap`.
3. Restart the broker so it starts serving the new listener and re-registers its advertised endpoint.
4. Wait until the broker is ready before moving to the next broker.

### Bootstrap resource

The bootstrap resource is shared, one `Service`, `Route`, or `Ingress` for the whole listener, and is reconciled last, after all brokers have rolled.
This avoids switching the shared entry point until the per-broker endpoints behind it are all consistent, and it makes the bootstrap change a single fast operation at the end.
This does not guarantee a reachable bootstrap throughout the roll: if the bootstrap's current state is itself part of the change (for example, mid-rollback it is still the wrong type), clients doing a cold start or metadata refresh through the bootstrap recover only once it flips at the end.
Clients holding cached per-broker metadata are unaffected (see below).

### Availability during a partial roll

A half-rolled cluster does not cause widespread failure for clients with cached metadata; it degrades to an ordinary rolling restart.

For any partition, such a client connects to the leader's currently-advertised address:

- leader on an already-rolled broker → metadata gives the new address, the new resource exists → works;
- leader on a not-yet-rolled broker → metadata gives the old address, the old resource still exists → works;
- leader on the broker currently restarting → brief unavailability while leadership fails over to an in-sync replica, covered by normal client retries (RF ≥ 2).

This also bounds the impact of address churn (Failure mode B).
Because each broker's resource is switched in lockstep with its restart and the rest of the cluster stays reachable, a client targeting a stale `nodeport` or `loadbalancer` address simply refreshes its metadata and connects to that broker's current address.
A reshuffled or cross-wired port affects at most one broker for a moment and is self-healed by a metadata refresh, rather than the sustained cluster-wide outage that occurs today when the whole cluster is switched at once and clients cannot refresh.

Remaining failure modes are the ordinary ones, not new to this change:

- **RF = 1 topics:** partitions led by the broker currently restarting are unavailable until it returns, which is true for any restart.
- **Cold-start or bootstrap-dependent clients:** see the bootstrap note above; they recover when the bootstrap flips at the end.
- **Momentary stale metadata:** a client that cached a broker's address immediately before that broker rolls gets a single failed connection and recovers on the next metadata refresh.

This is different from the current behaviour, where a not-yet-rolled broker's mismatched resources fail all connections to it for the entire roll.

### KRaft assumption

The partial-roll correctness argument depends on KRaft broker-registration semantics:

- Each broker registers its own `advertised.listeners` with the KRaft controller on start-up, and the controller propagates the full set of broker endpoints to all brokers.
  A broker re-registering a new endpoint when it rolls is therefore enough for the rest of the cluster, and for client metadata, to see the change.
  Strimzi does not need to rewrite every broker's configuration up front for the new addresses to become visible, which is what makes incremental per-broker convergence safe.
- Each broker's advertised address depends only on that broker's own resources, so its config can be rendered from its own resources alone, with no cross-broker dependency.
  For `nodeport` the advertised host is resolved by the pod from its scheduled node (via an env var) and only the port comes from the broker's `Service`; for `loadbalancer` the address comes from that broker's LB `Service` ingress; for `cluster-ip` from its per-broker DNS name; for `route` and `ingress` from that broker's `Route` or `Ingress` host.
- Controller-only KRaft nodes have no client listeners and are skipped by the per-broker resource and config steps.

### Implementation sketch

This is an ordering and wiring change in the Cluster Operator, and the building blocks already exist.

**1. Detect connectivity change, per broker.**
Before the roll, for each broker, compare the desired listener config against the observed per-broker resources to decide whether that broker needs a coupled reconcile.
If no broker needs one, the reconciliation keeps the current behaviour exactly (`listeners()` → `perBrokerKafkaConfiguration()` → `podSet()` → `rollingUpdate()`).
The new path is taken only for connectivity changes on external listeners (per the precondition above).

**2. Split `KafkaListenersReconciler.reconcile()` into reusable pieces.**
Today it reconciles the bootstrap plus all per-broker resources and returns one `ReconciliationResult` (with `advertisedHostnames` and `advertisedPorts`, both `Map<Integer, Map<String, String>>`).
Factor out:

- `reconcileSharedPrerequisites()` — listener `Secret`s and certificates, and other non-connectivity, non-bootstrap resources.
- `reconcileBrokerListener(NodeRef)` — reconcile one broker's per-broker resources (`Service`, plus `Route`/`Ingress` where applicable), wait for that broker's asynchronously-assigned address where relevant (`nodePort`, LB ingress, or route host), and return that broker's `advertisedHostnames` and `advertisedPorts` entries.
  This generalizes the existing per-listener-type readiness steps (`nodePortServicesReady`, `loadBalancerServicesReady`, `routesReady`, and so on) to operate on a single broker.
- `reconcileBootstrap()` — the shared bootstrap resource (`Service`/`Route`/`Ingress`) and the final listener status.

**3. Drive the per-broker resource reconcile from the exact pre-restart moment.**
The roll iterates node by node on a single-threaded executor via `maybeRollKafka(...)` → `KafkaRoller`.
A node can be considered many times and deferred (`UnforceableProblem` plus backoff) when `canRoll` fails, so the only correct place to switch a broker's resources is immediately before the pod is actually deleted, right before each `restartAndAwaitReadiness(...)` call, after the availability gate has passed.
There are three such call sites in `restartIfNecessary(...)`:

- the `forceRestart` branch, used for stuck, unresponsive, or old-revision pods, which is not the connectivity-change path (that path is `needsRestart`);
- the normal `needsRestart`/`needsReconfig` branch, reached only after `canRoll(...)` returns true;
- the force-roll-on-error fallback, reached only after `canRoll(..., true, ...)` returns true.

The hook (`reconcileBrokerListener(node)` plus that broker's `ConfigMap` update) must:

- fire only at those points, never when a node is merely considered, otherwise the broker's resources are switched while the still-running broker serves the old listener, which is the exact bug localized to one broker for the duration of a deferral;
- be idempotent, because a node may pass `canRoll` and still be retried;
- run before every restart of a not-yet-converged broker, including the `forceRestart` and force-roll-on-error branches (see below).

A connectivity change is signalled by `podNeedsRestart` returning a new reason (for example `LISTENER_CONFIGURATION_CHANGE`) for brokers that are not yet converged.
This yields `needsRestart = true`, so it is gated by `canRoll`, and it short-circuits the dynamic-reconfiguration path (`advertised.listeners` is not dynamically updatable anyway).

**Handling the `forceRestart` branch.**
Although a connectivity change itself maps to `needsRestart` rather than `forceRestart`, a not-yet-converged broker can still hit the `forceRestart` or force-roll-on-error path for an unrelated reason during the migration, for example a stuck, unresponsive, or old-revision pod.
If the listener hook only ran on the `needsRestart` branch, such a broker would be restarted with its resources and `ConfigMap` still on the old listener and would come back mismatched, converging only on a later reconcile.
The hook must therefore key off whether the broker is not yet converged and run before the restart on all branches.
Because the hook is idempotent and only reconciles that broker's own resources, doing this on the force paths is safe.

**4. Avoid the PodSet "double-roll".**
The advertised host and port hash is folded into the broker config hash and surfaced as a pod annotation (`perBrokerKafkaConfiguration()` → `podSetPodAnnotations()`), and `podSet()` runs before the roll and can itself restart pods when annotations change.
If the new advertised config were applied up front, `podSet()` would roll pods prematurely, with the new annotation but before the per-broker resource hook runs.
The implementation must therefore keep the up-front `podSet()` carrying the old advertised hash and apply each broker's new config and annotation during the roll (step 3), so a broker is rolled exactly once, by `KafkaRoller`, with its resources already switched.
This is the most delicate part of the implementation and must be covered by tests.

**5. Reconcile the bootstrap last**, then populate the listener status.

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
- `KafkaRoller`'s existing controller, quorum, and in-sync-replica safety checks are unchanged and continue to gate each broker's restart.

### Decision: in-roller hook (and the rejected alternative)

This proposal recommends the in-roller hook described above: a single `KafkaRoller` instance for the whole roll, with a strictly-placed, idempotent pre-restart hook that reconciles each broker's resources.

The decision favours safety over a clean boundary.
A single `KafkaRoller` instance holds the cluster-wide view needed to roll safely: controller-first ordering, unready-first ordering, and quorum/ISR `canRoll` checks evaluated across all nodes.
Preserving that global view is what protects availability during the roll, and it is the harder thing to get right.

The rejected alternative is to drive the roll from the listener reconciler by calling `maybeRollKafka(...)` with a single-node set per broker, reconciling that broker's resources between calls.
This keeps resource and `ConfigMap` I/O out of `KafkaRoller` and preserves its clean "pods plus admin client only" boundary.
The downside is that it spins up a fresh roller per broker, which loses the cross-node ordering and quorum/ISR batching, re-creates admin clients on every call, and re-derives ordering the roller otherwise does for free.
Giving up the roller's global safety reasoning to keep a code boundary tidy is the wrong trade for the operator's most safety-critical path, so this variant is rejected.
The cost of the chosen approach, widening `KafkaRoller`'s responsibility, is contained by the strict hook placement and idempotency requirements above and must be covered by tests.

### Rollout and the limits of gating

A feature gate (for example `CoupledListenerRollout`) is the obvious way to ship this, following Strimzi's usual Alpha/Beta/GA lifecycle.

It is worth being honest about how much a gate actually buys here.
The natural off-switch is the connectivity-change detector itself: with no connectivity-affecting change, the reconcile keeps exactly today's flow.
But the change is spread across the reconcile and roll path (`KafkaReconciler` ordering, the `KafkaListenersReconciler` split, and the `KafkaRoller` hook), so the new code sits in the common path even when the gate is off and cannot be cleanly isolated behind it.
That limits how much risk a gate removes, and is one of the reasons this change is hard to justify for the benefit it provides.

### Testing

The proposal is only credible with explicit coverage of the partial-roll states it claims to make safe:

- **Unit tests** for the per-broker convergence detector (mixed or half-migrated state, internal-listener guard) and for the `KafkaRoller` hook placement (fires only pre-restart after `canRoll`, fires on the `forceRestart` and error branches for not-yet-converged brokers, is idempotent under retries, and does not fire for considered-but-deferred nodes).
- **Integration tests** asserting that no PodSet double-roll occurs, so each broker rolls exactly once for a connectivity change.
- **System tests** that perform a listener `type` change and a rollback for each per-broker listener type (`nodeport`, `loadbalancer`, `cluster-ip`, `route`, `ingress`) on a multi-broker cluster, asserting that clients with cached metadata keep producing and consuming throughout the roll, and that an interrupted reconcile resumes rather than re-rolling.

## Affected/not affected projects

This proposal affects the Strimzi Cluster Operator only: `KafkaReconciler` (reconcile ordering), `KafkaListenersReconciler` (split into shared, per-broker, and bootstrap reconciliation), and `KafkaRoller` (a strictly-placed, idempotent pre-restart hook and just-in-time advertised host and port supply).

No CRD or API changes are required.
No other Strimzi projects are affected.
The correctness argument relies on KRaft broker registration (see [KRaft assumption](#kraft-assumption)).

## Compatibility

- **Gateable, but not cleanly:** a feature gate can default the behaviour off, but the new code is spread across the reconcile and roll path and cannot be fully isolated behind it (see [Rollout and the limits of gating](#rollout-and-the-limits-of-gating)).
- **Behaviour is unchanged** for non-connectivity-affecting reconciliations, and for connectivity changes on internal listeners, which fall back to the current behaviour.
- **No API or CRD change.**
- **Reconciliation takes longer** for connectivity-affecting changes, because per-broker resource reconciliation (and waiting for an asynchronously-assigned address) is interleaved with the roll.
  This is most pronounced for `loadbalancer`, where each per-broker LB is provisioned by the cloud (often minutes) and is now serialized across the roll; for large clusters (200+ brokers) the total can be very large.
  Per-broker waits are bounded by `operationTimeoutMs`, but the overall reconcile may approach or exceed the reconciliation interval, so overlap and lock behaviour and progress logging must be considered.
  The migration is resumable: an aborted reconcile continues with the not-yet-converged brokers on the next pass, the same way the operator already resumes any interrupted roll.

## Rejected alternatives

### Roll all resources, then delete all pods at once

Forcing a full simultaneous pod restart (as was done manually to recover the incident) makes the cluster consistent quickly, but it causes a hard total outage during the restart and defeats the purpose of rolling updates.
Rejected.

### Make clients tolerate the mismatch

Relying on client-side retry and metadata-refresh tuning to ride out the window does not work when the resource actively fronts the wrong listener (auth-mechanism mismatch) or the address no longer exists for a sustained period.
The operator must keep server-side resources consistent with the running brokers.
Rejected.

### Pin node ports deterministically

Giving `nodeport` listeners stable, formula-derived node ports (so a type change or rollback never reshuffles ports between listeners) was considered.
It only narrows Failure mode B for one listener type (`nodeport`) and does nothing for Failure mode A or for `loadbalancer`, `cluster-ip`, `route`, and `ingress`, so it is not a general fix.
It is also inflexible: formula-derived ports must fit within the cluster's `service-node-port-range`, can collide with other `NodePort` services, and pin the operator to a fixed allocation scheme.
Per-broker reconciliation addresses the root cause for all listener types instead.
Rejected.
