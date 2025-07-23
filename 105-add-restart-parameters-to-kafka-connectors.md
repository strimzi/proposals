# Add restart parameters to Kafka connectors

In order to support [Kafka KIP-745](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181308623) we need to accept two more parameters on restart kafka connectors: `includeTasks` and `onlyFailed`.

## Current situation

Currently when adding annotation restart these two new parameters are always false when calling the Kafka connect API.

## Motivation

We should be able to support customizing this parameters then user can choose behavior according with your requirements.

## Proposal

If you take a look on other very famous products like nginx, they allow most of configs using [annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/). Then my proposal is to do the same here, and we should have the following:

strimzi.io/restart # already exist
strimzi.io/restart-include-tasks # new, with boolean value
strimzi.io/restart-only-failed # new, with boolean value
strimzi.io/restart-task # already exist

## Affected/not affected projects

- http://github.com/strimzi/strimzi-kafka-operator/. 

## Compatibility

We will keep default value as false for both variables, keeping backward compatibility for users not using the new annotations.

## Rejected alternatives

Nothing to add here.
