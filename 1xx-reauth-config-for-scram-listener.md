# Add support `connections.max.reauth.ms` configuration for SCRAM listeners
This proposal adds support for configuring the Kafka `connections.max.reauth.ms` parameter at the listener level for SCRAM authentication in Strimzi.

## Current situation

Apache Kafka provides the [`connections.max.reauth.ms`](https://kafka.apache.org/documentation/#brokerconfigs_connections.max.reauth.ms) configuration parameter which can be set at broker level for all listeners or a specific listener.

Currently, Strimzi supports setting `connections.max.reauth.ms` broker-wide through `spec.kafka.config`. Per-listener settings cannot be configured this way, because `listener.` properties are forbidden.

The configuration is already supported at the listener level for [OAuth](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationOAuth-reference) through the  `authentication` field, and for [`custom` type authentication](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationCustom-schema-reference) through the `listenerConfig` field.

Note: OAuth configuration through `type: oauth` is planned to be deprecated. Future configurations would use `type: custom` with the appropriate OAuth settings instead.

## Motivation

The `connections.max.reauth.ms` is a useful configuration for SCRAM authentications enabled on a specific listener to define the maximum lifetime of a client connection before re-authentication is required. This ensures that clients with long lived connections periodically re-authenticate in case passwords already changed and reduces the risk of having connections with expired or compromised credentials. Since this configuration is already supported for OAuth and `custom` type authentications, it makes sense to support it for SCRAM authentications too.

## Proposal

The `maxSecondsWithoutReauthentication` field already exists for `KafkaListenerAuthenticationOAuth` class, so for consistency, the same field will be added to the `KafkaListenerAuthenticationScramSha512` class. Users can set the configuration as shown in the following example:

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

As this is a new configuration, there are no backwards compatibility issues relating to this proposal.

## Rejected alternatives

- Allow users configure per-listener options in `.spec.kafka.config` instead of adding a new API field. This can avoid ending up with too many various options like OAuth authentication, which is one of the reasons why it is getting deprecated. To implement this, one way would be to add this field to the [`FORBIDDEN_PREFIX_EXCEPTIONS`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/api/src/main/java/io/strimzi/api/kafka/model/kafka/KafkaClusterSpec.java#L70) with some form of regular expression, however, it could be quite an expensive check to do for all configurations. Given that there is not many other configurations that can be configured at listener level for SCRAM authentication according to the [Kafka doc](https://kafka.apache.org/documentation/#brokerconfigs), I think the risk of it ending up with too many API fields like OAuth did is small.

- This field can be already configured for `type: custom` authentication so we don't need to add support for it. However, if existing users of SCRAM authentication, want to set this field, they would have to reconfigure their entire authentication section in their Kafka CR to change it from `type: scram` to `type: custom` and then change/set all the neccessary fields for `type: custom`. 