# Make PKCS12 stores in CA and User `Secrets` optional

This proposal suggests adding a new configuration option to disable automatic generation of PKCS12 certificate stores in CAs and `KafkaUser` `Secrets`.

## Current situation

Strimzi currently uses PKCS12 certificate stores in multiple different places.
Some of them are user-facing and might be used by users:
* Cluster and Client CA certificate Secrets have PKCS12-based trust stores
* Secrets of `KafkaUser` resources with `type: tls` authentication generate a PKCS12 key store with a user certificate

Others are internal only:
* Use of PKCS12 stores in MirrorMaker 2 connectors configuration
* Use of PKCS12 stores in Cruise Control configuration
* Use of PKCS12 stores in Strimzi HTTP Bridge

The PKCS12 stores that are used only internally are being slowly phased out.
This was already done in the past for Kafka, Kafka Connect, and the Topic Operator.
And we have tracking issues for the remaining work (see [strimzi-kafka-operator#11294](https://github.com/strimzi/strimzi-kafka-operator/issues/11294) and its subtasks).

This proposal focuses only on the user-facing uses of PKCS12 stores.
It does not cover the internal ones, which should be removed independently of this proposal over time.

## History

The PKCS12 stores were originally included in our CA and user `Secrets` mainly because:
* PEM files were not supported by Java-based Apache Kafka clients
* PKCS12 stores were supported by Java-based Apache Kafka clients
* Unlike Java's native JKS stores, PKCS12 stores are easy to generate and work with using non-Java tools such as OpenSSL

Especially the first limitation does not exist anymore today.
Java-based Apache Kafka clients can now use PEM certificates directly.
So the need for the PKCS12 stores decreased significantly.
However, they might still be used by some of our users.

## Motivation

The PKCS12 stores make it hard to use Strimzi with custom base container images with various Java distributions and configurations.
For example, Java distributions focusing on [FIPS compliance](https://en.wikipedia.org/wiki/Federal_Information_Processing_Standards) that are based on the Bouncy Castle library do not support PKCS12 stores.
One popular example of such Java distributions are the [Chainguard Java FIPS container images](https://images.chainguard.dev/directory/image/jdk-fips/overview).
Right now, when you try to use Strimzi operators on top of this container image, they will fail while trying to generate the PKCS12 file (either in the Cluster Operator when creating/managing the Cluster and Client CAs, or in the User Operator when creating/managing the user certificates).

This proposal suggests working around this issue by adding a flag to disable PKCS12 stores in CA and user `Secrets`.
This is similar to what we already have for Network Policies or Pod Disruption Budgets.

## Proposal

A new environment variable named `STRIMZI_PKCS12_KEYSTORE_GENERATION` will be introduced to Cluster Operator and User Operator configurations.
It will default to `true` and in the default state the operators will continue to create and manage the PKCS12 stores in the CA and user `Secrets`.
Only when set to `false` by the user will the Cluster and User Operators skip PKCS12 generation, and the CA and user `Secrets` will be used without PKCS12 stores and their passwords.

The Cluster Operator will automatically propagate the `STRIMZI_PKCS12_KEYSTORE_GENERATION` configuration to the User Operator deployed as part of a Kafka cluster.
So when `STRIMZI_PKCS12_KEYSTORE_GENERATION` is set to `false` by the user in the Cluster Operator, it will automatically be set to `false` in the User Operator as well.

## Out of scope

The internal uses of PKCS12 stores in MirrorMaker 2 connectors, Strimzi HTTP Bridge, and Cruise Control will not be affected by this proposal and the new configuration option it introduces.

## Affected projects

This proposal affects the Strimzi Cluster and User Operators.
In terms of our Java code, it also affects the `operator-common` module and its `Ca` and `ClientsCa` classes and their tests.

## Backwards compatibility

This proposal is fully backwards compatible.
The newly introduced configuration option defaults to the current state.
So users will not experience any difference unless they actually use the new option.

## Rejected alternatives

### Phase-out use of PKCS12 stores completely using a feature gate

Originally, my preferred solution was to use a feature gate and completely phase out support for PKCS12 stores in CA and user Secrets over multiple Strimzi releases.
As Java clients can now use PEM files directly, the PKCS12 stores are not needed anymore and from my experience are not widely used.
However, further [discussions](https://cloud-native.slack.com/archives/C018247K8T0/p1776255709426059) revealed different preferences by different maintainers, so I decided to reject this alternative and propose a less disruptive solution based on a new configuration flag to optionally disable the PKCS12 store generation.
If we decide in the future that we should phase out the PKCS12 stores completely, we can always propose and implement it at a later point.

### Auto-detecting PKCS12 support

Another alternative that was considered was to automatically detect the support for the PKCS12 stores and skip them only when they would not be supported.
However, this alternative was rejected because some users might want to skip the PKCS12 stores for security reasons even when their Java distribution supports PKCS12 stores.
