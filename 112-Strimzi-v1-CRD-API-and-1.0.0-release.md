# Strimzi `v1` CRD API and 1.0.0 release

This proposal covers the introduction of a new Strimzi API version `v1`, migration to it from the old versions, and the release of Strimzi 1.0.0.

## Motivation

Strimzi 1.0.0 release is long overdue.
This is mainly caused by the ZooKeeper removal from Apache Kafka taking much longer than originally anticipated.
But there are other factors as well.
Some more information about the history can be found in this [blog post from 2023](https://strimzi.io/blog/2023/12/11/where-is-strimzi-1.0.0/).

One of the reasons why we decided to wait for ZooKeeper to be removed from Apache Kafka was that it would allow us to remove some parts of our CRD API before doing the `v1` API version which will be an integral part of Strimzi 1.0.0.
As ZooKeeper has now been removed from Strimzi, we can now proceed with introducing the `v1` API version and release Strimzi 1.0.0.

## Proposal

### Timeline

The migration from `v1beta2` will happen as a two-step process:
* The first step will be done in Strimzi 0.49.
    * It will introduce the `v1` API into the Strimzi CRDs
    * It will deprecate the old `v1beta2` API version (and for `KafkaTopic` and `KafkaUser` CRDs also the `v1alpha1` and `v1beta1`).
      The deprecation warning messages will point to the section of our documentation about the `v1` API and its migration.
      This message will be showed by tools such as `kubectl` when using the old APIs.
    * And it will transition the Topic and User Operators to use the `v1` API.
    * Users will need to make sure their `KafkaUser` resources do not use the deprecated/removed `operation` field.
* The second step will be done 3 releases later in Strimzi 0.52/1.0.0
    * This version will remove the `v1beta2` API (and for `KafkaTopic` and `KafkaUser` CRDs also the `v1alpha1` and `v1beta1`).
    * It will also transition the Cluster Operator to use the `v1` API as well.
    * Before upgrading to this version, users will need to convert all their custom resources and the CRDs.
      The conversion could be done using the conversion tool (described in more detail in one of the later sections) or manually.

This might correspond to the following release dates:
* 0.49.0 in early November 2025
* 0.50.0 early January 2026
* 0.51.0 late February 2026
* 0.52.0 / 1.0.0 in April 2026

*The dates are subject to change.*

More details and explanation to the different steps is provided in the sections below.

### Upgrade path

Users will have the time between the 0.49 and 0.52/1.0.0 releases to migrate their custom resources to the `v1` API.
The only slight exception will be the `KafkaUser` resources.
While users will be able to keep using the older versions of the `KafkaUser` API between 0.49.0 and 0.52.0/1.0.0, they will not be able to use the deprecated field `operation` in the ACL rule definition.
Any resources using this field will be rejected by the User Operator as invalid (however, the User itself will not be deleted from the Apache Kafka cluster).
This is because the User Operator will be already using the new `v1` API internally.
While technically the same applies to the Topic Operator and the `KafkaTopic` resources, the `KafkaTopic` resource has no deprecated fields that are being removed in the `v1` API version.
So there is no similar impact on the `KafkaTopic` resources as on the `KafkaUser` resources.

The upgrade process does not require including Strimzi 0.49.0 or 0.52.0/1.0.0 specifically.
However, upgrade paths must include:
* **At least one of Strimzi 0.49.0, 0.50.0, or 0.51.0**, which introduces the `v1` API and transitions the User and Topic Operators to use it.
* **Strimzi 0.52.0/1.0.0 or newer**, which removes the old API versions and transitions the Cluster Operator to the `v1` API.

For example, Strimzi users can:
* Start with Strimzi 0.47.0
* Upgrade to Strimzi 0.50.0
    * Before this upgrade, they should make sure they do not use the `operation` field in the `KafkaUser` resource
* Upgrade to Strimzi 1.1.0
    * CRD conversion of all custom resources to the `v1` API will need to be done for all custom resources before this upgrade

### First step

In the first phase, the Strimzi CRDs will be modified to add the new `v1` API to the existing CRDs.
The details of how the `v1` API will look for the different resources and how it will differ from the existing API versions are described in a separate section below.
In the first phase, we will also deprecate the old CRD API versions.
The CRD API version deprecation is a flag in the CRD resource.
Deprecated versions can still be used, but Kubernetes will issue a warning when the deprecated API is used.

Most of the CRDs will continue to use the `v1beta2` version as the stored version in the first phase.
Only the `KafkaUser` and `KafkaTopic` CRDs will already switch to `v1` as the stored version.
With the exception of the `KafkaUser` resource, users are not expected to do anything in the first phase other than upgrade the CRDs and the other Strimzi resources.

The `Kafka` CRD YAML will be generated with field descriptions disabled because otherwise it would be too big for `kubectl apply`.

#### The User and Topic Operator problem

One of the lessons we learned from the introduction of the Strimzi `v1beta2` CRD API and from the migration from the `v1alpha1` / `v1beta1` APIs, is that the `KafkaTopic` and `KafkaUser` resources require special care.
The reason for it is that they are operated by detached User and Topic Operators.
When the Strimzi upgrade happens, in the first phase:
* The CRDs are updated
* The Cluster Operator Pod is rolled

While these two events are not synchronized and are _eventually consistent_, they happen at roughly the same time.
The User and Topic Operators, on the other hand, are only rolled much later as part of the Kafka cluster reconciliation.
This can be many minutes after the CRDs were updated.

Additionally, the Topic and User Operator are deleting their Users / Topics from the Kafka cluster when the resources are deleted.
This is different from the Cluster Operator, which in general does not delete the operands as they are deleted by Kubernetes Garbage Collection.
(While this does not apply to the `KafkaConnector` resources which are deleted by the operator through the Connect REST API, the `KafkaConnector` resources are reconciled only when the corresponding `KafkaConnect` resource exists.)
As a result, if the Cluster Operator does not see the custom resources for some time, it will not delete anything.
In contrast, the User and Topic Operators would delete any topic or user configuration when they don't see the corresponding CRs (in case of the User Operator) or when they perceive that they have been deleted (in case of the Topic Operator).

This had significant impact on the introduction of `v1beta2` and migration from `v1beta1` to `v1beta2` APIs.
We used the following schedule:
* Strimzi 0.22 introduced the `v1beta2` API, but the operators kept using the old `v1beta1` API
* Strimzi 0.23 removed the `v1alpha1` and `v1beta1` APIs and operators started using the `v1beta2` API

_Note:_
_The schedule for the migration to `v1beta2` API was heavily influenced by Kubernetes dropping support for the `v1beta1` CRD API._
_That forced us to proceed very quickly._

This schedule worked for the Cluster Operator.
But it did not work for the Topic and User Operators.
When the old API versions were removed from the Kubernetes cluster, we had still the Topic and User Operators from the previous Strimzi version running.
And they were still using the old APIs.
As a result, they perceived all their `KafkaUser` and `KafkaTopic` resources to be deleted and deleted them from the Kafka cluster.

We discovered this issue early enough and decided to work around it by not removing the `v1alpha1` and `v1beta1` APIs from the Strimzi `KafkaTopic` and `KafkaUser` CRDs.
And only in Strimzi 0.24, we moved the User and Topic Operators to use the new `v1beta2` API.
And till today, these API versions are present there only for these two resources.

We need to learn from this for the `v1` migration.
Therefore, this proposal moves the User and Topic Operators to `v1` in the first step, so they’re already using `v1` when older API versions are removed.

However, moving the User and Topic Operators to the `v1` API early means that their custom resources need to be updated already before the upgrade to the first phase.
If the User and Topic Operators are transitioned to the `v1` API early, their custom resources must be updated before the first upgrade phase. 
Fields that were deprecated and removed from the `v1` API will no longer be visible to these operators once the transition occurs.

For example, when the User Operator using the `v1` API reconciles a `KafkaUser` resource still using the deprecated field `.spec.authorization.acls[].operation`, it will not see this field in the custom resource.
It will be simply missing.
So for the User Operator to work well, users have to make sure that their `KafkaUser` resources do not use the deprecated `.spec.authorization.acls[].operation` and use instead the `.spec.authorization.acls[].operations` array.
If any resource is not updated, the User Operator will report an error as it will be missing the `operation` field (due to the `v1` API schema) as well as the `operations` array (due to not being used by the user).
However, the User Operator will not delete the user.
Just raise the error and report the user as _not ready_.

Luckily, the `operation` is the only field that is deprecated and removed in the `KafkaUser` resource.
And no fields are deprecated in the `KafkaTopic` resource.
So there is nothing else to update.

The `operation` field was deprecated in 2022.
That is pretty long time ago.
So hopefully, it is not used anymore by most of our users.
It is also why I believe that this approach is acceptable and that it minimizes the overall effort on our users.

Alternative approaches might include moving the User and Topic operators to the `v1` version at later stage.
However, this would still need the users to update their `KafkaUser` resources.
And it would likely extend the time needed for the migration to `v1` which is not desired.

### Second step

The second step is expected to be done in Strimzi 0.52.0 / 1.0.0.
This version will update the stored version of the remaining custom resources to `v1`.
And it will remove the old deprecated API versions.
The field descriptions will be re-enabled for the `v1` API as with the `v1` version only it would be now small enough again to include them.
The Cluster Operator in this version will also use the `v1` API.

Before upgrading to the Strimzi 0.52.0 / 1.0.0, users will need to convert all their custom resources to the `v1` version.
The conversion can be done manually or using a provided conversion tool (more on the conversion tool in a separate chapter below).

The conversion tool will also do the initial update of the stored API version to `v1` and will remove the older API versions from the CRD status after the conversion to allow their removal.

After applying the conversion tool, users would upgrade to Strimzi 0.52.0 / 1.0.0.
During the upgrade — when applying the new CRDs — the old API version will be removed.

### Strimzi 1.0.0 version

The `v1` API and removal of the older APIs is the main obstacle in releasing Strimzi 1.0.0.
And given how long we have been postponing the 1.0.0 release, I believe we should get to it as soon as possible.
This is why this proposal assumes that the Strimzi release where we remove the older APIs will be the 1.0.0 release.

However, we can decide to change that and postpone the 1.0.0 release at any time before 1.0.0 is actually released.
Instead, we can continue with 0.5x releases (0.52, 0.53 etc.) until we feel ready for 1.0.0 release.

### Changes to individual custom resources

#### `KafkaTopic`

The following changes will be done to the `KafkaTopic` API in the `v1` version:
* In the current versions of the `KafkaTopic` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

_Note: As there are no deprecated fields in the current versions of the `KafkaTopic` API, no fields are being removed in the `v1` version._

##### Rejected changes

The following changes were considered and rejected:
* Making the `.spec.replicas` field required was considered.
  But we decided to reject it as the default value is configurable in Kafka and can be used to set different default value in various environments (e.g. one replication factor 1 in development and 3 in production).
* Renaming the `.spec.replicas` field to `.spec.replicationFactor` was considered.
  It was rejected because it would have major impact on the users and make the early transition of the Topic and User Operators to the `v1` API much harder.

#### `KafkaUser`

The following changes will be done to the `KafkaUser` API in the `v1` version:
* In the current versions of the `KafkaUser` resource, the `operation` field (`.spec.authorization.acls[].operation`) in the ACL rule section is deprecated (replaced with the `operations` list).
  This field will not be present in the `v1` API.
  The `v1` version will also make the `operations` field required.
* In the current versions of the `KafkaUser` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

#### `KafkaConnector`

The following changes will be done to the `KafkaConnector` API in the `v1` version:
* In the current versions of the `KafkaConnector` resource, the `.spec.pause` field is deprecated (replaced with the `state` field).
  This field will not be present in the `v1` API.
* In the current versions of the `KafkaConnector` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

##### Rejected changes

The following changes were considered and rejected:
* Making the `.spec.tasksMax` field required was considered.
  It was rejected as we wanted to keep the connector configuration look similar to what users might be used from pure Apache Kafka.

#### `KafkaConnect`

The following changes will be done to the `KafkaConnect` API in the `v1` version:
* The `deployment` section in `.spec.template` is deprecated and will be removed in `v1` without replacement.
* The `externalConfiguration` section in `.spec` is deprecated and will be removed in `v1`.
  It was replaced with various other fields.
* The `type: jaeger` tracing in `.spec.tracing` is deprecated and will be removed without replacement (`type: opentelemetry` tracing support remains unchanged).
* `.spec.replicas` field will be required in the `v1` API.
  During the CRD conversion, it will be set to `3` when not set (the original default value).
* The `type: oauth` authentication will be removed in the `v1` version.
  Users can use the `type: custom` authentication instead.
  Deprecation of the `type: oauth` authentication is subject of a separate [proposal](https://github.com/strimzi/proposals/pull/175) that needs to be approved and implemented in the same release as the `v1` is delivered.
* In the current versions of the `KafkaConnect` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

#### `KafkaMirrorMaker2`

The following changes will be done to the `KafkaMirrorMaker2` API in the `v1` version:
* In the current versions of the `KafkaMirrorMaker2` resource, the `pause` field in the connector configurations is deprecated (replaced with the `state` field).
  This field will not be present in the `v1` API.
* The `topicsBlacklistPattern` and `groupsBlacklistPattern` fields in the `mirrors` section are deprecated (replaced with `topicsExcludePattern` and `groupsExcludePattern` fields).
  These fields will not be present in the `v1` API.
* The `deployment` section in `.spec.template` is deprecated and will be removed in `v1` without replacement.
* The `type: jaeger` tracing in `.spec.tracing` is deprecated and will be removed without replacement (`type: opentelemetry` tracing support remains unchanged).
* `.spec.replicas` field will be required in the `v1` API.
  During the CRD conversion, it will be set to `3` when not set (the original default value).
* The `type: oauth` authentication will be removed in the `v1` version.
  Users can use the `type: custom` authentication instead.
  Deprecation of the `type: oauth` authentication is subject of a separate [proposal](https://github.com/strimzi/proposals/pull/175) that needs to be approved and implemented in the same release as the `v1` is delivered.
* In the current versions of the `KafkaMirrorMaker2` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

##### Rejected changes

The following changes were considered and rejected:
* Making the `.spec.tasksMax` field required was considered.
  It was rejected as we wanted to keep the connector configuration look similar to what users might be used from pure Apache Kafka.

#### `KafkaBridge`

The following changes will be done to the `KafkaBridge` API in the `v1` version:
* In the current versions of the `KafkaBridge` resource, the `enableMetrics` field in the connector configurations is deprecated (replaced with the `metricsConfig` field).
  This field will not be present in the `v1` API.
* The `type: jaeger` tracing in `.spec.tracing` is deprecated and will be removed without replacement (`type: opentelemetry` tracing support remains unchanged).
* `.spec.replicas` field will be required in the `v1` API.
  During the CRD conversion, it will be set to `1` when not set (the original default value).
* The `type: oauth` authentication will be removed in the `v1` version.
  Users can use the `type: custom` authentication instead.
  Deprecation of the `type: oauth` authentication is subject of a separate [proposal](https://github.com/strimzi/proposals/pull/175) that needs to be approved and implemented in the same release as the `v1` is delivered.
* In the current versions of the `KafkaBridge` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

#### `KafkaRebalance`

The following changes will be done to the `KafkaRebalance` API in the `v1` version:
* In the current versions of the `KafkaRebalance` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

#### `StrimziPodSet`

The following changes will be done to the `StrimziPodSet` API in the `v1` version:
* In the current versions of the `StrimziPodSet` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

#### `KafkaNodePool`

The following changes will be done to the `KafkaNodePool` API in the `v1` version:
* The `overrides` section in `type: persistent-claim` storage is deprecated and will be removed without replacement in `v1`.
* In the current versions of the `KafkaNodePool` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

#### `Kafka`

The following changes will be done to the `Kafka` API in the `v1` version:

* In the `.status` section:
    * The `.status.registeredNodeIds` field is deprecated and will be removed without replacement.
    * The `.status.kafkaMetadataState` will be deprecated and removed in the `v1` version without replacement.
    * The `type` field in `status.listeners` will be removed in the `v1` version.
      It is already replaced by the `name` field in the same section.

* In the `.spec` section:
    * The `.spec.zookeeper` section will be removed in the `v1` version without replacement.
    * The `.spec.jmxTrans` section will be removed in the `v1` version without replacement.
    * In the current versions of the `Kafka` resource, the `.spec` section is not marked as required.
      The `v1` API will mark it as required.

* In the `.spec.cruiseControl` section:
    * The `tlsSidecar` section from `.spec.cruiseControl` will be removed in the `v1` version without replacement.
    * The `tlsSidecarContainer` section from `.spec.cruiseControl.template` will be removed in the `v1` version without replacement.
    * The `cpuUtilization` and `disk` fields in `.spec.cruiseControl.brokerCapacity` are deprecated and will be removed without replacement in `v1` version.

* In the `.spec.kafkaExporter` section:
    * The `service` section in `.spec.kafkaExporter.template` is deprecated and will be removed in `v1` without replacement.

* In the `.spec.entityOperator` section:
    * The `tlsSidecar` section from `.spec.entityOperator` will be removed in the `v1` version without replacement.
    * The `tlsSidecarContainer` section from `.spec.entityOperator.template` will be removed in the `v1` version without replacement.
    * The `reconciliationIntervalSeconds` field in `.spec.entityOperator.topicOperator` is deprecated and will be removed in `v1` version.
      It is replaced by the `reconciliationIntervalMs` field.
    * The `zookeeperSessionTimeoutSeconds` field in `.spec.entityoperator.topicOperator` is deprecated and will be removed in `v1` version without replacement.
    * The `topicMetadataMaxAttempts` field in `.spec.entityoperator.topicOperator` is deprecated and will be removed in `v1` version without replacement.
    * The `reconciliationIntervalSeconds` field in `.spec.entityOperator.userOperator` is deprecated and will be removed in `v1` version.
      It is replaced by the `reconciliationIntervalMs` field.
    * The `zookeeperSessionTimeoutSeconds` field in `.spec.entityoperator.userOperator` is deprecated and will be removed in `v1` version without replacement.

* In the `.spec.kafka` section:
    * The `enableECDSA` field in the OAuth authentication is deprecated and will be removed in `v1` without replacement.
    * `secrets` section in `type: custom` authentication is deprecated and will be removed in `v1`.
      It is replaced by mounting secrets through the additional volumes feature in the `template` section.
    * The `statefulset` option of the `.spec.kafka.template` section is deprecated and will be removed in `v1` without replacement.
    * The `type: opa` authorization in `.spec.kafka.listeners` has been deprecated and will be removed in `v1`.
      It is replaced by `type: custom` authorization.
    * The `type: oauth` authentication will be removed in the `v1` version.
      Users can use the `type: custom` authentication instead.
      Deprecation of the `type: oauth` authentication is subject of a separate [proposal](https://github.com/strimzi/proposals/pull/175) that needs to be approved and implemented in the same release as the `v1` is delivered.
    * The `type: keycloak` authorization will be removed in the `v1` version.
      Users can use the `type: custom` authorization instead.
      Deprecation of the `type: keycloak` authorization is subject of a separate [proposal](https://github.com/strimzi/proposals/pull/175) that needs to be approved and implemented in the same release as the `v1` is delivered.

##### Rejected changes

The following changes were considered and rejected:
* Moving the remaining options from `.spec.kafka` directly to `.spec` was considered but rejected.
  The main reason was that it would be a major change for our users with very little added value and that the `.spec.kafka` section still corresponds to how Strimzi deploys the Apache Kafka cluster.
  The full discussion can be found in the [recording from the Strimzi `v1` CRD API triage call](https://youtu.be/mQZ7dLYCN1I).

### Conversion tool

A conversion tool will be provided to the users as part of Strimzi 0.49 when the `v1` API is introduced.
The conversion tool will allow:
* Automatic conversion of the custom resources
* Updating each CRD’s stored API version to `v1`

The automatic conversion of the resources will:
* Remove the fields that were deprecated without replacement if they are still used
* Update the fields that were deprecated and have replacements
* Update the custom resource structures where needed

There will be some exceptions where we cannot automatically convert the resources and users will be required to do this manually:
* The `type: opa` and `type: keycloak` authorization in the `Kafka` CR
* The `type: oauth` authentication in the `Kafka`, `KafkaConnect`, `KafkaMirrorMaker2`, and `KafkaBridge` resources
* The `enableMetrics` in the `KafkaBridge` resource

Conversion of these fields would be too complicated as it often involves other resources (Config Maps and/or Secrets).
So the conversion tool will fail and raise an error, prompting the user to fix these fields manually.

The conversion tool will be covered in detail in a separate proposal.

### Downgrades

The `v1` API currently doesn't introduce any new fields or structures.
So it is backwards compatible with the `v1beta2` APIs.
So users who upgrade to Strimzi 0.52 / 1.0 or later and use only the `v1` API will be able to downgrade to Strimzi 0.49 - 0.51.
They would just install the older versions and their existing `v1` resources would be usable as valid `v1beta2` resources.

Downgrading from Strimzi 0.49 – 0.51 to Strimzi 0.48 or older is more complicated.
The older versions do not support the `v1` API.
When downgrading, users would need to:
* (At least initially) keep the newer CRD files including the `v1` version in order to avoid problems with the User and Topic Operators that will be still using the `v1` version for some time
* Only after the User and Topic Operators are rolled, users can consider removing the `v1` API version.
  However, this would have to be done manually as it would not be supported by the conversion tool.

### Testing strategy

New system tests will be written to cover the CRD upgrades.
In addition to that, a new pipeline based on the existing tests will be created that will use the `v1` API from the system tests to manage the custom resources.
This pipeline will use the existing tests (any existing system tests testing deprecated fields will be disabled in this pipeline).

### Examples

The examples will be updated in the first phase to use the new `v1` API.
As our examples in general do not use any deprecated fields, the update will mostly mean only the change of the API version.

### Documentation

The Overview, Get Started, and Deploying and Managing guides will be updated to use the `v1` CRD version already in the first phase with Strimzi 0.49.0.
The update will especially include all commands and all YAML examples.
The example YAMLs will be updated in the API Reference as well.
In addition to that, the generated reference will be also updated to include the information about the fields that will be present only in some versions.

We will also add new sections to the documentation covering the automated and manual conversion and update the related upgrade and downgrade guides.

## Remaining open points

The following points are still open and might result in future changes in the `v1` schema:
* [Consider places missing additional validation](https://github.com/strimzi/strimzi-kafka-operator/issues/11765)
* [Enforcing internal topic names and `group.id` in Kafka Connect](https://github.com/strimzi/strimzi-kafka-operator/issues/10075)
* [Redesign of the `KafkaMirrorMaker2` CR/CRD](https://github.com/strimzi/strimzi-kafka-operator/issues/11842)

These open points will be clarified in separate issues or proposals.
They should not be blockers for this proposal.
However, they need to be done before we release the 0.49.0 version with the `v1` API, as we cannot remove anything from the `v1` API once it is released.

## Rejected alternatives

### Switching the User and Topic Operators at a different schedule

I considered the possibility of switching the Topic and User Operators to use the `v1` API as an additional step in the middle.
That would mean that the whole issue with the `KafkaUser` resources and the `operation` field would not happen right away but only later.

Such an updated schedule would be for example:
* Strimzi 0.49 introduces the `v1` API in the CRDs but will not use it in any of our operators.
* Strimzi 0.51 switches the User and Topic Operators to use the `v1` API.
  At this point, users would need to take care that they do not use the `operation` field in the `KafkaUser` resource.
* Strimzi 0.53/1.0.0 would switch the remaining operators to the `v1` API and remove the older APIs from the CRDs.

I rejected this alternative because:
* The transition would take longer
* The transition would include more steps the users would need to go through:
    * This means they need to pay more attention during multiple upgrades
    * It would be harder to skip some of the versions

### Conversion webhook

A conversion webhook can be used to help with the conversion of the custom resources.
This could for example help with the initial update of the `KafkaUser` resource (the deprecated `operation` field).
However, the installation and use of the conversion webhook is complicated as it has to run once per Kubernetes cluster and is configured directly in the CRD.
Some users might also not have the access rights needed to install the cluster-wide webhook.

Therefore it seems that relying on the webhook would cause more work and effort compared to what value it actually delivers.
And we might need to deliver the conversion tool together with it anyway.

### Rejected API changes

There were many different changes to the Strimzi `v1` CRD API that were discussed and eventually rejected.
They are documented in the `Changes to individual custom resources` section under the different custom resources.
