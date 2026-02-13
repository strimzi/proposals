# TLS support in test-container

This proposal adds TLS support to the Strimzi [test-container](https://github.com/strimzi/test-container), enabling users to test Kafka client applications over encrypted connections.

## Current situation

The Strimzi test-container library allows running Kafka clusters in containers for integration testing.
Currently, all communication (client, inter-broker, controller) uses plaintext.
There is no mechanism for generating or managing TLS certificates within the test-container.

## Motivation

Projects like [test-clients](https://github.com/strimzi/test-clients) or [http-bridge](https://github.com/strimzi/strimzi-kafka-bridge) support TLS to Kafka, but there is no way to verify that within their own repositories.
TLS is only tested in [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator) system tests, which means we have to release first and hope it works.

Adding TLS support to `test-container` lets us catch TLS issues early, directly in the project, without waiting for the full operator release cycle.

## Proposal

TLS encryption is applied to **all listeners** (i.e., client, inter-broker, and controller) to more closely resemble production deployments.
Each listener type gets its own keystore so that internal (i.e., inter-broker and controller) and external (i.e., client) trust domains are separated.

Certificates are generated inside the Docker container using `keytool`, which is always available since the container ships with a JDK.
This avoids external tool dependencies (no `openssl` or Bouncy Castle required) and keeps the implementation self-contained.

Two self-signed CAs form separate trust domains:
1. **Cluster CA**, which signs the broker certificate (client-facing listener) and the internal certificate (inter-broker and controller listeners). 
Both include SANs for `localhost`, `*.localhost`, `127.0.0.1`, and all broker network aliases (`broker-0` through `broker-{n-1}`).
2. **Clients CA**, which signs the client certificate (provided to test code).

> [!NOTE]
> Because the SANs include the broker network aliases (`broker-0`, `broker-1`, ...), the advertised listeners use these DNS names instead of dynamic container IPs. 
> This ensures hostname verification passes without needing to set `ssl.endpoint.identification.algorithm` to an empty string.

The broker truststore contains the clients CA (to verify client certs during TLS), while the client truststore contains the cluster CA (to verify the broker's identity).
Each component (broker, internal, client) uses its own independently generated password. 
All stores use PKCS12 format.

> [!NOTE] 
> All certificates are generated via `keytool` inside the container.

All brokers must share the same stores so the chain of trust is consistent.
Certs are generated on one broker and distributed to all others.

### API changes

```java
//  enable auto-generated CA-signed certificates
public StrimziKafkaClusterBuilder withTls() { }

// StrimziKafkaCluster expose client stores for
public boolean isTlsEnabled() { }
public byte[] getClientTrustStoreBytes() { }   // client truststore (contains CA cert)
public byte[] getClientKeyStoreBytes() { }    // client keystore
```

## Affected projects

This proposal affects the Strimzi [test-container](https://github.com/strimzi/test-container) project only.

## Compatibility

This is a purely additive change.
There is no impact on backwards compatibility.

## Rejected alternatives

### Certificate generation tools

#### Bouncy Castle

Adds an external dependency to the project and increases the maintenance surface.
Since `keytool` is already available inside the container's JDK, the same result can be achieved with no additional dependencies.

#### OpenSSL with ProcessBuilder

Requires `openssl` to be installed on the host machine, cross-platform issues.

#### Pure Java API (sun.security.x509)

JDK does not expose public APIs for X.509 certificate generation with SANs.
The internal `sun.security.x509` classes are encapsulated in modern JDKs and fragile across versions.

### Certificate design

#### Self-signed certificates (no CA)

I tried using a single self-signed broker certificate is the simplest approach but does not reflect production deployments.
There is no chain of trust, no separation between client and internal domains, and no ability to do TLS with a distinct client identity.

#### Single CA for all certificates

Another approach using one CA to sign both broker and client certificates is simpler than two CAs but collapses the trust domains.
Any certificate signed by the single CA would be trusted everywhere (i.e., basically a compromised client cert could impersonate a broker).
Separate CAs (cluster CA and clients CA) mirrors more production-like deployment and keep the trust boundaries distinct.
