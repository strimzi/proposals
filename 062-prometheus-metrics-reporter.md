# Prometheus Metrics Reporter

I propose updating the way Kafka metrics are collected by implementing a metrics reporter that directly exposes metrics in the Prometheus format.

PoC: https://github.com/mimaison/prometheus-metrics-reporter


## Current situation

Metrics are a critical aspect of monitoring Kafka. Today, this is the way metrics from Kafka are collected. First Kafka creates metrics objects in memory. For historical reasons Kafka uses two different libraries for metrics:
- a home grown library, `org.apache.kafka.common.metrics`. We’ll refer to metrics created via this as _KafkaMetrics_. This is used on the client side and for common metrics on the server side too.
- Yammer, `com.yammer.metrics.metrics-core`. This library is the predecessor of [Dropwizard](https://metrics.dropwizard.io/). We’ll refer to metrics created via this as _YammerMetrics_. This is only used on the broker side.

For each type of metrics, Kafka exposes a reporter interface to expose metrics to monitoring systems. KafkaMetrics use [org.apache.kafka.common.metrics.MetricsReporter](https://kafka.apache.org/35/javadoc/org/apache/kafka/common/metrics/MetricsReporter.html) and YammerMetrics use `kafka.metrics.KafkaMetricsReporter` which is not officially part of the public API. Kafka has built-in metrics reporter implementations for JMX for both types.

At the moment Strimzi relies on these default JMX reporters and uses [jmx_exporter](https://github.com/prometheus/jmx_exporter) which is a Java agent that retrieves metrics via JMX and exposes them over an HTTP endpoint in the prometheus format. Then Prometheus is configured to scrape that endpoint to retrieve the Kafka metrics.

![Current Situation](images/052-current.png)

- `org.apache.kafka.common.metrics.JmxReporter` is the reporter implementation for the Kafka Metrics.
- `org.apache.kafka.server.metrics.FilteringJmxReporter` is the reporter implementation for the Yammer Metrics. It’s named `FilteringJmxReporter` because it extends the JmxReporter class from the Yammer library and adds an option to select the metrics to report.


## Motivation

I propose updating the metrics collection pipeline for the following reasons:

1. The current metrics collection pipeline is pretty convoluted. We have metrics reporters first exposing metrics via JMX before using a Java agent to expose them again via HTTP. Using metrics reporters to directly expose metrics to Prometheus would significantly simplify it by removing JMX from the picture and removing jmx_exporter.
1. Each component along the pipeline has its own configurations and specificities. For example, since Kafka 3.4.0, it’s possible to disable the JmxReporter for KafkaMetrics by setting [auto.include.jmx.reporter](https://kafka.apache.org/documentation/#brokerconfigs_auto.include.jmx.reporter) to `false`. It is currently not possible to disable `FilteringJmxReporter` (it’s something I plan to address in the near future). Also both jmx_exporter and the built-in reporter allow selecting metrics to collect. 
1. The jmx_exporter Java agent supports complex metrics mapping rules. These rules allow renaming metrics and it can make it hard to investigate issues as metrics could have different names. This prevents using metrics like an API, whether this is to build grafana dashboards or for the operator to rely on metrics.
1. Due to the complex rules, it performs badly when there’s a very large number of metrics due to a lot of topic/partitions.


## Proposal

The proposal is to build metrics reporters that directly exposes metrics via an HTTP endpoint in the Prometheus format. This will be a new project/repository under the Strimzi organization.

![Proposal](images/052-proposal.png)

Today to enable jmx_exporter, users use:
```
metricsConfig:
  type: jmxPrometheusExporter
  valueFrom:
    configMapKeyRef:
      name: kafka-metrics
      key: kafka-metrics-config.yml
```

To enable metric reporters, they will instead use this for example:
```
metricsConfig:
  type: strimziMetricsReporter
  config:
    allowList: ".*"
```

Reporters will expose the following configurations:
- `prometheus.metrics.reporter.port`: The HTTP port to expose the metrics. Default: `8080`
- `prometheus.metrics.reporter.allowlist`: A comma separated list of regex patterns to specify the metrics to collect. Default: `.*`. The patterns must match the metric names emitted by the reporter.

Strimzi should set the port to 9404 to make the migration from jmx_exporter easy.

The reporter will also export JVM metrics similar to the ones exported by jmx_exporter. These are provided by the [Hotspot exports](io.prometheus.client.hotspot) from the Prometheus Java client.

This proposal ignores supporting HTTPS as today Strimzi does not allow configuring it with jmx_exporter. Note that jmx_exporter 0.19.0 added support for HTTPS. If needed we can add it now or later to the reporter.

This proposal will produce an implementation for each type of metrics reporter.
- `KafkaPrometheusMetricsReporter` usable on brokers (for Kafka metrics) and on clients (including Connect and Streams)
- `YammerPrometheusMetricsReporter` usable on brokers (for Yammer metrics)

The reporter for Kafka metrics will be usable outside of Strimzi by clients. To do so they will need to set the `metric.reporters` configuration to `KafkaPrometheusMetricsReporter` and set the reporter configurations accordingly.

## Affected/not affected projects

This affects `strimzi-kafka-operator `. Reporters are not usable with ZooKeeper and currently don't work well with client side application like Kafka Connect, MirrorMaker and the HTTP bridge. So Strimzi should keep support for jmx_exporter. We may reconsider dropping support once ZooKeeper is not supported anymore and if improvements in Apache Kafka make reporters usable with client side applications.

The reporter will be a new project and should be usable without Strimzi. As it's new code, it should be first release as a preview feature (disabled by default). Once KRaft becomes the preferred deployment option, we should make the reporter the recommended option to expose metrics for brokers and controllers.

In `strimzi-kafka-operator`, we will need new examples demonstrating how to use the reporter, new grafana dashboards with metric names from the reporter.

The OAuth plugin can be updated to rely on this reporter. Note that this will still be a workaround (like the current mechanism it uses to exports its metrics today) and that a much better approach for plugins to expose metrics will be provided by [KIP-877](https://cwiki.apache.org/confluence/display/KAFKA/KIP-877%3A+Mechanism+for+plugins+and+connectors+to+register+metrics).

## Compatibility

Differences with jmx_exporter metrics:

- The reporter does not compute 1/5/15 minute rate, mean, max, min, stddev metrics. It's preferable to compute them in Prometheus instead of on the client side.
- The reporter is missing the `kafka_server_app_info_starttimems` metric with the client/broker id label. (Due to [KAFKA-15186](https://issues.apache.org/jira/browse/KAFKA-15186))
- Kafka expose some non-numeric metrics. Prometheus only supports numeric values for metrics. Using jmx_exporter it's possible with rules to move the values into labels and still retrieve them in prometheus. With this proposal non-numeric metrics will be ignored and not exposed. I plan to raise a KIP in Kafka to provide alternative to non-numeric metrics. For example there is already [KIP-972](https://cwiki.apache.org/confluence/display/KAFKA/KIP-972%3A+Add+the+metric+of+the+current+running+version+of+kafka) in progress to address some of them.

Assuming jmx_exporter does not have any rules, this is the other main metric change:

- With the reporter, the `name` field is put directly in the metric name. For example this MBean, `kafka.server:type=ZooKeeperClientMetrics,name=ZooKeeperRequestLatencyMs` with the `Count` attribute is converted to `kafka_server_zookeeperclientmetrics_zookeeperrequestlatencyms_count`. By default jmx_exporter keeps the name as a label, `kafka_server_zookeeperclientmetrics_count{name="ZooKeeperRequestLatencyMs",}`.

### Comparison for broker metrics

Actually with the example rules from [kafka-metrics.yaml](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/kafka-metrics.yaml), jmx_exporter generates metric names very similar to the reporter.

See https://gist.github.com/mimaison/335bc52bd5fb39097b7e6865c8cd1bea that contains the output from jmx_exporter with the default [metrics.yaml](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/kafka-metrics.yaml) rules, and the proposed metrics reporter with the following configuration.

```
prometheus.metrics.reporter.allowlist=kafka_cluster.*,kafka_controller.*,kafka_log.*,kafka_network.*,kafka_server_(brokertopicmetrics|delayedoperationpurgatory|fetchsessioncache|kafkarequesthandlerpool|kafkaserver|replicaalterlogdirsmanager|replicafetchermanager|replicamanager|sessionexpirelistener|socket_server_metrics|zookeeperclientmetrics).*,kafka_utils.*
```

Both files have been sorted and the comments have been removed so they are easy to compare.

If we also remove the values, doing a diff highlights the following differences:

- The JMX metrics only exist with jmx_exporter.
    ```
    jmx_config_reload_failure_created
    jmx_config_reload_failure_total
    jmx_config_reload_success_created
    jmx_config_reload_success_total
    jmx_exporter_build_info{version="0.19.0",name="jmx_prometheus_javaagent",}
    jmx_scrape_cached_beans
    jmx_scrape_duration_seconds
    jmx_scrape_error
    ```
    This is expected as JMX is not used with the reporter.

- Via JMX a number of `java_lang` metrics are emitted. These are not retrieved by Prometheus Hotspot exports.

- The `PerSec` suffix is removed by jmx_exporter rules so a few metrics have slightly different names, for example:
    ```
    kafka_controller_controllerstats_uncleanleaderelections_total         # jmx_exporter
    kafka_controller_controllerstats_uncleanleaderelectionspersec_total   # reporter

    kafka_network_requestmetrics_errors_total{request="UpdateMetadata",error="NONE",}        # jmx_exporter
    kafka_network_requestmetrics_errorspersec_total{request="UpdateMetadata",error="NONE",}  # reporter
    ```
    This is due these [mapping rules](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/kafka-metrics.yaml#L136-L150). 

- An underscore `_` is added in front of the `percent` suffix by jmx_exporter rules so a few metrics have slightly different names, for example:
    ```
    kafka_network_processor_idle_percent{networkProcessor="0",} # jmx_exporter
    kafka_network_processor_idlepercent{networkProcessor="0",}  # reporter
    ```
    This is due these [mapping rules](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/kafka-metrics.yaml#L127-L135). 

- The `count` suffix is used instead of `total` for some Counters by jmx_exporter:
    ```
    kafka_network_acceptor_acceptorblockedpercent_count{listener="PLAINTEXT",} # jmx_exporter
    kafka_network_acceptor_acceptorblockedpercent_total{listener="PLAINTEXT",} # reporter
    ```
    Prometheus recommends using the `total` suffix and it's actually automatically injected by the Java library. The example jmx_exporter rules correctly replace the suffix in most metrics (like in the metrics mentioned above with `PerSec`) but they don't match all metrics hence the inconsistency.

- Some labels are wrapped twice in quotes by jmx_exporter
    ```
    # jmx_exporter
    kafka_log_logcleanermanager_uncleanable_bytes{logDirectory="\"/tmp/kafka-logs\"",}
    kafka_log_logcleanermanager_uncleanable_partitions_count{logDirectory="\"/tmp/kafka-logs\"",}
    kafka_log_logmanager_logdirectoryoffline{logDirectory="\"/tmp/kafka-logs\"",}
    # reporter
    kafka_log_logcleanermanager_uncleanable_bytes{logDirectory="/tmp/kafka-logs",}
    kafka_log_logcleanermanager_uncleanable_partitions_count{logDirectory="/tmp/kafka-logs",}
    kafka_log_logmanager_logdirectoryoffline{logDirectory="/tmp/kafka-logs",}
    ```
    I'm guessing this is done to support paths containing special characters. In the context of Strimzi I don't think this is necessary.


In terms of performance, in my very limited testing the reporter is much faster than jmx_exporter. This is not a benchmark or rigorous by any means, I've simply been running `time curl --head http://localhost:{PORT}/metrics` against a single broker on my laptop with both the reporter (on port 8080) and jmx_exporter (on port 9090) running. Since each partition has metrics, I used the number of partition as a proxy to increase the total number of metrics.

| # of partitions | jmx_exporter | reporter |
| --- | --- | --- |
| 1 | 600ms | 100ms |
| 500 | 900ms | 300ms |
| 1000 | 1300ms | 400ms |
| 2000 | 2000ms | 800ms | 

## Rejected alternatives

- I considered using [Micrometer](https://micrometer.io/) in the reporter. The benefit is that it would allow exporting metrics to different monitoring systems. The issue is that it requires converting the Kafka and Yammer metrics into the Micrometer format and then have Micrometer export that to Prometheus. As each metric library has its own characteristics, chaining several conversions can lead to slightly different semantics. Finally Prometheus seems to be the leading monitoring solution in the Kubernetes ecosystem, and I expect most other monitoring tools to integrate this it. For these reasons I decided to not use Micrometer. Ideally Kafka would export its metrics via Micrometer. It's something I've started to explore but it is definitively a very difficult task.
