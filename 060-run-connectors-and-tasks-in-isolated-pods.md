
# Run Connectors and Tasks in Isolated Pods

Deploy Connectors and Tasks in isolated pods, making use of the Static Assignments feature concurrently proposed for Kafka Connect.

## Current situation

Currently, Strimzi supports the Connect Distributed deployment model to execute KafkaConnector workloads.
In this deployment model, a Connect cluster is made up of workers, each consisting of a single process JVM.
In Strimzi, these workers are given uniform resources, and the number of workers in a cluster is controlled the Deployment's replicas.
At runtime, each worker is assigned part of the Connector & Task workload through an internal rebalance algorithm.
This rebalance algorithm does not distinguish the resource requirements of each job, and performs unweighted scheduling across all workers.
A single worker may be assigned multiple connectors or tasks, and these all share the resources provisioned to the hosting worker.

## Motivation

The Connect Distributed deployment mode makes it difficult or impossible to safely operate with a heterogeneous collection of connectors and tasks.
It prevents external mechanisms from scheduling the resources of connectors and tasks by internally assigning jobs to workers.
It also prevents accurately characterizing the resource usage of a single job by sharing a single JVM across multiple jobs.
Connectors which share a single cluster have no mechanisms to avoid noisy-neighbor resource starvation or outages.

Kubernetes (and Strimzi) are equipped to address the resource scheduling and isolation concerns at the process boundary.
The upcoming Kafka Connect Static Assignments feature allows each worker in a cluster to be given a single job assignment.
This allows an external orchestration framework (e.g. Strimzi) to control resource allocations and job scheduling.
This functionality will be added to Apache Kafka in [KIP-987](https://cwiki.apache.org/confluence/display/KAFKA/KIP-987%3A+Connect+Static+Assignments) and released in vX.Y.

## Proposal

In addition to the default shared-JVM deployment mode, Strimzi should also support a isolated-JVM deployment mode.
This mode will use the Static Assignments feature to assign connectors and tasks each to their own worker.
This will be opt-in, and able to be performed as a live migration to an existing cluster.

A KafkaConnectorSpec will accept a new optional ConnectorResourceRequirements field, to be used for the Connector instance.
When not specified, the Strimzi operator will let the Connect cluster control scheduling of the Connector instance.
When specified, the Strimzi operator will create a pod for that Connector instance with the specified resources applied to the pod.

A KafkaConnectorSpec will also accept a new optional TaskResourceRequirements field, to be used for the Task instances.
The TaskResourceRequirements will contain a Default field, which contains the resources to apply to a task if no override is specified.
Additionally, TaskResourceRequirements will contain an Override list, which contains a list of resource requirements to apply to the tasks, 0-indexed.
If the list of overrides is shorter than the list of tasks, each task without a requirement will use the Default requirements.
For each task with either an Override or Default requirement, the Strimzi operator will create a pod for that Task instance with the specified resources applied to that pod.

The Strimzi operator will regularly poll the REST API for the number of tasks, and create corresponding pods for those tasks.
If a pod is assigned for a connector or task which no longer exists in the REST API, the corresponding pod is removed.

For a cluster which has ResourceRequirements for all KafkaConnectors, the shared workers will typically be empty, functioning as hot standby instances.
Hot-standby workers are assigned connectors and tasks which do not have a live static worker, either because that pod is offline or Strimzi has not created it yet.
Connectors may dynamically add tasks, and if strimzi is offline or slow to respond, those tasks will be assigned to hot-standby workers.
If no hot-standby workers are available, tasks will remain unassigned until the associated static worker pod rejoins the group.
If no hot-standby workers are available, connectors and tasks will be disallowed from using shared JVMs.

If the ResourceRequirements for a KafkaConnector is changed, the corresponding static pods will be restarted with the new requirements applied.

## Affected/not affected projects

This proposal will require changes to the strimzi-kafka-operator for Connect. It will not change the operator for any other service.
It also will not affect any other Strimzi projects.

## Compatibility

### Upgrades

For existing clusters that wish to migrate to using isolated instances, an upgrade can proceed as follows:
1. Upgrade the Connect image to a version which supports static assignments feature.
2. Move the desired Connectors to isolated mode individually. For each connector:
   1. Estimate the resources needed for the connector and each task, possibly by dividing the KafkaConnect resource requirements by the average tasks-per-worker.
   2. Apply the resource requirements to the connector and task defaults, and wait for corresponding pods to be created.
   3. Verify the health of the connector & tasks, and compare the resource utilization to the estimates. Increase or decrease requirements as needed.
   4. If the Kubernetes cluster cannot schedule the pods due to resource constraints, consider adding a new node or reducing the KafkaConnect replicas.
   5. If the KafkaConnect workers are lightly loaded, consider reducing the KafkaConnect replicas.
3. If all Connectors now have resource requirements specified, reduce the KafkaConnect replicas to the number of desired hot-standby workers.
   1. The maximum number of recommended hot-standbys is the number of replicas that was previously in-use before applying per-connector/task ResourceRequirements.
   2. The minimum number of recommended hot-standbys is 0, which disables hot-standbys entirely and forces the use of isolated JVMs.

### Downgrades

On downgrade, clusters using isolated instances may experience resource exhaustion.
If resources are specified for a connector or task, the version downgrade will cause static assignments to be ignored.
If the connector/task ResourceRequirements are less than the KafkaConnect ResourceRequirements, the cluster will potentially over-assign the lower-resource workers.
KafkaConnector resource requirements should be removed and the number of replicas increased again, before downgrading the cluster to a version which does not support Static Assignments.

KafkaConnector resource requirements should also be removed before downgrading Strimzi to a version which does not include it, as the added ResourceRequirement fields will not be valid in earlier versions.

## Rejected alternatives

### Simplify the KafkaConnector ResourceRequirements to specify only one or two requirements

Most connectors use fewer resources for the Connector as compared to the Task instances.
If there was only one ResourceRequirements for a KafkaConnector, it would have to apply to the Connector and Tasks equally.
This would waste capacity on Connectors which consume significantly less resources than their accompanying task.

Similarly, there are connectors with heterogeneous tasks, or a special "task 0" which has higher requirements than other tasks.
It is desirable to have a way to specify the ResourceRequirements for those tasks in a way that still applies a "good-enough" default for other tasks

### Allow users to share static JVMs across jobs

Due to the low resource usage of Connectors, it could be beneficial to move all Connector instances in a cluster to a single static pod, to amortize the cost of the JVM overhead.

This would have the downside that adding or removing a connector would force all connectors to restart.
In the current design, users can approximate this by specifying ResourceRequirements for Tasks and not Connectors, so that Connectors remain assigned to the shared replicas.
This avoids needing to restart the shared static worker, as the shared replicas can dynamically add and remove jobs that don't have a static assignment.

Additionally, the user would need a way to specify when to share the JVM of static workers, and which jobs to share it with, effectively creating multiple pools of shared workers.
This is more complex to specify and operate, and doesn't have the resource isolation benefits that this feature is motivated by.

### Add per-task isolated hot-standbys

By starting up two pods with a static assignment for a single task, one of those pods will function as the preferred hot-standby if the other fails.
This would avoid the task joining the shared worker pool, and maintain resource isolation at the cost of a much higher resource cost.

This can be added later as an extension, but is omitted due to its poor operating cost tradeoff as compared to shared standby workers.
