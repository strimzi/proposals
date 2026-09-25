# Post-Quantum Hybrid Key Exchange (PQHKE) support in Strimzi

This proposal describes how Strimzi enables Post-Quantum Hybrid Key Exchange (PQHKE) using ML-KEM (via the `X25519MLKEM768` named group) on all TLS connections within a Strimzi-managed Kafka cluster, and how named group configuration for external listeners will be exposed, once the required upstream Apache Kafka work lands, as future work.

This proposal covers key exchange only by using [ML-KEM (FIPS 203)](https://csrc.nist.gov/pubs/fips/203/final) via [JEP-527 ("Hybrid Post-Quantum Key Exchange in TLS 1.3")](https://openjdk.org/jeps/527).
[ML-DSA (FIPS 204)](https://csrc.nist.gov/pubs/fips/204/final) certificate signing via [JEP-497](https://openjdk.org/jeps/497) is a separate, more invasive effort and is a non-goal of this proposal.
It will be tracked in a dedicated follow-up proposal once hybrid key exchange is in place.

## Current situation

Every TLS connection in a Strimzi-managed Kafka cluster (broker-to-broker, controller-to-controller, operator-to-broker, operator-to-Cruise-Control, operator-to-Kubernetes, Connect/MM2/Bridge-to-broker, Kafka Exporter-to-broker) uses classical key exchange algorithms (`ECDH`, `X25519`).

The JDK container images Strimzi ships are based on Java 21 and do not yet include JEP-527, which is the Java standard that wires ML-KEM into TLS 1.3 as the `X25519MLKEM768` named group.

Furthermore, Strimzi currently sets no `ssl.enabled.protocols` on the control plane (port 9090) or replication (port 9091) internal listeners, leaving the Kafka broker default (`TLSv1.2,TLSv1.3`), which means TLS 1.2 connections to these listeners are accepted even though all clients that connect to them support TLS 1.3.
Of course, the same applies for external listeners.

## Motivation

Quantum computers will eventually break classical key exchange algorithms.
The threat already exists as attackers can record encrypted Kafka traffic today and decrypt it later ("Harvest Now, Decrypt Later", HNDL).
Organizations in regulated industries are already being required to begin migration.

NIST standardised ML-KEM (FIPS 203) as the Post-Quantum key encapsulation mechanism for TLS.
The hybrid approach (`X25519MLKEM768`) combines classical `X25519` and `ML-KEM` in the same handshake, so the session key is secure as long as either algorithm holds.
Non-PQC peers fall back to classical key encapsulation transparently, with no connection failure and no configuration needed on non-PQC clients.

The cloud-native ecosystem around Strimzi is already moving in this direction.
Kubernetes 1.33 (built with Go 1.24) negotiates `X25519MLKEM768` by default on the API server side.
Go 1.24+ enables `X25519MLKEM768` by default in TLS 1.3 client connections, which means the Kafka Exporter already supports it on the client side (it's currently using Go 1.27).

## Proposal

Supporting PQHKE within a Strimzi-managed Kafka cluster requires two independent but complementary changes:

* updating container images to a JDK that includes JEP-527.
* forcing TLS 1.3 on the internal Kafka listeners.

### Update container images to Java 25 (October 2026 Critical Patch Update)

JEP-527 is the Java standard that integrates ML-KEM into TLS 1.3 as `X25519MLKEM768`.
It landed in Java 27 (September 2026) and is being backported to Java 25 LTS via the October 2026 Critical Patch Update (CPU).

Java 24 added ML-KEM as a cryptographic primitive (JEP 496) but did not integrate it into TLS.
`X25519MLKEM768` is only available as a TLS named group starting with Java 25 (post-October 2026 CPU) or Java 27.
For more details about the backport of JEP-527 to Java 25 LTS look at OpenJDK 25.0.5 release [here](https://wiki.openjdk.org/spaces/JDKUpdates/pages/170131468/JDK+25u).

Strimzi should not wait for Java 27 to be the baseline because it's not an LTS release and also because Java 25 LTS will receive JEP-527 via the October 2026 CPU and is already the supported LTS at the time of writing.
All JVM-based Strimzi container images (operator, Kafka, bridge, and so on) would be based on Java 25 LTS (post-October 2026 CPU).

With JEP-527 in the JDK, `x25519mlkem768` is added to the JVM's default named groups automatically and it's at the top of the list.
No `jdk.tls.namedGroups` configuration is needed at JVM level.
When both peers support hybrid groups, `X25519MLKEM768` is negotiated; when a peer only supports classical groups, the handshake falls back transparently.

This is not a code change in the operator itself, but it is a hard prerequisite: without JEP-527 in the JDK, no PQHKE is possible on any Java-based connection regardless of any other configuration.

### Force TLS 1.3 on internal Kafka listeners

Strimzi should add per-listener TLS 1.3 enforcement for the two internal-only listeners when setting un the nodes configuration within the `KafkaBrokerConfigurationBuilder` class:

```shell
listener.name.controlplane-9090.ssl.enabled.protocols=TLSv1.3
listener.name.replication-9091.ssl.enabled.protocols=TLSv1.3
```

Ports 9090 (control plane) and 9091 (replication) are internal-only listeners, not reachable by user Kafka clients.
Every component that connects to them (broker-to-broker ReplicaFetcher, operator Admin clients, Cruise Control Kafka clients, Kafka Exporter) already supports TLS 1.3.

It is worth noting that with both sides running Java 25 (post-October 2026 CPU), TLS 1.3 would already be negotiated by mutual preference even without this change: TLS negotiation picks the highest version both sides support, and both the Kafka broker and every internal Kafka client default to `ssl.enabled.protocols=TLSv1.2,TLSv1.3` with `ssl.protocol=TLSv1.3`, so TLS 1.3 is the preferred version.
With JEP-527 in place, `X25519MLKEM768` would therefore be negotiated automatically on those connections as well.

However, relying on negotiation preference alone leaves the security guarantee implicit rather than explicit.
Forcing `ssl.enabled.protocols=TLSv1.3` on ports 9090 and 9091 makes TLS 1.3 a server-enforced policy on those listeners, independently of what any client advertises.
Since all internal clients already support TLS 1.3, this is a safe hardening change with no backward compatibility concerns, and it makes the PQHKE guarantee auditable at the configuration level rather than inferred from runtime negotiation behavior.

With TLS 1.3 enforced at the server level, ML-KEM is guaranteed on all broker-to-broker and operator-to-broker connections once the JDK baseline is met.

External user-facing listeners are left untouched.
They continue to offer TLS 1.2 and TLS 1.3 to support Kafka clients that may not yet support TLS 1.3 or hybrid key exchange.
When both the broker and the client run Java 25, TLS 1.3 is negotiated by preference and ML-KEM key exchange happens automatically, with no configuration needed.

#### What works automatically after these two changes

| Connection | Port | TLS 1.3 | ML-KEM |
|---|---|---|---|
| Broker to broker (replication) | 9091 | Forced by listener config | Automatic (JEP-527 in broker JVM) |
| Controller to controller / broker | 9090 | Forced by listener config | Automatic (JEP-527 in broker JVM) |
| CO / TO / UO Admin client to broker | 9091 | Negotiated by preference (client `ssl.protocol=TLSv1.3` default) | Automatic (JEP-527 in operator JVM) |
| CO to Kafka Agent | 8443 | Hardcoded in `KafkaAgentClient` | Automatic (JEP-527 in broker JVM) |
| CO / TO to Cruise Control HTTP server | 9090 | Hardcoded in operator clients | Automatic (JEP-527 in Cruise Control JVM) |
| Cruise Control Kafka clients to broker | 9091 | Forced by listener config | Automatic (JEP-527 in Cruise Control JVM) |
| CO / TO / UO to Kubernetes API | 443 | Negotiated by preference | Automatic (K8s 1.33+ Go 1.24, JEP-527 in operator JVM) |
| Connect / MM2 / Bridge to broker | External | Negotiated by preference | Automatic (JEP-527 on both JVMs) |
| Kafka Exporter to broker | 9091 | Forced by listener config | Automatic (Go 1.24+ default in crypto/tls) |

## Future: named groups configuration for external listeners (pending KIP-1376)

The default hybrid behavior from JEP-527 (offering `X25519MLKEM768` alongside classical groups with transparent fallback) covers the common case.

For external listeners, three client scenarios are possible:

* a client connecting via TLS 1.2, able to negotiate only classical algorithms (e.g. `X25519`).
* a client connecting via TLS 1.3 but without ML-KEM support (e.g. a Java client on a JDK version without JEP-527, or a non-Java client that does not support ML-KEM), able to negotiate only classical algorithms as fallback.
* a client connecting via TLS 1.3 and supporting ML-KEM, using it as the key exchange mechanism.

This means that by updating to Java 25 LTS alone, external listeners already gain PQHKE support for capable clients, while gracefully falling back to classical key exchange for clients that do not yet support it.

However, some users need explicit control over which named groups are offered on a given listener:

* PQC-only enforcement: remove classical groups so the TLS handshake fails for peers that do not support any hybrid group. 
This ensures that no classical key exchange can occur even as a fallback, which may be required by compliance mandates in their final migration phase.
* Classical-only restriction: remove hybrid groups to prevent ML-KEM from being negotiated, for example during a testing or validation phase where PQC traffic needs to be isolated.
* Preference reordering: prefer a different hybrid group over the default `X25519MLKEM768` (for example `secp384r1mlkem1024` for higher security level), requiring the list to be reordered explicitly.

The Kafka client library currently has no per-client or per-listener configuration property for named groups equivalent to `ssl.enabled.protocols` or `ssl.cipher.suites`.
Named group configuration is only possible JVM-wide via `jdk.tls.namedGroups`, which affects all connections in the same JVM simultaneously and is therefore not suitable as a per-listener knob.

[KIP-1376](https://cwiki.apache.org/confluence/spaces/KAFKA/pages/451974516/KIP-1376+Support+setting+TLS+named+groups) 
proposes a new `ssl.named.groups` configuration property in Kafka with support for per-listener overrides.
Once KIP-1376 is implemented and available in a supported Kafka version, Strimzi will expose it via a new `spec.kafka.listeners[*].tls.namedGroups` field, allowing users to configure named groups per listener without relying on the JVM-wide property.
Exposing this field will require a dedicated Strimzi proposal at that time.

Until KIP-1376 lands, there is no underlying Kafka config property for a CRD field to set.
Users who need to override named groups JVM-wide in the interim can use `jvmOptions.javaSystemProperties` to set `jdk.tls.namedGroups`, but this is not recommended: it affects all connections in the broker JVM simultaneously, including internal listeners and other components sharing the same JVM.

## Affected/not affected projects

The cluster operator is affected by configuring TLS 1.3 on internal listeners.
The Docker base images images are updated to Java 25 TLS (post-October 2026 CPU).

## Compatibility

The `ssl.enabled.protocols=TLSv1.3` addition to the internal listeners has no backward compatibility risk because every component connecting to ports 9090 (control plane) and 9091 (replication) already supports TLS 1.3.
External user-facing listeners are unchanged.

PQHKE works as a hybrid: when both peers support `X25519MLKEM768`, they use it; when either side does not (for example, a component still running on Java 21), the handshake falls back to classical key exchange transparently.
No connection is broken.

Updating the container images to Java 25 (post-October 2026 CPU) is sufficient for PQHKE to become active on all connections, along with the listener config change.
Components running on Java 21 will continue to work using classical key exchange until their images are updated.

## Rejected alternatives

### Stay on Java 21

Staying on Java 21 was rejected because the JEP-527 backport to Java 21 LTS is not expected until 2027 H1 per the [Oracle JRE and JDK Cryptographic Roadmap](https://www.java.com/en/jre-jdk-cryptoroadmap.html).
Waiting for that backport would delay PQHKE by at least a year while organizations in regulated industries are already under pressure to begin migration.

### Wait for next Java LTS after 27 as the baseline

Waiting for the next LTS after Java 27, which will be Java 29 planned for September 2027, was also rejected.
Java 25 LTS already receives JEP-527 via the October 2026 Critical Patch Update and is the supported LTS at the time of writing.
Delaying until a Java LTS after Java 27 becomes the Strimzi baseline would unnecessarily block PQHKE for those same organizations with no technical justification.
