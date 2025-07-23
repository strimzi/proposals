# Server Side Apply - Implementation Split, Feature Gates, and Timelines

This proposal builds on the original [Server Side Apply proposal](052-k8s-server-side-apply.md), which outlines the technical specification and behavior of Kubernetes Server Side Apply (SSA).

Server Side Apply enables declarative updates to Kubernetes resources by tracking ownership of individual fields through the `.metadata.managedFields` property. 
This mechanism allows multiple controllers to safely modify different parts of the same resource without overwriting each other’s changes.

With Server Side Apply:
- Strimzi will explicitly claim ownership only of the fields it manages - by configuring the `fieldManager` field to `strimzi-kafka-operator`, which will then be tracked in the field ownership (inside `.metadata.managedFields`).
- Other tools can safely annotate or patch resources without triggering reconciliation loops.
- Reconciliation is simplified because Strimzi no longer needs to diff the entire resource or filter changes manually.

Strimzi will apply the full desired state of each resource during reconciliation and apply it using Server Side Apply, allowing the Kubernetes API server to handle merging and field ownership.
This eliminates the need for comparing actual and desired states, as well as related `GET` calls or the current `ResourceDiff` logic.

Server Side Apply operations will initially use `force = false`, retrying with `force = true` only when exception arise because of conflicts.
The exception will be logged, which will help us determine when these events may occur.
Also, it ensures Strimzi does not unintentionally take ownership of fields managed by other controllers unless explicitly intended.

## Current situation

When the original proposal was merged and a PR was opened with the implementation, we encountered complications in the implementation. 
We also discovered that the scope is too large to fit a single pull request and feature gate.
Due to inactivity on the pull request, we decided to close it - so the Server Side Apply feature has not yet been implemented.

## Motivation

Often, we receive issues in the operator repository concerning conflicts between Strimzi and other technologies. 
Typically, these conflicts arise when another operator attempts to update annotations on Custom Resources or Kubernetes resources that are managed by Strimzi.
Strimzi reverts the changes, but the other operator re-applies them, resulting in an endless loop of conflicting updates.

The Server Side Apply would resolve these issues, as described in the [original proposal](052-k8s-server-side-apply.md).
Based on community feedback, there is strong demand for Strimzi to implement Server Side Apply.

## Proposal

In order to implement Server Side Apply in Strimzi, we propose splitting the implementation of SSA support into phases.
Each phase will:

- Be controlled by its own feature gate
- Follow an independent graduation timeline
- Allow gradual adoption for specific resource types
- Enable users to test and provide feedback early
- Be scoped to the operator managing the affected resource

### Proposed Feature Gates and Timelines

We will begin by enabling Server Side Apply for the resources that most frequently cause user issues, especially those updated by other operators:

- `Service`
- `ServiceAccount`
- `ConfigMap`
- `PersistentVolumeClaim`
- `Ingress`

This initial implementation will be gated under a single feature flag:

| Feature Gate            | Resources Covered                                          | Affected Operators | Alpha (opt-in) | Beta (default-on) | GA     |
|-------------------------|------------------------------------------------------------|--------------------|----------------|-------------------|--------|
| `ServerSideApplyPhase1` | `Service`, `ServiceAccount`, `ConfigMap`, `PVC`, `Ingress` | CO                 | 0.48.0         | 0.51.0            | 0.54.0 |

Once this feature gate has been in use for some time, and based on the feedback and stability of the implementation, we plan to extend Server Side Apply support to additional resource types.
The introduction of further feature gates and the handling of additional resources will be addressed separately in future proposals.

**Note:** The release milestones above may change depending on the scheduling of Strimzi 1.0.0. 
However, each feature gate is expected to spend approximately three minor releases in each maturity phase before advancing.

## Affected/not affected projects

This proposal affects only Cluster Operator inside Strimzi operators repository - especially `cluster-operator` module with the Vert.x implementation.

## Compatibility

The implementation of Server Side Apply can cause issues, changes to the actual behavior, and it might impact how Strimzi interacts or interferes with other applications.
To not break any current deployment, each phase will be gated behind feature gate.

## Rejected alternatives

There is one rejected alternative that was described before - implementing Server Side Apply for everything in one PR and one feature gate.
