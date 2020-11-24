# Kafka Connect Build

This proposal improves our Kafka Connect deployment with support for _build_ for adding additional connector plugins to the Connect deployment.

## Current situation

Currently, Strimzi has two Kafka Connect deployments: `KafkaConnect` and `KafkaConnectS2I`. 

`KafkaConnect` currently requires that the user to manually prepare their own container image with any additional connector plugins they want to use.
They have to write their own `Dockerfile` which uses the Strimzi image as a base image and adds the additional connectors.
Then they have to build it and push into some container registry and configure the Kafka Connect deployment to use it.

The `KafkaConnectS2I` is supported only on OKD / OpenShift and uses the S2I build to build the new container image with additional connector plugins.
It is not support on Kubernetes since pure Kubernetes do not have any support for S2I builds.
It makes it a bit easier to prepare the new image, but only slightly.
User has to prepare a local directory with the desired connector plugins and pass it to the S2I build.
The S2I build adds it to the image and uses it automatically in the Connect deployment.
In order to add more connector plugins later, user needs to have the old connectors as well as the new connectors - it is not possible to just add the new connectors.

Unlike other Strimzi components, none of these works in a declarative way.

## Motivation

One of the aims of Strimzi is to make using Kafka on Kubernetes as native as possible.
A big part of that is being able to configure things in a declarative way.
That is currently missing for adding connector plugins to the Connect deployments.

Being able to configure the connector plugins in the `KafkaConnect` CR will make it easier to use for our users since they will not need to build the container images manually.
It will also make it easier to to have connector catalog on the website since we will be able to just share the KafkaConnect CR including the build section to add the connector.

## Proposal

A new section named `build` would be added to the `KafkaConnect` custom resource.
This section will allow users to configure:
* connector plugins
* build output

The connector plugins section will be a list where one or more connectors can be defined.
Each connector consists of a name and list of artifacts which should be downloaded.
The artifacts can be of different types.
For example:
* `jar` for a directly download of Java JARs. 
* `tgz` or `zip` to _download and unpack_ the artifacts. 
The archives will be just unpacked without any additional filtering for specific file types.
* `maven` to download JARs from Maven based on Maven coordinates. 
User will provide `groupId`, `artifactId` and `version` and we will try to download the JAR as well as all its runtime dependencies.
This might be also enhanced to support different Maven repositories etc.
* and possibly others if required

The second part - `output` - will configure how the newly built image will be handled.
It will support two types:
* `docker` to push the image to any Docker compatible container repository.
* `imagestream` to push the image to OpenShift ImageStream (supported only on OpenShift)

Strimzi will not run its own container repository.
So users of the `docker` type build will need to provide credentials to their own Docker compatible container registry.
This does not have to run within the same Kubernetes cluster - it can be also SaaS registry such as Docker Hub or Quay.io.

Following is an example of the Kafka Connect CR:

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  version: 2.6.0
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  build:
    output:
      type: docker
      image: docker.io/user/image:tag
      pushSecret: dockerhub-credentaials
    plugins:
      - name: my-connector
        artifacts:
          - type: jar
            url: https://some.url/artifact.jar
            sha512sum: abc123
          - type: jar
            url: https://some.url/artifact2.jar
            sha512sum: def456
      - name: my-connector2
        # ...
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
  config:
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
```

When the `build` section is specified in the `KafkaConnect` resource, Strimzi will generate `Dockerfile` which will correspond to the configured connectors and their artifacts.
It will build the container image from this `Dockerfile` in a separate pod and push it to the configured output (Docker registry / ImageStream).
It will take the digest of this image and configure the Kafka Connect cluster to use this image.

The build will use two different technologies:
* [Kaniko](https://github.com/GoogleContainerTools/kaniko) will be used on Kubernetes
* OpenShift Builds will be used on OKD / OpenShift 

Kaniko is a daemon-less container builder which does not require any special privileges and runs in user-space.
Kaniko is just a container to which we pass the generated `Dockerfile` and let it build the container and push it to the registry.
It does not require any special installation - neither from the user nor from Strimzi it self. 
We just use the container.

OpenShift Builds are part of the OKD / OpenShift Kubernetes platform and also do not require any special privileges.
So it should work for most users across wide variate of environments.

## Affected/not affected projects

The existing `KafkaConnectS2I` resource is not affected by it.
But it should be deprecated and later removed.

`KafkaMirrorMaker2` is also based on Kafka Connect.
But the build will not be available for `KafkaMirrorMaker2`.

## Compatibility

This proposal is fully backwards-compatible.
When users don't specify the `build` section inside the `KafkaConnect` CR, the build will be never activated and used.
So existing users who do not the new build will not be affected in any way.
They will be also still able to use the old manually built container images without any change.

## Rejected alternatives

There are several ways how to build container images.
In addition to Kaniko and OpenShift Builds, I also looked at Buildah.
But Kaniko and OpenShift Builds worked better (read as: _I didn't got Buildah to work_).

The current interface between Strimzi and the builder is based on a `Dockerfile`.
If needed, any other alternative technology which supports building containers from `Dockerfile` should be relatively easy to plugin as a replacement for the current ones.
