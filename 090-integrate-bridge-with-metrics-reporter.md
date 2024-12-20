# Integrate Bridge with Metrics Reporter

In [SIP-064](https://github.com/strimzi/proposals/blob/main/064-prometheus-metrics-reporter.md) we introduced the [Strimzi Metrics Reporter](https://github.com/strimzi/metrics-reporter).
This is a Kafka `MetricsReporter` plugin that directly exposes metrics in Prometheus format via an HTTP endpoint.

## Current situation

The HTTP Bridge allow users to enable or disable the metrics endpoint using the `KAFKA_BRIDGE_METRICS_ENABLED` environment variable.
When this variable is set to true, the Bridge creates the JMX Exporter's `JmxCollector` class with hard coded configuration.
This configuration is a JMX metrics exporter YAML distributed as an embedded resource within the Bridge JAR file.

When deploying the Bridge through the Cluster Operator, a similar configuration is available in the `KafkaBridge` CRD.

```sh
spec:
  enableMetrics: true
```

This is different from how the metrics endpoint is enabled in all the other major components deployed bu the Cluster Operator.
The Cluster Operator exposes metrics through the [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter), which can be configured using the shared `metricsConfig` schema.
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

## Motivation

We want to support the Strimzi Metrics Reporter as an alternative way of configuring metrics across all components.
When deploying configuring major components through the Cluster Operator, we want to provide a consistent user experience, so the HTTP Bridge should also support the `metricsConfig` schema. 

## Proposal

The Bridge will support both JMX Exporter and Strimzi Reporter as metrics configuration types.
It will also provide new configurations for the integration with the Cluster Operator.

### Metrics configuration types

The HTTP Bridge project will be updated to support multiple types of metrics configurations.
A new `bridge.metrics` property will be available within the `application.properties` configuration file, and will only accept `jmxPrometheusExporter` and `strimziMetricsReporter`.
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

### Cluster Operator deploy

Like for all the other major components, the `KafkaBridge` CRD will also include `metricsConfig` with support for `strimziMetricsReporter` type.

This is how the Strimzi Metrics Reporter configuration will look like:

```sh
spec:
  metricsConfig:
    type: strimziMetricsReporter
    values:
      allowList:
        - "kafka_log.*"
        - "kafka_network.*"
```

Three new environment variables will be introduced to pass the metrics configuration to the Bridge's container:

1. `STRIMZI_METRICS`: This will contain the `metricsConfig` types to enable (one of `jmxPrometheusExporter` and `strimziMetricsReporter`).
2. `KAFKA_BRIDGE_METRICS_JMX_CONFIG`: Used with `STRIMZI_METRICS=jmxPrometheusExporter` to pass the configuration file path.
3. `KAFKA_BRIDGE_METRICS_SMR_CONFIG`: Used with `STRIMZI_METRICS=strimziMetricsReporter` to pass the plugin configuration.

The `enableMetrics` property will be deprecated and removed in a future release.
In case they are both set, `metricsConfig` will take precedence over `enableMetrics`.

The JMX Exporter configuration file will be stored in a `ConfigMap` and mounted in the Bridge container under `/opt/strimzi/custom-config`.
The full configuration file passed to the Bridge container will be `/opt/strimzi/custom-config/metrics-config.yml`.

Metrics will be exposed through the Bridge HTTP server, so the reporter's listener will be disabled with `kafka.prometheus.metrics.reporter.listener.enable=false`.
The Strimzi Metrics Reporter configurations will be locked down with the exception of the `allowlist` property.

## Affected/not affected projects

The only affected projects are Cluster Operator and Kafka Bridge.

## Compatibility

All changes will be backwards compatible, but there will be some deprecations as detailed above.

## Rejected alternatives

Lock down and automate the Strimzi Metrics Reporter configurations for the standalone HTTP Bridge.
Historically, the HTTP Bridge allows user to customize Kafka clients and plugins using its properties file and providing commented examples.
