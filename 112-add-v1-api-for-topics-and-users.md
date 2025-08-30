# Add the `v1` version to `KafkaTopic` and `KafkaUser` CRD APIs

As part of move to the Strimzi `v1` CRD API and to Strimzi 1.0.0, this proposal introduces the `v1` versions for the `KafkaTopic` and `KafkaUser` resources.

## Motivation

One of the lessons we learned from the introduction of the Strimzi `v1beta2` CRD API and from the migration from the `v1alpha1` / `v1beta1` APIs, is that the `KafkaTopic` and `KafkaUser` resources require special care.
The reason for it is that they are operated by detached User and Topic Operators.
When the Strimzi upgrade happens, in the first phase:
* The CRDs are updated
* The Cluster Operator Pod is rolled

While these two events are not synchronized and are _eventually consistent_, they happen at roughly the same time.
The User and Topic Operators on the other hand, they are rolled only much later as part of the Kafka cluster reconciliation.
This can be many minutes after the CRDs were updated.

Additionally, the Topic and User Operator are deleting their Users / Topics from the Kafka cluster when the resources are deleted.
This is different from the Cluster Operator, which in general does not delete the operands as they are deleted by Kubernetes Garbage Collection.
(While this does not apply to the `KafkaConnector` resources which are deleted by the operator through the Connect REST API, the `KafkaConnector` resources are reconciled only when the corresponding `KafkaConnect` resource exists.)
As a result, if the Cluster Operator does not see the custom resources for some time, it will not delete anything.
In contrast, the User and Topic Operators would delete any topic or user configuration when they don't see the corresponding CRs (in case of the User Operator) or when the perceive that they have been deleted (in case of the Topic Operator).

This had significant impact on the introduction of `v1beta2` and migration from `v1beta1` to `v1beta2` APIs.
We used the following schedule:
* Strimzi 0.22 introduced the `v1beta2` API, but the operators kept using the old `v1beta1` API
* Strimzi 0.23 removed the `v1alpha1` and `v1beta1` APIs and operators started using the `v1beta2` API

_Note:_
_The schedule for the migration to `v1beta2` API was heavily influenced by Kubernetes dropping support for the `v1beta1` CRD API._
_That forced us to proceed very quickly._

This schedule worked for the Cluster Operator.
But it did not worked for the Topic and User Operators.
When the old API versions were removed from the Kubernetes cluster, we had still the Topic and User Operators from the previous Strimzi version running.
And they were still using the old APIs.
As a result, they perceived all their `KafkaUser` and `KafkaTopic` resources to be deleted and deleted them from the Kafka cluster.

We discovered this issue early enough and decided to work around it by not removing the `v1alpha1` and `v1beta1` APIs from the Strimzi `KafkaTopic` and `KafkaUser` CRDs.
And only in Strimzi 0.24, we moved the User and Topic Operators to use the new `v1beta2` API.
And till today, these API versions are present there only for these two resources.

We need to learn from this for the `v1` migration.
And in general, we should do the migration in three separate steps:
* In Strimzi version X we should add the new `v1` API but keep the operators using the `v1beta2` APIs
* In Strimzi version Y we should move the operators to use the `v1` APIs
* And finally in version Z we should remove the old `v1beta2` (and for the `KafkaTopic` and `KafkaUser` also the `v1alpha1` and `v1beta1`) APIs.

We should also ideally try to leave more time between the versions and not do all there steps in three subsequent Strimzi releases.
That should give users more freedom and avoid forcing them to upgrade through 3 subsequent Strimzi versions.

However, given we are now aware of the increased sensitivity of the User and Topic Operators, we should try to introduce the `v1` version for them as early as possible.
That should make it easier for our users to migrate to the `v1` API while keeping more relaxed upgrade schedules (i.e. skipping some Strimzi versions).
This should be also reasonably straight forward for the `KafkaUser` and `KafkaTopic` resources as there are minimal changes expected to them.

## Proposal

We will introduce the `v1` versions of the `KafkaUser` and `KafkaTopic` CRDs in Strimzi 0.49.0.
They will be added to all the installation files - Cluster Operator, standalone User and Topic Operators, Helm Chart, etc.
This proposal does not change the User and Topic Operators to use the new `v1` APIs when communicating with the Kubernetes cluster.
This change will be done only later together with the other custom resources.

### `KafkaTopic` `v1` API

The following changes will be done to the `KafkaTopic` API in the `v1` version:
* In the current versions of the `KafkaTopic` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

_Note: As there are no deprecated fields in the current versions of the `KafkaTopic` API, no fields are being removed in the `v1` version._

Apart from the changes above, the `v1` version of `KafkaTopic` will look exactly the same as the current `v1beta2` version.

### `KafkaUser` `v1` API

The following changes will be done to the `KafkaUser` API in the `v1` version:
* In the current versions of the `KafkaUser` resource, the `operation` field (`.spec.authorization.acls[].operation`) in the ACL rule section is deprecated (replaced with the `operations` list).
  This field will not be present in the `v1` API.
  The `v1` version will also make the `operations` field required.
* In the current versions of the `KafkaUser` resource, the `.spec` section is not marked as required.
  The `v1` API will mark it as required.

Apart from the changes above, the `v1` version of `KafkaUser` will look exactly the same as the current `v1beta2` version.

### Examples

For the time being, the examples will remain unchanged and will use `v1beta2` API (while of course avoiding using any deprecated fields).
The examples will be updated to the `v1` version later with the other custom resource kinds.

### Testing strategy

Unit tests will be used to see that the `v1` APIs can be used and work.
There is no impact expected on System Tests or any other tests.

## Affected projects

This proposal affects the Strimzi Operators only.
The changes are expected to be in the `api` module and in the resulting installation files.

## Backwards compatibility

As this proposal only adds the `v1` API version to the `KafkaUser` and `KafkaTopic` CRDs but does not really use it yet in the User and Topic operators, there is no impact on backwards compatibility.

## Out of scope

Introducing the `v1` versions to any Strimzi CRDs other than `KafkaUser` and `KafkaTopic` are out of scope of this proposal.
The other CRDs will use separate proposals.

This proposal also does not cover the future resource conversion which will be needed when removing the old API versions.
The conversion should be handled in a separate proposal for all our custom resource kinds.

## Rejected alternatives

There are currently no rejected alternatives.
