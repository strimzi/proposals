# Record Reconciled Version in Kafka Custom Resource status

For the `Kafka` Custom Resource it is often unclear when an upgrade has truly finished, whether all the brokers have rolled and to which version of the operator a cluster has been reconciled with and which version of Kafka.
The following proposal puts forward a mechanism to make it clear in the `Kafka` custom resources status whether an upgrade is complete, making it easy to programmatically check for upgrade completion. This could potentially be used/adopted in the upgrade system tests or upgrade verification steps for a user, simplifying the code and making for a better upgrade UX story.

## Current situation

Currently the only way to know if an upgrade has finished to completion is to check is all the zookeeper and kafka brokers have rolled.
https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/systemtest/src/test/java/io/strimzi/systemtest/upgrade/KafkaUpgradeDowngradeST.java#L292
Additionally, if a user updates the Kafka version, a user needs to check if their Kafka version has been updated by inspecting the Kafka image/Jar version that the Brokers are running with, and this could require two rolls of the brokers, not just a single roll.
Though this works, from a user standpoint it is far from ideal as it assumes a level of user knowledge about the cluster that may not be appropriate for all users.

## Motivation

This proposal allows a user to rely on the `Kafka` custom resources as their single source of truth, instead of watching/querying the individual kafka/zookeeper pods.
This proposal lays out all the information that is required, and at what points in the reconcile this information will be used. This information is already readily available within the Reconcile loop, so orchestrating and figuring out if a component has been fully upgraded will be a case of propagating this information into the Custom Resource via status fields.

## Proposal

This proposal will cover the `Kafka` custom resource but could apply to other CR types that use `StrimziPodSets` such as `KafkaConnect` and `KafkaMirrorMaker2`, but could be extended to include other Custom Resources if there would be value in it.
This proposal starts with the `Kafka` Custom Resource as the base example as it is the core component of Strimzi, and it deploys both `StrimziPodSet`s and `Deployment`s.

This proposal assumes that all key value pairs mentioned will be writen to the `status` field as this is managed by the operator and can be added to without impacting end users.

The following is the proposed new fields:

### status.versions.lastSuccessfulReconciliationBy

The `status.versions.lastSuccessfulReconciliationBy` field is patched at the end of a **successful** reconcile, and signals to a user that the operator at version `X` was able to reconcile to successful completion, meaning it reached the end of a reconcile without error.
This would be the field a user would query against to check whether an upgrade was complete - or atleast a complete roll of the Kafka brokers on the latest operator version. Details on how to verify that an upgrade has completed when the `Kafka.spec.kafka.version` is updated is covered in the next section.

Example field in status:
```
status:
  versions:
    lastSuccessfulReconciliationBy: '0.37.0'
```
If a reconcile does not reach completion, this field is not added or updated, however in the case where it already exists it is left at the previous value.
i.e.
`status.versions.lastSuccessfulReconciliationBy=0.35.0` would remain if a post-operator-upgrade reconciliation on `0.36.0` failed, as the information that a prior reconciliation passed would be useful to an end user.

### status.versions.kafka

The `status.versions.kafka` field is updated in the CR at the end of the KafkaReconciler in the `podsReady` method and signals to a user that the KafkaReconciler has finished reconciling the Kafka brokers to Kafka version `Y`, regardless of success or failure later in the reconcile.
```
status:
  versions:
    kafka: '3.5.1'
```

Note: This will update the field in the reconcilers internal status object and have it be written at the end of the reconciliation rather then having a separate status update API call each time this is updated.

This field is required as it signals that a reconcile to Kafka version `Y` has finished, and this extra information is important and needs to be available, as a user just checking `status.versions.lastSuccessfulReconciliationBy` is not sufficient for upgrades where multiple rolls of the brokers are required, such as a Kafka version update alongside an upgrade.

If the operator errors during the Kafka reconciliation this field will not be updated. 
If Kafka was successfully updated, the field will also be updated, meaning errors later in the reconcile, such as failing to deploy the `entity-operator` or `kafka-exporter` will still show `status.versions.kafka` correctly up to date, but `lastSuccessfulReconciliationBy` will not be updated, and the CR will no longer be in a `Ready` state.


## `Kafka` Custom Resource & Reconciler changes

- Update `KafkaStatus` with new `versions` field and sub fields as agreed on this proposal.

- The update to `status.versions.kafka` would happen after the `podsReady` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/0c4e96de69e20cb80e86fd8ccb3ab8baca431822/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaReconciler.java#L817) updating the Kafka version once all pods are ready.

- The update to `status.versions.lastSuccessfulReconciliationBy` would happen in the `createOrUpdate` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaAssemblyOperator.java#L137) alongside the `Ready` update where the operator version would be updated into the field.


## Examples

### Fresh Install

- On a fresh install, a user creates a Kafka CR:
  ```
  # the status is not actually set, but treat it as empty here
  status: {}
  ```
- The KafkaReconciler reconciles the CR and creates `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the `SPS`s creating the Kafka Brokers
- Kafka Reconciler picks up that the Kafka Brokers managed by the SPS are now Ready and finished installing.
  Updating the status with the reconciled kafka versions:
  ```
  status:
    versions:
      kafka: 3.5.0
  ```
- Kafka Reconciler continues to install other components such as entity operator.
  Once all components are deployed and updated, update `Kafka` CR with
  ```
  status:
    versions:
      lastSuccessfulReconciliationBy: 0.37.0
      kafka: 3.5.0
  ```
- User can now verify `Kafka.status.versions.lastSuccessfulReconciliationBy` and `Kafka.status.versions.kafka` to check if it has reconciled to correct/expected levels.
- if there is an error at any point prior to the Kafka Broker rollout `Kafka.status.versions.kafka` will not be updated.
- if there is an error at any point during reconcile `Kafka.status.versions.lastSuccessfulReconciliationBy` will not be updated.


### Upgrade (from no mechansim)

Same as fresh install, upgrade from 0.37.0 to 0.38.0 with Kafka cersion 3.5.1
- On an upgrade, a user has a Kafka CR:
  ```
  # the status.versions is not set, but treat it as empty here
  status: {}
  ```
- The KafkaReconciler reconciles the CR and creates `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the `SPS`s creating the Kafka Brokers
- Kafka Reconciler picks up that the Kafka Brokers managed by the SPS are now Ready and finished installing.
  Updating the status with the reconciled kafka versions:
  ```
  status:
    versions:
      kafka: 3.5.1
  ```
- Kafka Reconciler continues to install other components such as entity operator.
  Once all components are deployed and updated, update `Kafka` CR with
  ```
  status:
    versions:
      lastSuccessfulReconciliationBy: 0.38.0
      kafka: 3.5.1
  ```
- User can now verify `Kafka.status.versions.lastSuccessfulReconciliationBy` and `Kafka.status.versions.kafka` to check if it has reconciled to correct/expected levels.
- if there is an error at any point prior to the Kafka Broker rollout `Kafka.status.versions.kafka` will not be updated.
- if there is an error at any point during reconcile `Kafka.status.versions.lastSuccessfulReconciliationBy` will not be updated.


### Upgrade (with new mechansim)

This defines an example where it is imagined that `0.37.0` had this mechanism already implemented, and is now upgrading to a new version `0.38.0` (with kafka upgrading from `3.5.0` to `3.5.1`)

- On an upgrade, a user has a Kafka CR:
  ```
  status:
    versions:
      lastSuccessfulReconciliationBy: 0.37.0
      kafka: 3.5.0
  ```
- The KafkaReconciler reconciles the CR and creates `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the `SPS`s creating the Kafka Brokers
- Kafka Reconciler picks up that the Kafka Brokers managed by the SPS are now Ready and finished installing.
  Updating the status with the reconciled kafka versions:
  ```
  status:
    versions:
      lastSuccessfulReconciliationBy: 0.37.0
      kafka: 3.5.1
  ```
- Kafka Reconciler continues to install other components such as entity operator.
  Once all components are deployed and updated, update `Kafka` CR with
  ```
  status:
    versions:
      lastSuccessfulReconciliationBy: 0.38.0
      kafka: 3.5.1
  ```
- User can now verify `Kafka.status.versions.lastSuccessfulReconciliationBy` and `Kafka.status.versions.kafka` to check if it has reconciled to correct/expected levels.
- if there is an error at any point prior to the Kafka Broker rollout `Kafka.status.versions.kafka` will not be updated.
- if there is an error at any point during reconcile `Kafka.status.versions.lastSuccessfulReconciliationBy` will not be updated.

## `KafkaNodePool` considerations

Some users use the new `KafkaNodePool` CRs to facilitate upgrades through rolling out new brokers with new Kafka versions, reassigning the partitions and then delete the old brokers.
This proposal should work with this method of upgrade, since the Kafka version + readiness checks will be track by the `StrimziPodSet`, so that when a user had finished upgrading via this method, the new `status.versions.kafka` should be updated accordingly.
In the potential case where a user is using multiple `KafkaNodePool`s for a cluster using different Kafka Versions, it is my recommendation that `status.versions.kafka` is a comma delimitered (sequentially ordered) list of all these Kafka Versions rather than the minimum or maximum kafka version.
In discussions it was deemed unlikely that Strimzi would choose to support multiple Kafka versions at once like this, but this would be my proposal for how it is displayed to the user if it was supported.

## Affected/not affected projects

To the best of my knowledge only the `strimzi-kafka-operator` would be affected by this mechanism, including the code changes to the reconcilers and the doc changes.

## Compatibility

This mechanism is backwards compatible, in the sense that it works even if the status fields were not set on prior versions or reconciles.
No compatibility issues, unless this feature was later removed in which case a user would no longer be able to rely on this status field.

## Rejected alternatives

- Using two new metadata annotations both `strimzi.io/reconciled` and `strimzi.io/reconciling` annotations, which would be a two staged update, so as to distinguish between a start of a reconcile and a completed one on the current operator version. However annotations can be overwritten by users.

- Just checking the pods from the StrimziPodSets, this is the current mechanism, but is hardly ideal, given there can be quite a few pods, and the KafkaAssembly already has a step to wait for all pods to be rolled, we can add the logic for updating the status to happen after this.

- Having a field `Kafka.status.versions.managedBy` that is updated at the beginning of reconcile, this has little value to the user and might just cause more confusion.

- It may be useful to also include versions for the `inter-broker-protocol-version` and `log-message-format-version` in the status, such as `status.versions.kafkaInterBrokerProtocol` and `status.versions.kafkaLogMessageFormat` as these too could cause multiple reconciles and a user might want to verify that these have been picked up by the brokers. However these are not used in KRaft as `log.message.format.version` is ignored if `inter.broker.protocol.version` is 3.0 or higher. `inter.broker.protocol.version` is not deprecated formally. But it is not used in KRaft mode. So it will be essentially removed in Kafka `4.0`. So removing it from this proposal.

- The `Kafka.status.versions.kafka` field being an array of `String`s for the purposes of covering the multiple-kafka versions scenario in the [KafkaNodePool section](#-`KafkaNodePool`-considerations).
  i.e
  ```
  status:
    versions:
      kafka:
      - '3.5.0'
      - '3.5.1'
  ```
  since this is unlikely to be supported, and reading a string rather than an array is easier, a single string for the field seems preferable.