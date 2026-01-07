# Move Strimzi Operators to Java 21

Strimzi Kafka operators are currently developed, built, and distributed as a Java 17 project.
Java 17 has been used since late 2022, for almost four years.

The only exceptions are the `api`, `test`, `crd-annotations`, and `crd-generator` modules.
These modules are built with Java 17 but compiled to the Java 11 language level.

This proposal suggests moving the Strimzi Operators build, compilation, and container runtime to Java 21, with temporary exceptions for the `api`, `test`, `crd-annotations`, and `crd-generator` modules.

## Motivation

Strimzi uses the Red Hat builds of OpenJDK.
Red Hat’s current plan is to support Java 17 until mid-2027.
However, Java 17 appears to be getting less attention than the newer LTS versions, and backporting of bugfixes takes longer.
An example of this is the long waiting time for bug fixes in the support for running in containers (see https://github.com/orgs/strimzi/discussions/11966 as an example).

In addition, Java 17 is is superseded by two newer LTS releases (21 and 25).
Moving to a newer Java version helps keep the project up to date.

## Proposal

This proposal suggests that the Strimzi Operators repository and its projects move to Java 21 with immediate effect (expected with the Strimzi 0.50.0 release).
This should include the runtime in the container images as well as the language level of most of the modules.

The only exceptions should be the `api`, `test`, `crd-annotations`, and `crd-generator` modules.
Given the upcoming Strimzi 1.0.0 release and move to the `v1` API, this proposal suggests keeping them at the current Java 11 level until the Strimzi 0.51.0 release.
These modules will be moved to Java 21 only in a second phase with the Strimzi 1.0.0 release.
This allows users to move to Java 21 at the same time as adopting the `v1` API.

### Code changes

There are no major changes required to move to Java 21.
The container image builds will be updated to use Java 21 installation.
This is available within the current UBI9 images and does not require any other changes beyond updating the Java version.
In addition, several dependencies such as Mockito need to be updated to work on Java 21.

### Impact on JavaDocs

Java 21 introduces stricter warnings for Javadocs.
For example, it raises warnings for Javadoc comments without the main message.
However, we no longer treat Javadocs warnings as errors, as we rely on Checkstyle for validating Javadocs according to our own rules.
So while we should address the warnings to avoid lengthy warning messages in the build logs, it is not necessarily a blocker for moving to Java 21.

### `api` module

As mentioned previously, the `api` module and its dependencies (`test`, `crd-annotations`, and `crd-generator`) will remain at the Java 11 language level for Strimzi 0.50.0 and 0.51.0.
It will move to Java 21 only in Strimzi 1.0.0.
However, while moving the other components to Java 21 in Strimzi 0.50.0, we will also announce the plan to move the `api` module to Java 21 in 1.0.0.
That way, users can plan for this change.

### User impact

The move of the `api` module to Java 21 is the main part which might directly impact Strimzi users.
This is because the `api` module is the only part of Strimzi expected to be integrated into users' own applications.
However, there is limited visibility into how widely the `api` module is used directly by users and which Java versions are in use.
As a result, the exact impact is difficult to quantify.

The other components from the Strimzi Operators repo are used only as container images shipped by Strimzi.
So most users should not see any direct impact from a different Java version being used inside.
There may be an impact on users who maintain custom forks of Strimzi or build their own container images.
For these users, aligning their build and runtime environments with Java 21 may be necessary.

### Other projects

#### Drain Cleaner

Drain Cleaner will follow the Strimzi Operators and move to Java 21 as well.
Drain Cleaner is distributed and used as a container image - so minimal user impact is expected.

#### Access Operator

Access Operator will follow the Strimzi Operators and move to Java 21 as well.
As the Access Operator depends on the `api` module, it will need to move to Java 21 when upgrading to Strimzi 1.0.0.
Access Operator is distributed and used as a container image - so minimal user impact is expected.

#### Strimzi HTTP and MQTT Bridge

Strimzi HTTP Bridge and MQTT Bridge (two separate subprojects) will remain using Java 17 as the language level and maintain compatibility with Java 17.
They will move to Java 21 as the runtime in the container images to provide consistent experience with the other Strimzi operands.
But users running them on their own on bare-metal / virtual machines can continue to run them with Java 17.

This approach helps to address the _cgroups_ issues in Java 17 while leaving the users running the Bridge outisde of containers unaffected.
Both projects will move completely to Java 21 on their own schedule (subject to a separate proposal).

#### Small libraries

This proposal does not suggest any changes to our library-type subprojects such as the Configuration providers, Metrics Reporter, or the OAuth library.
These libraries are expected to be included and used by user applications.
Thus, changing the minimum Java version for these projects would have much bigger impact on users.
The desired Java version used by these projects should be decided independently on a project-by-project basis.

## Rejected alternatives

### Move directly to Java 25

Java 25 is currently the latest available Java LTS release.
Moving directly to Java 25 would let us to use the newest and most up-to-date Java version and Java features.
Given the minimal effort required to move to Java 21, the proposal recommends adopting Java 21 first. This will allow experience to be gathered before considering a move to Java 25 in mid‑2026.

### Keeping the `api` module as Java 11 or 17

The `api` module and its dependencies could remain at the Java 11 language level, or be moved only to Java 17 instead of Java 21.
Without more usage data, there appear to be no strong reasons to keep it at the older version.
This decision can be revisited if sufficient demand arises.
