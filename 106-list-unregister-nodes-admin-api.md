# Use Kafka Admin API to list registered brokers and manage their unregistration

This proposal suggests leveraging [KIP-1073](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1073:+Return+fenced+brokers+in+DescribeCluster+response) to use the Kafka Admin API for listing all registered brokers in an Apache Kafka cluster (including the fenced ones) and using that information for managing brokers unregistration.
It replaces the current mechanism that tracks registered brokers (together with the controllers) in the `Kafka` custom resource status.

## Current situation

Currently, the Strimzi operator tracks the list of registered KRaft nodes (both brokers and controllers) in the `Kafka` custom resource status via the `registeredNodeIds` field, as shown in the example below:

```yaml
status:
    clusterId: bXz0umhlQ0WLSobuC-Irdg
    #....
    registeredNodeIds:
    - 0
    - 1
    - 2
    - 3
    - 4
    - 5
```

When the Kafka cluster is scaled down, the operator compares the current running node IDs against the stored `registeredNodeIds` to determine which brokers should be unregistered via the Kafka Admin API, because they were removed.
This logic was introduced as part of the [Strimzi proposal 081](https://github.com/strimzi/proposals/blob/main/081-unregistration-of-KRaft-nodes.md), which implemented a workaround for the limited brokers unregistration support in Kafka at that time. There was no reliable way to list registered brokers (especially fenced ones), so Strimzi maintained this state internally.

However, this approach has limitations.
For example, when brokers are initially deployed as mixed-mode nodes (controller + broker) and later have their broker role removed, they remain registered.
Although these nodes (as brokers) become fenced and controller-only, they may still receive partition assignments for new topics.
The current implementation does not detect or handle this scenario, leading to missing unregistration.
This issue has been reported by the community in [GitHub issue #11477](https://github.com/strimzi/strimzi-kafka-operator/issues/11477).

## Motivation

With the introduction of [KIP-1073](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1073%3A+Return+fenced+brokers+in+DescribeCluster+response) in Apache Kafka 4.0.0, the Kafka Admin API now provides the ability to list all registered brokers, including fenced ones.
This makes it possible to manage unregistration entirely through standard Kafka APIs.

This removes the need for Strimzi to track nodes registration status manually and enables proper handling of edge cases like mixed-node role changes.
As a result, it also allows for a proper resolution of issue [#11477](https://github.com/strimzi/strimzi-kafka-operator/issues/11477).

## Proposal

This proposal suggests removing the custom registration tracking and instead relying on Apache Kafka's built-in registration tracking available via the Admin API.

### Key changes

The key changes are:

* deprecate the `status.registeredNodeIds` field from the `Kafka` custom resource. The field will remain in the CRD but the operator won't set it in the `Kafka` status.
* use the Kafka Admin client's `describeCluster()` method (with the option of including the fenced nodes) to retrieve the list of currently registered brokers. This call is done within a newly added `KafkaNodeUnregistration.listRegisteredBrokerNodes` method.
* compare the list of registered brokers to the current set of active nodes and determine which ones need to be unregistered.
* unregister brokers using the existing `KafkaNodeUnregistration.unregisterNodes()` method which would be renamed as `KafkaNodeUnregistration.unregisterBrokerNodes()` because this is what it actually does by calling Admin client's `unregisterBroker()` method internally.

### Reconciliation flow

During a reconciliation, the operator goes through the following steps:

* get current nodes IDs (both brokers and controllers) from the `KafkaCluster.nodes()` as is done today. It also extracts which ones are controller only nodes (not in mixed mode).
* call `KafkaNodeUnregistration.listRegisteredBrokerNodes` method using the Admin client connected to the brokers' bootstrap service to retrieve all registered brokers.
* build a `brokersIdsToUnregister` list with the brokers IDs to be unregistered.
* unregister the brokers via the `KafkaNodeUnregistration.unregisterBrokerNodes` method.

The `brokersIdsToUnregister` list is created in the following way: 

* define `previousBrokersIds` as the set of all registered brokers returned by Kafka.
* define `fencedControllerOnlyNodeIds` as the subset of fenced nodes that are now controller-only (i.e., no longer brokers from being mixed-node).
* finally create `brokersIdsToUnregister` by:
    * initializing it with the `previousBrokersIds`.
    * removing current running node IDs from it (to detect scale-down).
    * adding `fencedControllerOnlyNodeIds` (to unregister removed brokers in role change scenarios).

The operator doesn't fill the `status.registeredNodeIds` field in the `Kafka` custom resource anymore.

### Error handling

Any error encountered while listing registered brokers is logged as a warning.
The reconciliation does not fail and the operation is retried during the next reconciliation loop.
This behavior is consistent with the current handling of errors during the brokers unregistration process.

## Affected/not affected projects

The only project to be affected is the Strimzi cluster operator.
The `KafkaNodeUnregistration` class will implement a new method `listRegisteredBrokerNodes()` using the Kafka Admin client and the `unregisterNodes()` will be renamed as `unregisterBrokerNodes()`.
The `KafkaReconciler` will stop using `status.registeredNodeIds` and instead rely on `listRegisteredBrokerNodes()` for determining node unregistration.

## Compatibility

This feature will come in the next Strimzi release with the support for Apache Kafka 4.x versions only (with KIP-1073 implementation available).
When the operator is upgraded, it won't use the `status.registeredNodeIds` field anymore so users should deal with it if they are using such a field for any operational purpose.
However, [Strimzi proposal 081](https://github.com/strimzi/proposals/blob/main/081-unregistration-of-KRaft-nodes.md) clearly stated that the field was temporary and subject to be deprecated once Kafka provided native support and then removed in the future as part of the `v1` CRD API.

## Rejected alternatives

N/A
