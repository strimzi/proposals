# Monitoring of custom resources

In https://github.com/strimzi/strimzi-kafka-operator/issues/9802 there was a discussion about how to monitor the state of a custom resources (CR) like `KafkaTopic`, `KafkaUser`, ... .
For a simple and Kubernetes native monitoring, [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) (KSM) can be used which utilizes e.g. the status object of a Kubernetes resource.

## Current situation

All Strimzi operators (Cluster, User and Topic operators) provide metrics in Prometheus format.
However, currently, only the Cluster Operator provides a metric (`strimzi_resource_state`) describing the state of the custom resources (whether they are ready or not).
It provides this metric for the `Kafka`, `KafkaConnect`, `KafkaBridge`, KafkaMirrorMaker`, `KafkaMirrorMaker2`, and `KafkaRebalance` resources.
It does not currently provide this metric for `KafkaNodePool`, `KafkaConnector`, and `StrimziPodSet` custom resources.
The User and Topic Operators currently do not provide this metric for the `KafkaUser` and `KafkaTopic` resources.
TODO need input for the status of other CRs and their metrics, e.g. `Kafka`, `StrimziPodSet`, `KafkaRebalance`...

## Motivation

Strimzi already provides example configurations being compatible with the prometheus-operator via e.g. `PodMonitor` for scraping the metrics and e.g. `PrometheusRules` for alerting.

But in general, if a deployed Kubernetes resource is having a problem or a deprecated configuration is used, you want to get notified about that and can fix the problem, afterward.
Other tools like Flux handles the monitoring of CRs via KSM like documented in their [documentation](https://fluxcd.io/flux/monitoring/custom-metrics/).
In addition, Flux provides an [example repository](https://github.com/fluxcd/flux2-monitoring-example) for configuring KSM as part of [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart.
But the required configuration is just being part of the ConfigMap used by KSM as shown by @im-konge in this [issue](https://github.com/strimzi/strimzi-kafka-operator/issues/10276#issuecomment-2276088493).

## Proposal

So in order to have a monitoring in place for all CRs and implementing them in a scalable and Kubernetes native way KSM can be used.
KSM is (mostly) deployed via a simple Kubernetes `Deployment` and querries the Kubernetes api for e.g. the status object of a Kubernetes resource indicating that the CR having a problem or contains a deprecated configuration.
This can also be done for CRs exclusively by using the `--custom-resource-state-config` flag which is helpful for e.g. `KafkaTopic`, `KafkaUser`, ... CRs.
During the initial phase when implementing this proposal, the support in the examples will be implemented at least for the existing resources for which Strimzi publishes already existing metric(s), additionally for `KafkaUser` and `KafkaTopic` custom resources.

### Deployment

Strimzi should provide guidance for existing fields in status object in order to indicate if a CR is having a problem or uses a deprecated configuration in the official documentation.
By this users can extend monitoring of CRs on their own requirements.

In general, an example ConfigMap should be provided in the examples (sub-)directory `metrics/kube-state-metrics` of [strimzi/strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator/tree/main/examples/metrics/kube-state-metrics) to specify a good starting configuration for users.
The ConfigMap should look like e.g.:

```
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
  name: strimzi-kube-state-metrics-config
data:
  config.yaml: |
    spec:
      resources:
      - groupVersionKind:
          group: kafka.strimzi.io
          version: v1beta2
          kind: KafkaTopic
        metricNamePrefix: strimzi_kafka_topic
        metrics:
          - name: resource_info
            help: "The current state of a Strimzi kafka topic resource"
            each:
              type: Info
              info:
                labelsFromPath:
                  name: [ metadata, name ]
            labelsFromPath:
              exported_namespace: [ metadata, namespace ]
              partitions: [ spec, partitions ]
              replicas: [ spec, replicas ]
              ready: [ status, conditions, "[type=Ready]", status ]
              generation: [ status, observedGeneration ]
              topicId: [ status, topicId ]
              topicName: [ status, topicName ]
```

For YAML users the ConfigMap works fine when creating a dedicated deployment.

This works also fine when implement the configuration inside [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart.

For (all other) Helm users, this should be left to the user implementing the KSM based monitoring, as there are many different Helm charts but all of them will handle the same ConfigMap in the end.
This could be explained in a Blog Post with a static version of the [prometheus-community kube-state-metrics Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-state-metrics/) at least once.

In addition, example `PrometheusRules` should be defined for all CRs for having a problem or uses a deprecated configuration in a simple way, so that a cluster operator can have a look for an error message in the status object of the CR.
[This PR](https://github.com/strimzi/strimzi-kafka-operator/pull/10277) could be used as an implementation idea.

## Affected/not affected projects

The only affected projects are:
- CO

## Compatibility

Strimzi provides the `strimzi_resource_state` metric(s) implemented in CO.

The previously elaborated metrics(s) should be deprecated and replaced in favour of KSM based metrics.
This also applies for PrometheusRules which should be replaced.
The proposed way would be implementing the KSM and deprecating the current metrics in Strimzi CO in version 0.45 and removing them in version 0.49 to give users enough time adjusting their monitoring if needed.
So there is no immediate impact for users and enough time for migration.

## Rejected alternatives

Implementing them inside each operator (CO, TO, UO) which leads to enormous metrics exported by each component as discussed in this [issue](https://github.com/strimzi/strimzi-kafka-operator/issues/9802).
