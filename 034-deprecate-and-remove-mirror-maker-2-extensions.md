# Deprecate and remove Mirror Maker 2 extensions

## Current situation

When Mirror Maker 2 was first introduced, it was missing a policy for replicating topics between two Kafka clusters without changing their names.
We created the [_Mirror Maker 2 Extensions_ project](https://github.com/strimzi/mirror-maker-2-extensions) which contains the `IdentityReplicationPolicy` policy.
This policy makes sure that topics are not renamed while being mirrored.
The `IdentityReplicationPolicy` is the only _extension_ we provide ass part of this project.

In Kafka 3.0.0, Mirror Maker 2 introduces its own `IdentityReplicationPolicy`.
This was done in [KAFKA-9726](https://issues.apache.org/jira/browse/KAFKA-9726).
This policy provides the same features as Strimzi's own `IdentityReplicationPolicy`.
It does not seem to make sense to maintain our own policy in parallel to Apache Kafka.

## Proposal

After dropping support for Kafka versions older than 3.0.0 (expected to happen in Strimzi 0.28.0), we should deprecate our own policy and migrate users to the Kafka's identity replication policy.

Following steps should be taken as part of the 0.28.0 release:
* Update our documentation to use Kafka's `IdentityReplicationPolicy`.
* Update our examples to use Kafka's `IdentityReplicationPolicy`.
* Update the `CHANGELOG.md` file to indicate the deprecation.
* Release new version of the MM2 Extensions which would instead of implementing our own logic just extend Kafka's `IdentityReplicationPolicy`.
  That should ensure exactly the same behavior from both policies.
  It would also make sure that any users still using `io.strimzi.kafka.connect.mirror.IdentityReplicationPolicy` do not have any problems
* After the release, clearly deprecate and archive the `mirror-maker-2-extensions` repository.
  (If we ever need any other MM2 extension in the future, we can always unarchive it)

Even in 0.28 and newer releases, will still include the `mirror-maker-2-extensions` JARs to ensure compatibility.
The JAR will be removed only in Strimzi 0.32.0 or Strimzi 1.0.0 (whichever comes first).
From that version on, users will need to use the Apache Kafka policy in their configuration as `org.apache.kafka.connect.mirror.IdentityReplicationPolicy`.

Any users using the Mirror Maker 2 Extensions with older versions of Apache Kafka outside of Strimzi will still be able to get the previously released versions from our Maven repositories.
Also anyone building older version of Strimzi will be able to use the older versions and will not be impacted by the archiving of the GitHub repository.

## Affected/not affected projects

This proposal impacts the Mirror Maker 2 Extensions and Strimzi Kafka Operators projects.

## Backwards compatibility

The impacts on the backwards compatibility are described in the proposal.

## Rejected alternatives

Keep maintaining our own `IdentityReplicationPolicy` in parallel with Kafka.
