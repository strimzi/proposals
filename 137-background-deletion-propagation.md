# Background deletion propagation for Pods

This proposal introduces a new feature gate to enable background deletion propagation for Pods, ensuring network connectivity is maintained during graceful shutdowns in environments using CNIs like Cilium.

## Current situation

Strimzi currently utilizes different deletion propagation policies across its components.
While the `StrimziPodSetController` already uses the `Background` policy for some operations, other components like `KafkaRoller` and `ManualPodCleaner` rely on the shared `AbstractNamespacedResourceOperator` logic, which defaults to `Foreground` for cascading deletions.
Some components, such as `KafkaConnectRoller`, use non-cascading deletions which map to the `Orphan` policy.

When the `Foreground` policy is applied to Pods (the default for many operations in the operator), Kubernetes ensures that all child resources (dependents with an `ownerReference` pointing to the parent) are deleted before the parent resource itself is fully removed.
While Pods are generally considered leaf nodes in the Kubernetes hierarchy, certain infrastructure components—specifically CNI plugins like Cilium—create child resources (such as `CiliumEndpoint`) that are owned by the Pod.
Under the current `Foreground` policy, these network-related child resources are terminated immediately when the Pod is marked for deletion, which can cut off network access before the containers have finished their shutdown process.

## Motivation

The motivation for this proposal stems from reports by users operating in environments with the Cilium CNI.
When a Pod is deleted using the `Foreground` policy, Cilium removes the `CiliumEndpoint` immediately.
This results in an instant loss of network connectivity for the Kafka broker container, which is often still in the process of performing a graceful shutdown.

Without a working network, the broker is unable to communicate its shutdown status to the controller.
This causes the broker to fail to gracefully relinquish its leadership and heartbeats, leading the cluster to treat the shutdown as an ungraceful crash.
Users have reported that this leads to extended recovery times, often exceeding 10 minutes, as the cluster must re-establish quorum and leadership for the affected partitions upon the broker's restart.

Switching to `Background` deletion propagation allows the Pod object to be removed from the API while leaving its dependents (like the network endpoint) intact until the Pod's containers have fully exited.

## Proposal

The use of background deletion propagation for Pods will be gated behind a feature gate called `UseBackgroundPodDeletion`. 

| Phase | Strimzi versions       | Default state                                          |
|:------|:-----------------------|:-------------------------------------------------------|
| Alpha | 1.1.0 - 1.2.0          | Disabled by default                                    |
| Beta  | 1.3.0 - 1.4.0          | Enabled by default                                     |
| GA    | 1.5.0 and newer         | Enabled by default (without possibility to disable it) |

When this feature gate is enabled, the `PodOperator` will utilize `Background` deletion propagation for Pod deletion operations performed during rolling updates.
By using the `Background` strategy, Kubernetes ensures that the Pod enters the `Terminating` state while leaving child resources—specifically network-related objects like the `CiliumEndpoint`—intact until the Pod's containers have fully exited.

This change directly addresses the issue where brokers lose network connectivity immediately upon deletion, preventing them from:
* Communicating with the KRaft controller to relinquish leadership.
* Sending final heartbeats to avoid being treated as an ungraceful crash.
* Completing the graceful shutdown sequence, which currently leads to 10+ minute recovery times on restart.

If the feature gate is disabled, the operator will continue to use the current `Foreground` deletion behavior to ensure backward compatibility.
This change is primarily implemented in the `PodOperator` used by the `KafkaRoller`, `ManualPodCleaner`, and other components. Note that the `StrimziPodSetController` already uses `Background` deletion and is not affected by this change.
While `Foreground` deletion is generally safer for cascading cleanup of complex resources, `Background` deletion is the appropriate choice for Pods in CNI environments where endpoint lifecycle must match the container lifecycle.

## Affected/not affected projects

The only affected project is the `strimzi-kafka-operator` repository, particularly the following classes:

- `FeatureGates`
- `PodOperator`
- `KafkaRoller`

Other projects in the Strimzi organization are not affected.

## Compatibility

This proposal is fully backwards compatible.
The default behavior of the operator will remain unchanged (using `Foreground` deletion) unless the `UseBackgroundPodDeletion` feature gate is explicitly enabled by the user.

## Rejected alternatives

### Investigated workarounds

In [Issue #12212](https://github.com/strimzi/strimzi-kafka-operator/issues/12212), several workarounds were explored but found insufficient to resolve the network connectivity loss during graceful shutdowns:

*   **Custom `preStop` hooks:** Strimzi does not currently support defining custom lifecycle hooks for Kafka broker containers. Even if supported, a `preStop` hook would not prevent the CNI from deleting the network endpoint if Foreground propagation is used.
*   **Custom images:** Modifying the broker image to delay shutdown does not solve the underlying issue, as the loss of network connectivity is caused by the Kubernetes/CNI deletion propagation logic (specifically the immediate removal of the `CiliumEndpoint`) which happens outside the container's control.
*   **Cilium NetworkPolicies:** These policies control traffic flow but cannot influence the deletion order of the `CiliumEndpoint` or the timing of the datapath teardown.
*   **Pausing reconciliation:** While this stops the operator from performing further actions, it does not solve the problem when a roll is eventually required. Strimzi would still eventually delete the pods using the default `Foreground` propagation.
