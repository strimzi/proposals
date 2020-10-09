# Strmixi Canary

Implement a Kafka canary tool which will act as an indicator of whether Kafka clusters are operating correctly. This would be achieved by creating a canary topic and continously producing and consuming events on the topics. 

## Current situation

Currently  Strimzi does not have any canary style feature. It does have readiness checks for Kafka cluster which can be replaced/augmented with the canary.

## Motivation

The canary would provide the following:
  
  * Improved readiness check for Kafka clusters
  * SLI(s) for service admins
  * Metrics:
    * End-to-end latency
    * Succesfull/failed events
    * Disk usage
    * Disk errors

## Proposal

The canary would be implemented as part of the Kafka Roller component. 
A single _canary_ topic would be created on creation of a Kafka cluster. The topic would have one partition for Kafka broker. Events would then be continously produced and consumed to/from the canary topic. The internal TLS 9091 port would be used to connect to the brokers.

As this happens the relevant metrics would be collected and exported. Errors such as the inability to produce to the topic can then be used by the KafkaRoller to trigger a restart, acting as healthc check for the Kafka cluster. 


## Affected/not affected projects

This functionality should not affect other Strmizi projects.

## Compatibility

The canary would be fully backwards compatible.


## Rejected alternatives

Another option would be to create an external canary client to provide much of the above functionality. It could be deployed separately to the Strimzi operator and connect to the brokers thourgh the external route. However the benefit of building this tool into the Strimzi operator is the abilty to use it as a health check for Kafka clusters.