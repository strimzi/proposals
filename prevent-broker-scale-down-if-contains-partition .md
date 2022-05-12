
# Prevent scale down of the brokers if they contain partitions

This proposal is about adding the mechanism for preventing the scale down of the brokers in the cluster if a partition is assigned to it.
In the future, when we introduce the automatic scale up/down of the Kafka Cluster, this mechanism would help to prevent any data loss when we are scaling down the brokers when they have some partitions assigned to them.

## Current situation

Currently, when the number of Kafka replicas is decreased in the Kafka CR the StatefulSet will shut down the highest numbered brokers and there is no check to see if these brokers contain any topic-partition replicas.
The [documentation](https://strimzi.io/docs/operators/latest/configuring.html#scaling-clusters-str) recommends that all topic-partition replicas are reassigned before doing this scaling action, which means that if someone tries to scale down without re-assigning the brokers, it can cause data loss.

## Motivation

We should definitely introduce some sort of logic which can detect if the broker which is going to be removed still contains the partitions or not.
If the partitions are still available on the broker which is going to be scaled, then we should get some warning in the logs which will prompt users to do the reassignment and hopefully reduce the chance of data loss.

## Proposal

This proposal suggest how we can add the check to detect if the broker still contains any partitions and what to do if the broker scale down is not possible.

### Checking if the broker contains partition
The best way to detect if the broker contains any partitions is to query the topic metadata.
To do this, we will have to make use of the `AdminClient` class.
We should probably put the logic for this check inside the `scaleDown` method present in the `KafkaReconciler` class.
The `scaleDown` method gets triggered when we scale down the no. of replicas in the `Kafka CR`.
So this check will target the broker which are going to be removed i.e. the pods with the highest number.

Methods to be created in the process:

- `getTopicNamesAndDescriptions` (Boolean Type) - This method will be placed inside the `scaleDown()` method in `KafkaReconciler` class and will provide us with the Collection of `TopicDescription`.
- `describeTopics()` and `topicNames()`  - These two methods will provide us with the description and names of all the topics ie first we will get the names of the topics using the `topicNames` method and using these names we will extract the description of the topics.
- `brokerContainsPartitions` - This method will take as input a `Collection<TopicDescription>` and then generate a map of type `HashMap<Integer podId, Boolean containsPartitions>`.

### Flow :

When the replicas are changed inside the Kafka resource, reconciliation will happen.
During this reconciliation, we move to the `scaleDown()` method present in the `KafkaReconciler` class.
The `scaleDown()` method contains the complete mechanism on how the broker is scaled down. So there, we will first create our `adminClient` and provide it with the Secrets.
Then we will call the `getTopicNamesAndDescriptions()` method which will give us the `Collection<TopicDescription>`. 

The `getTopicNamesAndDescriptions()` method will call the `topicNames()` method to get the set of topic names and then these topic names will be used by the `describeTopics()` methods to get the `Collection<TopicDescription>`. 
Once the `Collection<TopicDescription>` is computed we will move back to the `scaledown()` method and there we can call `brokerContainsPartition()` method which will take the `Collection<TopicDescription>` and then create a `hasAssignedTopicPartitions` map which contains key as `podId` and value as `boolean` type.
Now we can use this map to check if the broker contains any partitions or not.

### What to do if a broker contains partitions?
If a partition is detected over a broker then we should trigger some log warnings and let the partitions remain the same as they were before.
For example, if we try to scale down the broker from 5 to 3, but the check detected that the brokers to be scaled down still contain partitions then we should keep the replicas as 5 only display the appropriate warnings.

To handle the case, if broker contain some partitions, we can  create a variable named `scaleDown` which will be set to `false` by default and if broker doesn't contain any partition, then we will set this variable to `true` else, it will stay `false`.
Then we can use can an `if` statement to check if the boolean result is `true` or `false`. We will set this `if` just before the logic where scale down w.r.t to `StatefulSet` or `StrimziPodSet` happens.
If the result is `false` then we will return a warning log as `Broker contains Partition` and return a succeeded future i.e. no need to scale down, but if the result is `true` then we can run the logic of scale down as per `StatefulSets` or `StrimziPodSets`.

### Error Handling

If we somehow fail to connect to the Kafka Cluster or there are some other error which happens during the reconciliation, in that case also we will set the `scaleDown` variable to `false`, and provide a warning log on reason why the `scaleDown` was not done.

## Affected/not affected projects

This change will affect the `KafkaReconciler` class and how the scale down generally happens in the operator.

## Rejected alternatives

No rejected alternatives.