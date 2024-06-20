# `KafkaRebalance` custom resource deletion handling

This proposal introduces a better mechanism of handling the deletion attempt of a `KafkaRebalance` custom resource by the user.
It's about providing a consistent user experience on getting feedback on the status of an ongoing operation on Cruise Control.

## Current situation

When the user creates a `KafkaRebalance` custom resource for getting an optimization proposal from Cruise Control and then running the corresponding actual rebalancing, such custom resource is also used to provide feedback on what's the status of the ongoing operation.
The `KafkaRebalance.status` includes a condition which uses the `type` field to describe the status of the request such as `PendingProposal`, `ProposalReady`, `Rebalancing`, `Ready` (rebalancing done), `NotReady` (error) and `Stopped`.
When the operation is done (proposal ready or rebalancing done), the `KafkaRebalance.status` also includes details about the optimization itself.
But during a proposal elaboration (see `ProposalPending` state) or a rebalancing running (see `Rebalancing` state), the user has the freedom to just delete the `KafkaRebalance` custom resource while the operation is still going on Cruise Control.

## Motivation

When the user deletes the `KafkaRebalance` custom resource and the corresponding operation is still running on Cruise Control, the user loses any possibility to get feedback about the status of the operation itself.
The only way they have is to take a look at the Cruise Control log which is not that great.
They also lose the opportunity to interact with Cruise Control, for example for stopping the ongoing operation.

## Proposal

This proposal is about using a finalizer on the `KafkaRebalance` custom resource to avoid its deletion when there is a corresponding operation still running in Cruise Control.

### Request for an optimization proposal

When the user creates the `KafkaRebalance` custom resource, the `KafkaRebalanceAssemblyOperator` adds a `strimzi.io/rebalance` finalizer to its metadata, then the corresponding request is issued to Cruise Control and the custom resource status moves to `PendingProposal`.
While Cruise Control is elaborating the optimization proposal, any attempts done by the user of deleting the `KafkaRebalance` custom resource is blocked by the Kubernetes API server because of the finalizer.
The user is still able to receive any feedback about how the operation is going on thanks to the `KafkaRebalanceAssemblyOperator` querying Cruise Control on each reconciliation and updating the `KafkaRebalance` custom resource.
When the optimization proposal processing ends, and the `KafkaRebalance` custom resource status moves to `ProposalReady`, the `KafkaRebalanceAssemblyOperator` removes the finalizer which unlocks the deletion (if the user tried to do so).

### Request for running the rebalancing

The same flow applies when the user approves a ready proposal and the `KafkaRebalance` custom resource moves to `Rebalancing` after the request to run a rebalancing is issued to Cruise Control.
The `KafkaRebalanceAssemblyOperator` applies the `strimzi.io/rebalance` finalizer which is removed only when the rebalancing is done and the status moves to `Ready`.
Any attempts by the user to delete the `KafkaRebalance` custom resource is blocked by the Kubernetes API server because of the finalizer.
The user is still able to receive any feedback about how the operation is going on thanks to the `KafkaRebalanceAssemblyOperator` querying Cruise Control on each reconciliation and updating the `KafkaRebalance` custom resource.

### Refreshed, errored and stopped rebalance

For both the previous scenarios, the `KafkaRebalanceAssemblyOperator` also adds the finalizer if the user applies the `strimzi.io/rebalance: refresh` annotation on the `KafkaRebalance` custom resource when it's in `ProposalReady` or `Ready`, asking to elaborate a new proposal.

Furthermore, the `KafkaRebalanceAssemblyOperator` removes the finalizer if the custom resource moves to a `NotReady` state because of an error or to the `Stopped` state because the user asked to stop the current operation.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator and more specifically the `KafkaRebalanceAssemblyOperator` class by adding the logic for handling the finalizer on the `KafkaRebalance` custom resource.

## Compatibility

Related to the `KafkaRebalance` custom resource deletion, the user experience is going to be different.
The user is not able to delete the custom resource straight away anymore but it will be blocked by the Kubernetes API server until the ongoing operation on Cruise Control reaches a stable state.
Of course, it can be considered a behavioral change from a user perspective but probably not a breaking change if we assume the current behavior is buggy.

## Rejected alternatives

No other proposed alternatives.
