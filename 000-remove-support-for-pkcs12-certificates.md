# Remove support for PKCS12 certificates

The support for generating PKCS12 format certificates in the CA and KafkaUser Secrets will be removed from Strimzi.

## Current situation

Strimzi uses PEM format certificates for all internal communications.
The only exception is Cruise Control which has an [open issue](https://github.com/strimzi/strimzi-kafka-operator/issues/12606) to move away from PKCS12.
As a community we have also [pushed back](https://github.com/strimzi/kafka-access-operator/issues/81) on requests to add PKCS12 support to new components such as Access Operator.

In the CA and KafkaUser Secret resources, Strimzi generates an additional PKCS12 file and related password. 
This is only generated if the environment variable `STRIMZI_PKCS12_KEYSTORE_GENERATION` on the cluster operator is set to true, which is currently the default value.

Based on some discussion during a community call in the [1.3.0 release](https://github.com/strimzi/strimzi-kafka-operator/releases/tag/1.3.0) the support for PKCS12 has been declared as deprecated.
However, the deprecation does not specify when the env var will be set to false, and when/if it will be removed.

## Motivation

Java-based Apache Kafka clients have supported using PEM certificates directly for some time, and as a result Strimzi has been moving away from PKCS12 format certificates. 
To this end [proposal 141](141-make-PKCS12-stores-in-CA-and-user-secrets-optional.md) introduced an env var to turn off the generation of PKCS12 certificates, and based on the discussion 
on the [community call on September 13th 2026](https://youtu.be/eyZpjVnTzQI?si=UmZMn_T4Or9LX1_1&t=188) there seems to be consensus that it is the right time to fully remove support.

## Proposal

The env var `STRIMZI_PKCS12_KEYSTORE_GENERATION` will be set to false by default in the 1.5.0 release of Strimzi. 
This is two releases after it was announced as deprecated.

The env var and any related code will be removed two releases later, in the 1.7.0 release.

## Affected/not affected projects

This affects the cluster operator and the user operator.

The Cruise Control component is the only one that hasn't been fully moved to use PEM files. 
However, the generation of the PKCS12 files is done in a script in the container on startup. 
This means we can safely remove any PKCS12 code from the cluster-operator certificate generation without affecting Cruise Control's use of PKCS12 certificates.

## Compatibility

These are breaking changes since files that users might be relying on will be removed from the relevant Secrets.

## Rejected alternatives

### Keep the env var as is

Leaving the default value as true means that PKCS12 files are being issued even if they are not used. 
This is unnecessary since most users are expected to have moved to PEM format certificates at this time.

### Leave the env var indefinitely

Rather than removing the env var, we could change the default to false and leave it there indefinitely. 
This would allow users to still make use of PKCS12 if they wish to, however it means the added overhead of generating and testing the code paths. 
It is unclear how many users are still using PKCS12 and cannot move to PEM, so we need more confirmation that this is needed to adopt this approach.
