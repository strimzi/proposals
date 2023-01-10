
# Prevent scale down of the brokers if they contain partition replicas

This proposal is about adding the mechanism for preventing the scale down of the brokers in the cluster if one or more partition replicas are hosted on it.
In the future also, when we introduce the automatic re-balancing when scaling up/down the Kafka Cluster ([proposal](https://github.com/strimzi/proposals/pull/57)), this mechanism would helpful to prevent any data loss when we are scaling down the brokers when they have some partition replicas assigned to them.

## Current situation

Currently, when removing brokers from the Kafka cluster, there is no check to see if these brokers contain any partition replicas.
The [documentation](https://strimzi.io/docs/operators/latest/configuring.html#scaling-clusters-str) recommends that all topic-partition replicas are reassigned before doing this scaling action, which means that if someone tries to scale down without re-assigning the brokers, it can cause availability issues or data loss.

## Motivation

We should introduce the logic which can detect if the broker which is going to be removed still contains the partition replicas or not.
If any partition replicas are still assigned to the broker which is going to be removed from the cluster, then we should get some warning in the status of the Kafka resource which will prompt users to do the reassignment and prevent the broker from being removed until the partition replicas are reassigned.

## Proposal

This proposal suggest how we can add the check to detect if the broker still contains any partition replicas and what to do if the broker scale down is not possible.

## Implementation

### Class

A new utility class named `PreventBrokerScaleDownUtils` will be added to the assembly package of cluster operator module

The reason for having this class is to keep all the utilities for checking the broker partition replicas separate such that we don't create a mess in the `KafkaReconciler` class by adding multiple large methods.
Also, it will help us to test the changes easily.

This class will have utility methods like `canScaleDownBrokers`, `canScaleDownBrokerCheck` and `brokerHasAnyReplicas` methods to check the if broker contains any partition replicas or not.

This class will also contain methods like `topicNames`, `describeTopics` to query the topic metadata.

#### Process:

- When the broker count is changed in the Kafka resource, the `reconcile` method of the `KafkaReconciler` will be triggered to reconcile the Kafka brokers.
- The `canScaleDownBrokers()` utility method will be present at the top of the compose chain in the `reconcile()` method of the `KafkaReconciler`.
- This method will check if the broker contains any partition replicas or not and will continue the process based on the outcome.
- For doing so, the topic metadata will be queried to detect if the broker contains any partition replicas.
- An Admin client instance will be used to connect with the cluster and get us the topic details(topic name and topic description)
- Then we can use this information to check if the broker contains any partition replicas or not.
- The scale down is done after we make sure that the brokers that are going to be removed doesn't contain any partition replicas. By doing this we avoid any partial scale down.

#### What to do if a broker contains partitions?

#### Flow:

- If partition replicas are found out on the broker we will revert back the kafka replicas to the previous count by changing the replica count directly in the kafka object and update the status with a warning stating that the scale down is not done and the replica count is changed back to previous one in the memory.
- During the check, if the admin client is not able to connect to the cluster (not able to get the topic details) or if the partition replicas are assigned to the brokers that are going to be removed, we will update the status of the Kafka CR with the respective warning and revert back the replica count.
- Later on the user can check the status and decide to either change the `spec.replicas` back to the previous count or reassign the partition replicas to other brokers, we can move the process accordingly.
  
## Affected/not affected projects

This change will affect the Strimzi Cluster Operator Module and mostly the `KafkaReconciler` class.

## Rejected alternatives

No rejected alternatives.