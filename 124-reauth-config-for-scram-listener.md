# Add support `connections.max.reauth.ms` configuration for SCRAM listeners
This proposal adds support for configuring the Kafka `connections.max.reauth.ms` parameter at the listener level for SCRAM authentication in Strimzi.

## Current situation

Apache Kafka provides the [`connections.max.reauth.ms`](https://kafka.apache.org/documentation/#brokerconfigs_connections.max.reauth.ms) configuration parameter which can be set at broker level for all listeners or a specific listener.

Currently, Strimzi supports setting `connections.max.reauth.ms` broker-wide through `spec.kafka.config`. Per-listener settings cannot be configured this way, because `listener.` properties are forbidden.

The configuration is already supported at the listener level for [OAuth](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationOAuth-reference) through the  `authentication` field, and for [`custom` type authentication](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationCustom-schema-reference) through the `listenerConfig` field.

Note: OAuth configuration through `type: oauth` is deprecated. Configurations use `type: custom` with the appropriate OAuth settings instead.

## Motivation

The `connections.max.reauth.ms` is a useful configuration for SCRAM authentications enabled on a specific listener to define the maximum lifetime of a client connection before re-authentication is required. This ensures that clients with long lived connections periodically re-authenticate in case passwords already changed and reduces the risk of having connections with expired or compromised credentials. Since this configuration is already supported for OAuth and `custom` type authentications, it makes sense to support it for SCRAM authentications too.

## Proposal

Allow users configure `listener.<listener_name>.connections.max.reauth.ms` in `.spec.kafka.config` where `<listener_name>` is listener name and listener port joined by hyphen (`-`). To allow only this particular option but still forbid other listener options, another list of exceptions similar to `FORBIDDEN_PREFIX_EXCEPTIONS` will be added to `KafkaClusterSpec` class. This new list will be used to build a set of prefixes for each listener, which will be then added to the existing forbidden prefix exceptions list. The list will contain `connections.max.reauth.ms` for now but allows us to easily support more per-listener configurations in the future if we need to.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: myproject
spec:
  kafka:
    listeners: 
      - name: external
        port: 9094
        type: route
        tls: true
        authentication:
          type: scram-sha-512
    config:
      listener.external-9094.connections.max.reauth.ms: 3600000 
```

## Affected/not affected projects

- strimzi-cluster-operator

## Compatibility

As this is a new configuration, there are no backwards compatibility issues relating to this proposal.

## Rejected alternatives

- Add a new API field via `KafkaListenerAuthenticationScramSha512` class so it can be configured under `spec.kafka.listeners.listener.authentication`. This solution has a risk of ending up with too many various options like OAuth authentication did, which is one of the reasons why it is getting deprecated. 

- Allow users configure per-listener options in `.spec.kafka.config` by adding it to [`FORBIDDEN_PREFIX_EXCEPTIONS`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/api/src/main/java/io/strimzi/api/kafka/model/kafka/KafkaClusterSpec.java#L70) with some form of regular expression, however, it could be quite an expensive check to do for all configurations. This also requires a fair amount of changes in `AbstractConfiguration` class, where it filters the forbidden prefix and apply the exceptions. This class is used by various other resources (e.g. KafkaConnectConfiguraiton) and per-listener configuration is only relevant to KafkaCluster resource. Therefore the proposed solution seems simpler to implement and cheaper compared to this one.

- This field can be already configured for `type: custom` authentication so we don't need to add support for it. However, if existing users of SCRAM authentication, want to set this field, they would have to reconfigure their entire authentication section in their Kafka CR to change it from `type: scram` to `type: custom` and then change/set all the neccessary fields for `type: custom`.