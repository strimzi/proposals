# Support tiered storage in Strimzi

## Current situation

From 3.6.0, Apache Kafka supports tiered storage (as early access currently). We should add support for it to Strimzi.

## Motivation

This proposal provides a readily available approach to manage tiered storage feature through Strimzi, so existing Strimzi customer can use the described approach to add custom plugin and easily turn on tiered storage feature in Kafka.

## Proposal

To support for it to Strimzi, there are several aspects that would need to be clarified:

1. How should Strimzi manage tiered storage specific configurations?
2. How should customers add remote storage plugins into Strimzi build?
3. How should Strimzi manage tiered storage related secrets?


### Pass along tiered storage specific configuration

The easiest and already available way to pass tiered storage specific configurations to Kafka is through
Kafka.spec.kafka.config. Tiered storage feature need configurations specified for remote log manager, remote storage manager and remote log metadata manager, we can add these configurations in Strimzi provided helm chart and directly pass them to Kafka server.

~~~
remote.log.storage.system.enable: true
remote.log.reader.threads: {{ rlmReaderThreads }}
remote.log.reader.max.pending.tasks: {{ rlmReaderMaxPendingTasks }}
remote.log.manager.thread.pool.size: {{ rlmThreadPoolSize }}
remote.log.storage.manager.impl.prefix: rsm.config.        
remote.log.metadata.manager.impl.prefix: rlmm.config.
remote.log.metadata.manager.class.name: org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager
remote.log.metadata.topic.num.partitions: {{ rlmTopicPartitions }}
~~~

The same mechanism can be used to pass along custom plugin configuration.
~~~
# custom plugin configurations
rsm.config.s3.bucket.name: {{ s3BucketName }}
rsm.config.s3.region: us-west-2
rsm.config.s3.multipart.upload.part.size: {{ multiPartUploadBytes }}
rsm.config.s3.upload.part.size: {{ partUploadBytes }}
rsm.config.s3.io.buffer.size: {{ s3IOBufferSize }}
rsm.config.s3.worker.pool.size: {{ s3WorkerPoolSize }}
~~~

#### Add remote storage plugins into Strimzi build

Tiered storage feature need to load custom plugin in order to fill implementation details regarding how to upload/download segments to remote storage. The most straightforward way to accomplish the goal through Strimzi is:
1. Add an new docker layer of custom plugin on top of Strimzi provided kafka image, put custom plugin jars under desired path e.g. /opt/kafka/plugins.

~~~
#run Dockerfile from custom plugin code
FROM Strimzi_provided_kafka_image
COPY ./s3/build/install/s3/* /opt/kafka/plugins/tiered-storage-s3/
~~~

2. Specify the custom plugin path in through  Kafka.spec.kafka.config
~~~
remote.log.storage.manager.class.name: com.apple.aiml.kafka.tiered.storage.s3.S3RemoteStorageManager
remote.log.storage.manager.class.path: /opt/kafka/plugins/tiered-storage-s3/*
~~~

3. We can use the same mechanism to include future plugins for tiered storage and other Kafka custom feature.

### Manage tiered storage related secrets

1. For tiered storage client to access SASL enabled internal topic __remote_log_metadata , we can refer strimzi generated certificates through env variable CERTS_STORE_PASSWORD , strimzi will replace it with real password before passing to Kafka broker.
~~~
# tiered storage internal topic client auth configurations
rlmm.config.remote.log.metadata.common.client.client.id: {{ tieredStorageClientId }}
{{- if authorizationEnabled }}
rlmm.config.remote.log.metadata.common.client.bootstrap.servers: {{ clusterName }}-kafka-brokers:9091
rlmm.config.remote.log.metadata.common.client.security.protocol: SSL
rlmm.config.remote.log.metadata.common.client.ssl.endpoint.identification.algorithm: ""
rlmm.config.remote.log.metadata.common.client.ssl.keystore.location: /tmp/kafka/cluster.keystore.p12
rlmm.config.remote.log.metadata.common.client.ssl.keystore.password: ${CERTS_STORE_PASSWORD}
rlmm.config.remote.log.metadata.common.client.ssl.keystore.type: PKCS12
rlmm.config.remote.log.metadata.common.client.ssl.truststore.location: /tmp/kafka/cluster.truststore.p12
rlmm.config.remote.log.metadata.common.client.ssl.keystore.password: ${CERTS_STORE_PASSWORD}
rlmm.config.remote.log.metadata.common.client.ssl.truststore.type: PKCS12
{{- end }}
~~~

2. We can leave custom security configuration management to custom plugins themselves, they have the option to pass configuration through Kafka.spec.kafka.config , or choose from other mechanism they deem fit. E.g. in our S3 tiered storage plugin, S3 access from plugin is managed by service account through S3 role and policy, our s3 plugin does not have any secret configuration, instead it relies on DefaultAWSCredentialsProviderChain to use authentication and authorization provided by service account.

## Affected projects

With this proposal, we will update Strimzi Kafka Operator helm charts to include necessary tiered storage configurations, as well as add associate onboarding documentation to explain how to use.

## Compatibility

This proposal only add new configuration into helm charts, so there is no backward compatibility issue for existing users.