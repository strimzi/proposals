# Extend Feature Gates to all Strimzi operators

Strimzi introduced _Feature Gates_ in [Proposal no. 22](https://github.com/strimzi/proposals/blob/main/022-feature-gates.md).
Feature gates allow to enable or disable some features or change the behavior of Strimzi.
They also have a maturity model that allows the feature gates to progress as the features they introduce mature.
For more details, please check out the [original proposal](https://github.com/strimzi/proposals/blob/main/022-feature-gates.md).
The original proposal introduced feature gates initially only to the Strimzi Cluster Operator.
This proposal provides the plan for extending them to all Strimzi operators.

## Current situation

Currently, the feature gates are used only in the Cluster Operator.
The feature gates are configured using `STRIMZI_FEATURE_GATES` environment variable where the individual gates can be enabled or disabled.
The configured feature gates are used across the Cluster Operator code.
Also the code implementing them is part of the `cluster-operator` module.
The feature gates do not extent to the other operators (User and Topic Operators).

## Motivation

Although the feature gates currently don't apply to the other operators, some of them still affect them:
* For example, the `UnidirectionalTopicOperator` feature gate controlled the mode in which the Topic Operator was running.
  But it was using different environment variables to configure the right mode Topic Operator and not the feature gates configuration.
* In the future, the `UseServerSideApply` feature gate (see [Proposal no. 52](https://github.com/strimzi/proposals/blob/main/052-k8s-server-side-apply.md) for more details) will apply to at least the User Operator.

Additional feature gates that apply to the other operators might be added in the future.
To have a unified way how to manage such feature gates, this proposal provides the plan for extending the feature gates to the User and Topic Operators as well.

## Proposal

The feature gates and the `STRIMZI_FEATURE_GATES` environment variable will keep its current format.
The support for the `STRIMZI_FEATURE_GATES` environment variable will be added to the User and Topic Operators.
The same feature gates will be defined and handled by every operator.
However, if the operator doesn't have any use for the feature gate, it can simply ignore it in its code.

When the User and Topic Operators are deployed through the Cluster Operator as part of the Kafka cluster, the user will configure the feature gates as today using the `STRIMZI_FEATURE_GATES` environment variable.
And the Cluster Operator will then automatically pass the environment variable to the User and Topic Operators when deploying them.
When the User and Topic Operators are deployed as standalone, user can configure the `STRIMZI_FEATURE_GATES` environment variable directly in their deployments.

### Trade-offs and limitations

This design has several trade-offs and limitations: 
* When a feature gate is enabled or disabled in the Cluster Operator, it will cause rolling update of all the User and Topic Operators managed by it.
  This will happen even if they in reality don't use given feature gate in their code and simply ignore it.
  That said, changing the feature gates configuration is not something what is expected to be done very often.
* The centrally managed feature gate configuration will also apply to all Kafka clusters managed by given Cluster Operator.
  It cannot be applied to individual Kafka clusters which can be seen as a limitation.
  But it does not differ significantly from the current state.

### Advantages

It has also aspects that can be considered as advantages:
* The configuration of the feature gates is unchanged compared to how it works today.
  It also maintains the simplicity of the configuration.
  The user needs to set only one environment variable in the Cluster Operator to configure all feature gates.
  This also means that we do not need any changes to our system tests as they can keep using the Feature Gates in the same way as they do today.
* Having a consistency and clarity how the operators are configured simplifies support as there are less variables that need to be understood before helping our users.
* While the implementation is different, the proposed behavior mirrors the current behavior:
    * The feature gates apply to all operands (i.e. to all User and Topic Operators managed by the same Cluster Operator)
    * It follows the behavior of the `UnidirectionalTopicOperator` feature gate that was also configured centrally and changing it caused rolling update of the Topic Operator(s).
* Given that one of the goals of the feature gates is to allow users easily test new features before introducing them to everyone when they are enabled by default, the central configuration makes sure the feature gate is properly tested.
  It for example helps to avoid situation when a user would by mistake test the `UseServerSideApply` feature gate only in the Cluster Operator, but not in the other operators.
* If there is a significant demand for configuring the feature gates per-operand or more dynamically (i.e. without restarts), it can be probably better addressed as part if the [OpenFeature support](https://github.com/strimzi/strimzi-kafka-operator/issues/7520) building on top of this proposal.

### Expected source code changes

To implement the proposed changes, the feature gates source code and the feature gates definitions will move to the `operator-common` module and be shared between the 3 different operators that will support the feature gates.
The feature gates configuration will be added to the configuration classes of the User and Topic Operators.

As of today, there are no (merged) feature gates that would apply to the User or Topic Operator.
So not other changes to their source code are needed.
But it will be used as part of the server-side-apply work.

## Affected projects

This proposal affects the following components of the Strimzi Operators project:
* Cluster Operator
* User Operator
* Topic Operator
* The `operator-common` module

## Backwards compatibility

There is no impact on backwards compatibility.
The existing feature gates will remain supported and will work exactly as they worked before this change.
Also their configuration remains unchanged using the `STRIMZI_FEATURE_GATES` environment variable 

## Rejected alternatives

### Having independent and independently configurable feature gates for each operator

Each operator will have its own set of feature gates that will be configured independently.
When Strimzi users want to configure the feature gates in the User or Topic Operator managed by the Cluster Operator through the `Kafka` custom resource, they can use the container template to configure them.
For example:
```yaml
# ...
entityOperator:
  template:
    userOperatorContainer:
      env:
        - name: STRIMZI_FEATURE_GATES
          value: +UseServerSideApply
# ...
```

This would lead to increased complexity from the user perspective:
* Users will need to understand which operator supports what feature gates
* Users will need to use more complex YAML to configure the feature gates
* Understanding the overall configuration with the increased variability will make it harder to support and help our users

There will be some advantages as well.
For example the possibility to configure the feature gate only for a particular Kafka cluster.
However, this will apply only to the feature gates in the User and Topic Operators.
It will not apply to the feature gates in Cluster Operator itself that will still apply to all managed clusters.
As explained in the previous section, this can be probably better addressed by improvements such as [OpenFeature support](https://github.com/strimzi/strimzi-kafka-operator/issues/7520). 
