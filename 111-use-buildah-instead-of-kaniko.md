# Buildah - replacement of Kaniko after its archivation

In our `KafkaConnect` custom resource, we have a possibility to specify plugins that we want to include inside our Connect and build the Connect image, without need of building the image externally.
To support this feature on both Kubernetes and OpenShift, we have two implementations.
For OpenShift we are using the Build API that OpenShift offers.
But on Kubernetes, we are using [Kaniko](https://github.com/GoogleContainerTools/kaniko).
However, the Kaniko project was [archived](https://github.com/GoogleContainerTools/kaniko/commit/236ba5690eda9170d0157aa8137ebbeb09d38685) and it's no longer developed or maintained.
This proposal aims to resolve this issue.

## Motivation

The main motivation behind this proposal is to resolve the issue that Kaniko is archived, and it will not receive any updates (no bug or CVE fixes).
Currently, it seems to not be a problem, but when some critical CVE or bug arises, we have to fix it.
Also, we need a support in case that there will be something blocking us on newer versions of Kubernetes, which we would not get in this case anymore.

## Proposal

For the Connect Build feature on Kubernetes, we will use [Buildah](https://buildah.io/) as a replacement of Kaniko.
Buildah is well-supported, widely used tool for building container images.
Also, it seems that OpenShift uses Buildah in Build API and its process - which makes Buildah a good alternative to Kaniko, with guarantees that it will be supported and maintained further.
Other than that, Buildah doesn't need any workaround in order to run it on Kubernetes rootlessly - the only thing that is needed during the build and push stages is to specify `--storage-driver=vfs` option.
VFS (Virtual File System) driver ensures user-space implementation, it stores each layer as a full copy of the files, but without need of root permissions.
The only trade-off is the build time (it's slower) and consumption of disk space.
But it seems that it is similar to what Kaniko does as well.

We will use the official Buildah images on Quay.io - currently `quay.io/buildah/stable:v1.40.1` - and similarly to Kaniko executor image - we will pull the official image, tag it with the version and push it to the Strimzi repository on Quay.
That's useful because of our versioning and we will have the image ready even if someone on Buildah side decides to remove it from their Quay repository.
Users then can specify their own Buildah image using `STRIMZI_DEFAULT_BUILDAH_IMAGE` (similarly to what is possible today with Kaniko).

The usage of Buildah will be gated behind feature gate called `UseConnectBuildWithBuildah` that will have following schedule:

| Alpha (opt-in) | Beta (default-on) | GA     |
|----------------|-------------------|--------|
| 0.48.0         | 0.51.0            | 0.54.0 |

The build of the Connect image will be done the same way as today, the main difference will be in the commands used in the build and push process.
Kaniko executor did everything at once, which can be done by Buildah as well, but I rejected to have it done in one command because:

- We will not get the SHA of image after the push.
- Once pushed, the image will not be stored locally, so we cannot check the SHA using some different command.

The SHA of the image is needed to keep the compatability with previous implementation using Kaniko.
Kaniko, once the image is build and pushed, is able to return the full name of the image together with the SHA to particular path.
That is done using the `--image-name-with-digest-file` option of the Kaniko executor.
Unfortunately, Buildah doesn't have such option in case that you want to build and push the image in one command (that is done using the `buildah build` command, in order to push directly, you need to prefix the image with `docker://`).
Buildah has option to store the SHA to specify file on path using `--digestfile` option - but only when `push` command is used.
It will just output the SHA, not the full image, so we need to build it from the SHA and built image and output it to `/dev/termination-log` in order to have it inside the message of the completed Pod.

We will use following commands:

```shell
buildah build --file=/dockerfile/Dockerfile --tag=__IMAGE__ --storage-driver=vfs __ADDITIONAL_BUILD_OPTS__
buildah push --storage-driver=vfs --digestfile=/tmp/digest __ADDITIONAL_PUSH_OPTS__ __IMAGE__
buildah images --digests --filter=digest=sha256:$(cat /tmp/digest) --format='{{.Name}}@{{.Digest}}' > /dev/termination-log
```
- `__IMAGE__` is placeholder for user desired name of the image (with registry, repository, and possibly tag)
- `__ADDITIONAL_BUILD_OPTS__` is placeholder for user desired additional build options
- `__ADDITIONAL_PUSH_OPTS__` is placeholder for user desired additional push options

We need to take these three steps in order to get the SHA of the image together with correct repository.
That will work in all cases - even considering cases when user will not specify the tag and will keep the default (`latest`).

We need to add two new fields to `DockerOutput` model in order to support additional options for both build and push options.
Those new fields will be:
- `additionalBuildahBuildOptions` - for additional options to the `build` command. Allowed options will be: `--annotation, --authfile, --cert-dir, --creds, --decryption-key, --env, --label, --logfile, --manifest, --retry-delay, --secret, --security-opt, --timestamp, --tls-verify`
- `additionalBuildahPushOptions` - for additional options to the `push` command. Allowed options will be: `--authfile, --cert-dir, --creds, --format, --quiet, --remove-signatures, --retry, --retry-delay, --sign-by, --tls-verify`

As for the Kaniko additional options, if the user-specified Buildah options will contain forbidden options (or not known), user will be notified by message inside the `.status` section of `KafkaConnect` resource.

In case that Buildah will be used (enabled by the feature gate) and user specifies some Kaniko additional options, they will be ignored and user will be notified with warning inside the `.status` section of `KafkaConnect` CR.

Finally, this feature will be available on Kubernetes only - it will not be available on OpenShift, which is mentioned as one of the rejected alternative.
So the implementation and usage of OpenShift Build API will be kept.

## Affected/not affected projects

The one and only affected project is `strimzi-kafka-operator` repository, especially following classes:
- `KafkaConnectBuild`
- `ConnectBuildOperator`
- `KafkaConnectAssemblyOperator`

These classes contain the logic of the Connect Build feature, thus they are affected the most.
We will need also changes to `DockerOutput` model and to the `KafkaConnect` CRD in order to support two new fields for specifying the additional Buildah options.

## Compatibility

Even though it's change in terms of the tool that builds the images, it will be gated behind the feature gate, so users can adapt to this change through few releases.
Until the feature gate is GA, we will keep the Kaniko implementation in place, ensuring the backwards compatability.
If the Buildah feature gate will be used, the `.spec.build.additionalKanikoOptions` field will be ignored and user will be notified in `.status` section of the `KafkaConnect` CR.
Once the Buildah feature gate will be promoted to Beta, we will deprecate the `.spec.build.additionalKanikoOptions` field.
Finally, after Buildah feature gate will be promoted to GA, we will remove the implementation of Kaniko, together with the Makefile for tagging the Kaniko executor image.

## Rejected alternatives

### Only one implementation of Connect Build - Buildah on both Kubernetes and OpenShift

When I started with Buildah PoC, I thought that we can have this implementation also for the OpenShift part, removing the implementation of OpenShift Build API and having just one approach once the feature gate is GA.
However, when I tried it on OpenShift, I found out that we would need to manage multiple other resources like extra `SecurityContext`, `ClusterRoleBinding`, `RoleBinding`, and more.
It's not a problem to create them, the problem comes in maintainability and testability of such feature.
Now when we use OpenShift Build API, we are sure that is properly tested.
With our own implementation, we would have to properly test it thoroughly, which seems more like a burden than actual benefit of having just one implementation.
And we are unsure if we didn't miss anything.

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
However, in order to build an image inside the container, it needs some hacking and workarounds, which are not that straighforward as in case of Buildah.
In order to make things "easy" and not confusing, I decided to reject this alternative and use Buildah instead.