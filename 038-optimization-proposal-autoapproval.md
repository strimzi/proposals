# Auto-approval mechanism for optimization proposals

This proposal is about allowing the user to create a `KafkaRebalance` custom resource for getting an optimization proposal and running a rebalance, but without the need for the manual approval via the `strimzi.io/rebalance=approve` annotation.
This means that the user can simply create the `KafkaRebalance` custom resource and the corresponding generated optimization proposal will be approved automatically.

## Current situation

Currently, when the users want to run a cluster rebalance, they have to create a `KafkaRebalance` custom resource in order to generate an optimization proposal first.
After getting the proposal, the only way to start the rebalancing operation is by approving it by annotating the custom resource with the `strimzi.io/rebalance=approve` annotation.
There is the need for a manual intervention of the user and this is a two steps process.

## Motivation

In some cases, the manual approval process of the rebalance proposal is not necessary.
Automatically approving an optimization proposal can save time from an operational point of view.
It enables more automation where just creating a `KafkaRebalance` custom resource can go straight to the cluster rebalance without any additional manual interaction.

## Proposal

The `KafkaRebalance` custom resource can be annotated with a new `strimzi.io/rebalance-auto-approval` annotation for this purpose.

### Auto-approval

The auto-approval can be enabled just annotating the `KafkaRebalance` custom resource with the `strimzi.io/rebalance-auto-approval=true` annotation.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
  annotations:
    strimzi.io/rebalance-auto-approval: true
spec:
  mode: # any mode
  # ...
```

The reason for not having a simple boolean `spec.autoApproval` field is for leaving space for a future extension, maybe adding some criteria or rules that have to be matched for auto-approving the optimization proposal.
It allows the community to take more time to understand if such a support is really useful and the direction to go.

### No auto-approval (default)

If the `strimzi.io/rebalance-auto-approval` annotation is not specified at all, the default behavior will be the current one, so the need for manual approval by the user annotating the `KafkaRebalance` custom resource with the `strimzi.io/rebalance=approve` annotation.
Of course, the annotation `strimzi.io/rebalance-auto-approval` can be set to `false` to get the same result.

### Flow

As described before, the user interaction flow assumes that the `strimzi.io/rebalance-auto-approval` annotation is specified in the `KafkaRebalance` custom resource.

1. The user creates a `KafkaRebalance` custom resource.
2. The cluster operator asks Cruise Control to generate an optimization proposal via the REST API.
3. When the optimization proposal is ready, the cluster operator checks if the user has specified the `strimzi.io/rebalance-auto-approval` annotation.
    * If the `strimzi.io/rebalance-auto-approval` annotation is not specified at all or it is set to `false`, the user has to approve the proposal manually as usual.
    * If the `strimzi.io/rebalance-auto-approval` is specified and set to `true`, the cluster operator approves the proposal automatically by annotating the `KafkaRebalance` custom resource with the `strimzi.io/rebalance=approve` annotation.

## Affected/not affected projects

This proposal impacts the Strimzi Cluster Operator only and mostly the `KafkaRebalanceAssemblyOperator`.

## Compatibility

The manual approval will be still in place as it is today.
As described before, if the `strimzi.io/rebalance-auto-approval` annotation is not specified or set to `false` in the `KafkaRebalance` custom resource, the default behavior will be the current one, so the need for manual approval by the user.

## Rejected alternatives

### Using `spec.autoApproval` boolean field

Having a simple boolean `spec.autoApproval` field was rejected because it is possible that we could need more extensibility for the future if it's needed to add more configuration like for example criteria or rules to be met for auto-approving the proposal.
Using a boolean, would have need an additional field for that like `spec.autoApprovalRules`.
It is anyway possible that if, at some point, the community agree that criteria and rules won't be supported anymore, the proposed annotation will be promoted to be such a boolean field.

### Using `spec.autoApproval` object field

Having a more complex `spec.autoApproval` field was rejected because currently there is no clear plan about how supporting criteria or rules for the auto-approval process.
It is actually not clear if we want them and what is the right shape.
Going through an annotation for now allows to use the feature but having more time to think about the possible criteria and rules support.
Even in this case, it is possible that the current proposed annotation will be promoted in a such more complex field to allow more configuration related to the auto-approval process.
