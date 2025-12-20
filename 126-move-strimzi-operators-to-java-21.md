# Move Strimzi Operators to Java 21

Strimzi Kafka operators are currently developed, built, and distributed as a Java 17 project.
Java 17 is used for more than 3 years since late 2022.

The only exception are the `api`, `test`, `crd-annotations`, and `crd-generator` modules.
These modules are currently built with Java 17, but use the Java 11 language level.

This proposal suggests to move on to Java 21.

## Motivation

Strimzi is using the Red Hat builds of OpenJDK.
Their current plan is to support Java 17 until mid-2027.
However, it seems that Java 17 is getting less attention than the newer LTS versions and backporting of bugfixes takes longer than with the newer versions.
An example of this is the long waiting time for bug fixes in the support for running in containers (see https://github.com/orgs/strimzi/discussions/11966 as an example).

In addition, Java 17 is now pretty old with two newer LTS releases already available (21 and 25).
So moving to a newer Java version makes sense even to just keep up to date.

## Proposal

This proposal suggests that the Strimzi Operators repository and its projects move to Java 21 with immediate effect (expected with the Strimzi 0.50.0 release).
This should include the runtime in our container images as well as the language level of most of the modules.

The only exception should be the `api`, `test`, `crd-annotations`, and `crd-generator` modules.
Given the upcoming Strimzi 1.0.0 release and move to the `v1` API, this proposal suggests to keep them under the current Java 11 level until Strimzi 0.51.0 release.
These modules will be moved to Java 21 only in a second phase with the Strimzi 1.0.0 release.
This should allow any users using them to transition to Java 21 together with transitioning to the `v1` API.

### Code changes

There are no major changes required to move to Java 21.
The container image builds will be updated to use Java 21 installation.
This is available within the current UBI9 images and does not require any other changes than just the Java version change.
In addition, several dependencies such as Mockito need to be updated to work on Java 21.

### JavaDocs

Java 21 introduces stricter warnings for Javadocs.
For example, it raises warnings for Javadoc comments without the main message.
However, we no longer treat Javadocs warnings as errors, as we rely on Checkstyle for validating Javadocs according to our own rules.
So while we should address the warnings to avoid lengthy warning messages in the build logs, it is not necessarily a blocker for moving to Java 21.

### `api` module

As mentioned above, the `api` module and its dependencies (`test`, `crd-annotations`, and `crd-generator`) will remain at the Java 11 language level for Strimzi 0.50 and 0.51.
It will move to Java 21 only in Strimzi 1.0.0.
However, while moving the other components to Java 21 in Strimzi 0.50, we will also announce the plan to move the `api` module to Java 21 in 1.0.0.
That way, the users can plan with it.

### User impact

The move of the `api` module to Java 21 is the main part which might directly impact Strimzi users.
This is because the `api` module is the only part of Strimzi that we expected to be directly integrated in user's own application.
However, we do not know if or how many users actually use it and what Java version they might use.
So it is hard to judge the actual impact.

The other components from the Strimzi Operators repo are used only as container images shipped by Strimzi.
So most users should not see any direct impact from a different Java version being used inside.
There is possible impact on some advanced users using their own forks of Strimzi or custom builds and their own base container images.
But we should expect that these users are experienced and are able to handle this change easily.

### Other projects

#### Drain Cleaner

Drain Cleaner will follow the Strimzi Operators and move to Java 21 as well.
Drain Cleaner is distributed and used as a container image - so minimal user impact is expected.

#### Access Operator

Access Operator will follow the Strimzi Operators and move to Java 21 as well.
As the Access Operator depends on the `api` module, moving the `api` module to Java 21 will require the Access Operator to move to Java 21 latest when moving to Strimzi 1.0.0 as a dependency.
Access Operator is distributed and used as a container image - so minimal user impact is expected.

#### Strimzi HTTP and MQTT Bridge

Strimzi HTTP and MQTT (two separate subprojects) will follow the Strimzi Operators and move to Java 21 as well.
Unlike the projects discussed above, they are distributed not only as container images but also as archives for running locally with pre-installed Java Runtime Environment.
However, they are not integrated into user applications.
So moving to Java 21 should mean installation / configuration of a newer JVM and should not have any bigger impact.

#### Small libraries

This proposal does not suggest any changes to our library-type subprojects such as the Configuration providers, Metrics Reporter, or the OAuth library.
These libraries are expected to be included and used by user applications.
Thus changing the minimal Java version for these projects would have much bigger impact on users.
The desired Java version used by these projects should be decided independently on a project-by-project basis.

## Rejected alternatives

### Move directly to Java 25

Java 25 is currently the latest available Java LTS release.
Moving directly to Java 25 would let us to use the newest and most up-to-date Java version and Java features.
However, there seems to be minimal effort to move to Java 21.
So I decided to propose moving to Java 21 first.
We can use it to collect some more experience and if desired, consider moving to Java 25 few months later in mid 2026.

### Keeping the `api` module as Java 11 or 17

We could keep the `api` module and its dependencies at the Java 11 language level, or move it only to Java 17 language level instead of moving it to Java 21.
But without more usage data, I do not see any strong reasons to keep it at the older Java version.
I'm happy to change this decision and update the proposal if a sufficient demand for it shows up.

### Keep Bridge at Java 17 and move only the container runtime to Java 21

We can consider keeping the Bridge at Java 17 and update only the Java version used in the container image to Java 21.
That would help us to get rid of the _cgroups_ issues in Java 17 while leaving the Bridge itself unimpacted.
But similarly to the `api` module, we do not seem to have any strong reasons to keep it at the older Java version.
