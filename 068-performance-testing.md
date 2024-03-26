#  Introducing Performance Testing

## Motivation

In the world of distributed systems, performance is one of the most important non-functional requirements, which has an impact on system design, reliability, scalability, and efficiency.
For Apache Kafka clusters managed by Strimzi, where data throughput, latency, and resource utilization are critical, ensuring optimal performance is paramount.
As enterprises rely on Kafka for real-time data processing and streaming analytics, the performance of the Strimzi operators directly influences the operational capabilities and service quality experienced by end-users.

Benefits of performance tests:
1. **Uphold High Performance Standards:**
    - Performance testing consistently meets and surpasses the performance benchmarks set by users for managing Kafka clusters in diverse environments.
2. **Enable Continuous Improvement:**
    - With dedicated performance benchmarks, Strimzi can continually assess and enhance its components, driving innovation while maintaining performance integrity.
3. **Build Confidence Among Users:**
    - By demonstrating a commitment to performance excellence, Strimzi strengthens trust with its user community, reassuring them that their Kafka infrastructures are managed with utmost efficiency. 

## Proposal: Integration within the strimzi-kafka-operator

We propose integrating performance testing directly alongside existing system tests, leveraging the established infrastructure and testing utilities. 
This method ensures that performance and system tests can share resources efficiently, streamlines the development process, facilitates immediate feedback on performance impacts, and reduces overhead.

**Advantages to integration**:
- Streamlines the development process by using existing test infrastructure.
- Facilitates immediate feedback on performance impacts within the same testing framework used for system tests.
- Reduces the overhead of maintaining separate modules for system and performance tests.

The initial PR will cover the Topic Operator (TO) component, but the plan is to extend it and test our other Strimzi components.

### Core parts

The following activities will be core to our implementation:

1. **JUnit5 Usage:** 
   1. Employ JUnit5 for structuring and executing performance tests. This choice allows us to benefit from JUnit5's advanced features such as dynamic tests, parameterized tests, and its powerful extension model, enhancing our test design and flexibility.
2. **Resource Management:** 
   1. Utilize the `ResourceManager` from the system test module to manage the lifecycle of resources.
   2. This approach ensures efficient setup, teardown, and management of Kafka clusters, topics, and other necessary components across both performance and system tests.
3. **Metrics Collection:** 
   1. Base our metrics collection on the system test module's `MetricsCollector`, with plans to possibly extend this functionality in the future from an external library like [Test Frame](https://github.com/skodjob/test-frame). 
   2. Additionally, introduce specialized collectors such as `TopicOperatorMetricsCollector` to gather detailed metrics relevant to the Strimzi Topic Operator, tailoring our data gathering to the nuances of Kafka and Strimzi.
4. **Performance Reporting:** 
   1. Develop a performance test reporting mechanism. This component will be essential for documenting and analyzing the outcomes of performance tests. 
   2. The reporter should be capable of generating detailed summaries and insights, 
   ideally supporting formats conducive to both human reading and automated parsing, such as Markdown or structured JSON.

**Inclusion within the `systemtest` module:** Integrate performance tests to ensure they coexist seamlessly with system tests, potentially categorizing tests to maintain clarity and prevent any confusion between system and performance testing efforts (i.e., using performance profile). 

### CI: Azure Pipelines

We assume Azure machines may not suffice for all performance tests, meaning we anticipate that lower-complexity performance tests, 
not requiring as much memory/CPU, could run on such infrastructure. 
However, for tests needing more memory in the context of performance, we prefer another approach (i.e., using a [Testing farm](https://docs.testing-farm.io/Testing%20Farm/0.1/index.html)) to execute these tests.

### CI: Possible integration with Testing Farm 

Special consideration will be given to the potential integration of performance tests with Testing farm, utilizing higher-capacity machines than those available in Azure, to ensure comprehensive performance evaluation. 
This future integration into our Continuous Integration (CI) pipeline will help detect any significant performance degradations introduced by new changes, safeguarding the efficiency and reliability of Strimzi-managed Kafka clusters.

We could basically automate this in our PRs or by manual trigger as first try:
```bash
/packit test --labels=performance
```
So, as a first step we could add this manual trigger, and if we see that the job is stable and won't cause any race conditions or false alarms we could include it in our PR as a form of performance check.

## Implementation details

As mentioned above we will have a set of test cases, which could be run by specification of the concrete test profile.
Execution will be handled by JUnit5, identical to the approach used in the `systemtests` module.

### Test case example

As an illustrative example, we can do the following:
1. Create a set of possible configurations for the system under test (SUT).
   In this case it's Topic Operator, where we want to see if configuring different batch sizes and batch linger times has a significant impact on the system.
2.  Create a test case instrumented by a specific configuration. In this case, we are looking at a performance test of an operation for a user named Alice.
```java
private static Stream<Arguments> provideConfigurationsForAliceBulkBatchUseCase() {
    return Stream.of(
            Arguments.of("100", "10", false)      // Lower batch size with short linger time for comparison
            Arguments.of("500", "10", false),     // Increased batch size with short linger time
            Arguments.of("1000", "10", false),    // Large batch size with short linger time
            Arguments.of("100", "30000", false),  // Lower batch size with 30 seconds linger time
            Arguments.of("500", "30000", false),  // Increased batch size with 30 seconds linger time
            Arguments.of("1000", "30000", false), // Large batch size with 30 seconds linger time
            Arguments.of("100", "100", false),    // Lower batch size with longer linger time for extended comparison
            Arguments.of("500", "100", false),    // Increased batch size with longer linger time
            Arguments.of("1000", "100", false)    // Large batch size with longer linger time
    );
}
```

```java
/**
 * This test case is designed to simulate a bulk ingestion/batch use case, as might be required by a user like Alice.
 * Alice's scenario involves the need to rapidly create hundreds of topics, perform data ingestion in parallel across all these topics,
 * subsequently consume from them in parallel, and finally delete all the topics. This use case is particularly focused on minimizing
 * the latency associated with the bulk creation and deletion of topics. In such scenarios, Kafka's performance and its ability to handle
 * batching operations efficiently become crucial factors.
 *
 * The test leverages Kafka's batching capabilities for both creation and deletion of topics to aid in this process. By using a larger batch size
 * and a longer linger time, the test aims to optimize the throughput and minimize the operational latency.
 *
 * This test scenario is implemented to help users in understanding the performance characteristics of Kafka when dealing with
 * high volumes of topics and to assist in configuring their systems for optimal performance during bulk operations.
 */
@ParameterizedTest
@MethodSource("provideConfigurationsForAliceBulkBatchUseCase")
public void testAliceBulkBatchUseCase(String maxBatchSize, String maxBatchLingerMs, boolean withClientsEnabled) throws IOException {
    final int numberOfTopics = 500; // Number of topics to test
    // other variables prohibited for brevity
    // ...
        
    try {
        // 1a. setup phase: creation of SUT i.e., Kafka and Topic Operator in this case
        // ...        
        resourceManager.createResourceWithWait(
                KafkaTemplates.kafkaMetricsConfigMap(testStorage.getNamespaceName(), testStorage.getClusterName()),
                KafkaTemplates.kafkaWithMetrics(testStorage.getNamespaceName(), testStorage.getClusterName(), brokerReplicas, controllerReplicas)
                .editSpec()
                    .editEntityOperator()
                        .editTopicOperator()
                            .withReconciliationIntervalSeconds(10)
                        .endTopicOperator()
                        .editOrNewTemplate()
                            .editOrNewTopicOperatorContainer()
                            // Finalizers ensure orderly and controlled deletion of KafkaTopic resources.
                            // In this case we would delete them automatically via ResourceManager
                                .addNewEnv()
                                    .withName("STRIMZI_USE_FINALIZERS")
                                    .withValue("false")
                                .endEnv()
                                .addNewEnv()
                                    .withName("STRIMZI_ENABLE_ADDITIONAL_METRICS")
                                    .withValue("true")
                                .endEnv()
                                .addNewEnv()
                                    .withName("STRIMZI_MAX_QUEUE_SIZE")
                                    .withValue(String.valueOf(Integer.MAX_VALUE))
                                .endEnv()
                                .addNewEnv()
                                    .withName("STRIMZI_MAX_BATCH_SIZE")
                                    .withValue(maxBatchSize)
                                .endEnv()
                                .addNewEnv()
                                    .withName("MAX_BATCH_LINGER_MS")
                                    .withValue(maxBatchLingerMs)
                                .endEnv()
                            .endTopicOperatorContainer()
                        .endTemplate()
                    .endEntityOperator()
                .endSpec()
                .build(),
                ScraperTemplates.scraperPod(testStorage.getNamespaceName(), testStorage.getScraperName()).build());

        // 1b. setup phase: Create a Thread for scraping metrics
        this.topicOperatorCollector = new TopicOperatorMetricsCollector.Builder()
                .withScraperPodName(this.testStorage.getScraperPodName())
                .withNamespaceName(this.testStorage.getNamespaceName())
                .withComponentType(ComponentType.TopicOperator)
                .withComponentName(this.testStorage.getClusterName())
                .build();

        this.topicOperatorPollingThread = new TopicOperatorPollingThread(this.topicOperatorCollector, "strimzi.io/cluster=" + this.testStorage.getClusterName());
        this.daemonThread = new Thread(this.topicOperatorPollingThread);
        this.daemonThread.setDaemon(true); // Set as daemon so it doesn't prevent JVM shutdown
        this.daemonThread.start();

        // 2a. Exercise: Creation of KafkaTopics
        startTimeNs = System.nanoTime();

        // Create topics
        KafkaTopicScalabilityUtils.createTopicsViaK8s(testStorage.getNamespaceName(), testStorage.getClusterName(), topicNamePrefix,
                numberOfTopics, 12, 3, 1);

        KafkaTopicScalabilityUtils.waitForTopicsReady(testStorage.getNamespaceName(), topicNamePrefix, numberOfTopics);

        endTime = System.nanoTime();
        createTopicsTimeNs = endTimeNs - startTimeNs;
        LOGGER.info("Time taken to create {} topics: {} ns", numberOfTopics, createTopicsTimeNs);

        endTimeWholeNs = System.nanoTime();
        totalTimeWholeNs = endTimeWholeNs - startTimeNs;

        LOGGER.info("Time taken to create {} topics and send and recv: {} messages: {} ns", numberOfTopics, NUMBER_OF_MESSAGES, totalTimeWholeNs);
        
        // Start measuring time for deletion of all topics
        long deletionStartTimeNs = System.nanoTime();

        // 2b. Exercise: Deletion of KafkaTopics
        ResourceManager.getInstance().deleteResourcesOfType(KafkaTopic.RESOURCE_KIND);

        long deletionEndTimeNs = System.nanoTime();
        totalDeletionTimeNs = deletionEndTimeNs - deletionStartTimeNs;
        LOGGER.info("Time taken to delete {} topics: {} ns", numberOfTopics, totalDeletionTimeNs);

        endTimeWholeNs = System.nanoTime();
        totalTimeWholeNs = endTimeWholeNs - startTimeNs;

        LOGGER.info("Total time taken to create {} topics, send and receive {} messages, and delete topics: {} ns", numberOfTopics, NUMBER_OF_MESSAGES, totalTimeWholeNs);
    } finally {
        // 4a. teardown: Gracefully stop the topic-operator thread
        // 4b: teardown: Now, it's safe to log performance data as the collection thread has been stopped
        PerformanceReporter.logPerformanceData(
                numberOfTopics, numberOfClientInstances, NUMBER_OF_MESSAGES, createTopicsTimeNs, totalSendAndRecvTimeNs, 
                totalDeletionTimeNs, totalTimeWholeNs, this.testStorage, ACTUAL_TIME, Environment.PERFORMANCE_DIR, 
                maxBatchSize, maxBatchLingerMs, this.topicOperatorPollingThread.getMetricsHistory()
        );
    }
}
```

Note that this is just an one possible example. There is no need to use @ParametrizedTest.
We could also use simple @Test to edge performance cases.

### Gathered metrics

During test execution we need to scrape metrics from the `SUT` pods.
Therefore, one way how to tackle such a problem is that we could use a `Thread`, which would run an infinite loop, where we would scrape all related metrics. For example:
```java
@Override
public void run() {
    LOGGER.info("Thread started with selector: {}", this.selector);

    while (!Thread.currentThread().isInterrupted()) {
        this.topicOperatorMetricsCollector.collectMetricsFromPods();
        // record specific time when metrics were collected
        Long timeWhenMetricsWereCollected = System.nanoTime();

        Map<String, List<Double>> currentMetrics = new HashMap<>();

        // metrics
        currentMetrics.put("strimzi_alter_configs_duration_seconds_sum", this.topicOperatorMetricsCollector.getAlterConfigsDurationSecondsSum(this.selector));
        // derived Metrics
        // the total time spent on the UTO's event queue. -> (reconciliations_duration - (internal_op0_duration + internal_op1_duration,...)).
        currentMetrics.put("strimzi_total_time_spend_on_uto_event_queue_duration_seconds",
                this.calculateTotalTimeSpendOnUtoEventQueueDurationSeconds(this.selector));

        try {
            // Sleep for a predefined interval before polling again
             TimeUnit.SECONDS.sleep(5); // Poll every 5 seconds, adjust as needed
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // restore interrupted status
            LOGGER.error("Thread was interrupted", e);
        } catch (Exception e) {
            LOGGER.error("Exception in thread", e);
        } finally {
            LOGGER.info("Thread finishing");
        }
    }
}
```

And in the test case we would simply interrupt the `Thread` to finish the scraping.
Also, we could make an interval for scraping such metrics configurable via an environment variable or system property.

Moreover, we propose that all metrics would be gathered in the `systemtest/target/performance` directory, with the same pattern as with the `logs` directory.
For instance, the performance directory could be structured as follows:
```bash
performance/
└── 2024-03-16-13-46-48
    └── aliceBulkBatchUseCase
        └── max-batch-size-100-max-linger-time-10
            ├── jvm_gc_memory_allocated_bytes_total.txt
            ├── jvm_gc_pause_seconds_max{action="end of minor GC",cause="G1 Evacuation Pause",gc="G1 Young Generation",}.txt
            ...
            ├── strimzi_add_finalizer_duration_seconds_max.txt
            ├── strimzi_add_finalizer_duration_seconds_sum.txt
            ...
            ├── strimzi_update_status_duration_seconds_sum.txt
            ├── system_cpu_count.txt
            ├── system_cpu_usage.txt
            ├── system_load_average_1m.txt
            ├── system_load_average_per_core_percent.txt
            └── test-performance-metrics.txt
... # more dates
```
where each of these files contains values scraped by `Thread` in the test. 
This is content of the `system_load_average_1m.txt` file:
```bash
Timestamp: 2024-03-16-13-51-13-997, Values: [3.87]
... # other records skipped for brevity
Timestamp: 2024-03-16-13-57-59-797, Values: [6.12]
Timestamp: 2024-03-16-13-58-05-067, Values: [6.19]
```

And these values/data could be used in the next phase (i.e., analyzing and reporting).

### Analyze and report 

We propose a `Reporter.class` that is responsible for retrieving metrics from `<module-name>/target/performance` directory.
Reporter will generate a table of all related metrics to that specific test case.
For instance, for the Topic Operator performance test case such a report could look like this:

| Experiment | Number of Topics | Creation Time (s) | Deletion Time (s) | Total Test Time (s) | STRIMZI_MAX_BATCH_SIZE | MAX_BATCH_LINGER_MS | Reconciliation Max Duration (s) | Max Batch Size | Max Queue Size | UTO Event Queue Time (s) | Describe Configs Max Duration (s) | Create Topics Max Duration (s) | Reconciliations Max Duration (s) | Update Status Max Duration (s) | Max System Load Average 1m Per Core (%) | Max JVM Memory Used (MBs) |
|------------|------------------|-------------------|-------------------|---------------------|------------------------|---------------------|---------------------------------|----------------|----------------|--------------------------|-----------------------------------|--------------------------------|----------------------------------|--------------------------------|-----------------------------------------|---------------------------|
| 1          | 500              | 80.582            | 137.925           | 218.507             | 500                    | 30000               | 0.620                           | 364.000        | 261.000        | 598.308                  | 0.234549834                       | 0.479047522                    | 0.620329716                      | 0.107349124                    | 17.9%                                   | 181.68                    |
| 2          | 500              | 84.553            | 169.537           | 254.090             | 1000                   | 30000               | 7.281                           | 385.000        | 324.000        | 3091.712                 | 0.237088658                       | 6.530933468                    | 7.281358907                      | 0.307568988                    | 22.8%                                   | 172.74                    |
| ...        | ...              | ...               | ...               | ...                 | ...                    | ...                 | ...                             | ...            | ...            | ...                      | ...                               | ...                            | ...                              | ...                            | ...                                     | ...                       |

We could use such report as a `post-processing` phase after test execution and all needed metrics were gathered from Pods.

### CI: Testing farm

We copy pattern from the system tests and just adapt specific label to it.
For example for performance tests we could start with adding such profile into our `.packit.yaml`.
Packit is a CI/CD service that helps with packaging software as well as with testing and deployment of packages by utilizing specifications directly from project repositories as described in their [documentation](https://packit.dev/docs/configuration/upstream/tests).
```bash
  ... # other profiles 
  ...
  - job: tests
    trigger: pull_request
    identifier: "performance"
    targets:
      - centos-stream-9-x86_64
      - centos-stream-9-aarch64
    skip_build: true
    manual_trigger: true
    env: { IP_FAMILY: ipv4 }
    labels:
      - performance
    tf_extra_params:
      test:
        tmt:
          name: "performance"
  ...
  ...
  ...
```
Moreover, we would create a tag (i.e., `performance`) where all test cases would use it, and therefore we would be able to e trigger such job using testing farm via:
```bash
/packit test --labels=performance
```

## Rejected Alternatives:

1. **Creating a performance module within strimzi-kafka-operator for exclusive performance testing**: 
Initially considered, this strategy involved establishing a separate performance module within the Strimzi project to house all performance-related tests and tools. 
Despite its potential for clean separation of performance tests from system tests and focused development on performance capabilities, this approach was ultimately not chosen. 
The decision favored a more integrated method, using the existing `systemtest` module to leverage established resources and simplify the testing process. 
The advantages of a dedicated performance module, such as organizational clarity and specialized tool integration, were weighed against the benefits of integration within the existing framework.
2. **Creating a Dedicated Repository for Performance Testing**:
While offering flexibility and the potential for broader ecosystem testing, this approach requires significant additional 
development and maintenance efforts. The need to recreate foundational components and ensure their integration with each 
Strimzi project adds complexity and resource demands.
3. **Adoption of external performance tools (e.g., Grafana k6)**:
Despite its powerful features and extensibility, this option introduces a steep learning curve due to the requirement for proficiency in Go/Javascript, 
which may not align with the skill set of the existing Strimzi community. 
Additionally, it necessitates the development and maintenance of Strimzi-specific extensions, complicating the testing process.