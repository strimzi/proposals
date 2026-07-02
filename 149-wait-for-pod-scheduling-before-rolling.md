# Wait for all desired pods to be scheduled before starting a rolling update

This proposal adds a new check to the Kafka reconciliation flow.
Before `KafkaRoller` deletes any pod for a rolling update, the operator waits until all pods desired by the cluster's `StrimziPodSets` exist and are scheduled to a node.
This closes a race condition between scale-up and rolling updates.
When both happen in the same reconciliation, a restarted pod can lose its node to a pod that was created moments earlier.
On clusters that use node-local storage, the restarted pod can then stay `Pending` forever.
Every following reconciliation fails until a user intervenes.

## Current situation

A single reconciliation of a `Kafka` custom resource can do two things at once.
It can create new pods (scale-up) and it can roll existing pods, for example because of a configuration change.
Since [PR #10746](https://github.com/strimzi/strimzi-kafka-operator/pull/10746), `KafkaReconciler.podSet()` waits for all pods in `KafkaCluster.addedNodes()` to become ready before `rollingUpdate()` starts `KafkaRoller`.

However, this wait is based on which nodes the current reconciliation thinks it is adding.
It does not look at the actual state of the pods in Kubernetes.
`addedNodes()` comes from `NodeIdAssignment.toBeAdded()`.
That method compares the desired replicas with the node IDs recorded in the `KafkaNodePool` status.
And `updateNodePoolStatuses()` stores the desired node IDs in the status early in the reconciliation, before `podSet()` runs.
As a result, the wait does not work when a reconciliation fails and is retried:

1. Reconciliation A stores the new node IDs in the `KafkaNodePool` status.
   It then fails at a later step, before any pods are created, for example because of a transient error while working with a `Secret`.
2. Reconciliation B builds its model from the updated status, so `addedNodes()` is empty.
   It still creates the missing pods through the `StrimziPodSet` diff, but it does not wait for them.
3. `KafkaRoller` deletes the first pod for the rolling update while the scheduler is still placing the new pods.

For most clusters, the worst outcome is that a pod ends up on a less suitable node.
But when the cluster uses node-local storage, the cluster can end up in a state it cannot recover from on its own.
Local storage provisioners such as `local-path`, OpenEBS LVM, or TopoLVM use `volumeBindingMode: WaitForFirstConsumer`.
This pins each pod to the node where its volume was provisioned.
Such clusters typically also use required pod anti-affinity so that at most one Kafka pod runs on each node.
While a pod is deleted during a rolling update, nothing protects its node.
The anti-affinity rule only considers pods that exist at scheduling time.
And the bound `PersistentVolume` of a pod that is not running does not keep other pods away.
If one of the new pods is scheduled to that node in this window, its own local volume is provisioned there.
The node is now taken for good.
The recreated pod is still pinned to the same node by its existing volume.
It stays `Pending` until a user manually deletes the new pod and its `PersistentVolumeClaim`.

## Motivation

We hit this issue in production with Strimzi 0.47.0.
A single change scaled a broker node pool from 10 to 40 replicas and changed the broker CPU resources at the same time.
The first reconciliation stored the new node IDs in the node pool status and then failed with a transient `Secret` error.
The retry created the 30 new broker pods and started the rolling update within the same second.
One of the new brokers was scheduled to the node of a KRaft controller pod that had just been deleted for the rolling update.
The recreated controller pod was pinned to that node by its metadata volume and stayed `Pending`.
The rolling update timed out with only two of the three controllers running.
Every following reconciliation failed until the pod and its `PersistentVolumeClaim` were deleted manually.

The operator seems to be the best place to address this.
Users cannot easily avoid the trigger.
GitOps tooling applies the scale-up and the configuration change as a single change.
And the retry that reopens the race happens inside the operator, where users do not see it.
Kubernetes cannot solve it either.
The replacement pod does not exist during the deletion window, so there is nothing the scheduler could use to protect the node.
The Kubernetes issues [#128164](https://github.com/kubernetes/kubernetes/issues/128164) (closed without a fix) and [#135771](https://github.com/kubernetes/kubernetes/issues/135771) describe related cases.
The operator is the component that decides when a pod is deleted.
So it can simply avoid deleting pods while other pods are still waiting to be scheduled.

## Proposal

A new wait will be added to `KafkaReconciler.podSet()`, after the `StrimziPodSet` resources are reconciled and before `rollingUpdate()` is called.
The operator will wait until every pod listed in the desired `StrimziPodSets` exists and is scheduled.
A pod counts as scheduled when its `PodScheduled` condition is `True`, which is equivalent to `spec.nodeName` being set.
This check looks at the actual state of the pods.
So it does not matter which reconciliation created them, and it keeps working when a previous reconciliation failed and was retried.
This is the main difference from the existing `addedNodes()` based wait.

Some more details about the behavior:

- The operator waits only for the pods to be scheduled, not for them to be ready.
  The node assignment is decided at scheduling time and usually takes seconds, while readiness can take much longer.
  And a rolling update is often exactly what is needed to fix an unready pod.
- The existing readiness wait for newly added nodes (`waitForNewNodes()`) stays unchanged.
  This proposal only adds a precondition for the rolling update and does not change how scale-up works.
- Pods that are not scheduled but would be restarted by the rolling update anyway are not waited for and are rolled first.
  An example is a pod with an old revision that is `Pending` because of a change to its resource requests.
  Deleting an unscheduled pod does not free up any node, so it cannot trigger the race.
  And `KafkaRoller` already waits for each restarted pod to become ready before it moves to the next one.
  Without this exception, the wait could block forever on a `Pending` pod that only the rolling update can fix.
- If a desired pod is not scheduled within the existing operation timeout, the reconciliation fails before any pod is deleted.
  The cluster keeps running and the `Pending` pod can be investigated.
  This is better than the cluster being stuck in the middle of a rolling update.
- The wait applies to all clusters, not only to those using node-local storage.
  It completes immediately when no new pods were created, so the cost is negligible.
  Trying to detect the storage classes and affinity rules for which the race matters would be complicated and fragile.

No API, CRD, or configuration changes are needed.
This proposal only changes the ordering of steps within the reconciliation.

### What this does not fix

This proposal only stops the operator from racing against itself.
A pod from another Kafka cluster, another namespace, or a completely different workload can still take the node while a pod is deleted during the rolling update.
Closing this window completely would need support from the Kubernetes scheduler, for example extending the `NominatedNodeName` reservation from KEP-5278 to pods with node-local volumes.
That is out of scope of this proposal.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator in `strimzi-kafka-operator`, and there mainly the `KafkaReconciler` class.
Other Strimzi projects, operands, and CRDs are not affected.

## Compatibility

This proposal does not change any APIs, only the behavior of the Cluster Operator.
The change is backwards compatible.
Reconciliations that create new pods and roll existing pods at the same time will start the rolling update a few seconds later.
Reconciliations that only roll pods or only scale up are not delayed.
When the new pods of a cluster cannot be scheduled at all, the reconciliation now fails before the rolling update instead of getting stuck in the middle of it.
This surfaces the same problem, but earlier and without leaving the cluster in a broken state.

## Rejected alternatives

### Fixing only the status update ordering

The failure path described above could also be closed by updating the `KafkaNodePool` status only after the pods were created.
But the wait would still depend on the operator correctly tracking which pods it added.
Any future code path that creates pods without registering them as added would introduce the same race again.
Checking the actual pod state is more robust against such changes.
The status update ordering can still be improved independently of this proposal.

### Pre-pinning the replacement pod to its node

Setting `spec.nodeName` on the replacement pod would bypass the scheduler completely.
A `nodeSelector` with the node's hostname does not reserve anything.
And neither helps, because the race is decided in the window when the replacement pod does not exist yet.

### Setting `nominatedNodeName` on the replacement pod

In its beta scope (Kubernetes 1.35), KEP-5278 only allows the kube-scheduler to write the `nominatedNodeName` field.
And it has the same problem as pre-pinning: during the deletion window there is no pod to set the field on.

### Waiting for readiness instead of scheduling

The operator could wait for all pods to be ready instead of just scheduled.
But a rolling update is often what fixes an unready pod, so this could deadlock the operator.
It would also delay the rolling update by minutes without making it any safer.
The node assignment is already settled once the pod is scheduled.

### Documenting the limitation instead of fixing it

The issue could be documented as a known limitation, advising users not to combine scale-up with changes that trigger a rolling update.
But GitOps tooling applies both as a single change.
The retry that triggers the race happens inside the operator, where users cannot see it.
And the consequences are severe: a quorum member stuck `Pending` and failing reconciliations.
Documentation alone therefore does not seem sufficient.
