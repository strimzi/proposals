# Support of additional volumes in pod

We propose to enhance the Strimzi Kafka Operator by adding support for specifying additional volumes in the CRDs. This will involve modifying the Kafka CRDs (see details below) to include a `volumes` field which will allow users to define extra volumes, and a `volumeMounts` field which will allow users to define extra volume mounts.

The specific usecase of ours, is that we want to set custom log4j logging for audit logs, to ensure that audit logs are *not* present in stdout, but can be picked up by a sidecar which uses OpenTelemtry or logstash to forward to the audit logs to a secure location.
See motivation for more use-cases and see also original discussion in <https://github.com/strimzi/strimzi-kafka-operator/issues/3693>

## Current situation

The operator handles many configurations and operational aspects but lacks support for specifying additional volumes in the Custom Resource Definition (CRD). This limitation can restrict users from accessing log files or integrating with external storage solutions directly.

## Motivation

Adding support for additional volumes in the CRD will allow users to:

 - Access log files for enhanced monitoring and debugging.
 - Use additional configurations or secret management tools that require mounting extra volumes.
 - Using for analyzing JVM issues (e.g. heap dumps)

And prepare support for future use-cases such as:

- Using Kafka Connect connectors to read/write files to/from the disk
- Using it for tiered storage
- CSI Secrets
- Adding plugins to the container image

Improve flexibility in managing Kafka deployments by allowing users to attach custom storage solutions based on their specific requirements.

## Proposal

### Considerations

- CRD Changes: Update the Kafka CRDs to include a `volumes` and a `volumeMounts` field.
- Validation: All additional mounted paths should be inside `/mnt` to ensure backwards compatibility for the evolution of kafka and this operator.

- Documentation: Update the Strimzi documentation to include examples and guidelines on how to use the new `volumes` and `volumeMounst` fields.
- Security: Ensure that the additional volumes do not introduce security vulnerabilities by validating the configurations.
- Testing: Comprehensive testing to cover various scenarios and use cases, ensuring that the feature works as expected.

### CRD Changes

Add a new `volumes` field to PodTemplate and a new `volumeMounts` field to ContainerTemplate. Update the associated code to create and mount the specified volumes. Support will be added for specifying `volumes`/`volumeMounts` in the PodTemplate/ContainerTemplate in the following locations:

|Pod              |CRD              |Schema Location                                                                       |Implementing class           |
|-----------------|-----------------|--------------------------------------------------------------------------------------|-----------------------------|
|Kafka            |Kafka            |spec -> kafka -> template -> pod/kafkaContainer/initContainer                         |KafkaCluster.java            |
|                 |KafkaNodePoolSpec|spec -> template -> pod/kafkaContainer/initContainer                                  |                             |
|Zookeeper        |Kafka            |spec -> zookeeper -> template -> pod/zookeeperContainer                               |ZookeeperCluster.java        |
|EntityOperator   |Kafka            |spec -> entityOperator -> template -> pod/topicOperatorContainer/userOperatorContainer|EntityOperator.java          |
|CruiseControl    |Kafka            |spec -> cruiseControl -> template -> pod/cruiseControlContainer                       |CruiseControl.java           |
|KafkaExporter    |Kafka            |spec -> kafkaExporter -> template -> pod/container                                    |KafkaExporter.java           |
|KafkaConnect     |KafkaConnect     |spec -> template -> pod/connectContainer/initContainer/buildContainer                 |KafkaConnectCluster.java     |
|KafkaBridge      |KafkaBridge      |spec -> template -> pod/bridgeContainer/initContainer                                 |KafkaBridgeCluster.java      |
|KafkaMirrorMaker2|KafkaMirrorMaker |spec -> template -> pod/connectContainer/initContainer/buildContainer                 |KafkaMirrorMaker2Cluster.java|


It is proposed to not add support in the following locations - if `volumes` or `volumeMounts` are specified in the PodTemplate or ContainerTemplate in any of these locations it will be ignored:

|Pod             |CRD             |Schema Location                                          |Reason                                 |
|----------------|----------------|---------------------------------------------------------|---------------------------------------|
|KafkaMirrorMaker|KafkaMirrorMaker|spec -> template -> pod/mirrorMakerContainer             |KafkaMirrorMaker has been deprecated   |
|Kafka           |Kafka           |spec -> jmxTrans -> template -> pod/container            |JmxTrans no longer supported           |
|EntityOperator  |Kafka           |spec -> entityOperator -> template -> tlsSidecarContainer|tlsSidecarContainer no longer supported|
|CruiseControl   |Kafka           |spec -> cruiseControl -> template -> tlsSidecarContainer |tlsSidecarContainer deprecated         |


An example configuration could look as follows:

```yaml
    kind: Kafka
    spec:
      kafka:
        template:
          pod:
            volumes:
            - name: example-configmap
              configMap:
                name: config-map-name
            - name: temp
              emptyDir: {}
            - name: example-csi-volume
              csi:
                driver: csi-driver-name
                readOnly: true
                volumeAttributes:
                  secretProviderClass: example-secret-provider-class
            - name: example-projected-volume
              projected:
                sources:
                  serviceAccountToken
          kafkaContainer:
            volumeMounts:
            - name: example-configmap
              path: /path/to/mount/cm-volume
            - name: temp
              path: /tmp/logs
            - name: example-csi-volume
              path: /path/to/mount/csi-volume
            - name: example-projected-volume
              path: /path/to/mount/proj-volume

```

### Supported volumes

The following types of volumes are proposed as initial support. More types could be added later.

- Secret
- ConfigMap
- EmptyDir
- CSI
- Projected volume

See initial work in this PR draft:
<https://github.com/strimzi/strimzi-kafka-operator/pull/10099>

## Affected/not affected projects

<https://github.com/strimzi/strimzi-kafka-operator>

Affected operands/components as described in table in "CRD Changes" section above

## Compatibility

The proposed changes are designed to be forward-compatible. Future updates to the Strimzi Kafka Operator can build upon this feature without breaking existing configurations.
The new `volumes` / `volumeMounts` fields will be optional. Existing CRDs and configurations without this field will continue to work without any modifications. This ensures backward compatibility and seamless upgrades for current users.


## Rejected Alternatives

### Alternative 1: External Log Aggregation Tools
While using external log aggregation tools like ELK stack or Fluentd is a viable solution for accessing logs, it does not address the need for mounting additional volumes for other use cases like external storage integration or custom configurations such as sending different types of logs to different locations.

### Alternative 2: Custom Sidecar Containers for accessings logs through /tmp

Adding custom sidecar containers to handle logs through the `/tmp` volumes was considered. However, this approach does not seem to match the Strimzi design choices behind `/tmp` and there is the technical constraint of it relying on memory. This would introduce significant resource overhead and complexity to add more memory to the `/tmp` volume to support storing log files for a period of time.


