# Strimzi Canary

Implement a Kafka canary tool which will act as an indicator of whether Kafka clusters are operating correctly.
This would be achieved by creating a canary topic and periodically producing and consuming events on the topic. 

## Current situation

Currently Strimzi does not have any canary style feature.
It does have health checks for the Kafka cluster which can be replaced/augmented with the canary.

## Motivation

The canary would provide the following in indication that the Kafka cluster is operating as expected from a users perspective. I.e. messages can successfully be produced and consumed. 
Some would be exported to reflect the canary's activity. Initial metrics would included:
  * records-produced-total
  * records-consumed-total
  * produce-error
  * consume-error
  * latency

In future the canary can be used by the Kafka Roller to improve health checks. 
However this is beyond the scope of the current proposed work.

## Proposal

The canary would be run as a separate pod alongside the Kafka cluster. 
Once the canary topic is created messages will be produced and consumed from it. 
Messages would be consumed at a rate not faster than 1 per second and possibly as slow as 10 per second. This rate may be configurable. 

The canary will be built using Golang and metrics will be exposed in Prometheus format though a REST API. 
Currently Sarama is being considered as the client library.

The current plan is to deploy the canary independently  alongside the Kafka cluster. 
However, in future, consideration should be given to integrating the canary with Strimzi in a manner similar to the Kafka Exporter or Cruise Cotnrol. I.e. the canary would be specified (optionally) in the Kafka custom resource and then deployed by the Strimzi operator.

### Topic Configuration

The topic will be configured to have a partition on each broker node and a replication factor which will be the minimum of _number of Kafka broker nodes_ and _3 (which is the most commonly used value)_.
The minimum in-sync replicas will be either 1 in case replication factor is 1 (there is only one broker in the cluster) or one less than the number of replicas.
The configuration should be something like the following:

* Partitions = N (where N is the number of brokers)
* Replication Factor = Min(number-of-brokers, 3)
* Min ISR = Max(1, Replication Factor - 1)

About the storage sizing, the topic should use a smaller segment size and retention (in bytes) in order to avoid to fill the disk space.

#### Pros

* Allows for a broker to be down, therefore accommodating rolling updates.
* Mimics the availability that a user would expect.

#### Cons

* Unsure what broker is down. This means if the tool indicated that a broker is down for a period of time, we are unsure if it is the same broker (indicating a problem) or a different broker as rolling update occurs (not a problem).

#### Considerations for this approach

The canary tool must ensure when creating the topic that the partitions and replicas are spread across all brokers. E.g. If there are 10 brokers (meaning 10 partitions) and 3 replicas, all replicas must not be on only 3 brokers.

How the partitions are handled when the number of brokers scales must also be considered. 
When scaling up, partitions must be added on the new additional brokers so that each of them is leader for one new partition

However, when scaling down, partitions cannot be removed (the topic would have to be deleted and recreated). 
The solution to this would be to allow ‘orphaned’ partitions to remain.
Their leaders will be elected on some of the remaining brokers and the canary tool will not produce to them.
However, if the number of brokers scales up again, new leader elections (preferred leader election) must take place to put the partition leaders on the new brokers.

### Message scheme

The messages exchanged between producer and consumer running in the canary tool will use a well defined scheme based on JSON payload.

The message `key` could be null because the producer is going to specify the partition to which to send the message.
The message payload will be in JSON format carrying the following fields:

* `producerId`: producer identifier to enable running different producers in the tool if needed. For example, for covering different scenarios with more than one producer sending messages directly to the Kafka cluster from inside Kubernetes or going through Ingresses reaching the cluster from outside; this could leade to have different meaningful metrics values.
* `messageId`: identifier of the message sent in order to correlate the right one on the consumer side to evaluate metrics (i.e. latency).
* `timestamp`: time in milliseconds when the message was sent used to evaluate metrics (i.e. latency)

## Affected/not affected projects

The creation of the canary topic would affect the topic operator to create the corresponding KafkaTopic resource; we should avoid to do that.
The topic operator should be modified in order to filter out the canary topic. 

## Compatibility

The canary does not impact any existing functionality.

## Rejected alternatives

The option to build the canary tool directly into the Strimzi operator was considered.
However, as the canary would be optional it was decided it is better to have it deployed as it's own pod. 

Java has been suggested as a language for the tool as it is the language of the Strimzi operator.
However, as the tool will run in a separate pod, using Java would mean another JVM , increasing the resource usage of the tool.

Alternative configurations for the topic were also explored.

### Option 1: Partition over replicas

Create a topic with 1 partition per broker node and a replication factor of zero.
The canary would produce/consume from each partition.

#### Pros

* Errors will indicate that a specific broker is unavailable.

#### Cons

* Brokers are expected to be unavailable during rolling updates after cluster configuration changes.

### Option 2: Replicas over partition

Create a topic with a single partition and a replication factor equal to the number of brokers and a minimum in-sync replicas of one less than the number of brokers.

#### Pros

* Allows for one (and only one) broker to be down, therefore accommodating rolling updates.
* Mimics the availability that a user would expect.
* Ensure that all brokers, minus one which can be down, are working as they need to be in-sync

#### Cons

* Unsure what broker is down. This means if the tool indicated that a broker is down for a period of time, we are unsure if it is the same broker (indicating a problem) or a different broker as rolling update occurs (not a problem).

