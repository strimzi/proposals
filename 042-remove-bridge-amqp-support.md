# Remove AMQP 1.0 support from the Strimzi bridge

This proposal is about removing the current support for the AMQP 1.0 protocol from the Strimzi bridge, leaving just the HTTP support.

## Current situation

Currently the Strimzi bridge provides support for the AMQP 1.0 protocol other than HTTP.
The AMQP 1.0 protocol support is provided by using the [Vert.x Proton](https://github.com/vert-x3/vertx-proton) component for handling the communication on the wire.
The current implementation doesn't follow any specific standard, like the new available [Event Stream Extensions for AMQP Version 1.0](https://docs.oasis-open.org/amqp/event-streams/v1.0/csd01/event-streams-v1.0-csd01.html).

## Motivation

The AMQP 1.0 protocol support seems not to be used extensively by the Strimzi community.
We haven't ever seen users opening GitHub issues or discussions about adding new features or fixing bugs on the AMQP 1.0 part.
The same applies on the CNCF Slack #strimzi channel and the mailing lists.
Even asking users about their AMQP 1.0 usage didn't get any answer.
It seems that the Strimzi bridge is mostly used for its HTTP protocol support, which is where we see more engagement from the users.
Right the AMQP 1.0 support is just bringing one more Vert.x dependency, the Vert.x Proton component, which seems to be useless.
Removing it allows to reduce the dependencies surface which could be impacted by bugs and CVEs.

## Proposal

This proposal suggests to remove the AMQP 1.0 protocol support from the Strimzi bridge.
It involves removing different parts of the codebase:

* The specific AMQP 1.0 related part in the `io.strimzi.kafka.bridge.amqp` package.
* The part which is used for tracking offsets for AMQP 1.0 in the `io.strimzi.kafka.bridge.tracker` package.
* Simplifying or even removing the `SinkBridgeEndpoint` and `SourceBridgeEndpoint` classes that currently are used as a common layer across AMQP 1.0 and HTTP for interacting with the Apache Kafka cluster.
* All the corresponding tests.
* All the documentation about the AMQP 1.0 support design and usage (see `amqp` folder) and configuration for using the [Qpid Dispatch Router](https://qpid.apache.org/components/dispatch-router/index.html) with it (see `qdrouterd` folder).

In the future, but without any actual roadmap or ETA for it, we could come back to have a support for AMQP 1.0 to Apache Kafka bridging with a separate component under the Strimzi organization and by implementing the [Event Stream Extensions for AMQP Version 1.0](https://docs.oasis-open.org/amqp/event-streams/v1.0/csd01/event-streams-v1.0-csd01.html) OASIS standard.

## Affected/not affected projects

This proposal affects the Strimzi bridge project only.
The Strimzi operator is not affected because currently it is not possible to enable the AMQP 1.0 support on the bridge through the `KafkaBridge` custom resource.

## Compatibility

When the removal is completed, the users won't be able to use the AMQP 1.0 protocol support with future bridge releases.
If they really need that, the only way is to stick with older bridge versions.

## Rejected alternatives

No rejected alternatives to mention.
