<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Enhance KafkaBridge resource with consumer inactivity timeout and HTTP consumer/producer parts enablement

Providing support in the Strimzi Kubernetes Operator for following properties supported in the Strimzi HTTP Kafka Bridge:
* `http.timeoutSeconds`   - For deleting inactive consumers after a timeout (disabled by default).
* `http.consumer.enabled` - To enable/disable the HTTP consumer part (enabled by default).
* `http.producer.enabled` - To enable/disable the HTTP producer part (enabled by default.

## Current situation

Properties are not yet supported in the Strimzi Kubernetes Operator.

## Motivation

Raised in the discussion here: https://github.com/strimzi/strimzi-kafka-operator/issues/8732 and triaged on 29.6.2023: We should enable the configuration of these fields. But how should the API look like? Should have a proposal to clarify the API changes.

## Proposal

Proposal is implemented here: https://github.com/strimzi/strimzi-kafka-operator/pull/9820, both Paolo and Jakub suggested to raise a proposal for the implementation of `http.consumer.enabled` and `http.producer.enabled` specifically, as it can be done in two ways, motivated here: https://github.com/strimzi/strimzi-kafka-operator/pull/9820#discussion_r1523500115.

### Version 1 (which is currently implemented):

Adding dedicated section under the `http` properties for defining enablement of producer and consumer:
```yaml
spec:
  http:
    type: object
    properties:
      timeoutSeconds:
        type: integer
        description: The timeout in seconds for deleting inactive consumers.
      producer:
        type: object
        properties:
          enabled:
            type: boolean
            description: Whether the HTTP producer should be enabled or disabled.
        description: Configurations for the HTTP Producer.
      consumer:
        type: object
        properties:
          enabled:
            type: boolean
            description: Whether the HTTP consumer should be enabled or disabled.
        description: Configurations for the HTTP Consumer.
```

### Version 2:

Adding enablement of producer and consumer to their own sections, example:
```yaml
spec:
  consumer:
    type: object
    properties:
      enabled:
        type: boolean
        description: Whether the HTTP consumer should be enabled or disabled.
      config:
        x-kubernetes-preserve-unknown-fields: true
        type: object
        description: "The Kafka consumer configuration used for consumer instances created by the bridge. Properties with the following prefixes cannot be set: ssl., bootstrap.servers, group.id, sasl., security. (with the exception of: ssl.endpoint.identification.algorithm, ssl.cipher.suites, ssl.protocol, ssl.enabled.protocols)."
    description: Kafka consumer related configuration.
  producer:
    type: object
    properties:
      enabled:
        type: boolean
        description: Whether the HTTP producer should be enabled or disabled.
      config:
        x-kubernetes-preserve-unknown-fields: true
        type: object
        description: "The Kafka producer configuration used for producer instances created by the bridge. Properties with the following prefixes cannot be set: ssl., bootstrap.servers, sasl., security. (with the exception of: ssl.endpoint.identification.algorithm, ssl.cipher.suites, ssl.protocol, ssl.enabled.protocols)."
    description: Kafka producer related configuration.```

## Affected/not affected projects

PR raised for the HTTP Bridge to use the new ENV parameters introduced from the operator: https://github.com/strimzi/strimzi-kafka-bridge/pull/882

## Compatibility

None.

## Rejected alternatives

None.
