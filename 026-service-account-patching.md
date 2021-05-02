# Service Account patching

This proposal suggests to introduce proper reconciliation of service accounts and handle them in the same way as other Kubernetes resources.

## Current situation

Strimzi currently does not reconcile service accounts it creates.
In the service account reconciliation loop, only creation and deletion actually does something.
Patching just returns success without doing anything.
This was originally done this way because the patching removed the attached secret and was causing a new authentication token being created every reconciliation.

## Motivation

For many resources we create, we offer customization using the template mechanism.
User can use the `template` sections in the different custom resources to customize some parts of the Kubernetes resources.
In some case, it allows customizing only labels and annotations.
In other there are more things to configure.
One of the advantages of this model is that users can declaratively configure things in the custom resources.

However, there is currently no template for service accounts.
And even if we add the template, it will work only at creation time because we do not patch service accounts right now
So that would create a weird situation where the template for service accounts behaves differently from other resources. 

At the same time, it is in many cases desired to have the labels and annotations configurable.
Some platforms - such as AWS - link their internal identities to the service accounts based on annotations.
For example, the annotation `eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<IAM_ROLE_NAME>` tells AWS that pods with this service account should have the role specified in the annotation (For more info, see [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html)).
This is especially useful for something like Kafka Connect with connectors interacting with other AWS services.
The connectors can use the AWS IAM role assumed by the annotation to authenticate against AWS services.

## Proposal

This proposal suggests several changes to how the Strimzi Cluster Operator handles the service accounts.

1) A new `serviceAccount` fields will be added to the `template` sections of the Strimzi CRDs which result in any service accounts being created (`Kafka`, `KafkaConnect`, `KafkaConnectS2I`, `KafkaMirrorMaker2`, `KafkaMirrorMaker` and `KafkaBridge`).
  It will allow customizing the labels and annotations of the service accounts in a declarative way.
2) The ServiceAccountOperator class in the `operator-common` module will be updated to patch the service accounts during reconciliation.
  To avoid issues with the tokens being recreated on every reconciliation, it will always copy the name of the token secret before patching (similarly as done for node ports in services etc.).
  That will ensure that the service accounts will not be disrupted by the patching.

Starting to patch the service accounts can cause issues to existing users.
Since we do not patch them today, lot of users simply annotate them manually after they are created or create the service accounts with the desired labels and annotations first before creating the Strimzi resources.
If we suddenly enable patching of the service account, it might remove the labels and annotations for these users and cause problems to their running applications.

To mitigate this, this proposal suggests to use [Feature Gates](https://github.com/strimzi/proposals/blob/main/022-feature-gates.md) to introduce the patching of service accounts over multiple releases and cause minimal disruption for the users.
A new feature gate `ServiceAccountPatching` will be added and disabled by default at first.
When disabled the operator will treat the service accounts as today and not patch them.
In such case, the service account template will be used only at creation.
When enabled, the operator will start patching the service accounts in every reconciliation and any changes to the service account templates will be applied.

The `ServiceAccountPatching` feature gate will mature over multiple releases as described in the proposed plan below.
Once it reaches GA, the feature gate will be removed and the service account patching will be enabled by default.
This will bring the handling of service account in-sync with how other Kubernetes resources are handled.

| Phase | Strimzi versions       | Default state                                          |
|:------|:-----------------------|:-------------------------------------------------------|
| Alpha | 0.24, 0.25, 0.26       | Disabled by default                                    |
| Beta  | 0.27, 0.28, 0.29       | Enabled by default                                     |
| GA    | 0.30 and newer         | Enabled by default (without possibility to disable it) |

_(The actual version numbers are subject to change)_

## Compatibility

The introduction of this feature is designed to minimize the compatibility impacts.

## Affected components

Only the Cluster Operator is impacted by this proposal.

## Rejected alternatives

Enabling the patching of service accounts immediately without the feature gate was considered.
But it was rejected because of the possible negative impact on users.
