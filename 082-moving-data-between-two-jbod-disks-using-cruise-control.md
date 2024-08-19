# Moving data between two JBOD disks using Cruise Control

This proposal is about integrating the [`remove_disks`](https://github.com/linkedin/cruise-control/blob/main/cruise-control/src/main/resources/yaml/endpoints/removeDisks.yaml) endpoint from Cruise Control into Strimzi cluster operator. 
This endpoint will allow us to move the data between two JBOD disks. 

## Current situation

Currently, we get a multiple requests from community users to add the ability for moving all Kafka logs between two disks on the JBOD storage array. This feature can be useful in following scenarios:
- The current disk is too small and the user wants to use a bigger one, or vice versa.
- When we want to use a different Storage Class with different parameters or different storage types.
- In case of disk removal to reduce the total storage.

For now, we can do this using the Kafka CLI `kafka-reassign-partitions.sh` tool, but it takes a lot of manual steps which is time-consuming and not so user-friendly.

## Motivation

We should introduce the logic to Strimzi to leverage Cruise Control integration and make it possible to move the data between two JBOD disks.
This feature will also allow us to remove the disks without the loss of data.

## Proposal

Cruise Control provides the `remove_disks` HTTP REST endpoint to move replicas from a specified disk to other disks for the same broker. The operation is only for intra-broker rebalancing, not moving data between brokers.
This endpoint triggers a rebalancing operation that moves replicas, starting with the largest and proceeding to the smallest, to the remaining disks while ensuring the following constraint is met:
```sh
1 - (remainingUsageAfterRemoval / remainingCapacity) > errorMargin
```
where:
```sh
remainingUsageAfterRemoval = current usage for remaining disks + additional usage from removed disks
remainingCapacity = sum of capacities of the remaining disks
errorMargin = configurable property (default 0.1); it makes sure that a disk percentage is always free when moving replicas
```

To use the `remove_disks` endpoint in the Strimzi cluster operator, it should be added to the [`CruiseControlApi`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/cruisecontrol/CruiseControlApi.java) interface, and the corresponding implementation developed.

### Implementation

To implement this feature, we will be adding a new mode to the `KafkaRebalanceMode` class.
* `remove-disks`: It moves replicas from a specified disk to other disks of the same broker. It always uses intra-broker re-balancing.
You can use this mode by changing the `spec.mode` to `remove-disks` in the `KafkaRebalance` resource.

A `KafkaRebalance` custom resource would look like this.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
spec:
  # setting the mode as `remove-disks` to move data between the JBOD disks
  mode: remove-disks
  # providing the list of brokers, and the corresponding volumes from which you want to move the replicas
  moveReplicasOffVolumes:
    - brokerId: 0
      volumeIds: [1, 2]
    - brokerId: 2
      volumeIds: [1]
# ...
```

### Flow

- The user should be using the `Kafka` resource with JBOD configured, making sure that they have more than one disk configured on the brokers.
- When the Kafka cluster is ready, the user creates a `KafkaRebalance` custom resource with the `spec.mode` field as `remove-disks` and provides a list of the brokers, and the corresponding volumes from which they want to move the replicas in the `spec.moveReplicasOffVolumes` field. In case, the `spec.moveReplicasOffVolumes` field is not set, then the `KafkaRebalance` resource will move to `NotReady` state prompting that `spec.moveReplicasOffVolumes` field is missing.
- The `KafkaRebalanceAssemblyOperator` interacts with Cruise Control via the `/remove_disks` endpoint to generate an optimization proposal (by using the dryrun feature).
- You can use `strimzi.io/rebalance-auto-approval:true` annotation on the `KafkaRebalance` resource for auto-approval of proposal. In case you want to do it manually you can do it by applying the `strimzi.io/rebalance=approve` annotation on it.
- The `KafkaRebalanceAssemblyOperator` interacts with Cruise Control via the `/remove_disks` endpoint to perform the actual rebalancing.

> **NOTE** The optimization proposal will not show the load before optimization, it will only show the load after optimization. This is because in upstream Cruise Control we don't have the verbose tag enabled with the `remove_disks` endpoint.

### Other Scenarios

- In case the user is not using JBOD storage and tries to generate the optimization proposal, the `KafkaRebalance` resource will move to `NotReady` state prompting invalid log dirs provided for the broker.
- If you are using JBOD with single disk configured on the brokers, in that case `KafkaRebalance` will move to `NotReady` state prompting that you don't have enough log dirs to move the replicas for that broker.
- If the disk capacity has exceeded for the broker, in that case `KafkaRebalance` will move to `NotReady` prompting that enough capacity is not remaining to move replicas for that broker.
- This feature works fine with `KafkaNodePool` resources.
- This feature works with KRaft only if Kafka version is greater than 3.7.0, as that version supports multiple JBOD disks on brokers.

Errors for these scenarios are reported by Cruise Control.
Based on these errors, we transition the `KafkaRebalance` resource to the `NotReady` state and update its status with the corresponding error message.

## Affected/not affected projects

This change impacts the Cruise Control API related classes and the `KafkaRebalanceAssemblyOperator` class.

## Rejected alternatives

No rejected alternatives.
