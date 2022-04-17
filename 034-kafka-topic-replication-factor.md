# Support Replication Factor Change for Kafka Topics

Apache Kafka introduced the Replication Factor (RF) as the configuration that determines the number of replicas kafka brokers will retain for a single data partition. A bigger number of the RF means both higher reliability for the data partition and higher infrastructure cost for topics.

Apache Kafka allows users to change the RF for any topic partition through a partition reassignment process in the runtime. However, it's not a simple process, because users usually need to decide which brokers to host new replicas and which replicas need to be dropped, or even conduct a cluster rebalance after the change of the RF.

The goal of this proposal is to offer users an easy approach to change the topic RF through the KafkaTopic Custom Resources and Cruise Control, in the cloud native, strimzi style.

## Current situation

Currently, users have two options to change the topic RF:
* Leverage the Native Kafka script to change the RF. Users need to generate the partition replica reassignment and balance the kafka clusters manually if needed. Clearly it's not as simple as it should be for the users.
* Delete the topic and recreate the topic with the desired RF. It's not acceptable because of losing data, unless the user accepts that.

When users attempt to change the RF in the strimzi KafkaTopic Resources, the Status of the KafkaTopic will become `NotReady` with `ReplicationFactorChangeException`. The following error message is shared with users.

`Changing 'spec.replicas' is not supported. This KafkaTopic's 'spec.replicas' should be reverted to 4 and then the replication should be changed directly in Kafka.`

## Motivation

With this new feature, it will become much easier to achieve the desired tradeoff between the cost and data reliability in the kafka clusters. 

For instance, users can lower the RFs for selective less critical topics to reduce the infrastructure cost (Compute, Storage and Network). 

Also, for some critical kafka topics, users can increase the data reliability through the bump of the RFs ONLY for these specific topics. 

In addition, users can temporally adjust the RF number to deal with Storage Capacity Shortage. For instance, we could avoid Disk Full Exceptions by temporally decrease the RF number for some topics. 

More importantly, users only need to specify the new RF number in the `KafkaTopic` resource, and it's as simple as changing any properties for a normal k8s Custom Resource. The heavy lifting work, such as new replica placement, obsolete replica deletion and possible cluster rebalance, is completely taken care by the strimzi and Cruise Control. 

## Proposal

### API

#### Strimzi KafkaTopic CRD
The `spec.replicas` property in the `KafkaTopic` CRD will be changed from ReadOnly to ReadWrite Mode. Any change to this `spec.replicas` property from users will trigger an RF change for the targeting kafka topic.

Under the hood, the Strimzi Topic Operator will watch the changes of `spec.replicas` from the `KafkaTopic` CR, interact with Cruise Control Service and return the result in the `Status` field of the `KafkaTopic` CR.

#### Strimzi Topic Operator
Since the Topic Operator needs to send requests to the Cruise Control Service for the RF changes, the Topic Operator should know the details of the Cruise Control Service Endpoint.

We plan to introduce the following Environment Variables in the Topic Operator. These Environment Variables will be set up by the EntityTopicOperator.

```java
public static final String TC_CRUISECONTROL_ENABLED = "STRIMZI_CRUISECONTROL_ENABLED";
public static final String TC_CRUISECONTROL_NAMESPACE = "STRIMZI_CRUISECONTROL_NAMESPACE";
public static final String TC_CRUISECONTROL_HOST = "STRIMZI_CRUISECONTROL_HOST";
public static final String TC_CRUISECONTROL_API_AUTH_ENABLED = "STRIMZI_CRUISECONTROL_API_AUTH_ENABLED";
public static final String TC_CRUISECONTROL_API_SSL_ENABLED = "STRIMZI_CRUISECONTROL_API_SSL_ENABLED";
public static final String TC_CRUISECONTROL_SECRET_NAME = "STRIMZI_CRUISECONTROL_SECRET_NAME";
public static final String TC_CRUISECONTROL_API_SECRET_NAME = "STRIMZI_CRUISECONTROL_API_SECRET_NAME";
```

If the Cruise Control Service is not enabled in the Kafka Cluster, this change of RF will be disabled. Users will be notified with an error message.

### Design
We propose to change the topic RFs with the support of `Cruise Control` and `Strimzi Topic Operator`. 

#### Cruise Control
Cruise Control Service will be leveraged to change the RF for topics in the kafka clusters, for the following reasons: 
* Strimzi supports the Cruise Control Service, and it's ready for use with simple configurations. 
* Cruise Control has native support for the RF changes. It automates the whole process, including selecting of new replicas and obsolete replicas, triggering the replica creation in the kafka brokers and monitoring the cluster during the process.

To be specific, the [Kafka Topic Configuration Endpoint](https://github.com/linkedin/cruise-control/wiki/REST-APIs#change-kafka-topic-configuration) from the Cruise Control Service will be leveraged to change the RF for Kafka Topics.

#### KafkaTopic Operator
Strimzi `KafkaTopic` Operator will watch the changes of `KafkaTopic` Resources.

For any new topic RF change event, the `Strimzi Topic Operator` will validate the change. It must satisfy the following conditions:
* The new RF should be bigger or equal to the min in-sync replica set size for the target topic.
* The new RF should be smaller or equal to the number of brokers.
* The new RF should not be bigger than the 5 (Upper Bound).

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
* Strimzi Topic Operator: The Strimzi Topic Operator will drive the whole RF change process.
* Kafka Rebalance Operator: We need to factor out the interaction with Cruise Control Service to the Operator-Common Module, so that they can be shared by Strimzi Topic Operator.

## Compatibility

There is no backward compatibility issue for this proposal, as there is no new persistent states introduced. 

The status of the RF changes will be cached in the memory of the `Strimzi Topic Operator`, and posted in the `Status` of the `KafkaTopic` resources.

This feature will be disabled, if `Cruise Control` is NOT enabled. Users will get the Error Response in the `Status` field of the `KafkaTopic` resource, if they intend to change the topic RFs.

## Rejected alternatives

* Leverage the State Machines to manage the state during this process, and it's the same way as how KafkaRebalance Operator works today. The main reason is that the RF change is a very lightweight process, compared with KafkaRebalance, and adding the State Machines is likely to make this process more complex than necessary.