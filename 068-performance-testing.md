# Strimzi Performance Testing

## Motivation for Performance testing in Strimzi

In the world of distributed systems, performance is not merely a feature but a fundamental aspect that dictates the
reliability, scalability, and efficiency of the system.
For Apache Kafka clusters managed by Strimzi, where data throughput, latency, and resource utilization are critical,
ensuring optimal performance is paramount.
As enterprises rely on Kafka for real-time data processing and streaming analytics, the performance of the Strimzi operators
directly influences the operational capabilities and service quality experienced by end-users.

There are a benefits if we introduce performance testing within the Strimzi such as:
1. **Uphold High Performance Standards:**
    - Performance testing ensures that Strimzi not only meets but exceeds the performance standards expected by users for managing Kafka clusters in diverse environments.
2. **Enable Continuous Improvement:**
    - With dedicated performance benchmarks, Strimzi can continually assess and enhance its components, driving innovation while maintaining performance integrity.
3. **Build Confidence Among Users:**
    - By demonstrating a commitment to performance excellence, Strimzi strengthens trust with its user community, reassuring them that their Kafka infrastructures are managed with utmost efficiency.

Moreover, we can integrate these performance tests into our Azure pipelines, enabling automatic checks to identify
if a PR could cause a significant performance degradation, thereby ensuring every change is vetted for efficiency before being merged (future vision).
This integration represents a proactive step towards maintaining high-performance standards and continuous improvement in Strimzi's development process.

There are several considerations for designing such testing, each with its own set of advantages and disadvantages, which are discussed in the subsequent section.

## Implementation Approaches

I suggest a few possible methods for integrating performance testing into Strimzi, each characterized by its unique benefits and drawbacks.
Our decision on which path to follow should align with our specific requirements and goals.

### Approach 1: Integration within the strimzi-kafka-operator STs Module

This method involves incorporating performance testing directly into the existing system test module of our strimzi-kafka-operator repository.
It's a relatively straightforward strategy, leveraging the wealth of methods, resources, and components already available
within the repository, such as ResourceManager, LogCollector, and MetricsCollector.
This integration significantly lowers the complexity and duplication of effort compared to setting up a separate dedicated repository for performance tests.
One of the drawbacks that we would need to add a new classes like PerformanceReporting and tools for generating Markdown documents, among others.
However, this integration might complicate the use of the system test module as a cohesive unit, given that certain
classes would be specifically tailored for performance testing, while others remain focused on system testing.
Despite this, embedding performance checks into our PR process would be more straightforward, providing
a robust mechanism to detect performance degradations introduced by new changes.

#### Note on Release Lifecycle:

Embedding performance tests within the strimzi-kafka-operator's system test module means performance
testing phases are tightly coupled with the operator's release cycle.
This could streamline the release process but might also necessitate additional considerations for test maintenance
and updates aligned with each release.

---

Nevertheless, this approach limits the applicability of the performance tests to the strimzi-kafka-operator, excluding potential integration
with other repositories such as strimzi-kafka-bridge and strimzi-kafka-oauth, leading us to consider a second approach.

### Approach 2: Creating a Dedicated Repository for Performance testing

This method proposes creating a separate, dedicated repository for performance testing across the Strimzi ecosystem,
addressing some challenges of the first approach while introducing new considerations.
A distinct advantage of this strategy is its facilitation of integration with various Strimzi projects,
enhancing the flexibility and reach of performance testing.

However, this approach requires re-implementing foundational components such as ResourceManager, SetupClusterOperator, LogCollector, and MetricsCollector.
While this might seem daunting, leveraging a framework like [Test Frame](https://github.com/skodjob/test-frame) could streamline the process.
Test Frame provides a robust starting point with essential classes that can be easily extended to fit our needs,
such as developing a TopicMetricsCollector by extending the MetricsCollector class.

Despite these benefits, integrating this dedicated repository with the strimzi-kafka-operator poses a more complex challenge.
Achieving seamless integration requires careful planning and coordination, ensuring that performance tests can be effectively
executed and monitored across different components of the Strimzi ecosystem.

#### Note on Release Lifecycle:

A dedicated repository for performance testing introduces a separate release lifecycle for the testing framework itself.
This separation offers flexibility in testing across different versions of Strimzi components but requires careful
synchronization to ensure compatibility and relevance of performance tests to the versions of Strimzi components being tested.
It also allows for independent evolution of the testing framework, accommodating new testing needs and methodologies
without being constrained by the release cycles of the Strimzi components it tests.

---

This approach may appear more effective than the initial one, yet it necessitates the creation of numerous support classes for:

1. Extracting metrics from Kubernetes Pods components
2. Monitoring the entire lifecycle
3. Documenting (i.e., reporting) the outcomes of performance tests
    4. this could all metrics scraped during performance test
    5. and then reporting via table of each parameter monitored during such test

For instance such result from tool could look like this:

| Experiment   | Number of Topics   | Creation Time (s)   | Deletion Time (s)   | Total Test Time (s)   | STRIMZI_MAX_BATCH_SIZE   | MAX_BATCH_LINGER_MS   | Reconciliation Max Duration (s)   | Max Batch Size   | Max Queue Size   | UTO Event Queue Time (s)   | Describe Configs Max Duration (s)   | Create Topics Max Duration (s)   | Reconciliations Max Duration (s)   | Update Status Max Duration (s)   | Max System Load Average 1m Per Core (%)   | Max JVM Memory Used (MBs)   |
|--------------|--------------------|---------------------|---------------------|-----------------------|--------------------------|-----------------------|-----------------------------------|------------------|------------------|----------------------------|-------------------------------------|----------------------------------|------------------------------------|----------------------------------|-------------------------------------------|-----------------------------|
| 1            | 500                | 80.582              | 137.925             | 218.507               | 500                      | 30000                 | 0.620                             | 364.000          | 261.000          | 598.308                    | 0.234549834                         | 0.479047522                      | 0.620329716                        | 0.107349124                      | 17.9%                                     | 181.68                      |
| 2            | 500                | 84.553              | 169.537             | 254.090               | 1000                     | 30000                 | 7.281                             | 385.000          | 324.000          | 3091.712                   | 0.237088658                         | 6.530933468                      | 7.281358907                        | 0.307568988                      | 22.8%                                     | 172.74                      |
| 3            | 500                | 81.317              | 149.702             | 231.020               | 100                      | 30000                 | 0.584                             | 100.000          | 217.000          | 375.047                    | 0.128191095                         | 0.469570205                      | 0.584339784                        | 0.141746915                      | 30.4%                                     | 176.84                      |
| 4            | 500                | 80.479              | 192.831             | 273.310               | 1000                     | 100                   | 0.650                             | 343.000          | 204.000          | 737.818                    | 0.243653084                         | 0.507064354                      | 0.650260969                        | 0.222171153                      | 17.0%                                     | 169.38                      |
| 5            | 500                | 81.132              | 181.297             | 262.429               | 100                      | 100                   | 0.694                             | 100.000          | 284.000          | 533.664                    | 0.096442905                         | 0.498362223                      | 0.693851682                        | 0.190127359                      | 25.0%                                     | 191.59                      |
| 6            | 500                | 81.278              | 139.751             | 221.029               | 100                      | 10                    | 0.464                             | 100.000          | 274.000          | 399.806                    | 0.0966407                           | 0.357316642                      | 0.463936625                        | 0.104226111                      | 14.4%                                     | 190.14                      |
| 7            | 500                | 82.744              | 161.603             | 244.347               | 500                      | 100                   | 0.732                             | 405.000          | 291.000          | 884.769                    | 0.267927291                         | 0.590955106                      | 0.731769998                        | 0.264752859                      | 29.9%                                     | 170.65                      |
| 8            | 500                | 80.222              | 135.259             | 215.481               | 500                      | 10                    | 0.697                             | 384.000          | 226.000          | 628.528                    | 0.227750243                         | 0.564784403                      | 0.697055392                        | 0.141776211                      | 24.9%                                     | 182.10                      |
| 9            | 500                | 81.656              | 159.475             | 241.131               | 1000                     | 10                    | 0.595                             | 345.000          | 272.000          | 710.384                    | 0.221294127                         | 0.462604201                      | 0.595031656                        | 0.112158161                      | 18.8%                                     | 180.02                      |

Consider eliminating points 2 and 3 by utilizing an existing performance tool that encompasses all these functionalities.

### Approach 3: Using known performance tools

[Grafana k6](https://k6.io/) is an open-source performance testing tool designed to assess the load capacity and performance of various systems and infrastructure.
It allows developers and testers to simulate traffic and user behavior towards applications, APIs,
and services to identify potential bottlenecks and performance limits under controlled test environments.
k6 integrates seamlessly with [Grafana](https://grafana.com/), a popular open-source analytics and monitoring solution,
enabling users to visualize test results and performance metrics in real-time through comprehensive dashboards.
This integration facilitates easy analysis and interpretation of data, making it simpler for teams to make informed
decisions based on the performance characteristics of their applications.

One of the key features of k6 is its extensibility through extensions, which allows users to tailor the tool to their specific testing needs.
For instance, the ability to extend k6's functionality to work with specific technologies like Kubernetes through
the xk6-kubernetes extension showcases its flexibility.
Similarly, this extensibility opens up the possibility of developing a dedicated extension for Strimzi, enabling specialized
performance testing for Strimzi-managed Apache Kafka clusters.
This approach would allow for targeted load testing of Kafka operators, providing insights into how Strimzi components
behave under various load conditions, thereby ensuring that Strimzi deployments are optimized for performance and reliability.

The primary challenge is that developing such an extension requires proficiency in Go/Javascript,
which might be less familiar to our community accustomed to Java.

####  Notes on Release Cycle for Using k6:

The release cycle for incorporating k6 into Strimzi testing would bear similarities to the dedicated repository approach,
with both pros and cons.
One advantage is the potential for rapid adoption and integration of new performance testing methodologies,
given k6's established user base and continuous updates.
However, a possible downside is the need to maintain compatibility with Strimzi's evolving infrastructure,
possibly requiring frequent updates to any Strimzi-specific extensions or configurations within k6.

## Conclusion

Wrapping this up, diving into performance testing within Strimzi isn't just about keeping things running smoothly—it's crucial.
Whether we're thinking about embedding tests right into the strimzi-kafka-operator, setting up a whole new repo for the cause,
or getting our hands on tools like Grafana k6, each path has its perks and quirks.
It's about striking the right balance that fits our goals and the ever-evolving Strimzi landscape.
Leveraging Grafana k6 might ask for a bit of a learning curve, but it’s a solid bet for making sure 
