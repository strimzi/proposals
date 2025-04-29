# Using `ImageVolumes` to improve extensibility of Strimzi operands

Strimzi allows users to provide and use custom plugins in multiple different places.
The most visible type of plugins are the Connector plugins in Apache Kafka Connect.
But plugins are supported also in other parts of Strimzi.
For example:

* Authentication and authorization plugins
* Tiered storage plugins
* Principal builders
* Config and topic policies
* Quota plugins
* Partitioners
* (De)serializers and transformations
* Cruise Control goals

When users want to use a custom plugin in one of these places, they typically need to build a new container image and tell Strimzi to use it:

1. Create a new `Dockerfile` that starts `FROM` the Strimzi container image as a base image
2. Add the plugin to the container image in the `Dockerfile`
3. Build new container image from the `Dockerfile` and push it into a container registry
4. Configure Strimzi to use the custom container image either using environment variable in the Strimzi Cluster Operator Deployment or using the `image` field in the custom resource

These steps can be done manually, but in most cases they are deployed as a job in whatever continuous integration platform is used by the user.

For Kafka Connect connectors, transformations and (de)serializers, Strimzi also offers _Kafka Connect Build_.
This feature will automatically build a new container image with the plugins specified in the `KafkaConnect` custom resource (in `.spec.build.plugins`) and push it into the container registry provided by the user.
This container image will be then automatically used by Strimzi in the corresponding Kafka Connect deployment.
This feature cannot be used for any other operands such as Kafka brokers and controllers, Cruise Control, etc.

## Motivation

Both mechanisms offered by Strimzi follow the best practices for using container based infrastructure.
They utilize a container image with the desired plugins and do not change the software at runtime.
This makes it clear what versions of the software are being used and makes it easy to reproduce and analyze any issues.

However, none of these options are very user friendly.
They require the user to have their own container registry (either operated directly by them or an account at one of the SaaS container registries).
This often also requires managing additional registry credentials for pushing and pulling the container images.
With both methods, dealing with CVEs in the base container image is also complicated as the container image needs to be rebuilt to receive any CVE fixes from an updated container image.
When using custom Dockerfile, Strimzi upgrades can get pretty complicated as well, because the custom container image differs for each Strimzi version.
So the container image needs to be updated _at the right time_ during the upgrade process.

This proposal tries to improve the user experience by providing an additional method for adding plugins that will separate the lifecycle of the plugin and of the Strimzi container images.

## Proposal

From version 1.31, Kubernetes added support for _image volumes_.
Image volumes allow mounting a container image (OCI artifact) as a volume in Kubernetes Pods.
The image volumes are protected by the `ImageVolume` feature gate.
This feature requires also support from the container runtime used by the Kubernetes cluster (e.g. CRI-O or Containerd)
As of Kubernetes 1.33, this feature gate moved to beta.

The container image / OCI artifact can contain binaries such as the plugins for Strimzi or Apache Kafka.
And this proposal suggests how Strimzi can use the image volumes to let users provide their plugins.

### Building container images with plugins

The container images with the plugin artifacts can be build using a `Dockerfile`.
As the container image is not executable one its own does not require any operating system features, it can start from `scratch` and just add the desired JARs with the plugin(s).
For example:

```Dockerfile
FROM scratch

COPY my-plugin/target/*.jar /
```

This `Dockerfile` can be built with the usual container tools and pushed into a container / OCI registry.
For example:

```
$ docker build -t <my-registry.io>/<organization>/<image>:<tag> .
$ docker push <my-registry.io>/<organization>/<image>:<tag>
```

Once this container image is pushed into the registry, it can be used as a source for the image volume.
The container image contains only the plugin (and its dependencies).
It does not contain any operating system artifacts or Strimzi JARs.
Its rebuild is therefore needed only when a new version of the plugin is released.

### Kafka Connect plugins (connectors, transformations, etc.)

Kafka Connect plugins continue to be handled in their own separate way even with image volumes through a new section in the `KafkaConnect` custom resource.
This is because they are the most commonly used type of Apache Kafka plugins and deserve the _improved_ user experience.
But it has its own technical reasons as well:
* The connector plugins are stored in a separately defined path defined by the `plugin.path` option in Apache Kafka.
  They are normally not included in the regular class path as most other plugins.
* Having a separate API allows us to mix the existing approaches (Kafka Connect Build or custom container image) with the connectors mounted from image volumes.
  This might be useful for example when some plugins are not (yet) available as container images.

The plugins will be configured in the new section in `.spec.plugins`.
The structure of this section is identical to the existing `.spec.build.plugins` used to define the plugins that should be added through the Kafka Connect Build feature.
But the artifact types will be different.

The `.spec.plugins` section will contain a list of plugins.
Each plugin will consist from one or more artifacts to allow compose the plugin from different libraries if needed.
As of this proposal, the only artifact type will be `image`.
It will let users specify the reference to the container image with the plugin that will be mounted and used.
The image can be specified using a tag or using a digest.
The advantage of the digest is that it exactly specified the container image that will be used.
The advantage of the tag is that you can always use the latest version of the plugin without updating the custom resource.
But it will not be deterministic what is the exact version used.

The following example shows the new YAML of the `KafkaConnect` resource:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  # ...
spec:
  # ...
  plugins:
    - name: echo-sink-connector
      artifacts:
        - type: image
          reference: ghcr.io/scholzj/echo-sink@sha256:7078b3ccbc0d6e76fefb832dd6e4bb6b704f83611428c9d5b060ab0b6c3e8712
          pullPolicy: Always
```

Based on the `KafkaConnect` custom resource, the Strimzi Cluster operator will add the artifacts as image volumes to the Pod definitions.
The volumes will be named using the following schema: `plugin-<pluginName>-<imageReferenceHashstub>`.
This will ensure the volume names are unique.
For example:

```yaml
  # ...
  volumes:
    # ...
    - image:
        pullPolicy: Always
        reference: ghcr.io/scholzj/echo-sink@sha256:7078b3ccbc0d6e76fefb832dd6e4bb6b704f83611428c9d5b060ab0b6c3e8712
  # ...
```

This volume will be afterwards mounted to the Kafka Connect container into a subdirectory structure on the `/opt/kafka/plugins/` path which is used when adding connectors to the custom container image as well.
The one or more containers will be mounted to `/opt/kafka/plugins/<pluginName>/<imageReferenceHashstub>/`.
For example:

```yaml
    # ...
    volumeMounts:
      - mountPath: /opt/kafka/plugins/echo-sink-connector/9e1cec06
        name: plugin-echo-sink-connector-9e1cec06
    # ...
```

When the same plugin name is used in `.spec.build.plugins`, the artifacts from both sections will end up in the same directory tree `/opt/kafka/plugins/<pluginName>/`.

### Other plugins

Mounting other plugins in other components is not as common as Kafka Connect connector plugins.
Most of the other plugins also need to be added to the regular class path of the application rather than to a special `plugin.path`.
Therefore mounting other plugins does not use a special API.
It uses the existing additional volumes instead and adds a new volume type named `image` to it to allow mounting image volumes.
This allows the users to mount the container image into an directory inside the `/mnt` path.

Afterwards, they can use an environment variable to add the JARs from the mounted container image to the class path.
For Kafka-based operands and Cruise Control the environment variable to use is `CLASSPATH`.
For Strimzi operands (User and Topic operators), the environment variable to use is `JAVA_CLASSPATH`.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  # ...
spec:
  kafka:
    # ...
    template:
      kafkaContainer:
        volumeMounts:
          - name: my-authorizer
            mountPath: /mnt/my-authorizer/
        env:
          - name: CLASSPATH
            value: "/mnt/my-authorizer/*"
      pod:
        volumes:
          - name: my-authorizer
            image:
              reference: ghcr.io/scholzj/my-authorizer:latest
              pullPolicy: IfNotPresent
```

#### Tiered storage plugins

Unlike other plugins used in Kafka brokers, the tiered storage plugin is not expected to be in the default class path.
Instead, tiered storage has its own class path configuration in `.spec.tieredStorage.remoteStorageManager.classPath`.
This field should be used instead of the `CLASSPATH` environment variable.
For example:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  # ...
spec:
  kafka:
    # ...
    template:
      kafkaContainer:
        volumeMounts:
          - name: my-tiered-storage-plugin
            mountPath: /mnt/my-tiered-storage-plugin/
      pod:
        volumes:
          - name: my-tiered-storage-plugin
            image:
              reference: ghcr.io/scholzj/my-tiered-storage-plugin:latest
              pullPolicy: IfNotPresent
    tieredStorage:
      type: custom
      remoteStorageManager:
        # ...
        classPath: /mnt/my-tiered-storage-plugin/*
        config:
          # ...
```

### Immutability

When using image volumes to provide Apache Kafka plugins, we do not rely anymore that all the software is provided from a single immutable container image provided by Strimzi and later modified by the user to add the custom plugin.
Instead we compose the running software from multiple images:
* The Strimzi image with the Strimzi software
* One or more plugin images mounted as volumes

For example, following the example Kafka Connect deployment with the configuration used in the corresponding section above:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  # ...
spec:
  # ...
  plugins:
    - name: echo-sink-connector
      artifacts:
        - type: image
          reference: ghcr.io/scholzj/echo-sink@sha256:7078b3ccbc0d6e76fefb832dd6e4bb6b704f83611428c9d5b060ab0b6c3e8712
          pullPolicy: Always
```

The software in this case will be defined as the following set of container images:
* `quay.io/scholzj/kafka@sha256:fd0a060b512240c6dc45e90de26fd738fbbcd470dbb1a3c4c8ec808392db6159` as the _Strimzi_ container image from my prototype build
* `ghcr.io/scholzj/echo-sink@sha256:7078b3ccbc0d6e76fefb832dd6e4bb6b704f83611428c9d5b060ab0b6c3e8712` as the Kafka Connect connector image

Each of these are immutable container images and clearly define the software being run in a given operand.

### Error handling

Strimzi cannot easily detect if the `ImageVolume` feature gate is enabled and if image volumes are supported in given Kubernetes cluster.
While we will use the documentation to warn users that this feature can be used only in Kubernetes cluster where it is supported, it might always happen that users try to use it even when not supported by Kubernetes.
In that case, Kubernetes will ignore the image volume and its volume mount and create the Pod without it (tested with Minikube on Kubernetes 1.31 and 1.32, and with OpenShift 4.18).
As a result, the plugin will be missing and the user has to choose another way how to add it.

In case the image volumes are enabled in Kubernetes but not supported by the container runtime, the container will end up in the `CreateContainerError` state with the following message:

```yaml
  status:
    state:
      waiting:
        message: 'Error response from daemon: invalid volume specification: '':/mnt/echo-sink-plugin/:ro'''
        reason: CreateContainerError
```

In such case, the user will need to edit the related Strimzi custom resource, remove the parts using the image volumes and apply the changes.
Afterwards, the Strimzi operator will recover and recreate the Pod with a valid configuration.

Use of a feature gate to prevent the use of image volumes on unsupported cluster was considered.
But since it is expected that we will be seeing Kubernetes clusters which do not support image volumes for a very long time, the feature gate which would be for a long time in alpha phase and would need to be always enabled does not seem to make sense.

### Testing strategy

The functionality from this proposal will be initially tested only using unit tests.
This should be sufficient in terms of test coverage given we only create the right Pod definition and the rest is up to Kubernetes.
Introducing a system test right now would mean that such a test would be executable only on very specific Kubernetes versions with specific configurations and would not add much value.
System tests using image volumes might be added at a later stage once the image volume support reaches more Kubernetes clusters.

## Out of scope

The image volumes have the potential to simplify how we add our own plugins to our Apache Kafka container images and how we structure them.
We might for example have just a simple Strimzi base image and have the operator mount the different components into it as image volumes on demand:
* The supported Kafka versions
* 3rd party libraries (i.e. plugins shipped by Strimzi out of the box)
* Cruise Control and Kafka Exporter

This would make it easier to handle base image CVEs but also CVEs in the different components.
It would also make sure that software that might contain CVEs will not be actually in the container image unless it s actually needed / used.

However, to be able to propose and implement something like this, we first need all Kubernetes clusters supported by Strimzi to also enable and support the ImageVolume feature gate.
And it might take several years until we get there.
Until then, the image volumes can be used only for the _optional_ Strimzi features that have some other alternatives.

## Affected projects

This proposal affects the Strimzi Operators repository.
Code changes are required in the Cluster Operator code and in the CRDs.

## Backwards compatibility

There is no impact on backwards compatibility.
All the pre-existing feature remain unchanged and can continue to be used by the users.

## Rejected alternatives

### Feature gate

As discussed in one of the earlier sections, a feature gate was considered to prevent users from using the image volumes feature while not supported in their Kubernetes cluster.
This idea was rejected, because it might take a ver long time (probably years) until we know for sure that this feature is supported in all Kubernetes versions supported by Strimzi.
And as such, it would be impossible to set any real graduation timeline for this feature gate.

## Additional resources

* Some of the aspects of this proposal are discussed in a recent blog post about [_Java, Pluggability, Kubernetes, and Image Volumes_](https://github.com/scholzj/scholzj/blob/main/blog-posts/java-pluggability-kubernetes-and-image-volumes.md).
  It covers some other ways how plugins might be added to the Strimzi operands that should be consider inferior to the image volumes.
* A [blog post from Gabriele Bartolini](https://www.gabrielebartolini.it/articles/2025/03/the-immutable-future-of-postgresql-extensions-in-kubernetes-with-cloudnativepg/) describes similar approach to the one discussed in this proposal that is adopted by the CloudNativePG operator for the PostgreSQL database.
* Kubernetes documentation on [Image Volumes](https://kubernetes.io/docs/tasks/configure-pod-container/image-volumes/)
