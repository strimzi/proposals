# Support for Apache Kafka 4.0

Apache Kafka is planning a new major release - Apache Kafka 4.0.
Unlike previous major releases that Strimzi handled before (2.0 and 3.0), this one is "special" because it drops the support for ZooKeeper-based clusters.
This proposal discusses how Strimzi will adopt the release of Apache Kafka 4.0 and transition from Apache Kafka 3.x to 4.x.

## Current situation

### Strimzi

Strimzi currently does a new minor release every roughly every 2 months.
Each release supports at least the two last Kafka minor versions.
For example 3.6.x and 3.7.x.
For each minor version, multiple patch releases might be supported, depending on their availability in Apache Kafka.
When a new minor Apache Kafka version is released, Strimzi will typically adopt the new minor version and drop support for the oldest supported minor Kafka version.
Typically, this happens every other Strimzi release as Apache Kafka has slower release cycles compared to Strimzi.
But in some cases, there might be more Strimzi releases in between (for example, Strimzi 0.40, 0.41 and 0.42 are all based on Kafka 3.6.x and 3.7.x).

Once a new minor Strimzi version is released, we will stop supporting the previous Strimzi release.
There are some exceptions, especially in the following cases:
* Serious bugs or CVEs
* The latest minor Strimzi release is new and some people might not have had chance to upgrade to it

#### KRaft support

Strimzi currently supports both ZooKeeper- and KRaft-based Apache Kafka clusters.
It also supports migration of ZooKeeper-based clusters to KRaft.
The current limitations to the KRaft support are based on limitations in Apache Kafka itself.
At the time of writing this proposal, limitations include:
* Scaling of KRaft controller nodes up or down is not supported
* Unregistering Kafka nodes removed from the Kafka cluster

In addition to the limitations, we are waiting for some further improvements in Apache Kafka:
* Ability to connect to KRaft controller nodes and manage them without rolling updates
* JBOD support in KRaft moving to GA

Some of these limitations might be addressed before Apache Kafka 4.0 is released.
But it might not apply to all of them.
Also, after they are released in Apache Kafka, additional time might be needed to adopt  and use them in Strimzi.

### Apache Kafka

Apache Kafka does a new minor or major release roughly every 4 months.
But this is subject to change when required by the circumstances.
The current Apache Kafka releases might be expected to happen in the following order:
* Apache Kafka 3.8.0 release (currently in the release process, expected release in July or August?)
* Apache Kafka 3.9.0 release with the remaining missing KRaft features (shorter release cycle - expected August or September?)
* Apache Kafka 4.0.0 release 3-4 months after the Kafka 3.9 release without ZooKeeper support (late 2024, early 2025?)

The release plan is of course subject to change.

Apache Kafka currently supports every minor release for 1 year.
During this time, each minor release would typically get one or two patch releases with bug or CVE fixes.
The 1 year period is currently expected to apply for the Apache Kafka 3.9 release that is expected to be the last with ZooKeeper support.

## Motivation

The move from Apache Kafka 3.x to 4.x and the related move from ZooKeeper-based Kafka clusters to KRaft is a major change for our users as well as for Strimzi itself.
Having a good plan for how we plan to deal with it is important as it will provide clarity to Strimzi users, developers, and create a plan of the things that need to be done.
It should also help our users to properly plan when to migrate from ZooKeeper to KRaft, as this is a major change that might require planning several months ahead of time.

## Proposal

Strimzi should approach adoption of Kafka 4.0 in the following way:

1) When Apache Kafka 3.9.0 is released, we will adopt it in Strimzi as any other Apache Kafka release.
   We will release it as part of a Strimzi release 0.4x which will support both Kafka 3.8.x and 3.9.x.
   This version will support both ZooKeeper- and KRaft-based clusters.
   This version will also add warnings to all users using ZooKeeper that it will be removed in the next Kafka release and that they should do the migration.

2) Given the expected timeline of 3-4 months between Kafka 3.9 and 4.0, we will do another minor Strimzi release 0.4y.
   This version might add support for some of the new features added to KRaft in Kafka 3.9.0.
   0.4y will keep the support for 3.8 and 3.9 as well as support for both KRaft and ZooKeeper.
   This version will be subject to the extended support described below in a dedicated section.
   It will also keep the warnings introduced in Strimzi 0.4x.

3) Once Strimzi 0.4y is released, we will proceed and remove ZooKeeper support from Strimzi.
   This would include things such as:
   * Deprecating the ZooKeeper related parts of the CRDs
   * Removing the ZooKeeper reconciler and all other parts of the code relevant to ZooKeeper
   * Remove the migration code
   * Remove ZooKeeper related system tests

   As upgrade to Strimzi 0.4z will not be possible without migration to KRaft, we will know for sure that the clusters run by Strimzi 0.4z had to be previously run by recent Strimzi versions.
   That will also allow us to remove other legacy code.
   For example the migration of Kafka clusters from StatefulSets to StrimziPodSets would not be needed anymore and would be removed.

   Once Apache Kafka 4.0 is released, we will adopt it and do a Strimzi release 0.4z.
   0.4z and all later Strimzi versions will support only KRaft based clusters.
   Strimzi 0.4z will support the last two Kafka minor versions, as usual - Kafka 3.9 and 4.0 - and will allow users to gradually upgrade as always.
   But it will support only KRaft and users with ZooKeeper-based clusters will need to migrate to KRaft first within the Strimzi 0.4x and 0.4y versions and Kafka 3.9.

4) The Strimzi releases after 0.4z will continue with the 1-2 months release cycles, as usual, and will continue to adopt the Kafka patch releases.
   Once Apache Kafka 4.1 is released, Strimzi will adopt Kafka 4.1 and continue its releases with support for Kafka 4.0 and 4.1.

### Enforcing the migration to KRaft

There is no way we can enforce the migration to KRaft or do it automatically.
Users have to create the controller nodes and initiate the migration.
This cannot be done automatically, as we would not know what the right configuration for the KRaft controller should be (storage, resources, affinity, tolerations, ...).

To try to get users to migrate to KRaft in time, we will produce warning log messages and warning conditions in the `.status` section in the last Strimzi version supporting ZooKeeper (0.4x and 0.4y).

In Strimzi 0.4z and newer versions we will check the Kafka clusters for the metadata type they use.
For any cluster not using KRaft it will immediately raise an error instead of trying to reconcile the cluster.
The check will be done based on the `Kafka` CR `.status` section.
Users will have to downgrade the operator, migrate the cluster and upgrade the operator again.

Newly deployed Kafka clusters will be expected to be KRaft only.

### Extended support for Kafka 3.9

Based on the feedback from the community, there seem to be some users who are very eager to move to KRaft.
But also many users who prefer to stick with ZooKeeper for longer.
We should try to accommodate these users by providing extended support for the last Strimzi release with support for Apache Kafka 3.9 and ZooKeeper-based clusters.
This extended support should include:
* Trying to fix critical CVEs
* Add support for new Apache Kafka 3.9 patch releases 
* Fixing critical bugs related to ZooKeeper, KRaft and migration from ZooKeeper to Kraft

The extended support will not include back-porting any new features.
It is also not expected to include any changes for compatibility with new Kubernetes versions released only after the Strimzi release.

Releases of new patch versions will be not be done with a frequency higher than 2 months and should be driven based on user demand.

The extended support will be done at most for as long as the Kafka 3.9 version is supported by the Apache Kafka project (i.e. 1 year after the Kafka 3.9 release).
But it might be terminated sooner in case we run into major issues with fixing any bugs or CVEs that would require unreasonably high effort.
(While it is rare, addressing certain CVEs might require significant dependency updates that require major changes to the codebase.)

### System tests

With the removal of the ZooKeeper support from the production code, we should also proceed and remove it from the system tests.
We will also remove the `kraft-regression` CI pipeline from the `main` branch and use the `regression` branch for running Kraft-based system tests.
The system test pipelines will remain unchanged in the 0.4y release branch for the extended support.
Given the extended support for the Strimzi 0.4y version, we should consider adding a dedicated upgrade test from a KRaft-based Strimzi 0.4y cluster to the latest Strimzi version as some users are expected to run Strimzi 0.4y for extended time and later migrate directly to a newer Strimzi version.

### Existing KRaft limitations

The existing KRaft limitations in Strimzi are caused by limitations in Apache Kafka itself.
While some of them might be addressed in Apache Kafka 3.9.0, additional time will be needed to adopt them in Strimzi and they might not be all addressed in the last Strimzi 0.4y release with ZooKeeper support.
They might be supported only in Strimzi 0.4z release based on Kafka 3.9 and 4.0 with KRaft-only support.
Support for these features will not be back-ported to the older Strimzi version with extended support.

### Risks

Leaving aside the questions about the quality of the KRaft implementation and the missing features, the biggest risks are last-minute changes to the Apache Kafka release timelines and general plans.
For example, a decision to do another 3.x minor version (3.10) and postponing the release of Kafka 4.0 some time after Strimzi 0.4y was released when the `main` branch of the Strimzi operators repo already removed ZooKeeper support might raise many issues.
It would likely force us to do development in two different branches of the operator:
* One with both ZooKeeper and KRaft support
* And one with KRaft only support

But this applies only to situations when the new Kafka 3.x version is inserted into the release plan after the Strimzi 0.4y release and after we start removing support for ZooKeeper from Strimzi.
If the decision to add Kafka 3.10 is done before the Strimzi 0.4y release when we still support both ZooKeeper and KRaft in the `main` branch, we can simply shift the plan from this proposal by one release. 

## Affected projects

This proposal affects the Strimzi operator.
Mainly the Cluster Operator which deals with KRaft and ZooKeeper.
But to some extent, it also affects other parts of the project (such as the API, container images, examples, system tests etc.)
The Topic and User operators as well as the Bridge, Drain Cleaner or configuration providers should not be affected by this proposal as they do not have any direct dependency on ZooKeeper.

## Backwards compatibility

While this proposal maintains the backwards compatibility for existing KRaft clusters, it requires all users to migrate the ZooKeeper-based Kafka clusters to KRaft first.
This is not a Strimzi's choice, but a result of the development in the Apache Kafka project.

## Rejected alternatives

### Skip the Strimzi 0.4y release

We could consider skipping the Strimzi 0.4y release and instead remove the support for ZooKeeper-based clusters right after releasing Strimzi 0.4x.
In that case, the Strimzi 0.4x will receive the extended support.
But assuming the interval between Kafka 3.9 and 4.0 is 3-4 months, it seems that keeping the 0.4y release can help us implement some more features and possibly fix some more bug fixes before we drop ZooKeeper support.
Doing the 0.4y release would also minimize the risk of Apache Kafka introducing a 3.10 release after the 3.9 release is done.
This should also simplify the extended support, since we will have to do it for a shorter time in a separate release branch and save some effort (from the 1 year of support window of Kafka 3.9, 1-2 months will be gone before we release the 0.4y with extended support).

### Transition from Apache Kafka 3.9 directly to 4.0

An alternative option would be to move from Kafka 3.9 directly to Kafka 4.0.
That would mean that we would have one or more Strimzi version(s) with support for Kafka 3.8.x and 3.9.x.
And once Kafka 4.0.0 will be released, we will drop support for 3.X completely and the next Strimzi version would support Kafka 4.0.x only.

This approach was rejected, because:
* It does not seem to offer support for 2 last minor Kafka versions which is something Strimzi offered for a long time.
* We would either need to wait for Kafka 4.0.0 to be released (3-4 months after 3.9 release) to prepare the release, or we would anyway need to do all the work already with Kafka 3.9 anyway.

So it would leave us possibly for a very long time without any release and does not seem to bring any real advantages over the approach suggested by this proposal.

### Continue ZooKeeper support until Kafka 4.1 is released

Another alternative option would be to continue releasing Strimzi versions based on Apache Kafka 3.9 and 4.0 until Kafka 4.1 is released while supporting ZooKeeper with Kafka 3.9.
Once Kafka 4.1 would be released, we would drop support for Kafka 3.9 and for ZooKeeper and continue with Strimzi versions supporting Kafka 4.0 and 4.1 with KRaft only.

The disadvantage of this approach is that it would require additional effort to validate the used versions and make sure that users are not trying to deploy the ZooKeeper based Kafka cluster with Kafka 4.0.
It might also lead to confusing user experience.
For example, a Kafka CR without explicitly specified Kafka version should use the latest supported Kafka version.
Would that mean that ZooKeeper based clusters without specified `.spec.kafka.version` would automatically end up in error state because they would use Kafka 4.0?
Or will we add some special handling that in such cases the version would default to Kafka 3.9?

It is also not clear if this approach will bring any clear benefits.
It would mean that we postpone the removal of the ZooKeeper related code by ~4 months (release interval between Kafka 4.0 and 4.1 releases).
But while this delay will not benefit us as the Strimzi developers, it also doesn't seem to provide any clear benefits to the users over the solution suggested by this proposal.
