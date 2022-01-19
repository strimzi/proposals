# Support Replication Factor Change for Kafka Topics

Apache Kafka introduced the Replication Factor (RF) as the configuration that determines the number replicas kafka brokers will retain for a single data partition. A bigger number of the RF means both higher reliability for the data partition and higher infrastructure cost for topics.

Apache Kafka allows users to change the RF for any topic partition through the native scripts during the runtime. However, it's not a lightweight process, because users usually need to decide which brokers to host new replicas and which replicas need to be dropped, or even conduct a cluster rebalance after the change of the RF.  

The goal of this proposal is to offer users a simple approach to change the topic RF through the KafkaTopic Custom Resources and Cruise Control, in the cloud native, strimzi style. 

## Current situation

Currently, users have two options to change the topic RF:
* Leverage the Native Kafka script to change the RF. Users need to generate the partition replica assignment and balance the kafka clusters manually if needed. 
* Delete the topic and recreate the topic with the desired RF.

Clearly none of the above approaches is user-friendly.

When users attempt to change the RF in the KafkaTopic Resources, the Status of the KafkaTopic will become `NotReady` with `ReplicationFactorChangeException`. The following error message is shared with users.

`Changing 'spec.replicas' is not supported. This KafkaTopic's 'spec.replicas' should be reverted to 4 and then the replication should be changed directly in Kafka.`

## Motivation

With this new feature, it will become much easier to achieve the desired tradeoff between the cost and data reliability in the kafka clusters. 

For instance, users can lower the RFs for selective less critical topics to reduce the infrastructure cost (Compute, Storage and Network). Also, for some critical kafka topics, users can increase the data reliability through the bump of the RFs ONLY for these specific topics. 

More importantly, users only need to specify the new RF number in the `KafkaTopic` resource and approve the change, in order to change the RF for kafka topics. The heavy lifting work, such as new replica placement, extra replica deletion and possible cluster rebalance, is completely taken care by the strimzi. 

## Proposal

### API
The Kafka Topic RF can be changed by users in the following steps:
* Update the `spec.replicas` properties in the target `KafkaTopic` Resource in the kubernetes cluster
* Watch the `Status` field of the `KafkaTopic Resource`, for the status update of this RF change.

### Design
We propose to change the topic RFs with the support of `Cruise Control` and `Strimzi Topic Operator`. 

#### Cruise Control
Cruise Control Service will be leveraged to change the RF for topics in the kafka clusters, for the following reasons: 
* Strimzi supports the Cruise Control Service, and it's ready for use with simple configurations. 
* Cruise Control has native support for the RF changes. It automates the whole process, including selecting of new replicas and extra replicas, triggering the replica creation in the kafka brokers and monitoring the cluster during the process.

#### Kafka Topic Operator
Strimzi Topic Operator will watch the changes of `KafkaTopic` Resource. 

For any new topic RF change event, the `Strimzi Topic Operator` will validate the change. For instance, the new RF should be >= the min insync replica set size for the target topic. 

For the valid changes, the `Strimzi Topic Operator` will send requests to `Cruise Control`. The Kafka Topic Operator will periodically poll the status from the Cruise Control, and update the status in the `Status` field of the `KafkaTopic` resource.

For the invalid changes, the `Strimzi Topic Operator` will reject it and send the reason in the `Status` field of the `KafkaTopic` resources.

### Work flow
In the high level, the RF change will be performed in the following steps:
* Users modify the value of the `spec.replicas` properties in the target `KafkaTopic` resource.
* Kafka Topic Operator watches this resource changes and send the request to the Cruise Control Service.
* Kafka Topic Operator will periodically poll the result from the Cruise Control Service, and update the status in the `KafkaTopic` resource.
* Once it's completed by the Cruise Control Service, the Kafka Topic Operator will mark the Status in the `KafkaTopic` resource as `Completed`.

## Affected/not affected projects

The following components will be affected by this proposal:
* Strimzi Topic Operator
* Kafka Rebalance Operator

## Compatibility

There is no backward compatibility issue for this proposal, as there is no new persistent states introduced. 

The status of the RF changes will be cached in the memory of the `Strimzi Topic Operator`, and posted in the `Status` of the `KafkaTopic` resources.

This feature will be disabled, if `Cruise Control` is NOT enabled. Users will get the Error Response in the `Status` field of the `KafkaTopic` resource, if they intend to change the topic RFs.

## Rejected alternatives

Call out options that were considered while creating this proposal, but then later rejected, along with reasons why.
