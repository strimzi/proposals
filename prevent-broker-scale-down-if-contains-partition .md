
# Prevent scale down of the brokers if they contain replicas

This proposal is about adding the mechanism for preventing the scale down of the brokers in the cluster if one or more replicas are hosted on it.
In the future, when we introduce the automatic re-balancing when scaling up/down the Kafka Cluster ([proposal](https://github.com/strimzi/proposals/blob/main/035-rebalance-types-scaling-brokers.md)), this mechanism would help to prevent any data loss when we are scaling down the brokers when they have some partitions assigned to them.

## Current situation

Currently, when the number of Kafka replicas is decreased in the Kafka CR the StatefulSet will shut down the highest numbered brokers and there is no check to see if these brokers contain any topic-partition replicas.
The [documentation](https://strimzi.io/docs/operators/latest/configuring.html#scaling-clusters-str) recommends that all topic-partition replicas are reassigned before doing this scaling action, which means that if someone tries to scale down without re-assigning the brokers, it can cause data loss.

## Motivation

We should introduce the logic which can detect if the broker which is going to be removed still contains the partition replicas or not.
If the partition replicas are still assigned to the broker which is going to be scaled up/down, then we should get some warning in the status of the Kafka resource which will prompt users to do the reassignment and hopefully reduce the chance of data loss.

## Proposal

This proposal suggest how we can add the check to detect if the broker still contains any partition replicas and what to do if the broker scale down is not possible.

### Checking if the broker contains partition

#### Flow :

- The best way to detect if the broker contains any partition replicas is to query the topic metadata.
- We can create an Admin client instance for this purpose which can be used to get connect with the cluster and get us the topic details(topic name and topic description)
- We can later on query the topic's description to get the information regarding the `topic partition replicas`.
- Then we can use this information to check if the broker contains any partition replicas or not.
- The scale down is done after we make sure that the brokers that are going to be removed aren't containing any partition replicas. By doing this we avoid any partial scale down.

#### What to do if a broker contains partitions?

#### Flow

- A check will be placed just before the scale down steps happens. 
- If a partition is detected over a broker then the Kafka CR will be updated with the desired warnings.
  The warnings will be displayed in the status.
- When the scale down is happening, if the admin client is not able to connect to the cluster, or not able to get the topic details, scale down will not be proceeded, and we will wait for the next reconciliation to happen.
Once the user decides to either change the `spec.replicas` back to the previous count or reassign the partition replicas to other brokers, we can move the process accordingly.
  


## Affected/not affected projects

This change will affect the `KafkaReconciler` class and how the scale down generally happens in the operator.

## Rejected alternatives

No rejected alternatives.