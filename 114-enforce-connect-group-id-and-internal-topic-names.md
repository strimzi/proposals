# Enforce the configuration of the internal Kafka Connect names and `group.id`

This proposal suggests to make the following Apache Kafka Connect options required and force the users to configure them:
* `group.id`
* `offset.storage.topic`
* `config.storage.topic`
* `status.storage.topic`

This will better align Strimzi with Connect and provide a better user experience.

## Current situation

Apache Kafka Connect uses 3 internal topics for storing:
* Configurations
* Statuses
* Offsets

Each Kafka Connect cluster has also its own `group.id` configuration to specify the `group.id` it would use.

When running Kafka Connect with Strimzi, users can configure these options in `.spec.config` section.
For example:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster
spec:
  # ...
  config:
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
    # ...
```

These options are required by Connect itself.
But they are only optional in Strimzi
And when they are not set, Strimzi will use its own static defaults:
* `group.id` defaults to `connect-cluster`
* `offset.storage.topic` defaults to `connect-cluster-offsets`
* `config.storage.topic` defaults to `connect-cluster-configs`
* `status.storage.topic` defaults to `connect-cluster-status`

## Motivation

The static defaults do not work very well because they are the same for every single Kafka Connect cluster.
And when multiple Connect clusters (i.e. multiple different `KafkaConnect` resources) that use the same Kafka cluster use the same defaults, they will not be independent anymore but form one big Connect cluster.
Unfortunately, this big Connect cluster does not work, because the different parts cannot communicate with each other.

In the past, after we realized this issue, we tried to mitigate this problem:
* We improved the [docs](https://strimzi.io/docs/operators/latest/full/deploying.html#con-config-kafka-connect-multiple-instances-str)
* We improved our examples to include the values

This helped to increase the awareness.
However we were not able to set the fields as required.
And we were not able to change the defaults.

With the migration to the [`v1` CRD API](https://github.com/strimzi/proposals/pull/174), we might have a chance to further improve things and make these options required.
While this will not ensure that users properly configure them to a different values for each Connect cluster, it will at least make sure these options are fully visible in the custom resource and users are aware of them.

## Proposal

We will introduce 4 new fields in the `.spec` section of the `KafkaConnect` resource.
These fields will be:
* `groupId`
* `configStorageTopicName`
* `statusStorageTopicName`
* `offsetStorageTopicName`

These fields will be optional in the `v1beta2` API.
But they will be required in the `v1` API.

As long as the `v1beta2` API will be supported (presumably up to and including Strimzi 0.51 - subject to change based on the [`v1` CRD API](https://github.com/strimzi/proposals/pull/174) proposal), the new fields will be treated as the primary source of the configuration.
When not set, the operator will try to use the corresponding values from `.spec.config`.
And when even the corresponding values in `.spec.config` would be not set, the operator will use the defaults.
That way, the existing `KafkaConnect` `v1beta2` resources will continue to work as before without any breaking change.

When we remove the `v1beta2` support and continue to use only the `v1` API (presumably in Strimzi 0.52 / 1.0.0 - subject to change based on the [`v1` CRD API](https://github.com/strimzi/proposals/pull/174) proposal), the new fields will be always required.
The corresponding values in `.spec.config` would be added to the list of forbidden values and the will be ignored.
However, we cannot make these values forbidden in the `v1` CRD schema as the `.spec.config` field is unstructured map.
So if the users have them set there, they will remain there (but will be ignored).
And the defaults values used by the operator when the `.spec.config` values were not set will be completely removed from the operator.

The existing `v1beta2` resources will be updated during the CRD `v1` conversion.
* If the values are set in `.spec.config`, they will be taken from there and used to set the new fields `groupId`, `configStorageTopicName`, `statusStorageTopicName`, and `offsetStorageTopicName`.
* If the values are not set in `.spec.config`, the  fields `groupId`, `configStorageTopicName`, `statusStorageTopicName`, and `offsetStorageTopicName` fields will be set to the default values used by the operator.

This conversion will be done automatically by the `v1` CRD Conversion Tool.
And it will be also included in the manual instructions provided to users who do not want to use the conversion tool.
This way, we make sure that all resources are updated before moving to the `v1` API and the group ID and topic names are set in all `KafkaConnect` resources.
While the conversion has to happen latest when migrating to the `v1` API before upgrading to Strimzi 0.52.0 / 1.0.0, users can start using the APIs already earlier while `v1beta2` is still present.

The other options (such as the configuration of the replication factor for the internal topics) will not be affected and will remain part of the `.spec.config` section.
The new `KafkaConnect` resource will look for example like this:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
spec:
  # ...
  groupId: my-connect-group
  configStorageTopicName: my-connect-config
  statusStorageTopicName: my-connect-status
  offsetStorageTopicName: my-connect-offsets
  config:
    config.storage.replication.factor: -1
    offset.storage.replication.factor: -1
    status.storage.replication.factor: -1
    # ...
```

### Documentation

The documentation will be updated to use the new fields in the `KafkaConnect` examples and related sections.
The migration steps to use the new fields will be also part of the `v1` migration guide.

### Examples

Our Connect examples will be updated to use the new fields instead `.spec.config` right when the new fields are introduced.

## Out of scope

While the same issue affects also our `KafkaMirrorMaker2` resource, it is not part of this proposal.
MirrorMaker 2 will have its own proposal covering changes similar to those described in this PR as well as some additional changes.

## Backwards compatibility

This proposal breaks backwards compatibility and forces users to change their `KafkaConnect` custom resources at the latest when migrating to the `v1` CRD API version.
However, in most cases we expect that our users will use the conversion tool, so this change should be automated and should add minimal additional effort.

## Rejected alternatives

### Enforcing the `.spec.config` fields using CEL rules

Enforcing the presence of the following fields in `.spec.config` using CEL validation rules was considered:
* `group.id`
* `offset.storage.topic`
* `config.storage.topic`
* `status.storage.topic`

However, the CEL validation rules cannot be used to enforce specific fields in unstructured maps such as `.spec.config`.
So this alternative is not technically possible.
