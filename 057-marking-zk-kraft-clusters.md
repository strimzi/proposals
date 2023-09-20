# Marking ZooKeeper and KRaft based clusters for KRaft enabled operators

This proposal is about providing to the Strimzi Cluster Operator a way to "detect" if an Apache Kafka cluster is ZooKeeper or KRaft based when the KRaft feature gate is enabled.

## Current situation

Currently, when the `UseKRaft` feature gate is enabled, the operator expects all the Apache Kafka clusters being KRaft-based.
There is no way to differentiate between clusters running in ZooKeeper or KRaft mode.
When a `Kafka` custom resource, configured to use ZooKeeper for metadata, is reconciled, the operator detects it with a missing KRaft controllers configuration logging the following warning:

```shell
io.strimzi.operator.common.model.InvalidResourceException: Tha Kafka cluster my-cluster is invalid: [At least one KafkaNodePool with the controller role and at least one replica is required when KRaft mode is enabled]
```

In this case, the reconciliation fails and the cluster is not operated at all.
It means that the operator doesn't take any actions on changes applied by the user to the `spec.kafka` or `spec.zookeeper` in the `Kafka` custom resource.

## Motivation

Leaving a ZooKeeper-based cluster not operated when the `UseKRaft` feature gate is enabled looks like a bad thing for the user, until the ZooKeeper support will be completely removed.
Before that, the user should be able to have both ZooKeeper and KRaft based clusters running and operated when the `UseKRaft` feature gate is enabled.
Also, taking into account the future ZooKeeper to KRaft migration process, as described in the proposal [PR#90](https://github.com/strimzi/proposals/pull/90), it would help the operator migration component to detect which cluster is ZooKeeper-based so "migrate-able" and which one is already KRaft-based.

## Proposal

The proposal is about adding a new `strimzi.io/kraft` annotation to the `Kafka` custom resource, to be applied by the user, in order to identify a ZooKeeper or KRaft based cluster.
The possible values would be:

* `disabled` (or missing): identifies a ZooKeeper-based cluster.
* `enabled`: identifies a KRaft-based cluster.

This way, during the reconciliation, the operator is able to detect a ZooKeeper-based cluster avoiding the warning, as described in the "Current" section, but allowing the user to operate it.
On the KRaft side, the annotation would be needed to have the operator reconciling the corresponding `Kafka` custom resource.
Without the annotation, but the `UseKRaft` feature gate enabled, the operator would try to handle it as a ZooKeeper-based one.
This approach is actually the same as for the node pools: enabling the `KafkaNodePools` feature gate on the operator is not enough and the user needs to apply the `strimzi.io/node-pools: enabled` annotation on the `Kafka` custom resource using node pools for brokers (and controllers, if KRaft enabled as well).

The same annotation could be even used in the proposal [PR#90](https://github.com/strimzi/proposals/pull/90) for handling the ZooKeeper to KRaft migration steps.

## Affected/not affected projects

The Strimzi Cluster Operator is the only project affected by this proposal.
Its logic needs to be updated in order to be able to handle the new `strimzi.io/kraft` annotation (or the missing of it).
No other projects in the Strimzi ecosystem are impacted by this proposal.

## Compatibility

If the user is running the operator with `UseKRaft` feature gate enabled, it will detect ZooKeeper-based cluster allowing to operate them again.
If the user has a KRaft-based cluster deployed, it would be ignored and not reconciled anymore, unless the user applies `strimzi.io/kraft: enabled` annotation on it.
Breaking this backward compatibility is expected taking into account that the KRaft support in Strimzi is still behind a feature gate and should be considered just for development purposes. 

## Rejected alternatives

As first iteration of the proposal there are no rejected alternatives.
