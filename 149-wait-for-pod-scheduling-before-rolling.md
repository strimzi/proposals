# Wait for all desired pods to be scheduled before starting a rolling update

This proposal adds an observation-based scheduling barrier between `StrimziPodSet` reconciliation and `KafkaRoller`: before the operator deletes any pod for a rolling restart, every pod desired by the cluster's `StrimziPodSets` must exist and be scheduled.
It closes a race in which a rolled pod using node-pinned local storage permanently loses its node to a pod created moments earlier, leaving it `Pending` forever and failing every subsequent reconciliation.

## Current situation

A single reconciliation of a `Kafka` custom resource can both create new pods (scale-up) and roll existing ones (revision change).
Since [PR #10746](https://github.com/strimzi/strimzi-kafka-operator/pull/10746), `KafkaReconciler.podSet()` waits for the readiness of all pods in `KafkaCluster.addedNodes()` before `rollingUpdate()` starts `KafkaRoller`.

This guard is derived from intent, not observation: `addedNodes()` comes from `NodeIdAssignment.toBeAdded()`, which compares `KafkaNodePool.status.nodeIds` with the desired replicas, and `updateNodePoolStatuses()` persists the *desired* node IDs early in the reconcile pipeline, before `podSet()` runs.
That makes it non-idempotent across a failed reconciliation:

1. Reconciliation A persists the new node IDs into `KafkaNodePool.status`, then fails at a later stage (for example a transient `Secret` error) before any pod is created.
2. Reconciliation B rebuilds its model from the updated status, so `addedNodes()` is empty; it still creates the missing pods via the `StrimziPodSet` diff, but has nothing to wait for.
3. `KafkaRoller` deletes its first pod while the scheduler is still placing the newly created pods.

For most clusters an unfortunate placement is only a performance nuisance, but with node-local storage it is a deadlock.
Local volumes (`local-path`, OpenEBS LVM, TopoLVM) with `volumeBindingMode: WaitForFirstConsumer` pin each pod to one node, typically combined with hard one-pod-per-host anti-affinity.
While a pod is deleted for a roll it has no claim on its node: anti-affinity only counts pods present at scheduling time, and a bound `PersistentVolume` of a non-running pod repels nothing.
If a new pod lands on the node in that window, its own local volume is provisioned there and the theft is permanent: the rolled pod's replacement is pinned to a taken node and stays `Pending` until a human deletes the squatting pod and its `PersistentVolumeClaim`.

## Motivation

This sequence was hit in production on Strimzi 0.47.0 by a single change that scaled a broker pool from 10 to 40 replicas and changed broker CPU resources.
A reconciliation persisted the new node IDs, failed on a transient `Secret` error, and the retry created the 30 broker pods and started rolling in the same second.
One new broker was scheduled onto the node of the just-deleted KRaft controller pod; the recreated controller was pinned there by its metadata volume and stuck `Pending`, the roll timed out with the quorum at two of three controllers, and every reconciliation failed until manual `PersistentVolumeClaim` and pod deletion.

The operator is the right place to fix this.
Users cannot avoid the trigger: GitOps applies scale-up and revision change as one manifest change, and the race is reopened by operator-internal retries invisible to the user.
Kubernetes cannot fix this side either, because during the deletion window the replacement pod does not exist, so no scheduler state can protect the node (kubernetes/kubernetes [#128164](https://github.com/kubernetes/kubernetes/issues/128164), closed unresolved, and [#135771](https://github.com/kubernetes/kubernetes/issues/135771) track adjacent variants).
Only the component that decides *when to delete* a pod can refuse to open the window while competing pods are unscheduled.

## Proposal

Add a scheduling barrier to `KafkaReconciler.podSet()`, after the `StrimziPodSet` reconciliation and before `rollingUpdate()`.
The barrier waits until every pod listed in the desired `StrimziPodSets` exists and is scheduled, meaning its `PodScheduled` condition is `True` (equivalently, `spec.nodeName` is set).
Because it is derived purely from observed pod state, it holds regardless of which reconciliation created the pods and is idempotent across failed and retried reconciliations, unlike the current `addedNodes()`-based guard.

- The barrier waits for scheduling only, not readiness: node claims are settled at scheduling time (seconds), while readiness can take minutes and a roll is often the remedy for an unready pod.
- The existing `waitForNewNodes()` readiness wait is kept unchanged; this proposal adds a precondition for rolling, not new scale-up semantics.
- Unscheduled pods that the roll itself would restart (for example an old-revision pod left `Pending` by a resource-request change) are exempt and rolled first.
Deleting an unscheduled pod vacates no node, so it cannot open the race window, and `KafkaRoller` already waits for each restarted pod's readiness — hence scheduling — before deleting the next one.
Without this exemption the barrier would block forever on a `Pending` pod whose fix is the roll itself.
- If a desired pod is not scheduled within the existing operation timeout, the reconciliation fails *before* any pod is deleted, leaving the cluster running with a diagnosable `Pending` pod instead of deadlocked mid-roll.
- The barrier applies unconditionally, not only when node-local storage is detected: the wait is cheap (a no-op when no pods were created) and avoids fragile detection of storage classes and affinity rules.

No API, CRD, or configuration change is required; this is an ordering fix in the reconcile flow.

### What this does not fix

A pod belonging to another Kafka cluster, another namespace, or a non-Strimzi workload can still take the node during a roll window.
Closing that general window requires scheduler-side support (for example extending the KEP-5278 `NominatedNodeName` reservation to volume-pinned pods) and is out of scope here.

## Affected/not affected projects

Affected: `strimzi-kafka-operator` cluster operator only (`KafkaReconciler`).
Not affected: all other Strimzi projects, operands, and CRDs.

## Compatibility

The change is behavioural only and backwards compatible.
Reconciliations that combine pod creation with a rolling update start the roll a few seconds later; roll-only or scale-only reconciliations pass the barrier immediately.
A cluster whose new pods cannot be scheduled at all now fails reconciliation before rolling instead of deadlocking after, surfacing the same problem earlier and with less damage.

## Rejected alternatives

### Fix only the status-ordering bug in the existing guard

Persisting the node pool status after pod creation would close the specific failure path, but the guard would remain intent-based and any future path that creates pods without registering them as "added" silently reopens the race.
The observation-based barrier is robust to all such paths; the status-ordering cleanup can still be done independently.

### Pre-pin the replacement pod to its node

`spec.nodeName` bypasses the scheduler entirely and a hostname `nodeSelector` reserves nothing; neither helps, because the race is won during the window when the replacement pod does not exist yet.

### Set `nominatedNodeName` on the replacement pod

KEP-5278's beta scope (Kubernetes v1.35) restricts writes to the kube-scheduler, and it suffers the same non-existence window as pre-pinning.

### Wait for readiness instead of scheduling

Gating all rolls on full readiness could deadlock the operator (a roll is often the remedy for an unready pod) and adds minutes of latency without adding safety over the scheduled state.

### Document the limitation instead of fixing it

GitOps tooling applies scale-up and revision change as one change, the race is reopened by retries the user never sees, and the failure mode (a deadlocked quorum member) is severe.
