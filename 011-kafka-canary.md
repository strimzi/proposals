# Strimzi Canary

Implement a Kafka canary tool which will act as an indicator of whether Kafka clusters are operating correctly. This would be achieved by creating a canary topic and periodically producing and consuming events on the topics. 

## Current situation

Currently Strimzi does not have any canary style feature. It does have health checks for Kafka cluster which can be replaced/augmented with the canary.

## Motivation

The canary would provide the following in indication that the Kafka cluster is operating as expecting from a customers perspective. I.e. messages can successfully be produced and consumed. Some would be exported to reflect the canary's activity. Initial metrics would included:

  * records-produced-total
  * records-consumed-total
  * produce-error
  * consume-error
  * latency

In future the canary can be used by the Kafka Roller to improve health checks. However this is beyond of scope of the current proposed work.

## Proposal

The canary would be run as separate pod alongside Strimzi operator. It would be specified (optionally) in the Kafka custom resource and deployed by the Strimzi controller. The code will live in a separate repository within the Strimzi project. This is similar setup to the Kafka Bridge. 

The canary will be built using Golang.

There are two main options for how the topic is configured:

### Option 1: Partition over replicas
Create a topic with 1 partition per broker node and a replication factor of zero. The canary would produce/consume from each partition. 

#### Pros

  * Errors will indicate that a specific broker is unavailable.

#### Cons

  * Brokers are expected to be unavailable during rolling updates after cluster configuration changes. 

### Option 2: Replicas over partition
Create a topic with a single parition and a replication factor equal to the number of brokers and a minimum in-sync replicas of one less than the number of brokers.

#### Pros

  * Allows for one (and only one) broker to be down, therefore accommodating rolling updates.
  * Mimics the availability that a user would expected.
  * Ensure that all brokers, minus one which can be down, are working as they need to be in-sync

#### Cons

  * Unsure what broker is down. This means if  the tool indicated that a broker is down for a period of time, we are unsure if it is the same broker (indicating a problem) or a different brokers as rolling update occurs (not a problem).


This proposal suggest going forward with option 2.

## Affected/not affected projects

This functionality would affect the Strimzi operators.

## Compatibility

The canary does not impact any existing functionality.


## Rejected alternatives

The option to build the canary tool directly into the Strimzi operator was considered. However, as the canary would be optional it was decided it is better to have it deployed as it's own pod. 

Java has been suggested as a language for the tool as it is the language of the Strimzi operator. However, as the tool will run in a separate pod, using Java would mean another JVM , increasing the resource usage of the tool.