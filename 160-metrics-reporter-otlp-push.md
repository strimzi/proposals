# OTLP push support for the Strimzi Metrics Reporter

This proposal adds support for pushing metrics over the [OpenTelemetry Protocol (OTLP)](https://opentelemetry.io/docs/specs/otlp/) to the [Strimzi Metrics Reporter](https://github.com/strimzi/metrics-reporter).
It also describes how the feature is exposed by the Strimzi Cluster Operator and the HTTP Bridge.
It addresses [strimzi/metrics-reporter#104](https://github.com/strimzi/metrics-reporter/issues/104).

Throughout the proposal, we refer to the Strimzi Metrics Reporter as *Reporter* and to the Prometheus JMX Exporter as *Exporter*.

## Current situation

All metrics integrations in Strimzi are pull based.
Both supported `metricsConfig` types, `jmxPrometheusExporter` and `strimziMetricsReporter`, expose an HTTP endpoint in the Prometheus format on port 9404.
The HTTP Bridge exposes its metrics on the `/metrics` path of its own HTTP server instead ([Proposal 092](./092-integrate-bridge-with-metrics-reporter.md)).
Strimzi provides `PodMonitor` examples so that Prometheus can scrape these endpoints.

Users who want to send Strimzi metrics to an OTLP backend currently have two options:
- Deploy an OpenTelemetry Collector next to each cluster, scrape the Prometheus endpoints with its `prometheus` receiver, and forward the metrics over OTLP.
- Configure the OpenTelemetry mode of the *Exporter* through its YAML configuration file, and keep using JMX.

Distributed tracing is configured separately through the `tracing` property with `type: opentelemetry` in `KafkaConnect`, `KafkaMirrorMaker2` and `KafkaBridge`.
Users configure the tracing exporter through the standard `OTEL_*` environment variables, for example `OTEL_SERVICE_NAME` and `OTEL_EXPORTER_OTLP_ENDPOINT`, set in the container template.

## Motivation

OTLP is becoming the vendor-neutral standard for transporting telemetry.
Most observability backends, including Prometheus itself, can ingest OTLP metrics, and some managed backends only accept pushed data.

Scraping the pods with a collector to convert the metrics to OTLP has several drawbacks:
- Every cluster needs an additional workload with its own configuration and RBAC resources.
- The scraping has to be coordinated, otherwise every pod is scraped by each collector replica and the metrics are duplicated.
  Running a single replica avoids the duplication, but it is a single point of failure and limits scaling.
- Many backends need the identity of the source pod, such as `k8s.pod.name` and `k8s.namespace.name`.
  When scraping, this identity has to be reconstructed through relabeling or the `k8sattributes` processor.
  This is not always possible, for example when a proxy or a service mesh hides the source pod IP address.

When each pod pushes its own metrics, the pipeline scales with the cluster and each pod identifies itself.

As described in the issue, supporting push in the *Reporter* also keeps it future-proof and gives users another reason to move from the *Exporter* to the *Reporter*.
Because the *Reporter* can also be used by any Kafka client application, the feature is useful outside of Strimzi deployments as well.

## Proposal

The proposal follows three principles:
- OTLP push is an additional output of the existing *Reporter*, not a new reporter.
  The registry, the collectors, the metric names and the allow list are shared by both outputs.
- The existing API, configuration names and conventions are kept wherever possible.
  OTLP push is configured as part of the existing `strimziMetricsReporter` type in the custom resources.
- Concepts that only make sense for pull are not forced onto push.
  Where push needs something new, such as the identity of the source pod, the proposal follows OpenTelemetry conventions instead of inventing Strimzi-specific ones.

The changes are split between the *Reporter*, the Cluster Operator and the HTTP Bridge.

### Strimzi Metrics Reporter

#### OTLP output

The *Reporter* gains an optional OTLP output next to the existing HTTP listener.
Both outputs are independent, so the *Reporter* can expose the Prometheus endpoint, push over OTLP, or do both at the same time.
Running both outputs at the same time is useful when migrating from one monitoring pipeline to the other.

No new reporter classes are added.
Separate classes would duplicate the collectors and the registry singletons, and `ServerYammerMetricsReporter` does not support creating a second instance.
Keeping a single set of classes also means that the allow list remains dynamically reconfigurable on brokers, and applies to both outputs.

The push is implemented with the `OpenTelemetryExporter` from the Prometheus Java client (`io.prometheus:prometheus-metrics-exporter-opentelemetry`).
It exports the same `PrometheusRegistry` that the *Reporter* already populates, so no additional metrics conversion code is needed in the *Reporter*.
The exporter must use the same version as the Prometheus client used by the *Reporter*.
It bundles a shaded copy of the OpenTelemetry SDK, so it cannot conflict with other OpenTelemetry libraries on the classpath, such as the ones used for tracing or by connector plugins.

#### Configuration

The following configurations are added to the *Reporter*:

| Name | Type | Default | Description |
|:-----|:-----|:--------|:------------|
| `prometheus.metrics.reporter.otlp.enable` | Boolean | `false` | Whether to push metrics to an OTLP endpoint. |
| `prometheus.metrics.reporter.otlp.endpoint` | String | Not set | The OTLP endpoint in the `http://<HOST>:<PORT>` or `https://<HOST>:<PORT>` format. Like `OTEL_EXPORTER_OTLP_ENDPOINT`, it is a base URL, so the `/v1/metrics` path is appended with the `http/protobuf` protocol. |
| `prometheus.metrics.reporter.otlp.protocol` | String | Not set | The OTLP transport protocol, either `grpc` or `http/protobuf`. |
| `prometheus.metrics.reporter.otlp.interval.seconds` | Integer | Not set | The interval in seconds between two pushes. |
| `prometheus.metrics.reporter.otlp.resource.attributes` | List | Empty | A comma-separated list of `key=value` pairs added as resource attributes to the pushed metrics, in the same format as the `OTEL_RESOURCE_ATTRIBUTES` environment variable. |

The configuration names use the existing `prometheus.metrics.reporter.` prefix so that all *Reporter* configurations stay consistent.

When a setting is not set, the standard OpenTelemetry configuration applies, as described in the next section.
If nothing is configured at all, the OpenTelemetry SDK defaults apply, which are the `grpc` protocol, the `http://localhost:4317` endpoint and a 60 seconds interval.

The new configurations are not dynamically reconfigurable.
Changing them requires restarting the component.

#### Precedence with the standard OpenTelemetry configuration

The OpenTelemetry SDK can also be configured through standard environment variables and system properties, for example `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` or `OTEL_RESOURCE_ATTRIBUTES`.
The *Reporter* uses the following order of precedence, from the highest to the lowest:
1. The *Reporter* configurations described above.
2. The metrics-specific OpenTelemetry settings, such as `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`.
3. The generic OpenTelemetry settings, such as `OTEL_EXPORTER_OTLP_ENDPOINT`.
4. The OpenTelemetry SDK defaults.

Resource attributes from all sources are merged, and the *Reporter* configuration wins for duplicate keys.

This order matters because the values set through the `OpenTelemetryExporter.Builder` methods have the lowest priority, and are overridden by any `OTEL_*` environment variable.
For example, a Kafka Connect user who sets `OTEL_EXPORTER_OTLP_ENDPOINT` for tracing would silently send the metrics to the tracing endpoint, even when a different metrics endpoint is configured in the *Reporter*.
To avoid that, the *Reporter* passes its configuration to the exporter through `PrometheusProperties` (`OpenTelemetryExporter.builder(PrometheusProperties)`), which has precedence over the `OTEL_*` settings.
The exporter does not append the `/v1/metrics` path to an endpoint passed this way, so the *Reporter* appends it itself when the `http/protobuf` protocol is used.

The settings that the *Reporter* does not expose, such as headers, timeouts, compression and TLS certificates, are configured through the standard `OTEL_EXPORTER_OTLP_*` environment variables or system properties.
This is the same approach that Strimzi already documents for tracing.

#### One exporter per JVM

Several reporter instances share the same registry in a single JVM.
For example, Kafka Connect runs reporters for the worker and for its admin, producer and consumer clients, and Kafka Streams applications create several clients.
If each reporter instance started its own exporter, every metric would be pushed several times.

Similar to the existing `HttpServers` class, a new `OtlpExporters` class keeps track of the exporters started in a JVM.
Reporter instances configured with the same OTLP settings share a single reference-counted exporter, which is closed when the last reporter using it is closed.
On brokers, the exporter is started by `ServerKafkaMetricsReporter`, in the same way as the HTTP listener today.
Because the exporter pushes the whole registry, it also pushes the metrics collected by `ServerYammerMetricsReporter`.

Each exporter pushes the whole registry.
If reporter instances in the same JVM are configured with different OTLP settings, for example through connector client overrides in Kafka Connect, each distinct set of settings gets its own exporter that pushes all metrics.
This is the same behavior as the existing HTTP listener when different listener addresses are configured.

#### Metric naming

The metric names and labels are the same as the ones exposed by the Prometheus endpoint, with the conversions applied by the `OpenTelemetryExporter` to follow the OpenTelemetry conventions:
- Labels become data point attributes.
- The `_total` suffix of counters is not part of the OTLP metric name.
- Unit suffixes are removed from the metric name and set as the metric unit instead.
  The Kafka and Yammer metrics collected by the *Reporter* have no units, so this only affects JVM metrics, such as `jvm_memory_used_bytes`.

Backends that convert OTLP metrics to the Prometheus format add the suffixes back.
For example, the default translation strategy of the Prometheus OTLP receiver adds the `_total` suffix to monotonic counters, and the unit suffix to metrics with a unit.
As a result, metrics pushed over OTLP to Prometheus should have the same names as the scraped ones.
The integration tests of the *Reporter* must verify this for gauges, counters and histograms, by pushing to a real OpenTelemetry Collector.

#### Dependencies

The `prometheus-metrics-exporter-opentelemetry` artifact is about 5.7 MB because it includes the shaded OpenTelemetry SDK.
The *Reporter* only loads its classes when `prometheus.metrics.reporter.otlp.enable` is `true`.
Client applications that do not use OTLP can exclude the dependency, and enabling OTLP without it fails with a clear error message.

### Strimzi Cluster Operator

#### API

The `strimziMetricsReporter` type gains two optional properties in `values`:
- `prometheusEndpoint` configures the existing Prometheus endpoint.
  It has a single `enabled` property, which defaults to `true`.
  Using an object instead of a boolean leaves room for future options, such as the TLS configuration that [Proposal 143](./143-enable-ssl-for-metrics-reporter.md) deferred to a separate proposal.
- `otlp` enables and configures the OTLP push.
  Its presence enables the push.

The `otlp` object has the following properties:

| Property | Type | Required | Description |
|:---------|:-----|:---------|:------------|
| `endpoint` | String | No | The OTLP endpoint, which must start with `http://` or `https://`. When not set, the standard OpenTelemetry configuration applies, for example the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable set in the container template. |
| `protocol` | Enum (`grpc`, `http/protobuf`) | No | The OTLP transport protocol. Defaults to `grpc`. |
| `intervalSeconds` | Integer | No | The interval in seconds between two pushes. Must be at least 1. Defaults to 60. |
| `resourceAttributes` | Map of strings | No | Additional resource attributes added to the pushed metrics. They override the attributes set by the operator for the same keys. |

The defaults in the table are the OpenTelemetry SDK defaults.
The operator does not write default values to the generated configuration, so the standard `OTEL_*` environment variables can still override them.

The following example pushes the metrics over OTLP and disables the Prometheus endpoint:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    metricsConfig:
      type: strimziMetricsReporter
      values:
        allowList:
          - "kafka_server_replicamanager_.*"
          - "kafka_controller_.*"
        prometheusEndpoint:
          enabled: false
        otlp:
          endpoint: http://otel-collector.observability.svc:4317
          protocol: grpc
          intervalSeconds: 30
          resourceAttributes:
            deployment.environment.name: production
```

When `prometheusEndpoint` is not set, the Prometheus endpoint stays enabled, so existing resources keep their current behavior.

The same API is available in all resources that already support the `strimziMetricsReporter` type:
- `Kafka` (brokers and controllers)
- `KafkaConnect`
- `KafkaMirrorMaker2`
- `KafkaBridge`

#### Validation

The following rules are validated by CEL rules in the CRDs and in the operator:
- At least one output must be enabled, so `prometheusEndpoint.enabled: false` is only valid when `otlp` is set.
- `otlp.endpoint` must start with `http://` or `https://`.
- `otlp.intervalSeconds` must be at least 1.

The operator percent-encodes the values of `otlp.resourceAttributes` when generating the configuration, as defined for `OTEL_RESOURCE_ATTRIBUTES`, so values can contain characters such as `,` and `=`.

#### Generated configuration

For brokers and controllers, the operator generates the following *Reporter* configuration for the example above:

```properties
prometheus.metrics.reporter.listener.enable=false
prometheus.metrics.reporter.allowlist=kafka_server_replicamanager_.*,kafka_controller_.*
prometheus.metrics.reporter.otlp.enable=true
prometheus.metrics.reporter.otlp.endpoint=http://otel-collector.observability.svc:4317
prometheus.metrics.reporter.otlp.protocol=grpc
prometheus.metrics.reporter.otlp.interval.seconds=30
prometheus.metrics.reporter.otlp.resource.attributes=<see the next section>
```

For `KafkaConnect` and `KafkaMirrorMaker2`, the same configurations are also generated with the `admin.`, `producer.` and `consumer.` prefixes, as is done today for the existing *Reporter* configurations.
The shared exporter described above ensures that the metrics are pushed only once per pod.

#### Resource attributes

Pull-based pipelines identify the source of the metrics from the scrape target.
With push, the pod has to identify itself, so the operator sets resource attributes that follow the OpenTelemetry [semantic conventions for Kubernetes](https://opentelemetry.io/docs/specs/semconv/non-normative/k8s-attributes/):

| Attribute | Value |
|:----------|:------|
| `k8s.namespace.name` | The namespace of the pod. |
| `k8s.pod.name` | The name of the pod. |
| `k8s.container.name` | The name of the container. |
| `k8s.node.name` | The name of the node the pod is running on. |
| `service.name` | The name of the custom resource, which is also the value of the `app.kubernetes.io/instance` label. |
| `service.namespace` | The namespace of the pod. |
| `service.instance.id` | `<namespace>.<pod name>.<container name>` |

Setting `service.instance.id` to a unique value per pod is important.
Some backends use it to identify the source, for example the Prometheus OTLP receiver maps it to the `instance` label.
Without it, the series from different pods would collide.

The operator does not set `service.name` when the user sets it, either in `otlp.resourceAttributes` or through the `OTEL_SERVICE_NAME` environment variable in the container template.
This way, the metrics and the traces of a component can share the same `service.name`, which is required by Strimzi's tracing configuration.

Values that are known when generating the configuration, such as the namespace, the resource name and the container name, are written as literals.
Values that are only known inside the pod are passed through environment variables populated from the Downward API, and referenced with the `${strimzienv:...}` placeholders of the `EnvVarConfigProvider`.
The operator already configures this provider and uses these placeholders, for example for `broker.rack` and `rest.advertised.host.name`.
The placeholders allow a single configuration to work for all pods of a component, including Kafka Connect, where all pods share the same configuration.

The generated configuration only changes when the custom resource changes, so the placeholders don't cause additional rolling updates.

#### Ports and network policies

Today, the operator opens the metrics container port and adds a `NetworkPolicy` ingress rule for it whenever metrics are configured.
When `prometheusEndpoint.enabled` is `false`, the operator does not create the metrics port or the `NetworkPolicy` rule, because nothing needs to connect to the pod.

Pushing metrics requires egress traffic from the pods to the OTLP endpoint.
Strimzi does not manage egress network policies, so users who restrict egress traffic have to allow it themselves.

#### TLS and authentication

The `https://` scheme in `otlp.endpoint` enables TLS.
By default, the server certificate is validated against the JVM default trust store.

Custom trusted certificates, client certificates and authentication headers are configured through the standard OpenTelemetry environment variables, such as `OTEL_EXPORTER_OTLP_CERTIFICATE` and `OTEL_EXPORTER_OTLP_HEADERS`.
Users set them in the container template, and they can mount certificates using additional volumes.
Credentials stay in Kubernetes `Secrets`, because container template environment variables can reference them with `valueFrom.secretKeyRef`.
If they were part of the custom resource, they would end up in the generated configuration, which is stored in a `ConfigMap`.

This is the same approach that Strimzi already documents for tracing.
First-class TLS configuration in the custom resource, using the `CertSecretSource` pattern used elsewhere in the Strimzi API, can be added later without breaking this API.

#### Rolling updates

The *Reporter* OTLP configurations are not dynamically reconfigurable.
Changing the `otlp` or `prometheusEndpoint` properties triggers a rolling update of the affected pods.
Changing the allow list keeps its current behavior, so brokers are reconfigured dynamically.

### HTTP Bridge

The HTTP Bridge serves its own metrics and needs a different approach.
It serves three kinds of metrics on its `/metrics` endpoint:
- Kafka client metrics, collected by the *Reporter* in the shared default `PrometheusRegistry`.
- JVM metrics.
- HTTP server metrics, collected by Vert.x in a separate Micrometer registry.

The Bridge also creates Kafka clients on demand.
If the Bridge relied on the OTLP output of the *Reporter*, the Vert.x metrics would not be pushed, and nothing would be pushed while no Kafka client exists.

For the same reasons that the Bridge disables the HTTP listener of the *Reporter* and serves the metrics itself, it also owns the OTLP push.
The Bridge starts the exporter when it starts and stops it when it shuts down.
It pushes the same metrics it serves on `/metrics`, by reusing the `OtlpExporters` class from the *Reporter* for both registries.
The OTLP output of the *Reporter* stays disabled in the Kafka clients created by the Bridge.

#### Standalone configuration

The Bridge gets the following configurations, which mirror the *Reporter* ones:

```properties
bridge.metrics=strimziMetricsReporter
# disables the /metrics endpoint, defaults to true
bridge.metrics.endpoint.enable=false
bridge.metrics.otlp.enable=true
bridge.metrics.otlp.endpoint=http://otel-collector.observability.svc:4317
bridge.metrics.otlp.protocol=grpc
bridge.metrics.otlp.interval.seconds=30
bridge.metrics.otlp.resource.attributes=deployment.environment.name=production
```

The defaults and the precedence with the standard OpenTelemetry configuration are the same as in the *Reporter*.
OTLP push is only supported with `bridge.metrics=strimziMetricsReporter`, to match the Cluster Operator API.
`bridge.metrics.endpoint.enable=false` is only valid when `bridge.metrics.otlp.enable=true`.

The Bridge resolves `${strimzienv:...}` placeholders in `bridge.metrics.otlp.resource.attributes` with the Kafka `EnvVarConfigProvider`.
This way, the operator can generate the resource attributes for the Bridge in the same way as for the other components.

#### Cluster Operator integration

When the `KafkaBridge` resource uses `otlp`, the operator generates the `bridge.metrics.otlp.*` configurations instead of the `kafka.prometheus.metrics.reporter.otlp.*` ones.
When `prometheusEndpoint.enabled` is `false`, the operator sets `bridge.metrics.endpoint.enable=false`.
The Bridge exposes its metrics on its HTTP port, so there is no metrics port or `NetworkPolicy` rule to remove.

### Interaction with tracing

`KafkaConnect`, `KafkaMirrorMaker2` and `KafkaBridge` can use OpenTelemetry tracing and OTLP metrics push at the same time.
Tracing uses its own, unshaded OpenTelemetry SDK, and the *Reporter* uses the shaded SDK bundled in the `OpenTelemetryExporter`.
Both SDKs read the same standard configuration, which has the following consequences.

#### Shared environment variables

The generic `OTEL_EXPORTER_OTLP_*` environment variables apply to both traces and metrics, unless the *Reporter* configuration sets the corresponding value.
Users who already use tracing can push metrics to the same collector without configuring the endpoint again, by setting an empty `otlp: {}` object.
Users who want to send the signals to different backends can set `otlp.endpoint`, or use the signal-specific environment variables, such as `OTEL_EXPORTER_OTLP_TRACES_HEADERS` and `OTEL_EXPORTER_OTLP_METRICS_HEADERS`.

#### The `otel.metrics.exporter` system property

When tracing is enabled, both the Strimzi tracing agent (used by Kafka Connect and MirrorMaker 2) and the Bridge set the `otel.metrics.exporter` system property to `none`, to disable the metrics exporter of the tracing SDK.
The shaded SDK in the `OpenTelemetryExporter` reads the same JVM-wide system property.
With this property set, the `OpenTelemetryExporter` fails to start with a `NullPointerException`, which we reproduced with version 1.9.0.

To fix this, the proposal includes the following changes:
1. The Strimzi tracing agent and the Bridge configure the tracing SDK instance itself, through `AutoConfiguredOpenTelemetrySdk.builder().addPropertiesCustomizer(...)`, instead of setting a JVM-wide system property.
2. The *Reporter* catches failures when it starts the exporter and reports a clear error message that points to the `otel.metrics.exporter` setting.
3. We contribute a change to the Prometheus Java client so that its exporter always enables the OTLP metrics exporter of its own SDK instance, in the same way it already disables the traces and logs exporters.

#### Initialization order

When it starts, the `OpenTelemetryExporter` calls `GlobalOpenTelemetry.get()` on the unshaded OpenTelemetry API, if it is on the classpath, to read the resource attributes of the OpenTelemetry Java agent.
This sets a no-op global instance, so a tracing SDK that registers itself as the global instance afterwards fails to start.
To avoid this, tracing must be initialized before the OTLP metrics push starts:
- In Kafka Connect and MirrorMaker 2, the Strimzi tracing agent initializes tracing when the JVM starts, before any reporter is configured.
- The Bridge initializes tracing before it starts its HTTP server and the OTLP metrics push.

### Testing

The *Reporter* gets integration tests that push metrics to an OpenTelemetry Collector running in a container.
They verify the metric names, the attributes and the resource attributes for both protocols.

The Cluster Operator gets unit tests for the generated configurations, the ports and the network policies, and a system test based on the existing `StrimziMetricsReporterST`.
The system test deploys an OpenTelemetry Collector and checks that metrics from brokers, controllers, Kafka Connect, MirrorMaker 2 and the Bridge arrive with the expected resource attributes.
It also covers tracing and OTLP metrics push enabled at the same time.

### Implementation plan

1. Implement the OTLP output in the *Reporter* and release it.
2. Change how the Strimzi tracing agent and the Bridge disable the metrics exporter of the tracing SDK.
3. Implement the OTLP push in the Bridge, using the new *Reporter* release.
4. Bump the *Reporter* in the Kafka container images, and implement the API and the configuration generation in the Cluster Operator.
5. Add the documentation and an example with an OpenTelemetry Collector.

## Affected/not affected projects

The following projects are affected:
- `metrics-reporter`: adds the OTLP output.
- `strimzi-kafka-operator`: adds the API and generates the configuration for `Kafka`, `KafkaConnect`, `KafkaMirrorMaker2` and `KafkaBridge`, and changes the tracing agent.
  The Kafka container images get the new *Reporter* version, including the exporter dependency.
- `strimzi-kafka-bridge`: adds the OTLP push and the option to disable the `/metrics` endpoint, and changes how it disables the metrics exporter of the tracing SDK.

The following projects are not affected:
- `strimzi-kafka-oauth` and `kafka-quotas-plugin`: their metrics already end up in the *Reporter* registry when it is used, so they are pushed as well.
- Cruise Control: it does not support the `strimziMetricsReporter` type.
- Kafka Exporter: it is not based on the *Reporter*.
  The re-implementation described in [Proposal 158](./158-kafka-exporter-re-implementation.md) could adopt OTLP push later.
- The metrics of the Strimzi operators themselves.

## Compatibility

All new configurations and API properties are optional, and push is disabled by default.
Existing configurations and custom resources keep their current behavior, including the Prometheus endpoint, which stays enabled unless it is explicitly disabled.
The new API properties are added to the existing `v1` API, so no new API version is needed.

The change to how tracing disables the metrics exporter of the tracing SDK does not change the tracing behavior.

Metrics pushed over OTLP follow the OpenTelemetry naming conventions described above.
Backends that convert them back to the Prometheus format should produce the same metric names as scraping.
However, the labels depend on how the backend converts resource attributes.
For example, the Prometheus OTLP receiver maps `service.name` and `service.instance.id` to the `job` and `instance` labels, and only promotes other resource attributes to labels when configured to do so.
The Grafana dashboards and Prometheus alerting rules provided by Strimzi rely on the labels added by the example `PodMonitor` resources, so they may need to be adapted for OTLP data.
Providing dashboards for OTLP data is not part of this proposal.

## Rejected alternatives

### A new `metricsConfig` type

A new type, for example `openTelemetryMetricsReporter`, would configure the *Reporter* for push only.
This would make validation slightly simpler, for example by requiring the endpoint.
However, it would suggest a different implementation, even though the same plugin, allow list and metric names are used.
It would also make it impossible to expose the Prometheus endpoint and push at the same time, which is useful during a migration.
OTLP is a transport of the same *Reporter*, so it is modeled as an option of the existing type.

### Separate reporter classes for OTLP

Dedicated OTLP reporter classes would need their own collectors and registry, or would have to coordinate with the existing ones.
`ServerYammerMetricsReporter` does not support a second instance, and the allow list would have to be configured and reconfigured twice.

### A new top-level telemetry API

A new top-level property, for example `spec.telemetry`, could configure both traces and metrics.
Tracing is already configured through its own `tracing` property, and is configured through environment variables, while metrics are configured through `metricsConfig`.
Merging them would be a much larger and breaking API change, so it is not part of this proposal.
The standard `OTEL_*` environment variables already allow sharing the exporter configuration between both signals.

### Configuring OTLP only through environment variables

The tracing configuration only relies on the `OTEL_*` environment variables.
Doing the same for metrics would avoid adding properties to the API.
However, the endpoint and the interval are the main settings of the feature, and having them in the custom resource makes them discoverable and validated.
The environment variables are still supported for the settings that are not part of the API.

### Deploying an OpenTelemetry Collector with the operator

The operator could deploy a collector that scrapes the Prometheus endpoints and forwards the metrics over OTLP.
This keeps all the drawbacks described in the motivation, and adds a new workload for the operator to manage.
Users can still deploy and manage their own collectors.

### Using the OpenTelemetry SDK or Micrometer directly

The *Reporter* could convert the Kafka and Yammer metrics to the OpenTelemetry SDK or to Micrometer directly.
This would duplicate the conversion that the Prometheus Java client already provides, and could lead to different metric names and semantics between the two outputs.
Using the unshaded OpenTelemetry SDK could also conflict with the OpenTelemetry version used by tracing or by connector plugins.

### Separate allow lists for each output

Each output could have its own allow list.
The allow list is applied when the metrics are collected into the shared registry, so separate allow lists would require separate registries.
The added complexity is not justified by a clear use case.

### Exposing headers and TLS certificates in the custom resource

The OTLP headers and TLS certificates could be configured in the custom resource.
Headers usually contain credentials, which would then be stored in the generated configuration `ConfigMap`.
The standard environment variables combined with `Secret` references and additional volumes already support these settings.
First-class TLS support can be added in a future proposal.
