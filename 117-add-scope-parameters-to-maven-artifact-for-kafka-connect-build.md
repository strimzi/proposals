# Add scope parameter to Maven artifacts for Kafka Connect build

Kafka Connect build creates a container image using the plugins configured through the `.spec.build.plugins` property of the `KafkaConnect` custom resource.

Plugin artifacts can be specified using Maven coordinates under `spec.build.plugins[].artifacts[]` to identify artifacts and dependencies so that they can be located and fetched during the build process.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster
spec:
  #...
  build:
    output:
      #...
    plugins:
      - name: my-plugin
        artifacts:
          - type: maven
            repository: https://mvnrepository.com
            group: <maven_group>
            artifact: <maven_artifact>
            version: <maven_version_number>
```

This proposal aims to support specifying the scope threshold for Maven artifacts, enabling users to configure which dependencies they would like to fetch and include in the container image.

## Current situation

Currently, no scope threshold is specified when pulling Maven dependencies, which results in downloading all dependencies, including test, provided, system, etc.

This can make the build process longer, container images larger, and potentially lead to version conflicts as discussed in [issue #11799](https://github.com/strimzi/strimzi-kafka-operator/issues/11799).

## Motivation

We should allow users to specify the scope threshold for Maven artifacts to configure which dependencies they would like to include in the container image.

Benefits include:
- Faster build process
- Smaller container images
- Reduced risk of version conflicts
- More control over the build process for users

## Proposal

The proposal is to allow users to specify a scope threshold for copying Maven dependencies during the Kafka Connect build, using a new property `scope` that applies only to Maven artifacts.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster
spec:
  #...
  build:
    output:
      #...
    plugins:
      - name: my-plugin
        artifacts:
          - type: maven
            repository: https://mvnrepository.com
            group: <maven_group>
            artifact: <maven_artifact>
            version: <maven_version_number>
            scope: runtime # new optional property
```

This property will be optional to allow backwards compatibility. 
- when specified - the value will be used for copying dependencies. 
- when not specified - no scope will be used, resulting in all dependencies being pulled, maintaining the current behavior.

The possible set of values for this property would be based on the [Maven Dependency Plugin's includeScope parameter](https://maven.apache.org/plugins/maven-dependency-plugin/copy-dependencies-mojo.html#includeScope):

- `runtime` - Includes runtime and compile dependencies
- `compile` - Includes compile, provided, and system dependencies
- `test` - Includes all dependencies (equivalent to default)
- `provided` - Includes only provided dependencies
- `system` - Includes only system dependencies

### Implementation details

This property will be added to the [MavenArtifact](https://strimzi.io/docs/operators/latest/configuring.html#type-MavenArtifact-reference) schema.

The logic in `cluster-operator/src/main/java/io/strimzi/operator/cluster/model/KafkaConnectDockerfile.java` will be updated to include this parameter `-DincludeScope=<value>` for the `dependency:copy-dependencies` step, if specified in the `KafkaConnect` resource.

## Affected projects

- [strimzi-kafka-operator](http://github.com/strimzi/strimzi-kafka-operator/)

## Compatibility

This property will be optional and when not specified, the current behavior of pulling all dependencies will remain, keeping backward compatibility for users not using the new property.

## Rejected alternatives

Other alternatives considered and the reasons why they were not chosen are as follows:

1. Changing the current implementation without introducing this optional property - This could cause unexpected behavior for current users.
2. Supporting both `includeScope` and `excludeScope` properties to align with names, meaning, and behavior of Maven `copy-dependency` plugin - The additional property might not be needed or used much.
