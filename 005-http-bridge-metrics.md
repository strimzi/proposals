# HTTP Kafka bridge metrics

This proposal is about defining "how to" expose useful metrics from the HTTP Kafka bridge, "which ones" and a related Grafana dashboard.
It's about exposing HTTP server related metrics on one side, and Apache Kafka clients (producer/consumer) related metrics on the other side.

## Exposing metrics

### Current status

After working on [#423](https://github.com/strimzi/strimzi-kafka-bridge/pull/423), [#426](https://github.com/strimzi/strimzi-kafka-bridge/pull/426) and [#431](https://github.com/strimzi/strimzi-kafka-bridge/pull/431), the bridge is already able to expose metrics in this way:

* HTTP server metrics are provided by Vert.x metrics library and exposed directly in the Prometheus format on the `/metrics` endpoint.
* Apache Kafka metrics are exposed via JMX by the native clients library and then an "embedded" JMX to Prometheus collector is used to expose them on the `/metrics` endpoint.

I also already have a Grafana dashboard which shows these metrics.

The JMX-Prometheus collector is the official one provided by Prometheus project and it's also the one used internally by the [JMX Exporter](https://github.com/prometheus/jmx_exporter) which Strimzi is using as an "agent" for exposing all the Kafka metrics to Prometheus.
It means that it uses a JMX Exporter configuration (YAML) file for mapping (filtering, relabeling, hiding ...) from JMX to Prometheus.

For the `Kafka` custom resource we already have such a configuration file in the `metrics` section which is used to enable/disable and configure metrics.
There is no such integration in the `KafkaBridge` custom resource right now.

Leaving the way bridge metrics are exposed right now means:

* no HTTP server metrics are exposed via JMX. There is no way to filter, relabeling, hiding them; no way for customization.
* the `metrics` field to add in the `KafkaBridge` custom resource is going to configure the Kafka clients metrics only which could create confusion for the users.
* JMXTrans integration cannot leverage on bridge HTTP server metrics but only on the Kafka ones.

## Metrics to expose

### HTTP server metrics

The HTTP metrics could be:

* The number of opened HTTP connections to the bridge.
* Requests being processed.
* Requests processed/secs grouped by HTTP method.
* The requests rate in total and even grouped by response codes (2XX, 4XX, 5XX).
* Bytes received /sec and Bytes sent /sec.

Furthermore, the HTTP requests/sec metrics coming from Vert.x bring labels like `code` (HTTP response code), `local` (bridge server listening), `remote` (client issuing the request), `method`, and `path` that can be useful for some filtering to define following metrics:

* SEND to `/topics/<topic>`
* SEND TO PARTITIONS to `/topics/<topic>/partitions/<partition>`
* CREATE CONSUMER to `/consumers/<group>`
* SUBSCRIBE to `/consumers/<group>/instances/<consumer>/subscription`
* DELETE CONSUMER to `/consumers/<group>/instances/<consumer>`
* POLL to `/consumers/<group>/instances/<consumer>/records`
* COMMIT to `/consumers/<group>/instances/<consumer>/offsets`

Even JVM and GC metrics are exposed out-of-box.

### Kafka clients metrics

Useful metrics about Kafka part could be the following.

* Number of consumers
* Producer:
    * The average number of records sent per second (grouped by topic)
    * The number of outgoing bytes sent to all brokers per second (grouped by topic)
    * The averate number of records per second that resulted in errors (grouped by topic)
* Consumer:
    * The average number of records consumed per second (grouped by clientId-topic)
    * The average number of bytes consumed per second (grouped by clientId-topic)
    * Partitions assigned (grouped by clientId)

They all bring labels like the `clientId` and the `topic`.

### Grafana Screenshots

![Metrics 01](/images/005-http-bridge-metrics/metrics_01.png)

![Metrics 02](/images/005-http-bridge-metrics/metrics_02.png)

![Metrics 03](/images/005-http-bridge-metrics/metrics_03.png)

## Enabling metrics

We should allow users to enable/disable the metrics exposure through Prometheus on the `/metrics` endpoint.
The proposal is to add a boolean `spec.metrics` property in the `KafkaBridge` custom resource, which is `false` as default when not specified.

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9092
  http:
    port: 8080
  metrics: true
```

The Cluster Operator would set an env var `BRIDGE_METRICS_ENABLED` based on the above property.
The bridge should configure the Vert.x options for HTTP metrics and the JMX - Prometheus collector for Kafka metrics accordingly to this env var; it should also expose or not the `/metrics` endpoint.

## Rejected alternatives

### Proposal 1

This proposal is about exposing the bridge HTTP server metrics via JMX first.
It will be consistent with having the Kafka clients metrics via JMX as well.

Removing the embedded JMX to Prometheus collector and using the same approach as we do for the Kafka brokers metrics, using the JMX Exporter as an "agent".

* both HTTP server and Kafka clients metrics are exposed through JMX.
* the `metrics` field to add in the `KafkaBridge` custom resource can be used for configuring both bridge HTTP and Kafka metrics.
* JMXTrans integration can leverage all the bridge metrics.
* consistency on how we are providing metrics in Strimzi: JMX first, JMX Exporter to make them available via Prometheus, JMXTrans for more different outputs.

### Proposal 2

It's quite the same as Proposal 1 with the only difference to leave the embedded JMX to Prometheus collector inside the bridge itself instead of using an external JMX Exporter running as an agent.

# Next steps

Agreeing on changing or not the way metrics are exposed choosing between current status, proposal 1 and proposal 2.
Agreeing on the current metrics used for the Grafana dashboard.
Building a final Grafana dashboard for the bridge.
Integrating bridge metrics configuration in the Strimzi Cluster Operator.