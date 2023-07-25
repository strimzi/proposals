# Record Reconciled Version in Kafka Custom Resource status

For the `Kafka` Custom Resource it is often unclear when an upgrade has finished, whether all the brokers have rolled and to which version of the operator a cluster has been reconciled with.
The following proposal puts forward a mechanism to make this clear in the `Kafka` custom resources status, making it easy to programmatically check for upgrade completion that could also be used/adopted in the upgrade system tests, simplifying the code and making for a better upgrade UX story.

## Current situation

Currently the only way to know if an upgrade has finished to completion is to check is all the zookeeper and kafka brokers have rolled.
https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/systemtest/src/test/java/io/strimzi/systemtest/upgrade/KafkaUpgradeDowngradeST.java#L292
Though this works, from a user standpoint it is far from ideal as it assumes a level of user knowledge about the cluster that may not be appropriate for all users.

## Motivation

This proposal allows a user to rely on the `Kafka` custom resources as their single source of truth, instead of watching/querying the individual kafka/zookeeper pods.
This proposal lays out all the information we need and at what points in the reconcile. This information is already available within the Reconcile loop, so orchestrating and figuring out if a component has been fully upgraded will be propagated into the Custom Resource via status fields.

## Proposal

This proposal will cover the `Kafka` custom resource but could apply to other CR types that use `StrimziPodSets` such as `KafkaConnect` and `KafkaMirrorMaker2`, but could be extended to include other Custom Resources if there would be value in it.
This proposal starts with the `Kafka` Custom Resource as the base example as it is the core component of Strimzi, and it deploys both `StrimziPodSet`s and `Deployment`s.

This proposal assumes that all key value pairs mentioned will be writen to the `status` field as this is managed by the operator and can be added to without impacting end users.

The following is the proposed new fields:

### status.versions.reconciled
The `status.versions.reconciled` field is patched at the end of a **successful** reconcile, and signals to a user that the operator at version `X` was able to reconcile to successful completion.
This would be the field a user or System Test would query against to check whether upgrade was complete.
```
status:
  versions:
    reconciled: '0.37.0'
```
If a reconcile does not reach completion, this field is not added or updated, however in the case where it already exists it is left at the previous value.
i.e.
`status.versions.reconciled=0.35.0` would remain if a reconcilliation on `0.36.0` failed as the information that a prior reconcilliation passed could be useful to an end user.

### status.versions.managedBy
The `status.versions.managedBy` field is patched to the CR at the start of reconcile in the `initialStatus` and signals to a user that the operator is attempting to reconcile the CR at version `Y`, regardless of success or failure.
```
status:
  versions:
    managedBy: '0.37.0'
```

This field is required for the following reasons:
- It signals that a reconcile on version Y has started, and that the operator reconciler & watcher are functional without checking the logs of the operator.
- The operator updates the `status.versions.managedBy` field on reconciliation startup without overriding the `status.versions.reconciled` field. If the operator errors during reconcile it will be clear that the currently reconciled version does not match the version that the operator is applying. A two stage commit mechanism here gives much more information regarding whether it started the reconciliation versus whether it finished successfully.

An important note here, in order to avoid unnecessary extra reconcilliations, and to avoid constant re-adding and removal of the field, this proposal suggests that `status.versions.managedBy` remains, even after a reconcile, and that the annotation is only ever updated once a new operator version attempts the reconcile.


## `Kafka` Custom Resource changes
- The update to `status.versions.managedBy` would happen in the `initialStatus` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaAssemblyOperator.java#L292)
- The update to `status.versions.reconciled` would happen in the `createOrUpdate` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaAssemblyOperator.java#L137) alongside the `Ready` update where the operator version would be updated into the field.


## Examples

For the following example any key set to `null` counts as it not being set for explicitness and clarity.
e.g. then 
```
reconciled: null
managedBy: null
```
would in a real scenario simply be:
```
status: {}
```

### Fresh Install

- On a fresh install, a user creates a Kafka CR:
  ```
  reconciled: null
  managedBy: null
  ```
- The Kafka Reconciler watcher picks up the CR does an initial status update:
  ```
  reconciled: null
  managedBy: 0.37.0
  ```
  and creates `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the `SPS`s
- Kafka Reconciler picks up that `SPS` is now Ready and finished installing.
  Kafka Reconciler continues to install other components such as entity operator.
  Once all components are deployed and updated, update `Kafka` CR with
  ```
  reconciled: 0.37.0
  managedBy: 0.37.0
  ```
- User can now verify `Kafka.status.versions.reconciled` to check if it has reconciled to correct/expected level.
- if there is an error at any point during reconcile `Kafka.status.versions.reconciled` will not be updated.


### Upgrade (from no mechansim)
Same as fresh install, upgrade from 0.37.0 to 0.38.0

- User has pre-existing Kafka CR:
  ```
  reconciled: null
  managedBy: null
  ```
- The Kafka Reconciler watcher picks up the CR does an initial status update:
  ```
  reconciled: null
  managedBy: 0.38.0
  ```
  and patches `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the `SPS`s with help of `KafkaRoller` and `ZookeeperRoller`
- Kafka Reconciler picks up that `SPS` is now Ready and finished upgrading.
  Kafka Reconciler continues to install other components such as entity operator.
  Once all components are upgraded, update `Kafka` CR with
  ```
  reconciled: 0.38.0
  managedBy: 0.38.0
  ```
- User can now verify `Kafka.status.versions.reconciled` to check if it has reconciled to correct/expected level.
- if there is an error at any point during reconcile `Kafka.status.versions.reconciled` will not be updated.


### Upgrade (with new mechansim)
This defines an example where it is imagined that `0.37.0` had this mechanism already implemented, and is now upgrading to a new version 0.38.0

- User has pre-existing Kafka CR:
  ```
  reconciled: 0.37.0
  managedBy: 0.37.0
  ```
- The Kafka Reconciler watcher picks up the CR does an initial status update:
  ```
  reconciled: 0.37.0
  managedBy: 0.38.0
  ```
  and patches `StrimziPodSet` CRs
- StrimziPodSet reconciler reconciles the `SPS`s with help of `KafkaRoller` and `ZookeeperRoller`
- Kafka Reconciler picks up that `SPS` is now Ready and finished upgrading.
  Kafka Reconciler continues to install other components such as entity operator.
  Once all components are upgraded, update `Kafka` CR with
  ```
  reconciled: 0.38.0
  managedBy: 0.38.0
  ```
- User can now verify `Kafka.status.versions.reconciled` to check if it has reconciled to correct/expected level.
- if there is an error at any point during reconcile `Kafka.status.versions.reconciled` will not be updated.


## Affected/not affected projects
To the best of my knowledge only the `strimzi-kafka-operator` would be affected by this mechanism, including the code changes to the reconcilers, the doc changes, and the updates to the system tests.

## Compatibility

This mechanism is backwards compatible, in the sense that it works even if the status fields were not set on prior versions.
No compatibility issues, unless this feature was later removed in which case a user would no longer be able to rely on this status field.

## Rejected alternatives

- Using two new metadata annotations both `strimzi.io/reconciled` and `strimzi.io/reconciling` annotations, which would be a two staged update, so as to distinguish between a start of a reconcile and a completed one on the current operator version. However annotations can be overwritten by users.

- Just checking the pods from the StrimziPodSets, this is the current mechanism, but is hardly ideal, given there can be quite a few pods, and the KafkaAssembly already has a step to wait for all pods to be rolled, we can add the logic for updating the status to happen after this.