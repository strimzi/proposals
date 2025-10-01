# `v1` CRD API Conversion Tool

This proposal covers the `v1` CRD API Conversion Tool, which will be available to users to help migrate from the `v1beta2` API to the `v1` API.

## Motivation

The migration of the Strimzi CRs from the `v1beta2` version to the `v1` version can be done manually.
We plan to cover the manual process as part of our documentation.
However, given the number of steps and, in some cases, a large number of resources that would need to convert, we will provide a Conversion Tool to help.
Providing a tool was approved as part of the initial [Strimzi `v1` CRD API and 1.0.0 release](./113-Strimzi-v1-CRD-API-and-1.0.0-release.md) proposal.
This proposal provides additional details.

## Proposal

### Conversion Tasks

The conversion tool should be able to fulfill the following tasks:
* Support conversion of the custom resources by modifying their structure to account for the fields that were renamed, moved, or removed:
    * Be able to do the conversion inside a Kubernetes cluster across all namespaces, in a single namespace, or for a particular custom resource only
    * Convert YAML files with the Strimzi custom resources in-place (by modifying the existing file) or by creating a new file
* Support updating the CRDs:
    * Change the storage version to `v1` in the CRD `.spec` section
    * "Touch" all custom resources to trigger a storage migration to make sure they are stored under the new version
    * Change the stored versions in the CRD `.status` section in order to allow the old versions to be removed

Users should be able to use only some of these tasks if needed and combine the Conversion Tool and the manual conversion process as needed.

### Architecture

The Conversion Tool will be written in Java and mostly follow the architecture of the [previous tool](https://github.com/strimzi/strimzi-kafka-operator/tree/0.22.0/api-conversion) we used last time when migrating from `v1beta1` to `v1beta2` API.
Using Java would simplify the release process as it would depend directly on the Strimzi API.

The Conversion Tool itself will be distributed as ZIP and TAR.GZ archives.
The archives will contain helper scripts for running it from the command line.
It will use Java 17 as with most of the other Strimzi components.

It will also be bundled in the Strimzi Operator container.
Bundling the tool in the container enables users who cannot (or prefer not to) install the Java Runtime Environment (JRE) to run it as a Kubernetes Job or locally as a Docker container.

### Command line interface structure

The conversion tool will use the [picocli](https://picocli.info/) library for handling command line options.
It will have 3 commands:

* `convert-file` for converting a YAML file.
  This option will either convert it into a standard output, a new file, or just update the original file.
* `convert-resource` or `convert-resources` will convert resources directly in an Kubernetes cluster.
  Users can use it to convert all Strimzi resources in all namespaces.
  Users can also specify only specific kinds and/or a single namespace to limit the resources that will be converted.
* The `crd-upgrade` option will _convert_ the CRDs.
  It will first update the stored version in the CRDs to `v1`.
  Then it will _touch_ all custom resources to make sure they are stored under the `v1` version.
  And finally, it will remove the `v1beta2` version (and `v1alpha1` / `v1beta1` version for `KafkaTopic` and `KafkaUser` resources) from the CRD status.

### Workflow

Users will be able to use the `convert-file` or `convert-resource` commands at any point between the Strimzi 0.49 and 0.52 / 1.0.0 release.
The conversion must be done after upgrading to Strimzi 0.49 or later.
Conversion is required no later than the upgrade to Strimzi 0.52.0 / 1.0.0.
The converted resources will work with the Strimzi versions 0.49.0 and newer.
Users can decide whether to convert resources directly in the cluster or, for example, convert YAML files offline and commit them to a GitOps repository.

Shortly before the upgrade to Strimzi 0.52 / 1.0.0, users will also be required to run the `crd-upgrade` command.
After successfully running this command, they will be able to install Strimzi 0.52 / 1.0.0 which will contain only the `v1` CRDs.
If they try to upgrade to Strimzi 0.52 / 1.0.0 without running the command, the installation will fail as the CRDs will not be updated.

### Limitations

#### Workflow

Requiring users to do the manual steps as part of the upgrade process is unfortunate.
However, from experience we know that many users use different versions of Strimzi on the same Kubernetes cluster.
So it is not easy to automate the conversion, for example, at Cluster Operator startup.
While that would make it easier for some users, it might be a breaking change for other users.
Therefore, we propose sticking with the manual process used for the `v1beta2` migration. 
It may not be perfect, but it is feasible for all users.

#### Conversions

Most of the changes in our custom resources between `v1beta2` and `v1` are fields that are deprecated and removed.
But some of the changes also include fields that changed, moved, etc.
The Conversion Tool will aim to handle as many of the changes as possible.
However, it will not be able to handle all of them (as already covered in the [Strimzi `v1` CRD API and 1.0.0 release](./113-Strimzi-v1-CRD-API-and-1.0.0-release.md) proposal).
The conversion is not possible, for example, when it involves the handling of secrets or environment variables that must be mounted or mapped.

The changes which the Conversion Tool will not be able to handle are:
* Change from `type: oauth` authentication to `type: custom` for both Kafka brokers as well as for the client-based operands
* Change from `type: keycloak` and `type: opa` authorization to `type: custom` authorization
* Removal of the support for mounting Secrets as part of the `type: custom` authentication
* Removal of the `externalConfiguration` section in the `KafkaConnect` CR

However, the Conversion Tool will detect these changes, fail the conversion, and request the user to do the conversion manually before re-running the tool.
It will also provide a link to the documentation for the manual conversion.

### Release process

The Conversion Tool will live as an additional module in the main Strimzi Operators repository (and will be removed after the 0.52 / 1.0.0 version once it is not needed anymore).
The module name will be `v1-api-conversion` to differentiate it from the previous version called `api-conversion`.
That will allow it to use the same `api` module version as is currently under development.
It will also allow us to release it as part of the regular Strimzi release process.

## Documentation

Documentation will include the procedure for how to use the Conversion Tool to convert the resources together with the documentation of the manual conversion process as well as the upgrade to the `v1` API version in the Strimzi 0.52 / 1.0.0 version.
The migration process is described in detail in the main [_Strimzi v1 CRD API and 1.0.0 release_ proposal](https://github.com/strimzi/proposals/blob/main/113-Strimzi-v1-CRD-API-and-1.0.0-release.md).

### Testing

The conversion of the Strimzi custom resources will be tested using unit and integration tests in the Conversion Tool module itself.
Additionally, System Tests will be added to test the whole lifecycle of the conversion:
* Starting with both old and new API versions
* Creating the resources
* Running the conversion
* Removing the old API versions (and eventually in the last phase also install the operator using the `v1` API version)

The System tests related to the conversion will be removed again after the Strimzi 0.52 / 1.0.0 release.

## Rejected alternatives

### Writing the Conversion Tool in Golang

I originally had the idea to write the Conversion Tool in Golang based on the [Strimzi Go API](https://github.com/scholzj/strimzi-go).
Using Golang would allow us to provide users with native binaries that are easy to use from the command line on the most common operating systems and architectures.
It would not require the installation of any Java runtime Environment.
It would also make it easy to use them from container images and Kubernetes Jobs.

While the [Strimzi Go API](https://github.com/scholzj/strimzi-go) is currently not part of the Strimzi project, the existence of the `v1` CRD conversion tool is limited by the end of the `v1` migrations.
So it might not be a limitation, and it does not justify moving it to Strimzi and forcing Strimzi to maintain it (although I'm happy to consider moving it to Strimzi if people are interested in taking over the maintenance of it).

However, after a [discussion on our Slack](https://cloud-native.slack.com/archives/C018247K8T0/p1758225953581479), the conclusion was that we lack the Golang knowledge to be able to review the Go-based code.
And as a result, the idea was rejected.

### Using native Java compilation with GraalVM

To provide easily distributable native binaries, we considered GraalVM native image.
However, this was rejected because the Java native compilation:
* Is slow and would provide major slowdowns of our systems
* Does not provide efficient cross-compilation support in order to allow us building native binaries for all required platforms in our CI (e.g. MacOS or Windows)
* Might require additional testing on the different platforms
