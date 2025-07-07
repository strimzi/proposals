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

### Proposed Feature Gates and timelines

| Feature Gate                | Resources that will be covered                                               | Affected operators | Alpha (needs to be enabled) | Beta (enabled by default) | GA     |
|-----------------------------|------------------------------------------------------------------------------|--------------------|-----------------------------|---------------------------|--------|
| `UseServerSideApplyCore`    | `Service`, `ServiceAccount`, `ConfigMap`, `PVC`                              | CO                 | 0.47.0                      | 0.49.0                    | 0.51.0 |
| `UseServerSideApplyRBAC`    | `ClusterRole`, `Role`, `ClusterRoleBinding`, `RoleBinding`                   | CO                 | 0.48.0                      | 0.50.0                    | 0.52.0 |
| `UseServerSideApplyKube`    | `Secret`, `NetworkPolicy`, `Ingress` etc.                                    | CO, UO             | 0.48.0                      | 0.50.0                    | 0.52.0 |
| `UseServerSideApplyOCP`     | `Route`, `BuildConfig`, `ImageStream`                                        | CO                 | 0.48.0                      | 0.50.0                    | 0.52.0 |
| `UseServerSideApplyStrimzi` | All Strimzi custom resources - `Kafka`, `KafkaConnect`, `StrimziPodSet` etc. | CO, UO, TO         | 0.49.0                      | 0.51.0                    | 0.53.0 |

We start with the resources that seem to be the most problematic for users - as they are frequently updated by other operators - `Service`, `ServiceAccount`, `ConfigMap`, `PVC`.
Then we will move to RBAC, Openshift-only resources and rest of the Kubernetes resources.
Finally, we will implement it for Strimzi Custom Resources.

Most of the feature gates will be usable only by the Cluster Operator, but `UseServerSideApplyKube` will be available for User Operator (because of the `Secret`), and `UseServerSideApplyStrimzi` will be available for all the operators.

The release versions when the particular feature gate will graduate can differ - depending on the release of Strimzi version 1.0.0.
But it should be 2 minor releases before graduating the feature gate further.

## Affected/not affected projects

The only affected project is Strimzi operators repository. 

## Compatibility

As there is currently no Server Side Apply implementation in Strimzi, and all new logic will be gated behind feature gates, there are no backward compatibility concerns.

## Rejected alternatives

There is one rejected alternative that was described before - implementing Server Side Apply for everything in one PR and one feature gate.
