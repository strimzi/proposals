<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Kubernetes Server Side Apply

Make the Operators use [Kubernetes Server Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply). This will enable third parties (e.g. Kyverno, Kopf) to have their own labels/annotations on Strimzi owned Kubernetes Resources without Strimzi attempting to remove them.

NOTE: Strimzi will still have ownership over the fields it wants.

## Current situation / cause of proposal

As per [Issue 6938](https://github.com/strimzi/strimzi-kafka-operator/issues/6938) third party Kubernetes operators are having a negative interaction with the existing way Strimzi does Kubernetes applies.
An example is [Kyverno](https://kyverno.io/) has a policy in a K8s cluster to add a label to all resources with a MutatingWebhookConfiguration, Strimzi detects this label as a resource diff and removes it as it doesn't own it / not included in [hard-coded list of ignored labels](https://github.com/strimzi/strimzi-kafka-operator/blob/c3522cf4b17004004a676854d37ba01bb9a44800/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/StatefulSetDiff.java#LL28C34-L28C49). This triggers the MutatingWebhookConfiguration which re-applies the label and the loop continues.

The precise log we saw was: 

```
2023-03-03 14:25:52 DEBUG ResourceDiff:38 - Reconciliation #178(timer) Kafka(kafka-backup-spike/kafka-backup-spike): StatefulSet kafka-backup-spike-zookeeper differs: {"op":"replace","path":"/metadata/annotations/policies.kyverno.io~1last-applied-patches","value":""}
```

This shows a diff found in the annotations of a `StatefulSet` (this was before StrimziPodSets) which generates the behaviour desribed above.

## Motivation

Once this change is made (which is relatively simple to implement), it will allow third party tools within the cluster to interact with Strimzi owned resources without looping and Strimzi will be operating as if it never knew about them.

## Proposal

This proposal makes Kubernetes keep track of each field's "owner" and any changes submitted to those fields will be compared to the remote state before being applied. 

This would let the Kubernetes apiserver handle the three-way diff, updates etc, and allow other controllers, webhooks etc to touch fields, labels, annotations etc the operator doesn't care about.

Conflicts can happen with Server Side Apply as described [here](https://github.com/fabric8io/kubernetes-client/blob/v6.5.1/doc/CHEATSHEET.md#server-side-apply), it is open to choose the method that Strimzi would use for this.

[Kubernetes suggest](https://kubernetes.io/docs/reference/using-api/server-side-apply/#conflicts:~:text=It%20is%20strongly%20recommended%20for%20controllers%20to%20always%20%22force%22%20conflicts%2C%20since%20they%20might%20not%20be%20able%20to%20resolve%20or%20act%20on%20these%20conflicts.) using the above _force_ method to make sure Strimzi have control over the parts they want. As described in the Kubernetes docs _"This forces the operation to succeed, changes the value of the field, and removes the field from all other managers' entries in managedFields"_, this obviously means if another party is _forcing_ the same field that they will create a loop of ownership but this would be rare as operators should only care about their own fields.

It appears as simple as modifying [this line](https://github.com/strimzi/strimzi-kafka-operator/blob/18d76bfabcfb9e91c71f9afda60b9dd880797f02/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/AbstractNamespacedResourceOperator.java#LL263C88-L263C102).
From:

```
....patch(PatchContext.of(PatchType.JSON), desired);
```

To:

<!-- Note: change the below examples to serverSideApply once we know how -->
```
....patch(new PatchContext.Builder().withPatchType(PatchType.SERVER_SIDE_APPLY).withForce(true).build(), desired);
```

Also, the `StrimziPodSetOperator` patches this method [here](https://github.com/strimzi/strimzi-kafka-operator/blob/main/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/StrimziPodSetOperator.java#L65), this would need updating too. 
It would also need confirming that all resource creates/updates go through `serverSideApply`, not something that should be done in this proposal.

And Kubernetes recommend it is used on CREATE also:
[From](https://github.com/strimzi/strimzi-kafka-operator/blob/18d76bfabcfb9e91c71f9afda60b9dd880797f02/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/AbstractNamespacedResourceOperator.java#L272):

```
ReconcileResult<T> result = ReconcileResult.created(operation().inNamespace(namespace).resource(desired).create());
```

To:

```
ReconcileResult<T> result = ReconcileResult.patched(operation().inNamespace(namespace).withName(name).patch(new PatchContext.Builder().withPatchType(PatchType.SERVER_SIDE_APPLY).withForce(true).build(), desired);
```

According to the [package being used](https://github.com/fabric8io/kubernetes-client/blob/v6.5.1/doc/CHEATSHEET.md#server-side-apply).

NOTE: The version of fabric8io being used is [6.6.1](https://github.com/strimzi/strimzi-kafka-operator/blob/18d76bfabcfb9e91c71f9afda60b9dd880797f02/pom.xml#L106) which [contains Server Side Apply](https://github.com/fabric8io/kubernetes-client/blob/v6.5.1/doc/CHEATSHEET.md#server-side-apply).

The operators will need modifying in terms of change discovery so that they only check fields that are owned by themselves, `metadata.managedFields`.

The [`ResourceDiff`](https://github.com/strimzi/strimzi-kafka-operator/blob/c3522cf4b17004004a676854d37ba01bb9a44800/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/ResourceDiff.java#L46-L78) will need to filter only the parts of the resource that Strimzi owns as to not trigger a reconcile loop for externally owned resource paths.
This will allow the removal of `ignorableFields` throughout the codebase as paths are being selected now, rather than filtering out fields.

It appears [this is a place](https://github.com/strimzi/strimzi-kafka-operator/blob/18d76bfabcfb9e91c71f9afda60b9dd880797f02/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/ServiceOperator.java#L123) that the `desired` state imports existing external annotations, this would need to change to not include existing external annotations within the `desired` state as then Kubernetes would create an ownership conflict with Server Side Apply.


### Modifications to existing behaviours

The only change to existing (non-Strimzi specific) behaviours is that Strimzi won't care about other fields it chooses not to care about. This does mean that another entity within the cluster can modify things without Strimzi's knowledge, but it shouldn't matter if the `metadata.managedFields` are chosen correctly.

I do not know the codebase for the strimzi kafka operator so cannot comment on specific behaviours that might change, as this is just a proposal I suspect that is better placed in the eventual PR if this proposal gets accepted.

### Three-way diff

The live object (A) is stored in the `.metadata`, `.spec` etc. The knowledge of which fields the client previously applied, and indeed all clients (B) is stored in the `.metadata.managedFields`. And the incoming object (C) is supplied by the client. Kubernetes knows which fields were previously set by the client (in the case that the new object no longer specifies some, and they can be removed), and it knows about which client set all the other fields, and will keep them around instead of wiping out all the other labels etc.

## Compatibility

This proposal aims to be feature-flagged as not to affect existing use of Strimzi, but also allow people to "upgrade" at their own / controlled pace.
A proposed name is `useServerSideApply`.
It should follow the same protocol as other feature gates, such as [`UseStrimziPodSets`](https://github.com/strimzi/strimzi-kafka-operator/blob/bcecad71b3676194ec7e17f245aef8d23556908a/CHANGELOG.md) (which was deployed in `0.28.0`, moved to beta in `0.30.0` and graduated in `0.35.0`).

## Rejected alternatives

We considered adding a new ENV VAR which could take a list of Kubernetes resource paths to be ignored (or labels/annotations directly), but Kubernetes Server Side Apply is designed to do this job and considering it's within Kubernetes it seems wise to take the "out-of-the-box" option.
