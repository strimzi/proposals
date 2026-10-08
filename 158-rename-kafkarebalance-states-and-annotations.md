# Rename KafkaRebalance states and annotations to make the process clearer

This proposal renames `KafkaRebalance` states and annotations to clear up confusion between the dry-run simulation and what actually executes.

## Current situation

The `KafkaRebalance` custom resource allows users to rebalance their cluster using Cruise Control.
During the rebalance process, it moves through the following states:

1. `New`: Initial state after the `KafkaRebalance` resource is created
2. `PendingProposal`: Waiting for Cruise Control to generate an optimization proposal
3. `ProposalReady`: Optimization proposal is available for review
4. `Rebalancing`: Partition reassignment is actively executing
5. `Ready`: Rebalancing completed successfully
6. `NotReady`: Rebalancing failed
7. `Stopped`: Rebalancing was stopped

Users can also apply the following annotations to interact with the `KafkaRebalance` resource:

- `strimzi.io/rebalance=approve`: Transitions from `ProposalReady` to `Rebalancing` (executes the rebalance)
- `strimzi.io/rebalance=refresh`: Re-generates the optimization proposal
- `strimzi.io/rebalance-auto-approval=true`: Automatically transitions from `ProposalReady` to `Rebalancing` without manual intervention
- `strimzi.io/rebalance=stop`: Stops the running rebalance

## Motivation

The current naming creates confusion about how Cruise Control rebalancing works.
The term `ProposalReady` suggests that users are reviewing the exact plan that will be executed.
In reality, what users see in `ProposalReady` is a **dry-run simulation** by Cruise Control based on cluster state at that moment.
When users apply `strimzi.io/rebalance=approve`, Cruise Control makes a fresh API call, generates a **new** optimization proposal based on **current** cluster state, and immediately executes it.
This new proposal can differ from the dry-run because cluster state may have changed (brokers added/removed, new topics, load shifts, etc.).
Similarly, the `approve` annotation implies approving a specific plan, when users are actually triggering a fresh execution.

This has caused:
- User misconceptions about proposal accuracy (see issue [#12199](https://github.com/strimzi/strimzi-kafka-operator/issues/12199))
- Questions about why executed rebalances differ from proposals
- Documentation challenges explaining dry-run vs execution

Using new terminology will provide:
- Clarity on what is actually happening during the rebalancing states
- Consistency with the Cruise Control terminology
- Correct expectations for users

## Proposal

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

| Current Annotation                        | New Annotation                              | Rationale                                                         |
|-------------------------------------------|---------------------------------------------|-------------------------------------------------------------------|
| `strimzi.io/rebalance=approve`            | `strimzi.io/rebalance=execute`              | "Execute" accurately describes triggering the rebalance operation |
| `strimzi.io/rebalance=refresh`            | `strimzi.io/rebalance=dry-run`              | Aligns with the dry-run terminology and intent                    |
| `strimzi.io/rebalance-auto-approval=true` | `strimzi.io/rebalance-auto-execute=true`    | Matches the `execute` terminology                                 |

**Annotations that remain unchanged:**
- `strimzi.io/rebalance=stop` - Clear and unambiguous
- `strimzi.io/rebalance-template="true"` - Used for auto-rebalancing templates, not affected by this change

### Implementation Details

Since both old and new enum values will exist in `KafkaRebalanceState` and `KafkaRebalanceAnnotation`, the existing `valueOf()` in `KafkaRebalanceUtils.rebalanceState()` will handle reading both old and new state names correctly without any changes.

In `KafkaRebalanceAssemblyOperator`, the `rebalanceAnnotation()` method will be updated to map both old and new annotation strings to the same new enum values - so `approve` and `execute` will both resolve to `KafkaRebalanceAnnotation.execute`, and `refresh` and `dry-run` will both resolve to `KafkaRebalanceAnnotation.dryrun`.
This means users applying either old or new annotation values will get the same behaviour, with a deprecation warning logged when old values are detected.
The handler methods `onPendingProposal` and `onProposalReady` will be renamed to `onDryRunInProgress` and `onDryRunComplete`.
Inside these handlers, the annotation switch cases for `approve` and `refresh` will be updated to use `execute` and `dryrun`, and all other references to old enum values will be updated throughout the class.
For the auto-execute annotation, the old key `strimzi.io/rebalance-auto-approval` will be passed as a fallback to `booleanAnnotation()` so both old and new keys will be accepted.

The `strimzi_reconciliations_*` metrics use state names as label values, so label values will change from `PendingProposal`/`ProposalReady` to `DryRunInProgress`/`DryRunComplete`.
Old metric labels will work as long as old values are supported, but will stop working once old values are removed in a future release.

### Documentation Updates

The Cruise Control concepts guide, KafkaRebalance API reference, and procedure docs (`proc-generating-optimization-proposals.adoc`, `proc-approving-optimization-proposal.adoc`) will need to be updated to reflect new state and annotation names, explain the dry-run nature of the initial proposal, and mark old values as deprecated.
The release notes will need to clearly call out the metric label change so users can update their dashboards and alerts before old values are removed.

## Affected/not affected projects

This change affects the Strimzi cluster operator, system tests, and documentation.
All other Strimzi projects are not affected as they do not interact with `KafkaRebalance`.

### Required API Changes

The following changes are required in the `api` module:

- `KafkaRebalanceState` - will add new enum values `DryRunInProgress` and `DryRunComplete` (old values `PendingProposal` and `ProposalReady` will be kept as deprecated)
- `KafkaRebalanceAnnotation` - will add new enum values `execute` and `dryrun` (old values `approve` and `refresh` will be kept as deprecated)
- `ResourceAnnotations` - will add new constant `ANNO_STRIMZI_IO_REBALANCE_AUTO_EXECUTE` (old constant `ANNO_STRIMZI_IO_REBALANCE_AUTOAPPROVAL` will be kept as deprecated)

## Compatibility

Old annotation values (`approve`, `refresh`, `strimzi.io/rebalance-auto-approval`) and old state names (`PendingProposal`, `ProposalReady`) remain supported for at least 2 releases.
The operator logs a deprecation warning whenever old values are used.
Old and new values can coexist during the transition — users can migrate at their own pace.
Deprecated values are removed in a later release.
