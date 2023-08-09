<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Support stopping connectors

This proposal aims at adding support for stopping Kafka Connect connectors.

## Current situation

At the moment Strimzi allows pausing connectors. This is done by a property, `pause`, in the `KafkaConnectorSpec` schema. Whenever it is set to `true`, the connectors and its tasks are paused.

For example:
```yaml
spec:
  class: org.apache.kafka.connect.file.FileStreamSourceConnector
  config:
    file: /opt/kafka/LICENSE
    topic: my-topic
  pause: true
  tasksMax: 1
```

## Motivation

Since Kafka 3.5, Kafka Connect connectors can also be stopped. This is done by using a new REST endpoint `PUT /connectors/{connector}/stop`. Compared to paused where the connector and tasks stay instantiated, when stopped only the configuration of the connector is kept but nothing is actually running.

A connector paused still uses some resources but it's faster to resume. So using pause is great if you want to halt data flowing for a short duration. For longer durations it's interesting to fully stop the connector with the new API. Also the stopped state will be required to use the offset delete/reset endpoints that should come with Kafka 3.6. For these reasons, Strimzi should support both pausing and stopping connectors.

The `PUT /connectors/{connector}/resume` REST endpoint is used to restart both paused and stopped connectors.

## Proposal

Since a connector can't be paused and stopped at the same time, the proposal is to replace the current `pause` property and add a new property, `state`, in the `AbstractConnectorSpec` schema. That new property will accept the `pause`, `stop` or `run` values.

The `pause` property will be marked as deprecated. In the next Strimzi API version, the `pause` field will be deleted.

This proposal does not intend to address deleting/resetting offsets. As explained in the motivation section, stopping connectors has value on its own. Adding support for deleting/resetting connector offsets will be tackled in a separate proposal once Strimzi adopts a Kafka version that supports this feature (expected to release in 3.6).

## Affected/not affected projects

This only affects `strimzi-kafka-operator`. This will be usable by regular connectors (via `KafkaConnectorSpec`) and by MirrorMaker connectors  (via `KafkaMirrorMaker2ConnectorSpec`).

## Compatibility

If `state` is not set, it will:
- default to `run` if `pause` is `false` or if `pause` is not set
- default to `pause` if `pause` is `true`
But this will not be done as a change to the custom resource, it will just be handled that way internally.

If `state` is set it will take precedence over the old `pause `field. If both fields are set, a warning will be emitted to notify the user of a potential conflict.

## Rejected alternatives

In the original issue I proposed adding a `stop` property alongside the existing `pause` property. As explained these states are mutually exclusive so it does not make sense to have both. 
