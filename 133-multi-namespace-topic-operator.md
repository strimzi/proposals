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

When `watchedNamespaces` is **not set or empty**, the behavior is unchanged — the TO watches only its own namespace (where the `Kafka` resource lives or the value of watchedNamespace), preserving backwards compatibility. When `watchedNamespaces` **is** set, it follows the standard Kubernetes convention that explicit user input *replaces* the default rather than extending it: the list means exactly those namespaces. The TO's own namespace is watched only if it is included in the list.

Multi-namespace watching is **gated**: `watchedNamespaces` is only accepted when the `STRIMZI_ENTITY_OPERATOR_WATCHED_NAMESPACE_ENABLED` environment variable is enabled (default: disabled). This is a security control discussed under [Security and multi-tenancy considerations](#security-and-multi-tenancy-considerations). it must be enabled by the cluster administrator before any cross-namespace watching is allowed.

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

The Cluster Operator creates a `Role` and `RoleBinding` in each watched namespace, granting the Entity Operator's `ServiceAccount` permissions on `KafkaTopic` resources. For standalone deployments, the user needs to create the `Role` and `RoleBinding` themselves in each namespace.

Following the principle of least privilege, the verbs are scoped per sub-resource: full lifecycle verbs on `kafkatopics`, status-only verbs on `kafkatopics/status`, and `update` on `kafkatopics/finalizers`.

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
    resources: ["kafkatopics"]
    verbs: ["get", "list", "watch", "create", "patch", "update", "delete"]
  - apiGroups: ["kafka.strimzi.io"]
    resources: ["kafkatopics/status"]
    verbs: ["get", "patch", "update"]
  - apiGroups: ["kafka.strimzi.io"]
    resources: ["kafkatopics/finalizers"]
    verbs: ["update"]
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
  # Kafka-managed deployment: the Entity Operator ServiceAccount.
  # For a standalone deployment, use `name: strimzi-topic-operator` instead.
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
    resources: ["kafkatopics/status"]
    verbs: ["get", "patch", "update"]
  - apiGroups: ["kafka.strimzi.io"]
    resources: ["kafkatopics/finalizers"]
    verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: strimzi-topic-operator-kafka-my-cluster
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/cluster-namespace: kafka
subjects:
  # Kafka-managed deployment: the Entity Operator ServiceAccount.
  # For a standalone deployment, use `name: strimzi-topic-operator` instead.
  - kind: ServiceAccount
    name: my-cluster-entity-operator
    namespace: kafka
roleRef:
  kind: ClusterRole
  name: strimzi-topic-operator-kafka-my-cluster
  apiGroup: rbac.authorization.k8s.io
```

When `watchedNamespaces` is updated (namespaces added or removed), the Cluster Operator reconciles the RBAC resources: creating new `Role`/`RoleBinding` pairs for added namespaces and deleting them for removed namespaces. RBAC resources created by the Cluster Operator are labeled with `strimzi.io/cluster` and `strimzi.io/cluster-namespace` so they can be identified and cleaned up.

### Topic name resolution

Topic name isolation between namespaces is a **prerequisite** of this proposal. If the Topic Operator watches multiple namespaces, users will expect that a topic declared by one team cannot silently collide with a topic of the same name declared by another team. To guarantee that by default, multi-namespace mode introduces namespace-prefixed topic naming.

**Built-in default (namespace prefix):** When `watchedNamespaces` is configured, the resolved Kafka topic name defaults to:

```
<namespace>.<metadata.name>
```

For example, a `KafkaTopic` named `orders` in namespace `team-a` is materialized in Kafka as `team-a.orders`, while an `orders` in `team-b` becomes `team-b.orders`. The two never collide, so each namespace gets an isolated topic namespace without any per-team configuration.

**Single-namespace mode is unchanged:** When `watchedNamespaces` is not set (single-namespace mode), no prefix is applied — `metadata.name` (or `spec.topicName`) resolves exactly as it does today. This preserves all existing topic names and requires no migration for current deployments.

**Opt-out via `spec.topicName`:** Setting `spec.topicName` explicitly overrides the prefix and uses the given name verbatim, with no namespace prefixing. This is the deliberate escape hatch for teams that need to address a topic by an exact name — for example to share a single Kafka topic across namespaces, or to adopt a topic created under a different convention. It is the only path that can produce a cross-namespace name collision, which is handled by conflict resolution below.

> This proposal defines only the minimal default (namespace prefix) needed to make multi-namespace watching safe. A richer, configurable policy mechanism — per-namespace allowed configs, mandatory prefixes, maximum partition counts, etc. — is tracked separately as a `KafkaTopicPolicy`-style CR (see [Future work](#future-work)). The namespace prefix described here is the built-in behavior that ships with this feature regardless of that follow-up.

### Cross-namespace conflict resolution

With the namespace prefix as the default, two `KafkaTopic` CRs in different namespaces can no longer resolve to the same Kafka topic. Conflict resolution therefore applies only to the `spec.topicName` opt-out case described above, where two CRs deliberately set the same explicit `spec.topicName`.

The Topic Operator already maintains an in-memory map for conflict detection within a single namespace — `BatchingTopicController.topicRefs` (`Map<String, List<KubeRef>>`, keyed by *resolved topic name* → the `KafkaTopic`s claiming it; a list with more than one entry is a conflict). The **resolved topic name** is `spec.topicName` if set, otherwise `<namespace>.<metadata.name>` in multi-namespace mode (or `metadata.name` in single-namespace mode).

Whether this map can detect conflicts *across* namespaces depends on the reconciliation architecture chosen below: with fully independent per-namespace controllers (the recommended Option C), each controller's `topicRefs` only covers its own namespace, so there is **no** in-memory cross-namespace detection. This is an acceptable consequence of the prefix default — cross-namespace collisions are only reachable on the `spec.topicName` opt-out path, and on that path the collision surfaces reactively when both controllers attempt to manage the same Kafka topic (the second create fails / one side observes the topic already exists and is marked `ResourceConflict`). Teams that deliberately opt out into a shared explicit name are coordinating on that name already.

When two CRs resolve to the same Kafka topic (only possible via matching `spec.topicName`), the rules are:

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

With default (prefixed) naming this branch is effectively unreachable; it exists to keep the opt-out path safe. How it is detected (in-memory vs. reactively via Kafka) follows from the architecture choice in the next section.

### Reconciliation architecture and noisy-neighbor isolation

A single Topic Operator reconciling events across many namespaces must not let a high-volume namespace starve reconciliation for the others. The design mirrors the structural isolation the Cluster Operator already provides for multi-namespace watching, while keeping the lightweight, batched reconciliation model the TO uses today.

#### How the TO works today (single namespace)

The current pipeline is **one of everything**, wired in `TopicOperator`:

- a single `SharedIndexInformer` scoped to one namespace (`Crds.topicOperation(client).inNamespace(config.watchedNamespace())`);
- a `TopicEventHandler` that turns informer callbacks into events and `offer()`s them onto;
- a single `BatchingLoop` — a `BlockingDeque<TopicEvent>` plus a worker thread (`LoopRunnable`) that collects a batch (trading a small configurable *linger* for throughput), de-duplicates same-topic events via an `inFlight` set, and hands the batch to;
- a single `BatchingTopicController`, which holds the reconciliation state (`topicRefs`, `topicIds`).

Two properties of the current code are load-bearing for the multi-namespace design:

1. **It is deliberately single-threaded.** `BatchingLoop` is constructed with `maxThreads = 1` (`new BatchingLoop(config, controller, 1, ...)`), and the controller's state maps are plain `HashMap`s. The class contract is *"any given `KafkaTopic` is only being reconciled by a single thread at any one time."* The controller is **not** safe to call concurrently as written.
2. **The watched namespace is a single value** baked into the informer, the `BatchingLoop`, and the controller (`config.watchedNamespace()`, with a `requireNonNull` at startup).

A naive multi-namespace change — point one informer at all namespaces but keep the single shared queue and worker — would funnel every namespace's events into one deque drained by one thread. A flood in one namespace would sit ahead of every other namespace's events: the noisy-neighbor problem. The design below avoids that.

#### Multi-namespace — per-namespace isolation

Reconciliation is **partitioned per namespace** so the unit of concurrency is one worker *per namespace*, not one thread per `KafkaTopic` event. Two watch topologies, matching the two RBAC modes:

- **Namespace list (namespace-scoped RBAC):** one informer + one `BatchingLoop` (queue + worker) **per watched namespace** — effectively running today's pipeline once per namespace. Each namespace reconciles independently; a backlog in one cannot delay another. Cost: N namespaces = N watch connections + N threads.
- **Wildcard `*` (cluster-scoped RBAC):** a single cluster-wide informer (`inAnyNamespace()`, one watch, with a shared item store/cache) whose event handler **routes** each event to a per-namespace `BatchingLoop` by the topic's `metadata.namespace`. This avoids one watch per namespace at very high namespace counts while still isolating throughput per namespace. Per-namespace loops are created (and torn down) as namespaces gain/lose `KafkaTopic`s, so loop lifecycle must be managed explicitly.

In both topologies thread count is proportional to the number of watched namespaces (1 worker per namespace), far lighter than per-event threading.

#### The controller-state decision

Partitioning the *queues* is necessary but not sufficient, because the `BatchingTopicController` is stateful and single-threaded (above). There are three ways to combine per-namespace queues with the controller; the proposal must pick one:

- **A — per-namespace controllers + a shared conflict map.** Extract `topicRefs` into a synchronized component shared by all controllers. Preserves in-memory cross-namespace conflict detection, but reintroduces a shared synchronization point and is the most invasive.
- **B — one shared controller made thread-safe.** Concurrent state maps + locking around reconciliation. But a coarse lock means a large batch in one namespace blocks others *inside the controller*, partially re-creating the starvation the per-namespace queues were meant to remove. Significant rework of a class built to be single-threaded.
- **C — fully independent per-namespace controllers, no shared state (recommended).** Each namespace gets its own `BatchingLoop` *and* its own `BatchingTopicController` with its own `topicRefs`/`topicIds`. No shared mutable state, true isolation, and the smallest change to the controller (it keeps assuming a single driving thread — there just are several controllers, one per namespace). The only thing given up is *in-memory* cross-namespace conflict detection, which the namespace-prefix default already makes unnecessary: collisions are only possible on the `spec.topicName` opt-out path and are caught reactively against Kafka (see [Cross-namespace conflict resolution](#cross-namespace-conflict-resolution)).

**This proposal adopts Option C.** It is the natural fit for the prefix default: because prefixing removes cross-namespace collisions by construction, the controllers do not need to share state, which is what makes the per-namespace isolation clean rather than a concurrency rewrite.

There is a practical ceiling on how many namespaces one TO instance can serve (threads, and in the namespace-list topology, watch connections). That is acceptable: in standalone mode the TO scales horizontally by deploying multiple independent instances, each owning a disjoint set of namespaces. The `systemtest` performance tests should be used to compare this architecture against the current single-namespace TO and to characterize that ceiling.

### Deletion semantics

The Topic Operator uses Kubernetes finalizers to ensure that deleting a `KafkaTopic` also deletes the corresponding Kafka topic. The `strimzi.io/topic-operator` finalizer is added and managed exclusively by the Topic Operator.

Under default (prefixed) naming every `KafkaTopic` resolves to a unique Kafka topic, so each is its own owner and deletion behaves exactly as in single-namespace mode. The owning/non-owning distinction below applies only when multiple resources share an explicit `spec.topicName` (the opt-out path).

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

The TO reconciles this topic and creates it in Kafka. Because the cluster is in multi-namespace mode, the resolved Kafka topic name is namespace-prefixed (`team-a.orders`):

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
  topicName: team-a.orders
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-11-24T17:00:00Z"
```

If Team B creates a `KafkaTopic` also named `orders` in their namespace, there is **no conflict** — it resolves to a different Kafka topic, `team-b.orders`, and reconciles successfully:

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
  topicName: team-b.orders
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-11-24T17:05:00Z"
```

A conflict only arises if both teams deliberately opt out of prefixing by setting the same explicit `spec.topicName` (e.g. both set `spec.topicName: orders`). In that case the oldest CR wins and the other gets a non-leaking `ResourceConflict`:

```yaml
status:
  conditions:
  - type: Ready
    status: "False"
    reason: ResourceConflict
    message: "Already managed by another topic operator"
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

> **⚠️ Migration is not transparent — existing topic names change unless pinned.** Multi-namespace mode prefixes the resolved Kafka topic name with the namespace for **every** watched namespace, including the Kafka CR's own namespace. A topic `orders` in namespace `kafka` resolves to `kafka.orders` once multi-namespace mode is enabled. Because Kafka topics cannot be renamed, the operator would create a new empty `kafka.orders` topic and stop managing the original `orders` topic (orphaning its data). **Before enabling multi-namespace mode, you must pin every existing topic's current name via `spec.topicName` (Step 1).**

#### Step 1: Pin existing topic names and add the `strimzi.io/cluster-namespace` label

While still in single-namespace mode, do two things to every existing `KafkaTopic`:

1. **Pin the current Kafka topic name** by setting `spec.topicName` to the name the topic resolves to today (its `metadata.name`, or its existing `spec.topicName`). In single-namespace mode this is a no-op, but it ensures the name does **not** change when prefixing is later turned on. For the `orders` example:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  topicName: orders   # pin the current name so prefixing does not rename it
  partitions: 12
  replicas: 3
```

2. **Add the new label** to all existing `KafkaTopic` resources. This is safe in single-namespace mode — the TO ignores the extra label since `STRIMZI_RESOURCE_LABELS` only matches on `strimzi.io/cluster`:

```bash
kubectl label kafkatopic --all -n kafka strimzi.io/cluster-namespace=kafka
```

Verify that topics continue to reconcile normally after both changes.

#### Step 2: Enable multi-namespace watching

**Kafka-managed mode:**

Update the `Kafka` CR to add `watchedNamespaces`. Because an explicit list means *exactly* those namespaces, the `kafka` namespace must be listed if you want existing topics there to keep being reconciled:

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
        - kafka      # the Kafka CR's own namespace — must be listed explicitly
        - team-a
        - team-b
```

The Cluster Operator will:
1. Create `Role`/`RoleBinding` in `kafka`, `team-a`, and `team-b`
2. Update `STRIMZI_RESOURCE_LABELS` to `strimzi.io/cluster=my-cluster,strimzi.io/cluster-namespace=kafka`
3. Restart the Topic Operator

Existing topics in the `kafka` namespace keep their original Kafka topic names because they were pinned with `spec.topicName` and relabeled in Step 1, and `kafka` is included in `watchedNamespaces`. Any existing topic that was **not** pinned in Step 1 will be re-resolved to a prefixed name (e.g. `orders` → `kafka.orders`) and its original Kafka topic will be orphaned

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

A shared Topic Operator instance watching multiple tenant namespaces has inherent multi-tenancy implications that operators should understand.

#### Gating multi-namespace mode (CVE-2026-55225)

[CVE-2026-55225](https://github.com/advisories/GHSA-mw9r-p8xp-wx96) established that cross-namespace watching by an operator must not be silently available, because granting an operator reach into other namespaces broadens its blast radius. The same principle applies here: while the Topic Operator only ever needs RBAC on `KafkaTopic` resources in watched namespaces, The specific *Secret-exfiltration* attack vector from the CVE does not apply — the broader principle does. Topic names can be confidential, and an operator with cross-namespace reach could be used to interfere with another tenant's topics.

Therefore, multi-namespace watching is **opt-in and gated**, consistent with how the singular `watchedNamespace` is handled after the CVE fix:

- `watchedNamespaces` is **rejected** (admission-time validation on the `Kafka` CRD; startup error in standalone mode) unless the `STRIMZI_ENTITY_OPERATOR_WATCHED_NAMESPACE_ENABLED` environment variable is enabled.
- The flag defaults to **disabled**. With it disabled, only the TO's own namespace is watched, exactly as today.
- The cluster administrator must consciously enable the flag to allow any cross-namespace watching, which keeps the decision in the hands of whoever controls the operator deployment rather than the tenant who edits the `Kafka` CR.

#### Exposure model

This proposal makes multi-namespace watching available **both** via the `Kafka` CR (`spec.entityOperator.topicOperator.watchedNamespaces`, Cluster-Operator–managed) **and** via the standalone deployment, in both cases behind the gate above.

There is a competing view (raised in review) that, for symmetry with the User Operator and to keep the Cluster Operator from minting cross-namespace RBAC on a tenant's behalf, multi-namespace watching should be available **only** in standalone mode and not through the `Kafka` CR. This proposal does not adopt that restriction — gating the feature behind a cluster-admin-controlled flag already keeps the privileged decision out of tenant hands, and exposing it through the `Kafka` CR preserves the managed-RBAC convenience that is the main reason to use the Cluster Operator at all. The standalone-only alternative is recorded under [Rejected alternatives](#standalone-only-exposure). Symmetry with a future multi-namespace User Operator is acknowledged as a cross-cutting design concern and is tracked with that work (see [Future work](#future-work)).

#### Other considerations

- **Information leakage:** With namespace-prefixed naming as the default, two teams using the same `metadata.name` never collide, so no cross-tenant information is surfaced. Leakage is only possible on the `spec.topicName` opt-out path, and even then the `ResourceConflict` message is deliberately generic (`"Already managed by another topic operator"`) and does not reveal the owning `namespace/name`. Teams that opt out into a shared explicit topic name are, by definition, coordinating on that name already.

- **Blast radius:** A misconfiguration or runaway workload in one namespace can affect reconciliation for all other namespaces served by the same Topic Operator instance. The per-namespace queue/worker isolation described in [Reconciliation architecture](#reconciliation-architecture-and-noisy-neighbor-isolation) limits this, but operators should still weigh a shared instance against per-team instances for strong isolation.

## Affected/not affected projects

Affected:
- **Topic Operator**: Extended to watch multiple namespaces; namespace-prefixed topic-name resolution (with `spec.topicName` opt-out); per-namespace isolation via an independent informer + `BatchingLoop` + `BatchingTopicController` per namespace (Option C — no shared controller state); reactive conflict handling for the opt-out case; honours the `STRIMZI_ENTITY_OPERATOR_WATCHED_NAMESPACE_ENABLED` gate.
- **Cluster Operator**: Creates/reconciles RBAC resources for watched namespaces, passes `watchedNamespaces` configuration and `STRIMZI_RESOURCE_LABELS` to the Entity Operator, and enforces the gating flag and the `watchedNamespace`/`watchedNamespaces` mutual-exclusion validation on the `Kafka` CRD.

Not affected:
- `strimzi.io/cluster` label value and semantics are unchanged across all resource types
- KafkaTopic CRD schema (no changes to spec fields)

## Compatibility

- The `watchedNamespaces` field is optional with an empty default, preserving existing single-namespace behavior (the Kafka cluster namespace when deployed as part of the CO, or the TO namespace when deployed as standalone)
- Multi-namespace watching is gated behind `STRIMZI_ENTITY_OPERATOR_WATCHED_NAMESPACE_ENABLED` (default disabled); when disabled, `watchedNamespaces` is rejected and behavior is identical to today. Existing single-namespace deployments are therefore unaffected regardless of the flag.
- When multi-namespace mode is enabled, topic names are namespace-prefixed by default (`<namespace>.<name>`). This does **not** affect single-namespace deployments. Existing topics that are migrated into a watched namespace must set `spec.topicName` to retain their original (unprefixed) Kafka topic name — otherwise the TO will resolve them to a new, prefixed name.
- No changes to the `KafkaTopic` CRD spec schema.
- `strimzi.io/cluster` label: unchanged. The label value remains the cluster name (e.g., `my-cluster`). No deprecation, no migration required for existing deployments.
- `strimzi.io/cluster-namespace` label: new, only required when `watchedNamespaces` is configured. Single-namespace deployments are not affected. In Kafka-managed mode, the Cluster Operator handles the `STRIMZI_RESOURCE_LABELS` configuration automatically.
- `watchedNamespace` (singular) is deprecated. Setting both `watchedNamespace` and `watchedNamespaces` simultaneously is rejected at admission time via CEL validation on the `Kafka` CRD. The migration path is: `watchedNamespace: team-a` → `watchedNamespaces: [team-a]` — no label, RBAC, or KafkaTopic changes needed.
- Existing `KafkaTopic` resources continue to work without modification in single-namespace mode. Enabling multi-namespace mode (`watchedNamespaces`) is **not** a transparent switch: because prefixing applies to all watched namespaces (including the home namespace), existing topics must be (a) pinned with `spec.topicName` to keep their current Kafka topic name and (b) labeled with `strimzi.io/cluster-namespace` *before* the switch, otherwise they will be re-resolved to prefixed names and their original Kafka topics orphaned. See the migration example for the required steps.

## Future work

- **Configurable topic-name / config policy CR.** The namespace prefix defined in [Topic name resolution](#topic-name-resolution) is the built-in default that ships with this feature. A richer, configurable policy mechanism — per-namespace allowed/forbidden topic configs, custom or mandatory prefixes, maximum partition counts, etc. — is deferred to a dedicated `KafkaTopicPolicy`-style CR, tracked in a separate issue: _(link the tracking issue opened in `strimzi/strimzi-kafka-operator`)_. This is follow-up only; the namespace prefix does not depend on it.
- **Multi-namespace support for the User Operator** (`KafkaUser` resources) — this will require a separate proposal addressing the security implications of cross-namespace ACL management (see [PR #137 discussion](https://github.com/strimzi/proposals/pull/137)). For consistency, the gating flag and exposure model chosen here should be revisited together with the UO so the two operators stay symmetric.

## Rejected alternatives

### New `KafkaNamespaceTopic` CRD

[Issue #1206](https://github.com/strimzi/strimzi-kafka-operator/issues/1206) proposed a `KafkaNamespaceTopic` CRD that would encode the Kubernetes namespace in the Kafka topic name (e.g., `team-a.orders`). This approach was rejected because:

1. **CRD maintenance cost**: Each custom resource is expensive to maintain. A new CRD requires its own controller logic, status management, validation webhooks, documentation, and migration tooling.
2. **Rigidly forces a naming convention**: The new-CRD approach bakes `<namespace>.<topic>` into a separate resource type with no escape hatch. This proposal also uses `<namespace>.<topic>` as the *default* (see [Topic name resolution](#topic-name-resolution)), but on the existing `KafkaTopic` CR and with an explicit opt-out via `spec.topicName` for users whose topics must not be prefixed — so it gets the same isolation guarantee without forcing the convention on everyone or requiring a new CRD.
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

### Standalone-only exposure

An alternative raised in review is to expose multi-namespace watching **only** through the standalone Topic Operator deployment, and not through the `Kafka` CR — so that the Cluster Operator never creates cross-namespace RBAC on a tenant's behalf, and the feature stays symmetric with how a future multi-namespace User Operator might be restricted.

This was not adopted because:

1. **The gate already moves the privileged decision to the right actor.** `STRIMZI_ENTITY_OPERATOR_WATCHED_NAMESPACE_ENABLED` defaults to disabled and can only be enabled by whoever deploys the operator (the cluster administrator), not by the tenant editing the `Kafka` CR. The security goal — no silent cross-namespace reach — is met without removing the Kafka-CR path.
2. **Managed RBAC is the main value of the Cluster Operator.** Forcing every multi-namespace user into standalone mode means hand-maintaining `Role`/`RoleBinding` resources in every watched namespace, which is exactly the toil the Cluster Operator exists to remove.
3. **Symmetry can be preserved at the gate, not the deployment mode.** The same flag and exposure model can be applied to the User Operator when it gains multi-namespace support, keeping the two operators consistent (see [Future work](#future-work)).

The trade-off is documented here so maintainers can weigh it explicitly; if symmetry with the UO is judged to outweigh the managed-RBAC convenience, restricting to standalone-only remains a viable fallback.
