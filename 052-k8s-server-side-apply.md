# Kubernetes Server Side Apply

Make the Operators use [Kubernetes Server Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply). This will enable third parties (e.g. Kyverno, Kopf) to have their own labels/annotations on Strimzi owned Kubernetes Resources without Strimzi attempting to remove them.

NOTE: Strimzi will still have ownership over the fields it wants.

## Current situation / cause of proposal

As per [Issue 6938](https://github.com/strimzi/strimzi-kafka-operator/issues/6938) third party Kubernetes operators are having a negative interaction with the existing way Strimzi does Kubernetes applies.
One example is [Kyverno](https://kyverno.io/), which has a policy to add an annotation (`policies.kyverno.io/last-applied-patches:...`) to all resources in a K8s cluster with a `MutatingWebhookConfiguration`. Strimzi detects this label as a resource diff and removes it as it doesn't own it  or it is not included in the [hard-coded list of ignored labels](https://github.com/strimzi/strimzi-kafka-operator/blob/main/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/ResourceDiff.java#L26). This, in turn, triggers the `MutatingWebhookConfiguration` which re-applies the label and the loop continues.

Another example could be a Kyverno policy that adds AWS IRSA annotations to a Service Account (specifically useful for a Kafka Sink/Source Connector). This would allow the annotation to be owned by Kyverno and therefore leave it in place without constantly trying to remove it.

The following log related to the behavior described shows a diff found in the annotations of a `StatefulSet` (before `StrimziPodSet` resources were introduced):

```
2023-03-03 14:25:52 DEBUG ResourceDiff:38 - Reconciliation #178(timer) Kafka(kafka-backup-spike/kafka-backup-spike): StatefulSet kafka-backup-spike-zookeeper differs: {"op":"replace","path":"/metadata/annotations/policies.kyverno.io~1last-applied-patches","value":""}
```

## Motivation

Once this change is made it will allow third party tools within the cluster to interact with Strimzi owned resources without looping.  Strimzi can continue to operate as if it never knew about them.

## Proposal

This proposal makes Kubernetes keep track of each field's "owner" and any changes submitted to those fields will be compared to the remote state before being applied. 

This would let the Kubernetes apiserver handle the three-way diff, updates etc, and allow other controllers, webhooks etc to touch fields, labels, annotations etc the operator doesn't care about.

Conflicts can happen with Server Side Apply as described [here](https://github.com/fabric8io/kubernetes-client/blob/v6.5.1/doc/CHEATSHEET.md#server-side-apply). The first pass of a Server-Side Apply would keep `force = false` and if a conflict were to arise then set `force = true`, which provides better visibility as to when these events may occur.

Kubernetes suggest [using the _force_ method](https://kubernetes.io/docs/reference/using-api/server-side-apply/#conflicts:~:text=It%20is%20strongly%20recommended%20for%20controllers%20to%20always%20%22force%22%20conflicts%2C%20since%20they%20might%20not%20be%20able%20to%20resolve%20or%20act%20on%20these%20conflicts.) to make sure Strimzi have control over the parts they want. As described in the Kubernetes docs: _"This forces the operation to succeed, changes the value of the field, and removes the field from all other managers' entries in managedFields"_.  This obviously means if another party is _forcing_ the same field that they will create a loop of ownership, but this would be rare as operators should only care about their own fields.

Kubernetes recommend [omitting the `GET` call from the Kubernetes API](https://kubernetes.io/docs/reference/using-api/server-side-apply/#upgrading-from-client-side-apply-to-server-side-apply:~:text=the%20object%20doesn%27t%20have%20to%20be%20read%20beforehand), given that we only want to send the API our desired fields and leave it up to the API to solve.

There are two ways to write controllers with Server Side Apply:
- Modification of the existing "Read-Modify-Update" method. This approach involves computing changes in a similar fashion to the current method used by Strimzi. However, the modifications are performed only on the fields owned by Strimzi.
- Reconstructing the entire resource for every reconciliation. This method is the recommended approach. It involves rebuilding the resource completely during each reconciliation of every resource and applying it through Kubernetes management.

The advantages and disadvantages are discussed in the [Kubernetes blog post on Server Side Apply](https://kubernetes.io/blog/2022/10/20/advanced-server-side-apply/#reconstructive-controllers). Notably, performing a NO OP Apply is not very taxing on the API and can significantly reduce the number of calls. **Strimzi will adopt the second method**, which eliminates the need for comparison logic, storing of state, and GET requests. Instead, the resource will be rebuilt every time and applied using Server Side Apply.

Existing Resources (Pods/ConfigMaps/etc) contain a patch strategy in the [upstream docs](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/), which explains how Server Side Apply will merge fields together. This will be unique to every field and if no strategy is listed then it can be assumed the whole field is changed each time.  

[Kubernetes Merge Strategy](https://kubernetes.io/docs/reference/using-api/server-side-apply/#merge-strategy) documentation defines how we can change CRDs to merge how we want them to (`StrimziPodSet`, for example). For reference, there are two main kinds: `atomic` and `map`/`set`/`granular`; with `atomic` set, any change replaces the whole field/list and is recursive downwards; with the others the behaviour is more fine-tuned.

### Code changes

We need to:
* Modify the PATCH/CREATE commands to server-side apply
* Remove the GET commands
* Remove the ResourceDiff (and other similar functions)
* Update tests

---

### **Modify the PATCH/CREATE commands to server-side apply**
Modify [this line](https://github.com/strimzi/strimzi-kafka-operator/blob/18d76bfabcfb9e91c71f9afda60b9dd880797f02/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/AbstractNamespacedResourceOperator.java#LL263C88-L263C102).
From:

```java
....patch(PatchContext.of(PatchType.JSON), desired);
```

To:

```java
....patch(new PatchContext.Builder().withPatchType(PatchType.SERVER_SIDE_APPLY).withFieldManager("strimzi-cluster-operator").withForce(true).build(), desired);
```

And to set without `Force`, as aforementioned for the first pass:

```java
....patch(new PatchContext.Builder().withPatchType(PatchType.SERVER_SIDE_APPLY).withFieldManager("strimzi-cluster-operator").withForce(false).build(), desired);
```

Also, the `StrimziPodSetOperator` patches this method [here](https://github.com/strimzi/strimzi-kafka-operator/blob/main/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/StrimziPodSetOperator.java#L65), which would need updating too. Every create/update call from the existing Operator should become a `serverSideApply` patch call.

And [Kubernetes](https://kubernetes.io/docs/reference/using-api/server-side-apply/#:~:text=through%20declarative%20configurations.-,Clients%20can%20create,-and%20modify%20their)/[Fabric8io](https://github.com/fabric8io/kubernetes-client/blob/master/doc/CHEATSHEET.md#server-side-apply:~:text=For%20any%20create%20or%20update) recommend it is used on CREATE also:
[From](https://github.com/strimzi/strimzi-kafka-operator/blob/18d76bfabcfb9e91c71f9afda60b9dd880797f02/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/AbstractNamespacedResourceOperator.java#L272):

```java
ReconcileResult<T> result = ReconcileResult.created(operation().inNamespace(namespace).resource(desired).create());
```

To set without `Force`, as aforementioned for the first pass:

```java
ReconcileResult<T> result = ReconcileResult.patched(operation().inNamespace(namespace).withName(name).patch(new PatchContext.Builder().withPatchType(PatchType.SERVER_SIDE_APPLY).withFieldManager("strimzi-cluster-operator").withForce(false).build(), desired);
```

To set with `Force`, if there is a conflict for a field and Strimzi should re-take ownership:

```java
ReconcileResult<T> result = ReconcileResult.patched(operation().inNamespace(namespace).withName(name).patch(new PatchContext.Builder().withPatchType(PatchType.SERVER_SIDE_APPLY).withFieldManager("strimzi-cluster-operator").withForce(true).build(), desired);
```

Currently `ReconcileResult.patched` is used for `patch` and `ReconcileResult.created` is used for `create` operations. We will no longer be making an initial GET request to determine the current resource, so will always be using a PATCH request and returning `ReconcileResult.patched`. There is no code specifically using the `ReconcileResult.Created` type to determine behaviour that we can see, it is only used in tests.

It seems there is some [usage](https://github.com/strimzi/strimzi-kafka-operator/blob/834bfc894475a3204029c2abd1426d4200ca3b22/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaExporterReconciler.java#L191C21-L192C46) around `ReconcileResult.noop`, the implementation piece should also address this and make sure the logic continues to work.

According to the [package being used](https://github.com/fabric8io/kubernetes-client/blob/v6.5.1/doc/CHEATSHEET.md#server-side-apply).

NOTE: The versions of fabric8io used by Strimzi [contain Server Side Apply](https://github.com/fabric8io/kubernetes-client/blob/v6.5.1/doc/CHEATSHEET.md#server-side-apply).

---
### **Remove the ResourceDiff**

* The [`ResourceDiff`](https://github.com/strimzi/strimzi-kafka-operator/blob/c3522cf4b17004004a676854d37ba01bb9a44800/operator-common/src/main/java/io/strimzi/operator/common/operator/resource/ResourceDiff.java#L46-L78) is no longer needed, we can generate the whole resource (from Strimzi's perspective) every time and send it as a PATCH with Server-Side Apply.
* This will allow the removal of the `GET` method, as the operator should not care about other fields. It should only know the desired state, and ask Kubernetes API to apply that, letting Kubernetes handle the merge.

The following table shows changes in existing Kubernetes behaviour, with examples:
| current value of field | desired value of field | field owner | requires update? | change in behaviour |
| --------               | --------               | ------      | ---------------- | ------------------- |
| x                      | x                      | Strimzi     | no               | yes                 |
| x                      | y                      | Strimzi     | yes              | no                  |
| x                      | not set                | Strimzi     | yes              | no                  |
| x                      | not set                | Not Strimzi | no               | yes                 |
| not set                | y                      | Strimzi     | yes              | no                  |
| not set                | y                      | Not Strimzi | yes              | no                  |

The key thing to take from this table is that currently Strimzi is trying to make changes to ALL fields (minus short `ignorableFields` list), once Strimzi has knowledge of the fields it owns - we don't want it to try and change the fields it doesn't own. Also Strimzi will be sending a PATCH call for every resource every reconciliation loop, so where no fields have changed it will be a NO-OP with little resource usage.

---
### Modifications to existing behaviours

The only change to existing (non-Strimzi specific) behaviours is that `ResourceDiff` won't care about other fields it chooses not to care about. This does mean that another entity within the cluster can modify things without Strimzi's knowledge, but it shouldn't matter if the `metadata.managedFields` are chosen correctly.

For users:

* If you want to configure something for a single object (e.g. a single Kafka cluster), and it's in scope of the configuration, use that
* If you want to configure something that's not the currently configurable fields, you can use other mechanisms to do so (Kyverno, other controllers), and Strimzi won't interfere
* If you must configure something through some other method, and it's not directly being set in a different way by Strimzi, then Strimzi won't interfere and it'll "just work" (e.g. cattle annotations, pvc resizing annotations, metallb annotations. The user didn't make a choice about this other than the "I want to run Strimzi on this cluster that uses those technologies")

This does not add to the possibility of conflicts that already exist. But it does reduce the likelihood of reaching a conflict and has some mechanisms in place to avoid them once they are found. Conflicts are still possible if 2 controllers are forcing ownership on the same field of the same resource.

### Three-way diff

The live object (A) is stored in the `.metadata`, `.spec` etc. The knowledge of which fields the client previously applied, and indeed all clients (B) is stored in the `.metadata.managedFields`. And the incoming object (C) is supplied by the client. Kubernetes knows which fields were previously set by the client (in the case that the new object no longer specifies some, and they can be removed), and it knows about which client set all the other fields, and will keep them around instead of wiping out all the other labels etc.
Kubernetes knows this information as it is set in the `patch` call. For example:
```java
PatchContext.Builder().withPatchType(PatchType.SERVER_SIDE_APPLY).withFieldManager("strimzi-cluster-operator").withForce(true).build();
```

Currently, if a field that Strimzi owns is unset by a user then Strimzi sets a default value. This behaviour will stay the same as the Operator can choose which fields to default if unset by a user when constructing the resource to send to Kubernetes API, which means that if `affinity` is unset then the Operator can set it before Applying and Kubernetes will know it is owned by Strimzi.

## Compatibility

This proposal aims to be feature-flagged to allow a gradual roll-out of this change to the users and give them time to adjust their own environments to it.
A proposed name is `useServerSideApply`.
It should follow the same protocol as other feature gates, such as [`UseStrimziPodSets`](https://github.com/strimzi/strimzi-kafka-operator/blob/bcecad71b3676194ec7e17f245aef8d23556908a/CHANGELOG.md) (which was deployed in `0.28.0`, moved to beta in `0.30.0` and graduated in `0.35.0`). The suggested protocol here is:
* alpha at `0.37.0`
* beta at `0.40.0`
* graduated at `0.43.0`

### Mock server

We have seen the [Fabric8 Mock Server](https://github.com/fabric8io/mockwebserver) in CRUD mode returns an error when using the `patch()` call (with server-side apply) when attempting to create a new resource, there is an existing issue [here](https://github.com/fabric8io/mockwebserver/issues/76) which would need fixing also.
The Mock Server could be put into Expectation Mode to allow us to override this behaviour, but it would require a lot more maintenance to upkeep.
The limitations will be addressed either in the Fabric8 Mock Server or by moving to another Mock Server as part of the server-side apply implementation.

## Rejected alternatives

We considered adding a new ENV VAR which could take a list of Kubernetes resource paths to be ignored (or labels/annotations directly), but Kubernetes Server Side Apply is designed to do this job and considering it's within Kubernetes it seems wise to take the "out-of-the-box" option.
Another consequence of using ENV VARs would be the overhead of using the existing approach (hard-coded ignore paths) and the new approach (server-side apply), rather than using a single approach. 
