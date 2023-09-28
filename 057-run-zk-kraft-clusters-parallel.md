# Allow running ZooKeeper and KRaft based clusters in parallel

This proposal is about changing the way the current `UseKRaft` feature gate works.
It changes the way the Strimzi Cluster Operator handles ZooKeeper and KRaft based clusters when it is enabled.

## Current situation

Currently, when the `UseKRaft` feature gate is enabled (together with the required `KafkaNodePools` one), the operator expects the Apache Kafka clusters already running to be KRaft-based.
It means that the `Kafka` custom resources are configured together with `KafkaNodePool`(s) with `broker` and `controller` roles.
There is no way to differentiate between clusters running in ZooKeeper or KRaft mode.
When a `Kafka` custom resource is configured to actually use ZooKeeper or it's just badly configured but supposed to be KRaft-based, the operator detects it as having a missing KRaft controllers configuration and logs the following warning:

```shell
io.strimzi.operator.common.model.InvalidResourceException: The Kafka cluster my-cluster is invalid: [At least one KafkaNodePool with the controller role and at least one replica is required when KRaft mode is enabled]
```

In this case, the reconciliation fails and the cluster is not operated at all.
It means that the operator doesn't take any actions on changes applied by the user to the `spec.kafka` or `spec.zookeeper` in the `Kafka` custom resource.

Furthermore, the `UseKRaft` feature gate was created for development and testing.
As we move forward, it will be soon available for regular clusters and the current state is not going to be sufficient anymore.

## Motivation

Leaving a ZooKeeper-based cluster not operated when the `UseKRaft` feature gate is enabled looks like a bad thing for the user, until the ZooKeeper support will be completely removed.
Before that, the user should be able to have both ZooKeeper and KRaft based clusters running and operated when the `UseKRaft` feature gate is enabled.
Also, taking into account the future ZooKeeper to KRaft migration process, as described in the proposal [PR#90](https://github.com/strimzi/proposals/pull/90), it would help the operator migration component to detect which cluster is ZooKeeper-based so "migrate-able" and which one is already KRaft-based.

## Proposal

The proposal is about adding a new `strimzi.io/kraft` annotation to the `Kafka` custom resource, to be applied by the user, in order to identify a ZooKeeper or KRaft based cluster.
The possible values would be:

* `disabled`, missing or any invalid value: identifies a ZooKeeper-based cluster.
* `enabled`: identifies a KRaft-based cluster.

This way, during the reconciliation, the operator is able to "detect" a ZooKeeper-based cluster avoiding the warning and allowing the user to operate it.
On the KRaft side, the annotation would be needed to have the operator reconciling the corresponding `Kafka` custom resource.
Without the annotation, but the `UseKRaft` feature gate enabled, the operator would try to handle it as a ZooKeeper-based one.

| Operator Feature Gate | `strimzi.io/kraft` annotation | Operator behaviour       |
|-----------------------|-------------------------------|--------------------------|
| `-UseKRaft`           | `enabled`                     | Ignore annotation        |
| `-UseKRaft`           | missing or anything else      | ZooKeeper reconciliation |
| `+UseKRaft`           | `enabled`                     | KRaft reconciliation     |
| `+UseKRaft`           | missing or anything else      | ZooKeeper reconciliation |

The `UseKRaft` feature gate does not currently support any upgrades of KRaft clusters.
It is expected to be used only for short-lived clusters used for development and testing.
So no clusters are expected to exist.
Therefore adding the annotation now does not present any backwards compatibility issues.
Only ZooKeeper-based clusters are expected and newly created KRaft-based clusters having the `strimzi.io/kraft: enabled`.
This approach is actually the same as for the node pools: enabling the `KafkaNodePools` feature gate on the operator is not enough and the user needs to apply the `strimzi.io/node-pools: enabled` annotation on the `Kafka` custom resource using node pools for brokers (and controllers, if KRaft is enabled as well).

As a non-goal of this proposal, the same annotation could be even used for handling the ZooKeeper to KRaft migration steps as described in the proposal [PR#90](https://github.com/strimzi/proposals/pull/90).

## Affected/not affected projects

The Strimzi Cluster Operator is the only project affected by this proposal.
Its logic needs to be updated in order to be able to handle the new `strimzi.io/kraft` annotation (or the missing of it).
No other projects in the Strimzi ecosystem are impacted by this proposal.

## Compatibility

If the user is running the operator with `UseKRaft` feature gate enabled, it will detect ZooKeeper-based cluster allowing to operate them again.
If the user has a KRaft-based cluster deployed, because of the missing annotation, the operator will try to handle it as it was ZooKeeper-based which would fail.
In this case, if the user applies `strimzi.io/kraft: enabled` annotation on it, there could be unexpected results on the reconciliation. So doing this is not supported.
In general, when this proposal is in place, the expectation is that there won't be KRaft-based clusters already running but there could be only ZooKeeper-based clusters running.
Breaking this backward compatibility is expected by taking into account that the KRaft support in Strimzi is still behind a feature gate and should be considered just for development purposes. 

## Rejected alternatives

As first iteration of the proposal there are no rejected alternatives.
