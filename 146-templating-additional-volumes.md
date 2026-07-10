# Templating Additional Volumes for Pods with Stable Identities

## Current Situation

Strimzi currently supports configuring [additional volumes](https://strimzi.io/docs/operators/latest/full/configuring.html#con-common-configuration-volumes-reference) that can be mounted to any Pod operated by Strimzi.
This is a useful feature that allows users to add their own Secrets, ConfigMaps, PersistentVolumeClaims, etc.

## Motivation

However, additional volumes cannot be configured at the level of individual Pods or nodes.
They are configured at the level of `Deployment` (for HTTP Bridge, Cruise Control, Kafka Exporter, and Entity Operator) or `StrimziPodSet` (for Kafka, Kafka Node Pool, Connect, and MirrorMaker 2).
As a result, the exact same volume would be mounted into all Kafka nodes belonging to a given node pool, all nodes belonging to the same Connect cluster, etc.

However, in some situations, users might want to mount different volumes into different Pods.
This might include:
* Mounting Secrets or ConfigMaps with per-broker configuration or credentials.
  The credentials or configuration from a third-party service, such as authorization or authentication, might be stored in different Secrets or ConfigMaps or in the same Secret or ConfigMap under different keys.
* Mounting PersistentVolumes through PersistentVolumeClaims for storing local data that should be persisted when `emptyDir` volumes cannot be used.
  This could include a local cache for authorization or authentication plugins or audit trails.
  Although a single shared storage volume might work in some cases, it can be suboptimal because data from multiple nodes would share the same volume.
  This makes it hard to manage the data on disk.
  It also increases the blast radius when the disk becomes full because it affects all nodes.
  Shared storage that is available across multiple availability zones might also not always be supported on user's infrastructure.
  Per-Pod PersistentVolumes would address these issues by providing clear isolation between brokers.
  PersistentVolumes are also already a prerequisite for running Strimzi for Apache Kafka log directories, so they should be available in all Kubernetes cluster environments.

This proposal suggests how Strimzi can allow users to _template_ additional volumes and configure different volumes for different nodes.

## Proposal

A new section `templatedVolumes` will be added to the `pod` template sections of the `Kafka`, `KafkaNodePool`, `KafkaConnect`, and `KafkaMirrorMaker2` custom resources.
The `templatedVolumes` section will support configuring:
* Secret-based volumes
* ConfigMap-based volumes
* PersistentVolumeClaim-based volumes

However, unlike regular _additional volumes_ in the `volumes` field, templated volumes will not support:
* EmptyDir volumes
* CSI volumes
* Image volumes

For `emptyDir` and `image` volumes, there does not seem to be a need to support templating.
`emptyDir` volumes are unique for each Pod by design, so there is no need for further templating.
Image volumes are typically used for adding plugins, and there is currently no clear use case for different image volumes in different Pods controlled by the same custom resource.

For [CSI volumes](https://kubernetes.io/docs/concepts/storage/volumes/#csi), templating might make sense.
However, CSI volume configuration is very generic and the actual values depend on the specific CSI plugin, so it is not clear at this point which fields should be templated.
CSI volumes are out of scope for this proposal and might be added later.

### Templating

Templating will be applied only to the actual volumes.
It will not apply to volume names, which can remain the same for each Pod.
It will also not be applied to volume mount templates, which reference the volumes by volume name.

Not all fields of the three supported volume types need to be supported.
Templating will be applied only to the following fields:
* `claimName` for PersistentVolumeClaim volumes
* Secret name and the item keys and paths for Secret volumes
* ConfigMap name and the item keys and paths for ConfigMap volumes

Templating will use the existing variables Strimzi already uses when templating the host and advertised host fields in listener configuration:
* The `{nodeId}` variable will be replaced with the ID of the node to which the template is applied.
* The `{nodePodName}` variable will be replaced with the Kubernetes pod name for the node where the template is applied.

The following example shows what the template section might look like:

```yaml
    template:
      pod:
        templatedVolumes:
          - name: authz-creds
            secret:
              # Same Secret, but different keys per node
              secretName: authz-creds
              items:
                - key: api-key-{nodeId}
                  path: api.key
          - name: authz-config
            configMap:
              # Different ConfigMap per node
              name: "{nodePodName}-authz-config"
          - name: audit-trail
            persistentVolumeClaim:
              # Different PVC per node
              claimName: audit-trail-{nodeId}
      kafkaContainer:
        # Unchanged by this proposal
        volumeMounts:
          - name: authz-creds
            mountPath: /mnt/authz-creds
          - name: authz-config
            mountPath: /mnt/authz-config
          - name: audit-trail
            mountPath: /mnt/audit-trail/
```

### Implementation Details

The new templated volumes will be added through a new class, `AdditionalTemplatedVolumes`.
The class will be very similar to the existing `AdditionalVolumes` class, but will contain only the selected volume types as discussed in the previous sections.
Using a separate class allows Strimzi to clearly define the volume types supported for templating in the API.

Currently, all components use the existing `PodTemplate` class, whether they are stateless Pods managed through `Deployment` or stateful Pods managed through `StrimziPodSet`.
However, only the stateful Pods should have the new templated volumes field.
Therefore, a new class, `StatefulPodTemplate`, will be added as well.
This class will extend the original `PodTemplate` class and will add the templated volumes as a new field.
The existing `PodTemplate` class will continue to be used for stateless Pod templates.
The new `StatefulPodTemplate` class will be used for stateful Pod templates.

The `TemplateUtils` class will be extended and new method `addAdditionalTemplatedVolumes(...)` will be added.
This method be responsible for templating and creating the additional templated volumes.
The existing `TemplateUtils.addAdditionalVolumes(...)` method will remain unchanged.
In the places where we prepare the volumes for non-stateful Pods, `TemplateUtils.addAdditionalVolumes(...)` will be called as today without any change.
In the places where we prepare the volumes for stateful Pods, the code will call both `TemplateUtils.addAdditionalVolumes(...)` and `TemplateUtils.addAdditionalTemplatedVolumes(...)` to add both the regular as well as the templated volumes.

`TemplateUtils.addAdditionalVolumes(...)` has already validation for duplicate volume names.
When duplicate volume name is found, an `InvalidResourceException` exception is raised.
The same will be done also in `TemplateUtils.addAdditionalTemplatedVolumes(...)`.
As we fail the reconciliation with exception in case of duplicate volume names, we can just run both methods one after each other and do not need to decide which one has priority.

#### Handling Conflicts

Templating itself does not create any new conflicts because it only controls which ConfigMap, Secret, or PVC is mounted.
The volume name itself will not be templated.
The volume name will still be checked for uniqueness and validity, the same as with regular additional volume names.

## Affected Projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards Compatibility

This proposal is fully backward compatible.
The existing additional volume configurations will continue to work without any changes.

## Rejected Alternatives

### Use the Existing Additional Volumes for Templating

One rejected alternative was to enable templating on the existing additional volumes.
We would not need a new API for the templated volumes.
Strimzi would simply always check the existing additional volume fields for template variables and replace them if they are present.
Because the variables are unique and unlikely to be present in regular volumes, this should not cause backward compatibility issues.

However, this might be confusing for users because templating could be applied only to some Pods (Kafka, Connect, and MirrorMaker 2) and not to others.
Therefore, this option is rejected.
It can be revisited if people prefer it.
