# Multi-namespace watching for the Topic Operator

Issue [#1206](https://github.com/strimzi/strimzi-kafka-operator/issues/1206)

## Current situation

The Topic Operator ([proposal 051](https://github.com/strimzi/proposals/blob/main/051-unidirectional-topic-operator.md)) watches `KafkaTopic` resources in a single configured namespace — by default the namespace where the `Kafka` resource is deployed, but this can be overridden via the `watchedNamespace` field or the `STRIMZI_NAMESPACE` environment variable. The Topic Operator reconciles topic state unidirectionally from Kubernetes to Kafka, uses `creationTimestamp`-based conflict resolution when multiple `KafkaTopic` CRs reference the same Kafka topic, and employs finalizer-based deletion.

The Topic Operator is deployed as a container within the Entity Operator pod, which the Cluster Operator manages. The `STRIMZI_NAMESPACE` environment variable controls which namespace the Topic Operator watches, or it can be run as a standalone deployment.

## Motivation

In Kubernetes clusters, it is common practice to isolate applications in separate namespaces. When a Kafka cluster is shared among multiple applications or teams, each application typically lives in its own namespace. Currently, the Topic Operator watches a single namespace, either the Kafka cluster's own namespace (default) or one alternative namespace configured via watchedNamespace field or STRIMZI_NAMESPACE environment variable. which can creates several problems:

- **Access control**: Teams that need to create topics must have RBAC permissions in the Kafka namespace, which may also grant access to other infrastructure resources they should not touch.
- **Resource management**: Kubernetes namespace-level quotas, and RBAC cannot be used to separate topic ownership per team.
- **GitOps workflows**: Application teams want to declare their Kafka topics alongside their application manifests in their own namespace, not in a shared infrastructure namespace.
- **Namespace hygiene**: Hundreds of `KafkaTopic` resources from different applications co-mingle in a single namespace, making management and auditing harder.

Allowing the Topic Operator to watch `KafkaTopic` resources across multiple namespaces solves these problems while maintaining the existing Topic Operator semantics.

## Proposal

This proposal extends the Topic Operator to watch `KafkaTopic` resources in multiple namespaces. It builds on the existing mechanisms for conflict resolution, deletion, and status reporting.

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

For the standalone Topic Operator deployment, the existing `STRIMZI_NAMESPACE` environment variable is extended to accept a comma-separated list of namespaces or `*`, consistent with how the Cluster Operator already supports multi-namespace watching. In standalone mode, the service account is named `strimzi-topic-operator` (rather than `<cluster>-entity-operator` used in the Kafka-managed deployment).

#### Deployment modes

The Topic Operator supports two deployment modes, both of which support multi-namespace watching:

- **Kafka-managed (default):** The Topic Operator runs as a container inside the Entity Operator pod, managed by the Cluster Operator. Multi-namespace watching is configured via `spec.entityOperator.topicOperator.watchedNamespaces` in the `Kafka` CR. The Cluster Operator is responsible for creating and maintaining the necessary RBAC resources.

- **Standalone:** The Topic Operator runs independently, outside of a `Kafka`-managed Entity Operator. Multi-namespace watching is configured via the `STRIMZI_NAMESPACE` environment variable (comma-separated list or `*`). The operator administrator is responsible for creating the necessary RBAC resources in each watched namespace.

This mirrors the deployment mode documentation for the Cluster Operator.

#### Deprecation of `watchedNamespace` (singular)

The existing `spec.entityOperator.topicOperator.watchedNamespace` field is deprecated in favor of `watchedNamespaces` (plural).

**Deprecation timeline:** `watchedNamespace` continues to work in its current form but logs a deprecation warning on every Cluster Operator reconciliation. It will be removed in a future major version.

**Migration steps:** Convert `watchedNamespace: team-a` to `watchedNamespaces: [team-a]`. Because switching to `watchedNamespaces` enables multi-namespace mode, the `STRIMZI_RESOURCE_LABELS` selector will now require both `strimzi.io/cluster` and `strimzi.io/cluster-namespace` labels. Before making this change, add the `strimzi.io/cluster-namespace` label to all existing `KafkaTopic` resources (see the migration example below). No RBAC changes are required — the Cluster Operator handles RBAC automatically.

**Interaction rules:** Setting both `watchedNamespace` and `watchedNamespaces` simultaneously is rejected at admission time via a CEL validation rule on the `Kafka` CRD. Users must migrate to `watchedNamespaces` before adding additional namespaces. When only the singular field is set, the TO behaves exactly the same as before.

**Label impact:** Migrating from the singular to the plural field does **not** require any changes to existing `strimzi.io/cluster` labels on `KafkaTopic` resources. The `strimzi.io/cluster` label value remains unchanged (e.g., `my-cluster`). When `watchedNamespaces` is configured, a new `strimzi.io/cluster-namespace` label is introduced to disambiguate cluster identity (see below).

#### New `strimzi.io/cluster-namespace` label

When `watchedNamespaces` is configured, a new label `strimzi.io/cluster-namespace` is introduced to disambiguate which Kafka cluster instance a `KafkaTopic` belongs to. The existing `strimzi.io/cluster` label is **unchanged** — it continues to hold the cluster name (e.g., `my-cluster`).

```yaml
labels:
  strimzi.io/cluster: my-cluster
  strimzi.io/cluster-namespace: kafka      # new — namespace of the Kafka CR
```

The value of `strimzi.io/cluster-namespace` is the `metadata.namespace` of the `Kafka` CR that owns the Topic Operator. For example, a `Kafka` resource named `my-cluster` in namespace `kafka` produces:
- `strimzi.io/cluster: my-cluster`
- `strimzi.io/cluster-namespace: kafka`

**When the label is required:** The `strimzi.io/cluster-namespace` label is only needed when `watchedNamespaces` is configured (multi-namespace mode). Single-namespace deployments that do not set `watchedNamespaces` continue to work with `strimzi.io/cluster` alone.

**Kafka-managed deployment:** When the Cluster Operator deploys the Topic Operator with `watchedNamespaces` configured, it automatically sets `STRIMZI_RESOURCE_LABELS` to `strimzi.io/cluster=<cluster-name>,strimzi.io/cluster-namespace=<kafka-namespace>`. The TO then requires both labels to match before reconciling a `KafkaTopic`. This ensures that two TOs watching the same namespace for different clusters with the same name (e.g., `my-cluster` in namespace `kafka` and `my-cluster` in namespace `staging`) never clash — their `strimzi.io/cluster-namespace` values differ.

**Standalone deployment:** The operator administrator sets both labels in `STRIMZI_RESOURCE_LABELS`:

```yaml
- name: STRIMZI_RESOURCE_LABELS
  value: "strimzi.io/cluster=my-cluster,strimzi.io/cluster-namespace=kafka"
```

**Behaviour without the label:** In multi-namespace mode, a `KafkaTopic` that has `strimzi.io/cluster` but lacks `strimzi.io/cluster-namespace` will **not** be reconciled by the TO — it will not match the label selector. The TO logs a WARN for such resources to help users identify missing labels. This prevents accidental misrouting when multiple TOs watch overlapping namespaces.

**Backward compatibility:** In single-namespace mode (no `watchedNamespaces`), the TO's `STRIMZI_RESOURCE_LABELS` continues to be set to `strimzi.io/cluster=<cluster-name>` only, exactly as it is today. Existing `KafkaTopic` resources work without modification.

**Scope:** This new label applies only to `KafkaTopic` resources managed by the Topic Operator in multi-namespace mode. Other resources that use `strimzi.io/cluster` (e.g., `KafkaUser`, `KafkaConnector`, `KafkaRebalance`) are not affected.

### RBAC

Watching additional namespaces requires the Topic Operator to have Kubernetes RBAC permissions in those namespaces. In the Kafka-managed deployment, the Cluster Operator is responsible for creating and managing these RBAC resources. In standalone mode, the operator administrator must create them manually; the standalone installation files provided by Strimzi will be updated to include the additional Role/RoleBinding templates needed for multi-namespace watching.

**For explicit namespace list:**

The Cluster Operator creates a `Role` and `RoleBinding` in each watched namespace, granting the Entity Operator's `ServiceAccount` permissions to watch, list, get, and update `KafkaTopic` resources and their status, for standalone deployments, the user needs to create the `Role` and `RoleBinding` themselves in each namespace

The Role and RoleBinding names include the Kafka cluster's namespace and name to ensure uniqueness: `strimzi-topic-operator-<kafka-namespace>-<kafka-name>`. This prevents two Kafka clusters watching the same tenant namespace from overwriting each other's RBAC resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: strimzi-topic-operator-kafka-my-cluster
  namespace: team-a
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
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
    strimzi.io/cluster-namespace: kafka
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
  name: strimzi-topic-operator-kafka-my-cluster
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
rules:
  - apiGroups: ["kafka.strimzi.io"]
    resources: ["kafkatopics"]
    verbs: ["get", "list", "watch", "create", "patch", "update", "delete"]
  - apiGroups: ["kafka.strimzi.io"]
    resources: [""kafkatopics/status"]
    verbs: ["get", "patch", "update"]
  - apiGroups: ["kafka.strimzi.io"]
    resources: ["kafkatopics/finalizers"]
    verbs: ["delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: strimzi-topic-operator-kafka-my-cluster
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
subjects:
  - kind: ServiceAccount
    name: my-cluster-entity-operator
    namespace: kafka
roleRef:
  kind: ClusterRole
  name: strimzi-topic-operator
  apiGroup: rbac.authorization.k8s.io
```

When `watchedNamespaces` is updated (namespaces added or removed), the Cluster Operator reconciles the RBAC resources: creating new `Role`/`RoleBinding` pairs for added namespaces and deleting them for removed namespaces. RBAC resources created by the Cluster Operator are labeled with `strimzi.io/cluster` and `strimzi.io/cluster-namespace` so they can be identified and cleaned up.

### Cross-namespace conflict resolution

The Topic Operator already maintains an in-memory map of *resolved topic name* → (`namespace`, `name`) pairs for conflict detection within a single namespace. This proposal extends that map to span all watched namespaces.

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
    message: "Already managed by another topic operator"
    lastTransitionTime: "2024-11-24T17:00:00Z"
```

This is identical to how the Topic Operator handles conflicts within a single namespace today, extended across namespace boundaries.

**Topic name isolation:** Conflict detection is the mechanism for preventing two teams from accidentally managing the same Kafka topic. It is intentionally reactive rather than preventive — teams *can* share a topic across namespaces (one namespace owns it, others receive a `ResourceConflict` status). For use cases that require strict naming isolation between teams, the recommended approach is to run separate Topic Operator instances per team rather than a shared one.

**Event starvation:** A single Topic Operator instance reconciling events across many namespaces may be starved if one namespace generates a very high volume of events, delaying reconciliation for other namespaces, to address this we would also implement er-namespace reconciliation queues

### Deletion semantics

The Topic Operator uses Kubernetes finalizers to ensure that deleting a `KafkaTopic` also deletes the corresponding Kafka topic. The `strimzi.io/topic-operator` finalizer is added and managed exclusively by the Topic Operator.

When the **owning** `KafkaTopic` (the oldest one) is deleted:
1. The finalizer logic deletes the Kafka topic from the Kafka cluster.
2. Any other `KafkaTopic` CRs in other namespaces that referenced the same Kafka topic will be recreated by the topic operator on their next reconciliation (becoming the new owner since no conflict exists).

When a **non-owning** `KafkaTopic` (one with `ResourceConflict` status) is deleted:
1. The finalizer is removed without deleting the Kafka topic (same as existing Topic Operator behavior for conflicting resources).
2. No impact on the owning `KafkaTopic` or the Kafka topic itself.

The existing `strimzi.io/managed: false` annotation continues to work as before — it allows a `KafkaTopic` to be deleted without deleting the Kafka topic, enabling ownership transfers.

### Identifying ownership via labels

The existing `strimzi.io/cluster` label identifies which Kafka cluster a `KafkaTopic` belongs to by name. In multi-namespace mode, the new `strimzi.io/cluster-namespace` label adds the namespace dimension to form a unique cluster identity.

On startup in multi-namespace mode, the TO builds its label selector from both labels. For a `Kafka` CR named `my-cluster` in namespace `kafka`, the selector is:

```
strimzi.io/cluster=my-cluster,strimzi.io/cluster-namespace=kafka
```

A `KafkaTopic` must carry both labels to be reconciled:

```yaml
# Multi-namespace mode (watchedNamespaces configured):
labels:
  strimzi.io/cluster: my-cluster
  strimzi.io/cluster-namespace: kafka

# Single-namespace mode (no watchedNamespaces — unchanged from today):
labels:
  strimzi.io/cluster: my-cluster
```

Two TOs watching overlapping namespaces for clusters with the same name (e.g., `my-cluster` in `kafka` and `my-cluster` in `staging`) do not conflict — their `strimzi.io/cluster-namespace` values differ, so each TO's label selector matches only its own resources. No detection logic is needed.

If a `KafkaTopic` is in scope for a TO but does not match its label selector (e.g., missing the `strimzi.io/cluster-namespace` label in multi-namespace mode), the TO skips it and logs a WARN. No status writes are performed on resources the TO does not own, which prevents reconciliation fights between competing instances.

#### Overlapping namespace scenarios

The two-label approach makes the behaviour explicit when multiple Topic Operators watch overlapping namespaces:

- **Two TOs watching the same namespace for the same cluster** (e.g., both select `strimzi.io/cluster=my-cluster` + `strimzi.io/cluster-namespace=kafka`): This is a misconfiguration. Each TO will attempt to reconcile the same `KafkaTopic` resources, causing reconciliation conflicts. Users must ensure each Kafka cluster has at most one active Topic Operator instance per watched namespace.

- **Two TOs watching the same namespace for different clusters** (e.g., one selects `strimzi.io/cluster-namespace=kafka`, the other `strimzi.io/cluster-namespace=staging`): This is valid and supported. Each TO only reconciles `KafkaTopic` resources matching its own label selector, so the two instances operate independently without interfering.

- **A namespace watched by a TO that no longer exists or has been removed from the watch list**: Covered in the namespace lifecycle section below. `KafkaTopic` resources with finalizers will block namespace deletion if the TO cannot process them; manual finalizer removal is required as a last resort.

### Namespace lifecycle considerations

**Adding a namespace to `watchedNamespaces`:**
The Cluster Operator creates RBAC resources in the new namespace and restarts/reconfigures the TO. The TO starts watching the new namespace and reconciles any existing `KafkaTopic` resources in it.

**Removing a namespace from `watchedNamespaces`:**
The Cluster Operator must follow this sequence to ensure finalizers can be cleaned up before RBAC is revoked:

1. While RBAC still exists, the Topic operator removes the `strimzi.io/topic-operator` finalizer from all `KafkaTopic` resources in the namespace being removed.
2. The CO updates the TO configuration to remove the namespace from `watchedNamespaces`.
3. The CO deletes the RBAC resources (`Role`/`RoleBinding`) for that namespace.

**Topic Operator restart:**
On startup, the Topic Operator scans all watched namespaces to rebuild its in-memory conflict resolution map before beginning reconciliation. This is the same process used today for a single namespace, extended to multiple namespaces. Startup time increases linearly with the number of watched namespaces and `KafkaTopic` resources.

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

Team A creates a topic in their namespace. Because the cluster uses `watchedNamespaces`, both `strimzi.io/cluster` and `strimzi.io/cluster-namespace` labels are required:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  namespace: team-a
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
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
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
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
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
status:
  conditions:
  - type: Ready
    status: "False"
    reason: ResourceConflict
    message: "Managed by team-a/orders"
    lastTransitionTime: "2024-11-24T17:05:00Z"
```

### Migration example: single-namespace to multi-namespace

This section walks through migrating an existing single-namespace deployment to multi-namespace, covering both the Kafka-managed and standalone modes.

#### Starting state

A Kafka cluster `my-cluster` in namespace `kafka`, with all topics in the `kafka` namespace:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  entityOperator:
    topicOperator: {}
```

Existing topics use only the `strimzi.io/cluster` label:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 12
  replicas: 3
```

#### Step 1: Add `strimzi.io/cluster-namespace` label to existing topics

Before enabling multi-namespace watching, add the new label to all existing `KafkaTopic` resources. This is safe to do while still in single-namespace mode — the TO ignores the extra label since `STRIMZI_RESOURCE_LABELS` only matches on `strimzi.io/cluster`:

```bash
kubectl label kafkatopic --all -n kafka strimzi.io/cluster-namespace=kafka
```

Verify that topics continue to reconcile normally after relabeling.

#### Step 2: Enable multi-namespace watching

**Kafka-managed mode:**

Update the `Kafka` CR to add `watchedNamespaces`:

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

The Cluster Operator will:
1. Create `Role`/`RoleBinding` in `team-a` and `team-b`
2. Update `STRIMZI_RESOURCE_LABELS` to `strimzi.io/cluster=my-cluster,strimzi.io/cluster-namespace=kafka`
3. Restart the Topic Operator

Existing topics in the `kafka` namespace continue to work because they were relabeled in Step 1.

**Standalone mode:**

Update the Topic Operator deployment:

```yaml
env:
  - name: STRIMZI_NAMESPACE
    value: "kafka,team-a,team-b"
  - name: STRIMZI_RESOURCE_LABELS
    value: "strimzi.io/cluster=my-cluster,strimzi.io/cluster-namespace=kafka"
```

Create the necessary `Role`/`RoleBinding` resources in each new namespace manually.

#### Step 3: Create topics in tenant namespaces

Teams can now create `KafkaTopic` resources in their own namespaces with both labels:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: events
  namespace: team-a
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
spec:
  partitions: 6
  replicas: 3
```

### Security and multi-tenancy considerations

A shared Topic Operator instance watching multiple tenant namespaces has inherent multi-tenancy implications that operators should understand:

- **Information leakage:** When two teams use the same topic name, the "losing" team's `KafkaTopic` will surface the "winner's" `namespace/name` in its `ResourceConflict` status message. This means teams can infer that another namespace is using a given topic name. For environments where this is unacceptable, the recommended mitigation is to run separate Topic Operator instances per team rather than a shared one.

- **Blast radius:** A misconfiguration or runaway workload in one namespace can affect reconciliation for all other namespaces served by the same Topic Operator instance. Platform operators should weigh this when deciding between a shared instance and per-team instances.

## Affected/not affected projects

Affected:
- **Topic Operator**: Extended to watch multiple namespaces, cross-namespace conflict resolution
- **Cluster Operator**: Creates RBAC resources for watched namespaces, passes `watchedNamespaces` configuration to the Entity Operator

Not affected:
- `strimzi.io/cluster` label value and semantics are unchanged across all resource types
- KafkaTopic CRD schema (no changes to spec fields)

## Compatibility

- The `watchedNamespaces` field is optional with an empty default, preserving existing single-namespace behavior (the Kafka cluster namespace when deployed as part of the CO, or the TO namespace when deployed as standalone)
- No changes to the `KafkaTopic` CRD spec schema.
- `strimzi.io/cluster` label: unchanged. The label value remains the cluster name (e.g., `my-cluster`). No deprecation, no migration required for existing deployments.
- `strimzi.io/cluster-namespace` label: new, only required when `watchedNamespaces` is configured. Single-namespace deployments are not affected. In Kafka-managed mode, the Cluster Operator handles the `STRIMZI_RESOURCE_LABELS` configuration automatically.
- `watchedNamespace` (singular) is deprecated. Setting both `watchedNamespace` and `watchedNamespaces` simultaneously is rejected at admission time via CEL validation on the `Kafka` CRD. The migration path is: `watchedNamespace: team-a` → `watchedNamespaces: [team-a]` — no label, RBAC, or KafkaTopic changes needed.
- Existing `KafkaTopic` resources continue to work without modification in single-namespace mode. When enabling multi-namespace mode (`watchedNamespaces`), existing topics must be labeled with `strimzi.io/cluster-namespace` before the switch (see the migration example in the proposal).

## Future work

- Multi-namespace support for the User Operator (`KafkaUser` resources) — this will require a separate proposal addressing the security implications of cross-namespace ACL management (see [PR #137 discussion](https://github.com/strimzi/proposals/pull/137)).
- Namespace-level policies to restrict which topic configurations can be set from a given namespace (e.g., allowing infra teams to retain control over certain topic-level configs, mandatory topic name prefixes, maximum partition counts).

## Rejected alternatives

### New `KafkaNamespaceTopic` CRD

[Issue #1206](https://github.com/strimzi/strimzi-kafka-operator/issues/1206) proposed a `KafkaNamespaceTopic` CRD that would encode the Kubernetes namespace in the Kafka topic name (e.g., `team-a.orders`). This approach was rejected because:

1. **CRD maintenance cost**: Each custom resource is expensive to maintain. A new CRD requires its own controller logic, status management, validation webhooks, documentation, and migration tooling.
2. **Forces naming convention**: Encoding the namespace in the topic name forces a specific naming convention (`<namespace>.<topic>`) that may not match existing topic naming patterns or be desirable for all users.
3. **Dual CRD confusion**: Having both `KafkaTopic` and `KafkaNamespaceTopic` with different sync semantics (the original proposal suggested unidirectional sync for the new CRD) creates confusion about which to use when.
4. **Unnecessary complexity**: Extending the existing Topic Operator to watch multiple namespaces achieves the same goal without introducing a new resource type, by building on the conflict resolution and lifecycle mechanisms it already has.

### Qualified `strimzi.io/cluster` label (`<namespace>/<name>` form)

An alternative considered was changing the `strimzi.io/cluster` label value to include the Kafka cluster's namespace (e.g., `strimzi.io/cluster: kafka/my-cluster` instead of `strimzi.io/cluster: my-cluster`), deprecating the existing unqualified form.

This approach was rejected because:

1. **Breaking change to a widely-used label:** `strimzi.io/cluster` is used by KafkaTopic, KafkaUser, KafkaConnector, KafkaRebalance, KafkaNodePool, Services, and NetworkPolicies. Changing the value semantics — even with deprecation — has a large blast radius across the Strimzi ecosystem.
2. **Migration burden:** Every existing `KafkaTopic` would eventually need to be relabeled from `my-cluster` to `kafka/my-cluster`. At scale (thousands of topics), this is significant operational work.
3. **`/` in label values is unusual:** While technically valid in Kubernetes label values, it is uncommon and may confuse tooling (e.g., Prometheus relabeling, Helm templates, GitOps label selectors).
4. **Scope creep:** If adopted for KafkaTopic, users and maintainers would expect the same pattern for KafkaUser, KafkaConnector, etc., creating a project-wide migration.
5. **Deprecation log spam:** Every existing unqualified topic would log a warning on every reconciliation cycle until migrated.

The separate `strimzi.io/cluster-namespace` label achieves the same disambiguation without changing the existing `strimzi.io/cluster` label, and is only required when multi-namespace watching is configured.
