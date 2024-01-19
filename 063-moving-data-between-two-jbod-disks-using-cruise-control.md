# Moving data between two JBOD disks using Cruise Control

This proposal is about integrating the `remove_disks` endpoint from Cruise Control into Strimzi cluster operator. 
This endpoint will allow us to move the data between two JBOD disks. 

## Current situation

Currently, we get a multiple requests to add the ability for moving all Kafka logs between two disks on the JBOD storage array. This feature can be useful in following scenarios:
- The current disk is too big and the user wants to use smaller one
- When we want to use different Storage Class with different parameters or different storage types.
For now, we can do this using the Kafka CLI tools but it is not very user-friendly.

## Motivation

We should introduce the logic to Strimzi to leverage Cruise Control integration and make it possible to move the data between two JBOD disks.
This feature will also allow us to remove the disks without the loss of data.

## Proposal

Cruise Control provides `remove_disks` HTTP REST endpoint to move replicas from a specified disk to other disks of the same broker.
This endpoint allows to trigger a rebalancing operation by moving replicas in a round-robin manner to the remaining disks, from the largest to the smallest, while checking the following constraint:
```sh
1 - (remainingUsageAfterRemoval / remainingCapacity) > errorMargin
```
where:
```sh
remainingUsageAfterRemoval = current usage for remaining disks + additional usage from removed disks
remainingCapacity = sum of capacities of the remaining disks
errorMargin = configurable property (default 0.1); it makes sure that a disk percentage is always free when moving replicas
```

In order to use the above endpoint in the Strimzi cluster operator, it would be added to the [`CruiseControlApi`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/cruisecontrol/CruiseControlApi.java) interface and developing the corresponding implementation.

### Implementation

For implementing this feature, We will be adding a new mode to the `KafkaRebalanceMode` class.
* `remove_disks`: It moves replicas from a specified disk to other disks of the same broker
You can use this mode by changing the `spec.mode` to `remove_disks` in the `KafkaRebalance` resource.

A `KafkaRebalance` custom resource would look like this.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: remove-disks
  brokerandlogdirs: [brokerid1-logdir1,brokerid2-logdir2...] # for eg. 0-/var/lib/kafka/data-0/kafka-log0  
# ...
```

### Flow

- The user should be using the `Kafka` resource with JBOD configured. 
- When the Kafka cluster is ready, the user creates a `KafkaRebalance` custom resource with the `spec.mode` field as `remove-disks` and the list of the broker and the corresponding logdirs to move in the `spec.brokerandlogdirs` field.
- The `KafkaRebalanceAssemblyOperator` starts the interaction with Cruise Control via the `/remove_disks` endpoint for getting an optimization proposal (by using the dryrun feature). 
- The user accepts the proposal by applying the `strimzi.io/rebalance=approve` annotation on it. 
- The `KafkaRebalanceAssemblyOperator` starts the interaction with Cruise Control via the `/remove_disks` endpoint for running the actual rebalancing.

Note: The movement of data between the JBOD disks doesn't affect the broker load, therefore there will be no changes in the before/after broker load.

## Affected/not affected projects

This change impacts the Cruise Control API related classes and the `KafkaRebalanceAssemblyOperator` class.

## Rejected alternatives

No rejected alternatives.
