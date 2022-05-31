
# Prevent scale down of the brokers if they contain replicas

This proposal is about adding the mechanism for preventing the scale down of the brokers in the cluster if one or more replicas are hosted on it.
In the future, when we introduce the automatic re-balancing when scaling up/down the Kafka Cluster ([proposal](https://github.com/strimzi/proposals/blob/main/035-rebalance-types-scaling-brokers.md)), this mechanism would help to prevent any data loss when we are scaling down the brokers when they have some partitions assigned to them.

## Current situation

Currently, when the number of Kafka replicas is decreased in the Kafka CR the StatefulSet will shut down the highest numbered brokers and there is no check to see if these brokers contain any topic-partition replicas.
The [documentation](https://strimzi.io/docs/operators/latest/configuring.html#scaling-clusters-str) recommends that all topic-partition replicas are reassigned before doing this scaling action, which means that if someone tries to scale down without re-assigning the brokers, it can cause data loss.

## Motivation

We should definitely introduce some sort of logic which can detect if the broker which is going to be removed still contains the partition replicas or not.
If the partition replicas are still available on the broker which is going to be scaled, then we should get some warning in the logs which will prompt users to do the reassignment and hopefully reduce the chance of data loss.

## Proposal

This proposal suggest how we can add the check to detect if the broker still contains any partition replicas and what to do if the broker scale down is not possible.

### Checking if the broker contains partition

#### Flow :

- The best way to detect if the broker contains any partition replicas is to query the topic metadata.
- We can create an Admin client instance for this purpose which can be used to get connect with the cluster and get us the topic details(topic name and topic description)
- We can later on query the topic's description to get the information regarding the `topic partition replicas`.
- Then we can use this information to check if the broker contains any partition replicas or not.
- We should make sure that there is no partial scale down by checking the replicas on the brokers(to be scaled down) first and only do the whole scale down after we make sure that the brokers that are going to be removed aren't containing any partition replicas.

#### What to do if a broker contains partitions?

- If a partition is detected over a broker then we should update the Kafka CR with the desired warnings
- In case, if the admin client is not able to connect to the cluster, or we are not able to somehow get the topic details then in that case we should not do any scale down and wait for the next reconciliation to happen, but we will update the Kafka CR with the corresponding error message. 
  Once the user decides to either change the `spec.replicas` back to the previous count or reassign the partition replicas to other brokers, we can move the process accordingly.

### Error Handling

If we somehow fail to connect to the Kafka Cluster or there are some other error which happens during the reconciliation, in that case also we will set the `scaleDown` variable to `false`, and provide a warning log on reason why the `scaleDown` was not done.

## Affected/not affected projects

This change will affect the `KafkaReconciler` class and how the scale down generally happens in the operator.

## Rejected alternatives

No rejected alternatives.