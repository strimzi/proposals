# Enhance KafkaBridge resource with consumer inactivity timeout and HTTP consumer/producer parts enablement

Providing support in the Strimzi Kubernetes Operator for following properties supported in the Strimzi HTTP Kafka Bridge:
* `http.consumer.timeoutSeconds`   - For deleting inactive consumers after a timeout (disabled by default).
* `http.consumer.enabled`          - To enable/disable the HTTP consumer part (enabled by default).
* `http.producer.enabled`          - To enable/disable the HTTP producer part (enabled by default).

## Current situation

Properties are not yet supported in the Strimzi Kubernetes Operator.

## Motivation

Raised in the discussion here: https://github.com/strimzi/strimzi-kafka-operator/issues/8732 and triaged on 29.6.2023: We should enable the configuration of these fields. But how should the API look like? Should have a proposal to clarify the API changes.

## Proposal

This proposal suggests based on discussions here: https://github.com/strimzi/strimzi-kafka-operator/pull/9820 to add the http enablement for the consumer and producer in their respective sessions, i.e. `spec.consumer.enabled` and `spec.producer.enabled`, as well as `spec.consumer.timeoutSeconds` property.

Suggestion:

```yaml
apiVersion: "kafka.strimzi.io/v1beta2"
kind: "KafkaBridge"
metadata:
  name: "test-kafka-bridge"
spec:
  replicas: 1
  image: "my-test-image"
  bootstrapServers: "my-cluster-kafka:9092"
  consumer:
    enabled: true
    timeoutSeconds: 60
    config:
      foo: "bar"
  producer:
    enabled: false
    config:
      foo: "buz"
  enableMetrics: false
```

## Affected/not affected projects

Strimzi Kafka Bridge - A PR has been raised for the HTTP Bridge to use the new ENV parameters introduced by the operator: https://github.com/strimzi/strimzi-kafka-bridge/pull/882

## Compatibility

Not specifying `spec.consumer.enabled` or `spec.producer.enabled` implies not configuring the corresponding bridge parameters `http.consumer.enabled` and `http.producer.enabled`, which default to true, thus there are no impacts on backward compatibility.

Not specifying `spec.consumer.timeoutSeconds` implies not configuring the corresponding bridge parameter `http.consumer.timeoutSeconds`, which defaults to -1 to have the same effect (no timeout), thus there are no impacts on backward compatibility.

## Rejected alternatives

### 1.
Initially in the implementation suggestion: https://github.com/strimzi/strimzi-kafka-operator/pull/9820, the enabled properties for producer and consumer were added to as part of the http properties, i.e. `spec.http.consumer.enabled` and `spec.http.producer.enabled`.

Alternative suggestion:

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
