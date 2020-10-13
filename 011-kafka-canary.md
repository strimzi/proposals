# Strimzi Canary

Implement a Kafka canary tool which will act as an indicator of whether Kafka clusters are operating correctly. This would be achieved by creating a canary topic and periodically producing and consuming events on the topics. 

## Current situation

Currently  Strimzi does not have any canary style feature. It does have health checks for Kafka cluster which can be replaced/augmented with the canary.

## Motivation

The canary would provide the following:
  
  * Improved health check for Kafka clusters
  * SLI(s) for service admins
  * Metrics:
    * Kafka producer metrics
    * Kafka consumer metrics
    * Latency
    * Succesfull/failed events
    * Disk usage
    * Disk errors

## Proposal

The initial goal is hav the canary as an optional feature enabled during creation of a KAfka cluster. If enabled a single _canary_ topic would be created on creation of the cluster. The topic would have one partition for Kafka broker. Events would then be continously produced and consumed to/from the canary topic. The internal TLS 9091 port would be used to connect to the brokers. As this happens the relevant metrics would be collected and exported.

There is also option for the Kafka roller component to use the canary to aid in health checks. 


## Affected/not affected projects

This functionality would affect the Strimzi operators.

## Compatibility

The canary does not impact any existing functionality.


## Rejected alternatives

Another option would be to create an external canary client to provide much of the above functionality. It could be deployed separately to the Strimzi operator and connect to the brokers thourgh the external route. However the benefit of building this tool into the Strimzi operator is the abilty to use it as a health check for Kafka clusters.