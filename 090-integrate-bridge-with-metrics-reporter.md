# Integrate Bridge with Metrics Reporter

In [SIP-064](https://github.com/strimzi/proposals/blob/main/064-prometheus-metrics-reporter.md) we introduced the [Strimzi Metrics Reporter](https://github.com/strimzi/metrics-reporter).
This is a Kafka plugin that directly exposes metrics in Prometheus format via an HTTP endpoint, and can be used as an alternative to the [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter). 
In the following paragraphs, we will refer to the former as *Reporter*, and the latter as *Exporter*.

## Current situation

The Bridge allows users to enable or disable the metrics endpoint using the `KAFKA_BRIDGE_METRICS_ENABLED` environment variable.
When this variable is set to true, the Bridge creates the Exporter's JmxCollector class with hard coded configuration for Kafka related metrics.
This configuration is an *Exporter* YAML distributed as an embedded resource within the Bridge's JAR file.
In addition to Kafka metrics, the Bridge also exposes JVM and HTTP metrics through Vert.x, that are also stored om JmxCollector's Prometheus registry.

When deploying the Bridge through the Cluster Operator, a similar configuration is available in the KafkaBridge's CRD:

```sh
spec:
  enableMetrics: true
```

This is different from how the metrics endpoint is enabled in all the other components deployed by the Cluster Operator.
The Cluster Operator exposes metrics through the Exporter, which can be configured using the shared `metricsConfig` schema.
This schema has a type property that only allows the `jmxPrometheusExporter` value, and a reference to a ConfigMap containing its configuration.

```sh
metricsConfig:
  type: jmxPrometheusExporter
  valueFrom:
    configMapKeyRef:
      name: my-metrics
      key: metrics-config.yml
```

At runtime, the above configuration enables a Java agent that exposes Kafka JMX metrics in Prometheus format through an HTTP endpoint on port 9404.
Note that the *Exporter* requires `org.apache.kafka.common.metrics.JmxReporter`.

## Motivation

We want to support the *Reporter* as an alternative way of configuring metrics across all Cluster Operator components.
When deploying any component through the Cluster Operator, we want to provide a consistent user experience, so the Bridge should also support the `metricsConfig` schema. 

## Proposal

The Bridge will support both *Exporter* and *Reporter* as metrics configuration types.
It will also provide new configurations for the integration with Cluster Operator.

### Metrics configuration types

The Bridge project will be updated to support multiple metrics configurations types.
A new `bridge.metrics` property will be available within the application.properties configuration file, and will only accept `jmxPrometheusExporter` and `strimziMetricsReporter`.
Any other value will raise an error and the application will fail to start with an appropriate error message.

When running in standalone mode with `strimziMetricsReporter`, the user will be able to configure any reporter property using the "kafka." prefix.
The following example will be documented and also provided as a comment in the default application.properties file:

```sh
bridge.metrics=strimziMetricsReporter
kafka.metric.reporters=io.strimzi.kafka.metrics.KafkaPrometheusMetricsReporter
kafka.prometheus.metrics.reporter.listener.enable=false
kafka.prometheus.metrics.reporter.allowlist=.*
```

The MetricsReporter class will be updated to also include a new StrimziCollectorRegistry that will work similarly to the JmxCollectorRegistry.
The StrimziCollectorRegistry will include a reference to PrometheusRegistry.defaultRegistry, which is the same instance used by the *Reporter* to collect metrics.

The kafka_bridge_config_generator.sh script is used to generate the image configuration based on environment variables.
This script will be also updated to include `bridge.metrics` and related configurations (see the following section).

The Bridge will try to load the Exporter's configuration file from the path specified by the `bridge.metrics.jmx.exporter.config.path` property.
If the property is not specified or the file is not found, the Bridge will fall back to the hard coded configuration.
This feature is not strictly required to support the *Reporter*, but will be used by the Cluster Operator.

### Cluster Operator integration

The KafkaBridge's CRD will also support `metricsConfig` with the addition of `strimziMetricsReporter` type.
This is how the Strimzi Reporter configuration will look like:

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

- `STRIMZI_METRICS`: This will contain the `metricsConfig` types to enable (one of `jmxPrometheusExporter` and `strimziMetricsReporter`).
- `KAFKA_BRIDGE_METRICS_JMX_CONFIG`: Used with *Exporter* to pass the configuration file path.
- `KAFKA_BRIDGE_METRICS_SMR_CONFIG`: Used with *Reporter* to pass the plugin configuration.

The *Exporter* configuration file will be stored in a ConfigMap and mounted in the Bridge's container.
The full configuration file passed to the Bridge's container will be `/opt/strimzi/custom-config/metrics-config.yml`.

Metrics will be exposed through the Bridge's HTTP server, so the *Reporter*'s listener will be disabled.
Other configurations will be locked down with the exception of the `prometheus.metrics.reporter.allowlist` property.

## Affected/not affected projects

The affected projects are Cluster Operator and Kafka Bridge.

## Compatibility

### Bridge

The `KAFKA_BRIDGE_METRICS_ENABLED` environment variable will be deprecated in 0.32.0 and removed in 0.34.0.
When set, the user will get a warning suggesting to use the `bridge.metrics` property.
In case they are both set, `bridge.metrics` will take precedence over `KAFKA_BRIDGE_METRICS_ENABLED`.

### Cluster Operator

The `enableMetrics` property in KafkaBridge CRD will be deprecated in 0.46.0 and removed in 0.48.0.
When set, the user will get a warning suggesting to use the `metricsConfig` configuration.
In case they are both set, `metricsConfig` will take precedence over `enableMetrics`.

## Rejected alternatives

Lock down and automate the *Reporter* configurations for the standalone Bridge.
This was rejected because the Bridge allows user to customize Kafka clients and plugins using the application.properties file, which includes commented examples.
