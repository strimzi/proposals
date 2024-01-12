#  Support tiered storage natively in Strimzi

## Current situation

Kafka introduced tiered storage feature since 3.6. The feature is in early stage and it’s nice to support tiered storage configuration natively via Strimzi.

## Motivation

Supporting tiered storage via Strimzi allow Strimzi users to enable tiered storage natively with Strimzi config. By allowing to config custom configuration for `RemoteStorageManager`, users are able to adopt tiered storage feature on Strimzi easily.

For simplicity, the proposal doesn't add support to customize `RemoteLogMetadataManager`. If tiered storage is enabled, the default implementation class `TopicBasedRemoteLogMetadataManager` in Kafka will be used.

This proposal focuses on tiered storage support with a custom plugin brought in by the user. Other proposals in the future might add some built-in support for selected storage implementations such as S3. It should also be noted that, it’s on the end-user to ensure that their tiered storage setup works with Strimzi; the operator itself is strictly responsible for pushing down the necessary config to the broker pods.

## Proposal

Add a `spec.kafka.tieredStorage` field in the `Kafka` CR, which would support configuration for all tiered storage related configuration.

The `tieredStorage` config contains configuration for some selected configuration of tiered storage feature. Most importantly, the configuration for `RemoteStorageManager` is needed.

```
kafka:
  tieredStorage:
    type: custom
    rsm.className:
    rsm.classPath:
    rsm.impl.prefix: rsm.config.
```

### RemoteStorageManager (RSM)

The RSM config only supports custom type. The config allows users to specify the RSM class information.

Because the `RemoteStorageManager` dependency is provided by Kafka plugin, the implementation could vary case by case. It’s important to allow user to pass in additional configuration values within the supplied prefix range. The custom configuration can be supplied via  `spec.kafka.config` with the predefined prefix.

### RemoteLogMetadataManager (RLMM)

For simplicity, the proposal doesn't add customization for RemoteLogMetadataManager. If tiered storage is enabled, the default implementation using `TopicBasedRemoteLogMetadataManager` will be used, and corresponding configuration will be appended. 

Since the default RLMM will create an internal Kafka client to talk to the server, there might be some additional configuration needed, like the SSL related setting. Those config can be put in `.spec.kafka.config` directly using the `rlmm.config.` prefix. Example can be found below.

## Example configuration

The below config define a sample configuration for tiered storage setup, using a `custom` RSM type.
```
kafka:
  tieredStorage:
    type: custom
    rsm.className: com.example.kafka.tiered.storage.s3.S3RemoteStorageManager
    rsm.classPath:  /opt/kafka/plugins/tiered-storage-s3/*
    rsm.impl.prefix: rsm.config.

  config:
    ...
    rlmm.config.remote.log.metadata.common.client.bootstrap.servers: broker-bootstrap.com:9091
    rlmm.config.remote.log.metadata.common.client.security.protocol: SSL
```

The configuration above will get translated to the below Kafka broker config:
```
remote.log.storage.system.enable: true
remote.log.storage.manager.class.name: com.example.kafka.tiered.storage.s3.S3RemoteStorageManager
remote.log.storage.manager.class.path: /opt/kafka/plugins/tiered-storage-s3/*
remote.log.storage.manager.impl.prefix: rsm.config.
# The RLMM config will be added by default
remote.log.metadata.manager.impl.prefix: rlmm.config.
remote.log.metadata.manager.class.name: org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager
# The RLMM config below are passed down from kafka config spec.
rlmm.config.remote.log.metadata.common.client.bootstrap.servers: broker-bootstrap.com:9091
rlmm.config.remote.log.metadata.common.client.security.protocol: SSL
```
## Docker image

This proposal doesn't include built in plugin support in any Strimzi built Kafka image. Users need to specify the path via the config, and customize the docker image to include the library.

An example to build such custom image is via a custom Dockerfile:

```
FROM quay.io/strimzi/kafka:latest-kafka-3.6.0
ARG REMOTE_PLUGIN_PATH
COPY ${REMOTE_PLUGIN_PATH} /opt/kafka/plugins/tiered-storage-s3/
```

## Testing strategy

Unit tests will be added to validate all the RLM,RSM, RLMM changes are propagated through.
Because the RSM and RLMM are typically provided by user and there is no built in support for any third party cloud provider, there won’t be any integration tests covering the cloud provider interaction. It’s the user’s responsibility to ensure the RSM and RLMM works with the intended cloud provider.

For testing purpose, an integration test with LocalTieredStorage implementation can be added to ensure end to end functionality.

## Affected Projects

Strimzi operator will be affected with the change.

## Compatibility

This proposal only add new configuration and API, so there is no backward compatibility issue for existing APIs.

## Alternate consideration

A simple way is to set everything via Helm chart config: https://github.com/strimzi/proposals/pull/98

That option is not favored because it's not applicable to usage without helm chart.

