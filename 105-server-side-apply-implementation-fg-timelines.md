# Server Side Apply - Implementation Split, Feature Gates, and Timelines

Additional information to the original [Server Side Apply proposal](052-k8s-server-side-apply.md) - describing the split of the implementation, how it will be gated behind feature gates, and what are the planned timelines.

## Current situation

When the original proposal was merged and PR with the implementation was opened, we discovered that there are complications in the matter of the implementation itself and that the scope is too large to fit a single pull request and feature gate.
Due to inactivity on the pull request, we decided to close it - so the Server Side Apply feature wasn't implemented yet.

## Motivation

From time to time, we get question, discussion or issue opened on the operators repository, that mention difficulties when using various technologies together with Strimzi and when the particular technology (other operator) is trying to update Custom Resources or Kubernetes resources with annotations managed by Strimzi - which is the most common case. 
Strimzi reverts this update, but in a while the resource is updated by the other operator again, creating an endless loop of updates.

The Server Side Apply would fix these issues, as it was mentioned in the [original proposal](052-k8s-server-side-apply.md).
Also, based on the comments from the community, it seems this is desired feature that should be implemented in Strimzi.

## Proposal

In order to implement Server Side Apply in Strimzi, we decided to split the implementation into few parts - or phases.
Each phase will be gated by its own feature gate and will have its own graduation timeline.
This will make the implementation easier, community users would be able to provide early feedback on the implementation and issues they might encounter, and it will allow users to use this feature gradually for each group of resources - without need of enabling it for all at once.
Finally, each of the feature gate will be also available in the operator that will manage such resource covered by the feature gate.

### Proposed Feature Gates and Timelines

We will begin by enabling Server Side Apply for the resources that have historically caused the most user issues — particularly those that are frequently updated by other operators — namely: `Service`, `ServiceAccount`, `ConfigMap`, and `PersistentVolumeClaim`.

This initial implementation will be gated under a single feature flag:

| Feature Gate             | Resources Covered                               | Affected Operators | Alpha (opt-in) | Beta (default-on) | GA     |
|--------------------------|-------------------------------------------------|--------------------|----------------|-------------------|--------|
| `UseServerSideApplyCore` | `Service`, `ServiceAccount`, `ConfigMap`, `PVC` | CO                 | 0.48.0         | 0.51.0            | 0.54.0 |

Once this feature gate has been in use for some time, and based on the feedback and stability of the implementation, we plan to extend Server Side Apply support to additional resource types. 
These will be introduced under separate feature gates:

| Feature Gate                   | Resources Covered                                                             | Affected Operators |
|--------------------------------|-------------------------------------------------------------------------------|--------------------|
| `UseServerSideApplyKubeAndOCP` | `ClusterRole`, `Role`, `Secret`, `Route`, and other Kube/OCP types            | CO, UO             |
| `UseServerSideApplyStrimzi`    | All Strimzi custom resources (`Kafka`, `KafkaConnect`, `StrimziPodSet`, etc.) | CO, UO, TO         |

The exact timelines for these additional feature gates will be defined after evaluating the rollout of the initial `UseServerSideApplyCore` gate and assessing user interest and implementation complexity.

**Note:** The release milestones above may change depending on the scheduling of Strimzi 1.0.0. 
However, each feature gate is expected to spend approximately three minor releases in each maturity phase before advancing.

## Affected/not affected projects

The only affected project is Strimzi operators repository. 

## Compatibility

As there is currently no Server Side Apply implementation in Strimzi, and all new logic will be gated behind feature gates, there are no backward compatibility concerns.

## Rejected alternatives

There is one rejected alternative that was described before - implementing Server Side Apply for everything in one PR and one feature gate.
