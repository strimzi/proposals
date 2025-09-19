# Buildah - replacement of Kaniko after its archival

In the `KafkaConnect` custom resource, users can specify plugins to include in the Connect image, without needing the build the image externally.
To support this feature on both Kubernetes and OpenShift, we have two implementations.
For OpenShift we are using the Build API that OpenShift offers.
But on Kubernetes, we are using [Kaniko](https://github.com/GoogleContainerTools/kaniko).
However, the Kaniko project was [archived](https://github.com/GoogleContainerTools/kaniko/commit/236ba5690eda9170d0157aa8137ebbeb09d38685) and it's no longer developed or maintained.
This proposal aims to resolve this issue.

## Motivation

The main motivation behind this proposal is to resolve the issue that Kaniko is archived, and it will not receive any updates (no bug or CVE fixes).
Currently, it seems to not be a problem, but when some critical CVE or bug arises, we have to fix it.
We also need support in case a blocking issue arises on newer Kubernetes versions, which we would no longer receive with Kaniko.

## Proposal

For the Connect Build feature on Kubernetes, we will use [Buildah](https://buildah.io/) as a replacement of Kaniko.
Buildah is a well-supported, widely used tool for building container images.
It also appears that OpenShift’s Build API relies on Buildah, which makes Buildah a strong alternative to Kaniko, with the added assurance of long-term support and maintenance.
Other than that, Buildah doesn't need any workaround in order to run it on Kubernetes rootlessly - the only thing that is needed during the build and push stages is to specify `--storage-driver=vfs` option.
The VFS (Virtual File System) driver provides a user-space implementation. 
It stores each layer as a full copy of the files, but does not require root permissions.
The only trade-off is the build time (it's slower) and consumption of disk space.

Even though we will run Buildah with these options, which should make it rootless, it still needs some capabilities to run without issues during the build phase.
That means we cannot run the build Pod with restricted Pod Security profile, as these capabilities are dropped in these profiles.
This is more described in the [Running Buildah in restricted environment](#running-buildah-in-restricted-environment) section of this proposal.

We will use the official Buildah images on Quay.io (currently `quay.io/buildah/stable:v1.40.1`).
As with the Kaniko executor image, we will pull the official image, tag it with the version, and push it to the Strimzi repository on Quay.
That's useful because of our versioning. 
We will have the image ready even if someone on the Buildah side decides to remove it from their Quay repository.
The full name of our Buildah image will be `quay.io/strimzi/buildah:OPERATOR_VERSION` - for `main` branch it will use the `latest` tag, otherwise it will be tagged by the operator version for the particular release.

We will upgrade the Buildah image always to the latest stable version available, primarily to address CVEs and other security issues.

Users then can specify their own Buildah image using `STRIMZI_DEFAULT_BUILDAH_IMAGE` (similarly to what is possible today with Kaniko).
In case that users will use Kaniko, the Buildah environment variable will be ignored, and vice versa.
The users will be responsible for changing these environment variables based on the mode in which they run the Connect build - in case that they have customized Deployment files.

The usage of Buildah will be gated behind a feature gate called `UseConnectBuildWithBuildah` that will have following schedule:

| Alpha (opt-in) | Beta (default-on) | GA     |
|----------------|-------------------|--------|
| 0.49.0         | 0.52.0            | 0.55.0 |

The build of the Connect image will be done the same way as today, the main difference will be in the commands used in the build and push process.
Kaniko executor did everything at once, which can be done by Buildah as well, but I decided not to implement it as a single command because:

- We will not get the SHA of image after the push.
- Once pushed, the image will not be stored locally, so we cannot check the SHA using some different command.

The SHA of the image is needed to keep the compatibility with the previous implementation using Kaniko.
Kaniko, once the image is built and pushed, is able to return the full name of the image together with the SHA to a specified path.
That is done using the `--image-name-with-digest-file` option of the Kaniko executor.
Unfortunately, Buildah doesn't have such option in case that you want to build and push the image in one command (that is done using the `buildah build` command, in order to push directly, you need to prefix the image with `docker://`).
Buildah has an option to store the SHA in a specified file using the `--digestfile` option, but only when the `push` command is used.
It will just output the SHA, not the full image, so we need to build it from the SHA and built image and output it to `/dev/termination-log` in order to have it inside the message of the completed Pod.

We will use following commands:

```shell
buildah build --file=/dockerfile/Dockerfile --tag=<IMAGE> --storage-driver=vfs <ADDITIONAL_BUILD_OPTS>
buildah push --storage-driver=vfs --digestfile=/tmp/digest <ADDITIONAL_PUSH_OPTS> <IMAGE>
buildah images --digests --filter=digest=sha256:$(cat /tmp/digest) --format='{{.Name}}@{{.Digest}}' > /dev/termination-log
```
- `<IMAGE>` is placeholder for user desired name of the image (with registry, repository, and possibly tag)
- `<ADDITIONAL_BUILD_OPTS>` is placeholder for user desired additional options for the `buildah build` command
- `<ADDITIONAL_PUSH_OPTS>` is placeholder for user desired additional options for the `buildah push` command

We need to take these three steps in order to get the SHA of the image together with correct repository.
This will work in all scenarios, including when the user does not specify a tag and the default (`latest`) is used.

Together with this feature, and upcoming v1 API, we will do following changes inside the `DockerOutput` model:

1. we will deprecate `additionalKanikoOptions` field as the name of it is connected directly to Kaniko and more generic name can be used (this field will be removed as part of the v1 API).
2. we will add two new fields - `additionalBuildOptions` and `additionalPushOptions` - to cover additional options for both phases

The `additionalKanikoOptions` will still be used with Kaniko until the `UseConnectBuildWithBuildah` feature gate moves to GA or until it is removed in the CRD v1 API, whatever comes first.
An automated warning about deprecation will be added inside the `.status` section of the `KafkaConnect` CR in case it is used as for all other deprecated fields.
Instead of `additionalKanikoOptions`, users should use `additionalBuildOptions`, but not `additionalPushOptions`.
We will not split the Kaniko's additional options to two groups, and we will keep everything configured in one.
That's because Kaniko doesn't have the build and push phases split, but everything is done in one command.

Buildah, on the other hand, has these phases split and has its own set of options for both.
In order to allow specifying different options and their values for both of these phases, we will add two fields.
Additionally, if user will use Kaniko and specify both `additionalKanikoOptions` and `additionalBuildOptions`, `additionalBuildOptions` takes precedence.

For Buildah, we will check only the `additionalBuildOptions` and `additionalPushOptions` fields.
In case that user will specify `additionalKanikoOptions` with Buildah enabled feature gate, only the deprecation warning as a condition will be added into the `.status` section of `KafkaConnect` CR, but these options will be ignored.

As in case of Kaniko, Buildah will have also some allowed options that can be specified for the `build` and `push` phases:

- `additionalBuildOptions`
  - `--authfile` - path to the authentication file (usually created using the buildah login).
  - `--cert-dir` - path to directory with certificates that should be used.
  - `--creds` - the `[username[:password]]` to use to authenticate with the registry if required.
  - `--decryption-key` - the `[key[:passphrase]]` to be used for decryption of images.
  - `--retry` - number of times to retry in case of failure during image pull (the base one).
  - `--retry-delay` - duration between retry attempts.
  - `--tls-verify` - skip TLS verification for insecure registries.

- `additionalPushOptions`
  - `--authfile` - path to the authentication file (usually created using the `buildah login`).
  - `--cert-dir` - path to directory with certificates that should be used for connecting to registries.
  - `--creds` - the `[username[:password]]` to use to authenticate with the registry if required.
  - `--retry` - number of times to retry in case of failure during image pull (the base one).
  - `--retry-delay` - duration between retry attempts.
  - `--tls-verify` - skip TLS verification for insecure registries.
  - `--quiet` - when writing the output image, suppress progress output.

If the user-specified Buildah options will contain forbidden options (or not known), the `InvalidResourceException` will be thrown (logged inside the operator log and in the `.status` section of `KafkaConnect` CR) and the build will fail.

Finally, this feature will be available on Kubernetes only - it will not be available on OpenShift, which is mentioned as one of the rejected alternative.
So the implementation and usage of OpenShift Build API will be kept.

### YAML example of configuring Connect Build with Buildah options

The following YAML example shows how the `KafkaConnect` CR with build configuration (and Buildah additional options) will look like:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  version: 4.1.0
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        pattern: "*.crt"
  config:
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
    config.storage.replication.factor: -1
    offset.storage.replication.factor: -1
    status.storage.replication.factor: -1
  build:
    output:
      type: docker
      image: my-internal-registry:5000/repository/image:latest
      additionalBuildOptions:
        - "--tls-verify=false"
      additionalPushOptions:
        - "--quiet"
    plugins:
      - name: kafka-connect-file
        artifacts:
          - type: maven
            group: org.apache.kafka
            artifact: connect-file
            version: 4.1.0
```

### Running Buildah in restricted environment

To successfully run Buildah in container on Kubernetes, the Pod needs to have some capabilities.
The required capabilities are `CAP_KILL`, `CAP_SETGID`, and `CAP_SETUID`, which are used during the build phase.
Based on my understanding, the capabilities are used for:
- `CAP_KILL` - used when Buildah needs to terminate helper processes during the container build - especially cleanups or intermediate processes that may not share the same UID.
- `CAP_SETGID` - used for setting group ownership during the container build during `RUN`, `USER` and also in file-extraction steps. Without that Buildah would not be able to reproduce the metadata of files or drop to the right group inside a container.
- `CAP_SETUID` - similar to `CAP_SETGID`, but for configuring user ownership - and during the `USER` step.

In case that user will use restricted Pod Security profile, Buildah will not work - as these capabilities are dropped in this profile.
That is similar to Kaniko - which is not able to run in this restricted profile as well.
Additionally, that is a reason why we will not implement Buildah for OpenShift - and it's described as [one of the rejected alternatives](#only-one-implementation-of-connect-build---buildah-on-both-kubernetes-and-openshift).

## Affected/not affected projects

The only affected project is the `strimzi-kafka-operator` repository, particularly the following classes:
- `KafkaConnectBuild`
- `ConnectBuildOperator`
- `KafkaConnectAssemblyOperator`

These classes contain the logic of the Connect Build feature, thus they are affected the most.
We will need also changes to `DockerOutput` model and to the `KafkaConnect` CRD in order to support two new fields for specifying the additional Buildah options.

## Compatibility

Even though it's a change in terms of the tool that builds the images, it will be gated behind the feature gate, so users can adapt to this change through few releases.
Until the feature gate is GA, we will keep the Kaniko implementation in place, ensuring the backwards compatability.
The `.spec.build.additionalKanikoOptions` field will be deprecated from the start, however it will be usable until the Buildah feature gate is moved to Beta phase or removed when moving to the `v1` API.
After that, the `additionalKanikoOptions` field will be completely ignored and later (with v1 API) it will be removed.
But we will give users enough time to change between `additionalKanikoOptions` and `additionalBuildOptions` before we reach the Beta phase, so it shouldn't be a problem.
Finally, once Buildah feature gate will be promoted to GA, we will remove the implementation of Kaniko, together with the Makefile for tagging the Kaniko executor image.

## Rejected alternatives

### Only one implementation of Connect Build - Buildah on both Kubernetes and OpenShift

When I started with Buildah PoC, I thought that we can have this implementation also for the OpenShift part, removing the implementation of OpenShift Build API and having just one approach once the feature gate is GA.
However, when I tried it on OpenShift, I found out that we would need to manage multiple other resources and grant more roles to the Cluster Operator than we have today.
For OpenShift, the Cluster Operator would have to create two `RoleBinding`s for the Connect build's `ServiceAccount` - one for granting the `image-builder` role (for getting credentials to push to the internal OpenShift registries), and one for granting the `anyuid` security context.
That would mean that Cluster Operator would need another `ClusterRole` that would grant it permissions to `bind` these roles to the `ServiceAccount`.

The `anyuid` SCC is needed for Buildah during the image build.
I don't know the exact internals, but based on my findings, Buildah needs to create an user namespace and also manipulate with UIDs.
The use of the `anyuid` SCC and why it's needed is briefly described in [Buildah's tutorial about running Buildah on OCP](https://github.com/containers/buildah/blob/main/docs/tutorials/05-openshift-rootless-build.md#create-service-account-for-building-images).

I tried to workaround the issues I was experiencing with the `restricted-v2` SecurityContext, but unfortunately without luck.
The same outcome was with `non-root-v2`, only once I applied the `anyuid` SCC, it started to work.

Finally, to have this solution in place, it would require additional privileges also for the cluster admins granting the permissions to the Cluster Operator, which makes it un-usable for some users.
Because of this fact and also potential security risk, we decided to reject this alternative.

### Using fork of Kaniko

As the Kaniko was archived, it was then [forked by Chainguard](https://github.com/chainguard-dev/kaniko).
Currently, it's maintained, but they are not providing any image - which means that we would have to build our own.
It's not a blocker for us, but it would mean some kind of maintenance to be done on our side.
The main reason why I rejected this alternative is, that in their [README](https://github.com/chainguard-dev/kaniko?tab=readme-ov-file#history-and-status) they mention that "If another active and community-supported fork emerges, we'll happily shut this one down and migrate to that".
Which means for us that we would have to migrate to something else possibly in few months again, and maybe doing this proposal process once again.
Having something that is supported, maintained, and used widely is better alternative for us currently.

### Forking and maintaining Kaniko repository on our own

Another alternative was to fork the Kaniko repository and maintain it on our own.
Even if we wouldn't add any new features to it, it's still a maintenance burden - we would have to fix bugs and CVEs that may occur during the time.
Also, we would then have to maintain it for longer time - in case that there will be non-Strimzi (but also Strimzi) users using it in their own builds.
That would mean that even if we for example deprecate and remove the Connect Build feature, we would still have to maintain the Kaniko repository for the other users.
Finally - it's written in Go and based on our previous experience with Go component (Canary), our team doesn't have enough knowledge of Go in order to maintain it fully.

### Removing Connect Build feature completely

Even removing the Connect Build feature completely was a possible alternative.
But based on the discussions on Slack or GitHub, and also issues on GitHub, it's still quite often used by our Strimzi users.
So removing it now when the alternative - Image Volumes - is not used widely and supported by all Kubernetes versions Strimzi supports, is not a good idea.

### Using Image Volumes feature in Kubernetes only

The alternative of using Image Volumes feature is connected to the previous section.
As I already mentioned, the Image Volumes feature is not widespread, and it's not supported by all Kubernetes versions Strimzi supports.
Even if it's possible solution for the future, we need to wait until it's used and supported widely before moving to it.

### Using other tools like BuildKit

During implementing and testing the PoC with Buildah, I tried also different tools like BuildKit.
BuildKit is mainly rootful, but it can be used rootless.
However, in order to build an image inside the container, it needs some hacking and workarounds, which are not that straightforward as in case of Buildah.
In order to make things "easy" and not confusing, I decided to reject this alternative and use Buildah instead.