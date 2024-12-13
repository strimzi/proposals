# Support running Kafka Connect in test-container

The Strimzi [test-container](https://github.com/strimzi/test-container) library allows running Kafka clusters in containers. This is useful for integration and system tests that require a cluster.

This proposes adding support for running Kafka Connect clusters. 

## Motivation

With [test-container](https://github.com/strimzi/test-container) and [test-clients](https://github.com/strimzi/test-clients), it's possible to build test environments for all Apache Kafka components but Kafka Connect. 

Being able to easily start Kafka Connect clusters would be useful for testing client-side components, such as [metrics-reporter](https://github.com/strimzi/metrics-reporter) and [strimzi-kafka-oauth](https://github.com/strimzi/strimzi-kafka-oauth), with Kafka Connect.

## Proposal

Create one new public class in the `io.strimzi.test.container` package of `test-container` called `StrimziConnectCluster` to represent a Kafka Connect cluster. 

### StrimziConnectCluster API

```java
/**
 * A Kafka Connect cluster using the latest image from quay.io/strimzi/kafka with the given version.
 * Kafka Connect is started in distributed mode. Users must use the exposed REST API to start, stop and manage connectors.
 */
public class StrimziConnectCluster {

    /**
     * Get the workers of this Kafka Connect cluster.
     * @return collection of GenericContainer containers
     */
    public Collection<GenericContainer> getWorkers() { }

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

### StrimziKafkaCluster

At the moment the bootstrap servers returned by `StrimziKafkaCluster.getBootstrapServers()` are meant to be used by applications running the host and they can't be used by other containers.
To address this issue, this also proposes adding a new method to `StrimziKafkaCluster`: 

```java
/**
 * Get the bootstrap servers that containers on the same network should use to connect
 * @return a comma separated list of Kafka bootstrap servers
 */
public String getNetworkBootstrapServers() { }
```


## Affected projects

This proposal affects the Strimzi [test-container](https://github.com/strimzi/test-container) project only. The plan is to use this new feature in [metrics-reporter](https://github.com/strimzi/metrics-reporter).

## Backwards compatibility

There is no impact on backwards compatibility.

## Rejected alternatives

There are currently no rejected alternatives.
