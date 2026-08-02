# Strimzi-native Configurable Certificate Key Strength

This proposal adds a `keySize` property to the `CertificateAuthority` configuration in the `Kafka` custom resource allowing users to configure the RSA key size used by Strimzi's internal CAs for both the CA keys themselves and the certificate keys they issue.
For differentiating between CA certificates and certificates **issued** by a CA, the proposal uses the following terms: 
- **Root certificates**: The CA certificates.
- **Leaf certificates**: The certificates **issued** by a CA, also known as end-entity (EE) certificates, the certificates used to identify the Kafka brokers, Strimzi Operators, Cruise Control, and KafkaUser components.

## Current situation

Strimzi's internal certificate issuer generates RSA keys with hardcoded sizes that are not user-configurable:

* **Root keys**: The cluster CA and clients CA keys are generated with a **4096-bit** RSA key, hardcoded as a string literal `"4096"` passed to `openssl genrsa` in [`OpenSslCertIssuer.generateCaCert()`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/certificate-issuer/src/main/java/io/strimzi/certs/OpenSslCertIssuer.java#L269).
* **Leaf keys**: The leaf keys used by Kafka brokers, Strimzi Operators, Cruise Control, and Kafka users are generated 
with a **2048-bit** RSA key which is [OpenSSL's default](https://docs.openssl.org/4.0/man1/openssl-genpkey/?dsa-parameter-generation-options#rsa-key-generation-options) when no explicit key size is specified for RSA key generation. 
These keys are generated when [`OpenSslCertIssuer.generateCsr()`](https://github.com/strimzi/strimzi-kafka-operator/blob/main/certificate-issuer/src/main/java/io/strimzi/certs/OpenSslCertIssuer.java#L474-L477) calls `openssl req -new` without specifying an explicit key size.

The [`CertificateAuthority` API model](https://strimzi.io/docs/operators/latest/configuring#type-CertificateAuthority-reference) currently exposes the following configuration:
* `generateCertificateAuthority` (boolean)
* `generateSecretOwnerReference` (boolean)
* `validityDays` (integer)
* `renewalDays` (integer)
* `certificateExpirationPolicy` (enum)

There is no property to control key sizes.
Users who require a specific key size for compliance can disable Strimzi's built-in CA entirely (`generateCertificateAuthority: false`) and manage certificates externally or use the cert-manager integration described in [Strimzi proposal 100: "External Certificate Manager"](100-external-certificate-manager.md) once it is implemented.
However, in both cases the leaf certificate key size is set to 2048 bits with no way to configure it so neither workaround fully addresses compliance requirements for all keys.

## Motivation

Multiple national cybersecurity authorities have issued guidance requiring RSA key sizes larger than 2048 bits:

| Authority                        | Guidance       | Requirement                             |
|----------------------------------|----------------|-----------------------------------------|
| BSI TR-02102-1 (Germany) [1]     | Since 2023     | ≥3000-bit RSA keys                      |
| ANSSI RGS B1 (France) [2]        | From 2030      | ≥3072-bit RSA keys                      |
| NIST SP 800-57 (US) [3]          | From 2031      | ≥3072-bit RSA keys                      |
| eIDAS / ETSI TS 119 312 (EU) [4] | Since Dec 2025 | ≥3000-bit RSA keys for TLS              |
| ASD ISM (Australia) [5]          | Since 2024     | ≥2048-bit (≥3072-bit for classified)    |

Similar requirements exist in CA/Browser Forum Baseline Requirements [6], NSA CNSA 1.0 [7], and Spain's CCN-STIC-221 [8].

[1]: https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/TechGuidelines/TG02102/BSI-TR-02102-1.pdf?__blob=publicationFile&v=14
[2]: https://messervices.cyber.gouv.fr/documents-guides/security-recommendations-for-tls_v1.1.pdf
[3]: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf
[4]: https://ec.europa.eu/digital-building-blocks/sites/download/attachments/467109280/eIDAS%20Cryptographic%20Requirement%20v.1.4.1_final.pdf
[5]: https://www.cyber.gov.au/sites/default/files/2026-06/22.%20ISM%20-%20Guidelines%20for%20cryptography%20%28June%202026%29.pdf
[6]: https://www.digicert.com/blog/code-signing-baseline-requirements-to-require-larger-keys
[7]: https://media.defense.gov/2025/May/30/2003728741/-1/-1/0/CSA_CNSA_2.0_ALGORITHMS.PDF
[8]: https://www.aoc.cat/en/blog/2025/nova-mida-claus-rsa-3072/

Organizations in regulated environments like banking, healthcare, EU/DE public sector, and defense-adjacent industries cannot configure the key size of Strimzi's generated certificates to meet cryptographic standards. 
As shown above, BSI (Germany) and eIDAS (EU) already require RSA keys of at least 3000 bits, making Strimzi's 2048-bit default non-compliant today. 
The remaining authorities mentioned above: ANSSI, NIST, and ASD ISM will follow by 2030–2031 when 2048-bit RSA keys are prohibited.

As described above, the existing workarounds do not fully address compliance requirements for all keys and both introduce significant operational complexity.

Adding a configurable key size is a minimal, backward-compatible change that allows these users to remain on Strimzi's built-in CA while meeting their compliance requirements.
  
## Proposal

### API changes

A new `keySize` property will be added to the `spec.clusterCa` and `spec.clientsCa` sections of the `Kafka` custom resource:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
spec:
  clusterCa:
    keySize: 3072
    validityDays: 365
    renewalDays: 30
  clientsCa:
    keySize: 3072
    validityDays: 365
    renewalDays: 30
  # ...
```

The `keySize` property controls the RSA key size in bits for **all** keys generated by Strimzi, both the root certificates and the leaf certificates.

This follows the same pattern as `validityDays` which also applies to all certificates managed under the CA.

#### Defaults and validation

* **Default value:** `3072`, which meets all current and upcoming regulatory requirements (≥3000-bit RSA keys).
  This changes the effective key size for root keys from `4096` (Strimzi's current hardcoded value) to `3072`.
  This will change the effective key size for leaf certificates from `2048` bits, OpenSSL's implicit default, to `3072` bits.
  This is a deliberate change: relying on an implicit OpenSSL default is fragile and aligning leaf certificate keys with the root certificate key size is a safer baseline.
  For users who need the previous 2048-bit leaf key size, they can set `keySize: 2048` explicitly.

* **Minimum value:** `2048` since smaller RSA keys do not meet any current security standard and therefore should not be supported.

* **Recommended key sizes:** Common RSA key sizes and their security strengths are documented in [NIST SP 800-57 Part 1 Rev. 5, Table 2](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf).
  A table with the common key sizes will be added to the Strimzi documentation as a guideline for users selecting a key size.

#### Interaction with `generateCertificateAuthority: false`

When `generateCertificateAuthority` is set to `false` and the user provides their own CA certificate and key.
In this case, `keySize` only affects the leaf certificate keys that Strimzi generates.
The `keySize` does not affect the root certificate keys themselves since root certificate keys are provided by the user.

#### Interaction with cert-manager integration (proposal 100)

When the `spec.clusterCa.type` and/or `spec.clientsCa.type` is set to `cert-manager.io` in the `Kafka` custom resource (as described in [Strimzi proposal 100: "External Certificate Manager"](100-external-certificate-manager.md)), Strimzi delegates certificate issuance and private key generation to cert-manager.
In this case, `keySize` only affects the leaf certificate keys that cert-manager generates.
Strimzi will use the value of `keySize` to populate the `spec.privateKey.size` field in the `Certificate` resource it creates.
Then, cert-manager will read that `Certificate` resource to configure and generate the leaf certificate keys.
The `keySize` field does not affect the root certificate keys since those are configured by the user through the `Certificate` resources that the user creates, which are outside Strimzi's control.

### Implementation

Following the existing pattern established by `validityDays` we would add a `keySize` field to `CertificateAuthority.java` like this:

```java
public class CertificateAuthority implements UnknownPropertyPreserving {
    //... existing fields
    public static final int DEFAULT_KEY_SIZE = 3072;

    // ... existing fields 
    private int keySize;

    @Description("The RSA key size in bits for CA and end-entity certificate keys. " +
            "Must be at least 2048." +
            "Default is 3072.")
    @Minimum(2048)
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    public int getKeySize() {
        return keySize;
    }

    public void setKeySize(int keySize) {
        this.keySize = keySize;
    }
    // ... existing methods 
}
```

#### User Operator considerations

The `keySize` for the clients CA must be forwarded to the User Operator so it can generate `KafkaUser` certificates with the correct key size.
This will be done by adding a new environment variable `STRIMZI_CLIENTS_CA_KEY_SIZE` to the Entity Operator deployment set by the Cluster Operator based on the `spec.clientsCa.keySize` value in the `Kafka` resource.

### Behavior on key size change

When a user changes the `keySize` on an existing cluster, the behavior depends on the `certificateExpirationPolicy`:

* **No immediate key regeneration.** Changing `keySize` alone does not trigger an immediate root key replacement or leaf certificate regeneration.
  The new key size takes effect the next time a key is generated during root key replacement (triggered by `certificateExpirationPolicy: replace-key` at renewal time or a manual force-replace annotation) or when new leaf certificates are issued (e.g. scaling up, adding a new `KafkaUser`).
* **Existing certificates remain valid.** Certificates generated with the previous key size continue to function until they are naturally renewed or replaced.
  This avoids unnecessary rolling restarts.
* **Gradual rollout.** To immediately apply a new key size to all certificates, the user should:
  1. Set the desired `keySize` in the `Kafka` resource.
  2. Trigger a root key replacement using the `strimzi.io/force-replace` annotation on the root key Secret.
  This will generate a new root key with the configured size and trigger re-issuance of all leaf certificates followed by rolling restarts.
* **Upgrade behavior.** This same behavior applies when upgrading to a Strimzi version that introduces the `keySize` property.
The new default key size does not trigger immediate regeneration of existing keys, it only takes effect when keys are next generated during renewal, replacement, or new certificate issuance.
* **Downgrade behavior.** When downgrading to a Strimzi version without the `keySize` property, existing keys of any size remain in the Secrets and continue to function.
On renewal or replacement, the older version generates new keys at its hardcoded sizes (4096-bit for root keys, 2048-bit for leaf keys).

### Trade-offs

* **Performance vs. security:** Larger key sizes (e.g. 4096-bit) strengthen cryptographic protection but make every TLS handshake slower which could matter for a Kafka cluster handling thousands of connections. 
Exposing keySize as a configurable field lets users choose the right balance for their environment.

## Affected/not affected projects

**Affected:**
* `api/` — `CertificateAuthority` model, CRD schema
* `certificate-issuer/` — `CertIssuer` interface, `OpenSslCertIssuer`
* `operator-common/` — `Ca`, `InternalCa`, `CaConfig`
* `cluster-operator/` — `CaReconciler`
* `user-operator/` — `KafkaUserModel` (reads key size from environment variable)
* `documentation/` — API reference, deploying guide (certificate configuration section)

## Compatibility

This change is **backward compatible**:

* The `keySize` property defaults to `3072` when not set.
* For root keys, the effective default changes from `4096` (Strimzi hardcoded) to `3072` (Strimzi configurable default).
  Users who need to preserve 4096-bit root keys can set `keySize: 4096` explicitly.
* For leaf keys, the effective default changes from `2048` (OpenSSL implicit) to `3072` (Strimzi explicit).
  Users who need to preserve 2048-bit leaf keys can set `keySize: 2048` explicitly.
  These are behavioral changes, but they are in the direction of stronger security for leaf keys and only affect newly generated keys, existing certificates are not regenerated.
* No existing `Kafka` custom resources need to be modified.
* No rolling restarts are triggered by the introduction of this feature alone.

## Rejected alternatives

### Separate key sizes for root and leaf certificates

Having two properties (e.g., `caKeySize` and `certKeySize`) was considered to allow independent control over root and leaf key sizes.
This was rejected because:
* Organizations subject to key size requirements typically need the same minimum across all keys.
* A single property is simpler to understand, configure, and implement and reduces code complexity.
* The interaction with `generateCertificateAuthority: false` where the root key size is user-controlled already provides the ability to have different sizes for root vs leaf keys.
* More granular control can be added in a future proposal if there is demand without breaking backward compatibility.