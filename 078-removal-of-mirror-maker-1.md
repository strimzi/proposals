# Removal of Mirror Maker 1

Mirror Maker 1 is planned to be removed in Apache Kafka 4.0.
This proposal discusses its removal from Strimzi.

## Current situation

Apache Kafka currently supports two versions of Mirror Maker.
Mirror Maker 1, which is the original Mirror Maker version.
And Mirror Maker 2 that is a newer replacement for Mirror Maker 1 with new architecture and features.
Strimzi currently supports both Mirror Maker versions

Mirror Maker 1 is currently deprecated in Apache Kafka as well as in Strimzi.
In Apache Kafka, it is planned to be removed in Apache Kafka 4.0 release.

## Motivation

Strimzi cannot support Mirror Maker 1 once it is removed from Apache Kafka.
But Strimzi releases typically support two minor Kafka versions.
And therefore the point when the Mirror Maker 1 support will be removed is not obvious.
Will it be removed when right away Kafka 4.0 is adopted?
Or only once support for Kafka 3.x is completely dropped?

This proposal aims to follow up on the [proposal #77](https://github.com/strimzi/proposals/blob/main/077-support-for-kafka-4.0.md) which discussed the adoption of Kafka 4.0 and support for KRaft and ZooKeeper based clusters.
It formalizes the removal timeline for Mirror Maker 1 and clarifies what will be done by Strimzi and what will be done by the users.
It does not aim to provide any step-by-step migration guide.

## Expectations

This proposal uses the same expectations as the [proposal #77](https://github.com/strimzi/proposals/blob/main/077-support-for-kafka-4.0.md) with regards to Kafka releases:
* Apache Kafka 3.8.0 release (currently in the release process, expected release in July or August?)
* Apache Kafka 3.9.0 release with the remaining missing KRaft features (shorter release cycle - expected August or September?)
* Apache Kafka 4.0.0 release 3-4 months after the Kafka 3.9 release without Mirror Maker 1 (late 2024, early 2025?)

## Proposal

Strimzi will remove support for Mirror Maker 1 together with ZooKeeper support.
This would mean that the last version supporting Mirror Maker 1 will be 0.4y.
This is also the last version with ZooKeeper support and the version selected for the extended support in [proposal #77](https://github.com/strimzi/proposals/blob/main/077-support-for-kafka-4.0.md).
Support for Mirror Maker 1 will be dropped in Strimzi 0.4z.

The following table shows how the Mirror Maker 1 (MM1) support fits with Kafka 4.0, ZooKeeper, and Kraft support.

| Version | Supported Kafka versions | Supported metadata modes | Extended support | MM1 support |
| :-----: |:-------------------------|:-------------------------|:-----------------|:------------|
| ...     | 3.7.x, 3.8.x             | ZooKeeper, KRaft         | no               | yes         |
| 0.4x    | 3.8.x, 3.9.x             | ZooKeeper, KRaft         | no               | yes         |
| 0.4y    | 3.8.x, 3.9.x             | ZooKeeper, KRaft         | **yes**          | yes         |
| 0.4z    | 3.9.x, 4.0.x             | KRaft                    | no               | no          |
| ...     | 4.0.x, 4.1.x             | KRaft                    | no               | no          |

Users will be responsible for migrating their Mirror Maker 1 clusters to Mirror Maker 2 (or any other alternative they might choose) before upgrading to Strimzi 0.4z.
While it is possible to configure Mirror Maker 2 to closely resemble the functionality of Mirror Maker 1, this proposal does not cover the detailed steps of how users should migrate from Mirror Maker 1 to Mirror Maker 2.

If the Mirror Maker 1 cluster is not migrated before upgrade to Strimzi 0.4z or later, Strimzi will stop operating it.
But it will keep it running and not delete it.
When the user is done with the migration, they can simply delete the `KafkaMirrorMaker` custom resource and that will also delete the old Mirror Maker 1 cluster and all its resources through Kubernetes garbage collection.

Users will be also responsible for deleting the `KafkaMirrorMaker` CRD once they upgrade to Strimzi 0.4z or later (and once they migrate all of their remaining clusters, as the CRD deletion would also delete the custom resources and the related Deployments/Pods).

### Strimzi changes

The following changes will be done to Strimzi to remove support for Mirror Maker 1:
* Remove the `KafkaMirrorMaker` CRD from Strimzi installation files, Helm Chart and OperatorHub entries
* Remove the environment variable for configuring the Mirror Maker 1 images from the Cluster Operator Deployment file
* Remove Mirror Maker 1 from our examples (including the Mirror Maker 1 dashboard etc.)
* Delete the Mirror Maker 1 API classes from the `api` module
* Remove the Mirror Maker 1 docs apart from a simple note that Mirror Maker 1 is not supported anymore
* Remove `KafkaMirrorMakerAssemblyOperator`, `KafkaMirrorMakerCluster` and other related classes from the `cluster-operator` module

All these changes are expected to be done before the Strimzi 0.4z release.

### Warnings

Mirror Maker 1 is already deprecated in Strimzi.
Warnings about its upcoming removal (in logs as well as in the `KafkaMirrorMaker` resource conditions) are already being added in Strimzi 0.43.
When we know the exact Strimzi version where the support for Mirror Maker 1 to be removed, we will update the warning messages.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

This proposal removes the support for Mirror Maker 1 without any automatic migration.
As such it does not provide full backwards compatibility.
But it does not affect the backwards compatibility of any other Strimzi operands.

## Rejected alternatives

### Automatic migration

Mirror Maker 2 is the replacement for Mirror Maker one.
And with the right configuration, it can behave similarly to Mirror Maker 1.
The _right_ configuration would include for example:
* Use of the Identity Replication Policy
* Disabling consumer group synchronization

But even with the matching configuration, there would still be many difficulties:
* Monitoring of Mirror Maker 2 is different from Mirror Maker 1 (different dashboards, alerts, etc.)
* Mirror Maker 2 is based on Kafka Connect, so it requires additional topics, configurations and access rights.
  In particular, automatically finding the correct configuration for the Connect cluster that will not conflict with other clusters or ensuring the users have the required privileges might be very complicated.
  Also the resource requirements for Mirror Maker 1 and Mirror Maker 2 might differ.
* It might not be possible to ensure smooth transition without any message loss or without any duplicate messages.

Due to the reasons above, the automatic migration might work only in some simple scenarios.
But might not work for more advanced and customized configurations.
Therefore, the automated migration from Mirror Maker 1 to Mirror Maker 2 was rejected.

In addition to that, Mirror Maker 2 has many new features that were not available in Mirror Maker 1 such as support for active-active mirroring (using automatic topic prefixing) or consumer group offset synchronization.
Having the users do the migration manually might make sure they also investigate and consider using these new and more advanced features.

### Forced deletion

Instead of simply removing the support for Mirror Maker 1 from the Strimzi Cluster Operator, we could consider updating the Mirror Maker 1 reconciliation code to forcibly delete the Mirror Maker 1.
This would mean that we would:
* Keep the `KafkaMirrorMaker` custom resource
* Rewrite the `KafkaMirrorMakerAssemblyOperator` to delete all resources for any Mirror Maker 1 cluster it finds
* Only after several Strimzi releases, we would completely drop the Mirror Maker 1 support and remove it from our codebase.

This would require additional effort to:
* Update the `KafkaMirrorMakerAssemblyOperator` code and all related unit / integration tests
* Update the system tests accordingly

The advantage would be that the deletion of the Mirror Maker 1 cluster would make sure that the users still using it notice that it is not supported anymore, because it will be deleted, all metrics will be gone and messages will stop mirroring.

On the other hand, it would mean that at the point user upgrades the operator to Strimzi versions without Mirror Maker 1 support, we will immediately delete it.
This might negatively impact their Kafka based applications and if they were not aware of the Mirror Maker 1 being removed and have to first start learning about Mirror Maker 2, it might take a significant amount of time to replace the deleted Mirror Maker 1 instance with Mirror Maker 2.
If we keep it running, the users will have more time to consider the alternatives and plan the replacement.
Since this change is coupled with other major changes such as the removal of ZooKeeper support and Kafka 4.0, I believe we can make sure most users are aware of the Mirror Maker 1 removal as well.
And for those who won't be aware of it, not deleting it forcefully seems like a preferred approach.
