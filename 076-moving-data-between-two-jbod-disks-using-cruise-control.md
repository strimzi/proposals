# Moving data between two JBOD disks using Cruise Control

This proposal is about integrating the [`remove_disks`](https://github.com/linkedin/cruise-control/blob/main/cruise-control/src/main/resources/yaml/endpoints/removeDisks.yaml) endpoint from Cruise Control into Strimzi cluster operator. 
This endpoint will allow us to move the data between two JBOD disks. 

## Current situation

Currently, we get a multiple requests from community users to add the ability for moving all Kafka logs between two disks on the JBOD storage array. This feature can be useful in following scenarios:
- The current disk is too big and the user wants to use a smaller one and vice versa is true.
- When we want to use a different Storage Class with different parameters or different storage types.
- In case of disk removal to reduce the total storage.

For now, we can do this using the Kafka CLI tool i.e. `kafka-reassign-partitions.sh` tool, but it takes a lot of manual steps which is time-consuming and not so user-friendly.

## Motivation

We should introduce the logic to Strimzi to leverage Cruise Control integration and make it possible to move the data between two JBOD disks.
This feature will also allow us to remove the disks without the loss of data.

## Proposal

Cruise Control provides `remove_disks` HTTP REST endpoint to move replicas from a specified disk to other disks of the same broker.
This endpoint triggers a rebalancing operation by moving replicas in a size-based manner to the remaining disks, from the largest to the smallest, while checking the following constraint:
```sh
1 - (remainingUsageAfterRemoval / remainingCapacity) > errorMargin
```
where:
```sh
remainingUsageAfterRemoval = current usage for remaining disks + additional usage from removed disks
remainingCapacity = sum of capacities of the remaining disks
errorMargin = configurable property (default 0.1); it makes sure that a disk percentage is always free when moving replicas
```

In order to use the `remove_disks` endpoint in the Strimzi cluster operator, it would be added to the [`CruiseControlApi`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/cruisecontrol/CruiseControlApi.java) interface and developing the corresponding implementation.

### Implementation

To implement this feature, we will be adding a new mode to the `KafkaRebalanceMode` class.
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
  mode: remove_disks
  brokerAndVolumeIds:
    - brokerId: 0
      volumeId: 2
    - brokerId: 1
      volumeId: 0
# ...
```

### Flow

- The user should be using the `Kafka` resource with JBOD configured. Make sure that you have more than one disk configured on the brokers.
- When the Kafka cluster is ready, the user creates a `KafkaRebalance` custom resource with the `spec.mode` field as `remove_disks` and the list of the broker and the corresponding logdirs to move in the `spec.brokerandlogdirs` field.
- The `KafkaRebalanceAssemblyOperator` interacts with Cruise Control via the `/remove_disks` endpoint to generate an optimization proposal (by using the dryrun feature).
- You can use `strimzi.io/rebalance-auto-approval:true` annotation on the `KafkaRebalance` resource for auto-approval of proposal. In case you want to do it manually you can do it by applying the `strimzi.io/rebalance=approve` annotation on it.
- The `KafkaRebalanceAssemblyOperator` interacts with Cruise Control via the `/remove_disks` endpoint to perform the actual rebalancing.

Note: The optimization proposal will not show the load before optimization, it will only show the load after optimization. This is because in upstream Cruise Control, we don't have the verbose tag enabled  with the `remove_disks` endpoint.

### Other Scenarios

- In case the user is not using JBOD storage and tries to generate the optimization proposal, the `KafkaRebalance` resource will move to `NotReady` state prompting invalid log dirs provided for the broker.
- If you are using JBOD with single disk configured on the brokers, in that case `KafkaRebalance` will move to `NotReady` state prompting that you don't have enough log dirs to move the repicas to for that broker.
- If the disk capacity has exceeded for the broker, in that case `KafkaRebalance` will move to `NotReady` prompting that enough capacity is not remaining to move replicas for that broker.
- This feature works fine with KafkaNodePools. 
- This feature work with Kraft only if Kafka version is greater than 3.7.0, as that version supports multiple JBOD disks on brokers.

## Affected/not affected projects

This change impacts the Cruise Control API related classes and the `KafkaRebalanceAssemblyOperator` class.

## Rejected alternatives

No rejected alternatives.
