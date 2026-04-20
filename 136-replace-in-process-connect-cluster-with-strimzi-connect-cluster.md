# Replace in-process ConnectCluster with StrimziConnectCluster

The `strimzi-kafka-operator` integration tests use an in-process `ConnectCluster` class that embeds the Kafka Connect runtime.
This proposal aims to replace it with the containerized `StrimziConnectCluster` from [test-container](https://github.com/strimzi/test-container), removing three Kafka Connect dependencies from the operator.

## Current situation

The `cluster-operator` module has two test-only classes for Kafka Connect integration testing: 
(i.) **`ConnectCluster`**, which starts an in-process Kafka Connect cluster using `org.apache.kafka.connect.cli.ConnectDistributed`;
(ii.) **`TestingConnector`**, a source connector with configurable delays and failures, used by `KafkaConnectApiIT` and `KafkaConnectorIT`.

These classes require three test-scoped dependencies:
- `org.apache.kafka:connect-runtime`
- `org.apache.kafka:connect-file`
- `org.apache.kafka:connect-api`

The `connect-runtime` dependency, in particular, pulls in a large transitive dependency tree that includes libraries such as Jetty.
These transitive dependencies can conflict with the versions used by the operator itself, blocking CVE fixes.
For example, [updating Jetty to address CVEs](https://github.com/strimzi/strimzi-kafka-operator/pull/12494) was blocked because the operator Jetty upgrade clashed with the version pulled in by `connect-runtime`, and there was no simple override to resolve it.

Since [Strimzi Proposal #91](091-add-connect-to-test-container.md), the test-container project already provides `StrimziConnectCluster`, a containerized Kafka Connect cluster.
The operator can use this directly instead of maintaining its own in-process implementation.

## Motivation

The main motivation is that it removes unnecessary dependencies (i.e., three Connect libraries are only used by two test classes).
And with that it also reduce overall maintenance for us having both in-memory nad container variant.
Other already mentioned point is that also shrinks the dependency tree and eliminates a recurring source of CVE-related upgrade blockers.

## Proposal

### New repository: strimzi/test-connectors

Create a new repository under the Strimzi organization [`strimzi/test-connectors`](https://github.com/strimzi/test-connectors) to host Kafka Connect connectors used for testing purposes.
This repository:

- Owns an independent release lifecycle
- Produces connector JARs as release artifacts
- Can host multiple test connectors in the future (e.g., source and sink connectors with different testing behaviors)

#### StrimziTestSourceConnector

The initial connector is a source connector (moved from the operator's `TestingConnector`) with configurable fault-injection behavior:

- Configurable startup/shutdown delays (`start.time.ms`, `stop.time.ms`, `task.start.time.ms`, `task.stop.time.ms`)
- Configurable task poll behavior (`task.poll.time.ms`, `task.poll.records`)
- Configurable failures (`fail.on.start`, `task.fail.on.start`)
- Configurable topic and partition count (`topic.name`, `num.partitions`)

This class depends on `org.apache.kafka:connect-api`.

#### StrimziTestSourceConnectorConfig

A constants-only class that exposes the connector's configuration keys and class name **without** depending on the Kafka Connect API:

```java
public final class StrimziTestSourceConnectorConfig {
    public static final String CONNECTOR_CLASS_NAME = "io.strimzi.test.connectors.StrimziTestSourceConnector";
    public static final String FAIL_ON_START = "fail.on.start";
    public static final String TASK_FAIL_ON_START = "task.fail.on.start";
    public static final String START_TIME_MS = "start.time.ms";
    public static final String STOP_TIME_MS = "stop.time.ms";
    // ...
}
```

This allows consumers (like the operator) to reference connector configuration without adding `connect-api` to their classpath.

### Changes in test-container-images

The built connector JARs from `strimzi/test-connectors` are bundled into the Kafka Connect test container images at build time.
The JAR is placed at `/opt/kafka/plugins/<connector-name>/` inside the image.

### Changes in test-container

#### StrimziConnectCluster builder addition

Two new builder methods enable testing connectors:

```java
public StrimziConnectClusterBuilder withTestConnector() { }
public StrimziConnectClusterBuilder withTestConnector(String name) { }
```

The no-arg overload defaults to the `StrimziTestSourceConnector`.
The `String name` overload allows selecting a specific connector by name, which supports adding more connectors to `strimzi/test-connectors` in the future (e.g., a sink connector) without changing the builder API.

When enabled, the `plugin.path` is configured to include `/opt/kafka/plugins/<connector-name>/`, where the pre-built connector JAR is already present in the image.

### Changes in strimzi-kafka-operator

These changes happen after the test-connectors repo, test-container-images, and test-container changes are in place:

1. Remove `ConnectCluster.java` and `TestingConnector.java` from the operator.
2. Remove `org.apache.kafka:connect-runtime`, `org.apache.kafka:connect-file`, and `org.apache.kafka:connect-api` dependencies.
3. Add a dependency on `strimzi/test-connectors` (just for `StrimziTestSourceConnectorConfig` and there will be no `connect-api` transitive dependency).
4. Update `KafkaConnectApiIT` and `KafkaConnectorIT` to use `StrimziConnectCluster` with `.withTestConnector()`.

## Compatibility

This adds new public API surface to test-container (the `withTestConnector()` builder method), but it is strictly opt-in.
The testing connector is only included when the user explicitly calls `.withTestConnector()` on the builder (i.e., the default behavior of `StrimziConnectCluster` remains unchanged).

On the operator side, the changes are test-only and do not affect any public APIs.

## Rejected alternatives

### Keep the in-process ConnectCluster

The current approach works, but it introduces unnecessary dependencies that have already caused issues with CVEs.

### Build the connector JAR at runtime

An approach where the connector classes are extracted and assembled into a JAR at container startup (inside `containerIsStarting`) was prototyped.
While functional, this requires hardcoding class file paths or implementing dynamic class matching rules, making it fragile and harder to maintain.
Pre-building the JAR as part of a proper release lifecycle and bundling it into the image is cleaner.

### Adopt the echo-sink connector

The existing [echo-sink connector](https://github.com/scholzj/echo-sink) could be moved under the Strimzi organization.
However, it would need delay and failure simulation features added to match the current `TestingConnector` capabilities.

### Embed connector code in test-container

Placing the connector source code directly in the test-container project would require building the connector JAR at runtime, since test-container does not produce container images itself.
It is cleaner to keep the connector in a separate repository and bundle the pre-built JAR during the test-container-images build process instead.

### Add connect-api as a compile dependency in test-container

Making `connect-api` a compile dependency would force all test-container consumers to pull in the Connect API even if they don't use it.
A separate repository avoids this entirely.