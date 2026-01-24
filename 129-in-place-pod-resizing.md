# In-place Pod resizing

Kubernetes 1.35 graduated the in-place Pod resizing feature to stable ([blog post](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga)).
In-place Pod resizing allows updating the Pod resource requirements dynamically without restarting the Pod or the container(s) inside it.
This proposal covers how to integrate and support this feature in Strimzi.

## Motivation

In-place resizing helps reduce the number of Kafka node restarts.
While for Java-based applications such as Apache Kafka the benefit of dynamically resizing memory requests and limits might be limited, it works well for CPU.
Support for in-place resizing might also help to better leverage Vertical Pod Autoscaling.

## How in-place Pod resizing works in Kubernetes

In-place Pod resizing is done through the Kubernetes API using a special `resize` sub-resource.
It allows dynamically changing resource requirements of all regular containers (resource requirements cannot be changed for init containers).
It allows increasing or decreasing the CPU and memory limits and requests.

The API call to the `resize` sub-resource changes the resource configuration in the Pod specification.
This instructs Kubernetes to resize the container(s).
The Pod status section indicates whether the resize was successful or not.
There are three general error states when the resizing cannot be applied:
* `Infeasible` resizing state indicating that the requested resource requirements are impossible on a given Kubernetes worker node.
  For example, when requesting more memory or CPU than the worker node's capacity allows.
  Users can typically react to this state by either rolling the Pod and having it scheduled on another node with sufficient capacity.
* `Deferred` resizing state indicates that the resource request could be feasible on a given Kubernetes worker node but is currently not possible due to available resources.
  It might, however, be possible in the future, for example once some other Pods are deleted from the worker node.
  Users can react to this state by either waiting until the resizing is possible or restarting a Pod and having it scheduled on another worker node with sufficient capacity.
* Pod resizing `Error` indicates that the requested resizing failed.
  It could fail, for example, when the requested reduction of the memory requirements is not possible because the container is consuming more memory than the new memory limit allows.

_Note: When the resizing is unsuccessful and ends in one of the 3 states described above, the Pod continues to run with the original resource configuration._

In-place Pod resizing does not allow removing resource limits (or requests when limits are not defined from the beginning), only reconfiguring them.
If a user wants to remove limits completely from the container(s), the Pod needs to be restarted.

## Proposal

### Enabling the in-place resizing

In-place resizing will be disabled by default.
Users will be able to enable it on a per-resource basis using the `strimzi.io/in-place-resizing: "true"` annotation set on the `Kafka`, `KafkaConnect`, or `KafkaMirrorMaker` resource.

Enabling this through an annotation on the custom resource was chosen because:
* This feature depends on the Kubernetes version, so a feature gate with a fixed graduation plan is not suitable for this feature.
* This feature might be very useful for some use-cases and situations and not so useful in others.
  Enabling it through annotation fits this purpose.
  A feature gate would end up with the feature being enabled by default which is not desired.
  A cluster operator configuration option would allow enabling or disabling the feature, but for all operands.
* The annotation is not part of the Strimzi CRD API itself.
  We can easily change it based on the feedback we receive.

### Implementation

This proposal suggests support for in-place resizing for Kafka, Kafka Connect, and Kafka Mirror Maker 2 nodes.
These nodes/pods run through the `StrimziPodSet` resources and are managed directly by Strimzi.

#### Current situation

Currently, when something in the Pod specification changes, it triggers a change to the `strimzi.io/revision` annotation, which contains a hash of the Pod specification.
During every reconciliation, the operator (its Kafka and Kafka Connect roller components) compares the annotation from the current and desired Pods.
When the annotations are the same, the operator knows that the Pod configuration did not change and that it should not be rolled.
When the annotations differ, it tells the operator that something changed and the operator will roll the Pod.
Today, the `strimzi.io/revision` annotation is derived from the whole Pod specification including the resource requirements of its containers.

#### Planned changes

To support in-place Pod resizing, this annotation will be split in two:
* The existing `strimzi.io/revision` annotation will contain the hash of the whole Pod specification with the exception of its resource requirement configurations.
* A new `strimzi.io/resource-revision` annotation will be introduced to contain a hash based on the resource requirements of the different containers in the Pod.

When the in-place Pod resizing is disabled, the operator will use these annotations in the Kafka and Kafka Connect rollers to decide if the Pods need to be rolled in the same way as before.

When the in-place resizing is enabled, it will be handled in two different parts of the code.
1. The `StrimziPodSet` controller will be responsible for the in-place resizing as part of its `StrimziPodSet` / `Pod` reconciliation.
   It will:
    * Check if the in-place resizing is enabled.
    * If it is enabled, it will compare the `strimzi.io/resource-revision` annotations of the current and desired Pods.
    * If the annotations differ, it will check if the resource requirement change is valid for an in-place update (e.g. if resource limits are not being removed as mentioned earlier).
    * If the change is valid, it will use the `resize` sub-resource of the Pod to apply the changes and patch the `strimzi.io/resource-revision` annotation.
2. The Kafka and Kafka Connect rollers will be responsible for executing the rolling update of the Pod if it is needed.
   This will be done in their `reasonsToRestartPod` and `needsRollingRestart` methods.
   When the in-place Pod resizing is enabled, these methods will:
    * Check if the requested resource change is invalid (e.g. resource limits were removed as mentioned earlier).
      If it is invalid, it will trigger a rolling update of the Pod.
    * Check if the in-place resizing ended up in one of the error states (`Infeasible`, `Error`, etc.).
      If it ended up in one of these states, it will trigger a rolling update of the Pod.

#### Deferred resizing

By default, when the in-place resizing ends up in the `Deferred` state that indicates that there are not enough free resources to resize the Pod, Strimzi will do a rolling update of the Pod to have it rescheduled on another node with sufficient capacity.
However, users will be able to set the `strimzi.io/in-place-resizing-wait-for-deferred: "true"` annotation on the `Kafka`, `KafkaConnect`, or `KafkaMirrorMaker2` resources.
This annotation will indicate that Strimzi should not roll the Pod when the resizing was `Deferred` and instead wait for capacity to be freed on its current node.

_Note: This will not prevent the rolling of the Pod for any other reasons._
_So if - for example - some other configuration is changed that needs a rolling update, the Pod will be rolled regardless of the `Deferred` resizing state._

### Documentation

For the time being, this feature will be documented as _experimental_ and possibly _subject to change_ in the future.
That will allow users interested in it to use it and discover its value.
But it will ensure users are aware of the maturity level of this feature.
The documentation will also mention the various limitations (see the section below).

### Limitations

While in-place Pod resizing is a useful tool, there are also many architectural limitations to it.
This chapter covers them to make it clearer what to expect from this feature and to better understand when it might be useful.
Despite these limitations, this is a useful feature.

#### Large Kafka clusters and dedicated nodes

Large Kafka clusters often run on dedicated nodes.
The dedicated node typically runs only the Kafka brokers and various infrastructure utilities (monitoring tools etc.).
As the brokers have the whole node's capacity at their disposal all the time, there is no need to change their resource requirements or autoscale them.
So this feature provides only a little value for these clusters.

#### Smaller clusters and free capacity

While some Strimzi users might have very large clusters with dedicated nodes, there are also many other users with smaller clusters.
These clusters often run on shared nodes together with other applications.
Changing the Kafka resource requirements or autoscaling the Kafka nodes based on the current load makes a lot more sense for these clusters.
But it still requires the worker nodes to have free capacity available to benefit from dynamic in-place resizing.

If the user has a very tightly bin-packed Kubernetes cluster with no free capacity and relies on cluster auto-scaling to add capacity when needed, the in-place updates might not be efficient and might often require rolling updates to trigger the Kubernetes cluster auto-scaling.

#### Java and memory

While new Java versions have support for Linux containers and can autodetect the container memory capacity, they currently do this only when the JVM is starting.
So, dynamically resizing the memory might lead to:
* Unused memory when the memory capacity is increased but Java is not able to use it due to small `Xmx` configuration.
* Running out of memory when the memory capacity is decreased, but the `Xmx` remains too high.

This might improve in the future as newer Java versions adopt new features such as [automatic heap sizing](https://openjdk.org/jeps/8359211).

#### Cruise Control

This feature allows dynamically resizing the Kafka Pods.
However, the Cruise Control configuration includes the capacity (resources) of the Kafka brokers.
This configuration is currently not dynamically configurable.
As a result, while the Kafka broker Pod might not roll when the resource requirements change, the Cruise Control Pod will always roll in such a situation.

#### Vertical Pod Autoscaling

While this feature can support various forms of Vertical Pod Autoscaling, it does not integrate Strimzi with the Kubernetes Vertical Pod Autoscaler.
It might make it easier to write tools to bridge the Vertical Pod Autoscaler recommendations into Strimzi.
It might also make it easier to do other forms of autoscaling.

## Future improvements

A future improvement that might be built on top of the in-place pod resizing is _startup resource requirements_.
Java-based applications are known for being relatively CPU hungry at startup.
A startup resource requirements feature might allow starting the Pod with additional (CPU) resources and reducing them later when the Pod is ready.
This feature is not part of this proposal.

## Affected projects

This proposal affects the Strimzi Cluster Operators and related documentation.

### Not affected

This proposal affects only the StrimziPodSet-based Pods.
Pods managed by Kubernetes workload controllers (`Deployments`) such as HTTP Bridge, Topic and User Operators, Kafka Exporter, and Cruise Control are not affected by this.
As the Pods are not managed directly by Strimzi, it should be possible to use standard Kubernetes tooling on them.
Also - especially for Topic and User Operators, Kafka Exporter, and Cruise Control - changing their resource requirements seems pretty rare as they are helper tools that do not necessarily need to scale with the Kafka cluster traffic.

## Backwards compatibility

This feature is disabled by default, so it does not introduce any backwards compatibility issues.
And even when enabled, it changes the workflow and the behavior of the Cluster Operator when resource requirements are reconfigured rather than introducing any breaking changes.

## Rejected alternatives

There are currently no rejected alternatives.
