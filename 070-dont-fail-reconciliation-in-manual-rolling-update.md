# Don't fail reconciliation when Manual Rolling Update fails

This proposal addresses the Strimzi issue [strimzi/strimzi-kafka-operator#9654](https://github.com/strimzi/strimzi-kafka-operator/issues/9654).

## Current situation

Today, we support manual rolling update for following operands:
* ZooKeeper
* Kafka brokers
* Kafka Connect
* Kafka MirrorMaker 2

The manual rolling update is triggered by adding the `strimzi.io/manual-rolling-update="true"` annotation to a `Pod` or `StrimziPodSet` resource.
And it is executed as part of the next reconciliation for a given operand.
When the manual rolling update fails, then it fails the whole reconciliation.

The manual rolling update is done as one of the first steps in the reconciliation process.
The reason for doing it as early as possible is to do the manual rolling update with the original definition of the related Kubernetes resources (e.g. StrimziPodSets, PVCs, Services, ConfigMaps etc.).
If rolling the pod later in the reconciliation, the related Kubernetes resources would have already changed due to unrelated updates to the Strimzi custom resources, which can lead to issues.

## Motivation

There can be many different reasons why the manual rolling update might fail.
For example:
* Due to partition-replicas not being in-sync
* Due to the pod not reaching a Ready state

In some situations, the reason for the manual rolling update failing is a problem that would be fixed later in the reconciliation.
But the operator never gets to it because the manual rolling update fails and that fails the whole reconciliation, so that it doesn't continue.

One such example that we saw multiple times is described in the [strimzi/strimzi-kafka-operator#9654](https://github.com/strimzi/strimzi-kafka-operator/issues/9654) issue:

1. Due to a storage issue, one of the Kafka nodes (node X) is deleted including its PVC and PV
2. At the same time, another Kafka node (node Y) is annotated for manual rolling update (either by the user but possibly also by Drain Cleaner)
3. The StrimziPodSet controller will restart the failed pod X, but without the PVC/PV it will be in a Pending state
4. Next periodical reconciliation starts and tries to roll the annotated pod Y. 
   But the rolling update fails because of the pod X being in Pending state and its partition replicas not being in-sync.
   Rolling the pod Y as requested by the annotation would break availability.
   As a result the manual rolling update fails and the whole reconciliation fails as well.
5. However, the PVC creation step is only after the manual rolling update in the reconciliation process.
   So the PVC is never recreated and the Pending pod X remains stuck.
6. When the next reconciliation happens, the same problem repeats again because these two events block each other.

## Proposal

This proposal suggests to change the way the errors of manual rolling update are handled.
Instead of failing the reconciliation when the manual rolling update fails, we should continue with the reconciliation.
That would allow the operator to continue and possibly fix some of the issues.
For example - in case of an issue such as the one described above - the reconciliation would recreate the missing PVC, which would allow the Pending Pod to start and sync-up the data and later allow the manual rolling update of the annotated Pod.

Proceeding with the reconciliation might cause the related Kubernetes resources to change.
So it in a way goes against the reason why the manual rolling update is done early in the reconciliation process:

> The reason for doing it as early as possible is to do the manual rolling update with the original definition of the related Kubernetes resources (e.g. StrimziPodSets, PVCs, Services, ConfigMaps etc.).
> If rolling the pod later in the reconciliation, the related Kubernetes resources would have already changed due to unrelated updates to the Strimzi custom resources, which can lead to issues.

However, it provides a reasonable compromise in doing the manual rolling update as early as possible as a _best effort_, but not getting stuck with it forever.

This change will be applied to all of the operands supporting manual rolling updates.

### `ContinueReconciliationOnManualRollingUpdateFailure` feature gate

As this proposal changes the behavior of the Strimzi Cluster Operator, the change will be introduced though a feature gate.
The feature gate will be named `ContinueReconciliationOnManualRollingUpdateFailure`.
When this feature gate is enabled, a failure of the manual rolling update will not cause a failure of the whole reconciliation but the reconciliation will continue with a warning instead.
When the feature gate is disabled, the reconciliation will fail with an error as it does today.

This expected roadmap for the feature gate is as follows:
* Introduced in Strimzi 0.41.0 as alpha level feature gate
* Move to _beta_ phase and be enabled by default in Strimzi 0.43.0
* Move to _GA_ phase and be permanently enabled in Strimzi 0.45.0

## Affected projects

This proposal affects only the Strimzi Cluster Operator and the ZooKeeper, Kafka brokers, Kafka Connect and Kafka Mirror Maker 2 operands.

## Backwards compatibility

This proposal does not change any Strimzi APIs but changes the behavior of the Strimzi Cluster Operator.
A feature gate is used to introduce this change to minimize the impact on existing users.

## Rejected alternatives

### Doing the manual rolling update as part of the regular rolling update

One of the considered alternatives was to merge the manual rolling update into the regular rolling update.
That would also address the issue described in the [strimzi/strimzi-kafka-operator#9654](https://github.com/strimzi/strimzi-kafka-operator/issues/9654) issue.
However, it could also create a new problems when an unrelated issue to the related Kubernetes resource (e.g. new load balancer not being provisioned) would block the manual rolling updates needed due to infrastructure disruptions such as node draining etc.
Therefore this alternative was rejected.
