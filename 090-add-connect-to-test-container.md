# Support running Kafka Connect in test-container

The Strimzi [test-container](https://github.com/strimzi/test-container) library allows running Kafka clusters in containers. This is useful for integration and system tests that require a cluster.

This proposes adding support for running Kafka Connect clusters. 

## Motivation

With [test-container](https://github.com/strimzi/test-container) and [test-clients](https://github.com/strimzi/test-clients), it's possible to build test environments for all Apache Kafka components but Kafka Connect. 

Being able to easily start Kafka Connect clusters would be useful for testing client-side components, such as [metrics-reporter](https://github.com/strimzi/metrics-reporter) and [strimzi-kafka-oauth](https://github.com/strimzi/strimzi-kafka-oauth), with Kafka Connect.

## Proposal

Create 2 new classes in the `io.strimzi.test.container` package of `test-container`:
- `StrimziConnectCluster`: This represents a Kafka Connect cluster. This is the main API users will call to create and interact with Kafka Connect clusters.
- `StrimziConnectContainer`: This represents a Kafka Connect worker.

The 2 classes are analogous to the existing `StrimziKafkaCluster` and `StrimziKafkaContainer` classes that represent a Kafka cluster and broker respectively.

### StrimziConnectCluster API

```java
/**
 * A Kafka Connect cluster using the latest image from quay.io/strimzi/kafka with the given version.
 * Kafka Connect is started in distributed mode. Users must use the exposed REST API to start, stop and manage connectors.
 */
public class StrimziConnectCluster {

    /**
     * Get collection of StrimziConnectContainer containers
     * @return collection of StrimziConnectContainer containers
     */
    public Collection<StrimziConnectContainer> getWorkers() { }

    /**
     * Start the Kafka Connect cluster. 
     * This starts all the workers and waits for them to all be healthy and ready to be used.
     */
    public void start() { }

    /**
     * Stop the Kafka Connect cluster.
     */
    public void stop() { }

    /**
     * Return the REST API endpoint of one of the available workers
     * @return the REST API endpoint
     */
    public String getRestEndpoint() { }

    /**
     * Builder class for {@code StrimziConnectCluster}.
     * <p>
     * Use this builder to create instances of {@code StrimziConnectCluster}.
     * You must at least call {@link #withKafkaCluster(StrimziKafkaCluster)}, and 
     * {@link #withGroupId(String)} before calling {@link #build()}.
     * </p>
     */
    public static class StrimziConnectClusterBuilder {

        /**
         * Sets the Kafka cluster the Kafka Connect cluster will use to.
         * @param kafkaCluster the {@link StrimziKafkaCluster} instance
         * @return the current instance of {@code StrimziConnectClusterBuilder} for method chaining
         */
        public StrimziConnectClusterBuilder withKafkaCluster(StrimziKafkaCluster kafkaCluster) { }

        /**
         * Sets the number of Kafka Connect workers in the cluster.
         * If not called, the cluster has a single broker
         *
         * @param workersNum the number of Kafka Connect workers
         * @return the current instance of {@code StrimziConnectClusterBuilder} for method chaining
         */
        public StrimziConnectClusterBuilder withNumberOfWorkers(int workersNum) { }

        /**
         * Adds additional Kafka Connect configuration parameters.
         * These configurations are applied to all workers in the cluster.
         *
         * @param additionalConnectConfiguration a map of additional Kafka Connect configuration options
         * @return the current instance of {@code StrimziConnectClusterBuilder} for method chaining
         */
        public StrimziConnectClusterBuilder withAdditionalConnectConfiguration(Map<String, String> additionalConnectConfiguration) { }

        /**
         * Specifies the Kafka version to be used for the Connect workers in the cluster.
         * If not called, the latest Kafka version available from {@link KafkaVersionService} will be used.
         *
         * @param kafkaVersion the desired Kafka version for the Connect cluster
         * @return the current instance of {@code StrimziConnectClusterBuilder} for method chaining
         */
        public StrimziConnectClusterBuilder withKafkaVersion(String kafkaVersion) { }

        /**
         * Whether to include the FileStream connectors.
         * If not called, the FileStream connectors are enabled.
         *
         * @param includeFileConnectors Use false to not include the FileStream connectors
         * @return the current instance of {@code StrimziConnectClusterBuilder} for method chaining
         */
        public StrimziConnectClusterBuilder withIncludeFileConnectors(boolean includeFileConnectors) { }

        /**
         * Specifies the group.id of the Connect cluster.
         *
         * @param groupId the group id
         * @return the current instance of {@code StrimziConnectClusterBuilder} for method chaining
         */
        public StrimziConnectClusterBuilder withGroupId(String groupId) { }

        /**
         * Builds and returns a {@code StrimziConnectCluster} instance based on the provided configurations.
         *
         * @return a new instance of {@code StrimziConnectCluster}
         */
        public StrimziConnectCluster build() { }
    }
}
```

### StrimziConnectContainer

This classes extends `GenericContainer`, so it will also expose all the usual methods such as `start()`/`stop()` to allow
interacting with a specific worker.

```java
/**
 * A worker in a {@code StrimziConnectCluster} cluster.
 */
public class StrimziConnectContainer extends GenericContainer<StrimziConnectContainer> {

    /**
     * Return the REST API endpoint of this worker
     * @return the REST API endpoint
     */
    public String getRestEndpoint() { }
}
```

## Affected projects

This proposal affects the Strimzi [test-container](https://github.com/strimzi/test-container) project only. The plan is to use this new feature in [metrics-reporter](https://github.com/strimzi/metrics-reporter).

## Backwards compatibility

There is no impact on backwards compatibility.

## Rejected alternatives

There are currently no rejected alternatives.
