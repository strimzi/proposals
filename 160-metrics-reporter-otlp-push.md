# Support for OTLP push metrics in the Strimzi Metrics Reporter

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
- Deploy an OpenTelemetry Collector that scrapes the Prometheus endpoints with its `prometheus` receiver, and forwards the metrics over OTLP.
  A single Collector can scrape several Kafka clusters and other components.
- Configure the OpenTelemetry mode of the *Exporter* through its YAML configuration file, and keep using JMX.
  The resource attributes in this file are static values, and Strimzi uses a single configuration file for all pods of a component, so the pods cannot identify themselves.
  The `OTEL_RESOURCE_ATTRIBUTES` environment variable does not help either, because the container template only supports static values or references to `Secrets` and `ConfigMaps`, and not the Downward API.

Distributed tracing is configured separately through the `tracing` property with `type: opentelemetry` in `KafkaConnect`, `KafkaMirrorMaker2` and `KafkaBridge`.
Users configure the tracing exporter through the standard `OTEL_*` environment variables, for example `OTEL_SERVICE_NAME` and `OTEL_EXPORTER_OTLP_ENDPOINT`, set in the container template.

## Motivation

OTLP is becoming the vendor-neutral standard for transporting telemetry.
Most observability backends, including Prometheus itself, can ingest OTLP metrics, and some managed backends only accept pushed data.

Scraping the pods with a collector to convert the metrics to OTLP has several drawbacks:
- An additional workload is needed, with its own configuration and the RBAC resources to discover the pods.
- The scraping has to be coordinated between the Collector replicas.
  Each replica discovers and scrapes all the pods on its own, so with more than one replica, for example for high availability, every metric is collected and forwarded several times.
  Avoiding this requires sharding the targets between the replicas, for example with the target allocator of the OpenTelemetry Operator.
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
  In the custom resources, OTLP push is configured with a new `strimziMetricsReporterOtlp` metrics type, next to the existing `strimziMetricsReporter` type, with the same allow list.
- Concepts that only make sense for pull are not forced onto push.
  Where push needs something new, such as the identity of the source pod, the proposal follows OpenTelemetry conventions instead of inventing Strimzi-specific ones.

The changes are split between the *Reporter*, the Cluster Operator and the HTTP Bridge.

### Strimzi Metrics Reporter

#### OTLP output

The *Reporter* gains an optional OTLP output next to the existing HTTP listener.
Both outputs are independent, so the *Reporter* can expose the Prometheus endpoint, push over OTLP, or do both at the same time.
Applications using the *Reporter* directly can run both outputs while they migrate from one monitoring pipeline to the other.
In the Strimzi custom resources, the type of the metrics configuration selects one of the outputs, as described in [Migration](#migration).

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
| `prometheus.metrics.reporter.otlp.endpoint` | String | Not set | The OTLP endpoint in the `http://<HOST>:<PORT>` or `https://<HOST>:<PORT>` format. Like `OTEL_EXPORTER_OTLP_ENDPOINT`, it is a base URL, so the `/v1/metrics` path is appended with the `http/protobuf` protocol, as defined in the [OTLP exporter specification](https://opentelemetry.io/docs/specs/otel/protocol/exporter/#endpoint-urls-for-otlphttp). When not set, the standard OpenTelemetry configuration applies, which defaults to `http://localhost:4317`. |
| `prometheus.metrics.reporter.otlp.protocol` | String | Not set | The OTLP transport protocol, either `grpc` or `http/protobuf`. When not set, the standard OpenTelemetry configuration applies, which defaults to `grpc`. |
| `prometheus.metrics.reporter.otlp.interval.seconds` | Integer | Not set | The interval in seconds between two pushes. When not set, the standard OpenTelemetry configuration applies (`OTEL_METRIC_EXPORT_INTERVAL`), which defaults to 60 seconds. |
| `prometheus.metrics.reporter.otlp.resource.attributes` | List | Empty | A comma-separated list of `key=value` pairs added as resource attributes to the pushed metrics, in the same format as the `OTEL_RESOURCE_ATTRIBUTES` environment variable. |
| `prometheus.metrics.reporter.otlp.ssl.truststore.location` | String | Not set | The path to a PEM file with the trusted certificates used to verify the OTLP endpoint. |
| `prometheus.metrics.reporter.otlp.ssl.truststore.certificates` | String | Not set | The trusted certificates used to verify the OTLP endpoint, in PEM format. |
| `prometheus.metrics.reporter.otlp.ssl.keystore.location` | String | Not set | The path to a PEM file with the private key and the certificate chain used for mTLS. |
| `prometheus.metrics.reporter.otlp.ssl.keystore.certificate.chain` | String | Not set | The certificate chain used for mTLS, in PEM format. |
| `prometheus.metrics.reporter.otlp.ssl.keystore.key` | Password | Not set | The private key used for mTLS, in PEM format. |

The configuration names use the existing `prometheus.metrics.reporter.` prefix so that all *Reporter* configurations stay consistent.

Nothing is pushed unless `prometheus.metrics.reporter.otlp.enable` is set to `true`.
Otherwise, the *Reporter* ignores the other OTLP configurations and the standard OpenTelemetry environment variables and system properties.
When OTLP is enabled and a setting is not set, the standard OpenTelemetry configuration applies, as described in the next section.
If neither the *Reporter* nor the standard OpenTelemetry configuration sets them, the OpenTelemetry SDK defaults apply, which are the `grpc` protocol, the `http://localhost:4317` endpoint and a 60 seconds interval.

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

The settings that the *Reporter* does not expose, such as headers, timeouts and compression, are configured through the standard `OTEL_EXPORTER_OTLP_*` environment variables or system properties.
This is the same approach that Strimzi already documents for tracing.

#### TLS

The `https://` scheme in the endpoint enables TLS.
By default, the server certificate is validated against the JVM default trust store.

The TLS configurations follow the names of the Kafka PEM configurations, such as `ssl.truststore.certificates`, `ssl.keystore.certificate.chain` and `ssl.keystore.key`.
Only the PEM format is supported, like in [Proposal 143](./143-enable-ssl-for-metrics-reporter.md).
The certificates can be provided either as a path to a PEM file or inline, but not both.
The inline values allow using Kafka configuration providers, which is how the Cluster Operator passes certificates from `Secrets`.
The private key must be an unencrypted PKCS#8 key, which is the only format the OpenTelemetry SDK supports, so there is no equivalent of the Kafka `ssl.key.password` configuration.
The keys generated by Strimzi, for example for `KafkaUser` resources, already use this format.

The OpenTelemetry SDK accepts the trusted certificates, the client certificate and the client key as file paths, so the *Reporter* writes the inline values to temporary files that only the current user can read.
The `PrometheusProperties` of the exporter do not have options for these settings yet, so we contribute them to the Prometheus Java client, mapped to the metrics-specific `otel.exporter.otlp.metrics.certificate`, `otel.exporter.otlp.metrics.client.certificate` and `otel.exporter.otlp.metrics.client.key` OpenTelemetry settings.
Until then, the *Reporter* sets these settings as system properties while it creates the exporter, because the OpenTelemetry SDK reads the files at that time.
Afterwards, it restores the previous values of the system properties and deletes the temporary files.

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

### Strimzi Cluster Operator

#### API

A new `strimziMetricsReporterOtlp` type is added to `metricsConfig`.
It uses the same *Reporter* as the `strimziMetricsReporter` type, with the same allow list and metric names, but it pushes the metrics over OTLP instead of exposing them on a Prometheus endpoint.
The existing `strimziMetricsReporter` type keeps its current behavior.

The `values` of the new type have the following properties:

| Property | Type | Required | Description |
|:---------|:-----|:---------|:------------|
| `allowList` | List of strings | No | A list of regex patterns to filter the metrics to collect, with the same default as for the `strimziMetricsReporter` type. |
| `endpoint` | String | No | The OTLP endpoint in the `http://<HOST>:<PORT>` or `https://<HOST>:<PORT>` format. When not set, the standard OpenTelemetry configuration applies, for example the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable set in the container template. |
| `protocol` | Enum (`grpc`, `http/protobuf`) | No | The OTLP transport protocol. Defaults to `grpc`. |
| `intervalSeconds` | Integer | No | The interval in seconds between two pushes. Must be at least 1. Defaults to 60. |
| `resourceAttributes` | List | No | The resource attributes added to the pushed metrics, as a list of `name` and `value` or `valueFrom` pairs. See [Resource attributes](#resource-attributes). |
| `tls` | Object | No | The TLS configuration, with the `trustedCertificates` used to verify the OTLP endpoint. See [TLS and authentication](#tls-and-authentication). |
| `authentication` | Object | No | The authentication configuration. The `tls` type uses a client certificate for mTLS. See [TLS and authentication](#tls-and-authentication). |

The defaults in the table, except for `allowList`, are the OpenTelemetry SDK defaults.
The operator does not write these default values to the generated configuration, so the standard `OTEL_*` environment variables can still override them.

The following example pushes the metrics of a Kafka cluster over OTLP:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    metricsConfig:
      type: strimziMetricsReporterOtlp
      values:
        allowList:
          - "kafka_server_replicamanager_.*"
          - "kafka_controller_.*"
        endpoint: http://otel-collector.observability.svc:4317
        protocol: grpc
        intervalSeconds: 30
        resourceAttributes:
          - name: deployment.environment.name
            value: production
```

The new type is available in all resources that support the `strimziMetricsReporter` type:
- `Kafka` (brokers and controllers)
- `KafkaConnect`
- `KafkaMirrorMaker2`
- `KafkaBridge`

#### Validation

The CRD schema validates the supported `protocol` values and that `intervalSeconds` is at least 1.
The format of the endpoint is validated by the *Reporter*.
The operator validates the `allowList` in the same way as for the `strimziMetricsReporter` type.
The CRD schema also validates that each resource attribute has exactly one of `value` and `valueFrom`, in the same way as the environment variables of the container template.
The operator validates that `valueFrom.fieldRef` only uses the fields supported by the Downward API for environment variables, so that an invalid field is reported in the custom resource status instead of failing when the pods are created.

The operator percent-encodes the names and values of `resourceAttributes` when generating the configuration, as defined for `OTEL_RESOURCE_ATTRIBUTES`, so they can contain characters such as `,` and `=`.

#### Generated configuration

For brokers and controllers, the operator generates the following *Reporter* configuration for the example above.
The HTTP listener of the *Reporter* is always disabled with this type.


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
With push, the source has to be identified by resource attributes.
They can be added centrally by the OpenTelemetry Collector, for example with the `k8sattributes` processor, or by the pods themselves.

The operator does not set any resource attributes by default.
Users configure them in `resourceAttributes`, as a list of `name` and `value` pairs.
Instead of a static `value`, an attribute can take its value from the pod through `valueFrom.fieldRef`, which supports the same fields as the Kubernetes Downward API for environment variables, such as `metadata.name`, `metadata.namespace` and `spec.nodeName`.
Reading these fields does not require any RBAC permissions, because the kubelet provides them.

The following example sets the attributes of the OpenTelemetry [semantic conventions for Kubernetes](https://opentelemetry.io/docs/specs/semconv/non-normative/k8s-attributes/) that identify the pod:

```yaml
metricsConfig:
  type: strimziMetricsReporterOtlp
  values:
    resourceAttributes:
      - name: k8s.namespace.name
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: k8s.pod.name
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: service.namespace
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: service.instance.id
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
```

Some backends need a unique identity for each pod.
For example, the Prometheus OTLP receiver maps `service.namespace`, `service.name` and `service.instance.id` to the `job` and `instance` labels.
Most Kafka metrics do not have a label identifying the pod, so without a unique identity, either from the pods or added by the Collector, the series from different pods would collide.

For each attribute using `valueFrom`, the operator adds an environment variable populated from the Downward API to the container, and references it with a `${strimzienv:...}` placeholder of the `EnvVarConfigProvider`.
The operator already configures this provider and uses these placeholders, for example for `broker.rack` and `rest.advertised.host.name`.
The placeholders allow a single configuration to work for all pods of a component, including Kafka Connect, where all pods share the same configuration.

The generated configuration only changes when the custom resource changes, so the placeholders don't cause additional rolling updates.

#### Ports and network policies

Today, the operator opens the metrics container port and adds a `NetworkPolicy` ingress rule for it whenever metrics are configured.
With the `strimziMetricsReporterOtlp` type, the operator does not create the metrics port or the `NetworkPolicy` rule, because nothing needs to connect to the pod.

Pushing metrics requires egress traffic from the pods to the OTLP endpoint.
Strimzi does not manage egress network policies, so users who restrict egress traffic have to allow it themselves.

#### TLS and authentication

TLS is configured in the same way as the TLS connection of Kafka Connect to the Kafka cluster:
- `tls.trustedCertificates` is a list of certificates from `Secrets`, used to verify the OTLP endpoint.
- `authentication` with `type: tls` configures mTLS, with the client certificate and key from a `Secret` in `certificateAndKey`.

```yaml
metricsConfig:
  type: strimziMetricsReporterOtlp
  values:
    endpoint: https://otel-collector.observability.svc:4317
    tls:
      trustedCertificates:
        - secretName: otel-collector-ca
          certificate: ca.crt
    authentication:
      type: tls
      certificateAndKey:
        secretName: my-otlp-client
        certificate: tls.crt
        key: tls.key
```

As for Kafka Connect, the operator collects the trusted certificates from all the `Secrets` listed in `trustedCertificates` into an internal `<component_name>-otlp-trusted-certs` `Secret`.
It passes the trusted certificates and the client certificate and key with the `KubernetesSecretConfigProvider` (`strimzisecrets`) to the inline `ssl.*` configurations of the *Reporter*, and gives the pods access to the `Secrets` they need.
The certificates and the key are therefore never stored in the generated configuration.

Other authentication methods, such as tokens sent in headers, can be configured through the standard `OTEL_EXPORTER_OTLP_HEADERS` environment variable in the container template, using `valueFrom.secretKeyRef` so that the credentials stay in a `Secret`.
New types can be added to `authentication` in the future.

#### Rolling updates

The *Reporter* OTLP configurations are not dynamically reconfigurable.
Changing the configuration of the `strimziMetricsReporterOtlp` type, or switching between the `strimziMetricsReporter` and `strimziMetricsReporterOtlp` types, triggers a rolling update of the affected pods.
The *Reporter* only reads the certificates when it starts, so the operator also rolls the pods when the content of the trusted certificates or of the client certificate and key changes, for example when they are renewed.
It does so with a pod annotation containing a hash of the certificates, in the same way as for the certificates of the Kafka Connect connection to the Kafka cluster.
Changing only the allow list keeps its current behavior, so brokers are reconfigured dynamically.

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
bridge.metrics=strimziMetricsReporterOtlp
bridge.metrics.otlp.endpoint=http://otel-collector.observability.svc:4317
bridge.metrics.otlp.protocol=grpc
bridge.metrics.otlp.interval.seconds=30
bridge.metrics.otlp.resource.attributes=deployment.environment.name=production
```

Like in the Cluster Operator API, the `strimziMetricsReporterOtlp` value of `bridge.metrics` selects the OTLP push.
With it, the Bridge pushes the metrics and does not serve them on the `/metrics` endpoint.
The HTTP server keeps running, because it serves the REST API.
The Bridge also supports the `bridge.metrics.otlp.ssl.*` configurations, with the same names as the TLS configurations of the *Reporter*.
The defaults and the precedence with the standard OpenTelemetry configuration are the same as in the *Reporter*.

Like the Kafka clients, the Bridge resolves the configuration provider placeholders in the `bridge.metrics.otlp.resource.attributes` and `bridge.metrics.otlp.ssl.*` configurations.
It resolves `${strimzienv:...}` placeholders with the Kafka `EnvVarConfigProvider`, and `${strimzisecrets:...}` placeholders with the `KubernetesSecretConfigProvider`, which it only creates when a configuration uses it.
This way, the operator can generate the resource attributes and the TLS configuration for the Bridge in the same way as for the other components.

#### Cluster Operator integration

When the `KafkaBridge` resource uses the `strimziMetricsReporterOtlp` type, the operator sets `bridge.metrics=strimziMetricsReporterOtlp` and generates the `bridge.metrics.otlp.*` configurations instead of the `kafka.prometheus.metrics.reporter.otlp.*` ones.
The Bridge exposes its metrics on its HTTP port, so there is no metrics port or `NetworkPolicy` rule to remove.

### Interaction with tracing

`KafkaConnect`, `KafkaMirrorMaker2` and `KafkaBridge` can use OpenTelemetry tracing and OTLP metrics push at the same time.
Tracing uses its own, unshaded OpenTelemetry SDK, and the *Reporter* uses the shaded SDK bundled in the `OpenTelemetryExporter`.
Both SDKs read the same standard configuration, which has the following consequences.

#### Shared environment variables

The generic `OTEL_EXPORTER_OTLP_*` environment variables apply to both traces and metrics, unless the *Reporter* configuration sets the corresponding value.
Users who already use tracing can push metrics to the same collector without configuring the endpoint again, by using the `strimziMetricsReporterOtlp` type without an `endpoint`.
Users who want to send the signals to different backends can set the `endpoint`, or use the signal-specific environment variables, such as `OTEL_EXPORTER_OTLP_TRACES_HEADERS` and `OTEL_EXPORTER_OTLP_METRICS_HEADERS`.
In the same way, the `OTEL_SERVICE_NAME` set for tracing also applies to the metrics, unless `service.name` is set in `resourceAttributes`.

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

### Migration

Because the type of the metrics configuration selects the output, a component cannot expose the Prometheus endpoint and push over OTLP at the same time.
Changing the type from `strimziMetricsReporter` to `strimziMetricsReporterOtlp` rolls the pods, and from then on their metrics are only pushed.
Each custom resource has its own metrics configuration, so the components can be migrated one at a time, for example the Kafka Connect clusters before the Kafka clusters.

To migrate to an OTLP backend without losing metrics, we recommend the following steps:
1. Deploy an OpenTelemetry Collector that scrapes the existing Prometheus endpoints with its `prometheus` receiver, and forwards the metrics over OTLP to the new backend.
   The Collector can discover the pods in the same way as the example `PodMonitor` resources.
2. Check the metrics, the dashboards and the alerts in the new backend, while Prometheus keeps scraping the pods.
3. Change the metrics configuration to the `strimziMetricsReporterOtlp` type, with the Collector or the backend as the endpoint.
   The pods roll and start pushing their metrics.
4. Remove the scraping configuration from the Collector, and the `PodMonitor` resources from Prometheus.

The Collector only scrapes the pods during the migration, so a single replica is enough, and the drawbacks described in the motivation are acceptable for that time.
To keep the same series in the new backend, the resource attributes of the pushed metrics should match the labels that the Collector adds when it scrapes the pods.

Users who need to keep feeding an existing Prometheus after the migration can use the Collector the other way around.
It receives the pushed metrics over OTLP, and either exposes them with its `prometheus` exporter or sends them to Prometheus with remote write.
Prometheus can also ingest the pushed metrics directly with its OTLP receiver.

Migrating back is done in the same way, by changing the type back to `strimziMetricsReporter`.

### Testing

The *Reporter* gets integration tests that push metrics to an OpenTelemetry Collector running in a container.
They verify the metric names, the attributes and the resource attributes for both protocols, with and without TLS.

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
- `strimzi-kafka-operator`: adds the `strimziMetricsReporterOtlp` type and generates the configuration for `Kafka`, `KafkaConnect`, `KafkaMirrorMaker2` and `KafkaBridge`, and changes the tracing agent.
  The Kafka container images get the new *Reporter* version, including the exporter dependency.
- `strimzi-kafka-bridge`: adds the OTLP push with the `strimziMetricsReporterOtlp` metrics type, and changes how it disables the metrics exporter of the tracing SDK.

The following projects are not affected:
- `strimzi-kafka-oauth` and `kafka-quotas-plugin`: their metrics already end up in the *Reporter* registry when it is used, so they are pushed as well.

## Out of scope

The following metrics are not covered by this proposal, and OTLP push for them can be addressed separately:
- The metrics of the Strimzi Cluster Operator, Topic Operator and User Operator, which are collected with Micrometer and not with the *Reporter*.
- The metrics of the other Strimzi projects, such as the Strimzi Drain Cleaner and the Strimzi Access Operator.
- Cruise Control, which supports neither the `strimziMetricsReporter` type nor the new `strimziMetricsReporterOtlp` type.
- Kafka Exporter, which is not based on the *Reporter*.
  The re-implementation described in [Proposal 158](./158-kafka-exporter-re-implementation.md) could adopt OTLP push later.

## Compatibility

The new `strimziMetricsReporterOtlp` type is added next to the existing metrics types, which keep their current behavior.
The new *Reporter* and Bridge configurations are optional, and push is disabled by default.
The new type is added to the existing `v1` API, so no new API version is needed.

The change to how tracing disables the metrics exporter of the tracing SDK does not change the tracing behavior.

Metrics pushed over OTLP follow the OpenTelemetry naming conventions described above.
Backends that convert them back to the Prometheus format should produce the same metric names as scraping.
However, the labels depend on how the backend converts resource attributes.
For example, the Prometheus OTLP receiver maps `service.name` and `service.instance.id` to the `job` and `instance` labels, and only promotes other resource attributes to labels when configured to do so.
The Grafana dashboards and Prometheus alerting rules provided by Strimzi rely on the labels added by the example `PodMonitor` resources, so they may need to be adapted for OTLP data.
Providing dashboards for OTLP data is not part of this proposal.

## Rejected alternatives

### Adding OTLP to the `strimziMetricsReporter` type

The OTLP push could be configured with an `otlp` property of the existing `strimziMetricsReporter` type, together with a `prometheusEndpoint.enabled` property to disable the Prometheus endpoint.
This would allow exposing the Prometheus endpoint and pushing at the same time, for example during a migration.
However, the Prometheus endpoint would stay enabled by default even when only push is wanted, and a single type would mix the options of pull and push, which makes the API harder to understand.
The migration is a temporary need that can be addressed in other ways, as described in [Migration](#migration), so the cleaner API was preferred.
