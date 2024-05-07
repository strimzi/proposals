# Enhance Kafka Spec with cert algorithm management

Allow the end user to manage cert configs, used to generate servers and users certificates, via the `Kafka` CRD.

## Current situation

Properties are not yet supported in the Strimzi Kafka CRD.

## Motivation

Requested by the community here: https://github.com/strimzi/strimzi-kafka-operator/issues/9372

## Proposal

The proposal will introduce cert configs for the cluster and client CA, this is so that it is possible to use other algorithms than the currently hardcoded RSA. The proposal will add three new attributes to the `clusterCa` and `clientsCa` specs:

* `keyAlgorithm`: The algorithm for generating the private key.
* `keySize`: The size of the private key generated.
* `signatureAlgorithm`: The hashing algorithm used for signing.

Suggestion:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-kafka-cluster
spec:
  # ...
  clusterCa:
    keyAlgorithm: rsa
    keySize: 4096
    signatureAlgorithm: SHA256
  clientsCa:
    keyAlgorithm: ecdsa
    keySize: 521
    signatureAlgorithm: ecdsa-with-SHA512
  # ...
```

## Affected/not affected projects

No other projects affected than the Strimzi Operator.

## Compatibility

The three new attributes should default to the current defaults, to make sure that there are no compatibility issues. `keyAlgorithm` should default to rsa, `keySize` to 4096 and the `signatureAlgorithm` to sha256.

## Rejected alternatives

No rejected previous alternatives. 