# Multi-namespace watching for the Topic Operator

Issue [#1206](https://github.com/strimzi/strimzi-kafka-operator/issues/1206)

## Current situation

The Unidirectional Topic Operator (UTO, [proposal 051](https://github.com/strimzi/proposals/blob/main/051-unidirectional-topic-operator.md)) watches `KafkaTopic` resources in a single namespace — typically the namespace where the `Kafka` resource is deployed. The UTO reconciles topic state unidirectionally from Kubernetes to Kafka, uses `creationTimestamp`-based conflict resolution when multiple `KafkaTopic` CRs reference the same Kafka topic, and employs finalizer-based deletion.

The UTO is deployed as a container within the Entity Operator pod, which the Cluster Operator manages. The `STRIMZI_NAMESPACE` environment variable controls which namespace the UTO watches.

## Motivation

In Kubernetes clusters, it is common practice to isolate applications in separate namespaces. When a Kafka cluster is shared among multiple applications or teams, each application typically lives in its own namespace. Currently, all `KafkaTopic` resources must be created in the Kafka cluster's namespace, which creates several problems:

- **Access control**: Teams that need to create topics must have RBAC permissions in the Kafka namespace, which may also grant access to other infrastructure resources they should not touch.
- **Resource management**: Kubernetes namespace-level quotas, and RBAC cannot be used to separate topic ownership per team.
- **GitOps workflows**: Application teams want to declare their Kafka topics alongside their application manifests in their own namespace, not in a shared infrastructure namespace.
- **Namespace hygiene**: Hundreds of `KafkaTopic` resources from different applications co-mingle in a single namespace, making management and auditing harder.

Allowing the Topic Operator to watch `KafkaTopic` resources across multiple namespaces solves these problems while maintaining the existing UTO semantics.

## Proposal

This proposal extends the UTO to watch `KafkaTopic` resources in multiple namespaces. It builds on the UTO's existing mechanisms for conflict resolution, deletion, and status reporting.

### Configuration

A new field `watchedNamespaces` is added to the Topic Operator configuration in the `Kafka` custom resource:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  entityOperator:
    topicOperator:
      watchedNamespaces:
        - team-a
        - team-b
        - team-c
```

The `watchedNamespaces` field accepts:
- A list of specific namespace names (e.g., `["team-a", "team-b"]`)
- A wildcard `["*"]` to watch all namespaces

The Topic Operator's own namespace (where the `Kafka` resource lives) is **always** watched implicitly. It does not need to be listed in `watchedNamespaces`. When `watchedNamespaces` is not set or is empty, the behavior is unchanged — the TO watches only its own namespace.

For the standalone Topic Operator deployment, the existing `STRIMZI_NAMESPACE` environment variable is extended to accept a comma-separated list of namespaces or `*`, consistent with how the Cluster Operator already supports multi-namespace watching.

#### Deprecation of `watchedNamespace` (singular)

The existing `spec.entityOperator.topicOperator.watchedNamespace` field is deprecated in favor of `watchedNamespaces` (plural).

**Deprecation timeline:** `watchedNamespace` continues to work in its current form but logs a deprecation warning on every Cluster Operator reconciliation. It will be removed in a future major version.

**Migration steps:** Convert `watchedNamespace: team-a` to `watchedNamespaces: [team-a]`. No other changes are required — no label changes, no RBAC changes, and no modifications to existing `KafkaTopic` resources.

**Interaction rules:** When both `watchedNamespace` and `watchedNamespaces` are set, `watchedNamespaces` takes precedence and a warning is logged indicating that the singular field is being ignored. When only the singular field is set, the TO behaves exactly as it does today — watching the Kafka cluster's own namespace plus the single additional namespace specified.

**Label impact:** Migrating from the singular to the plural field does **not** require any changes to `strimzi.io/cluster` labels on existing `KafkaTopic` resources. The TO accepts both the unqualified and qualified label forms regardless of which configuration field is used. However, the unqualified form is deprecated (see below) and should be migrated to the qualified `<namespace>/<name>` form.

#### Deprecation of unqualified `strimzi.io/cluster` label

The unqualified `strimzi.io/cluster` label (e.g., `my-cluster`) is deprecated in favor of the qualified `<namespace>/<name>` form (e.g., `kafka/my-cluster`).

**Deprecation timeline:** The TO accepts both forms, but logs a deprecation warning for each `KafkaTopic` that uses the unqualified form on every reconciliation. The unqualified form will be removed in a future major version.

**Migration steps:** Change the `strimzi.io/cluster` label value from `my-cluster` to `kafka/my-cluster` (i.e., `<kafka-namespace>/<kafka-cluster-name>`). No other changes to the `KafkaTopic` resource are required.

**Rationale:** The qualified form is unambiguous by design — it eliminates any risk of misrouting when multiple Kafka clusters share the same name across different namespaces, without requiring detection logic in the TO.

**Scope:** This deprecation applies only to `KafkaTopic` resources managed by the Topic Operator. Other resources that use the `strimzi.io/cluster` label (e.g., `KafkaUser` resources managed by the User Operator) are not affected and continue to use the unqualified form.

### RBAC

Watching additional namespaces requires the Topic Operator to have Kubernetes RBAC permissions in those namespaces. The Cluster Operator is responsible for creating and managing these RBAC resources.

**For explicit namespace list:**

The Cluster Operator creates a `Role` and `RoleBinding` in each watched namespace, granting the Entity Operator's `ServiceAccount` permissions to watch, list, get, and update `KafkaTopic` resources and their status.

The Role and RoleBinding names include the Kafka cluster's namespace and name to ensure uniqueness: `strimzi-topic-operator-<kafka-namespace>-<kafka-name>`. This prevents two Kafka clusters watching the same tenant namespace from overwriting each other's RBAC resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: strimzi-topic-operator-kafka-my-cluster
  namespace: team-a
  labels:
    strimzi.io/cluster: my-cluster
rules:
  - apiGroups: ["kafka.strimzi.io"]
    resources: ["kafkatopics", "kafkatopics/status"]
    verbs: ["get", "list", "watch", "create", "patch", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: strimzi-topic-operator-kafka-my-cluster
  namespace: team-a
  labels:
    strimzi.io/cluster: my-cluster
subjects:
  - kind: ServiceAccount
    name: my-cluster-entity-operator
    namespace: kafka
roleRef:
  kind: Role
  name: strimzi-topic-operator-kafka-my-cluster
  apiGroup: rbac.authorization.k8s.io
```

**For wildcard (`*`):**

The Cluster Operator creates a `ClusterRole` and `ClusterRoleBinding` instead:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: strimzi-topic-operator-my-cluster
  labels:
    strimzi.io/cluster: my-cluster
rules:
  - apiGroups: ["kafka.strimzi.io"]
    resources: ["kafkatopics", "kafkatopics/status"]
    verbs: ["get", "list", "watch", "create", "patch", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: strimzi-topic-operator-my-cluster
  labels:
    strimzi.io/cluster: my-cluster
subjects:
  - kind: ServiceAccount
    name: my-cluster-entity-operator
    namespace: kafka
roleRef:
  kind: ClusterRole
  name: strimzi-topic-operator-my-cluster
  apiGroup: rbac.authorization.k8s.io
```

When `watchedNamespaces` is updated (namespaces added or removed), the Cluster Operator reconciles the RBAC resources: creating new `Role`/`RoleBinding` pairs for added namespaces and deleting them for removed namespaces. RBAC resources created by the Cluster Operator are labeled with `strimzi.io/cluster` so they can be identified and cleaned up.

### Cross-namespace conflict resolution

The UTO already maintains an in-memory map of *resolved topic name* → (`namespace`, `name`) pairs for conflict detection within a single namespace. This proposal extends that map to span all watched namespaces.

The **resolved topic name** is `spec.topicName` if set, otherwise `metadata.name`. This is critical — two `KafkaTopic` CRs in different namespaces can reference the same Kafka topic through different mechanisms:

- Namespace A: `KafkaTopic` with `metadata.name: my-topic` (no `spec.topicName`, resolves to `my-topic`)
- Namespace B: `KafkaTopic` with `metadata.name: app-topic`, `spec.topicName: my-topic` (resolves to `my-topic`)

Both reference the same Kafka topic. The conflict resolution rules are:

1. The `KafkaTopic` with the **oldest** `metadata.creationTimestamp` is the owner, regardless of which namespace it is in.
2. All other `KafkaTopic` CRs referencing the same Kafka topic get:

```yaml
status:
  conditions:
  - type: Ready
    status: "False"
    reason: ResourceConflict
    message: "Managed by team-a/my-topic"
    lastTransitionTime: "2024-11-24T17:00:00Z"
```

This is identical to how the UTO handles conflicts within a single namespace today, extended across namespace boundaries.

### Deletion semantics

The UTO uses Kubernetes finalizers to ensure that deleting a `KafkaTopic` also deletes the corresponding Kafka topic. This behavior is unchanged in the multi-namespace case.

When the **owning** `KafkaTopic` (the oldest one) is deleted:
1. The finalizer logic deletes the Kafka topic from the Kafka cluster.
2. Any other `KafkaTopic` CRs in other namespaces that referenced the same Kafka topic will have their status updated to reflect that the topic no longer exists. They can then be cleaned up by their respective namespace owners, or if left in place, they will attempt to recreate the topic on their next reconciliation (becoming the new owner since no conflict exists).

When a **non-owning** `KafkaTopic` (one with `ResourceConflict` status) is deleted:
1. The finalizer is removed without deleting the Kafka topic (same as existing UTO behavior for conflicting resources).
2. No impact on the owning `KafkaTopic` or the Kafka topic itself.

The existing `strimzi.io/managed: false` annotation continues to work as before — it allows a `KafkaTopic` to be deleted without deleting the Kafka topic, enabling ownership transfers.

### Identifying ownership via `strimzi.io/cluster`

The existing `strimzi.io/cluster` label identifies which Kafka cluster a `KafkaTopic` belongs to. This proposal introduces a qualified `<namespace>/<name>` form and deprecates the existing unqualified form.

On startup, the TO computes its identity as `<namespace>/<name>` (e.g., `kafka/my-cluster`). **New deployments** should use the qualified form for the `strimzi.io/cluster` label:

```yaml
# Qualified form (recommended):
labels:
  strimzi.io/cluster: kafka/my-cluster

# Unqualified form (deprecated — logs a warning per resource on each reconciliation):
labels:
  strimzi.io/cluster: my-cluster
```

The TO accepts both forms — a `KafkaTopic` labeled `my-cluster` or `kafka/my-cluster` is matched by the TO for `kafka/my-cluster`. However, the unqualified form is **deprecated**: the TO logs a deprecation warning for each `KafkaTopic` that uses it, on every reconciliation. The unqualified form will be removed in a future major version.

The qualified form is unambiguous by design. Two TOs watching overlapping namespaces (e.g., `kafka/my-cluster` and `staging/my-cluster`) do not conflict because their qualified identities are distinct label values, and each TO ignores resources labeled for the other. No detection logic is needed.

If a `KafkaTopic` is in scope for a TO but lacks the expected `strimzi.io/cluster` label entirely (e.g., due to a misconfiguration), the TO skips it silently and logs a WARN. No status writes are performed on resources the TO does not own, which prevents reconciliation fights between competing instances.

### Namespace lifecycle considerations

**Adding a namespace to `watchedNamespaces`:**
The Cluster Operator creates RBAC resources in the new namespace and restarts/reconfigures the TO. The TO starts watching the new namespace and reconciles any existing `KafkaTopic` resources in it.

**Removing a namespace from `watchedNamespaces`:**
The Cluster Operator must follow this sequence to ensure finalizers can be cleaned up before RBAC is revoked:

1. While RBAC still exists, the CO removes the `strimzi.io/topic-operator` finalizer from all `KafkaTopic` resources in the namespace being removed.
2. The CO updates the TO configuration to remove the namespace from `watchedNamespaces`.
3. The CO deletes the RBAC resources (`Role`/`RoleBinding`) for that namespace.

This ordering is critical: deleting RBAC before removing finalizers would leave the TO unable to reach the `KafkaTopic` resources, causing finalizers to block any subsequent namespace deletion.

**Watched namespace is deleted:**
If the TO is running, it processes the finalizers normally — deleting Kafka topics as appropriate. If the TO is not running (or the namespace was already removed from the watch list), `KafkaTopic` resources with finalizers will block namespace deletion. In this case, manual finalizer removal is necessary (e.g., `kubectl patch kafkatopic <name> -n <ns> --type=json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'`). This is an existing limitation of the UTO that applies equally to the single-namespace case, and should be documented as operational guidance.

**TO restart:**
On startup, the TO scans all watched namespaces to rebuild its in-memory conflict resolution map before beginning reconciliation. This is the same process used today for a single namespace, extended to multiple namespaces. Startup time increases linearly with the number of watched namespaces and `KafkaTopic` resources.

### Example

A Kafka cluster in namespace `kafka`, with topics managed by teams in `team-a` and `team-b`:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  entityOperator:
    topicOperator:
      watchedNamespaces:
        - team-a
        - team-b
```

Team A creates a topic in their namespace using the qualified `strimzi.io/cluster` label, which is the required form going forward (the unqualified form `my-cluster` is deprecated):

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  namespace: team-a
  labels:
    strimzi.io/cluster: kafka/my-cluster
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: "604800000"
```

The TO reconciles this topic and creates it in Kafka:

```yaml
metadata:
  name: orders
  namespace: team-a
  labels:
    strimzi.io/cluster: kafka/my-cluster
  finalizers:
    - strimzi.io/topic-operator
status:
  topicName: orders
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-11-24T17:00:00Z"
```

If Team B creates a `KafkaTopic` also named `orders` in their namespace, it receives a conflict error:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  namespace: team-b
status:
  conditions:
  - type: Ready
    status: "False"
    reason: ResourceConflict
    message: "Managed by team-a/orders"
    lastTransitionTime: "2024-11-24T17:05:00Z"
```

## Affected/not affected projects

Affected:
- **Topic Operator**: Extended to watch multiple namespaces, cross-namespace conflict resolution
- **Cluster Operator**: Creates RBAC resources for watched namespaces, passes `watchedNamespaces` configuration to the Entity Operator

Not affected:
- User Operator, Kafka Connect operator, Kafka MirrorMaker operator
- KafkaTopic CRD schema (no changes to spec fields)

## Compatibility

- The `watchedNamespaces` field is optional with an empty default, preserving existing single-namespace behavior.
- No changes to the `KafkaTopic` CRD spec schema.
- `strimzi.io/cluster` label: the TO accepts both the unqualified form (e.g., `my-cluster`) and the qualified `<namespace>/<name>` form (e.g., `kafka/my-cluster`), but the unqualified form is **deprecated** and logs a warning on each reconciliation. No label changes are strictly required for existing deployments to keep working, but users should migrate to the qualified form. The unqualified form will be removed in a future major version.
- `watchedNamespace` (singular) is deprecated. When `watchedNamespaces` (plural) is set, the singular field is ignored with a deprecation warning logged. The migration path is: `watchedNamespace: team-a` → `watchedNamespaces: [team-a]` — no label, RBAC, or KafkaTopic changes needed.
- Existing `KafkaTopic` resources continue to work without modification.

## Future work

- Multi-namespace support for the User Operator (`KafkaUser` resources) — this will require a separate proposal addressing the security implications of cross-namespace ACL management (see [PR #137 discussion](https://github.com/strimzi/proposals/pull/137)).
- Namespace-level policies to restrict which topic configurations can be set from a given namespace (e.g., allowing infra teams to retain control over certain topic-level configs).

## Rejected alternatives

### New `KafkaNamespaceTopic` CRD

[Issue #1206](https://github.com/strimzi/strimzi-kafka-operator/issues/1206) proposed a `KafkaNamespaceTopic` CRD that would encode the Kubernetes namespace in the Kafka topic name (e.g., `team-a.orders`). This approach was rejected because:

1. **CRD maintenance cost**: Each custom resource is expensive to maintain. A new CRD requires its own controller logic, status management, validation webhooks, documentation, and migration tooling.
2. **Forces naming convention**: Encoding the namespace in the topic name forces a specific naming convention (`<namespace>.<topic>`) that may not match existing topic naming patterns or be desirable for all users.
3. **Dual CRD confusion**: Having both `KafkaTopic` and `KafkaNamespaceTopic` with different sync semantics (the original proposal suggested unidirectional sync for the new CRD) creates confusion about which to use when.
4. **Unnecessary complexity**: Extending the existing UTO to watch multiple namespaces achieves the same goal without introducing a new resource type, by building on the conflict resolution and lifecycle mechanisms the UTO already has.

