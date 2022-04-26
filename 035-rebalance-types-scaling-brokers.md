# Extending the `KafkaRebalance` resource with rebalance types to help scaling the Apache Kafka cluster

This proposal is about leveraging the Cruise Control integration in the Strimzi cluster operator in order to run an Apache Kafka cluster rebalancing when it's scaled up or down.
It's related to add the integration with two specific Cruise Control API endpoints for helping to move replicas to newly added brokers or move them out from removed ones, during the scaling operations.
The proposal doesn't assume any automation to run the rebalancing, it is always user driven by using the improved `KafkaRebalance` custom resource. See the [Flow](#flow) section for more details.
It has to be considered as "phase 1" towards more automation.
The future plan is about having the user able to specify some criteria to match in order to have the Strimzi cluster operator able to run the rebalancing operation automatically during the scale up and down, reducing the user manual intervention.

## Current situation

Currently, when Cruise Control is deployed alongside the Apache Kafka cluster, no automatic rebalancing happens when the cluster itself is scaled up or down.

It means that, if the cluster is scaled up, no replicas will be moved to the newly added brokers.
These brokers will only get partition replicas of newly created topics.
Furthermore, by using the `KafkaRebalance` custom resource to run a rebalancing operation after the scaling up, it will move replicas across all the already existing brokers and not only to the new ones.
It actually runs a full-rebalance with the drawback of more data movement, CPU usage and so on.

In the same way, if the cluster is scaled down, the replicas hosted on the target brokers to remove are not moved out before the scaling down happens.
Right now the scaling down is not blocked but doing it is not part of this proposal.
This could result in some partitions having less ISR (In-Sync Replicas) than before the scaling down, with an impact on availability of the cluster.
The current Cruise Control integration has no support, via the `KafkaRebalance` custom resource, to allow moving these replicas out of the target brokers before the scaling down happens.

## Motivation

When the Apache Kafka cluster is scaled up, it should be possible to use the Cruise Control integration to run a rebalancing by moving some replicas to the newly added brokers without involving the already existing ones.
This would provide a faster rebalancing, not involving the entire cluster.

Not stopping the scaling down operation when replicas are hosted on the target brokers to remove is not part of this proposal but it should be possible to move replicas out of the target brokers before the scaling down happens.
It would allow to remove "empty" brokers which are not hosting any replicas, avoiding less ISR problem.

## Proposal

### Cruise Control add and remove brokers endpoints

Cruise Control provides two specific HTTP REST endpoints to rebalance a cluster when scaling up or down:

* [`/add_broker`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#add-a-list-of-new-brokers-to-kafka-cluster): this endpoint allows to trigger a rebalancing operation by moving replicas from the already existing brokers to the new ones but not moving them among existing brokers. It can be used right after the scaling up, for getting a proposal (as dryrun) and running the actual rebalancing.
* [`/remove_broker`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#decommission-a-list-of-brokers-from-the-kafka-cluster): this endpoint allows to trigger a rebalancing operation by moving replicas out from the target brokers to remove, in order to leave them "empty" and not hosting any replicas. It can be used just before the scaling down, for getting a proposal (as dryrun) and running the actual rebalancing.

In order to use the above two endpoints in the Strimzi cluster operator, they would be added to the [`CruiseControlApi`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/cruisecontrol/CruiseControlApi.java) interface and developing the corresponding implementation.

### KafkaRebalance modes

The current usage of the `KafkaRebalance` custom resource allows to get an optimization proposal and start a full-rebalance with data movement across all the brokers in the cluster.
At Cruise Control API level, it's reflected by the `KafkaRebalanceAssemblyOperator` using the `/proposals` endpoint for getting an optimization proposal first and then the `/rebalance` endpoint for starting the full-rebalance.

In order to start a rebalancing on a cluster being scaled up or down, the proposal is about adding a `spec.mode` field in the `KafkaRebalance` custom resource with the following possible values:

* `full-rebalance`: it starts a full-rebalance across all the brokers in the cluster as it works today. It is the default behavior if not specified (for backward compatibility).
* `add-broker`: it starts a rebalancing by moving replicas out from the already existing brokers to the new added ones after a scale up.
* `remove-broker`: it starts a rebalancing by moving replicas out from the target brokers to remove before a scale down.

When the rebalancing is related to cluster being scaled up or down, the newly added brokers or the ones that are going to be removed have to be specified through a new `spec.brokers` field as an array.
This needs come from how the Cruise Control API endpoints works. There is a specific query parameter which needs that list.

A `KafkaRebalance` custom resource would look like this.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: <value> # full-rebalance, add-broker or remove-broker. Assuming full-rebalance by default if the field is not specified.
  brokers:
    - 3
    - 4
  goals:
    - RackAwareGoal
    - ReplicaCapacityGoal
```

### Flow

The proposal doesn't assume any automation to run the rebalancing when the cluster is scaled up or down.
Using the `StatefulSet` for Kafka brokers, the corresponding broker IDs are implied in one direction (added from the higher one up, removed from the higher one down) and the cluster operator could get them for the rebalancing.
But on the other side, Strimzi is moving to use the [`StrimziPodSet`](https://github.com/strimzi/proposals/pull/44) and towards a ZooKeeper-less cluster support so that the broker IDs are handled differently and the user has the flexibility to scale down by removing specific brokers or scale up by specifying new broker IDs.
For this reason the proposal is about having the user to specify the added or removed brokers for the rebalancing.
If needed, it also provides the flexibility to scale up the cluster but running the rebalancing taking into account only a subset of the newly added brokers and leaving others just for new topics.
The same could happen by running a rebalance for removing brokers to make them "empty" but then not doing the actual scale down.

#### Rebalance on scale up

When scaling up the cluster, the user can go through the following procedure:

1. The user increases the number of replicas through the `spec.kafka.replicas` of the `Kafka` custom resource.
2. When the scaling up is done, the user creates a `KafkaRebalance` custom resource with the `spec.mode` field as `add-broker` and the list of the new brokers in the `spec.brokers` field.
3. The `KafkaRebalanceAssemblyOperator` starts the interaction with Cruise Control via the `/add_broker` endpoint for getting an optimization proposal (by using the dryrun feature).
4. The user accepts the proposal by applying the `strimzi.io/rebalance=approve` annotation on it.
5. The `KafkaRebalanceAssemblyOperator` starts the interaction with Cruise Control via the `/add_broker` endpoint for running the actual rebalancing.

#### Rebalance on scale down

When scaling down the cluster, the user can go through the following procedure:

1. The user creates a `KafkaRebalance` custom resource with the `spec.mode` field as `remove-broker` and the list of the target brokers to remove in the `spec.brokers` field.
2. The `KafkaRebalanceAssemblyOperator` starts the interaction with Cruise Control via the `/remove_broker` endpoint for getting an optimization proposal (by using the dryrun feature).
3. The user accepts the proposal by applying the `strimzi.io/rebalance=approve` annotation on it.
4. The `KafkaRebalanceAssemblyOperator` starts the interaction with Cruise Control via the `/remove_broker` endpoint for running the actual rebalancing.
5. When the rebalancing is done, the user can finally scale down the cluster by decreasing the number of replicas through the `spec.kafka.replicas` of the `Kafka` custom resource.

It could happen that before the scaling down but after the rebalancing a new topic is created and replicas are added on the target brokers to remove.
It is not part of this proposal, but the Strimzi cluster operator should prevent the scale down in this case.

## Affected/not affected projects

This proposal impacts the Strimzi Cluster Operator only related to the Cruise Control API and the `KafkaRebalanceAssemblyOperator` more specifically.

## Compatibility

The proposed changes are backward compatible because the default mode for the `KafkaRebalance` custom resource is the current full-rebalance when it's not specified.

## Rejected alternatives

No rejected alternatives.
