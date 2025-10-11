# Deprecate and remove `.spec.kafka.resources` from the `Kafka` custom resource

This proposal recommends deprecating the `.spec.kafka.resources` field of the `Kafka` custom resource and removing it in the `v1` CRD API.

## Current situation

Resource requests and limits for Apache Kafka nodes can be configured in two different places:
* In the `.spec.kafka.resources` field in the `Kafka` CR.
* In the `.spec.resources` field in the `KafkaNodePool` CR

The resources configured in the `Kafka` CR apply to all Apache Kafka nodes.
The resources configured in the `KafkaNodePool` CR apply only to the nodes belonging to a given node pool.
When both fields are set, the configuration from the `KafkaNodePool` is used.

## Motivation

Being able to configure the resources centrally for all nodes from the `Kafka` resource seems like a useful feature at first glance.

But while there might be different situations and scenarios why users would use multiple node pools, the most common one will be one node pool for controller nodes and another one for brokers.
This is the architecture recommended for production.

However, the resource requirements for controller nodes and broker nodes are expected to be different in most cases (brokers would typically need significantly more resources than controllers).
So in this situation, configuring the resources centrally does not seem very useful.
It might even be harmful if the user configures the same resources for both controllers and brokers centrally, as it would lead to the cluster underperforming or to significant resource waste.
Removing the `.spec.kafka.resources` field from the `Kafka` CR should make the resource configuration process clearer to users.

There are other scenarios where users might use multiple node pools.
For example, users might use node pools per availability zone to simplify node scheduling.
In some of these scenarios, configuring resources through the `Kafka` CR might make more sense.
However, controller-only node pools are still expected in these cases.
So it would still not apply to all nodes, and users would still need to configure resources in one or more `KafkaNodePool` CRs.
Removing the resource configuration from the `Kafka` CR supports these scenarios, but would also help standardize resource management and encourage users to adopt a node pool–based configuration model.

## Proposal

The `.spec.kafka.resources` section will be deprecated in the `v1beta2` API and will be removed in the `v1` CRD API.
It will work as before and remain fully supported while the `v1beta2` API is active, up to Strimzi 0.52 / 1.0.0.

### API Conversion

The API conversion always converts only a single custom resource.
So it cannot move the resource configuration from the `Kafka` CR to the `KafkaNodePool` CRs.
It can either:
* Remove the `.spec.kafka.resources` without any replacement
* Fail the conversion and request the user to deal with the field manually (by removing it or moving it to the `KafkaNodePool` resources)

Dropping the resources definitions might cause significant problems for the Kafka clusters.
This proposal therefore suggests failing the conversion and requesting users to deal with it manually.

We should expect this to affect many users, because most clusters that existed before node pools had the resources configured in the `Kafka` CR as it was the only option at that time.
So in some cases, even if the users have different configurations in the `KafkaNodePool` resources, they might still have the old settings in the `Kafka` CR as well.
This means that this proposal might have a significant impact on users.
However, other fields will need to be converted manually, so this change is justified if it contributes to a cleaner `v1` CRD API.

### Implementation

The proposal will be implemented in the `api` and `cluster-operator` modules (and in the planned API Conversion Tool).
The implementation is expected to be straightforward.
In the first phase, as part of the 0.49.0 release:
* The `.spec.kafka.resources` field will be deprecated and marked as present in the `v1beta2` API only.
* The deprecation warnings will be suppressed in the `cluster-operator` module where the field is used.
* The API Conversion Tool will be updated to make sure it handles the conversion (raises the error).

In the second phase, as part of the 0.52 / 1.0.0 release:
* The `.spec.kafka.resources` field will be removed.
* Its uses in the `cluster-operator` module will be removed as well.

### Documentation

The documentation will be updated with this change.
The regular documentation will not show or recommend the deprecated field.
The (manual) conversion will be covered as part of the `v1` CRD API conversion documentation.
It will also be included in the release notes as it is a breaking change.

## Backwards compatibility

**This proposal is not backwards compatible!**
Users might need to modify their custom resources to keep their existing configuration.
So it is important that users are made aware of this change and can prepare for it.
However, it will be part of the wider `v1` CRD API changes.
So it will not be the only change users will have to deal with.

## Rejected alternatives

There are no rejected alternatives.
