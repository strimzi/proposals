# Feature Gates

This proposal suggests to add support for Feature Gates in the Strimzi operators.
Feature Gates should give us additional options how to control and mature different behaviors in the operators.

## Current situation

Currently, any features or behaviors of the operator are controlled by one of these three mechanisms:

* The Strimzi custom resources (as a form of an API)
* The state of the Kubernetes resources
* Environment variables in the operator deployment

The following examples show how are these mechanisms used to give a better idea:

* The `strimzi.io/use-connector-resources` annotation in the `KafkaConnect` custom resource defines whether the Connector Operator should be enabled or not for given Kafka Connect resource.
* The `useServiceDnsDomain` in the listener configuration of the `Kafka` custom resource defines what kind of address should be used for given listener.
  Whether fully qualified DNS name or not.
* When the operator during Kafka upgrade sees that the old pods are still using the TLS sidecars, it will automatically loosen the TLS configuration to make sure the upgrade can be completed.
* When the environment variable `STRIMZI_CUSTOM_RESOURCE_SELECTOR` is configured, it controls what custom resources will the operator see and process.

## Motivation

In many cases, the mechanisms described above work fine and are well suited.
But there are also limitations:

* Changes to the custom resources APIs have upgrade and forwards compatibility concerns.
  We can remove fields from the CRDs only when changing the API versions.
  So adding fields to the Kafka CR is not a good option to gradually roll out changes or change a default behavior over multiple releases.
* Sometimes, you also want to just change the behavior of the operator in general.
  But you want to avoid having endless list of `enableFeatureX`, `disableFeatureY`, or `useZ` fields in the custom resources.
  For such fields, it would be also hard to change their defaults later in backwards-compatible way.
  So in case such feature would be disabled by default at first, users would need to explicitly enable it even later when the feature is mature.
* The use of the state of the existing Kubernetes resources works well for one-of operations.
  That fits well for upgrades or for triggering actions such as restarting a pod.
  But not for features which change the operator behavior during the whole run of the operator.

This shows the there might be space for other ways how to enable or disable features.

## Proposal

Strimzi operators should support _feature gates_ to gate selected functionality and switch it on or off.
Each feature gate will have a default state (enabled or disabled by default) and users will be able to manually enable or disable them.
The feature gates will apply for the whole operator and alter its behavior.

The feature gates should go through 3 different maturity stages:
* Alpha (typically disabled by default)
* Beta (typically enabled by default)
* General Availability / GA (typically enabled by default or always enabled)

Alpha stage indicates that the feature might be unstable (i.e. it might change in the future) or buggy.
Beta stage indicates a that the feature is already well tested and most probably should not change.
GA means that the feature is stable and should not change anymore.
GA features might optionally also change to _always enabled features_.
In this case the feature gate will be removed and it will not be possible to disable the feature anymore.
The alpha and beta features might be removed in case they are deemed as not useful or not the solution.

Typically, the gated features should be introduced using a Strimzi Proposal
The proposal should also suggest the plan for how the feature will move through the different maturity stages.
The feature gates, their default state (disabled / enabled) and their current maturity level will be also listed in the documentation.

### Use cases 

The feature gates should be used for the following use-cases:

_Note: The examples are just examples. Approving this proposal does not approve their implementation using feature gates._

#### Gradually introducing changes to behavior while minimizing disruption

The feature gate will be introduced at the alpha stage and disabled by default.
Users will be able to manually enable it to test or use the new feature.
After one or more releases, the feature gate will move to beta stage and change to enabled by default.
Users who are not ready to use it will be still able to disable it manually.
After another one or more releases, the feature gate might move to GA and be removed / always enabled.

An example of this use case might be the introduction of the separate control plane listener.
The listener it self can be added by default.
But the feature gate would control whether it is actually used or not.
At first, it will be disabled by default allowing smooth upgrades without going through every single Strimzi version.
But users who want to use this feature will be able to enable it and benefit from it.
Later it would be enabled by default - most users will use it, but those upgrading from much older versions without the control plane listener would still be able to disable it, upgrade and enable it.
After sufficient amount of releases, the feature gate will be removed and the separate control plane listener will be used by all the time.

#### Introducing experimental or unstable APIs

Experimental features or APIs might benefit from the feature gate maturity model.
They might start as alpha features and be disabled by default.
During this stage, they can be evaluated and tested.
Features in the alpha state might be seen as experimental because of being considered risky (with high probability of bugs and issues) or because the feature / behavior might change in the future.
As the new feature matures and becomes less experimental, it might move to beta and be enabled by default..
Finally, when the feature is seen as stable and ready for _production_, the feature gate should move to GA and possibly removed / always enabled.

An example of this use-cases might be for example using a feature gate for the StatefulSet removal or possibly some features related to KIP-500 / ZooKeeper removal.

#### Enabling / disabling a feature

In some cases, different users might have different preferences and it might be hard to find a middle ground which would make everyone happy and satisfied.
In such case, the feature gates might be used to modified the behavior of the operator and enable / disable certain features.
In use case like this, the feature gate might not strictly follow the maturity model.
It might be disabled by default even when in the GA maturity stage - the GA stage would in this case indicate that the feature is stable and should not change.
In some cases it might also make sense for these features to be introduced directly in the GA stage - skipping the alpha and beta stages.
Feature gates used for this use cases might be never always enabled and removed.

An example of this use case might be feature gates to disable automatic creation of Pod Disruption Budgets or Network Policies.

### Implementation

Initially the feature gates support should be implemented in Cluster Operator.
But the same logic can be also applied to User and Topic Operators.
The feature gates will be configured in an environment variable `STRIMZI_FEATURE_GATES`.
They should be configures as a comma separated list of feature gate names prefix with `+` for enabling or `-` for disabling the feature gate.
For example `-PodDisruptionBudgets,-NetworkPolicies,+ControlPlaneListener`.
If any feature gate appears in the list more then once, the operator will throw an exception and exit.
The operator will throw an exception and exit also in case a non-existing or removed feature gate appears in the list.
Feature gates not in the list will have the default value.

The feature gates will be parsed and stored in the `ClusterOperatorConfig` class / object.
The `ClusterOperatorConfig` object is already used in different places in the operator and makes the feature gates easily accessible without any major changes to the code structure.

## Compatibility

There are no compatibility issues.

## Rejected alternatives

It was considered to be in addition able to enable / disable the feature gates per resource.
For example using annotations.
This feature should not be implemented as part of this proposal.
We should try to see how do the feature gates work in practice as per-operator configuration.
And only if we find out that per-resource configuration is necessary, it should be implemented as part of another proposal.
