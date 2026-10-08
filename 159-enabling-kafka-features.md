# Adding support for enabling Kafka features in Strimzi

This proposal suggests adding support for managing Kafka feature flags through the Strimzi Kafka CR.

## Current situation

Every time there is a new Kafka feature that has a significant impact on behavior, it is gated behind a feature flag.
These features can be enabled using the Kafka Admin client or the `kafka-features.sh` script that Kafka provides.
We have encountered features like this in the past — for example KRaft mode, where we were able to enable it based on a Strimzi feature gate.
Another example is Share Groups (Kafka Queues).
Some of the previous features are now enabled by default, but there will be more in the future.

Today, there is no way to enable these features through the Strimzi Kafka resource — users have to use the `kafka-features.sh` script or their own Kafka Admin client implementation to do so.

## Motivation

The main motivation is to provide users with a convenient way to enable these features — such as Share Groups — without needing to run a script from a Pod or locally.
Users should be able to enable a desired feature using the Kafka CR.
Additionally, it should be visible which features are enabled and at what versions, again without needing to run scripts.

## Proposal

This proposal suggests adding new fields for managing Kafka features to our API.
The new fields will be added to the Kafka CR, under `.spec.kafka`:

```yaml
spec:
  # ...
  kafka:
    features:
      - name: share.version
        version: 1
```

The `features` field takes a list of features that the user wants to upgrade, downgrade, or enable.
There will be a list of forbidden features that cannot be configured using this field — for example `kraft.version` or `metadata.version`.
`metadata.version` should be configured using the `metadataVersion` field in the Kafka spec.

A validation method in `KafkaCluster.fromCrd()` will check for duplicates and usage of forbidden features.
In both cases, an `InvalidResourceException` will be thrown — either indicating that a forbidden feature is configured or that there are duplicate entries for the same feature.

If the list of features is valid, the operator will use a new class named `KafkaFeatureManager` to configure them.
Today, we are already using the Admin client's features endpoint for changing `metadata.version` — this is handled inside `KRaftMetadataManager`.
The `KafkaFeatureManager` will extract the shared feature update logic from `KRaftMetadataManager`.
From then on, `KRaftMetadataManager` will use `KafkaFeatureManager` for changing `metadata.version`, instead of handling it directly, avoiding code duplication.

Before processing individual features, the `KafkaFeatureManager` will call `describeFeatures()` once to get both the supported and finalized features from Kafka.
The supported features will be used to verify that the features specified by the user actually exist in the running Kafka version.
If a feature from the user's list is not present in the supported features, it will be skipped and a warning will be stored in the Kafka's `.status` section with `KafkaFeatureUnsupported` reason.
The finalized features from the same call will be used to determine the current version of each feature.

For each valid feature in the list, the `KafkaFeatureManager` will go through the following stages:

* Using the finalized features from the initial `describeFeatures()` call, it will determine the current version of the feature (features not present in the finalized map are treated as version 0 — disabled).
  * If the feature is not enabled or has a lower version than the desired version, it will mark the operation as an upgrade and use the `UPGRADE` flag.
  * If the feature is enabled and the current version is higher than the desired one, it will mark the operation as a downgrade and use the `SAFE_DOWNGRADE` flag.
* It will try to update the feature using the `updateFeatures` Admin client operation.
  * In case of an error during the update, the error will be stored in the Kafka's `.status` section with `KafkaFeatureUpdateFailed` reason. The remaining features will still be processed.
* At the end, the operator will list all enabled features in the Kafka's `.status` section.

The status section will then look like this:
```yaml
  status:
    #...
    conditions:
    - lastTransitionTime: "2026-10-08T21:05:54.120449331Z"
      status: "True"
      type: Ready
    kafkaFeatures:
    - name: group.version
      version: 1
    - name: streams.version
      version: 1
    - name: transaction.version
      version: 2
    - name: eligible.leader.replicas.version
      version: 1
    - name: share.version
      version: 1
```

And in case of a failed update operation:
```yaml
  status:
    # ...
    conditions:
      - lastTransitionTime: "2026-10-08T21:05:54.002322524Z"
        message: Failed to update share.version to 5
        reason: KafkaFeatureUpdateFailed
        status: "True"
        type: Warning
      - lastTransitionTime: "2026-10-08T21:05:54.120449331Z"
        status: "True"
        type: Ready
    kafkaFeatures:
      - name: group.version
        version: 1
      - name: streams.version
        version: 1
      - name: transaction.version
        version: 2
      - name: eligible.leader.replicas.version
        version: 1
      - name: share.version
        version: 1
```

### Feature downgrade

As with `metadata.version`, the operator will only attempt safe downgrades using `SAFE_DOWNGRADE`.
If the downgrade would cause changes that might break the cluster, users must perform the required operations manually and downgrade the feature version using the `kafka-features.sh` script.
Strimzi should not perform operations that could bring the cluster into a non-functional state.

### Kafka version upgrade/downgrade

There will be no special handling during Kafka version upgrades or downgrades.
The operator will behave the same way as during normal reconciliation.
It will try to configure the features on the Kafka side.
In case of issues — for example, a feature existing in one version of Kafka that does not exist in another — the error will be reported in the `.status` section of the Kafka CR.

### Feature versions during Kafka version upgrades

When upgrading to a new Kafka version, the default finalized version for some features may increase (e.g., a feature that defaulted to version 1 in Kafka 4.2 might default to version 2 in Kafka 4.3).
Strimzi will not automatically upgrade feature versions — only features explicitly listed in `spec.kafka.features` are managed by the operator.
If a user has `share.version: 1` configured and upgrades to a Kafka version where the default is `share.version: 2`, the feature stays at version 1 until the user updates the CR.

## Affected/not affected projects

Changes made by this proposal affect the `strimzi-kafka-operator` repository.
Mainly the Kafka API module and the `KafkaReconciler` and `KRaftMetadataManager` classes.

## Compatibility

There are no backwards compatibility issues.

## Rejected alternatives

There are no rejected alternatives at the moment.
