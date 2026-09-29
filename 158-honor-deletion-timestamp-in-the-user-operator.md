# Honor the `deletionTimestamp` on `KafkaUser` resources

This proposal suggests that the User Operator stops reconciling a `KafkaUser` once Kubernetes marks the resource for deletion, and that it reports this state in the resource status.
It describes the _middle state_ outlined by the maintainers when [strimzi-kafka-operator#12835](https://github.com/strimzi/strimzi-kafka-operator/issues/12835) was discussed on the community call on 9.7.2026 - while the deletion of a `KafkaUser` is pending, the operator neither recreates nor deletes anything.

## Current situation

The User Operator watches `KafkaUser` resources and the `Secrets` it generates for them.
Every event from these two informers, as well as every periodic reconciliation, enqueues the affected `KafkaUser` for reconciliation.
`KafkaUserOperator#reconcile` then decides what to do based on a single condition - when the `KafkaUser` is present in the informer cache, the user is created or updated, and when it is not present, the user is deleted from Kafka together with its `Secret`.

The `metadata.deletionTimestamp` field is not taken into account anywhere in this flow.
A `KafkaUser` which is marked for deletion, but is still present in the Kubernetes API because it has a pending finalizer, is therefore reconciled as a regular resource.

The `Secret` generated for a `KafkaUser` has an owner reference pointing to the `KafkaUser` with `blockOwnerDeletion: true`.
Together with the missing `deletionTimestamp` check, this can delay the foreground cascading deletion of a `KafkaUser`:

1. The `KafkaUser` is deleted with foreground propagation - for example with `kubectl delete kafkauser my-user --cascade=foreground`, or by a GitOps tool such as Argo CD, which uses foreground deletion for the resources it manages.
2. Kubernetes sets `metadata.deletionTimestamp` and adds the `foregroundDeletion` finalizer to the `KafkaUser`.
3. The garbage collector deletes the dependent `Secret` and waits for all dependents which block the owner deletion to disappear before it removes the finalizer.
4. The `Secret` informer receives the `DELETED` event, the operator reconciles the `KafkaUser`, which still exists, and recreates the `Secret`.
5. The garbage collector deletes the recreated `Secret` and removes the `foregroundDeletion` finalizer once no dependent blocks the deletion anymore.

The fourth and the fifth step are a race between the garbage collector and the User Operator.
In most cases the garbage collector removes the finalizer before the operator recreates the `Secret`, the `KafkaUser` is deleted, and the whole deletion completes without anyone noticing.
This is why the flow described above usually cannot be reproduced on a cluster with a small number of `KafkaUser` resources, and it should not be understood as a set of steps which reproduce the problem on demand.

When the operator wins the race, the recreated `Secret` becomes a new dependent which blocks the deletion of the `KafkaUser` and the garbage collector has to delete it again.
Every such round delays the deletion of the `KafkaUser` and the last two steps can repeat several times.
This has been observed on an Amazon EKS cluster where more than a hundred `KafkaUser` resources are deleted at the same time, which slows down both the operator and the Kubernetes API.
It happens once or twice a month in that environment.
The following log shows two rounds of the cycle, three seconds apart:

```
2026-09-22 22:00:02 INFO  UserController:146 - Secret my-test-user in namespace kafka was DELETED
2026-09-22 22:00:02 INFO  UserControllerLoop:102 - Reconciliation #426810(watch) KafkaUser(kafka/my-test-user): KafkaUser will be reconciled
2026-09-22 22:00:02 INFO  UserController:146 - Secret my-test-user in namespace kafka was ADDED
2026-09-22 22:00:02 INFO  UserControllerLoop:123 - Reconciliation #426810(watch) KafkaUser(kafka/my-test-user): reconciled
2026-09-22 22:00:02 INFO  UserControllerLoop:102 - Reconciliation #426824(watch) KafkaUser(kafka/my-test-user): KafkaUser will be reconciled
2026-09-22 22:00:03 INFO  UserControllerLoop:123 - Reconciliation #426824(watch) KafkaUser(kafka/my-test-user): reconciled
2026-09-22 22:00:05 INFO  UserController:146 - Secret my-test-user in namespace kafka was DELETED
2026-09-22 22:00:05 INFO  UserControllerLoop:102 - Reconciliation #426971(watch) KafkaUser(kafka/my-test-user): KafkaUser will be reconciled
2026-09-22 22:00:05 INFO  UserController:146 - Secret my-test-user in namespace kafka was ADDED
```

How many rounds happen, and whether any happens at all, depends on the timing and on the load of the cluster.
What happens every time, and what this proposal is about, is that the operator fully reconciles a `KafkaUser` which is already being deleted.

The same reconciliation happens with any other finalizer which delays the deletion of the `KafkaUser`, no matter whether it was added by a GitOps tool, by an admission webhook or by the user directly.
With such a finalizer, the `KafkaUser` stays in the deleting state until the finalizer is removed, and the operator recreates the `Secret` and keeps reconciling the resource for the whole time.
With the default background propagation, the `KafkaUser` is removed from the Kubernetes API immediately, so the current behaviour is not affected by this problem.

For a `KafkaUser` with `type: scram-sha-512` and a generated password, every recreation of the `Secret` also generates a new password and updates the SCRAM credentials in Kafka.
Clients which still use the old password stop working, even though the deletion which affects them has not completed and can stay blocked by the finalizer for an arbitrarily long time.

## Motivation

Foreground deletion is a standard Kubernetes deletion mode and is used by common tooling, but for a `KafkaUser` it competes with the operator, which keeps recreating the `Secret` the garbage collector is waiting for.
While the resource is in the deleting state, the operator keeps reconciling it, and nothing in the resource explains why.

Reconciling a resource which the API server already accepted for deletion produces work which will be thrown away.
In the case of a generated SCRAM-SHA-512 password, it is not only wasted work - it rotates the credentials of a user which is on its way out, and breaks the clients which still use them.

Skipping the reconciliation of a resource which is being deleted also keeps the semantics of finalizers intact.
A finalizer is a mechanism to block the deletion of a resource, and users rely on it to protect a `KafkaUser` from an accidental deletion.
In the proposed state, nothing is deleted early, so a finalizer still protects the user in Kafka.
But nothing is recreated either, so the deletion is not fighting the operator while it is pending.

Once the `deletionTimestamp` is set, the resource cannot return to a normal state.
The only way out of this state is the deletion of the resource, which happens as soon as the last finalizer is removed.
Skipping the reconciliation therefore cannot leave a `KafkaUser` in an in-between state for any longer than its finalizers keep it there.

## Proposal

The User Operator will treat a `KafkaUser` with a non-`null` `metadata.deletionTimestamp` as a resource which is being deleted and will skip the create-or-update part of the reconciliation.

The check will be added to `UserControllerLoop#reconcile`, next to the existing check for the paused reconciliation and before `KafkaUserOperator#reconcile` is called.
The `deletionTimestamp` check will be evaluated first, because a resource which is being deleted is being deleted regardless of whether its reconciliation is also paused.

In this state, the operator will do the following:

- It will not create or update anything - the `Secret` is not created or recreated, and the SCRAM-SHA-512 credentials, ACLs and quotas in Kafka are left untouched.
- It will not delete anything - neither the remaining `Secret` nor the user configuration in Kafka is removed.
- It will set a condition in the `KafkaUser` status to make the state visible to the user.
- It will count the reconciliation as successful in the `strimzi_reconciliations_successful_total` metric, in the same way as it does for a paused resource, because nothing failed.
- It will log a `WARN` message on every reconciliation, because a `KafkaUser` which stays in this state over several reconciliations is not an expected situation and should be visible in the logs.

The deletion itself is not changed by this proposal.
It is still triggered only when the `KafkaUser` is gone from the Kubernetes API, which is the existing `kafkaUser == null` branch in `KafkaUserOperator#reconcile`.
For the flow described above, this means that the garbage collector is now able to finish the deletion of the `Secret` and to remove the `foregroundDeletion` finalizer.
The `KafkaUser` is then deleted, and the following reconciliation removes the user configuration from Kafka as it does today.

The periodic reconciliation will keep enqueuing a `KafkaUser` which is being deleted, and every such reconciliation will hit the new branch and do nothing.

### Status condition

The status of a `KafkaUser` which is being deleted will use the existing `NotReady` condition with a new reason:

```yaml
status:
  conditions:
    - type: NotReady
      status: "True"
      reason: Deleting
      message: The resource is being deleted and will not be reconciled.
      lastTransitionTime: "2026-09-08T10:12:34.123456789Z"
  observedGeneration: 3
```

This follows the way the User Operator already reports any other non-ready state, where `StatusUtils#setStatusConditionAndObservedGeneration` builds a `NotReady` condition with the status `True` and a reason.
A resource which is being deleted is not ready, so it does not need a condition type of its own.

The condition replaces the `Ready` and `ReconciliationPaused` conditions in the list.
The `Ready` column of `kubectl get kafkauser` selects the condition with the type `Ready`, so it shows no value for such a resource, which is the same behaviour as for a resource with a paused or a failed reconciliation.

The `username` and `secret` fields are left out of the status, because the operator did not reconcile the resource and has no new information to report.

The status of a resource which is being deleted can still be updated, because Kubernetes only restricts changes to the object itself and to its finalizers, not to its status subresource.
The existing `StatusDiff` check in `UserControllerLoop#maybeUpdateStatus` makes sure that the status is written only when it actually changes, so a `KafkaUser` which waits on a finalizer for a long time is not updated on every reconciliation.

### Metrics

This proposal does not add any new metric.
The state is visible through the status condition, and the existing `strimzi_resources_paused` metric is deliberately not reused for it, because it tracks resources paused with the `strimzi.io/pause-reconciliation` annotation.

### Documentation

The new behaviour will be described in the documentation as part of the implementation, together with a note that the credentials of a `KafkaUser` remain valid in Kafka until the deletion of the resource completes.

### Testing

The implementation will be covered by unit tests in the User Operator, which verify that no `Secret` is created, that no ACLs, quotas or SCRAM credentials are reconciled, and that the status condition is set.
No system test is planned, because the only part which the unit tests do not cover is the behaviour of the Kubernetes garbage collector, which is not code owned by this project.

## Affected/not affected projects

The only affected project is the `strimzi-kafka-operator` repository, and within it only the User Operator and its documentation.

No other project in the Strimzi organisation is affected.
Applying the same pattern to other custom resources, such as `KafkaTopic` or the resources handled by the Cluster Operator, is out of the scope of this proposal.

## Compatibility

The proposal does not change the `KafkaUser` API.
The `metadata.deletionTimestamp` field is a standard Kubernetes field, and the status reuses the existing `NotReady` condition with a new reason, so no change of the CRD is needed.

The behaviour changes only for a `KafkaUser` which has a `deletionTimestamp` and is still present in the Kubernetes API, which happens only when a finalizer delays its deletion.
Deletions without a finalizer, which is the default, are not affected at all.

For users who use a finalizer to protect a `KafkaUser` from deletion, the difference is that the `Secret` is no longer recreated while the deletion is pending, and that the SCRAM-SHA-512 password of such a user is no longer rotated.
The credentials which existing clients hold therefore keep working until the deletion completes, and the user configuration in Kafka is removed only after the resource is gone, as it is today.

## Rejected alternatives

### Deleting the user when the `deletionTimestamp` is set

The operator could run its regular deletion logic as soon as the `deletionTimestamp` is set, and remove the `Secret` and the user configuration from Kafka while the resource is still present.
This was rejected because it defeats the purpose of finalizers.
The custom resource would survive, but everything it describes would be gone, which breaks the workflows of users who use finalizers to protect a `KafkaUser` from an accidental deletion.

### Adding a Strimzi finalizer to `KafkaUser`

The operator could add its own finalizer to every `KafkaUser`, clean up Kafka when the resource is being deleted and remove the finalizer afterwards, optionally controlled by an environment variable such as `STRIMZI_USE_FINALIZERS`.
This was rejected for two reasons.

First, it does not solve the reported problem, which is caused by reconciling a resource that is already terminating, and which would happen with a Strimzi finalizer in place as well.

Second, the User Operator does not need finalizers to guarantee the cleanup.
It treats Kubernetes as the single source of truth and, on every periodic reconciliation, `KafkaUserOperator#getAllUsers` lists the users which exist in Kafka as SCRAM credentials, ACLs or quotas, and reconciles them together with the users defined as custom resources.
Any user in Kafka without a matching `KafkaUser` resource is removed by this mechanism, which also covers the users deleted while the operator was down.
Adding finalizers would only add failure modes, such as resources which cannot be deleted when the operator is not running or when its RBAC rules change.

### Skipping the reconciliation without reporting it

The original fix proposed in the issue only skipped the reconciliation and logged an `INFO` message.
This was rejected because the `KafkaUser` would keep its last status, and would still report itself as ready while its `Secret` is missing, with no indication of why nothing is happening.

### Adding a dedicated `Deleting` condition type

The state could be reported with a new condition type, such as `Deleting`, instead of the existing `NotReady` condition.
This was rejected because the User Operator reports every other non-ready state with the `NotReady` condition and a reason, and because the `Ready` column of `kubectl get kafkauser` shows no value in both cases.
A new condition type would therefore add another type to the API without any benefit for the user.

### Reusing the `ReconciliationPaused` condition

The state could be reported with the existing `ReconciliationPaused` condition instead of a new condition type.
This was rejected because it would conflate two different states - a reconciliation paused with the `strimzi.io/pause-reconciliation` annotation, which the user can resume, and a deletion in progress, which cannot be undone.

### Removing the owner reference from the user `Secret`

The operator could stop setting the owner reference on the `Secret` and delete the `Secret` itself, which would prevent the garbage collector from deleting it during a foreground deletion.
This was rejected because the cleanup of such a `Secret` would have to be guaranteed by a finalizer, with all the problems described above, and because GitOps tools would treat a `Secret` without an owner reference as an orphaned resource and try to prune it.

### Documenting foreground deletion as unsupported

The current behaviour could be kept and the documentation could state that a `KafkaUser` does not support foreground deletion.
This was rejected because the failure mode is silent - the operator keeps reconciling a resource which is being deleted, the credentials of the user are rotated in the meantime, and the deletion can be delayed - while standard Kubernetes tooling and GitOps tools use foreground deletion by default in many setups.
