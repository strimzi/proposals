# Promotion of the `UseKRaft` feature gate

The `UseKRaft` feature gate allows users and developers of the Strimzi operator to deploy and manage a KRaft-based Kafka cluster.
It was originally introduced in May 2022 in the [Strimzi Proposal #36 - KRaft support: ZooKeeper-less Kafka](https://github.com/strimzi/proposals/blob/main/036-kraft-mode.md).
When introduced, KRaft mode in both Apache Kafka and Strimzi had significant limitations.
Consequently, no specific timeline was established for the graduation of the feature gate.
Currently, the feature gate remains in the _alpha_ stage and is disabled by default.
While there are still some remaining limitations in Apache Kafka and Strimzi, they are less significant than they were when the original proposal was written.
This proposal aims to create a plan for the graduation of the `UseKraft` feature gate and the changes related to it.

## Current limitations

The current support for KRaft in Strimzi (and Apache Kafka) has the following major limitations:
* Support for migration of ZooKeeper-based clusters to KRaft
  * Tracked in [strimzi/strimzi-kafka-operator#9433](https://github.com/strimzi/strimzi-kafka-operator/issues/9433), [strimzi/strimzi-kafka-operator#9447](https://github.com/strimzi/strimzi-kafka-operator/issues/9447), and [strimzi/strimzi-kafka-operator#9448](https://github.com/strimzi/strimzi-kafka-operator/issues/9448)
  * This is currently work in progress and waiting for Apache Kafka 3.7.0 release with several bug-fixes and implementation of [KIP-919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration)
  * Support for the migration is critical for existing users and has to be done before support for ZooKeeper-based cluster is dropped.
    But missing support for migration does not affect any users who would want to deploy and run new clusters in KRaft mode.
  * This is currently expected to be implemented in Strimzi 0.40 or 0.41.
* Scaling of KRaft controller nodes
  * Tracked in [strimzi/strimzi-kafka-operator#9429](https://github.com/strimzi/strimzi-kafka-operator/issues/9429)
  * Currently not supported are:
    * Scale-down of dedicated or mixed controller nodes that breaches the controller quorum
    * Scale-up of dedicated controller nodes (the scale-up eventually succeeds, but the broker availability is not maintained)
  * This is currently blocked by the support for [KIP-853](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes).
    This KIP might not be implemented earlier than in Apache Kafka 4.0.
    Support for scaling controllers is currently not seen as blocker for moving the `UseKRaft` feature gate to _GA_.
* JBOD support
  * Tracked in [strimzi/strimzi-kafka-operator#9437](https://github.com/strimzi/strimzi-kafka-operator/issues/9437)
  * Expected to be implemented in Apache Kafka 3.7.0.
    The Strimzi implementation will follow the release of Kafka 3.7.0 in Strimzi 0.40 or 0.41.

## Proposed timeline

This proposal outlines the following timeline for the graduation of the `UseKRaft` feature gate:
* Move to _beta_ phase and be enabled by default in Strimzi 0.40.0
* Move to _GA_ phase and be permanently enabled in Strimzi 0.42.0

It is worth noting that:
* In addition to the `UseKRaft` feature gate, the KRaft clusters are enabled / disabled using the `strimzi.io/kraft` annotation.
  As a result, moving the `UseKRaft` feature gate to _beta_ or _GA_ does not mean that all new Kafka clusters have to use KRaft or that this has any impact on existing ZooKeeper-based clusters.
  KRaft will be applied only to the Kafka clusters with the right annotation.
* Moving the `UseKRaft` feature to beta or GA does not mean we will drop support for ZooKeeper-based clusters.
  While moving the `UseKRaft` feature gate to _GA_ defines the earliest moment when support for ZooKeeper based clusters can be dropped, it is currently not expected to happen right after Strimzi 0.42 and this proposal does not establish any such plan.

Moving the `UseKRaft` feature gate to _beta_ or _GA_ serves mainly the following objectives:
* Signify the progress of the KRaft implementation and improvements in production-readiness.
* Simplify running KRaft clusters by eliminating the need to enable any feature gate.

## Proposed changes

Apart from the change to the feature gate status itself, this proposal also includes several other changes.

### CRD changes

KRaft is usable only with the use of `KafkaNodePool` custom resources that configure the number of replicas and storage configuration.
As part of promoting the `UseKRaft` feature gate to _beta_ stage, we will make the following fields in the `Kafka` custom resource optional instead of required:
* `.spec.kafka.replicas`
* `.spec.kafka.storage`

In addition, the KRaft clusters do not need the ZooKeeper configuration.
So the `.spec.zookeeper` section will be made optional as well.

For ZooKeeper-based Kafka clusters, the validation of these fields will be done inside the Strimzi operator.
Additionally, CEL validation rules will be considered as they might allow us to do additional validation on the Kubernetes level.
The validation will check if the cluster is ZooKeeper-based and in that case require the fields mentioned above.

For Kafka clusters using the node pools, a warning will be raised by the operator when the ignored `.spec.kafka.replicas` and `.spec.kafka.storage` fields are used.
For KRaft-based clusters, a warning will be raised when `.spec.zookeeper` section is used.

### Safety check for existing clusters

For existing clusters, a safety check will be implemented to prevent users from switching existing ZooKeeper-based clusters to Kraft-based cluster (or vice-versa) by mistake.
This check will use the `.status.kafkaMetadataState` field in the `Kafka` custom resource to prevent any unintentional switching between ZooKeeper-based and Kraft-based clusters. 
Switching cluster management modes must be performed through a migration process.
This field is already defined in [Strimzi Proposal #59 - ZooKeeper to KRaft migration](https://github.com/strimzi/proposals/blob/main/059-zk-kraft-migration.md).
The check will compare the desired cluster type with existing type and if they do not match, it will throw an exception and end the reconciliation.
It will be implemented as part of the migration work if it is shipped in Strimzi 0.40.0.
Or separately if the migration is postponed to Strimzi 0.41.0.

### Examples

The existing examples will be updated.
We will also change the structure of the examples in the following way
* The file `examples/kafka/nodepools/kafka.yaml` will be moved to `examples/kafka/kafka-with-nodepools.yaml`
* The `examples/kafka/nodepools/` directory with the remaining files will be renamed to `examples/kafka/kraft/`

### Other changes

The unit, integration, and system tests as well as the documentation will be updated to be in sync with this proposal as well.

## Affected projects

This proposal affects only the Strimzi Cluster Operator.

## Backwards compatibility

This proposal has no impact on backwards compatibility.
