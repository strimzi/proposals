# Watching multiple namespaces

Issues [#5895](https://github.com/strimzi/strimzi-kafka-operator/issues/5895) and [#879](https://github.com/strimzi/strimzi-kafka-operator/issues/879)  discuss the need to declare a `KafkaUser` or `kafkaTopic` in namespaces other than the one where the User Operator is deployed, which also applies to `KafkaConnector` resources.
First proposal on `KafkaTopic` resource: [issue #1206](https://github.com/strimzi/strimzi-kafka-operator/issues/1206)
The goal of a Kubernetes operator is to enable the creation and configuration of multiple resources across Kubernetes namespaces. 
This proposal seeks to extend that capability to `KafkaUser`, `KafkaTopic`, and `KafkaConnector` resources so that they can be effectively managed in namespaces beyond where the operator is deployed.

## Current situation

Currently, Strimzi has limited support for managing resources across namespaces. For `KafkaUser` and `KafkaTopic` resources, Strimzi deploys the User Operator and Topic Operator in the same namespace as the `Kafka` resource.
For `KafkaConnector` it's almost the same thing except that `cluster-operator` has the capacity to detect `kafkaConnector` resources in all namespaces, but after have found a `KafkaConnector` resource it search a `KafkaConnect` cluster in the namespace of this ressources and it doesn't exists because the `KafkaConnector` is deploy in the same namespace of `Kafka` resource.

## Motivation

In a multi-tenant Kubernetes cluster, it is common practice to define authorization at namespace level or to deploy many applications in different namespaces. In scenarios where a Kafka cluster is shared among multiple applications or customers, it is more flexible and secure to deploy each application in its own dedicated namespace.

Strimzi allows us to create a Helm chart for our applications that automatically deploy and configure customer-specific `KafkaUser`, `kafkaTopic` and `KafkaConnector` resources. However, all customer resources must reside in the same namespace where the `Kafka` resource is deployed. This means all customer-specific resources must be managed in the same namespace, which becomes difficult to manage at scale with multiple customers—whether it’s a single team managing hundreds of namespaces with different applications, users, and topics, or multiple teams wanting to self-service their topics and users in a shared Kafka cluster across namespaces.

Allowing customer-specific resources to be deployed in separate namespaces  simplifies management, enabling quotas and limits to be configured at the namespace level, and enhancing security (for customer-specific and  Kafka namespaces) with network policies. Most importantly, it provides isolation between Kafka components and customer applications, so that each customer’s resources are separated and easier to manage.


## Proposal

This proposal addresses [issue #1206](https://github.com/strimzi/strimzi-kafka-operator/issues/1206), but attempts to apply it on `kafkaUser, KafkaTopic, and KafkaConnector` resources.


### KafkaTopic

Create a new `CustomResourceDefinition` to handle topics that are created in namespaces, called `KafkaNamespaceTopic`. This CRD will allow the user to create a topic that will be automatically namespaced within Kafka by the Topic Operator. It will encode the actual topic name using the Kubernetes namespace in which the resource resides, and either the `metadata.name` or `spec.topicName` (if defined) of the instance of a `KafkaNamespaceTopic`.

`{Kubernetes namespace name}.{metadata.name|spec.topicName}`

> NOTE: Kubernetes namespaces and `metadata.names` cannot use the character `.`

The `KafkaNamespaceTopic` will include all configuration that is available in a `KafkaTopic` and the `strimzi.io/cluster` labels for apply this topic to an existing `Kafka` cluster.

The Kafka CRD will be extended to include an additional property in the topicOperator specification that includes a YAML list value (or the `*` must be used for watch all namespaces.) of all the namespaces that are watched for `KafkaNamespaceTopic` CRD’s, called `watchNamespaceTopics` or the `*` must be used for watch all namespaces.

`entityOperator.topicOperator.watchNamespaceTopics: [{Namespace 1},...,{Namespace N}] or *`

When a `KafkaNamespaceTopic` is created in a watched namespace then the Topic Operator will perform the following actions.

- Create the topic in Kafka.
- Update the `KafkaNamespaceTopic` with the actual topic name (Ex. myproject.mytopic). A label could be added to the resource with the fully encoded name of the topic, strimzi.io/namespace-topic-name.

When a `KafkaNamespaceTopic` is updated in a watched namespace then the Topic Operator will reconcile the changes to the topic in Kafka.

When a `KafkaNamespaceTopic` is deleted in a watched namespace then the Topic Operator will delete the topic in Kafka.

Unlike KafkaTopics, if changes to underlying Kafka topics are made out of band of the `KafkaNamespaceTopic` CRD instance then the operator will not synchronize those changes to the `KafkaNamespaceTopic`.

This proposal preserves the existing behaviour of KafkaTopic for backwards compatibility. Documentation updates can be made to recommend that the user use `KafkaNamespaceTopic` for any topics they want to manage within a Kubernetes application’s lifecycle and use KafkaTopic only for high level administration of topics within Kafka.

### KafkaUser

Create a new `CustomResourceDefinition` to handle users that are created in namespaces, called `KafkaNamespaceUser`. This CRD will allow the user to create a user that will be automatically namespaced within Kafka by the User Operator. It will encode the actual user name using the Kubernetes namespace the resource resides and the `metadata.name` of the instance of a `KafkaNamespaceUser`.

`{Kubernetes namespace name}.{metadata.name}`

> NOTE: Kubernetes namespaces and `metadata.names` cannot use the character `.`

The `KafkaNamespaceUser` will include all configuration that is available in a `KafkaUser` and the `strimzi.io/cluster` labels for apply this user to an existing `Kafka` cluster

The Kafka CRD will be extended to include an additional property in the userOperator specification that includes a YAML list value (or the `*` must be used for watch all namespaces.) of all the namespaces that are watched for `KafkaNamespaceUser` CRD’s, called `watchNamespaceUsers`.

`entityOperator.userOperator.watchNamespaceUsers: [{Namespace 1},...,{Namespace N}] or *`

When a `KafkaNamespaceUser` is created in a watched namespace then the User Operator will perform the following actions.

- Create the user in Kafka.
- Create the KafkaUser secret in the namespace of `KafkaNamespaceUser` resource
- If `KafkaUser` use a mtls configuration, when the Strimzi Certificate Authority / certificate client is renewed, the user secret must be updated.
- Update the `KafkaNamespaceUser` with the actual user name (Ex. myproject.myuser). A label could be added to the resource with the fully encoded name of the user, strimzi.io/namespace-user-name.

When a `KafkaNamespaceUser` is updated in a watched namespace then the User Operator will reconcile the changes to the user in Kafka.

When a `KafkaNamespaceUser` is deleted in a watched namespace then the User Operator will delete the user in Kafka.

Unlike KafkaUsers, if changes to underlying Kafka users are made out of band of the `KafkaNamespaceUser` CRD instance then the operator will not synchronize those changes to the `KafkaNamespaceUser`.

This proposal preserves the existing behaviour of KafkaUser for backwards compatibility. Documentation updates can be made to recommend that the user use `KafkaNamespaceUser` for any users they want to manage within a Kubernetes application’s lifecycle and use KafkaUser only for high level administration of users within Kafka.


### KafkaConnector 

Create a new `CustomResourceDefinition` to handle connectors that are created in namespaces, called `KafkaNamespaceConnector`. This CRD will allow the user to create a connector that will be automatically namespaced within Kafka by the Cluster Operator. It will encode the actual connector name using the Kubernetes namespace the resource resides and  the `metadata.name` of the instance of a `KafkaNamespaceConnector`.

`{Kubernetes namespace name}.{metadata.name}`

> NOTE: Kubernetes namespaces and `metadata.names` cannot use the character `.`

The `KafkaNamespaceConnector` will include all configuration that is available in a `KafkaConnector` and the `strimzi.io/cluster` labels for apply this connector to an existing `KafkaConnect` cluster

The KafkaConnect CRD will be extended to include an additional property in the specification that includes a YAML list value (or the `*` must be used for watch all namespaces.) of all the namespaces that are watched for `KafkaNamespaceConnector` CRD’s, called `watchNamespaceConnectors`.

`spec.watchNamespaceConnectors: [{Namespace 1},...,{Namespace N}] or *`

When a `KafkaNamespaceConnector` is created in a watched namespace then the Cluster Operator will perform the following actions.

- Create the connector in KafkaConnect.
- Update the `KafkaNamespaceConnector` with the actual connector name (Ex. myproject.myconnector). A label could be added to the resource with the fully encoded name of the connector, strimzi.io/namespace-connector-name.

When a `KafkaNamespaceConnector` is updated in a watched namespace then the Cluster Operator will reconcile the changes to the connector in Kafka.

When a `KafkaNamespaceConnector` is deleted in a watched namespace then the Cluster Operator will delete the connector in Kafka.

Unlike KafkaConnectors, if changes to underlying Kafka connectors are made out of band of the `KafkaNamespaceConnector` CRD instance then the operator will not synchronize those changes to the `KafkaNamespaceConnector`.

This proposal preserves the existing behaviour of KafkaConnector for backwards compatibility. Documentation updates can be made to recommend that the user use `KafkaNamespaceConnector` for any users they want to manage within a Kubernetes application’s lifecycle and use KafkaConnector only for high level administration of connectors within Kafka.

## RBAC

Cluster operator for `KafkaConnector` seems to have the good rbac with the clusterRole `strimzi-cluster-operator-watched`:

```yaml
  resources:
  - kafkas
  - kafkanodepools
  - kafkaconnects
  - kafkaconnectors
  - kafkamirrormakers
  - kafkabridges
  - kafkamirrormaker2s
  - kafkarebalances
  verbs:
  - get
  - list
  - watch
  - create
  - patch
  - update

```

But for TopicOperator and UserOperator it will be necessary to create RBAC for CRUD operations in `watchNamespaceUser|watchNamespaceTopic` or all namespace if `*` it's used. If `*` it's used we can create a clusterRole and clusterRoleBinding either by `helm chart` or by `kubernetes manifest`. for CRUD in all namespaces. If a list of namespace is provide then we create a role `cluster-entity-operator` and rolebinding in each namespace.

## Affected/not affected projects

The only affected projects are:
- CO
- TO
- UO

## Compatibility

The `kafkaNamespaceTopic`, `KafkaNamespaceConnector` and `KafkaNamespaceUser` crds will consume the specifications of the standard `kafkaTopic,KafkaUser?KafkaConnector` crds. In fact, they will be technically compatible and will not cause a break in compatibility. 

However, it will be necessary to ensure that changes made to the `kafkaTopic,KafkaUser,KafkaConnector` CRDS do not compromise the operation of the new CRDS.

As things stand, the configuration of these new CRDS should not have any impact on existing CRDS.

## Rejected alternatives

The addition of a `strimzi.io/cluster-namespace: mynamespacewhereismycluster` label on resources, enabling TO/UO/CO to create the necessary configurations if this label is present.

In the case of the operator cluster, when a connector is created in a different namespace from where the KafkaConnect cluster is deployed, thanks to the `strimzi.io/cluster-namespace: mynamespacewhereismycluster` label it would know where its KafkaConnect cluster was located and could have launched the reconciliation tasks.

In the case of the Topic Operator and User Operator, we could have imagined using the same method as for the cluster operator, i.e. being able to monitor the resources of all the namespaces and still using this label to find the Kafka cluster on which to configure the resources. This would require a change to the reconciliation tasks for these two operators and to the rbac as well. I've ruled this out because I think it goes beyond the way operators work in Kubernetes, which by default can monitor and configure resources in multiple namespaces.
