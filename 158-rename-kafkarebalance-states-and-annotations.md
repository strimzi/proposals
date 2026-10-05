# Rename KafkaRebalance states and annotations to make the process clearer

This proposal aims to change the naming of `KafkaRebalance` states and annotations to address the user confusion about the relationship between the displayed optimization proposal (a dry-run simulation) and what actually executes.

## Current situation

The `KafkaRebalance` custom resource allows the users to rebalance their cluster using Cruise Control 
During the rebalance process, it moves through the following states:

1. `New`: Initial state after the `KafkaRebalance` resource is created
2. `PendingProposal`: Waiting for Cruise Control to generate an optimization proposal
3. `ProposalReady`: Optimization proposal is available for review
4. `Rebalancing`: Partition reassignment is actively executing
5. `Ready`: Rebalancing completed successfully
6. `NotReady`: Rebalancing failed
7. `Stopped`: Rebalancing was stopped

Users can also apply the following annotation to interact with `KafkaRebalance` resource:

- `strimzi.io/rebalance=approve`: Transitions from `ProposalReady` to `Rebalancing` (executes the rebalance)
- `strimzi.io/rebalance=refresh`: Re-generates the optimization proposal
- `strimzi.io/rebalance-auto-approval=true`: Automatically transitions from `ProposalReady` to `Rebalancing` without manual intervention
- `strimzi.io/rebalance=stop`: Stops the running rebalance

## Motivation

The current naming creates confusion about how Cruise Control rebalancing works.
The term `ProposalReady` currently suggests that an optimization proposal is generated and the users are going to apply that proposal.
But in reality, what users see in `ProposalReady` is a **dry-run simulation** by Cruise Control based on cluster state at that moment.
When users apply `strimzi.io/rebalance=approve`, Cruise Control makes a fresh API call, generates a **new** optimization proposal based on **current** cluster state, and immediately executes it.
This new proposal can differ from the dry-run because cluster state may have changed (brokers added/removed, new topics, load shifts, etc.).
When the rebalance resource is in `PendingProposal`, it doesn't mean that a new optimization proposal is being generated, it means that a **dry-run** is currently in progress by Cruise Control. 
In a similar way, the `approve` annotation implies that we are approving a specific optimization plan, when users are actually triggering execution with a fresh optimization proposal.

This has caused:
- User misconceptions about proposal accuracy (see issue [#12199](https://github.com/strimzi/strimzi-kafka-operator/issues/12199))
- Questions about why executed rebalances differ from proposals
- Documentation challenges explaining dry-run vs execution

Using a new terminology for the states and annotation will provide:
- Clarity on what is actually happening during the rebalancing states
- Consistency with the Cruise Control terminology
- Correct expectation for the users

## Proposal

This proposal introduces new names for KafkaRebalance states and annotations that correctly reflect the dry-run nature of optimization proposals and what action is actually being performed.

### State Renamings

| Current State     | New State          | Rationale                                                  |
|-------------------|--------------------|------------------------------------------------------------|
| `PendingProposal` | `DryRunInProgress` | Clearly indicates a dry-run simulation is being calculated |
| `ProposalReady`   | `DryRunComplete`   | Indicates the dry-run is completed                         |

**States that remain unchanged:**
- `New` - Clear initial state
- `Rebalancing` - Accurately describes active partition movement
- `Ready` - Standard terminal success state
- `NotReady` - Standard terminal failure state
- `Stopped` - Clear terminal state for manual stops

### Annotation Renamings

| Current Annotation                        | New Annotation                             | Rationale                                                            |
|-------------------------------------------|--------------------------------------------|----------------------------------------------------------------------|
| `strimzi.io/rebalance=approve`            | `strimzi.io/rebalance=execute`             | "Execute" accurately describes triggering the rebalance operation    |
| `strimzi.io/rebalance=refresh`            | `strimzi.io/rebalance=dry-run`             | Aligns with the dry-run terminology and intent                       |
| `strimzi.io/rebalance-auto-approval=true` | `strimzi.io/auto-dry-run-and-execute=true` | Correctly tells we will first dry-run and then execute the rebalance |

**Annotations that remain unchanged:**
- `strimzi.io/rebalance=stop` - Clear and unambiguous
- `strimzi.io/rebalance-template="true"` - Used for auto-rebalancing templates, not affected by this change

### Implementation Details

Since both old and new enum values exist in `KafkaRebalanceState` and `KafkaRebalanceAnnotation` during Phase 1, the existing `valueOf()` in `KafkaRebalanceUtils.rebalanceState()` handles reading both old and new state names correctly without any changes.

In `KafkaRebalanceAssemblyOperator`, the `rebalanceAnnotation()` method is updated to map both old and new annotation strings to the same new enum values - so `approve` and `execute` both resolve to `KafkaRebalanceAnnotation.execute`, and `refresh` and `dry-run` both resolve to `KafkaRebalanceAnnotation.dryrun`. This means users applying either the old or new annotation value get the same behaviour, with a deprecation warning logged when old values are detected. The handler methods `onPendingProposal` and `onProposalReady` are renamed to `onDryRunInProgress` and `onDryRunComplete`. Inside these handlers, the annotation switch cases for `approve` and `refresh` are updated to use `execute` and `dryrun`, and all other references to old enum values are updated throughout the class. For the auto-execute annotation, the old key `strimzi.io/rebalance-auto-approval` is passed as a fallback to `booleanAnnotation()` so both old and new keys are accepted.

The `strimzi_reconciliations_*` metrics use state names as label values, so the label values will change from `PendingProposal`/`ProposalReady` to `DryRunInProgress`/`DryRunComplete`. Old metric labels will work as long as old values are supported, but will stop working once old values are removed in a future release.

### Backward Compatibility

Old annotation values (`approve`, `refresh`, `strimzi.io/rebalance-auto-approval`) and old state names (`PendingProposal`, `ProposalReady`) remain supported for at least 2 releases. The operator logs a deprecation warning whenever old values are used. Old and new values can coexist during the transition — users can migrate at their own pace. Deprecated values are removed in a later release.

### Documentation Updates

The Cruise Control concepts guide, KafkaRebalance API reference, and procedure docs (`proc-generating-optimization-proposals.adoc`, `proc-approving-optimization-proposal.adoc`) need to be updated to reflect new state and annotation names, explain the dry-run nature of the initial proposal, and mark old values as deprecated. The release notes should clearly call out that the `strimzi_reconciliations_*` metric label values for state will change from `PendingProposal`/`ProposalReady` to `DryRunInProgress`/`DryRunComplete`. Old metric labels will work as long as old values are supported, but will stop working once they are removed in a future release. Users should update their dashboards and alerts before then.

## Affected/not affected projects

This change affects the Strimzi cluster operator, system tests, and documentation. 

### Required API Changes

The following changes are required in the `api` module:

- `KafkaRebalanceState` - add new enum values `DryRunInProgress` and `DryRunComplete` (old values `PendingProposal` and `ProposalReady` kept as deprecated)
- `KafkaRebalanceAnnotation` - add new enum values `execute` and `dryrun` (old values `approve` and `refresh` kept as deprecated)
- `ResourceAnnotations` - add new constant `ANNO_STRIMZI_IO_REBALANCE_AUTO_EXECUTE` (old constant `ANNO_STRIMZI_IO_REBALANCE_AUTOAPPROVAL` kept as deprecated)

## Compatibility

This change is backward compatible - old annotation values and state names remain supported for at least 2 releases before being removed.

