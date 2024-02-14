# JBOD support in KRaft mode

JBOD (Just a Bunch Of Disks) support in Apache Kafka means the ability to use multiple data directories in a single Kafka node to store the message logs.
Typically, the different data directories would represent different data volumes mounted on the Kafka node.
This proposal introduces JBOD support to KRaft-based Apache Kafka clusters powered by Strimzi.

## Current situation

Strimzi understands two different things under JBOD storage:

1. The `type: jbod` storage is a part of the `Kafka` custom resource API and allows users to define one or more disks in the form of a YAML list.
   The Strimzi storage type is not directly part of the configuration of the Apache Kafka nodes.
   It is only used as a basis to generate the value of the `log.dirs` configuration option that will contain one or more directories depending on the number of volumes defined in the `Kafka` CR.
   As a result, the Kafka nodes are not aware of the Strimzi storage type used in the `Kafka` CR.
   They are aware only of the number of directories / volumes used.
2. The actual use of the JBOD storage in the Apache Kafka nodes where multiple disks are used in each Kafka node.
   If the `log.dirs` configuration option contains more than one directory, JBOD storage is used.
   If it contains only one directory, JBOD storage is not used regardless of whether the `Kafka` CR uses `type: jbod` storage or not.

Until version 3.7.0, Apache Kafka nodes do not support the use of JBOD storage in KRaft mode.
When running Apache Kafka in KRaft mode with Strimzi, we support using the `type: jbod` storage, but only with one data volume present.

## Motivation

In version 3.7.0, Apache Kafka introduces support for JBOD storage in KRaft as _early-access_.
It is expected to be GA in Apache Kafka 3.8.0.
Use of JBOD storage is one of the popular features among Strimzi (and Apache Kafka) users and we want to support it.
This proposal defines how the support will be added to Strimzi.
It focuses on new KRaft-based Apache Kafka clusters.
If needed, a separate proposal might cover any changes related to the migration process of ZooKeeper-based Kafka clusters using JBOD storage to KRaft.

### JBOD support in KRaft-mode

From the user perspective, the main difference between JBOD support in ZooKeeper-based and KRaft-based clusters is in how the metadata are stored.
ZooKeeper-based Apache Kafka clusters store the metadata in ZooKeeper.
Kraft-based clusters store the metadata in a special metadata log.

The metadata log is present on all types of nodes - regardless of whether the nodes have the broker or controller roles.
The metadata log is located either in the directory specified by the `metadata.log.dir` configuration option or in the first directory from the `log.dirs` configuration option.
When the Apache Kafka node starts, it will find out what directory should be used to store the metadata based on its configuration.
It will also scan all the (meta)data directories to find out whether the metadata log is already present.
If the metadata log is already present and is not in the desired directory, the node startup will fail.
As a result, when changing the directory that should be used for the metadata log, any old metadata log present in one of the other directories should be deleted first.
There is currently no way how to _programmatically_ move the metadata log to another directory similarly to moving partition replicas.
While such feature might be added in the future, this proposal does not require it.

#### Strimzi challenges

The JBOD support in KRaft-based Kafka clusters provides several challenges that are addressed in this proposal:
* What level of control should the users have over the placement of the metadata log?
* How does Strimzi move or prevent moving of the metadata log between directories?
* Should Strimzi support dedicated metadata directories?
* Should Strimzi support JBOD storage in controller-only nodes?

These challenges are addressed in the next sections.

## Proposal

### API changes

A new optional property named `kraftMetadata` will be added to the volume definition in the Strimzi `Kafka` custom resource API.
This property will have an Enum value and the supported values will be:
* `null` (or not set)
* `shared`

The volume marked with `kraftMetadata: shared` will be used to store the metadata log.
In a `type: jbod` storage configuration, only one volume will be permitted to have the `kraftMetadata` property set to `shared`.
This will be validated in the Cluster Operator and when it is set for multiple volumes at the same time, an `InvalidResourceException` will be thrown.
It is also possible to not set this option for any volume as well.
In that case, the operator will decide which volume should be used for the metadata log on its own.

Following example shows how the configuration might look like in order to use the second volume to store the metadata log:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - broker
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        deleteClaim: false
      - id: 1
        type: persistent-claim
        size: 100Gi
        deleteClaim: false
        kraftMetadata: shared
```

The reason why the `kraftMetadata` property is not just a boolean is to make it possible to extend the support in the future.
One such possibility would be to support dedicated metadata volumes, which is discussed in a separate section.

The `kraftMetadata` option will be ignored when used with ZooKeeper-based clusters.

### Kafka configuration

Currently, we only use the `log.dirs` option to configure the log directories.
With this proposal, the `log.dirs` option will be configured in the same way as present.
In addition to that, we will also configure the `metadata.log.dir` option that we do not use today.
This option will be configured only for KRaft clusters.
It will be set to the volume marked as `kraftMetadata: shared`.
Or - if the flag is not set for any of the volumes - it will be set to the node with the lowest configured volume ID.

The currently used `metadata.log.dir` will be also stored in an annotation on the `StrimziPodSet` representing the node pool.
This annotation will be used to detect changes to the metadata log volume and used to log such events.
In the future, if necessary, it could also be used, for example, to prevent changes or to reassign the metadata log in a different way.
However, such functionality is not planned as part of this proposal.

### Changing the metadata log directory / volume

The volume / directory used for the metadata log might change in several situations:
* When a user changes the `kraftMetadata` flag and moves it to another volume
* When the volume originally used for the metadata is removed
* When a new volume with lower volume ID is added to the JBOD list and the `kraftMetadata` flag is not set

Changes to the `metadata.log.dir` always require rolling update of the affected Kafka nodes.
The change of the directory will be handled in the Kafka start-up scripts in our container image.
The scripts will do the following:
1) Try to find the existing metadata log
2) Compare the current location of the metadata log with the desired location
3) If the locations differ, the old metadata log will be deleted
4) The broker is started and either continues to use the existing metadata log or creates a new one and syncs it from the (other) controller nodes

This is a similar to a situation when a new node is started after a scale-up or to recover from a storage failure.

### JBOD usage in controller-only nodes

Controller-only nodes host only the metadata log.
They do not host any regular partition replicas.
The metadata log is always placed in a single directory / volume.
As such, there is no real value of using multiple volumes in controller-only nodes in terms of any performance improvements.
But using multiple volumes for a short period of time might be used to address various situations such as changing the type of the volume used for the metadata etc.
Therefore, Strimzi will not prevent users from configuring multiple volumes for controller-only nodes.
But it will issue a warning when multiple volumes are specified for controller-only nodes.

### Dedicated metadata value

On brokers or on Kafka nodes with mixed roles, the volume / directory used for the metadata log will be used to also host regular partition-replicas.
In some situations, this might not be desired as the metadata log might compete with the regular Kafka logs for storage or performance.
Technically, it is possible to use a different data volume for the metadata log and for the regular Kafka logs.
However, it is not expected that this would be a common configuration because:
* Clusters sensitive to performance are expected to use dedicated controller-only nodes.
  And on the controller-only nodes, there are no regular partition-replicas.
  So the metadata log has its own dedicated volume even without any special support for it.
* On broker-only nodes, the impact of shared volumes for data and metadata is expected to be small.
  And using a dedicated volume just for a metadata is not expected to be very efficient in terms of the utilization of the dedicated disk.

Based on the above, there will be no support for dedicated metadata log volumes.
This should help us to save unnecessary effort on the development and testing of this feature.

However, if needed, the support for it can be added later if we see the demand for it:
* Update the API and add a new valid value `dedicated` to the `kraftMetadata` field in the storage definition
* Configure this volume only as the `metadata.log.dir` and do not use it in `log.dirs`
* Add a validation to prevent changing of an existing volume used for regular logs to metadata-only log without all partition-replicas being moved to other volumes first.

### Example YAML files

The examples YAML files will be adapted to cover the API changes:
* A new KRaft JBOD example will be added with multiple disks and the `kraftMetadata` used for one of them
* The existing KRaft example with a single persistent volume will be adapted to add the `kraftMetadata: shared` option to the custom resource

### Gating

The JBOD storage will be supported only for KRaft clusters running Kafka 3.7.0 and newer with KRaft metadata set to at least `3.7`.
Since the JBOD support is expected to be GA in Kafka 3.8.0, there is no plan to introduce a feature gate for it.
Documentation and the YAML examples will warn users that JBOD storage is only in early access in Kafka 3.7.0.

#### Risks

The JBOD storage is in early access only in Kafka 3.7.0.
The testing of it in the 3.7.0 release candidates discovered several bugs.
It is also not guaranteed that there will be no changes to how it works in Kafka 3.8.0 (for example required to address the bugs etc.).
However, as JBOD support is one of the last missing KRaft features, and since Kafka 3.8.0 is expected to be the last version with ZooKeeper support, we cannot delay the implementation of the JBOD support until Kafka 3.8.0 is released with the final implementation.

## Affected projects

This proposal affects only the Strimzi Cluster Operator.
No other components are affected.

## Backwards compatibility

This proposal has no impact on any existing features or on the JBOD support in current ZooKeeper-based clusters.
The changes implemented by this proposal have also no impact on the current `Kafka` custom resources and existing KRaft clusters.
They will work without requiring any changes.

## Not part of this proposal

This proposal does not include any possible impact of JBOD storage for ZooKeeper to KRaft migration.

## Rejected alternatives

### Feature gate for JBOD support

Adding a new feature gate for the JBOD support was considered.
It might be useful to protect users from unintentionally using JBOD storage with Kafka 3.7.0 without being aware that it is an early access only.
However, this idea was rejected because:
* The feature gate would be very short-lived as the JBOD storage is expected to GA in the next Kafka version
* The feature gate does not address the different levels of support across different Kafka versions
    * It would not prevent use of JBOD storage in Kafka 3.6, which lacks support for JBOD storage entirely
    * Once the feature gate is enabled by default after support for Kafka 3.8.0 is added, it would not prevent Kafka 3.7.0 users from using JBOD storage without knowing that it is early access only

### Supporting JBOD storage only with Kafka 3.8.0

Another considered option was to add the support for JBOD only from Kafka 3.8.0.
However, this would significantly limit the amount of testing we would be able to do and prevent us from discovering the issues early and having them fixed in Kafka 3.8.0.

### Block removal of the metadata log volume

One of the considered options was to block removal of the volume used for the metadata log instead of automatically moving it to another volume.
This would be technically possible since it does not significantly differ from the current storage validation checks.
However, the only way to move the metadata log to another volume is by deleting the directory.
So it seems like the end result would be the same regardless of whether the user does it by deleting the volume or by changing the `kraftMetadata` flag.
So it does not seem to be necessary to block this.
