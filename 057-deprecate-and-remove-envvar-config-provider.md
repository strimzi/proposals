# Deprecate and remove EnvVarConfigProvider

This proposes deprecating the [kafka-env-var-config-provider](https://github.com/strimzi/kafka-env-var-config-provider) project. It also proposes a timeline to archive the project and stop including the JAR in Strimzi releases.

## Current situation

In [proposal 30](https://github.com/strimzi/proposals/blob/main/030-env-var-config-provider.md), we added an implementation of the [ConfigProvider](https://kafka.apache.org/35/javadoc/org/apache/kafka/common/config/provider/ConfigProvider.html) interface of Apache Kafka that allows retrieving configuration values at runtime from environment variables. This implementation lives in the `kafka-env-var-config-provider` project. This is especially useful to safely configure sensitive settings in virtualized/containerized environments like Kubernetes.

For example, a Kafka client can use this provider with specified environment variables:
```properties
config.providers=env
config.providers.env.class=io.strimzi.kafka.EnvVarConfigProvider
option1=${env::FIRST_ENV_VAR}
```

## Motivation

Since Apache Kafka 3.5.0, there's a built-in `ConfigProvider` implementation that works with environment variables. It was added via [KIP-887](https://cwiki.apache.org/confluence/display/KAFKA/KIP-887%3A+Add+ConfigProvider+to+make+use+of+environment+variables). It provides the same functionality as Strimzi's implementation.

Example usage:
```properties
config.providers=env
config.providers.env.class=org.apache.kafka.common.config.provider.EnvVarConfigProvider
option1=${env:FIRST_ENV_VAR}
```

## Proposal

Once Strimzi only supports Kafka versions >= 3.5.0, we should:
- deprecate the [kafka-env-var-config-provider](https://github.com/strimzi/kafka-env-var-config-provider) project
- update all references to `io.strimzi.kafka.EnvVarConfigProvider` by `org.apache.kafka.common.config.provider.EnvVarConfigProvider`. This includes code, examples, documentation, etc

After 4 Strimzi releases have happened or Strimzi reached 1.0.0 (whichever comes first), we will archive the `kafka-env-var-config-provider` project and stop including the JAR in Strimzi releases.

## Affected/not affected projects

This impacts:
- [kafka-env-var-config-provider](https://github.com/strimzi/kafka-env-var-config-provider): To be deprecated and archived
- [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator): Update all references to `io.strimzi.kafka.EnvVarConfigProvider` to use Kafka's implementation

## Compatibility

Users relying on Strimzi's `EnvVarConfigProvider` will have to migrate to Kafka's implementation.

## Rejected alternatives

Do nothing and keep maintaining our own environment variable ConfigProvider implementation.
