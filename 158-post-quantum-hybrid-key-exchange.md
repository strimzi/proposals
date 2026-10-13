# Post-Quantum Hybrid Key Exchange (PQHKE) support in Strimzi

This proposal describes how Strimzi enables Post-Quantum Hybrid Key Exchange (PQHKE) using ML-KEM (via the `X25519MLKEM768` named group) by updating the JDK baseline, and how it enforces TLS 1.3 on all Kafka listeners by default via a Gatekeeper plugin so that PQHKE is guaranteed on every connection where both peers support it.

This proposal covers key exchange only by using [ML-KEM (FIPS 203)](https://csrc.nist.gov/pubs/fips/203/final) via [JEP-527 ("Hybrid Post-Quantum Key Exchange in TLS 1.3")](https://openjdk.org/jeps/527).
[ML-DSA (FIPS 204)](https://csrc.nist.gov/pubs/fips/204/final) certificate signing via [JEP-497](https://openjdk.org/jeps/497) is a separate, more invasive effort and is a non-goal of this proposal.
It will be tracked in a dedicated follow-up proposal once hybrid key exchange is in place.

## Current situation

Every TLS connection in a Strimzi-managed Kafka cluster (broker-to-broker, controller-to-controller, operator-to-broker, operator-to-Cruise-Control, operator-to-Kubernetes, Connect/MM2/Bridge-to-broker, Kafka Exporter-to-broker) uses classical key exchange algorithms (`ECDH`, `X25519`).

The JDK container images Strimzi ships are based on Java 21 and do not yet include JEP-527, which is the Java standard that wires ML-KEM into TLS 1.3 as the `X25519MLKEM768` named group.

Furthermore, Strimzi currently sets no `ssl.enabled.protocols` on the control plane (port 9090) or replication (port 9091) internal listeners, leaving the Kafka broker default (`TLSv1.2,TLSv1.3`), which means TLS 1.2 connections to these listeners are accepted even though all clients that connect to them support TLS 1.3.
Of course, the same applies for external listeners.
While users can configure `ssl.enabled.protocols` globally via `spec.kafka.config`, this applies to all listeners and per-listener overrides are not possible.

## Motivation

Quantum computers will eventually break classical key exchange algorithms.
The threat already exists as attackers can record encrypted Kafka traffic today and decrypt it later ("Harvest Now, Decrypt Later", HNDL).
Organizations in regulated industries are already being encouraged to begin migration:

* [OMB M-23-02](https://www.whitehouse.gov/wp-content/uploads/2022/11/M-23-02-M-Memo-on-Migrating-to-Post-Quantum-Cryptography.pdf): *"agencies prepare now to implement post-quantum cryptography (PQC) [...] mitigating as much of the quantum risk as is feasible by 2035"*
* [NSM-10](https://bidenwhitehouse.archives.gov/briefing-room/presidential-actions/2022/05/04/national-security-memorandum-on-promoting-united-states-leadership-in-quantum-computing-while-mitigating-risks-to-vulnerable-cryptographic-systems/): *"the United States must prioritize the timely and equitable transition of cryptographic systems to quantum-resistant cryptography"*
* [NSA CNSA 2.0](https://media.defense.gov/2025/May/30/2003728741/-1/-1/0/CSA_CNSA_2.0_ALGORITHMS.PDF): *"NSA expects the transition to QR algorithms for NSS to be complete by 2035 [...] Custom applications and legacy equipment: update or replace by 2033"*
* [EU Coordinated Roadmap](https://digital-strategy.ec.europa.eu/en/policies/post-quantum-cryptography): *"All Member States should start transitioning to PQC by the end of 2026"*
* [NIST](https://www.nist.gov/cybersecurity-and-privacy/what-post-quantum-cryptography): *"We encourage organizations to begin their transition to these standards immediately to ensure their data remains secure in the quantum era"*

NIST standardised ML-KEM (FIPS 203) as the Post-Quantum key encapsulation mechanism for TLS.
The hybrid approach (`X25519MLKEM768`) combines classical `X25519` and `ML-KEM` in the same handshake, so the session key is secure as long as either algorithm holds.
The fallback is possible because the server's named groups list contains both `X25519MLKEM768` and classical groups such as `X25519`.
A non-PQC client that does not advertise any hybrid group negotiates the highest classical group that both sides list, with no connection failure and no configuration needed on either side.
If the server were to offer only `X25519MLKEM768` and no classical groups, non-PQC clients would fail to connect.

The cloud-native ecosystem around Strimzi is already moving in this direction.
Kubernetes 1.33 (built with Go 1.24) negotiates `X25519MLKEM768` by default on the API server side.
Go 1.24+ enables `X25519MLKEM768` by default in TLS 1.3 client connections, which means the Kafka Exporter already supports it on the client side (it's currently using Go 1.27).

## Proposal

Supporting PQHKE within a Strimzi-managed Kafka cluster requires two independent but complementary changes:

* updating container images to a JDK that includes JEP-527.
* enforcing TLS 1.3 on all Kafka listeners by default via a Gatekeeper plugin.

### Update container images to Java 25 (October 2026 Critical Patch Update)

JEP-527 is the Java standard that integrates ML-KEM into TLS 1.3 as `X25519MLKEM768`.
It landed in Java 27 (September 2026) and is being backported to Java 25 LTS via the October 2026 Critical Patch Update (CPU).

Java 24 added ML-KEM as a cryptographic primitive (JEP 496) but did not integrate it into TLS.
`X25519MLKEM768` is only available as a TLS named group starting with Java 25 (post-October 2026 CPU) or Java 27.
For more details about the backport of JEP-527 to Java 25 LTS look at OpenJDK 25.0.5 release [here](https://wiki.openjdk.org/spaces/JDKUpdates/pages/170131468/JDK+25u).

Strimzi should not wait for Java 27 to be the baseline because it's not an LTS release and also because Java 25 LTS will receive JEP-527 via the October 2026 CPU and is already the supported LTS at the time of writing.

This is a runtime update only and no code changes to Strimzi itself are required.
The change affects every JVM-based Strimzi container image: the operator, Kafka brokers and controllers, Kafka Connect, MirrorMaker 2, HTTP Bridge, Cruise Control, Drain Cleaner and MQTT bridge.
All of these images will be updated to use Java 25 LTS (post-October 2026 CPU) as their JDK runtime base.

With JEP-527 in the JDK, `x25519mlkem768` is added to the JVM's default named groups automatically and it's at the top of the list.
No `jdk.tls.namedGroups` configuration is needed at JVM level.
When both peers support hybrid groups, `X25519MLKEM768` is negotiated.
When a peer supports only classical groups, TLS group negotiation picks the best group that appears in both peers' lists; because the JVM's default list retains `X25519` and other classical groups below `X25519MLKEM768`, a classical-only peer finds a match and the handshake succeeds transparently.

This is a hard prerequisite: without JEP-527 in the JDK, no PQHKE is possible on any Java-based connection regardless of any other configuration.

#### Impact on custom base images

Users who build their own container images based on alternative distributions (Alpine Linux, Chainguard, etc.) can enable PQHKE by updating those images to a JDK runtime that includes JEP-527: Java 25 (October 2026 CPU or later), Java 27+, or a backported build of JEP-527 for Java 21 (expected H1 2027 per the [Oracle JRE and JDK Cryptographic Roadmap](https://www.java.com/en/jre-jdk-cryptoroadmap.html)) or previous versions.
Strimzi will continue to work correctly with custom images based on a JDK without JEP-527 support, but PQHKE will not be available and TLS connections will fall back to classical key exchange transparently.

### Enforce TLS 1.3 on all Kafka listeners via a Gatekeeper plugin

Strimzi provides the [Gatekeeper](https://github.com/strimzi/proposals/blob/main/144-Strimzi-Gatekeeper-plugin-system.md) plugin system that allows mutating plugins to modify a `Kafka` custom resource at the beginning of each reconciliation before the operator processes it.
This proposal ships a new default mutating Gatekeeper plugin, `Tls13EnforcerGatekeeperPlugin`, that injects the following into `spec.kafka.config` during the entry phase of every `Kafka` reconciliation:

```yaml
ssl.protocol: TLSv1.3
ssl.enabled.protocols: TLSv1.3
```

Both `ssl.protocol` and `ssl.enabled.protocols` are explicitly allowed in `spec.kafka.config` (they are exceptions to the forbidden `ssl.` prefix) and apply globally to all Kafka listeners, both internal and external.

The same plugin also handles `KafkaConnect`, `KafkaMirrorMaker2`, and `KafkaBridge` reconciliations.
For these components the injected settings are client-side, they control which TLS version the workload advertises when connecting to the Kafka broker.

For `KafkaConnect`, the plugin injects `ssl.protocol: TLSv1.3` and `ssl.enabled.protocols: TLSv1.3` into `spec.config`.
Both properties are allowed there (they are exceptions to the forbidden `ssl.` prefix).

For `KafkaMirrorMaker2`, MM2 has a single target cluster at `spec.target` and a list of mirrors at `spec.mirrors`, each with a `source` entry.
The plugin injects `ssl.protocol: TLSv1.3` and `ssl.enabled.protocols: TLSv1.3` into both `spec.target.config` and into `spec.mirrors[*].source.config` for every mirror.
Both properties are allowed there (they are exceptions to the forbidden `ssl.` prefix).

For `KafkaBridge`, the Bridge exposes separate Kafka client configurations for its three client roles: `spec.consumer.config`, `spec.producer.config`, and `spec.adminClient.config`.
The plugin injects `ssl.protocol: TLSv1.3` and `ssl.enabled.protocols: TLSv1.3` into all three.
Both properties are allowed in each of them (they are exceptions to the forbidden `ssl.` prefix).

The plugin is added to the `DEFAULT_GATEKEEPER_PLUGINS` list in the Cluster Operator, so it is active out of the box.
Users can opt out by overriding the `STRIMZI_GATEKEEPER_DEFAULT_PLUGINS` environment variable on the Cluster Operator deployment, or by setting `ssl.enabled.protocols` and `ssl.protocol` explicitly in their `Kafka`, `KafkaConnect`, `KafkaMirrorMaker2`, and `KafkaBridge`, in which case the plugin respects the user-supplied values and does not override them.

Enforcing TLS 1.3 on all listeners and components' clients makes the PQHKE guarantee explicit and auditable at the configuration level rather than inferred from runtime negotiation behavior.
It also addresses the HNDL threat on the data plane: external user-facing listeners now require TLS 1.3, so capable clients (i.e. Java 25 with JEP-527, Go 1.24+) negotiate `X25519MLKEM768` and the session key is quantum-safe from the moment it is established.

#### Impact on external listener compatibility

With TLS 1.3 enforced on all listeners, three client scenarios are possible on an external listener:

* a client connecting via TLS 1.2 will be rejected because TLS 1.2 is no longer offered by the broker. Users with such clients should either disable the plugin via `STRIMZI_GATEKEEPER_DEFAULT_PLUGINS` or set `ssl.enabled.protocols: TLSv1.2,TLSv1.3` in `spec.kafka.config` to restore dual-protocol behavior.
* a client connecting via TLS 1.3 but without ML-KEM support (e.g. a Java client on a JDK version without JEP-527, or a non-Java client that does not support ML-KEM) will connect normally and negotiate classical key exchange as a fallback.
* a client connecting via TLS 1.3 and supporting ML-KEM will negotiate `X25519MLKEM768` as the key exchange mechanism.

TLS group negotiation for scenarios 2 and 3 is intersection-based: each side advertises its supported groups and the highest-preference group that appears in both lists is selected.
PQC-only enforcement (removing classical groups so that non-ML-KEM clients cannot connect at all) is covered under Future Work via KIP-1376.

#### What works after these two changes

| Connection                              | Port     | TLS 1.3                        | ML-KEM                                                |
|-----------------------------------------|----------|--------------------------------|-------------------------------------------------------|
| Broker to broker (replication)          | 9091     | Enforced by Gatekeeper plugin  | Automatic (JEP-527 in broker JVM)                     |
| Controller to controller / broker       | 9090     | Enforced by Gatekeeper plugin  | Automatic (JEP-527 in broker JVM)                     |
| CO / TO / UO Admin client to broker     | 9091     | Enforced by Gatekeeper plugin  | Automatic (JEP-527 in operator JVM)                   |
| CO to Kafka Agent                       | 8443     | Hardcoded in `KafkaAgentClient`| Automatic (JEP-527 in broker JVM)                     |
| CO / TO to Cruise Control HTTP server   | 9090     | Hardcoded in operator clients  | Automatic (JEP-527 in Cruise Control JVM)             |
| Cruise Control Kafka clients to broker  | 9091     | Enforced by Gatekeeper plugin  | Automatic (JEP-527 in Cruise Control JVM)             |
| CO / TO / UO to Kubernetes API          | 443      | Negotiated by preference       | Automatic (K8s 1.33+ Go 1.24, JEP-527 in operator JVM)|
| Connect / MM2 / Bridge to broker        | External | Enforced by Gatekeeper plugin  | Automatic (JEP-527 on both JVMs)                      |
| Kafka Exporter to broker                | 9091     | Enforced by Gatekeeper plugin  | Automatic (Go 1.24+ default in crypto/tls)            |

## Future work: explicit named groups configuration for external listeners (pending KIP-1376)

What is out of scope for this proposal is explicit control over which named groups are offered on a given external listener.
Some users need this level of control beyond the default hybrid behavior:

* PQC-only enforcement: remove classical groups so the TLS handshake fails for peers that do not support any hybrid group.
This ensures that no classical key exchange can occur even as a fallback, which may be required by compliance mandates in their final migration phase.
* Classical-only restriction: remove hybrid groups to prevent ML-KEM from being negotiated, for example during a testing or validation phase where PQC traffic needs to be isolated.
* Preference reordering: prefer a different hybrid group over the default `X25519MLKEM768` (for example `secp384r1mlkem1024` for higher security level), requiring the list to be reordered explicitly.

The Kafka client library currently has no per-client or per-listener configuration property for named groups equivalent to `ssl.enabled.protocols` or `ssl.cipher.suites`.
Named group configuration is only possible JVM-wide via `jdk.tls.namedGroups`, which affects all connections in the same JVM simultaneously and is therefore not suitable as a per-listener knob.

[KIP-1376](https://cwiki.apache.org/confluence/spaces/KAFKA/pages/451974516/KIP-1376+Support+setting+TLS+named+groups) proposes a new `ssl.named.groups` configuration property in Kafka with support for per-listener overrides.
Once KIP-1376 is implemented and available in a supported Kafka version, Strimzi will expose it via a new `spec.kafka.listeners[*].tls.namedGroups` field, allowing users to configure named groups per listener without relying on the JVM-wide property.
Exposing this field will require a dedicated Strimzi proposal at that time.

## Affected/not affected projects

The cluster operator is affected by the addition of the `Tls13EnforcerGatekeeperPlugin` default Gatekeeper plugin.
The Docker base images are updated to Java 25 LTS (post-October 2026 CPU) for the following components:

* the operators
* Kafka brokers and controllers
* Kafka Connect
* Kafka Mirror Maker 2
* HTTP Bridge
* Cruise Control
* MQTT Bridge
* Drain Cleaner

The Drain Cleaner benefits from the Java 25 image update for its two TLS connections: the inbound HTTPS webhook server (receiving calls from the Kubernetes API server) and the outbound Kubernetes API client (Fabric8, same pattern as the operators).
With Java 25 on the Drain Cleaner side and the Kubernetes API server built with Go 1.24+, `X25519MLKEM768` is negotiated automatically on both connections.

The proposal doesn't affect the Kafka Exporter component which is a pre-compiled Go binary already compiled with Go 1.27, which enables `X25519MLKEM768` by default in TLS 1.3 with no changes needed.

## Compatibility

The `Tls13EnforcerGatekeeperPlugin` default Gatekeeper plugin enforces TLS 1.3 on all Kafka listeners and components' clients.
For internal connections (ports 9090 and 9091), this has no backward compatibility risk because every component connecting to them already supports TLS 1.3.
For external user-facing listeners, this is a breaking change for Kafka clients that only support TLS 1.2: those clients will be rejected while the plugin is active.
Users with such clients can disable the plugin or set `ssl.enabled.protocols: TLSv1.2,TLSv1.3` in `spec.kafka.config` to restore the previous behavior.

PQHKE works as a hybrid: when both peers support `X25519MLKEM768`, they use it; when either side does not (for example, a component still running on Java 21), the handshake falls back to classical key exchange transparently.
No connection is broken by the JDK upgrade alone.

Updating the container images to Java 25 (post-October 2026 CPU) is sufficient for PQHKE to become active on all connections, along with the Gatekeeper plugin.
Components running on Java 21 will continue to work using classical key exchange until their images are updated.

## Rejected alternatives

### Stay on Java 21

Staying on Java 21 was rejected because the JEP-527 backport to Java 21 LTS is not expected until 2027 H1 per the [Oracle JRE and JDK Cryptographic Roadmap](https://www.java.com/en/jre-jdk-cryptoroadmap.html).
Waiting for that backport would delay PQHKE by at least a year while organizations in regulated industries are already under pressure to begin migration.

### Wait for next Java LTS after 27 as the baseline

Waiting for the next LTS after Java 27, which will be Java 29 planned for September 2027, was also rejected.
Java 25 LTS already receives JEP-527 via the October 2026 Critical Patch Update and is the supported LTS at the time of writing.
Delaying until a Java LTS after Java 27 becomes the Strimzi baseline would unnecessarily block PQHKE for those same organizations with no technical justification.

### Use BouncyCastle as a JCE provider

Using BouncyCastle as a JCE security provider to get ML-KEM support ahead of JEP-527 was rejected.
BouncyCastle provides its own TLS stack (`BCJSSE`) and JCE provider that support PQC algorithms including ML-KEM and ML-DSA, and could in principle be registered as a JVM security provider to enable PQHKE without upgrading the JDK.

However, this approach introduces significant drawbacks:

* Additional dependency: BouncyCastle would become a production runtime dependency for every Strimzi component, increasing the maintenance burden and the supply chain risk surface.
* Separate TLS implementation: `BCJSSE` is a distinct TLS stack from the JDK's JSSE. Replacing or augmenting JSSE with a third-party TLS implementation requires careful compatibility validation across all TLS connections in Strimzi and introduces a separate security audit requirement.
* Not really necessary: JEP-527 is available in Java 25 (October 2026 CPU) and provides ML-KEM natively in the JDK's JSSE with no external dependencies. The JDK-native path is available now and is the standard solution endorsed by the Java platform.

BouncyCastle remains relevant for test certificate generation, which is already covered by the existing `SystemTestCertGenerator` dependency on BouncyCastle. This is a test-scoped concern and not a reason to introduce it as a production runtime provider.

### Hardcode TLS 1.3 on internal listeners only

An earlier version of this proposal hardcoded `listener.name.controlplane-9090.ssl.enabled.protocols=TLSv1.3` and `listener.name.replication-9091.ssl.enabled.protocols=TLSv1.3` directly in `KafkaBrokerConfigurationBuilder`.
This approach was rejected for three reasons.
First, it gave users no escape hatch: since the `listener.` prefix is forbidden in `spec.kafka.config`, there was no way to override or disable the enforcement.
Second, it left external user-facing listeners on TLS 1.2+1.3, which undermines the HNDL security goal: an attacker targeting data plane traffic can capture it on the external listener, making the internal hardening ineffective in isolation.
Third, the Gatekeeper plugin approach is more consistent with Strimzi's extensibility model, applies uniformly to all listeners, and gives users an explicit opt-out mechanism.
