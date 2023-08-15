# Record Reconciled Version in Kafka Custom Resource status

With the `Kafka` Custom Resource, it is often unclear when an upgrade has truly finished. It can be difficult to ascertain if all brokers have finished rolling, as well as to determine the reconciled versions of both the operator and Kafka for the cluster.
The following proposal puts forward a mechanism to make it clear in the status of the `Kafka` custom resource whether an upgrade is complete, making it easy to programmatically check for upgrade completion. This could potentially be adopted in the upgrade system tests or upgrade verification steps for a user, simplifying the code and making for a better upgrade UX story.

## Current situation

Currently the only way to know if an upgrade has finished to completion is to check if all the ZooKeeper and Kafka brokers have rolled.
https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/systemtest/src/test/java/io/strimzi/systemtest/upgrade/KafkaUpgradeDowngradeST.java#L292
Additionally, if a user updates the Kafka version, they need to check if their Kafka version has been updated by inspecting the Kafka image/Jar version that the brokers are running with, and this could require two rolls of the brokers, not just a single roll.
Though this works, from a user standpoint it is far from ideal as it assumes a level of user knowledge about the cluster that may not be appropriate for all users.

## Motivation

This proposal allows a user to rely on the `Kafka` custom resources as their single source of truth, instead of watching/querying the individual Kafka and ZooKeeper pods.
This proposal lays out all the information that is required, and at what points in the reconcile this information will be used. This information is already readily available within the Reconcile loop, so orchestrating and figuring out if a component has been fully upgraded will be a case of propagating this information into the custom resource via status fields.

## Proposal

This proposal will cover the `Kafka` custom resource, but could apply to other CR types that use `StrimziPodSets` such as `KafkaConnect` and `KafkaMirrorMaker2`. It could be extended to include other custom resources if there is value in it.
This proposal starts with the `Kafka` custom resource as the base example as it is the core component of Strimzi, and it deploys `StrimziPodSet` and `Deployment` resources.

This proposal assumes that all key value pairs mentioned will be writen to the `status` field as this is managed by the operator and can be added to without impacting end users.

The following are the proposed new fields:

### status.operatorLastSuccessfulVersion

The `status.operatorLastSuccessfulVersion` field is patched at the end of a **successful** reconcile, and signals to a user that the operator at version `X` was able to reconcile to successful completion, meaning it reached the end of a reconcile without error.
This would be the field a user would query against to check whether an operator upgrade was complete - or at least a complete roll of the Kafka brokers on the latest operator version. Details on how to verify that an upgrade has completed when the `Kafka.spec.kafka.version` is updated is covered in the next section.

Example field in status:
```
status:
  operatorLastSuccessfulVersion: '0.37.0'
```
If a reconcile does not reach completion, this field is not added or updated, however in the case where it already exists it is left at the previous value.
i.e.
`status.operatorLastSuccessfulVersion=0.35.0` would remain if a post-operator-upgrade reconciliation on `0.36.0` failed, as the information that a prior reconciliation passed would be useful to an end user.

### status.kafkaVersion

The `status.kafkaVersion` field is updated in the CR at the end of the KafkaReconciler in the `podsReady` method and signals to a user that the `KafkaReconciler` has finished reconciling the Kafka brokers to Kafka version `Y`, regardless of success or failure later in the reconcile.
```
status:
  kafkaVersion '3.5.1'
```

Note: This will update the field in the reconciler's internal status object and have it be written at the end of the reconciliation rather then having a separate status update API call each time this is updated.

This field is required as it signals that a reconcile to Kafka version `Y` has finished, and this extra information is important and needs to be available, as a user just checking `status.operatorLastSuccessfulVersion` is not sufficient for upgrades where multiple rolls of the brokers are required, such as a Kafka version update alongside an upgrade.

If the operator errors during the Kafka reconciliation this field will not be updated, and will instead remain set to its prior set value, or unset if no previous value was set.
During an upgrade, the value in the `status.kafkaVersion` field may not accurately reflect the Kafka version for all brokers. However, this behavior is intentional. The operator updates the status field only upon completing an upgrade, allowing users to determine upgrade completion at the cluster level, rather than on a broker-by-broker basis. For a more granular check, users seeking individual broker versions can query the Kafka broker versions through pod queries.
If Kafka was successfully updated, the field will also be updated, meaning errors later in the reconcile such as failing to deploy the `entity-operator`, `cruise-control`, `kafka-exporter`, or any other components added in the future to the `KafkaAssembly`, will still show `status.versions.kafka` correctly up to date.  However, `operatorLastSuccessfulVersion` will not be updated, and the CR will no longer be in a `Ready` state.


## `Kafka` Custom Resource & Reconciler changes

- Update `KafkaStatus` with new `versions` field and sub fields as agreed on this proposal.

- The update to `status.kafkaVersion` would happen after the `podsReady` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/0c4e96de69e20cb80e86fd8ccb3ab8baca431822/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaReconciler.java#L817) updating the Kafka version once all pods are ready.

- The update to `status.operatorLastSuccessfulVersion` would happen in the `createOrUpdate` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaAssemblyOperator.java#L137) alongside the `Ready` update where the operator version would be updated into the field.


## Examples

### Fresh Install

- On a fresh install, a user creates a Kafka CR:
  ```
  # the status is not actually set, but treat it as empty here
  status: {}
  ```
- The `KafkaReconciler` reconciles the CR and creates `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the StrimziPodSets creating the Kafka brokers
- Kafka reconciler picks up that the Kafka brokers managed by the StrimziPodSets are now `Ready` and have finished installing.
  Updating the status with the reconciled kafka versions:
  ```
  status:
    kafkaVersion: 3.5.0
  ```
- Kafka reconciler continues to install other components such as Entity Operator.
  Once all components are deployed and updated, update `Kafka` CR with:
  ```
  status:
    operatorLastSuccessfulVersion: 0.37.0
    kafkaVersion: 3.5.0
  ```
- User can now verify `Kafka.status.operatorLastSuccessfulVersion` and `Kafka.status.kafkaVersion` to check if it has reconciled to the expected versions.
- if there is an error at any point prior to the Kafka broker rollout, `Kafka.status.kafkaVersion` will not be updated.
- if there is an error at any point during reconcile, `Kafka.status.operatorLastSuccessfulVersion` will not be updated.


### Upgrade - from no mechanism

Same as fresh install, upgrade from 0.37.0 to 0.38.0 with Kafka version 3.5.1
- On an upgrade, a user has a Kafka CR:
  ```
  # the status.versions is not set, but treat it as empty here
  status: {}
  ```
- The `KafkaReconciler` reconciles the CR and creates `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the StrimziPodSets creating the Kafka brokers
- Kafka reconciler picks up that the Kafka brokers managed by the StrimziPodSets are now `Ready` and have finished installing.
  Updating the status with the reconciled kafka versions:
  ```
  status:
    kafkaVersion: 3.5.1
  ```
- Kafka reconciler continues to install other components such as Entity Operator.
  Once all components are deployed and updated, update `Kafka` CR with:
  ```
  status:
    operatorLastSuccessfulVersion: 0.38.0
    kafkaVersion: 3.5.1
  ```
- User can now verify `Kafka.status.operatorLastSuccessfulVersion` and `Kafka.status.kafkaVersion` to check if it has reconciled to the expected versions.
- if there is an error at any point prior to the Kafka broker rollout, `Kafka.status.kafkaVersion` will not be updated.
- if there is an error at any point during reconcile, `Kafka.status.operatorLastSuccessfulVersion` will not be updated.


### Upgrade - with new mechanism

This defines an example where it is assumed that `0.37.0` had this mechanism already implemented, and is now upgrading to a new version `0.38.0` (with Kafka upgrading from `3.5.0` to `3.5.1`)

- On an upgrade, a user has a Kafka CR:
  ```
  status:
    operatorLastSuccessfulVersion: 0.37.0
    kafkaVersion: 3.5.0
  ```
- The `KafkaReconciler` reconciles the CR and creates `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the StrimziPodSets creating the Kafka brokers
- Kafka reconciler picks up that the Kafka brokers managed by the StrimziPodSets are now `Ready` and have finished installing.
  Updating the status with the reconciled Kafka versions:
  ```
  status:
    operatorLastSuccessfulVersion: 0.37.0
    kafkaVersion: 3.5.1
  ```
- Kafka reconciler continues to install other components such as Entity Operator.
  Once all components are deployed and updated, update `Kafka` CR with:
  ```
  status:
    operatorLastSuccessfulVersion: 0.38.0
    kafkaVersion: 3.5.1
  ```
- User can now verify `Kafka.status.operatorLastSuccessfulVersion` and `Kafka.status.kafkaVersion` to check if it has reconciled to the expected versions.
- if there is an error at any point prior to the Kafka broker rollout, `Kafka.status.kafkaVersion` will not be updated.
- if there is an error at any point during reconcile, `Kafka.status.operatorLastSuccessfulVersion` will not be updated.

## `KafkaNodePool` considerations

It might be that in the future each `KafkaNodePool` CR supports having its own Kafka version which might differ from other `KafkaNodePools` in the same Kafka cluster.
This proposal should work with this method of upgrade, since the Kafka version + readiness checks will be tracked by the `StrimziPodSet`, so that when a user had finished upgrading via this method, the new `status.kafkaVersion` should be updated accordingly.
In the potential case where a user is using multiple `KafkaNodePool`s for a cluster using different Kafka versions, it is my recommendation that `status.kafkaVersion` is a comma delimitered (sequentially ordered) list of all these Kafka Versions rather than the minimum or maximum Kafka version.
In discussions it was deemed unlikely that Strimzi would choose to support multiple Kafka versions at once like this, but this would be my proposal for how it is displayed to the user if it was supported.

## Affected/not affected projects

To the best of my knowledge only the `strimzi-kafka-operator` would be affected by this mechanism, including the code changes to the reconcilers and the doc changes.

## Compatibility

This mechanism is backwards compatible, in the sense that it works even if the status fields were not set on prior versions or reconciles.
No compatibility issues, unless this feature was later removed in which case a user would no longer be able to rely on this status field.

## Rejected alternatives

- Using `strimzi.io/reconciled` and `strimzi.io/reconciling` annotations, two new metadata annotations which would be a two-stage update, so as to distinguish between a start of a reconcile and a completed one on the current operator version. However annotations can be overwritten by users.

- Just checking the pods from the StrimziPodSets, this is the current mechanism, but is hardly ideal, given there can be quite a few pods, and the `KafkaAssembly` already has a step to wait for all pods to be rolled, we can add the logic for updating the status to happen after this.

- Having a field `Kafka.status.versions.managedBy` that is updated at the beginning of reconcile, which has little value to the user and might just cause more confusion.

- It may be useful to also include versions for the `inter-broker-protocol-version` and `log-message-format-version` in the status, such as `status.versions.kafkaInterBrokerProtocol` and `status.versions.kafkaLogMessageFormat` as these too could cause multiple reconciles and a user might want to verify that these have been picked up by the brokers. However these are not used in KRaft as `log.message.format.version` is ignored if `inter.broker.protocol.version` is 3.0 or higher. `inter.broker.protocol.version` is not deprecated formally. But it is not used in KRaft mode. So it will be essentially removed in Kafka `4.0`. So removing it from this proposal.

- The `Kafka.status.kafkaVersion` field being an array of `String`s for the purposes of covering the multiple-kafka versions scenario in the [KafkaNodePool section](#-`KafkaNodePool`-considerations).
  For example:
  ```
  status:
    kafkaVersion:
    - '3.5.0'
    - '3.5.1'
  ```
Since this is unlikely to be supported, and reading a string rather than an array is easier, a single string for the field seems preferable.

- Using `status.versions.lastSuccessfulReconciliationBy` and `status.versions.kafka`, but it was decided the `status.versions` field was not required and instead both fields could sit in the top-level status