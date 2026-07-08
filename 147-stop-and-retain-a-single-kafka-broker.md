# Stop and retain a single Kafka broker

This proposal adds a way to stop a single Kafka broker — no `Pod`, no Kafka process — while retaining its node ID, storage, and configuration, until it is explicitly started again.
The rest of the cluster keeps reconciling normally.
It is opt-in, set by a single self-contained annotation on the `KafkaNodePool` resource.

## Current situation

A Strimzi-managed Kafka node is either running or removed — there is no "off, but kept" state.
Deleting a broker pod does not help: the `StrimziPodSet` controller recreates it within seconds.
Scaling the node pool down keeps the broker down, but it is a permanent removal: the replicas must be moved away first, the node is unregistered from the KRaft metadata, and its ID and storage are released, so the broker that comes back is a new node with empty disks.
Pausing reconciliation of the whole `Kafka` resource and deleting the pod works, but freezes configuration, certificate renewals, scaling, and rolling updates for every other node too.

## Motivation

The driving use case is planned host maintenance where the same broker must come back with its data intact, and where the pod cannot simply be rescheduled to another Kubernetes node:

- The broker uses local or host-attached storage, so its pod can only run on that one host.
- The host or its disks are themselves the maintenance target, so the broker must be down on that host rather than moved elsewhere.
- A broker must be kept offline for investigation without the operator restarting it.
- A broker must be guaranteed not to take partition leadership or replicate data for a while — for example when its disks are suspect — which only a fully stopped process can ensure.

For brokers on networked storage, draining the Kubernetes node already works — the pod is recreated elsewhere and the volume follows it — so this proposal targets the cases where that is not possible.

## Proposal

Add an annotation on the `KafkaNodePool` resource listing the node IDs in that pool whose pods must not be (re)created:

```
strimzi.io/stopped-nodes="[2]"
```

The value uses the same node-ID list format as the existing `strimzi.io/next-node-ids` and `strimzi.io/remove-node-ids` annotations.
It lives on the `KafkaNodePool` because node IDs are owned by the pool and because it must survive pod deletion — a pod annotation cannot, since stopping deletes the pod.

While a node ID is listed, the `StrimziPodSet` controller will not create a pod for that node, and the operator treats the node as intentionally absent — it is excluded from rolling updates and readiness waits.
The annotation does not delete a running pod, change pool membership, or touch storage.

### Stopping and starting a broker

To stop broker 2 in a pool:

1. Add the node ID to `strimzi.io/stopped-nodes`.
2. Delete the pod (`kubectl delete pod`), or let the host drain evict it.

The broker shuts down gracefully as with any pod deletion, and the pod is not recreated.
Because the annotation never deletes anything, it can be set in advance of a drain.

To start the broker again, remove the node ID from the annotation.
The pod is recreated with the same name, node ID, and `PersistentVolumeClaim`s; the broker runs log recovery, rejoins the ISR, and only replicates the data written while it was down.

A stopped node remains a full pool member: its `PersistentVolumeClaim`s, certificates, and per-broker configuration keep being reconciled, it is not unregistered from the KRaft metadata, and its pod definition stays up to date in the `StrimziPodSet`, so it restarts with the current configuration.

### Mechanism

The change lives in three places:

- `KafkaCluster` resolves the `stopped-nodes` annotations when generating the `StrimziPodSet`s and marks the affected pod entries with an internal annotation.
- The `StrimziPodSet` controller skips creating a marked pod that does not exist; an existing pod is untouched.
- The reconciler excludes stopped nodes from the node set given to `KafkaRoller` and from its cluster-level readiness waits.

The last point is what makes the annotation self-contained.
A stopped pod can never become ready, so without the exclusion the operator would wait for it until the operation timeout and mark the whole `Kafka` resource as not ready on every reconciliation.
There is no configuration in which a user stops a node but wants the operator to wait for it, so this behavior is implied by the annotation rather than delegated to a separate control.

### Scope

Stopping applies to broker-only nodes.
A node ID that resolves to a node with the controller role (whether controller-only or with both the controller and broker roles) is ignored and kept managed, with the reason surfaced in the status condition — stopping a controller shrinks the KRaft quorum and needs dedicated safety checks, which is future work.

The operator does not limit how many brokers can be stopped at once; the consequences are the same as taking those brokers down manually and are the user's responsibility.

### Safety

Stopping a broker takes its replicas offline: partitions it hosts lose a replica, and if one falls below `min.insync.replicas`, producers with `acks=all` fail for it until the broker is back.
The operator does not prevent this — the annotation is an explicit administrative action and the impact is temporary and self-healing on restart.
The existing availability check still defers any roll of another broker that would make a partition unavailable, exactly as today.

Scaling down a pool with a stopped member needs no new logic: the existing [scale-down check](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md) refuses to remove a broker that still holds replicas, and a stopped broker cannot move its replicas away.

Certificates keep being renewed, but the stopped broker is not running to load them, so a stop should not be held across a CA replacement or a cluster upgrade.

The operator never starts a stopped node on its own; starting is always an explicit removal of the ID from the annotation.

### Status and observability

A `.status` condition on the `Kafka` resource lists the stopped node IDs and any ignored (controller or invalid) IDs.
The operator logs a warning while a node is stopped and emits Kubernetes events on stop, start, and refusal.
No new metrics are added; the condition and the annotation are visible to Kube State Metrics for alerting.

No feature gate is used: the feature is opt-in by annotation and an absent annotation leaves behavior exactly as today.

## Affected/not affected projects

Affected: `strimzi-kafka-operator` only — annotation parsing and validation, pod-entry marking in `KafkaCluster`, the skip-creation check in the `StrimziPodSet` controller, the roller and readiness exclusions in the reconciler, and the status condition.
No other Strimzi project is affected.

## Compatibility

Fully opt-in; without the annotation, behavior is unchanged.
An older operator ignores the annotation and recreates the pod, so users must start all stopped nodes before downgrading, and the documentation must call this out.

## Rejected alternatives

- **A `KafkaNodePool.spec.stoppedNodes` field.** The first version of this proposal; rejected in review as too heavy — an annotation in the existing node-ID family is lighter and just as durable.
- **Having the operator delete the pod itself.** Keeping the annotation passive is simpler and safer: setting it has no immediate effect and deletion remains an explicit human or drain action.
- **An idle pod that does not run Kafka.** Wastes the pod's resources and still occupies the host, so the host is not freed for maintenance.
- **Scale down and later scale up.** Permanent removal with full re-replication on return; wrong semantics for "the same broker back".
- **Pausing reconciliation of the whole `Kafka` resource.** Far too coarse for a single-node problem.
- **A separate annotation for the roller and readiness exclusions.** A stopped node must always be excluded from readiness waits or the reconciliation wedges, so a second annotation would be a mandatory incantation rather than a choice; the exclusion is implied by `stopped-nodes` instead.
- **A feature gate.** Not warranted for an opt-in annotation with no change to default behavior.
