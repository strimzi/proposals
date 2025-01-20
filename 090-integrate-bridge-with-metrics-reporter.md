# Integrate Kafka Bridge with Metrics Reporter

In [Strimzi Proposal 064](https://github.com/strimzi/proposals/blob/main/064-prometheus-metrics-reporter.md), we introduced the [Strimzi Metrics Reporter](https://github.com/strimzi/metrics-reporter).
This is a Kafka plugin that directly exposes metrics in Prometheus format via an HTTP endpoint, and can be used as an alternative to the [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter). 

In this proposal, we outline how to integrate the Metrics Reporter with the Kafka Bridge.
Throughout the proposal, we refer to the Strimzi Metrics Reporter as *Reporter* and the Prometheus JMX Exporter as *Exporter*.

## Current situation

The Kafka Bridge allows users to enable or disable the metrics endpoint using the `KAFKA_BRIDGE_METRICS_ENABLED` environment variable.
When this variable is set to `true`, the Bridge creates the `JmxCollector` class of the *Exporter* with hard-coded configuration for Kafka related metrics.
This configuration is an *Exporter* YAML distributed as an embedded resource within the Bridge's JAR file.
In addition to Kafka metrics, the Kafka Bridge also exposes JVM and HTTP metrics through Vert.x, which are also stored in the JmxCollector's Prometheus registry.

When deploying the Bridge through the Cluster Operator, the same configuration is available in the `KafkaBridge` CRD:

```yaml
spec:
  enableMetrics: true
```

This is different from how the metrics endpoint is enabled in all the other components deployed by the Cluster Operator.
The Cluster Operator exposes metrics through the *Exporter*, which can be configured using the shared `metricsConfig` schema.
This schema has a type property that only allows the `jmxPrometheusExporter` value, and a reference to a ConfigMap containing its configuration.

```yaml
metricsConfig:
  type: jmxPrometheusExporter
  valueFrom:
    configMapKeyRef:
      name: my-metrics
      key: metrics-config.yml
```

At runtime, the above configuration enables a Java agent that exposes Kafka JMX metrics in Prometheus format through an HTTP endpoint on port 9404.

Note that the *Exporter* requires `org.apache.kafka.common.metrics.JmxReporter`.
This is enabled by default, but should be explicitly added to the `metric.reporters` Kafka configuration when passing a custom list of metrics reporter implementations. 

## Motivation

We want to support the *Reporter* as an alternative way of configuring metrics across all Cluster Operator components.
When deploying any component through the Cluster Operator, we want to provide a consistent user experience, so the Kafka Bridge should also support the `metricsConfig` schema. 

## Proposal

The Kafka Bridge will support both *Exporter* and *Reporter* as metrics configuration types.
It will also provide new configurations to integrate with the Cluster Operator.

### Metrics configuration types

The Kafka Bridge project will be updated to support two metrics configuration types.
A new `bridge.metrics` property will be available within the `application.properties` configuration file, and will only accept `jmxPrometheusExporter` and `strimziMetricsReporter`.
Any other value will raise an error and the application will fail to start with an appropriate error message.

When running in standalone mode, the user will be able to configure *Reporter* properties using the "kafka." prefix.

```properties
# enable Strimzi Metrics Reporter
bridge.metrics=strimziMetricsReporter
# filter the exposed metrics using a list of regexes
kafka.prometheus.metrics.reporter.allowlist=.*
```

The following default values will be used in case the user does not configure them:

```properties
# set the Kafka metrics reporter implementation
kafka.metric.reporters=io.strimzi.kafka.metrics.KafkaPrometheusMetricsReporter
# disable the Reporter's HTTP listener (metrics exposed through Kafka Bridge listener)
kafka.prometheus.metrics.reporter.listener.enable=false
```

The MetricsReporter class will be updated to also include a new `StrimziCollectorRegistry` that will work similarly to the `JmxCollectorRegistry`.
Creating a collector registry for each metrics type ensure better isolation, and makes it easier to add or remove metrics types.
The `StrimziCollectorRegistry` will include a reference to `PrometheusRegistry.defaultRegistry`, which is the same instance used by the *Reporter* to collect metrics.

The Kafka Bridge will try to load a custom Exporter's configuration file from the path specified by the `bridge.metrics.exporter.config.path` property.
If the property is not specified or the file is not found, the Kafka Bridge will fall back to the hard-coded configuration file.
This feature is not strictly required to support the *Reporter*, but will be used by the Cluster Operator.

### Cluster Operator integration

The `KafkaBridge` CRD will also support `metricsConfig` with the addition of `strimziMetricsReporter` type.
This is how the *Reporter* configuration will look like:

```yaml
spec:
  metricsConfig:
    type: strimziMetricsReporter
    values:
      allowList:
        - ".*"
```

The `KafkaBridgeConfigurationBuilder` will be updated to add metrics configuration when enabled through the `KafkaBridge` resource.

The *Exporter* user provided configuration file will be stored in a ConfigMap and mounted in the Bridge's container.
The file path passed to the Bridge's container using the `bridge.metrics.exporter.config.path` property will be `/opt/strimzi/custom-config/metrics-config.yml`.

Metrics will be exposed through the Kafka Bridge's HTTP server, so the listener of the *Reporter* will be disabled.
Other configurations will be locked down with the exception of the `prometheus.metrics.reporter.allowlist` property.

## Affected/not affected projects

The affected projects are Cluster Operator and Kafka Bridge.

## Compatibility

### Bridge

The `KAFKA_BRIDGE_METRICS_ENABLED` environment variable will be deprecated in the next release, and removed in the first major or minor release of 2026.
When set, the user will get a warning suggesting to use the `bridge.metrics` property.
In case they are both set, `bridge.metrics` will take precedence over `KAFKA_BRIDGE_METRICS_ENABLED`.

#### Migration

Once the `KAFKA_BRIDGE_METRICS_ENABLED` property will be removed, users will have to set `bridge.metrics=jmxPrometheusExporter` in the `application.properties`.

### Cluster Operator

The `enableMetrics` property in `KafkaBridge` CRD will be deprecated in the next release and removed with the release of Strimzi v1 API.
When set, the user will get a warning suggesting to use the `metricsConfig` configuration.
In case they are both set, `metricsConfig` will take precedence over `enableMetrics`.

#### Migration

Once the `enableMetrics` property will be removed, users that want to keep using the *Exporter* will have to create a secret with its configuration and reference from the resource spec:

```yaml
spec:
  metricsConfig:
    type: jmxPrometheusExporter
    valueFrom:
      configMapKeyRef:
        name: my-metrics
        key: my-metrics-config.yml
```

As usual, Strimzi will provide an example YAML file.

## Rejected alternatives

Lock down and automate the *Reporter* configuration in the standalone Kafka Bridge.
This would be convenient, but different from the current approach of using the `application.properties` file with examples.
