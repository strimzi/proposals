#  Support tiered storage natively in Strimzi

## Current situation

Kafka introduced tiered storage feature since 3.6. The feature is in early stage and it’s nice to support tiered storage configuration natively via Strimzi.

## Motivation

Supporting tiered storage via Strimzi allow Strimzi users to enable tiered storage natively with Strimzi config. By allowing to config custom configuration for RemoteStorageManager and RemoteLogMetadataManager, users are able to adopt tiered storage feature on Strimzi easily.

It should also be noted that, it’s on the end-user to ensure that their tiered storage setup works with Strimzi; the operator itself is strictly responsible for pushing down the necessary config to the broker pods.

## Proposal

### RemoteStorage

Add a RemoteStorage class under storage config (under https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaClusterSpec-schema-reference), which would support configuration for all tiered storage related configuration.

The RemoteStorage class contains configuration for various aspects, including system config, RSM specific config, RLMM specific config.

```
kafka:
  storage:
    remote:
      enabled: true
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
        class.name:
        class.path:
        reader.threads:
        reader.max.pending.tasks:
        metadata.common.client.config:
          bootstrap.servers:
          security.protocol:
          additional-property1:
          additional-property2:
          ...
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

