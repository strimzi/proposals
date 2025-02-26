# Split metrics-reporter into client and server modules

## Current situation

The metrics-reporter is currently a single artifact (JAR) that can be used with both Apache Kafka clients and servers. 

## Motivation

Having a single artifact is simple and works well for the basic functionality. We're now trying to add more advanced features and it's proved hard to keep the same logic for both client and server.

For example [support for dynamic configurations](https://github.com/strimzi/metrics-reporter/issues/55) is a feature only available for servers. Adding support for this feature is difficult without impacting the client logic. I opened a PoC [PR](https://github.com/strimzi/metrics-reporter/pull/64), and while I got it to work, this caused a lot of complexity due to `KafkaPrometheusMetricsReporter` needing to support both clients and servers.

Another benefit of having separate modules is the ability to enforce dependencies. Depending on where the reporter runs, a Kafka client or a Kafka server, different dependencies are available at runtime. We need to make sure that logic used by clients does not load any classes only available on servers. We hit this issue in the past ([issue #48](https://github.com/strimzi/metrics-reporter/issues/48)). This is not something that can easily be tested so currently it relies on maintainers being diligent when making code changes to ensure we don't hit this issue again in the future.

I'm also considering adding support for [KIP-714](https://cwiki.apache.org/confluence/display/KAFKA/KIP-714%3A+Client+metrics+and+observability) to metrics-reporter. Again this feature is only implemented by server-side metrics reporters. Adding support for this feature will be addressed in a separate proposal.

## Proposal

I propose splitting the project into 2 Java modules:

1. `client-metrics-reporter`: This module will only depend on kafka-clients and the Prometheus libraries. It will be used by Apache Kafka clients (Producer, Admin, Consumer, Connect and Streams) by setting the `metric.reporters` configuration to `io.strimzi.kafka.metrics.prometheus.ClientMetricsReporter`. All the existing metrics-reporter configurations will stay the same. The differences are:
   - the reporter class name (it used to be `io.strimzi.kafka.metrics.KafkaPrometheusMetricsReporter`)
   - the dependency to add to the classpath
   
   This reporter will use its metric context to validate it runs in a client and will fail at startup if it's not the case.

2. `server-metrics-reporter`: This module will depend on the client-metrics-reporter module and also on Apache Kafka server JARs required to capture Yammer metrics (as described in [Proposal #64](https://github.com/strimzi/proposals/blob/main/064-prometheus-metrics-reporter.md)). It can be used by Apache Kafka servers (brokers and controllers) by setting the `metric.reporters` configuration to `io.strimzi.kafka.metrics.prometheus.ServerKafkaMetricsReporter` and the `kafka.metrics.reporters` configuration to `io.strimzi.kafka.metrics.prometheus.ServerYammerMetricsReporter`. All the existing metrics-reporter configurations will stay the same. The differences are:
   - the reporters class names (it used to be `io.strimzi.kafka.metrics.KafkaPrometheusMetricsReporter` and `io.strimzi.kafka.metrics.YammerPrometheusMetricsReporter`)
   - the dependencies to add to the classpath

The project will publish 2 artifacts, one per module. The build will also produce archives including all the dependencies:
- The client-metrics-reporter archive will contain the client-metrics-reporter JAR and all its dependencies
- The server-metrics-reporter archive will contain the server-metrics-reporter JAR and all its dependencies (including the client-metrics-reporter JAR)
## Affected/not affected projects

In addition of `metrics-reporter`, this will also impact:

- `strimzi-kafka-operator`: There is a PR ongoing to add support for the metrics-reporter. This will need to be updated to use the new class names and dependency.
- `strimzi-kafka-bridge`: The proposal to add support for the metrics-reporter is not impacted, but the implementation will need to be updated to use the new class names and dependency.

## Compatibility

Upgrading from 0.1.0 to a newer release of the metrics-reporter will require manual changes. Users will have to download the right dependencies depending if they are using the reporter with clients or servers and update their client or servers configurations to use the new class names.

Since the project is still in early access, now is the time if we want to make breaking changes. It will be much harder to do so once the reporter is supported by other projects (`strimzi-kafka-operator`, `strimzi-kafka-bridge`) and starts being used by Strimzi users.

## Rejected alternatives

- Keep a single module: It's _possible_ to implement [support for dynamic configurations](https://github.com/strimzi/metrics-reporter/issues/55) and [support for KIP-714](https://github.com/strimzi/metrics-reporter/issues/72) but this causes a lot of complexity.