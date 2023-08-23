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

Since Kafka 3.5, Kafka Connect connectors can also be stopped. This feature was added via [KIP-875](https://cwiki.apache.org/confluence/display/KAFKA/KIP-875%3A+First-class+offsets+support+in+Kafka+Connect). Stopping a connector is done by using a new REST endpoint `PUT /connectors/{connector}/stop`. Compared to paused where the connector and tasks stay instantiated, when stopped only the configuration of the connector is kept but nothing is actually running.

A paused connector still uses some resources but it's faster to resume. So using pause is well-suited if you want to halt data flowing for a short duration. For longer durations it could be beneficial to free up memory and other resources by fully stopping the connector with the new API. The stopped state will be required to use the offset delete/reset endpoints that should come with Kafka 3.6. For these reasons, Strimzi should support both pausing and stopping connectors.

The `PUT /connectors/{connector}/resume` REST endpoint is used to restart both paused and stopped connectors.

## Proposal

Since a connector can't be paused and stopped at the same time, the proposal is to replace the current `pause` property and add a new property, `state`, in the `AbstractConnectorSpec` schema. That new property will accept the `paused`, `stopped` or `running` values.

The `pause` property will be marked as deprecated. In the next Strimzi API version, the `pause` field will be deleted.

This feature requires at least Kafka 3.5 to work. In case it is merged when Strimzi still support older releases, a note in the documentation will be added to notify of this limitation. When trying to stop a connector with Kafka 3.4, the connector will instead be paused.

Example YAML for stopping a connector:
```yaml
spec:
  class: org.apache.kafka.connect.file.FileStreamSourceConnector
  config:
    file: /opt/kafka/LICENSE
    topic: my-topic
  state: stopped
  tasksMax: 1
```

This proposal does not intend to address deleting/resetting offsets. As explained in the motivation section, stopping connectors has value on its own. Adding support for deleting/resetting connector offsets will be tackled in a separate proposal once Strimzi adopts a Kafka version that supports this feature (expected to release in 3.6).

## Affected/not affected projects

This only affects `strimzi-kafka-operator`. This will be usable by regular connectors (via `KafkaConnectorSpec`) and by MirrorMaker connectors  (via `KafkaMirrorMaker2ConnectorSpec`).

## Compatibility

If `state` is not set, it will:
- default to `running` if `pause` is `false` or if `pause` is not set
- default to `paused` if `pause` is `true`
But this will not be done as a change to the custom resource, it will just be handled that way internally.

If `state` is set it will take precedence over the old `pause `field. If both fields are set, a warning will be emitted to notify the user of a potential conflict and a message will be added to the `condition` of the resource.

## Rejected alternatives

In the original issue I proposed adding a `stop` property alongside the existing `pause` property. As explained these states are mutually exclusive so it does not make sense to have both. 
