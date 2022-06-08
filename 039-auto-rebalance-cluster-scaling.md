# Automatic rebalance on cluster scaling

This feature is about enabling the cluster operator to get an optimization proposal, which is auto-approved, in order to run the corresponding rebalancing when the user scales the cluster up or down by changing the `spec.kafka.replicas` field on the `Kafka` custom resource.
Such a rebalancing can happen right after the scale up to move some partition replicas to the newly added brokers or right before the scale down to move partition replicas off of the brokers to be removed.

## Current situation

Currently, there is no automatic rebalancing when the user scales the cluster up or down.
It means that:

* after scaling up the cluster, in order to rebalance the cluster and move some partition replicas to the newly added brokers, the user has to create a `KafkaRebalance` custom resource using the `spec.mode: add-brokers` mode and add the new broker IDs to the the `spec.brokers` field.
* before scaling down the cluster, in order to rebalance the cluster and move partition replicas off of the brokers to be removed, the user has to create a `KafkaRebalance` custom resource using the `spec.mode: remove-brokers` mode and add the broker IDs of the brokers to be removed to the `spec.brokers` field.

For more details, the proposal [035](https://github.com/strimzi/proposals/blob/main/035-rebalance-types-scaling-brokers.md) introduced the rebalancing modes mentioned above.

## Motivation

The rebalancing operation during scaling up and down is a manual two steps process.

* scaling up the cluster, then running a rebalance.
* running a rebalance, then scaling down the cluster.

In order to simplify the process, it would be great for the user to deal with just scaling the cluster by changing the `spec.kafka.replicas` field and then having the operator taking care of running a rebalancing operation automatically if requested.

Taking into account the `StrimziPodSets` usage (instead of the `StatefulSets`), which is the default since the [0.30.0](https://github.com/strimzi/strimzi-kafka-operator/releases/tag/0.30.0) release, a more general use case will be about adding new brokers and/or removing existing ones, not talking just about scaling up and down anymore.

This proposal takes into account this future plan by referring to more general operations like add and remove brokers instead of scaling up and down.

## Proposal

A new `KafkaRebalanceTemplate` custom resource could be a solution allowing the user to define the configuration for the auto-rebalancing but leaving to the operator the creation of the corresponding `KafkaRebalance` custom resource.
This new CRD should be quite similar to the `KafkaRebalance` one.
The cluster operator has to copy over the configuration from the template to the actual rebalancing resource.
Compared to the rejected alternative, of adding the configuration to the `Kafka` custom resource, this will avoid polluting the `Kafka` custom resource and increasing its size.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalanceTemplate
metadata:
  name: my-rebalance-template
spec:
  # NOTE: mode and brokers fields are not available as in the KafkaRebalance,
  #       they are automatically set on the corresponding KafkaRebalance by the operator
  goals:
    - CpuCapacityGoal
    - NetworkInboundCapacityGoal
    - DiskCapacityGoal
    - RackAwareGoal
    - MinTopicLeadersPerBrokerGoal
    - NetworkOutboundCapacityGoal
    - ReplicaCapacityGoal
  skipHardGoalCheck: true
  # ... other rebalancing related configuration
```

The user has the flexibility to specify the [goals and the other rebalancing related configurations](https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaRebalance-reference) the same way they already do today on a `KafkaRebalance` custom resource.
This allows the user to not rely on the default Cruise Control goals configuration which would be common for any rebalance without overriding goals.
They could define different goals based on the status of the Kafka cluster or skip hard goals in some scenarios for examples.
As already mentioned, specifying goals but more in general customizing the optimization proposal request is already possible in the current usage of a `KafkaRebalance` custom resource so it should be allowed for the auto-rebalancing as well.
In the end, the cluster operator will use that configuration from the `KafkaRebalanceTemplate` to fill the corresponding `KafkaRebalance` but adding the `spec.mode` and `spec.brokers` accordingly.
It will also add the `spec.autoApproval` field, as described by this [proposal](https://github.com/strimzi/proposals/pull/56), for enabling auto-approval.

We would also need to extend the `Kafka` custom resource to allow the user specifying that he wants auto-rebalancing on scaling by using a template.
A new `spec.cruiseControl.autoRebalance` field could be added for this purpose.
Because we are talking about auto-rebalancing, the corresponding field could also be not limited to the rebalancing on scaling but a more general full "continuous” auto-rebalancing to enable on the cluster.

The `spec.cruiseControl.autoRebalance` field allows to specify a list of auto-rebalancing modes with the corresponding templates to be used.
To specify when running the auto-rebalancing, a `spec.cruiseControl.autoRebalance.mode` field could be used to specify if running auto-rebalancing only on adding or removing brokers, or a full rebalance is requested.
Having a full auto-rebalancing is out of scope for this proposal, the field is added for enabling a future implementation.

The template to be used is referenced by a new mandatory `spec.cruiseControl.autoRebalance.template` field, as a [LocalObjectReference](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/local-object-reference/), and the cluster operator uses that template to create a corresponding `KafkaRebalance` custom resource.
If the referenced `KafkaRebalanceTemplate` custom resource doesn't exist, the cluster operator will fail the reconcile.

In the following example, a user can decide to have different rebalancing templates when adding or removing brokers and when they want a full rebalancing during the normal cluster execution.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    # ...
  cruiseControl:
    # ...
    autoRebalance:
      # using the same rebalancing template when adding or removing brokers
      - mode: add-brokers
        template: 
          name: my-add-remove-brokers-rebalancing-template
      - mode: remove-brokers
        template: 
          name: my-add-remove-brokers-rebalancing-template  
      # using a different template for a full rebalancing  
      - mode: full
        template: 
          name: my-full-rebalancing-template
```

The user can even decide to not have full auto-rebalancing and, for example, using different templates when adding or removing brokers.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    # ...
  cruiseControl:
    # ...
    autoRebalance:
      # using a specific template when adding brokers
      - mode: add-brokers
        template: 
          name: my-add-brokers-rebalancing-template
      # using a different template when removing brokers  
      - mode: remove-brokers
        template: 
          name: my-remove-brokers-rebalancing-template
      # NOTE: no full auto-rebalancing   
```

The user could even decide to have auto-rebalancing only when adding or removing brokers but not for both.

This way provides the greatest flexibility to describe in which case to run the auto-rebalancing and with which configuration.

### Auto-rebalancing on scaling up

Following the operator flow when auto-rebalancing on scaling up.

1. The user scales up the cluster by increasing the `spec.kafka.replicas` field.
2. The `KafkaReconciler` takes care of it by scaling up the cluster as usual, by adding additional brokers.
3. If the `spec.cruiseControl.autoRebalance` contains an entry with `add-brokers` mode, the `KafkaReconciler` creates a `KafkaRebalance` custom resource named as `<cluster-name>-auto-rebalance-add-brokers` by using the corresponding template and sets:
   * the `spec.mode` as `add-brokers` and the `spec.brokers` with the list of brokers already added to the cluster.
   * the `spec.autoApproval` as described by this [proposal](https://github.com/strimzi/proposals/pull/56).
4. The `KafkaRebalanceAssemblyOperator` detects the new rebalancing resource and acts as usual by requesting Cruise Control to generate an optimization proposal via the REST API on the `/add_broker` endpoint.
5. When the optimization proposal is ready, the `KafkaRebalanceAssemblyOperator` approves it automatically, because of the `spec.autoApproval` field, by annotating the `KafkaRebalance` custom resource with the `strimzi.io/rebalance=approve` annotation as defined by this [proposal](https://github.com/strimzi/proposals/pull/56).
6. If/When the rebalancing ends successfully, the corresponding `KafkaRebalance` custom resource is deleted by the `KafkaReconciler`.

#### Error handling

During the above flow, errors can happen after the scaling up:

* There is an error when getting the proposal.
* After auto-approving the proposal, there is an error when running the rebalance.

In both scenarios, the reconciliation ends with the `KafkaRebalance` custom resource in the `NotReady` state as per the `KafkaRebalanceAssemblyOperator` states machine.
The cluster operator logs a warning about the rebalancing not happened.
It's the user who should deal with this scenario:

* Checking the reason why the `KafkaRebalance` is in the `NotReady` state and deleting the custom resource.
* If the user still wants rebalancing, they have to create a new `KafkaRebalance` custom resource manually.

### Auto-rebalancing on scaling down

1. The user scales down the cluster by decreasing the `spec.kafka.replicas` field.
2. If the `spec.cruiseControl.autoRebalance` contains an entry with `remove-brokers` mode, the `KafkaReconciler` creates a `KafkaRebalance` custom resource named as `<cluster-name>-auto-rebalance-remove-brokers` by using the corresponding template and sets:
   * the `spec.mode` as `remove-brokers` and the `spec.brokers` with the list of brokers that will be removed.
   * the `spec.autoApproval` as described by this [proposal](https://github.com/strimzi/proposals/pull/56).
3. The `KafkaRebalanceAssemblyOperator` detects the new rebalancing resource and acts as usual by requesting Cruise Control to generate an optimization proposal via the REST API on the `/remove_broker` endpoint.
4. When the optimization proposal is ready, the `KafkaRebalanceAssemblyOperator` approves it automatically, because of the `spec.autoApproval` field, by annotating the `KafkaRebalance` custom resource with the `strimzi.io/rebalance=approve` annotation as defined by this [proposal](https://github.com/strimzi/proposals/pull/56).
5. If/When the rebalancing ends successfully, the corresponding `KafkaRebalance` custom resource is deleted by the `KafkaReconciler`.
6. Finally, the `KafkaReconciler` can continue to scale down the cluster by deleting the brokers.

#### Error handling

During the above flow, errors can happen before the scaling down:

* There is an error when getting the proposal.
* After auto-approving the proposal, there is an error when running the rebalance.

In both scenarios, the reconciliation ends with the `KafkaRebalance` custom resource in the `NotReady` state as per the `KafkaRebalanceAssemblyOperator` states machine.
The cluster operator logs a warning about the rebalancing not happened.

The reconcile ends but the scale down cannot proceed because of the brokers to be removed are still hosting some partition replicas.
This behavior is related to this [proposal](https://github.com/strimzi/proposals/pull/54).

On the next reconciliation, the cluster operator will try to do a scale down again, so running the flow as before.
The previous `KafkaRebalance` custom resource in the `NotReady` state is overridden to generate a new optimization proposal.

Of course, the user could change his mind by reverting back the `spec.kafka.replicas` field to avoid the scale down.

## Affected/not affected projects

This proposal impacts the Strimzi Cluster Operator only.
The `KafkaReconciler` is impacted on handling the `KafkaRebalanceTemplate` custom resource and generating/deleting the corresponding `KafkaRebalance`.
The `KafkaRebalanceAssemblyOperator` is impacted mostly on the auto-approval side but it is already defined by this [proposal](https://github.com/strimzi/proposals/pull/56).
It handles the `KafkaRebalance` custom resource the same way as it was created by the user but it has to annotate the custom resource automatically.

## Compatibility

The auto-rebalancing on scaling up and down is optional.
The user has to specify the `spec.cruiseControl.autoRebalance` on the `Kafka` custom resource for the desire of having the operator doing the rebalancing automatically.
Without that addition, the rebalancing can be done with the manual two step process as today.

## Rejected alternatives

When the `spec.cruiseControl.autoRebalance` is specified, the cluster operator can create an “empty” `KafkaRebalance` custom resource automatically.
By “empty” custom resource we mean that the default Cruise Control configuration will be used for goals and all other options.
The only fields that the cluster operator has to specify, when creating the `KafkaRebalance` custom resource, based on scaling up or down, are the:

* `spec.mode` : should be `add-brokers` or `remove-brokers` if scaling up or down.
* `spec.brokers` : with the list of brokers to add/remove.
* `spec.autoApproval` : for allowing the auto-approval as defined by this [proposal](https://github.com/strimzi/proposals/pull/56).

The drawback of this solution is that if the user wants to customize the rebalancing, he has to use the `spec.cruiseControl.config` field in the Kafka custom resource. It’s important to mention that it affects the overall Cruise Control configuration so that any other “empty” rebalancing request, not related to scaling, will use the same options.
Users won't also be able to skip hard goals to move forward with a rebalancing if they are not met anyway.

To allow customization, a potential solution would be to add rebalancing related configuration (goals, and so on) under the `spec.cruiseControl.autoRebalance` directly.
It would be a kind of copy of the fields from the `KafkaRebalance` custom resource excluding the mode and brokers.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    # ...
  cruiseControl:
    # ...  
    autoRebalance:
      # using the same rebalancing configuration when adding or removing brokers ... a lot of duplication
      - mode: add-brokers
        goals:
          - CpuCapacityGoal
          - NetworkInboundCapacityGoal
          - DiskCapacityGoal
          - RackAwareGoal
          - MinTopicLeadersPerBrokerGoal
          - NetworkOutboundCapacityGoal
          - ReplicaCapacityGoal
        skipHardGoalCheck: true
        # ... other rebalancing related configuration
      - mode: remove-brokers
        goals:
          - CpuCapacityGoal
          - NetworkInboundCapacityGoal
          - DiskCapacityGoal
          - RackAwareGoal
          - MinTopicLeadersPerBrokerGoal
          - NetworkOutboundCapacityGoal
          - ReplicaCapacityGoal
        skipHardGoalCheck: true
        # ... other rebalancing related configuration  
      - mode: full
        goals:
          - CpuCapacityGoal
          - NetworkInboundCapacityGoal
          - DiskCapacityGoal
        skipHardGoalCheck: false
        # ... other rebalancing related configuration  
```

For handling the auto-approval, a `spec.autoApproval` field, as described by this [proposal](https://github.com/strimzi/proposals/pull/56), would be needed as well.
This proposal was rejected because it pollutes the `Kafka` custom resource with too many details and configuration related to a rebalancing operation when the user wants to specify specific desires compared to the default ones in the Cruise Control configuration.
To understand how big the configuration for each rebalancing mode could be, [here](https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaRebalance-reference) the options that we support today which could anyway increase in the future.
