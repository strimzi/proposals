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

Proposal is implemented here: https://github.com/strimzi/strimzi-kafka-operator/pull/9820, both Paolo and Jakub suggested to raise a proposal for the implementation of `http.consumer.enabled` and `http.producer.enabled` specifically, as it can be done in two ways, current status is that you can just enable/disable the usage of consumer or producer on the HTTP side, which means being or not being able to use HTTP to producer and/or consume (so by using the corresponding Kafka part).
Question is whether the enablement should be located as part of the `http` properties, or in each of the corresponding `consumer` or `producer` properties.

### Version 1 (which is currently implemented):

Adding dedicated section under the `http` properties for defining enablement of producer and consumer:
```yaml
apiVersion: "kafka.strimzi.io/v1beta2"
kind: "KafkaBridge"
metadata:
  name: "test-kafka-bridge"
spec:
  replicas: 1
  image: "my-test-image"
  bootstrapServers: "my-cluster-kafka:9092"
  http:
    timeoutSeconds: 60
    producer:
      enabled: false
    consumer:
      enabled: true
  consumer:
    config:
      foo: "bar"
  producer:
    config:
      foo: "buz"
  enableMetrics: false
```

### Version 2:

Adding enablement of producer and consumer to their own sections, example:
```yaml
apiVersion: "kafka.strimzi.io/v1beta2"
kind: "KafkaBridge"
metadata:
  name: "test-kafka-bridge"
spec:
  replicas: 1
  image: "my-test-image"
  bootstrapServers: "my-cluster-kafka:9092"
  http:
    timeoutSeconds: 60
  consumer:
    enabled: true
    config:
      foo: "bar"
  producer:
    enabled: false
    config:
      foo: "buz"
  enableMetrics: false
```

## Affected/not affected projects

PR raised for the HTTP Bridge to use the new ENV parameters introduced from the operator: https://github.com/strimzi/strimzi-kafka-bridge/pull/882

## Compatibility

None.

## Rejected alternatives

None.
