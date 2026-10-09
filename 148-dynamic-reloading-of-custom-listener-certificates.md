# Dynamic reloading of custom listener certificates

Reload custom listener certificates (`brokerCertChainAndKey`) on broker nodes through Kafka's dynamic configuration instead of rolling the pods.
This addresses [strimzi-kafka-operator#9994](https://github.com/strimzi/strimzi-kafka-operator/issues/9994).

## Current situation

When the certificate in a `brokerCertChainAndKey` Secret changes, the operator rolls all broker nodes.
It tracks the certificate through a thumbprint in the `strimzi.io/custom-listener-cert-thumbprints` Pod annotation, which is part of the Pod revision.

Since Strimzi 0.49 ([#11447](https://github.com/strimzi/strimzi-kafka-operator/pull/11447)), listener keystores are no longer files.
The operator copies the custom certificate and key into the per-node Secrets, and the broker configuration references them as PEM values through the `KubernetesSecretConfigProvider`:

```properties
listener.name.external-9094.ssl.keystore.certificate.chain=${strimzisecrets:myproject/my-cluster-kafka-0:external-9094.crt}
listener.name.external-9094.ssl.keystore.key=${strimzisecrets:myproject/my-cluster-kafka-0:external-9094.key}
listener.name.external-9094.ssl.keystore.type=PEM
```

## Motivation

Users rotating short-lived certificates from external PKI (cert-manager, Vault, and similar) get a full rolling update on every renewal.
Kafka has supported reloading listener keystores without a restart since KIP-226.

The blockers named in #9994 no longer apply:

* Reloading certificates with a different DN or SANs needs KIP-978 (`ssl.allow.dn.changes`, `ssl.allow.san.changes`), available since Kafka 3.8.0 and therefore in every Kafka version Strimzi supports today.
* Preparing keystore files inside the container is obsolete since the PEM change above: the broker reads the certificate from the Secret whenever the listener is (re)configured.
* Tracking the currently used certificate does not require reading sensitive values back from Kafka: the operator already records the applied thumbprint in the Pod annotation.

## Proposal

A broker node qualifies for a certificate reload when the feature gate is enabled and the only change detected for it is a custom listener certificate thumbprint.
For such a node, the operator reloads the certificate instead of restarting it:

1. The per-node Secrets are updated with the new certificate and key (happens today already), before any reload call, so the config provider reads the new material.
2. The operator sets `listener.name.<listener>.ssl.keystore.certificate.chain` and `.key` as per-broker dynamic configurations via `incrementalAlterConfigs`, using the same `${strimzisecrets:...}` placeholders as the static configuration.
3. The broker resolves the placeholders from the Secret, validates the new keystore, and swaps the SSL context of the listener.
4. On success, the operator patches the thumbprint annotation on the Pod.
5. On failure, it falls back to restarting the node through the KafkaRoller (new restart reason, e.g. `CustomListenerCertificatesUpdated`).

The reload is a dedicated step with its own `incrementalAlterConfigs` call, separate from the normal configuration reconciliation, whose diff deliberately skips listener-prefixed options (see the cleanup section below).
The reload is done node by node, like dynamic configuration updates today.
Because the config provider reads the Secret from the Kubernetes API, there is no dependency on kubelet volume propagation.

The thumbprint annotation is the record of what is applied: it is only updated after a successful reload, and a reload is only triggered when it differs from the desired thumbprint.
This keeps the mechanism idempotent; if the operator crashes in between, the next reconciliation just reloads the same certificate again.
With the feature gate enabled, the annotation is excluded from the Pod revision calculation and evaluated separately in the rolling update decision, so it can be patched without triggering a restart.

Everything else keeps the current rolling update behavior: CA renewals and replacements, cluster-CA-signed server certificates, and any other configuration or Pod change.
KRaft controller listeners cannot be dynamically reconfigured in Kafka at all, so controllers are out of scope.
The behavior is introduced behind a new feature gate (e.g. `DynamicCertificateReload`) with the usual alpha, beta, GA progression, and each reload is recorded as a Kubernetes event on the Pod.

### DN and SAN validation

With the feature gate enabled, the operator sets `ssl.allow.dn.changes=true` and `ssl.allow.san.changes=true` on broker nodes.
These options are broker-wide, so they also lift the DN and SAN checks for dynamic keystore updates on the replication and control plane listeners, not just on the custom-certificate listeners.
This does not weaken the security model in practice: dynamic keystore updates can only be issued by the operator or by someone who already holds equivalent Kafka admin access, and the certificate source of truth remains the user-provided Secret, protected by RBAC.
Both options are static, so enabling the gate on an existing cluster costs one last rolling update.

### Cleaning up the dynamic overrides

The overrides set by this mechanism are per-broker dynamic configurations that live in the cluster metadata, independent of the pod.
`KafkaConfigurationDiff` will not manage them: it already classifies listener-prefixed options as custom configuration and skips them in both directions, neither setting nor removing them (see the note on `isCustomConfigurationOption` in `KafkaConfiguration`).
This is convenient during reloads, but it means the overrides are never cleaned up automatically, and a stale override takes precedence over the static configuration.

The operator therefore removes the overrides explicitly in two cases:

* When the listener stops using a custom certificate (`brokerCertChainAndKey` removed from the `Kafka` resource); otherwise the broker keeps serving the old custom certificate from the stale override.
* When the feature gate is disabled; the operator deletes the overrides and falls back to the rolling update behavior.

After a broker restart the static and dynamic configurations resolve to the same value, so a leftover override is not harmful in itself, but removing it keeps the running configuration honest.

### To be validated with a proof of concept

The PEM keystore configs (`ssl.keystore.key`, `ssl.keystore.certificate.chain`) are in Kafka's reconfigurable SSL config set, so per-listener dynamic updates of them are supported.
Their behavior together with config providers is not documented, but the Kafka code supports both halves of the reload trigger:

* When a dynamic override arrives from the metadata log, the broker merges it with the static configuration and constructs a new `KafkaConfig`; the merged configuration contains the `config.providers` definitions from `server.properties`, so `${strimzisecrets:...}` placeholders are resolved at that point and the Secret is read fresh from the Kubernetes API.
* Setting an override to its existing value is not a no-op: for broker resources, the KRaft controller emits the configuration record even when the value is unchanged, precisely to trigger such reloads on the brokers ([KAFKA-14136](https://issues.apache.org/jira/browse/KAFKA-14136)).

Together this means the operator can set the same placeholder value on every rotation and rely on the broker to re-resolve it.
Since neither behavior is a documented contract, the implementation starts with a proof of concept that exercises the full path, and the system tests pin it down for future Kafka versions.

If placeholders turn out not to resolve in dynamic configurations, the fallback is to set the PEM values inline, at the cost of persisting the private key in the KRaft metadata log.

## Affected/not affected projects

Only the Cluster Operator is affected (KafkaRoller, StrimziPodSet revision handling, and the new reload and cleanup logic).
The KafkaAgent, the other operands, and plugins such as the quotas plugin are not affected.

## Compatibility

There are no API changes.
All supported Kafka versions include KIP-226, KIP-651, and KIP-978, so there is no per-version handling.
Disabling the feature gate makes the operator remove the dynamic overrides it created and return to the rolling update behavior.
Downgrading to an operator without this feature is a corner case: the old operator does not know about the overrides, so they should be removed (by disabling the gate first, or manually) before the downgrade to avoid a stale override taking precedence over the static configuration.

## Rejected alternatives

### Watching certificate files inside the container

This approach is obsolete since the PEM change: there are no keystore files to watch or rebuild, and kubelet volume propagation has unbounded latency.

### Setting the PEM values inline in the dynamic configuration

Setting the values inline would guarantee a value change on every rotation, but it persists the private key in the KRaft metadata log.
It is kept only as a fallback if placeholders turn out not to work in dynamic configurations.

### Reloading operator-managed server certificates too

CA renewals also change trust stores on brokers, controllers, operands, and clients, and controllers cannot reload at all, so the restart-free path would cover only a fraction of the affected components.
This can be revisited once the mechanism is proven for custom listener certificates.
