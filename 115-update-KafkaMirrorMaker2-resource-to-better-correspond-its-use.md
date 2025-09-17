# Update `KafkaMirrorMaker2` resource structure to better correspond to its use

This proposal suggests several different changes to the `KafkaMirrorMaker2` custom resources.
These changes should allow better validation and should make the layout of the custom resource better correspond to how it really works.

## Current situation

The current `KafkaMirrorMaker2` resource was created when MirrorMaker 2 was introduced.
At that time, it was not necessarily clear what features will be added to it in the future and how it will be used.
We also decided that from the various options to run MirrorMaker 2, we will use Kafka Connect and manage the MirrorMaker 2 connectors on top of it as Connect connectors.

The following example shows a simple `KafkaMirrorMaker2` resource:

```yaml
kind: KafkaMirrorMaker2
metadata:
  name: my-mirror-maker-2
spec:
  version: 4.0.0
  replicas: 1
  connectCluster: "cluster-b"
  clusters:
  - alias: "cluster-a"
    bootstrapServers: cluster-a-kafka-bootstrap:9092
  - alias: "cluster-b"
    bootstrapServers: cluster-b-kafka-bootstrap:9092
    config:
      config.storage.replication.factor: -1
      offset.storage.replication.factor: -1
      status.storage.replication.factor: -1
  mirrors:
  - sourceCluster: "cluster-a"
    targetCluster: "cluster-b"
    sourceConnector:
      tasksMax: 1
      config:
        replication.factor: -1
        offset-syncs.topic.replication.factor: -1
        sync.topic.acls.enabled: "false"
        refresh.topics.interval.seconds: 600
    checkpointConnector:
      tasksMax: 1
      config:
        checkpoints.topic.replication.factor: -1
        sync.group.offsets.enabled: "false"
        refresh.groups.interval.seconds: 600
    topicsPattern: ".*"
    groupsPattern: ".*"
```

The `.spec.clusters` array contains a list of Kafka clusters this MirrorMaker instance connects to.
Each cluster in this array has its own unique alias.
The clusters include one or more source clusters from which the Kafka records will be mirrored.
And it also includes the target cluster (used by the underlying Kafka Connect).
The cluster that will be used by the underlying Connect cluster is specified by its alias in the `.spec.connectCluster` field.

The `.spec.mirrors` array contains the list of the different MirrorMaker connectors.
Each _mirror_ can specify target (`targetCluster`) and source cluster (`sourceCluster`) by referring to it through the `alias`.
However, the `targetCluster` has to always be only the same cluster which already serves as the Connect cluster.
Therefore the `targetCluster` field in the mirror specification has to be the same as the `.spec.Connect` cluster.

Each mirror can configure 3 different connectors:
* Source connector for mirroring Kafka records
* Checkpoint connector for mirroring committed offsets
* Heartbeat connector

As explained in [strimzi/strimzi-kafka-operator#11842](https://github.com/strimzi/strimzi-kafka-operator/issues/11842), the heartbeat does not really fit the pattern as it should be used in the opposite direction than the source and checkpoint connectors.
It is also rarely used by Strimzi users.

## Motivation

The current structure of the `KafkaMirrorMaker2` resources does not correspond to the current way it is used.
* The way the clusters are configured (as a Connect, source, and target clusters) makes it hard to validate in the CRD validation schema.
  We often can validate these things only in the operator.
  That often means users miss the errors as they are visible in the logs and the `.status` section of the custom resource but not as a result of the `kubectl` commands to create or change the resource.
* The `targetCluster` field in the mirror configuration has to be always the same value as the `.spec.connectCluster` and is not really _configurable_.
* It makes it hard to enforce and validate the configuration of the internal Connect topics and consumer group ID.
  See the [_Enforce the configuration of the internal Kafka Connect names and `group.id`_ proposal](https://github.com/strimzi/proposals/pull/176) for more details.
* The heartbeat connector is confusing to configure using the existing `KafkaMirrorMaker2` CR.
  The heartbeat connector should work in the opposite direction than the source connector.
  So if the source connectors is mirroring from `cluster-a` to `cluster-b` (the the `cluster-b` is the target and also the Connect cluster), the heartbeat connector needs to be configured to use the `cluster-a` as the target and Connect cluster.
  This is not possible in the current `KafkaMirrorMaker2` CR without overriding the source/target configurations directly in the connector `config` section.
  For more details, see [strimzi/strimzi-kafka-operator#11842](https://github.com/strimzi/strimzi-kafka-operator/issues/11842).

This proposal aims to improve these things.

## Proposal

This proposal suggests deprecating the following fields and remove them in the `v1` CRD API version:
* `.spec.connectCluster` field
* `.spec.clusters` list
* `heartbeatConnector` section in the mirror configuration (`.spec.mirrors[].heartbeatConnector`)
* `targetCluster` field in the mirror configuration (`.spec.mirrors[].targetCluster`)

These will be replaced with the following new fields:
* `.spec.targetCluster` section for configuring the cluster that will be used as target and Connect cluster
  The new `targetCluster` configuration will differ from the current `.spec.clusters` configuration:
    * New fields `groupId`, `configStorageTopic`, `statusStorageTopic`, and `offsetStorageTopic` will be added to configure the consumer group and internal topics names similar to Kafka Connect.
      Please check the [separate proposal](https://github.com/strimzi/proposals/pull/176) for the detailed motivation.
    * The `alias`, `bootstrapServers`, `tls`, `authentication` and `config` sections will remain unchanged.
* `.spec.sourceClusters` list for configuring one or more source clusters.
  The source cluster configuration itself will remain identical to the configuration we use today in `.spec.clusters`.

The `.spec.targetCluster`, `.spec.sourceClusters` and `.spec.mirrors` fields will be required in the `v1` CRD API.
Within the `.spec.targetCluster` section, the `bootstrapServers`, `groupId`, `configStorageTopic`, `statusStorageTopic`, and `offsetStorageTopic` will be required in both `v1beta2` and `v1` (this whole section will be optional in `v1beta2`, so this will not break backwards compatibility).
To allow early migration to the new API while the `v1beta2` API is still in use, the `spec.connectCluster` field will be changed to not be required anymore in `v1beta2` (it is completely removed in `v1`).

The following example shows the new `KafkaMirrorMaker2` layout:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaMirrorMaker2
metadata:
  name: my-mirror-maker-2
spec:
  version: 4.0.0
  replicas: 1
  targetCluster:
    bootstrapServers: cluster-c-kafka-bootstrap:9092
    groupId: my-mm2-group
    configStorageTopic: my-mm2-config
    statusStorageTopic: my-mm2-status
    offsetStorageTopic: my-mm2-offsets
    config:
      config.storage.replication.factor: -1
      offset.storage.replication.factor: -1
      status.storage.replication.factor: -1
  sourceClusters:
    - alias: "cluster-a"
      bootstrapServers: cluster-a-kafka-bootstrap:9092
    - alias: "cluster-b"
      bootstrapServers: cluster-b-kafka-bootstrap:9092
  mirrors:
    - sourceCluster: "cluster-a"
      sourceConnector:
        tasksMax: 1
        config:
          replication.factor: -1
          offset-syncs.topic.replication.factor: -1
          sync.topic.acls.enabled: "false"
          refresh.topics.interval.seconds: 600
      checkpointConnector:
        tasksMax: 1
        config:
          checkpoints.topic.replication.factor: -1
          sync.group.offsets.enabled: "false"
          refresh.groups.interval.seconds: 600
      topicsPattern: ".*"
      groupsPattern: ".*"
    - sourceCluster: "cluster-b"
      sourceConnector:
        tasksMax: 1
        config:
          replication.factor: -1
          offset-syncs.topic.replication.factor: -1
          sync.topic.acls.enabled: "false"
          refresh.topics.interval.seconds: 600
      checkpointConnector:
        tasksMax: 1
        config:
          checkpoints.topic.replication.factor: -1
          sync.group.offsets.enabled: "false"
          refresh.groups.interval.seconds: 600
      topicsPattern: ".*"
      groupsPattern: ".*"
```

The new fields will be immediately available to Strimzi users.
When both the new and old fields are configured, Strimzi will always prefer the configuration from the new fields.
During this phase, we would expect the custom resource to use either the new or the old API.
Mixed use of the APIs (for example, using `.spec.targetCluster` while also using `.spec.clusters`) will be treated as an error and the reconciliations will be failed with the `InvalidResourceException`.
The old configuration will remain fully supported as long as we support the `v1beta2` API.
Once we drop support for the `v1beta2` API, we will also clean the legacy code from the Strimzi Cluster Operator and support only the new API.

Users who decide not to migrate and use the new fields early will migrate to them when moving to the `v1` API.
This will happen automatically through the conversion tool or manually if following the manual conversion procedure.

### Documentation

The documentation will be updated to use the new fields in the `KafkaMirrorMaker2` examples and related sections.
The migration steps to use the new fields will also be part of the `v1` migration guide.
The documentation will also suggest that users who want to use the heartbeat connector can deploy it as a separate Kafka Connect + Connector combo.

### Examples

Our MirrorMaker 2 examples will be updated to use the new fields right when the new fields are introduced.

## Backwards compatibility

This proposal breaks backwards compatibility and forces users to change their `KafkaMirrorMaker2` custom resources at the latest when migrating to the `v1` CRD API version.
However, in most cases we expect that our users will use the conversion tool, so this change should be automated and should add minimal additional effort.

The previous design also allowed users to have two different cluster configurations for Connect and target cluster as long as they used the same bootstrap server.
The new design does not allow users to have a separate cluster configurations for Connect and for target clusters with the same bootstrap server.
However, the different cluster configurations were only partially used (because of the nature of the _source_ type connectors in Kafka Connect).
And we assume it is used by a minimal number of users.

Any users using this would need to migrate to a single cluster definition.
This is a change that cannot be handled automatically by the conversion tool.
So users will need to convert their resources manually.
But the conversion tool will detect this situation and request the user to update it manually.

## Rejected alternatives

There are currently no rejected alternatives.
