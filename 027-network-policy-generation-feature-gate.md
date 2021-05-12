# Network Policy Generation Feature Gate

The feature would consist of a new Feature Gate which can be used to disable the generation of Network Policies by Strimzi, where the user wishes to set up a more restrictive Network Policy set.

## Current situation

Currently, Strimzi generates Network Policies which allow all pods in the deployment to connect to Kafka, Zookeeper and any other service that it has generated.

There is no way to stop this generation.

It is possible, in the select case of listeners, to further restrict the Ingress using NetworkPolicyPeers, however this does not extend to all Network Policy generation, for example Metrics.
It is also not possible to set up Egress Network Policies. 

## Motivation

The current Network Policies that are generated are great for getting Strimzi up and running in a deployment where a deny-all network policy is in place.

However, this does not allow users of Strimzi to reach a fully secured Network Policy set, due to Strimzi not providing the full feature set of Kubernetes Network Policies.

By adding a feature gate, it allows users to disable this generation, and instead write fully secure Network Policies.

This setup is valid in 2 situations:

* A deployment with no deny-all Network Policy, where all pods can already communicate with each other.
* A deployment where the consumer wishes to write their own secure Network Policies.

## Proposal

The proposal would be to introduce a Feature Gate which would provide the ability to turn of the generation of Network Policies by Strimzi.

As the functionality already exists to generate Network Policies, a sensible naming of this gate would be `NetworkPolicyGeneration` which by default is set to `true`.
If users require secure Network Policies, they can disable this using the regular Feature Gate syntax.
This Feature Gate will be a long lived Feature Gate, and is not meant to phase in a feature. Instead, it should remain so users can turn off this functionality when needed.

When this feature gate is disabled, all generation of Network Policies from the Strimzi Operator is also disabled.

This extends to:

* CruiseControl
* KafkaCluster
* KafkaConnectCluster
* ZookeeperCluster

When enabled, as is the default, the Network Policy Generation will act as it does currently, creating Network Policies for the Strimzi services.

This will leave Strimzi in a position to support all secure setups of Network Policy. And means the native support by Strimzi will be 
"Generation of Network Policies is provided which limit on port, and in the case of listeners Ingress NetworkPolicyPeers. All other Network Policy behaviour can be achieved by disabling the automatic generation and adding Network Policies manually.".

## Affected/not affected projects

This only impacts the [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator) project, which currently generates these network policies.

## Compatibility

By setting this Feature Gate to `true` by default, this maintains existing compatibility.

## Rejected alternatives

The other alternative to adding a feature gate was to attempt to support the full functionality of Network Policies within Strimzi.
The origin requests for this ticket was to allow more strict rules for the Metrics ports, however further requests also exists for adding Egress support in these Network Policies.

This option was rejected as this introduces a lot of customisation that will not be used by the majority of users.
When a user is expecting secure network policies, they are most likely accepting of having to write these themselves, rather than Strimzi having to recreate the same syntax already present in Network Policies.
