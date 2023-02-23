
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

### Process:

- When the broker count is changed in the Kafka resource, the `reconcile` method of the `KafkaReconciler` will be triggered to reconcile the Kafka brokers.
- The `canScaleDownBrokers()` utility method will be present at the top of the compose chain in the `reconcile()` method of the `KafkaReconciler` to make sure that every other method  which requires the replica count use the correct replica count based on the outcome of the check.
- The `canScaleDownBrokers()` method will only run if we see the current Kafka replicas (replicas before the Kafka custom resource is modified) count gets greater than the Kafka replicas present in the Kafka custom resource.
  We can get the desired Kafka replica count by using `kafka.getReplicas()` where `kafka` is an object of `KafkaCluster` class .
- This method will check if the broker contains any partition replicas or not and will continue the process based on the outcome.
- For doing so, the topic metadata will be queried to detect if the broker contains any partition replicas.
- An Admin client instance will be used to connect with the cluster and get us the topic details(topic name and topic description)
- Then we can use this information to check if the broker contains any partition replicas or not.
- The scale down is done after we make sure that the brokers that are going to be removed doesn't contain any partition replicas.
By doing this we avoid any partial scale down.

### What to do if a broker contains partitions?

#### Flow:

- If partition replicas are found out on the broker we will revert back the Kafka replicas to the previous count by setting replicas directly in the `KafkaCluster` class using `setReplicas()` method. 
  Changing the Kafka replica count directly in the Kafka Cluster would help us ensure that we keep same replicas everywhere like while generating certificates, services, ingresses, routes etc.
- The broker certificates, services, ingresses, routes etc. will be treated with the original number of nodes and the rest of the reconciliation will be done normally.
- We also generate a new condition which will be added to Kafka resource status depicting that the scale down is not done. It will also contain the `spec.replicas` count(which is being currently being used) in the condition message.
  ```yaml
  status:
    clusterId: DoRj5f84Sruq_7TJ31y7Zw
    conditions:
      - lastTransitionTime: "2023-02-22T10:18:56.578009768Z"
        message: 'Cannot Scale down since broker contains partition replicas. The `spec.kafka.replicas` should be reverted back to 4 directly in the Kafka resource'.
        reason: ScaleDownException
        status: "True"
        type: Warning
      - lastTransitionTime: "2023-02-22T10:18:57.664668863Z"
        status: "True"
        type: Ready
  ```
  Note :  By the time the replicas are reverted back, the storage validation will be already complete based on the replica count present in Kafka custom resource. This can can cause some issues if someone tries to make some forbidden changes (changes that might not be supported) to the storage during this time frame. To fix this problem, the user should revert back the `spec.kafka.replicas` in the Kafka custom resource back to the replica count currently being used by the `KafkaCluster` class and the next reconciliation will pick up those changes.

### How to bypass the broker scale down mechanism

- To bypass the broker scale down mechanism you can use the annotation `strimzi.io/bypass-broker-scaledown-check: "true"` on the Kafka custom resource. For e.g.
  ```sh
  kubectl annotate Kafka my-cluster strimzi.io/bypass-broker-scaledown-check: "true"
  ```

### Other Scenarios

- During the check, if the admin client is not able to connect to the cluster (not able to get the topic details), we will update the status of the Kafka CR with the respective warning and revert back the replica count in the `KafkaCluster` class.
- If the Kafka cluster is just initialized and the pods are not ready, the `canScaleDownBrokers()` utility method will not work because the current Kafka replicas (replicas before the Kafka custom resource is modified) count will be equal to 0 and the Kafka replicas present in the Kafka custom resource will also be 0. Hence, the mechanism will not run since the condition requires the current replica count to be greater than the Kafka replicas present in the Kafka custom resource
- If the current Kafka replicas/pods are 0 the mechanism will not work since if there are zero brokers it will not be considered a scaledown.
 
## Affected/not affected projects

This change will affect the Strimzi Cluster Operator Module and mostly the `KafkaReconciler` class.

## Rejected alternatives

No rejected alternatives.