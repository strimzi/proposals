# Add support `connections.max.reauth.ms` configuration for SCRAM listeners
This proposal adds support for configuring the Kafka `connections.max.reauth.ms` parameter at the listener level for SCRAM authentication in Strimzi.

## Current situation

Apache Kafka provides the [`connections.max.reauth.ms`](https://kafka.apache.org/documentation/#brokerconfigs_connections.max.reauth.ms) configuration parameter which can be set at broker level for all listeners or a specific listener.

Currently, Strimzi supports setting `connections.max.reauth.ms` broker-wide through `spec.kafka.config`. Per-listener settings cannot be configured this way, because `listener.` properties are forbidden.

The configuration is already supported at the listener level for [OAuth](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationOAuth-reference) through the  `authentication` field, and for [`custom` type authentication](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationCustom-schema-reference) through the `listenerConfig` field.

Note: OAuth configuration through `type: oauth` is deprecated. Configurations use `type: custom` with the appropriate OAuth settings instead.

## Motivation

The `connections.max.reauth.ms` is a useful configuration for SCRAM authentication enabled on a specific listener to define the maximum lifetime of a client connection before re-authentication is required. This ensures that clients with long-lived connections periodically re-authenticate in case passwords have changed. 
It also reduces the risk of having connections with expired or compromised credentials. 
Since this configuration is already supported for OAuth and `custom` type authentications, it makes sense to support it for SCRAM authentication too.

## Proposal

Allow users to configure `listener.name.<listener_name>.connections.max.reauth.ms` in `.spec.kafka.config` where `<listener_name>` is listener name and listener port is joined by a hyphen (`-`). 
To allow only this particular option but still forbid other listener options, another list of exceptions similar to `FORBIDDEN_PREFIX_EXCEPTIONS` will be added to `KafkaClusterSpec` class. 
This list will contain `connections.max.reauth.ms` for now but allows us to easily support more per-listener configurations in the future if we need to. 
The new list, along with the configured listener names will be used to dynamically build fully qualified configuration key (e.g., `listener.name.external-9094.connections.max.reauth.ms`).
These keys are then added to the existing exception list by the `KafkaConfiguration` class before the list is passed to the configuration filter that removes forbidden entries, allowing these specific configurations to be set.

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
      listener.name.external-9094.connections.max.reauth.ms: 3600000 
```

## Affected/not affected projects

- strimzi-cluster-operator

## Compatibility

As this is a new configuration, there are no backwards compatibility issues relating to this proposal.

## Rejected alternatives

- Add a new API field via `KafkaListenerAuthenticationScramSha512` class, so that it can be configured under `spec.kafka.listeners.listener.authentication`. This solution runs the risk of having too many configuration options like OAuth authentication, which is one of the reasons the `oauth` configuration type was deprecated.  

- Allow users to configure per-listener options in `.spec.kafka.config` by adding it to [`FORBIDDEN_PREFIX_EXCEPTIONS`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/api/src/main/java/io/strimzi/api/kafka/model/kafka/KafkaClusterSpec.java#L70) with some form of regular expression. 
However, this would require more complex validation logic to be applied to all configuration entries. 
It would also require non-trivial changes to the `AbstractConfiguration` class, which is shared by multiple resources such as `KafkaConnectConfiguration`. 
Per-listener configuration is only relevant to the `KafkaCluster` resource, so applying this logic globally would increase complexity and maintenance overhead. 
For these reasons, the proposed solution is simpler to implement and easier to maintain.

- This field can already be configured for `type: custom` authentication, so we don't need to add support for it. However, if existing users of SCRAM authentication, want to set this field, they would have to reconfigure their entire authentication section in their Kafka CR to change it from `type: scram` to `type: custom` and then change/set all the necessary fields for `type: custom`.