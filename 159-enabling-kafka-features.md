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
  * If the current version of the feature is the same as the desired version, skip all steps below.
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

### Interaction with Kafka version upgrades and downgrades

Finalized feature levels are stored in the KRaft metadata log and persist across binary changes.
They are not part of the broker configuration or the container image.
Changing the Kafka version in the CR triggers a rolling restart with a new binary, but the finalized feature levels in the metadata log remain unchanged.

#### Upgrades

Kafka upgrades are safe with regards to features.
Newer Kafka versions support the same or higher feature versions than older ones.
During a Kafka binary upgrade, features stay at their current finalized level — they are not automatically upgraded to the new Kafka version's defaults.
The operator processes feature updates after the rolling restart completes, same as with `metadata.version`.
Only features explicitly listed in `spec.kafka.features` are managed by the operator.

If a user has `share.version: 1` configured and upgrades to a Kafka version where the default is `share.version: 2`, the feature stays at version 1 until the user updates the CR.

#### Downgrades

Kafka downgrades require more care.
If a feature is finalized at a version that the target Kafka binary does not support, brokers will refuse to start and the rolling update will stall.
This means features must be downgraded or disabled **before** the Kafka binary is downgraded.

The recommended procedure for downgrading Kafka when features are configured:

1. Update the `features` list in the CR — downgrade features to versions compatible with the target Kafka version, or set features that do not exist in the target version to `version: 0` to disable them.
2. Wait for the operator to reconcile and apply the feature changes.
3. Optionally remove the disabled features from the list.
4. Update `spec.kafka.version` to the target Kafka version.

If the user changes both the features and the Kafka version in the CR at the same time, the operator will first perform the rolling restart with the new binary.
If the finalized feature levels are incompatible with the new binary, brokers will fail to start and the rolling update will stall.
This is consistent with how `metadata.version` downgrades are handled — the metadata version must be downgraded before the Kafka binary.

The downgrade procedure for features must be documented as part of the implementation PR.

### Removing a feature from the list

When a feature is removed from the `features` list, the operator stops managing it.
The feature stays at whatever version it was last set to in the KRaft metadata log.
The operator will not reset it to any default value.

There is no Kafka Admin API to determine the default feature version for a given Kafka release.
The default mappings are hardcoded in the Kafka server binary and only exposed through the `kafka-features.sh version-mapping` CLI command, which is not suitable for use during reconciliation.
Maintaining a copy of these mappings in Strimzi would create a maintenance burden and a risk of divergence with each Kafka release.

To disable a feature, users should explicitly set it to `version: 0` in the `features` list rather than removing it.
This makes the intent clear and avoids leaving a feature at an unintended version.

## Affected/not affected projects

Changes made by this proposal affect the `strimzi-kafka-operator` repository.
Mainly the Kafka API module and the `KafkaReconciler` and `KRaftMetadataManager` classes.

## Compatibility

There are no backwards compatibility issues.

## Rejected alternatives

### Automatically upgrading features to match Kafka version defaults

One option considered was to automatically upgrade feature versions to match the defaults of the deployed Kafka version during Kafka upgrades.
This was rejected because it violates the declarative principle — the operator should only set what the user explicitly declares in the CR.
Automatically upgrading features could lead to unexpected behavior changes after a Kafka version upgrade.

### Resetting features to defaults when removed from the list

Another option considered was to reset features to their Kafka release defaults when removed from the `features` list.
This was rejected because there is no Kafka Admin API to determine the default feature versions for a given release.
Maintaining a mapping of default versions per Kafka release in Strimzi would be error-prone and require updates with every Kafka release.
Instead, removing a feature from the list means the operator stops managing it, and users should set `version: 0` explicitly to disable a feature.
