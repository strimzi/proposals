# Support of additional volumes in pod

We propose to enhance the Strimzi Kafka Operator by adding support for specifying additional volumes in the CRD. This will involve modifying the Kafka CRDs to include an `additionalVolumes` field, which will allow users to define extra volumes and volume mounts.

The specific usecase of ours, is that we want to set custom log4j logging for audit logs, to ensure that audit logs are *not* present in stdout, but can be picked up by a sidecar which uses OpenTelemtry or logstash to forward to the audit logs to a secure location.

## Current situation

The operator handles many configurations and operational aspects but lacks support for specifying additional volumes in the Custom Resource Definition (CRD). This limitation can restrict users from accessing log files or integrating with external storage solutions directly.

## Motivation

Adding support for additional volumes in the CRD will allow users to:

 - Access log files for enhanced monitoring and debugging.
 - Use additional configurations or secret management tools that require mounting extra volumes.

Improve flexibility in managing Kafka deployments by allowing users to attach custom storage solutions based on their specific requirements.

## Proposal

### Considerations

- CRD Changes: Update the Kafka CRDs to include an `additionalVolumes` field.
- Validation: Ensure that the additional volumes are validated correctly and do not conflict with existing volume definitions such as `/var/lib/kafka/` and `/tmp`.
- Documentation: Update the Strimzi documentation to include examples and guidelines on how to use the new `additionalVolumes` field.
- Security: Ensure that the additional volumes do not introduce security vulnerabilities by validating the configurations.
- Testing: Comprehensive testing to cover various scenarios and use cases, ensuring that the feature works as expected.

### Possible Delivery Mechanisms

- CRD Updates: Modify the existing CRDs to include the new field.
  - This will be done by extending the `PodTemplate.java` class to include the new `additionalVolumes` field as a list.
- Operator Logic: Update the operator logic to handle the additional volumes during the deployment process.
  - This should be possible to implement in the `KafkaCluster.java` class.

The configuration could look as follows:

```yaml
    additionalVolumes:
    - name: example-configmap
      path: /path/to/mount/volume
      #subPath: mykey # Add this to mount only a specific key of the configmap/secret
      configMap:
        name: config-map-name
    - name: temp
      path: /tmp/logs
      emptyDir: {}
    - name: example-csi-volume
      path: /path/to/mount/volume
      #subPath: "subpath" # Add this to mount the CSI volume at a specific subpath
      csi:
        driver: csi-driver-name
        readOnly: true
        volumeAttributes:
          secretProviderClass: example-secret-provider-class
    - name: example-projected-volume
      path: /path/to/mount/volume
      projected:
        sources:
          serviceAccountToken
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

## Compatibility

The proposed changes are designed to be forward-compatible. Future updates to the Strimzi Kafka Operator can build upon this feature without breaking existing configurations.
The new `additionalVolumes` field will be optional. Existing CRDs and configurations without this field will continue to work without any modifications. This ensures backward compatibility and seamless upgrades for current users.

## Rejected Alternatives

### Alternative 1: External Log Aggregation Tools
While using external log aggregation tools like ELK stack or Fluentd is a viable solution for accessing logs, it does not address the need for mounting additional volumes for other use cases like external storage integration or custom configurations such as sending different types of logs to different locations.

### Alternative 2: Custom Sidecar Containers for accessings logs through /tmp

Adding custom sidecar containers to handle logs through the `/tmp` volumes was considered. However, this approach does not seem to match the Strimzi design choices behind `/tmp` and there is the technical constraint of it relying on memory. This would introduce significant resource overhead and complexity to add more memory to the `/tmp` volume to support storing log files for a period of time.


