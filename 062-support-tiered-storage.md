#  Support tiered storage natively in Strimzi

## Current situation

Kafka introduced tiered storage feature since 3.6. The feature is in early stage and it’s nice to support tiered storage configuration natively via Strimzi.

## Motivation

Supporting tiered storage via Strimzi allow Strimzi users to enable tiered storage natively with Strimzi config. By allowing to config custom configuration for RemoteStorageManager and RemoteLogMetadataManager, users are able to adopt tiered storage feature on Strimzi easily.

This proposal focuses on tiered storage support with a custom plugin brought in by the user. Other proposals in the future might add some built-in support for selected storage implementations such as S3. It should also be noted that, it’s on the end-user to ensure that their tiered storage setup works with Strimzi; the operator itself is strictly responsible for pushing down the necessary config to the broker pods.

## Proposal

### RemoteStorage

Add a `tieredStorage` config under KafkaClusterSpec, which would support configuration for all tiered storage related configuration.

The CustomTieredStorage class contains configuration for various aspects, including RLM config, RSM specific config, RLMM specific config. 

```
kafka:
  tieredStorage:
      type: custom
      rlm:
        # See below detail
      rsm:
        # See below detail
      rlmm:
        # See below detail
```

### RLM

The RLM config supports `default` type only.  The config allows users to specify basic configuration for RemoteLogManager in Kafka:

```
      rlm:
        threadPoolSize:
        reader.threads:
        reader.maxPendingTasks:
```

### RSM

The RSM config only supports custom type. The config allows users to specify the RSM class information as well as other properties. 

Because the `RemoteStorageManager` dependency is provided by Kafka plugin, the implementation could vary case by case. It’s important to allow user to pass in additional configuration values within the supplied prefix range, thus config field is provided to pass in and generate such properties.

```
      rsm:
        type: custom
        class.name:
        class.path:  
        impl.prefix: rsm.config.
        config:
          property1:
          property2:
          ...
```

### RLMM

The RLMM config configuration for RemoteLogMetadataManager class. The config allows users to specify RLMM class information as well as other RLMM related propertied. A CustomRLMM class is supported to allow user to supply information on a custom RLMM.

In a custom implementation, Because the dependency is provided by Kafka plugin, the implementation could vary case by case. It’s important to allow user to pass in additional configuration values within the supplied prefix range, thus config field is provided to pass in and generate such properties. Those properties will be concatonated with the prefix value and get applied at Kafka config.

```
      rlmm:
        type: custom
        impl.prefix: rlmm.config.
        class.name:
        class.path:
        config:
          property1:
          property2:
          ...
```

RLMM has a built in implementation class called `TopicBasedRemoteLogMetadataManager` . This class is supported in Kafka by default, rlmm config will support another type called TopicBasedRLMM . 

The TopicBasedRemoteLogMetadataManager class has a internal Kafka client, talking to the cluster handling the metadata topic read/write operations. We need to allow users to set custom configuration for this Kafka client.

```
      rlmm:
        type:topicBased
        impl.prefix: rlmm.config.
        reader.threads:
        reader.max.pending.tasks:
        metadata.common.client.config:
          bootstrap.servers:
          security.protocol:
          additional-property1:
          additional-property2:
          ...
```

## Example configuration

The below config define a sample configuration for tiered storage setup, using a `custom` RSM type and `topicBased` type RLMM config.
```
kafka:
  tieredStorage:
      type: custom
      rlm:
        threadPoolSize: 10
        reader.threads: 10
        reader.maxPendingTasks: 100
      rsm:
        type: custom
        class.name: com.example.kafka.tiered.storage.s3.S3RemoteStorageManager
        class.path:  /opt/kafka/plugins/tiered-storage-s3/*
        impl.prefix: rsm.config.
        config:
          s3.bucket.name: my-bucket
          s3.region: us-west-2
          s3.fetch.pool.size: 10
          s3.upload.pool.size: 10
      rlmm:
        type:topicBased
        impl.prefix: rlmm.config.
        reader.threads: 10
        reader.max.pending.tasks: 100
        metadata.topic.num.partitions: 50
        metadata.common.client.config:
          bootstrap.servers: broker-bootstrap.example.com:9091
          security.protocol: SSL
          ssl.endpoint.identification.algorithm: ""
```

The configuration above will get translated to the below Kafka broker config:
```
remote.log.storage.system.enable: true
remote.log.reader.threads: 10
remote.log.reader.max.pending.tasks: 100
remote.log.manager.thread.pool.size: 10
remote.log.storage.manager.class.name: com.example.kafka.tiered.storage.s3.S3RemoteStorageManager
remote.log.storage.manager.class.path: /opt/kafka/plugins/tiered-storage-s3/*
remote.log.storage.manager.impl.prefix: rsm.config.
remote.log.metadata.manager.impl.prefix: rlmm.config.
remote.log.metadata.manager.class.name: org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager
remote.log.metadata.topic.num.partitions: 50
rlmm.config.remote.log.reader.threads: 10
rlmm.config.remote.log.reader.max.pending.tasks: 100
rlmm.config.remote.log.metadata.common.client.bootstrap.servers: broker-bootstrap.example.com:9091
rlmm.config.remote.log.metadata.common.client.security.protocol: SSL
rlmm.config.remote.log.metadata.common.client.ssl.endpoint.identification.algorithm: ""
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

