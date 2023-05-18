<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Kubernetes Server Side Apply

Make the Topic Operator use [Kubernetes Server Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply). This will enable third parties (e.g. Kyverno, Kopf) to have their own labels/annotations on Strimzi owned Kubernetes Resources without Strimzi attempting to remove them.

NOTE: Strimzi will still have ownership over the fields it wants.

## Current situation

As per [Issue 6938](https://github.com/strimzi/strimzi-kafka-operator/issues/6938) third party Kubernetes operators are having a negative interaction with the existing way Strimzi do Kubernetes applies to it's resources.
An example is [Kyverno](https://kyverno.io/) has a policy in a K8s cluster to add a label to all resources with a MutatingWebhookConfiguration, Strimzi detects this label as a resource diff and removes it as it doesn't own it / not included in [hard-coded list of ignored labels](https://github.com/strimzi/strimzi-kafka-operator/blob/c3522cf4b17004004a676854d37ba01bb9a44800/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/StatefulSetDiff.java#LL28C34-L28C49). This triggers the MutatingWebhookConfiguration which re-applies the label and the loop continues.

## Motivation

Once this change is made (which is relatively simple to implement), it will allow third party tools within the cluster to interact with Strimzi owned resources without looping and Strimzi will be operating as if it never knew about them.

## Proposal

Provide an introduction to the proposal. Use sub sections to call out considerations, possible delivery mechanisms etc.
This proposal makes Kubernetes keep track of each field's "owner" and any changes submitted to those fields will be compared to the remote state before being applied. 

This would let the Kubernetes apiserver handle the three-way diff / updates etc, and allow other controllers / webhooks to touch fields / labels / annotations the operator doesn't care about.

### Three-way diff

The live object (A) is stored in the `.metadata`, `.spec` etc. The knowledge of which fields the client previously applied, and indeed all clients (B) is stored in the `.metadata.managedFields`. And the incoming object (C) is supplied by the client. Kubernetes knows which fields were previously set by the client (in the case that the new object no longer specifies some, and they can be removed), and it knows about which client set all the other fields, and will keep them around instead of wiping out all the other labels etc.

## Compatibility

This proposal aims to be feature-flagged as not to affect existing use of Strimzi, but also allow people to "upgrade" at their own / controlled pace.

## Rejected alternatives

We considered adding a new ENV VAR which could take a list of Kubernetes resource paths to be ignored (or labels/annotations directly), but Kubernetes Server Side Apply is designed to do this job and considering it's within Kubernetes it seems wise to take the "out-of-the-box" option.
