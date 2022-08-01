# Auto-approval mechanism for optimization proposals

This proposal is about allowing the user to create a `KafkaRebalance` custom resource for getting an optimization proposal and running a rebalance, but without the need of the manual approval via the `strimzi.io/rebalance=approve` annotation.
It means that the user can just create the `KafkaRebalance` custom resource and the corresponding generated optimization proposal will be automatically approved.

## Current situation

Currently, when the users want to run a cluster rebalance, they have to create a `KafkaRebalance` custom resource for getting an optimization proposal first.
After getting the proposal, the only way to start the rebalancing operation is by approving it by annotating the custom resource with the `strimzi.io/rebalance=approve` annotation.
There is the need for a manual intervention of the user and this is a two steps process.

## Motivation

In some cases, the manual approval process of the rebalance proposal is not necessary.
Automatically approving an optimization proposal can save time from an operational point of view.
It enables more automation where just creating a `KafkaRebalance` custom resource can go straight to the cluster rebalance without any additional manual interaction.

## Proposal

The `KafkaRebalance` custom resource can be extended with a new `spec.autoApproval` field for this purpose.

### Auto-approval

The auto-approval can be enabled just adding an empty `spec.autoApproval` field.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: # any mode
  autoApproval: {}
```

The reason for not having a simple boolean `spec.autoApproval` field is for leaving space for a future extension, maybe adding some criteria or rules that have to be matched for auto-approving the optimization proposal.

### No auto-approval (default)

If the `spec.autoApproval` field is not specified at all, the default behavior will be the current one, so the need for manual approval by the user annotating the `KafkaRebalance` custom resource with the `strimzi.io/rebalance=approve` annotation.

### Flow

As described before, the user interaction flow assumes that the `spec.autoApproval` field is specified in the `KafkaRebalance` custom resource.

1. The user creates a `KafkaRebalance` custom resource.
2. The cluster operator asks Cruise Control to generate an optimization proposal via the REST API.
3. When the optimization proposal is ready, the cluster operator checks if the user has specified the `spec.autoApproval`.
    * If the `spec.autoApproval` is not specified at all, the user has to approve the proposal manually as usual.
    * If the `spec.autoApproval` is specified, the cluster operator approves the proposal automatically by annotating the `KafkaRebalance` custom resource with the `strimzi.io/rebalance=approve` annotation.

## Affected/not affected projects

This proposal impacts the Strimzi Cluster Operator only and mostly the `KafkaRebalanceAssemblyOperator`.

## Compatibility

The manual approval will be still in place as it is today.
As described before, if the `spec.autoApproval` field is not specified in the `KafkaRebalance` custom resource, the default behavior will be the current one, so the need for manual approval by the user.

## Rejected alternatives

Having a simple boolean `spec.autoApproval` field was rejected because the current proposal allows more extensibility for the future if it's needed to add more configuration like for example criteria or rules to be met for auto-approving the proposal.
Using a boolean, would have need an additional field for that like `spec.autoApprovalRules`.
