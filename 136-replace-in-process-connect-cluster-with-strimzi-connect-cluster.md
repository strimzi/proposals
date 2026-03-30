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

The `connect-runtime` dependency, in particular, pulls in a large transitive dependency tree.
This has repeatedly caused issues within CVEs (one of them might be https://github.com/strimzi/strimzi-kafka-operator/pull/12494).

Since [Strimzi Proposal #91](091-add-connect-to-test-container.md), the test-container project already provides `StrimziConnectCluster`, a containerized Kafka Connect cluster.
The operator can use this directly instead of maintaining its own in-process implementation.

## Motivation

The main motivation is that it removes unnecessary dependencies (i.e., three Connect libraries are only used by two test classes).
And with that it also reduce overall maintenance for us having both in-memory nad container variant.
Other already mentioned point is that also shrinks the dependency tree and eliminates a recurring source of CVE-related upgrade blockers.

## Proposal

### Changes in test-container

Add a built-in testing connector to the test-container project so consumers can use it without depending on Kafka Connect libraries.

#### StrimziTestingConnector

A source connector (moved from the operators `TestingConnector`) with configurable behavior:

- Configurable startup/shutdown delays (`start.time.ms`, `stop.time.ms`, `task.start.time.ms`, `task.stop.time.ms`)
- Configurable task poll behavior (`task.poll.time.ms`, `task.poll.records`)
- Configurable failures (`fail.on.start`, `task.fail.on.start`)
- Configurable topic and partition count (`topic.name`, `num.partitions`)

This class depends on `connect-api` (with `provided` scope).

#### StrimziTestingConnectorConfig

A constants-only class that exposes the connectors configuration keys and class name **without** depending on the Kafka Connect API:

```java
public final class StrimziTestingConnectorConfig {
    public static final String CONNECTOR_CLASS_NAME = "io.strimzi.test.container.StrimziTestingConnector";
    public static final String FAIL_ON_START = "fail.on.start";
    public static final String TASK_FAIL_ON_START = "task.fail.on.start";
    public static final String START_TIME_MS = "start.time.ms";
    public static final String STOP_TIME_MS = "stop.time.ms";
    // ... 
}
```

This allows consumers (like the operator) to reference connector configuration without adding `connect-api` to their classpath.

#### StrimziConnectCluster builder addition

A new builder method enables the testing connector:

```java
public StrimziConnectClusterBuilder withTestingConnector() { }
```

When enabled, the connector JAR is automatically built and copied into the container at `/opt/kafka/plugins/strimzi-testing-connector/`, and `plugin.path` is configured accordingly.

### Changes in strimzi-kafka-operator

The main changes are removal of `ConnectCluster.java` and `TestingConnector.java`, which would be moved to the test-container.
With that we would remove `org.apache.kafka:connect-runtime`, `org.apache.kafka:connect-file` and `org.apache.kafka:connect-api` from the operator repo.
And eventually we update integration test where we replace in-process with new one (i.e., inside test-container).

## Compatibility

This adds new public API surface to test-container (`StrimziTestingConnector`, `StrimziTestingConnectorConfig`, and `withTestingConnector()` builder method), but it is strictly opt-in.
The testing connector is only included when the user explicitly calls `.withTestingConnector()` on the builder (i.e., the default behavior of `StrimziConnectCluster` remains unchanged).

On the operator side, the changes are test-only and do not affect any public APIs.

## Rejected alternatives

### Keep the in-process ConnectCluster

The current approach works, but it introduces unnecessary dependencies that have already caused issues with CVEs.

### Add connect-api as a compile dependency in test-container

The `StrimziTestingConnector` needs the Connect API to implement the `SourceConnector` interface.
Making it a compile dependency would force all test-container consumers to pull in the Connect API even if they don't use it.
Using `provided` scope keeps it optional, and `StrimziTestingConnectorConfig` gives consumers access to configuration constants without the dependency.
