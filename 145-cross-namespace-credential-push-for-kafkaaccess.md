# Cross-namespace Credential Distribution for KafkaAccess

This proposal introduces a push-based model for the Kafka Access Operator to securely deliver Kafka credentials across namespaces, replacing the current pull-based design that allows any user with namespace access to obtain Kafka credentials without explicit authorization from the credential owner.

## Current situation

The Kafka Access Operator currently uses a **pull-based model**.
A user creates a `KafkaAccess` CR in their own namespace (e.g. `dev`) and references `Kafka` and `KafkaUser` resources that reside in a different namespace (e.g. `kafka`).
The operator then creates a `Secret` in `dev` — the namespace of the `KafkaAccess` CR — containing all the Kafka connection details and credentials.

```yaml
apiVersion: access.strimzi.io/v1alpha1
kind: KafkaAccess
metadata:
  name: kafka-access
  namespace: dev                 # KafkaAccess lives here; Secret will be created here too
spec:
  kafka:
    name: kafka-cluster
    namespace: kafka             # cross-namespace reference
    listener: plain
  user:
    apiGroup: kafka.strimzi.io
    kind: KafkaUser
    name: kafka-user
    namespace: kafka             # cross-namespace reference
```

The Kafka Access Operator is granted a `ClusterRole` that allows it to read `KafkaUser` and `Kafka` resources and their associated secrets across all namespaces, and to create/update secrets in any namespace where a `KafkaAccess` might be placed.

## Motivation

The current pull-based design has a significant security concern: any user who has permission to create `KafkaAccess` CRs in any namespace can obtain Kafka credentials for any `KafkaUser` they can name, even if they have no legitimate access to the Kafka infrastructure.
Any user with write access to a namespace can create a `KafkaAccess` object referencing an existing `KafkaUser` in another namespace, and the Kafka Access Operator will create a `Secret` containing full Kafka credentials in that user's namespace.
There is no mechanism today that allows the owner of the `KafkaUser` to control which namespaces are permitted to receive its credentials.

This violates the principle of least privilege in two ways:

1. **For end users**: A user with write access to one namespace can escalate privileges by obtaining Kafka credentials they are not supposed to have.
2. **For the operator itself**: The operator requires broad write permissions across all namespaces to create Secrets wherever a `KafkaAccess` CR may appear, making the blast radius of a compromised operator much larger than necessary.

## Proposal

This proposal changes the Kafka Access Operator to a **push-based model** with two variants depending on whether a `KafkaUser` CR is involved.

**Without KafkaUser**, the `KafkaAccess` CR is created by the Kafka namespace admin and specifies a `targetNamespace` to push the credential `Secret` to. Only users with write access to the Kafka namespace can authorize credential delivery.

```yaml
apiVersion: access.strimzi.io/v1alpha1
kind: KafkaAccess
metadata:
  name: kafka-access
  namespace: kafka                 # created by kafka admin
spec:
  kafka:
    name: kafka-cluster
    namespace: kafka
    listener: plain
  targetNamespace: dev             # push Secret to here
```

**With KafkaUser**, the `KafkaUser` owner must annotate the CR with an allowlist of permitted target namespaces. The `KafkaAccess` CR is still created by the Kafka namespace admin, and the operator validates the annotation before pushing the Secret.

```yaml
# Step 1: kafka admin annotates KafkaUser with allowed namespaces
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: kafka-user
  namespace: kafka
  annotations:
    access.strimzi.io/allowed-namespaces: "dev"
spec:
  authentication:
    type: scram-sha-512
```

```yaml
# Step 2: kafka admin creates KafkaAccess in the source namespace
apiVersion: access.strimzi.io/v1alpha1
kind: KafkaAccess
metadata:
  name: kafka-access
  namespace: kafka                 # created by kafka admin, same as Kafka/KafkaUser
spec:
  kafka:
    name: kafka-cluster
    namespace: kafka
    listener: plain
  user:
    kind: KafkaUser
    apiGroup: kafka.strimzi.io
    name: kafka-user
    namespace: kafka
  targetNamespace: dev             # push Secret to here
```

The operator validates both conditions before creating the Secret:

1. The `KafkaAccess` resides in the same namespace as the referenced `Kafka` and `KafkaUser`
2. The `KafkaUser` annotation permits the `spec.targetNamespace`

### API changes

A new optional field `spec.targetNamespace` is added to the `KafkaAccess` CR.
When set, the operator creates the binding `Secret` in the specified namespace instead of in the namespace of the `KafkaAccess` CR itself.

```yaml
apiVersion: access.strimzi.io/v1alpha1
kind: KafkaAccess
metadata:
  name: kafka-access
  namespace: kafka                   # source namespace (where Kafka/KafkaUser live)
spec:
  kafka:
    name: kafka-cluster
    namespace: kafka
    listener: plain
  user:
    apiGroup: kafka.strimzi.io
    kind: KafkaUser
    name: kafka-user
    namespace: kafka
  targetNamespace: dev               # new field: where to push the Secret
```

When `spec.targetNamespace` is not set, the Secret is created in the same namespace as the `KafkaAccess` CR, preserving backwards compatibility for single-namespace deployments.

`spec.targetNamespace` is only honoured when the `KafkaAccess` CR resides in the same namespace as the referenced `Kafka` and `KafkaUser` resources. If a `KafkaAccess` CR in a different namespace sets `spec.targetNamespace`, the operator rejects it with a `Ready=False` status condition and reason `CrossNamespacePushNotAllowed`.

### KafkaUser namespace allowlist annotation

When both `spec.user` and `spec.targetNamespace` are set, the `KafkaUser` CR **must** carry the annotation `access.strimzi.io/allowed-namespaces` listing every namespace permitted to receive its credentials.
If the annotation is absent or the target namespace is not in the list, the operator rejects the reconciliation with a `Ready=False` status condition and reason `NamespaceNotAllowed`.

This is required rather than optional because a `KafkaUser` carries real authentication credentials whose blast radius depends on its ACL permissions.
Requiring an explicit allowlist forces the `KafkaUser` owner to declare intended recipients at the time the user is created, providing a safety net against accidental misconfiguration in `KafkaAccess`.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: kafka-user
  namespace: kafka
  annotations:
    access.strimzi.io/allowed-namespaces: "dev,staging"
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  authentication:
    type: scram-sha-512
```

When `spec.user` is not set, no annotation check is performed.

### Owner references and finalizers

The current implementation sets an `OwnerReference` on the created `Secret` pointing to the `KafkaAccess` CR, allowing Kubernetes garbage collection to automatically delete the `Secret` when the `KafkaAccess` is deleted.
Kubernetes does not support cross-namespace owner references, so this mechanism cannot be used when the `Secret` is pushed to a different namespace via `spec.targetNamespace`.

When `spec.targetNamespace` differs from the `KafkaAccess` namespace, the operator must instead:

1. Add a **finalizer** to the `KafkaAccess` CR on creation.
2. On deletion of the `KafkaAccess` CR, explicitly delete the `Secret` in the target namespace before removing the finalizer.

When `spec.targetNamespace` is not set, the existing `OwnerReference`-based cleanup is preserved.

### RBAC implications

With the push-based model, the operator's required RBAC permissions become more predictable and narrower:

- **Read permissions** remain on `Kafka`, `KafkaUser`, and their secrets in the source namespace (unchanged).
- **Write permissions** for Secrets are now scoped to explicitly declared target namespaces rather than all namespaces.

This allows operators to be deployed with more restrictive ClusterRole or Role bindings.
In a future iteration, the operator could accept per-`KafkaAccess` RBAC delegation (e.g. via a projected ServiceAccount token) to further restrict write scope to individual Secrets, but that is out of scope for this proposal.

### Migration from pull-based to push-based

Existing `KafkaAccess` deployments where the CR and the Kafka resources are in the same namespace continue to work without any changes.

For existing **cross-namespace** deployments (where `KafkaAccess` is in a different namespace from the `Kafka`/`KafkaUser` it references), a migration path is required:

1. The operator will emit a **deprecation warning** in its logs and via a `Warning`-level status condition when it detects a cross-namespace pull pattern (i.e. `KafkaAccess` namespace differs from the referenced `KafkaUser`/`Kafka` namespace, and `spec.targetNamespace` is not set).
2. Users are expected to migrate by:
   - Moving the `KafkaAccess` CR into the source namespace.
   - Adding `spec.targetNamespace` pointing to the namespace where the Secret is needed.
3. After a suitable deprecation period the pull-based cross-namespace behavior will be removed.

### Status conditions

The `KafkaAccess` status is updated to reflect push outcomes:

- A new condition type `SecretDelivered` is introduced with reason `NamespaceNotAllowed` when the `KafkaUser` annotation allowlist check fails.
- Existing `Ready` condition semantics are unchanged.

## Affected/not affected projects

**Affected:**

- `strimzi/kafka-access-operator` — API changes to `KafkaAccess` CRD, reconciler logic, RBAC manifests, documentation, and examples.

**Not affected:**

- `strimzi/strimzi-kafka-operator` — No changes to `Kafka`, `KafkaUser`, or any other core operator CRDs are strictly required.
  The `access.strimzi.io/allowed-namespaces` annotation is added to `KafkaUser` as an advisory annotation read only by the Access Operator; it does not modify the `KafkaUser` CRD schema.

## Compatibility

Adding `spec.targetNamespace` as an optional field is backwards compatible at the **schema level** — existing `KafkaAccess` CRs without `spec.targetNamespace` continue to be accepted by the CRD.

At the **behavioral level**, compatibility is time-bounded. Existing cross-namespace pull deployments (where `KafkaAccess` is in a different namespace from the referenced `Kafka`/`KafkaUser`) will continue to function during the deprecation period but will receive a deprecation warning. Once the pull pattern is removed in a future version, any unmigrated `KafkaAccess` CRs will stop working.

Users with cross-namespace pull deployments are expected to migrate within the deprecation window by:

1. Creating a new `KafkaAccess` in the source namespace with `spec.targetNamespace` set.
2. Verifying the Secret is correctly pushed and the application is healthy.
3. Deleting the old `KafkaAccess` CR. The Secret created by the old `KafkaAccess` will be automatically garbage collected by Kubernetes via the existing `OwnerReference`.

The `access.strimzi.io/allowed-namespaces` annotation is required only for `KafkaAccess` CRs that reference a `KafkaUser` and use `spec.targetNamespace`. Existing single-namespace deployments without `spec.targetNamespace` are unaffected.

## Rejected alternatives

### ReferenceGrant (Gateway API)

Using `gateway.networking.k8s.io/v1beta1/ReferenceGrant` as the authorization object (as suggested by the issue author) would require users to install the Gateway API CRDs and introduce a dependency on an API group whose long-term home is [still unresolved](https://github.com/kubernetes/enhancements/issues/3766).
Additionally, `ReferenceGrant` is not enforced by Kubernetes itself; the operator would still need custom logic to check it, making it an indirect dependency with no clear benefit over a native annotation or a dedicated Strimzi CR.

### Custom grant CRD

Introducing a new `KafkaAccessGrant` CRD in the source namespace to model the authorization would be more explicit and discoverable than an annotation.
However, for the initial implementation, an annotation on `KafkaUser` is simpler and avoids adding another CRD to the operator's footprint.
A dedicated CRD can be considered in a follow-up if richer grant semantics (e.g. time-bounded grants, per-listener scoping) are needed.

### No change / annotation-only on KafkaAccess

An alternative of only adding a `spec.allowedNamespaces` list to the existing `KafkaUser` annotation without changing where `KafkaAccess` lives keeps the pull model intact but merely validates it server-side.
This still requires the operator to hold broad write permissions across all namespaces and does not reduce the blast radius of a compromised operator.
The push model is architecturally sounder and is therefore preferred.
