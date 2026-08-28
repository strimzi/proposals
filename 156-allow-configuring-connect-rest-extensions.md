# Allow Configuring REST Extensions in Kafka Connect and MirrorMaker 2

This proposal would allow Kafka Connect and Kafka MirrorMaker 2 users to configure their own REST extensions.

## Current Situation

Apache Kafka Connect supports REST extensions.
REST extensions are plugins that implement the `ConnectRestExtension` interface.
Kafka Connect loads these extensions into its REST API, enabling users to add custom logic such as filtering and validation.
REST extensions are configured using the `rest.extension.classes` option in the Kafka Connect worker configuration.
This option accepts a list of extension classes, which Kafka Connect loads and runs in the configured order.

Currently, Strimzi users cannot use custom REST extensions because all configuration options with the `rest.` prefix are forbidden.

## Motivation

REST extensions can implement various customizations.
They can also provide custom authentication and authorization, for example by using workload identities injected through an external mechanism such as the Istio service mesh.
Since Strimzi currently does not secure the Kafka Connect REST API, allowing users to supply their own security extensions would give them greater flexibility.
REST extensions can also support use cases beyond authentication and authorization.

## Proposal

We will continue to block most configuration options with the `rest.` prefix because some of them are essential to keeping the Connect cluster operational.
However, we will add the `rest.extension.classes` option to the list of allowed exceptions so that users can configure their own REST extensions.

### Documentation

The Strimzi documentation will describe the requirements that custom REST extensions must meet to work correctly:
* They must allow unrestricted access to the `/health` endpoint for health checks.
* They must allow Kafka Connect inter-node communication.
* When the Connector Operator is enabled, they must allow the Strimzi Cluster Operator to perform all operations on the Kafka Connect REST API that are required to manage connectors.

Following these requirements will ensure that custom REST extensions are compatible with Strimzi.

### Risks

This is an advanced option that can be configured in way that break the deployments.
However, we can expect that as an advance option, this would be used by _expert_ users.
We should therefore expect users who enable it to:
* Understand its behavior.
* Exercise caution and test their extensions before deployment.

Because enabling this option usually requires users to extend the container image with custom extensions, it is unlikely to be enabled accidentally without understanding the possible consequences.

### Impact on Future Strimzi Development

Strimzi will likely have its own REST extensions at some point, for example to support authentication and authorization in Kafka Connect.
However, as mentioned earlier, the `rest.extension.classes` option defines an ordered list of extensions.
Strimzi can inject its own extensions at the beginning or end of the chain.
This approach is similar to how Strimzi uses configuration providers today.
Therefore, the possibility of Strimzi providing its own extensions should not prevent users from configuring theirs.

## Affected Projects

This proposal affects only the Strimzi Cluster Operator.
It impacts the Kafka Connect and Kafka MirrorMaker 2 operands only.

## Backwards Compatibility

This proposal is fully backward compatible.

## Rejected Alternatives

There are no rejected alternatives.
