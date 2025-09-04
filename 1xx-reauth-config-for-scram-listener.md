# Support `connections.max.reauth.ms` configuration for SCRAM listeners

## Current situation

Apache Kafka provides the [`connections.max.reauth.ms`](https://kafka.apache.org/documentation/#brokerconfigs_connections.max.reauth.ms) configuration parameter which can be set at broker level for all listeners or a specific listener.

Currently, Strimzi doesn't have support for specifying such a parameter at both broker or listener level because the `listener.` prefix is part of the list of the forbidden configurations for the `spec.kafka.config` section.

However, this configuration is already supported for [OAuth](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationOAuth-reference) at listener level under `authentication` field and can be set for [`custom` type authentication](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationCustom-schema-reference) under `listenerConfig` field. 

## Motivation

The `connections.max.reauth.ms` is a useful configuration for SCRAM authentications enabled on a specific listener to define the maximum lifetime of a client connection before re-authentication is required. This ensures that clients with long lived connections periodically re-authenticate in case passwords already changed and reduces the risk of having connections with expired or compromised credentials. Since this is already supported/can be set for both OAuth and `custom` type authentications, it makes sense to support it for SCRAM authentications too.

## Proposal

`maxSecondsWithoutReauthentication` field already exists for `KafkaListenerAuthenticationOAuth` class, so for consistency, the same field will be added `KafkaListenerAuthenticationScramSha512` class. User can set the configuration shown in the following example:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: myproject
spec:
  kafka:
    # ...
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: true
        authentication:
          type: scram-sha-512
          maxSecondsWithoutReauthentication: 3600
```

## Affected/not affected projects

- strimzi-cluster-operator

## Compatibility

As this is a new configuration, there is no backwards compatibility issues relating to this proposal.

## Rejected alternatives
