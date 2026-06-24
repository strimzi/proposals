# Alerting for stuck reconciliations

This proposal adds a new metric to help users detect reconciliations which started but never completed.
It follows the discussion in [strimzi-kafka-operator#11634](https://github.com/strimzi/strimzi-kafka-operator/issues/11634).

## Current situation

Strimzi operators already expose metrics about reconciliations, including counters for started, successful, failed, and locked reconciliations, and a timer for completed reconciliation duration.

The Cluster Operator and User Operator also log progress while a reconciliation is running.
For example, the Cluster Operator starts a periodic progress warning in `AbstractOperator.withLock(...)`, and the User Operator does the same through `AbstractControllerLoop.reconcileWrapper(...)`.
The Topic Operator uses a different batching loop, but it also has per-topic reconciliation timer boundaries in `BatchingTopicController`.
These warnings are useful when somebody is already looking at the logs.

But if a reconciliation gets stuck and never completes, it does not become a failed reconciliation.
The completion path is not reached, the reconciliation duration timer is not stopped, and the lock remains held for the affected resource.
Future changes to the same custom resource might then be skipped or requeued because another reconciliation is still in progress.
This can also block important maintenance work such as certificate renewal.

## Motivation

Users should be able to alert on reconciliations which have been running for too long without having to search logs for repeated `Reconciliation is in progress` messages.

This proposal is intended to give Prometheus rules enough information to:

* Identify the affected resource by kind, namespace, and name.
* Calculate how long the reconciliation has been running from the start timestamp.
* Stop matching completed reconciliations, including completed deletion reconciliations.
* Leave the alert threshold to Prometheus rules instead of adding a fixed operator timeout.
* Work with Prometheus alerting rules.

## Proposal

The Strimzi Cluster Operator, Topic Operator, and User Operator will expose a new gauge metric for active reconciliations:

```text
strimzi_reconciliations_in_progress_start_time_seconds{kind="Kafka",namespace="myproject",name="my-cluster"} 1778630400
```

The value is the Unix epoch timestamp, in seconds, when the currently running reconciliation started.
Seconds are used because Prometheus `time()` also returns Unix time in seconds, and because `_seconds` is the standard Prometheus unit suffix.

The internal Micrometer name should follow the existing dot-separated convention.

```text
strimzi.reconciliations.in.progress.start.time.seconds
```

### Metric lifecycle

The metric exists only while a reconciliation is running.
It is set when the resource is accepted for reconciliation and removed from the local `MeterRegistry` when the reconciliation completes.
For the Cluster Operator and User Operator, this means after the reconciliation lock is acquired.
For the Topic Operator, this means after the topic is accepted into the in-flight batch.
If the resource is not accepted for reconciliation, for example because the lock is not acquired or the topic is already in flight, the metric is not created.

Reconciliations that are skipped also do not register as stuck.
For example, when a custom resource has the `strimzi.io/pause-reconciliation` annotation, the reconciliation completes without doing the normal resource work.
The meter is removed when that skipped reconciliation completes, so the alert does not match.

If the reconciliation gets stuck and the completion path is never reached, the metric remains exported with the original start timestamp.
Prometheus can then alert based on elapsed time:

```promql
time() - strimzi_reconciliations_in_progress_start_time_seconds > 6 * 60 * 60
```

The operator will not define a built-in timeout.
Different Strimzi deployments can have different reconciliation durations, so the threshold should be part of the user's Prometheus alerting rule.
For example, one deployment might alert after one hour while another might choose several hours.

Removing the local meter also avoids tombstone values such as `-1`.
The metric should not use an explicit Prometheus sample timestamp; the Unix epoch is only the gauge value.
After Strimzi removes the local meter, the next successful scrape of the same operator endpoint no longer returns the series, so Prometheus marks the series as stale.
For evaluation timestamps after that stale marker, instant-vector selectors no longer return the series, so the alert expression stops matching the old value.
The stale marker is internal to Prometheus and does not require users to match a special metric value in PromQL.
This means alert resolution is driven by the Prometheus scrape and rule evaluation intervals, not by the exact moment when Strimzi removes the local meter.
If Prometheus cannot scrape the operator endpoint at all, endpoint availability should be covered by the existing target health alerts such as `up == 0`.
The local meter cleanup follows the same model already used by per-resource metrics such as `strimzi.resource.state`, where the operator removes local meters when they no longer apply.

The same applies when the custom resource is deleted.
If the deletion reconciliation completes, the meter is removed and the series becomes stale.
If the deletion reconciliation, or an earlier reconciliation blocking the deletion, gets stuck, the metric remains exported and should alert because the operator still has an unfinished reconciliation for that resource.

If the operator restarts, any in-progress metric from the previous process disappears and Prometheus marks the old series stale.
A new reconciliation started by the new operator process will create a new metric value.

### Implementation

The implementation should be a small addition around the existing reconciliation wrappers, not a new scheduler, timeout, or status update path.

A common metrics helper can register the start-time gauge when a reconciliation starts and remove the matching meter when it completes.
The helper would use the existing reconciliation identity as metric labels:

* `kind`
* `namespace`
* `name`

The existing reconciliation paths already have the required identity and lifecycle hooks:

* `Reconciliation` exposes `kind()`, `namespace()`, and `name()`.
* In the Cluster Operator, `AbstractOperator.withLock(...)` starts the progress-warning timer after the lock is acquired and cancels it when the asynchronous reconciliation completes.
* In the User Operator, `AbstractControllerLoop.reconcileWrapper(...)` starts the progress-warning task before calling `reconcile(...)` and cancels it in the `finally` block.
* In the Topic Operator, `BatchingTopicController` already starts and stops per-topic reconciliation timer samples for each `ReconcilableTopic`.

These paths also have access to metrics objects.
`AbstractOperator` has an `OperatorMetricsHolder` through `metrics()`.
`AbstractControllerLoop` exposes a `ControllerMetricsHolder` through its abstract `metrics()` method.
`BatchingTopicController` already has a `TopicOperatorMetricsHolder` field used by the existing Topic Operator reconciliation metrics.
So the new helper can be added to the existing metrics holders instead of passing a new metrics dependency into the reconciliation wrappers.

The new metric should follow the same boundary.
For the Cluster Operator and User Operator, it should be created after the reconciliation lock is acquired and before the reconciliation callable is executed.
For the Topic Operator, it should be created after a topic is accepted for reconciliation and is tracked as in-flight.
It should be removed in the matching completion path that stops the existing reconciliation timer and clears the in-progress state.
If the resource is not accepted for reconciliation, no in-progress metric should be created because that execution path is not actively reconciling the resource.
Skipped reconciliations, such as a paused custom resource, briefly pass through this lifecycle.
The meter is removed when the skipped reconciliation completes, so no long-lived series is produced.

The implementation can reuse the existing `removeMetric(...)` pattern used by other per-resource metrics.
This is important for deleted resources because the desired terminal state is no exported series, not a special value.

Tests should cover the main lifecycle cases:

* The metric is created while a reconciliation is running.
* The metric is removed after success or failure.
* The metric is not created when the resource is not accepted for reconciliation.
* Reconciliations that are skipped, such as for a paused custom resource, do not produce a long-lived metric.
* The metric remains exported while a reconciliation future has not completed.

### Documentation and examples

The Strimzi documentation should describe the new metric and its labels.
The Prometheus alerting examples can include an example rule for stuck reconciliations using `time() - strimzi_reconciliations_in_progress_start_time_seconds`.

The example threshold should be documented as an example only, not as a Strimzi default.

## Affected/not affected projects

This proposal affects:

* `strimzi-kafka-operator`
* Strimzi documentation and metrics examples

The main implementation areas in `strimzi-kafka-operator` are the Cluster Operator, Topic Operator, and User Operator reconciliation lifecycles and their metrics tests.

This proposal does not affect Strimzi custom resource API schemas, Strimzi Metrics Reporter, Kafka broker or client configuration, or operand pod templates.

## Compatibility

This proposal is backward compatible.
Existing reconciliation metrics keep their current names and behavior, and existing users are not required to configure the new alert.

The new metric adds one time series per active reconciliation.
In normal operation, this should be a small number because reconciliations for the same resource are serialized by the reconciliation lock.
Series are removed when reconciliations complete.

## Rejected alternatives

### Reconciliation ID gauge

One option was to expose a gauge with the current or last reconciliation ID and alert when the value does not change for a long time.
This proposal does not use it because the alert has to derive stuck state indirectly from lack of changes.
Additional state, a tombstone value, or more complex PromQL would be needed to distinguish an inactive resource from a stuck reconciliation.

An alert based on `absent()` or `changes()` has a similar problem.
It would need to infer the reconciliation state from series presence or lack of updates instead of alerting directly on the age of the currently running reconciliation.

### Binary in-progress gauge

Another option was to expose a gauge with value `1` while a reconciliation is running and `0` otherwise.
This proposal does not use it because it does not directly expose the reconciliation age.
Alerting would need a range query such as checking whether the gauge stayed at `1` for a long window.

### `LongTaskTimer`

Micrometer's `LongTaskTimer` is conceptually close to this problem because it tracks long-running tasks.
This proposal does not use it because the proposed alerting only needs one value: the start time of the active reconciliation.
`LongTaskTimer` would expose additional active / max / sum-style metrics and would still need the same per-resource cleanup behavior.

### Watchdog timer in the operator

Another option was to add a second timer inside the operator which would log warnings or emit events after a configured threshold.
This proposal does not use it because the threshold is deployment-specific.
Putting the threshold in Prometheus keeps the operator behavior simple and lets users choose alerting windows that fit their environment.

### Kubernetes events or status conditions

Kubernetes events or status conditions could make stuck reconciliations visible without Prometheus.
This proposal does not use them because a stuck reconciliation might not reach the normal status update path, and Kubernetes events are not a durable alerting mechanism.
