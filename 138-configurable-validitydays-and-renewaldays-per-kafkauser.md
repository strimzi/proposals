# Configurable `validityDays` and `renewalDays` per `KafkaUser`

Allow configuring mTLS certificate `validityDays` and `renewalDays` individually per `KafkaUser` resource.

## Current situation

The `validityDays` and `renewalDays` fields are currently only configurable at the cluster level via the `Kafka` custom resource:

```yaml
spec:
  clientsCa:
    validityDays: 365
    renewalDays: 30
```

These values apply to all `KafkaUser` certificates managed by the User Operator and there is no way to override them for individual users.

## Motivation

Different clients may have different security requirements.
Short-lived certificates are generally preferable for sensitive clients, while long-lived certificates may be acceptable for others.
Forcing a single expiry policy across all users makes it impossible to apply the principle of least privilege at the certificate level.

This is a common pattern in the ecosystem. For example, cert-manager supports per-`Certificate` `duration` and `renewBefore` fields, and per-`Ingress` annotations (`cert-manager.io/duration`, `cert-manager.io/renew-before`).

## Proposal

Extend the `KafkaUser` spec with optional `validityDays` and `renewalDays` fields under `spec.authentication`:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: my-user
  namespace: my-kafka
spec:
  authentication:
    type: tls
    validityDays: 7
    renewalDays: 2
```

When set, these values override the cluster-level defaults from the `Kafka` resource for that specific user.
When omitted, the cluster-level defaults continue to apply.

If `validityDays` is set or reduced on an existing `KafkaUser` whose current certificate would already be expired or would exceed the new validity period, the User Operator must renew the certificate immediately on the next reconciliation rather than waiting for the natural renewal window.

The User Operator must enforce that `renewalDays` < `validityDays`.
If this constraint is violated, the `KafkaUser` reconciliation must fail with a descriptive status condition.

These fields only apply when using `spec.authentication.type: tls`.
Since the OpenAPI schema cannot enforce cross-field constraints, a CEL validation rule will be used to reject these fields when `type` is not `tls`, ensuring a misconfiguration is caught at admission time.

## Affected/not affected projects

The only affected project is the `strimzi-kafka-operator` repository, especially the User Operator part of the code.

## Compatibility

The new `validityDays` and `renewalDays` fields are optional.
Existing `KafkaUser` resources without these fields continue to behave exactly as before.
This change is fully backwards compatible.

## Rejected alternatives

### Annotations on `KafkaUser`

Using `strimzi.io/validityDays` and `strimzi.io/renewalDays` annotations was considered as a lighter-weight approach (similar to cert-manager's `Ingress` annotations):

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: my-user
  namespace: my-kafka
  annotations:
    strimzi.io/validityDays: 7
    strimzi.io/renewalDays: 2
spec:
  ...
```

However, spec fields are preferable for persistent configuration: schema validation (type checking, range constraints) is possible, they are self-documenting, and consistent with how `validityDays` and `renewalDays` are already defined in the `Kafka` CR.
Annotations are free-form strings with no built-in validation, making it easier to misconfigure values silently.
