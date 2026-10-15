# Configurable delay after Kafka node restarts during rolling updates

This proposal adds an optional delay that the Cluster Operator waits after a restarted Kafka node becomes ready and before it considers restarting the next node.
The purpose is not to make rolling updates slower in general, but to give the restarted node time to show that it stays in the in-sync replicas, so that the next restart does not push partitions below `min.insync.replicas` if the previous node turns out not to be stable yet.
The delay is set operator-wide with a new environment variable, `STRIMZI_POST_KAFKA_RESTART_DELAY_MS`, and overridden per cluster with a new annotation on the `Kafka` resource, `strimzi.io/post-kafka-restart-delay-ms`.
Both default to no delay, so the current behaviour is unchanged unless a user opts in.

## Current situation

Rolling updates of Kafka nodes are performed by the `KafkaRoller`.
Before restarting a node, it checks with `KafkaAvailability` that no partition would drop below `min.insync.replicas`.
After the restart, it waits for the pod to become ready and immediately moves on to the next node, often in less than a second.

The availability check is a point-in-time check.
It passes as soon as the restarted node is back in the in-sync replicas, and there is no option to wait before running it for the next node.

## Motivation

A node that has just rejoined the in-sync replicas can drop out again shortly afterwards, for example because of slow disks or log recovery that is still catching up.
The availability check cannot see this coming, so the next restart goes ahead.
If the node drops out while the next one is down, partitions with replicas on both nodes fall below `min.insync.replicas` and `acks=all` producers get `NOT_ENOUGH_REPLICAS`.

This happened during a Kubernetes node upgrade where two nodes hosting brokers A and B were drained in parallel and the Strimzi Drain Cleaner annotated both pods.
The `KafkaRoller` restarted A, A became ready and rejoined the in-sync replicas, and about 10 seconds later B was restarted.
About 20 seconds after B began its controlled shutdown, the controller fenced A because of a slow disk, and producers got `NOT_ENOUGH_REPLICAS` for a few minutes.
Waiting for the in-sync replicas would not have helped, because A was already in them when B was checked.
A delay of 30 seconds would have deferred the restart of B.

A delay between restarts was already part of the new `KafkaRoller` design in [PR #103 in strimzi/proposals](https://github.com/strimzi/proposals/pull/103), which was closed without being merged.
This proposal adds only that piece to the existing `KafkaRoller`.

## Proposal

### Behaviour

When the effective delay is greater than zero, the `KafkaRoller` waits for that time after a restarted pod becomes ready and before it runs the availability check for the next node.
The delay is also applied after the last restarted node, so that a following rolling update, in the same or a later reconciliation, does not restart another node right away.

The delay applies to every restart the `KafkaRoller` performs, including those triggered by Kafka version upgrades and downgrades, configuration changes, CA renewals, the `strimzi.io/manual-rolling-update` annotation, and the Strimzi Drain Cleaner.
It is not applied when a node is only reconfigured without a restart, or when a restart fails.
It is not preserved across a Cluster Operator restart.
It does not cover pods deleted outside the Cluster Operator, or Kafka Connect and MirrorMaker 2, which use a different roller.

The reconciliation stays in progress during the delay, so each restarted node adds the delay to the duration of the rolling update, and a rolling update can continue past the end of a maintenance time window.

### Environment variable `STRIMZI_POST_KAFKA_RESTART_DELAY_MS`

A new Cluster Operator environment variable sets the delay for all Kafka clusters managed by the operator.

| Property | Value |
|:---------|:------|
| Name | `STRIMZI_POST_KAFKA_RESTART_DELAY_MS` |
| Type | Non-negative integer, milliseconds |
| Default | `0`, no delay |
| Scope | All `Kafka` clusters managed by the operator |

The name follows the existing duration settings such as `STRIMZI_OPERATION_TIMEOUT_MS`, which all use milliseconds and the `_MS` suffix.
It includes `KAFKA` because it applies only to Kafka nodes, and `POST` because the delay is applied after a restart.

An operator-wide setting fits the motivating case, where a Kubernetes node upgrade drains nodes under every cluster the operator manages.

### Annotation `strimzi.io/post-kafka-restart-delay-ms`

A new annotation on the `Kafka` resource overrides the operator-wide value for that cluster.

| Property | Value |
|:---------|:------|
| Name | `strimzi.io/post-kafka-restart-delay-ms` |
| Resource | `Kafka` |
| Type | Non-negative integer, milliseconds |
| Default | Not set, the environment variable applies |

This lets users enable the delay on the clusters where a restarted node needs time to stabilise, for example production clusters, and keep immediate restarts on others.

The name mirrors the environment variable, so the two settings are recognisable as a pair in the documentation and in the code.

The annotation is read when the `KafkaRoller` is created, so a change during a rolling update takes effect from the next reconciliation.

### Precedence

The annotation, when set, always wins over the environment variable for its cluster.

| Environment variable | Annotation | Effective delay for the cluster |
|:---------------------|:-----------|:--------------------------------|
| Not set | Not set | None, current behaviour |
| Set | Not set | The environment variable value |
| Not set | Set | The annotation value |
| Set | Set | The annotation value, including `0`, which disables the delay for that cluster |

If the annotation value is not a non-negative integer, the operator logs a warning and treats the annotation as not set.

### Implementation

- `ClusterOperatorConfig` gets a `POST_KAFKA_RESTART_DELAY_MS` parameter backed by the environment variable.
- `Annotations` gets an `ANNO_STRIMZI_IO_POST_KAFKA_RESTART_DELAY_MS` constant.
- `KafkaReconciler` and `CaReconciler`, the two places that create a `KafkaRoller`, resolve the effective delay from the annotation with the environment variable as fallback and pass it to the roller.
- `KafkaRoller` takes the delay as a constructor parameter and sleeps for it after a restarted pod becomes ready, logging an `INFO` message with the pod name and the delay.

No new state is persisted and no new coordination between reconciliations is introduced.
Unit tests cover the environment variable parsing, the delay being applied after each restart and not after reconfigurations or failed restarts, and the annotation precedence including `0` and invalid values.

The environment variable is implemented in [PR #13212 in strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator/pull/13212).
The annotation can be added to the same pull request or to a follow-up one.

## Affected/not affected projects

Only the Cluster Operator in `strimzi-kafka-operator` is affected.
The Strimzi Drain Cleaner is not changed, since it only annotates pods and the delay is applied by the `KafkaRoller`.
Kafka Connect, MirrorMaker 2, the HTTP Bridge, the Topic Operator, and the User Operator are not affected.

## Compatibility

The change is fully backwards compatible.
The environment variable defaults to `0` and the annotation is optional, so existing deployments behave as before.
A future replacement of the `KafkaRoller` can keep both settings with the same meaning.
