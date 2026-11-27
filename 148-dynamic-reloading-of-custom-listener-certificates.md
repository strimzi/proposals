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
* Knowing which certificate a broker uses does not need support from Kafka: the operator can connect to the listener and look at the certificate it serves.

## Proposal

A broker node qualifies for a certificate reload when the new `DynamicCertificateReload` feature gate is enabled and the only reason to restart it is a changed custom listener certificate thumbprint.
Any other restart reason restarts the node as today, and the new certificate is loaded at startup.
Controller-only nodes are always restarted, because KRaft controller listeners cannot be reconfigured dynamically.
CA renewals and cluster-CA-signed server certificates are out of scope.

### Reload and verification

The reload is done by the `KafkaRoller`, node by node:

1. The per-node Secrets already contain the new certificate and key, because `certificateSecrets()` runs before `rollingUpdate()` in the Kafka reconciliation.
2. The operator sets `listener.name.<listener>.ssl.keystore.certificate.chain` and `.key` as per-broker dynamic configurations through `incrementalAlterConfigs`, using the same `${strimzisecrets:...}` placeholders as the static configuration.
   Setting a broker configuration to its current value is not a no-op: the KRaft controller still emits the record ([KAFKA-14136](https://issues.apache.org/jira/browse/KAFKA-14136)), and the broker re-resolves the placeholders, reading the Secret from the Kubernetes API.
3. The operator opens a TLS connection to the listener of the pod (`<pod>.<cluster>-kafka-brokers.<namespace>.svc:<port>`), completes the handshake, and compares the thumbprint of the served certificate with the desired one.
   It retries until they match, for at most 60 seconds or the operation timeout, whichever is shorter.
4. Only when the served certificate matches does the operator patch the thumbprint annotation on the Pod.
5. If any step fails or times out, the node is restarted with a new restart reason, `CustomListenerCertificatesChanged`, and the usual restart event.

The verification step answers why the annotation can be trusted.
A successful `incrementalAlterConfigs` call only means that the receiving broker validated the request; the target broker applies it asynchronously when it replays the metadata record, and `describeConfigs` redacts the PEM values.
The served certificate is the only observable state, so the annotation records what was seen on the wire, not what was requested.
A completed handshake also proves that the broker holds the matching private key; Kafka itself does not catch a mismatched key when the certificate is not trusted by the listener's own truststore.
For listeners with TLS client authentication, the operator connects without a client certificate; with TLS 1.3 the broker rejects it only after the handshake completes, so the check works.
Such listeners limited to TLS 1.2 fail the check and keep being restarted.

The steps are idempotent.
If the operator stops between steps 2 and 4, the next reconciliation finds a stale annotation and repeats the reload, which is harmless.

The reload does not take the broker down, so the `KafkaRoller` availability checks (`canRoll`) are skipped for it.
A reload can be combined with a dynamic configuration update of the same broker in the same pass.

### Pod revision and the PodSet

The annotation stays in the PodSet Pod template, so new or recreated pods get the right value.
The StrimziPodSet controller does not update existing pods, so the only writer of the annotation on a running pod is the `KafkaRoller` after a verified reload.

`PodRevision.getRevision()` removes the annotation before hashing, the same way it already strips container resources for in-place resizing.
`ReconcilerUtils.reasonsToRestartPod()` compares the annotation of the pod with the PodSet template separately (`PodRevision.hasChanged()`) and adds `CustomListenerCertificatesChanged` when they differ.
The `KafkaRoller` then reloads instead of restarting when the gate is enabled and this is the only reason.

The exclusion does not depend on the feature gate.
With the gate disabled, the separate comparison still adds the restart reason, so a changed certificate restarts the node as today.
Because the revision formula changes, clusters with custom listener certificates get one rolling update when the operator is upgraded.

### DN and SAN changes

With the feature gate enabled, the operator sets `listener.name.<listener>.ssl.allow.dn.changes=true` and `listener.name.<listener>.ssl.allow.san.changes=true` for listeners with custom certificates only.
The replication and control plane listeners keep Kafka's DN and SAN checks.
Both options are static, so enabling or disabling the gate rolls the brokers once.

### Network policies

The verification connects from the operator to the listener.
With the feature gate enabled, the operator adds itself to the generated `NetworkPolicy` rule of each listener that has custom certificates and `networkPolicyPeers` set.
Listeners without `networkPolicyPeers` are already open.
If the operator cannot reach the listener, for example because of a user-managed policy, the verification times out and the node is restarted.

### Cleaning up dynamic overrides

The overrides are per-broker dynamic configurations stored in the cluster metadata.
`KafkaConfigurationDiff` skips listener-prefixed options in both directions, so it neither sets nor removes them.

An override for a listener that still uses a custom certificate is harmless: it contains the same placeholders as the static configuration.
An override is stale when its listener was removed or no longer uses a custom certificate, because it points to Secret keys that no longer exist.
Kafka still starts with a stale override (it logs an error and uses the static configuration), but other dynamic configuration updates of that broker fail until it is removed.

These changes always restart the broker, so the `KafkaRoller` deletes the stale overrides of a broker right before restarting it, whether the gate is enabled or not.
They are not deleted earlier because a running broker would fall back to the keystore it loaded at startup.
A failed deletion is logged and does not block the restart; it is retried before the next restart.

## Proof of concept

An implementation is available in [acgtun/strimzi-kafka-operator#4](https://github.com/acgtun/strimzi-kafka-operator/pull/4).
It was tested on minikube with Kafka 4.3.1, 3 controllers, 3 brokers, and Calico enforcing network policies, with producers, consumers, and a TLS probe running against the listeners:

* Rotations with the same placeholders reload on all brokers without restarts or client errors, including DN and SAN changes, NodePort and TLS client authentication listeners, and several listeners at once.
* The operator is allowed by the generated `NetworkPolicy`, and other pods are still blocked.
* Blocking the operator from the listener, or rotating to a certificate with a mismatched key, fails the verification and restarts the node.
* With the gate disabled, rotations restart the brokers as today; toggling the gate rolls the brokers once.
* A broker with an unresolvable dynamic override starts with its static configuration.

System tests will pin the behavior down for future Kafka versions.

## Affected/not affected projects

Only the Cluster Operator is affected (`KafkaRoller`, Pod revision handling, broker configuration, and network policy generation).
The KafkaAgent, the other operands, and plugins such as the quotas plugin are not affected.

## Compatibility

There are no API changes.
All supported Kafka versions include KIP-226, KIP-651, and KIP-978, so there is no per-version handling.
The feature gate starts as alpha and disabled by default.
After downgrading to an operator without this feature, leftover overrides stay harmless as long as the listeners keep their custom certificates.
If a listener later stops using a custom certificate, its overrides have to be removed manually with `kafka-configs.sh`.

## Rejected alternatives

### Watching certificate files inside the container

This approach is obsolete since the PEM change: there are no keystore files to watch or rebuild, and kubelet volume propagation has unbounded latency.

### Setting the PEM values inline in the dynamic configuration

Setting the values inline would guarantee a value change on every rotation, but it persists the private key in the KRaft metadata log.
The proof of concept shows that placeholders work, so this is not needed.

### Trusting the `incrementalAlterConfigs` result

The call returns before the target broker applies the change, so the operator would record a certificate that might not be served yet, or ever.

### Kafka support for reporting the loaded certificate

Kafka could expose a version or identifier of the loaded certificate, so the operator would not need to connect to the listener.
This needs a KIP, and the TLS check gives the same guarantee with the Kafka versions Strimzi supports today.
It can replace the TLS check if such support is added.

### Reloading operator-managed server certificates too

CA renewals also change trust stores on brokers, controllers, operands, and clients, and controllers cannot reload at all, so the restart-free path would cover only a fraction of the affected components.
This can be revisited once the mechanism is proven for custom listener certificates.
