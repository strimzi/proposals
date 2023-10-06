# KRaft upgrades and downgrades

This proposal introduces the Kafka upgrade mechanism for KRaft clusters.
It describes how will Strimzi upgrade or downgrade KRaft clusters from one Apache Kafka version to another.

## Current situation

Strimzi currently does not support Apache Kafka upgrades / downgrades on KRaft clusters.
The expectation is that the KRaft-based Apache Kafka clusters are deleted and freshly created both when upgrading / downgrading the Strimzi Cluster Operator as well as when upgrading / downgrading Apache Kafka from one version to another.

### ZooKeeper-based clusters

Strimzi currently supports upgrades of ZooKeeper-based Kafka clusters.
The upgrades of ZooKeeper-based clusters are multi-stage process:
1. First, ZooKeeper nodes are rolled to use the new container image from the new Kafka version and possibly also the new ZooKeeper version (if the new Kafka version uses new ZooKeeper version).
2. Next, Kafka brokers are rolled to use the new container image with the new Apache Kafka version.
3. Finally, the Kafka brokers are rolled one more time to update the `inter.broker.protocol.version` (in case the `inter.broker.protocol.version` changed).
   This step is done either when the user changes the `inter.broker.protocol.version` in `Kafka.spec.kafka.config`.
   Or automatically in the next reconciliation when `inter.broker.protocol.version` is not set at all in `Kafka.spec.kafka.config`.

Strimzi also supports downgrades of ZooKeeper-based Kafka clusters.
The downgrades of ZooKeeper-based clusters are multi-stage process:
1. If the `inter.broker.protocol.version` is set to a higher version then supported by the Kafka version being downgraded to, it has to be set to a version supported by the older Kafka version.
   This step has to be done manually by the user by setting the version in `Kafka.spec.kafka.config`.
   The Cluster Operator will roll the Kafka brokers to set the new value but will still use the container images of the newer Kafka version.
   Strimzi is validating the `inter.broker.protocol.version` and will not proceed with the next steps until a version compatible with the older Kafka version is used.
2. Next, ZooKeeper nodes are rolled to use the new container image from the older Kafka version and possibly also the older ZooKeeper version (if the older Kafka version uses older ZooKeeper version).
3. Finally, the Kafka brokers are rolled one more time to us the container image with the older Kafka version.

The upgrade/downgrade happens in two situations:
* When the Kafka version is changed in in `Kafka.spec.kafka.version`
* When the Strimzi Cluster operator is upgraded / downgraded and uses a different default version of Apache Kafka

## Motivation

KRaft-to-Kraft upgrades and downgrades are one of the missing parts of Strimzi's KRaft support.
Supporting upgrades and downgrades is required for calling KRaft production ready.
This work can be done in parallel with the work on other proposals, such as migration of ZooKeeper-based clusters to KRaft based clusters.

## Proposal

Strimzi should follow the Apache Kafka [procedure for upgrading / downgrading](https://kafka.apache.org/documentation/#upgrade_350_kraft) Kafka.

### KRaft upgrades according to Apache Kafka

Apache Kafka [KRaft upgrade procedure](https://kafka.apache.org/documentation/#upgrade_350_kraft) consists of 2 different steps:
1. Roll out the new Kafka version and verify everything is working fine.
2. Update the `metadata.version` using the `kafka-reafutres.sh` command line tool (or using the Kafka Admin API).
   Unlike updating the `inter.broker.protocol.version`, updating the `metadata.version` does not require a rolling update of all Kafka nodes.

Downgrade is possible as well:
1. Downgrading the `metadata.version`
2. Rolling out the older Kafka version

Changes to the metadata formats - that are defined by the `metadata.version` - might not be backwards compatible.
As a result, the first step of the downgrade procedure - downgrading the `metadata.version` - can be done safely without any metadata loss only for selected metadata versions.
For other versions, the downgrade will be possible only as _unsafe_ downgrade that might result in loosing some metadata information and might have negative impact on the cluster.

#### Existing downgrade limitations

As of today, the situation is following:
1. The unsafe downgrade is currently not supported by Apache Kafka.
2. None of the `metadata.versions` supported by current Kafka versions support the safe downgrade.
   _(You should be able to downgrade from 3.5-IV1 to 3.5-IV0. But you cannot downgrade between 3.5 and 3.4 metadata versions.)_

The first point should be addressed in the future when the support for the unsafe downgrade is implemented.
The second point is expected to remain an issue.
While in the future the metadata formats might become more stable and it might be possible to safely downgrade the metadata between different Kafka versions, there still might be some versions that do not support the safe downgrade.

Despite Kafka currently not really supporting downgrades, this proposal designs the Strimzi implementation based on how the downgrade is expected to work.

### Strimzi implementation

Strimzi will follow the same approach to upgrading / downgrading KRaft cluster as it does today to for ZooKeeper-based clusters.

#### Configuring the metadata version

In ZooKeeper based clusters, the `inter.broker.protocol.version` is stored in the Kafka configuration (`Kafka.spec.kafka.config`).
The `metadata.version` is not part of the Kafka configuration and cannot be managed declaratively.
To allow users configure it in our Kafka custom resource, a new String-type field named `metadataVersion` will be added to `Kafka.spec.kafka`.
This field will be validated by the operator to contain a valid Kafka metadata version.

Users can use this new field to configure the metadata version.
The default value of this field when not set will be the metadata version corresponding to the current Kafka version.
The default version corresponding to given Kafka version will be stored in `kafka-versions.yaml` in the same way we today store the default `inter.broker.protocol.version` and `log.message.format.version`.

The operator will use this field to set the metadata version in the Kafka cluster.
In the initial deployment of the Kafka cluster, it will be set using the `kafka-storage.sh` tool.
In existing Kafka clusters, it will be queried and changed using the Kafka Admin API.

The current metadata version will be tracked in the `.status` section of the Kafka CR in `.status.kafkaMetadataVersion`.
This field will be used for validations and to track if upgrade / downgrade can be executed.

##### Upgrading metadata version

When upgrading the metadata version, the operator will validate the desired version and check if it is expected to work with the used Kafka version (e.g. is not higher than `metadata.version` supported by the Kafka version).
If it passes the validation, Kafka Admin API will be used to upgrade the metadata.
If the validation fails, the existing metadata version will be used and the error will be raised in log and in the `.status` section of the Kafka CR.

##### Downgrading metadata version

When downgrading the `metadata.version` in Kafka, the operator will always try to do a _safe_ downgrade only.
And if the safe downgrade fails, it will report an error (in log and in `.status` section of the Kafka CR) and continue reconciliation without changing the metadata version.

In case the safe downgrade is not supported, an _unsafe_ downgrade can be attempted (as explained in one of the earlier sections, the _unsafe_ downgrade is not implemented in current Kafka versions, but it is planned to be supported in the future).
Unsafe upgrade might result in metadata loss that might cause various problems to the Kafka cluster.
Users who decide they want to do the unsafe downgrade can do so manually:
* Pause the reconciliation of the Kafka CR
* Update the Kafka CR to the updated metadata version (to avoid the operator upgrading it back immediately after the reconciliation is unpaused)
* Use the Kafka Admin API or CLI tools to do the unsafe downgrade
* Unpause the reconciliation of the Kafka CR

##### Configuring the initial metadata version

When creating a new Kafka cluster, the initial metadata version can be configured using the `--release-version` option of the `kafka-storage.sh` script.
To allow users to deploy Kafka clusters with older metadata version, Strimzi will also store the desired `metadata.version` in the per-broker configuration Config Map. 
The value will be mounted into the Kafka pods and used when formatting the storage of a new Kafka cluster.
That way - unlike when using environment variable - we will not need to roll the Pod every time the `metadata.version` changes (since in such cases we can update it dynamically without rolling the pods).

#### Upgrade procedure

Upgrade can happen in two situations:
* After the upgrade of the Strimzi Cluster Operator:
    * When the new version of the Cluster Operator supports a new default Kafka version and `Kafka.spec.kafka.version` is not set.
    * When the user changes `Kafka.spec.kafka.version` in parallel with the Strimzi Cluster Operator upgrade.
* When user requests Kafka upgrade by changing `Kafka.spec.kafka.version`.

In both cases, the Strimzi Cluster Operator will:
1. Validate that the metadata version currently used is compatible with the new Kafka version (it will check that it is lower than the new Kafka version).
   This check is expected to pass for upgrades since a Kafka cluster that would not pass it would be invalid already before the upgrade.
2. Roll all Kafka pods to use the containers with the new Kafka version.

Once all the Kafka pods are rolled, the next step will depend on whether `Kafka.spec.kafka.metadataVersion` is set or not.
When it is not set, Strimzi will automatically update the metadata version in the Kafka cluster using Kafka Admin API to the default version corresponding to the new Kafka version.
The operator will not change the `Kafka.spec.kafka.metadataVersion` field - it will remain unset.
And the Kafka upgrade will be complete with this.

In case the `Kafka.spec.kafka.metadataVersion` field is set, it will just check the Kafka cluster has the desired metadata version as in any other reconciliation.
The user will be expected to verify the Kafka cluster and change the `Kafka.spec.kafka.metadataVersion` when everything seems to work fine.
Only after changing the metadata version, the upgrade will be considered complete.
Similarly to today's ZooKeeper-based implementation, a warning will be issued (in log and in Kafka CR `.status` section) when the metadata version does not correspond to the Kafka version and tell the user to complete the upgrade by updating the metadata version.

The upgrades will be supported to skip multiple Strimzi and Kafka versions in single step.
For example, to upgrade from Strimzi 1.1.0 / Kafka 4.1.0 to Strimzi 1.5.0 / Kafka 4.4.0 (the exact versions are for demonstration purposes only).
The exact number of versions a user can skip during the upgrade might be limited by other changes in Strimzi and by Apache Kafka itself.

#### Downgrade procedure

Downgrade can happen only in one situation: when user requests Kafka downgrade by changing `Kafka.spec.kafka.version`.

Unlike with upgrades, downgrading the Strimzi Cluster Operator is expected to be done only step-by-step and does not support skipping Strimzi versions:
1. First downgrade the Kafka version to the oldest version supported by the newer Strimzi version
2. Only then downgrade the Cluster Operator while keeping the same Kafka version
3. Repeat if you need to downgrade to an older Strimzi/Kafka versions

In both cases, the user is expected to ensure before triggering the downgrade that the metadata version is compatible with the Kafka version we should downgrade to.
If the `metadata.version` was already bumped before the user decided to downgrade, the user can downgrade it by changing `Kafka.spec.kafka.metadataVersion`.
Setting the exact version is required even if the metadata version was not set in the `Kafka` CR and was bumped automatically.
(See the section about configuring metadata versions for the limitations of downgrading metadata.)
Unlike the upgrades that might happen automatically, downgrades are expected to be always driven by the users.
So this level of upfront preparation is acceptable.

In both cases, the operator will:
1. Validate if the metadata version is suitable for the Kafka version we are downgrading to (i.e. the same or lower).
   If the validation fails, it will fail the reconciliation and expect the user to address it.
   Since the user triggers the downgrade by manual action (editing the Kafka CR), it is expected that this error will be caught on early and the user will fix the problem or revert the downgrade.
2. Next, the operator will roll the Kafka pods to use the new container with the desired version.

_Note:_
_This corresponds to what Strimzi supports today for ZooKeeper-based clusters._
_Today, the users are responsible for downgrading the `inter.broker.protocol.version` before the Kafka downgrade by updating their `Kafka` CR._
_Similarly, even today they have to downgrade Strimzi versions step by step in the same way as described in this section._

#### (Stretch goal) Downgrade of Kafka after Strimzi Cluster Operator downgrade

As a _stretch_ goal, we should also support Kafka downgrades right after the Strimzi downgrade (i.e. jumping multiple Strimzi and Kafka versions during downgrade).
Implementing this has additional technical complications because it needs to understand Kafka versions that did not exist at the time the software was released.
While this feature might be useful in some situations, it does not need to be implemented to consider our KRaft support _production-ready_.
So it might be implemented only later after the other parts.

Even in this downgrade scenario, the user has to make sure the metadata version used by Strimzi is valid even for the Kafka version that we should downgrade to.
After the Cluster Operator is downgraded, the operator will validate the Kafka version and the metadata version.
And if the validation passes, it will automatically roll all Kafka pods to use the Kafka version we are downgrading to.
If the validation fails, it will be up to the user to address the issue.
This might require moving back to the original Strimzi and Kafka version.

#### Risks

Kafka provides APIs to manage the `metadata.version`.
Strimzi has no way to block the users from using these APIs.
If the users decide to manipulate the `matadata.version` them self, this can lead to unexpected issues.
This risk is deemed acceptable, because we expect that:
* Users would prefer to have the `metadata.version` managed through Strimzi.
* Users would have no reason to manipulate the `metadata.version` on their own.

The risk can be also at least partially mitigated through documentation.

## Affected projects

This proposal affects only the Strimzi Cluster Operator.
No other components are affected.

## Backwards compatibility

This proposal has no impact on any existing features or on the upgrades / downgrades of ZooKeeper-based clusters.

## Rejected alternatives

### Leaving the `metadata.version` updates to users

One of the alternatives considered was to have Strimzi only update the container images (software).
And leave the update of the `metadata.version` to the user.
However, this would not allow users to execute the whole upgrade declaratively.
There would be also risk that users won't follow with the `metadata.version` and keep using the old version and that might have negative consequences in the future when the Kafka cluster runs on too old `metadata.version`.
