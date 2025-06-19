# Add KIP-714 support to metrics-reporter

Apache Kafka 3.7.0 introduced the ability for Kafka clients to send their metrics to brokers via [KIP-714](https://cwiki.apache.org/confluence/display/KAFKA/KIP-714%3A+Client+metrics+and+observability).

The Kafka clients have this feature built-in and enabled by default. However for this feature to be usable, administrators must implement and provide a broker-side plugin to collect the metrics. The plugin must be a [`MetricReporter`](https://kafka.apache.org/40/javadoc/org/apache/kafka/common/metrics/MetricsReporter.html) instance that also implements the [`ClientTelemetry`](https://kafka.apache.org/40/javadoc/org/apache/kafka/server/telemetry/ClientTelemetry.html) interface.

Then administrators must set metrics subscriptions to define the metrics clients will send. There are no default subscriptions. Subscriptions can be set, updated and deleted at runtime via the `kafka-configs.sh` or `kafka-client-metrics.sh` tools, or via the `Admin` API. For example:
```sh
./bin/kafka-client-metrics.sh --bootstrap-server localhost:9092 \
  --alter --name topic-metrics \
  --metrics org.apache.kafka.producer.topic. \
  --interval 30000
```
A subscription is composed of:
- A name: This can be provided by the administrators or generated using the `--generate-name` flag. This is used to unique identify the subscription to describe, alter or delete it.
- A list of metric prefixes: They indicate which metrics the clients should send to the brokers. It can be set to `*` to request all metrics.
- An optional list of client matching filters: They indicate which clients this subscription is for. The filters are `client_id`, `client_instance_id`, `client_software_name`, `client_software_version`, `client_source_address`, `client_source_port`. If not specified, the subscription applies to all clients.
- An interval: This indicates how often clients matching this subscription should send their metrics. The interval is in milliseconds and it must be between 100 and 3600000 (1 hour). If not specified, it defaults to 300000 (5 minutes).

This proposes adding support for KIP-714 to the server-side metric reporter, `ServerKafkaMetricsReporter`.

## Motivation

This proposal shares its motivations with KIP-714. Monitoring applications is essential to ensure they function correctly. While brokers emit themselves a lot of metrics, it's often necessary to get client metrics to diagnose issues. However collecting metrics from all applications can be challenging. This can be due to multiple reasons:
- Applications deployed in distributed and heterogeneous environments
- Applications run and owned by separate teams
- Kafka clients embedded in complex applications

When clients are able to send their metrics to broker this eases their collection, and greatly simplifies diagnosing issues. The mechanism to set subscriptions also allows users to precisely adjust metrics at runtime to collect the most relevant metrics. 

## Proposal

Now that we separated the metrics-reporter into client-side and server-side modules ([Proposal 96](./096-split-metrics-reporter-into-modules.md)), we can make the server-side module implement the `ClientTelemetry` interface to support KIP-714.

### Naming

The metrics reporter will expose client metrics with the `clients_` prefix. For example if a subscription is created for `org.apache.kafka.producer.topic` and a client emits the `org.apache.kafka.producer.topic.byte.rate` metric, it will be exposed as `clients_org_apache_kafka_producer_topic_byte_rate`.

When a client retrieves its metric subscriptions, it is assigned a unique client instance Id (UUID). This Id is used to identify all the metrics for a specific client instance, even if it sends them to different brokers. This Id is always added as a label using the `client_instance_id` name to all metric series.

### Configurations

The `client_instance_id` label is nice to identify all metrics from a client but does not enable to identify which client it is. Fortunately every time clients send metrics, they attach a bunch of metadata to identify themselves. This metadata is an [`AuthorizableRequestContext`](https://kafka.apache.org/40/javadoc/org/apache/kafka/server/authorizer/AuthorizableRequestContext.html) object that contains the client address, client Id, correlation Id, listener name, principal, request type, request version and security protocol from the client.

A few of these fields are of very low value for metrics:
- request type: This is always `72` which is the [`PushTelemetry`](https://kafka.apache.org/protocol#The_Messages_PushTelemetry) API key.
- request version: This is the version of the `PushTelemetry` request. As of Kafka 4.0, it is always `0`.
- correlation Id: This starts at zero for each client and keeps increasing every time the client sends metrics. This shouldn't be used as a label as it would effectively create a new metric series each time.

The reporter can convert the other fields as labels and add them to the metric series. In many cases it does not make sense to add all of them, for example if the cluster is behind a proxy the client address will always be the same. Also using all of them can create high cardinality labels series which can be problematic in Prometheus.

I propose introducing a configuration to select the metadata fields to use as labels: 
- `prometheus.metrics.reporter.telemetry.labels`: List of label names in client metrics. The valid names are `client_id`, `listener_name`, `security_protocol`, `principal`, `client_address`. This defaults to `client_id`. This configuration is reconfigurable at runtime.

### Client metrics life cycle

When the reporter receives metrics from a client, it converts them and adds them into Prometheus registry, ready to be collected when the `GET /metrics` endpoint is scraped. The subscription specifies an interval for the client to send metrics. However when metrics are received, the interval or any details about the subscription that caused this emission is not available. 

So we need a mechanism to remove metrics from the registry when a client stops emitting metrics, otherwise if a client shuts down, the metrics reporter will keep reporting its metrics. As we don't have the interval, the simplest approach is to delete client metrics whenever they are scraped. While this may create gaps in metric series, this ensures that metrics collected are always valid. 

Typically the subscription interval and the scrapping interval should be of the same order of magnitude. [Prometheus default scaping interval](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration-file) is 1 minute, and the [default metric interval](https://github.com/apache/kafka/blob/trunk/server/src/main/java/org/apache/kafka/server/metrics/ClientMetricsConfigs.java#L89) is 5 minutes, so this should work fine and not cause too much confusion.

### Monitoring client metric collection

Kafka brokers expose a number of metrics to monitor the client metrics collection.

- `kafka_server_client_metrics_instance_count`: This is the number of clients currently emitting metrics.
- `kafka_server_client_metrics_plugin_error_count{client_instance_id="<ID>"}`/`kafka_server_client_metrics_plugin_error_rate{client_instance_id="<ID>"}`: The count and rate of errors while handling client metrics. An error means the metrics reporter threw an exception while converting client metrics to Prometheus.
- `kafka_server_client_metrics_plugin_export_count{client_instance_id="<ID>"}`/`kafka_server_client_metrics_plugin_export_rate{client_instance_id="<ID>"}`: The count and rate of calls to the metric reporter to convert client metrics to Prometheus.
- `kafka_server_client_metrics_plugin_export_time_avg{client_instance_id="<ID>"}`/`kafka_server_client_metrics_plugin_export_time_max{client_instance_id="<ID>"}`: The average and maximum time the metrics reporter takes to convert client metrics to Prometheus.
- `kafka_server_client_metrics_throttle_count{client_instance_id="<ID>"}`/`kafka_server_client_metrics_throttle_rate{client_instance_id="<ID>"}`: `PushTelemetry` requests can be throttled if they violate quotas defined by the cluster administrator. The count and rate of throttled `PushTelemetry` requests.
- `kafka_server_client_metrics_unknown_subscription_request_count`/`kafka_server_client_metrics_unknown_subscription_request_rate`: The count and rate of `PushTelemetry` requests received from clients that have not retrieved their subscriptions.

Administrators should monitor these metrics when they set subscriptions.

### Updates to strimzi-kafka-operator

The `metricsConfig` definition will support the new configuration if the `type` is set to `strimziMetricsReporter`, via a new value `telemetryLabels`. For example:
```yaml
metricsConfig:
  type: strimziMetricsReporter
  values:
    allowList:
      - "kafka_log.*"
      - "kafka_network.*"
    telemetryLabels:
      - "client_id"
      - "principal"
```
If this value is not set, it will default to `client_id` like in the metrics reporter.

## Affected/not affected projects

This affects metrics-reporter and strimzi-kafka-operator.

## Compatibility

This feature will only have an effect when subscriptions are set. If none are set, which is the default, there should be no change in behavior.

## Rejected alternatives

### Configuration to disable/enable client metrics

- Administrators must define subscriptions to specify the clients and the metrics they are interested in collecting. By default there are no subscriptions, meaning no clients are sending metrics. For that reason, I don't propose adding a configuration to the metrics-reporter to enable or disable client metrics collection, this should be managed via the subscriptions.