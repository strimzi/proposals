[![Strimzi](./logo/strimzi.png)](https://strimzi.io/)

# Strimzi Proposals

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
[![Twitter Follow](https://img.shields.io/twitter/follow/strimziio?style=social)](https://twitter.com/strimziio)

This repository lists proposals for the Strimzi project. A template for new proposals can be found [here](./000-template.md).

|  #  | Title                                                                 |
| :-: |:----------------------------------------------------------------------|
| 83  | [MirrorMaker Connector Offsets Support](./083-mm2-connector-offsets-support.md) |
| 82  | [Moving data between two JBOD disks using Cruise Control](./082-moving-data-between-two-jbod-disks-using-cruise-control.md) |
| 81  | [Unregistration of KRaft nodes](./081-unregistration-of-KRaft-nodes.md) |
| 80  | [Deprecation and removal of Storage overrides](./080-deprecation-and-removal-of-storage-overrides.md) |
| 79  | [Removal of Mirror Maker 1](./079-removal-of-mirror-maker-1.md) |
| 78  | [Auto-rebalancing on cluster scaling](./078-auto-rebalancing-cluster-scaling.md) |
| 77  | [Support for Apache Kafka 4.0](./077-support-for-kafka-4.0.md) |
| 76  | [Connector Offsets Support](./076-connector-offsets-support.md) |
| 75  | [Support of additional volumes](./075-additional-volumes-support.md) |
| 74  | [Extend Feature Gates to all Strimzi operators](./074-extend-feature-gates-to-all-operators.md) |
| 73  | [Improve handling of CA renewals and replacements in operands based on the Kafka clients](./073-improve-handling-of-CA-renewals-and-replacements-in-client-based-operands.md) |
| 72  | [Enhance KafkaBridge resource with consumer inactivity timeout and HTTP consumer/producer parts enablement](./072-kafkabrige-consumer-producer.md) |
| 71  | [Deprecate and remove OpenAPI v2 (Swagger) support on the Strimzi HTTP bridge](./071-deprecate-bridge-openapi-2.md) |
| 70  | [Don't fail reconciliation when Manual Rolling Update fails](./070-dont-fail-reconciliation-in-manual-rolling-update.md) |
| 69  | [Introducing Performance Testing](./069-performance-testing.md) |
| 68  | [Quotas management](./068-quotas-management.md) |
| 67  | [JBOD support in KRaft mode](./067-kraft-jbod-support.md) |
| 66  | [Topic replication factor change](./066-topic-replication-factor-change.md) |
| 65  | [Support custom tiered storage plugins](./065-support-tiered-storage.md) |
| 64  | [Prometheus Metrics Reporter](./064-prometheus-metrics-reporter.md) |
| 63  | [Pod Disruption Budget Generation Environment Variable](./063-pdb-generation-environment-variable.md) |
| 62  | [Promotion of the `UseKRaft` feature gate](./062-UseKRaft-feature-gate-promotion.md) |
| 61  | [KRaft upgrades and downgrades](./061-kraft-upgrades-and-downgrades.md) |
| 60  | [Kafka Roller KRaft Support](./060-kafka-roller-kraft.md) |
| 59  | [ZooKeeper to KRaft migration](./059-zk-kraft-migration.md) |
| 58  | [Deprecate and remove EnvVarConfigProvider](./058-deprecate-and-remove-envvar-config-provider.md) |
| 57  | [Allow running ZooKeeper and KRaft based clusters in parallel](./057-run-zk-kraft-clusters-parallel.md) |
| 56  | [Add ability to create Cruise Control REST API users](./056-cruise-control-api-users.md) |
| 55  | [Infinite auto-restart of Apache Kafka connectors](./055-infinite-auto-restart-of-Kafka-connectors.md) |
| 54  | [Support stopping Kafka Connect connectors](./054-stopping-kafka-connect-connectors.md) |
| 53  | [Record Reconciled Version in Kafka Custom Resource status](./053-record-reconciled-versions.md) |
| 52  | [Kubernetes Server Side Apply](./052-k8s-server-side-apply.md) |
| 51  | [Unidirectional Topic Operator](./051-unidirectional-topic-operator.md) |
| 50  | [Kafka Node Pools](./050-Kafka-Node-Pools.md) |
| 49  | [Prevent broker scale down if it contains partition replicas](./049-prevent-broker-scale-down-if-it-contains-partition-replicas.md) |
| 48  | [Avoid broker restart when in log recovery state](./048-avoid-broker-restarts-when-in-recovery.md) |
| 47  | [Cluster Wide Volume Usage Quota Management](./047-cluster-wide-volume-usage-quota-management.md) |
| 46  | [KRaft liveness and readiness probs](./046-kraft-liveness-readiness.md) |
| 45  | [Stable identities for Kafka Connect worker nodes](./045-Stable-identities-for-Kafka-Connect-worker-nodes.md) |
| 44  | [StrimziPodSets graduation](./044-StrimziPodSets-graduation.md) |
| 43  | [Deprecate and remove JMX Trans](./043-deprecate-and-remove-jmxtrans.md) |
| 42  | [Remove AMQP 1.0 support from the Strimzi bridge](./042-remove-bridge-amqp-support.md) |
| 41  | [User Operator: Configurable exclusion of labels](./041-user-operator-configurable-exclusion-of-labels.md) |
| 40  | [Refactor KafkaConfig files in Strimzi Client Examples](./040-refactor-client-examples.md) |
| 39  | [Reduce Strimzi test-client's images](./039-reduce-test-clients-images.md) |
| 38  | [Auto-approval mechanism for optimization proposals](./038-optimization-proposal-autoapproval.md) |
| 37  | [Pluggable Pod Security Profiles](./037-pluggable-pod-security-profiles.md) |
| 36  | [KRaft support: ZooKeeper-less Kafka](./036-kraft-mode.md) |
| 35  | [Extending the `KafkaRebalance` resource with rebalance types to help scaling the Apache Kafka cluster](./035-rebalance-types-scaling-brokers.md) |
| 34  | [Deprecate and remove MirrorMaker 2 extensions](./034-deprecate-and-remove-mirror-maker-2-extensions.md) |
| 33  | [Service Binding](./033-service-binding.md) |
| 32  | [Custom Authentication in Kafka Brokers ](./032-custom_authentication_in_kafka_brokers.md) |
| 31  | [StatefulSet Removal](./031-statefulset-removal.md) |
| 30  | [EnvVar Configuration Provider for Apache Kafka](./030-env-var-config-provider.md) |
| 29  | [Adopt the Drain Cleaner utility](./029-adopt-the-drain-cleaner-utility.md) |
| 28  | [Network Policy Generation Environment Variable](./028-network-policy-generation-environment-variable.md) |
| 27  | [Kubernetes Configuration Provider for Apache Kafka](./027-kubernetes-config-provider.md) |
| 26  | [Service Account patching](./026-service-account-patching.md) |
| 25  | [Control Plane Listener](./025-control-plain-listener.md) |
| 24  | [Adopt the Kafka Static Quota plugin](./024-adopt-the-kafka-static-quota-plugin.md) |
| 23  | [Using Red Hat Universal Base Image 8 as the new Strimzi base image](./023-using-ubi8-as-base-image.md) |
| 22  | [Feature Gates](./022-feature-gates.md) |
| 21  | [Special repository for ST clients](./021-special-repository-for-st-clients-based-on-example-clients.md) |
| 20  | [Rename the default branch of Strimzi GitHub repositories](./020-rename-default-branch-of-strimzi-github-repositories.md) |
| 19  | [Restructure the installation files](./019-restruture-the-installation-files.md) |
| 18  | [Use the admin-server REST API in strimzi-ui](./018-rest-admin-api.md) |
| 17  | [Proxy-Based Kafka Per-Topic Encryption](./017-kafka-topic-encryption.md) |
| 16  | [Modularizing Strimzi UI](./016-modularizing-strimzi-ui.md) |
| 15  | [Kafka Connect build](./015-kafka-connect-build.md) |
| 14  | [Move Container Images to Quay.io](./014-move-docker-images-to-quay.io.md) |
| 13  | [Strimzi canary](./013-kafka-canary.md) |
| 12  | [Create an Admin Server](./012-admin-server.md) |
| 11  | [Strimzi UI](./011-strimzi-ui.md) |
| 10  | [Auth additions for UI and Admin Server](./010-ui-and-admin-server-security.md)  |
|  9  | [Roadmap to using the CRD v1 API](./009-crd-v1-roadmap.md)  |
|  8  | [TLS encrypting the Kafka Connect REST API](./008-tls-encrypt-the-kafka-connect-rest-api.md)  |
|  7  | [Restarting Kafka Connect connectors and tasks](./007-restarting-kafka-connect-connectors-and-tasks.md) |
|  6  | [Design Documentation](./006-design-docs.md) |
|  5  | [Improving configurability of Kafka listeners](./005-improving-configurability-of-kafka-listeners.md) |
|  4  | [GitHub repository restructuring](./004-github-repository-restructuring.md) |
|  3  | [Remove deprecated Topic Operator deployment in Kafka CRD](./003-remove-deprecated-topic-operator-from-kafka-crd.md) |
|  2  | [Documentation improvements](./002-documentation-improvements.md) |
|  1  | [Move Strimzi Kafka operators to Java 11](./001-move-strimzi-kafka-operators-to-java-11.md) |

---

Strimzi is a <a href="http://cncf.io">Cloud Native Computing Foundation</a> incubating project.

![CNCF ><](./logo/cncf-color.png)