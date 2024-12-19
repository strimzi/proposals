# Integrate Bridge with Metrics Reporter

In [SIP-064](https://github.com/strimzi/proposals/blob/main/064-prometheus-metrics-reporter.md) we introduced the [Strimzi Metrics Reporter](https://github.com/strimzi/metrics-reporter).
This is a Kafka `MetricsReporter` plugin that directly exposes metrics in Prometheus format via an HTTP endpoint.

## Current situation

The Cluster Operator exposes Kafka metrics through the [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter), which can be configured using the shared `metricsConfig` schema.
This schema has a `type` property that only allows the `jmxPrometheusExporter` value, and a reference to a `ConfigMap` containing its configuration.

```sh
metricsConfig:
  type: jmxPrometheusExporter
  valueFrom:
    configMapKeyRef:
      name: <component>-metrics
      key: metrics-config.yml
```

At runtime, this configuration enables a Java agent that exposes Kafka JMX metrics in Prometheus format through an HTTP endpoint on port 9404.
Note that this agent depends on the [Kafka JMX Reporter](https://github.com/apache/kafka/blob/3.9.0/clients/src/main/java/org/apache/kafka/common/metrics/JmxReporter.java) plugin, which is enabled by default.

The only major component not using the `metricsConfig` schema is the [HTTP Bridge](https://github.com/strimzi/strimzi-kafka-bridge), as it directly instantiates the JMX Exporter's `JmxCollector` class with hard coded configuration.
The `KafkaBridge` CRD does not support a type system for metrics configuration, but a simple boolean configuration to enable/disable metrics.

```sh
spec:
  enableMetrics: true
```

## Motivation

We want to support the Strimzi Metrics Reporter as an alternative way of configuring metrics across all components.
In order to provide a consistent user experience, the HTTP Bridge should align with all the other components in terms of metrics configuration. 

## Proposal

As specified in SIP-064, a new metrics config type called `strimziMetricsReporter` will be added to `metricsConfig` schema along with its configuration.
Only the Strimzi Metrics Reporter's `prometheus.metrics.reporter.allowlist` property will be configurable by users when using the Cluster Operator.
This configuration is a comma separated list of regex patterns that is used by the plugin to filter metrics to collect.
All other plugin configuration properties will be locked down, including the port that will be set to 9404 to avoid conflicts with other ports.

```sh
spec:
  kafka:
    metricsConfig:
      type: strimziMetricsReporter
      values:
        allowList:
          - "kafka_log.*"
          - "kafka_network.*"
```

### HTTP Bridge

The HTTP Bridge project will be updated to support multiple types of metrics configurations.
A new `bridge.metrics` property will be available, and will only accept `jmxPrometheusExporter` and `strimziMetricsReporter`.
Any other value will raise an error and the application will fail to start with an appropriate error message.

When running in standalone mode with `strimziMetricsReporter`, the user will be able to configure any Strimzi Metrics Reporter property using the `kafka.` prefix.
At a minimum `kafka.metric.reporters=io.strimzi.kafka.metrics.KafkaPrometheusMetricsReporter` and `kafka.prometheus.metrics.reporter.listener.enable=false` will be needed.
The following example will be provided as a comment in the default properties file.

```sh
bridge.metrics=strimziMetricsReporter
kafka.metric.reporters=io.strimzi.kafka.metrics.KafkaPrometheusMetricsReporter
kafka.prometheus.metrics.reporter.listener.enable=false
kafka.prometheus.metrics.reporter.allowlist=.*
```

The `KAFKA_BRIDGE_METRICS_ENABLED` env var will be deprecated and removed in a future release.
When set, the user will get a warning suggesting to use the new `bridge.metrics` configuration.
In case they are both set, the `bridge.metrics` configuration will take precedence over `KAFKA_BRIDGE_METRICS_ENABLED`.

The `MetricsReporter` class will be updated to also include a new `StrimziCollectorRegistry` that will work similarly to the `JmxCollectorRegistry`.
The `StrimziCollectorRegistry` will include a reference to `PrometheusRegistry.defaultRegistry`, which is the same instance used by the Strimzi Metrics Reporter to collect metrics.

The `kafka_bridge_config_generator.sh` script is used to generate the image configuration based on environment variables.
This script will be updated to also include `bridge.metrics` and related configurations (see the following section).

The Bridge will also try to load the JMX Exporter configuration file from the path specified by the `bridge.metrics.jmx.exporter.config.path` property.
If not specified or not found, it will fall back to the hard coded configuration.

### Cluster Operator

Like for all the other major components, the `KafkaBridge` CRD will also include `metricsConfig` with support for `strimziMetricsReporter` type.

Three new environment variables will be introduced to pass the metrics configuration in the HTTP Bridge container:

1. `STRIMZI_METRICS`: This will contain the `metricsConfig` types to enable (one of `jmxPrometheusExporter` and `strimziMetricsReporter`).
2. `KAFKA_BRIDGE_METRICS_JMX_CONFIG`: Used with `STRIMZI_METRICS=jmxPrometheusExporter` to pass the configuration file path.
3. `KAFKA_BRIDGE_METRICS_SMR_CONFIG`: Used with `STRIMZI_METRICS=strimziMetricsReporter` to pass the plugin configuration.

The `enableMetrics` property will be deprecated and removed in a future release.
In case they are both set, `metricsConfig` will take precedence over `enableMetrics`.

All Strimzi Metrics Reporter configurations will be locked down with the exception of `allowList`.
Unlike the other major components, the metrics endpoint port will be set to 8080, which is the one already used to expose the REST API.

The JMX Exporter configuration file contained in a `ConfigMap` will be mounted in the HTTP Bridge container under `/opt/strimzi/custom-config/` to be loaded at runtime.

## Affected/not affected projects

The only affected projects are Cluster Operator and Kafka Bridge.

## Compatibility

All changes will be backwards compatible, but there will be some deprecations as detailed above.

## Rejected alternatives

Lock down and automate the Strimzi Metrics Reporter configurations for the standalone HTTP Bridge.
Historically, the HTTP Bridge allows user to customize Kafka clients and plugins using its properties file and providing commented examples.
