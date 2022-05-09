# KRaft support: ZooKeeper-less Kafka

## Motivation

The Apache Kafka project is working on removing its dependency on ZooKeeper.
Because of the complexity of this task, the work has already been in progress for more than a year.
It was started with the [Kafka Improvement Proposal 500 (KIP-500)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum) and will continue for at least six to twelve months until all work is done and it is production-ready.
Step-by-step, Apache Kafka will remove all dependencies on ZooKeeper. 
These range from smaller dependencies, such as CLI and management tools, to more complex dependencies involving  quorum and metadata management. 
A consensus protocol called _Apache Kafka Raft_ (or _KRaft_) will replace ZooKeeper. 
This is what gives the name to this proposal as well as to the feature gate discussed later.

Strimzi aims to support the ZooKeeper-less Kafka and the KRaft protocol.
The issue [strimzi/strimzi-kafka-operator#5615](https://github.com/strimzi/strimzi-kafka-operator/issues/5615) can be used to track the overall progress and production-ready support for KRaft.
Since the announcement of the KIP-500, we have been making sure that Strimzi is as ready as possible for when the KRaft mode becomes production-ready.
For example, Strimzi has adopted the new Kafka Admin APIs that use Kafka instead of ZooKeeper in components like the User Operator and Cruise Control.
But right now, you cannot really use Strimzi to run Kafka with the KRaft mode enabled.
That makes it hard to continue working on some of the additional tasks needed to have Strimzi production-ready with KRaft in the future.
For example, work is pending on readiness / liveness probes, Topic Operator improvements, and upgrades.
Task such as these will be easier to work on with Strimzi supporting the KRaft mode out-of-the-box.

## Proposal

This proposal suggests adding provisional support for the KRaft mode.
It will be disabled by default and will be protected by a new feature gate.
When the feature gate is enabled, it will use the KRaft mode to deploy the Kafka cluster.
It will simplify development and testing of new Strimzi features in the KRaft mode.
And it will also help with testing of the KRaft mode and raising any issues that arise in the Apache Kafka project.

The KRaft support proposed here is called provisional, because the API in the Kafka CR for configuring the Kafka nodes is not final and will be changed later.
Most of the code implemented by this proposal is expected to be used for the final production-ready implementation.

**The KRaft mode support suggested by this proposal will not be production-ready and should not be used outside of development.**

### API changes

In the initial implementation, there will be no changes to the `Kafka` custom resource and its API.
When the KRaft mode is enabled, the operator will ignore any of the unsupported but required fields such as `.spec.zookeeper`.
It will also validate the resource for any other unsupported features (such as authorization, JBOD storage etc.).

The API is expected to change at a later phase before the KRaft support is considered production-ready.

### KRaft deployment

The initial implementation will support only a single type of KRaft deployment.
All Kafka nodes will be created according to `.spec.kafka` section of the `Kafka` CR.
It will respect the `.spec.kafka.replicas` field and deploy the corresponding number of Kafka nodes.
All nodes will be assigned both the `controller` and `broker` KRaft roles (see the [KIP-500](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum) for more details about the `controller` and `broker` roles).
This deployment architecture is suitable for development and testing clusters.

Other architectures, such as separate controller and broker nodes, will be not supported from the start.
This is expected to change before the KRaft support is considered production-ready.

### Feature Gate

The new feature gate will be called `UseKRaft`.
It will be introduced in an alpha state and will be disabled by default.
At this point, there is no timeline for graduation of this feature gate to beta or GA phase since it depends on things outside of our control.
The schedule will be updated later as the KRaft development progresses both in Apache Kafka as well as in Strimzi.

In the initial implementation, enabling or disabling this feature gate with pre-existing Kafka clusters will not be supported.
Users will need to delete all clusters before enabling or disabling the feature gate.

#### Dependency on other feature gates

The `UseKRaft` feature gate will be designed to work only with the `UseStrimziPodSets` feature gate enabled (see the [StatefulSet removal proposal](https://github.com/strimzi/proposals/blob/main/031-statefulset-removal.md) for more details).
The StrimziPodSets will be what Strimzi will use in the future instead of the StatefulSets.
There is no plan to support the StatefulSets in the KRaft mode.
The feature gate settings will be validated when the operator starts to ensure that both feature gates are set correctly.

### Limitations

The initial implementation of this proposal has a very limited set of features.
These features might be missing because they are not supported at this point either by Apache Kafka or by Strimzi.
Some of the known limitations are included in the following list:

* Moving from Kafka clusters with ZooKeeper to KRaft clusters or the other way around.
* Upgrades / downgrades of Apache Kafka versions or of the Strimzi operator are not supported. 
  Users might need to delete the cluster, upgrade the operator and deploy a new Kafka cluster.
* Entity Operator (both User and Topic operator) are not supported.
* Authorization is not supported.
* SCRAM-SHA-512 users are not supported.
* JBOD storage is not supported (the `type: jbod` storage can be used, but the JBOD array can contain only one disk).
* Liveness and readiness probes are disabled.
* KRaft architectures using separate controller and broker nodes.

As the implementation progresses, these features might be added later with separate PRs without a dedicated Strimzi Proposal.

## Compatibility

All changes introduced by this proposal will be feature-gated.
This proposal should have no impact on any existing Kafka clusters deployed with ZooKeeper.

## Rejected alternatives

One of the alternatives which were rejected was to develop and maintain the KRaft implementation in a separate branch instead of doing it in the `main` branch using a feature gate.
This was rejected, because it would make it harder to contribute the new features and it would also make it less accessible for regular users who might be interested to help with testing or further development of this feature.

## Risks

This proposal suggests adding a new feature in a very early development phase.
Maintaining this feature in the `main` branch might cause additional effort.
It is also possible that some of the early code might be thrown away later.
The expected benefits seem to currently out-weigh the risks.
