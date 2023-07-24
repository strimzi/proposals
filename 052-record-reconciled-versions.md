# Record Reconciled Version in Custom Resource

For the `Kafka` Custom Resource it is often unclear when an upgrade has finished, whether all the brokers have rolled and to which version of the operator a cluster has been reconciled with.
The following proposal puts forward a mechanism to make this clear in the custom resource, making it easy to programmatically check for upgrade completion, and could also be used/adopted in the upgrade system tests, simplifying the code and making for a slightly better upgrade UX story.

## Current situation

Currently the only way to know if an upgrade has finished to completion is to check is all the zookeeper and kafka brokers have rolled.
https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/systemtest/src/test/java/io/strimzi/systemtest/upgrade/KafkaUpgradeDowngradeST.java#L292
Though this works, from a user standpoint it is far from ideal as it assumes a level of user knowledge about the cluster that may not be appropriate for all users.

## Motivation

This proposal allows a user to rely on the `Kafka` custom resource as their single source of truth, instead of watching/querying the individual kafka/zookeeper pods.
This proposal lays out all the information we need and at what points in the reconcile. This information is already available within the Reconcile loop, so orchestrating and figuring out if a component has been fully upgraded will be propagated into the Custom Resource via metadata.

## Proposal

This proposal will cover the `Kafka` and `StrimziPodSet` custom resources, but could be extended to include other Custom Resources if there would be value in it. This proposal starts with the `Kafka` Custom Resource as this is the core component of Strimzi, and its reliance on `StrimziPodSet` means we should implement it for this CR type as well.

This proposal assumes that all key value pairs mentioned will be writen to the `metadata.annotations` field [docs here](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).
Alternatively, there is an argument for putting them in the [metadata.labels field](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/), but given Strimzi's heavy use of the labels field, in order to not confuse a user who might think they had accidentally set the field themselves, this proposal has chosen to use annotations.


The following are the two proposed new annotations:

The `strimzi.io/reconciled` annotation is patched at the end of a **successful** reconcile, and signals to a user that the operator at version `X` was able to reconcile to successful completion.
This would be the annotation a user or System Test would query against to check whether upgrade was complete.
```
strimzi.io/reconciled: '0.37.0'
```

The `strimzi.io/reconciling` annotation is patched to the CR at the start of reconcile and signals to a user that the operator is attempting to reconcile the CR at version Y.
```
strimzi.io/reconciling: '0.37.0'
```
This annotation is required for the following reasons:
- It signals that a reconcile on version Y has started, and that the operator is functional without checking the logs of the operator.
- The operator updates the `strimzi.io/reconciling` annotation value on reconciliation startup without overriding the `strimzi.io/reconciled` annotation. If the operator errors during reconcile it will be clear that the currently reconciled version does not match the version that the operator is applying. A two stage commit mechanism here gives much more information regarding whether it started the reconciliation versus whether it finished successfully.

An important note here, in order to avoid unnecessary extra reconcilliations, and to avoid constant re-adding and removal of the annotation, this proposal suggests that `strimzi.io/reconciling` remains, even after a reconcile, and that the annotation is only ever updated once a new operator version attempts the reconcile.

## `Kafka` Custom Resource
- The update to `strimzi.io/reconciling` would happen in the `initialStatus` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaAssemblyOperator.java#L292)
with `strimzi.io/reconciled` remaining unchanged, leaving it unset if unset, or leaving it as whatever it was set to previously if set.
- The update to `strimzi.io/reconciled` would happen in the `createOrUpdate` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaAssemblyOperator.java#L137) alongside the `Ready` update where the operator version would be updated into the field.
- `KafkaReconciler` will also need to be modified to wait for all SPS to be updated, I believe in the `podsReady` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaReconciler.java#L817C4-L817C4) where instead of checking the pods the operator will insteady wait for the SPS CR to have `strimzi.io/reconciled` set to the operator version signalling its completion.


## `StrimziPodSet` Custom Resource
- The update to `strimzi.io/reconciling` would happen in the `reconcile` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/StrimziPodSetController.java#L382)
with `strimzi.io/reconciled` remaining unchanged, leaving it unset if unset, or leaving it as whatever it was set to previously if set.
- The update to `strimzi.io/reconciled` would happen in the `reconcile` [method](https://github.com/strimzi/strimzi-kafka-operator/blob/2a1fdf9d8695bb22a2bf977b0ba5414291530207/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/StrimziPodSetController.java#L407) alongside the finish of the status update where the operator version would be updated into the field.


## Examples

For the following example any key set to `null` counts as it not being set for explicitness and clarity.
e.g. then 
```
annotations:
  strimzi.io/reconciled: null
  strimzi.io/reconciling: null
```
would in a real scenario simply be:
```
annotations: {}
```

### Fresh Install

- On a fresh install, a user creates a Kafka CR:
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: null
  ```
- The Kafka Reconciler watcher picks up the CR and does an initial status update
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: 0.37.0
  ```
  and creates `StrimziPodSet` CRs with annotations:
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: null
  ```
- StrimziPodSet reconciler reconciles the SPS CR with initial status update:
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: 0.37.0
  ```
  and starts deploying Kafka brokers one at a time, and on completion updates the SPS CR to:
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.37.0
  ```
- Kafka Reconciler picks up that SPS is now Ready and finished, installing other components
  Once all other components are installed, the `Kafka` CR updates to:
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.37.0
  ```
- User can see `metadata.annotations[strimzi.io/reconciled] = 0.37.0` so they know it's reconciled to correct level


### Upgrade (from no mechansim)
Same as fresh install:

- On an upgrade from 0.36.0, a user has a Kafka CR:
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: null
  ```
- The Kafka Reconciler watcher picks up the CR and does an initial status update
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: 0.37.0
  ```
  and creates `StrimziPodSet` CRs with annotations:
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: null
  ```
- StrimziPodSet reconciler reconciles the SPS CR:
  ```
  strimzi.io/reconciled: null
  strimzi.io/reconciling: 0.37.0
  ```
  and starts deploying Kafka brokers one at a time, and on completion updates the SPS CR to:
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.37.0
  ```
- Kafka Reconciler picks up that SPS is now Ready and finished, installing other components
  Once all other components are installed, the `Kafka` CR updates to:
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.37.0
  ```
- User can see `metadata.annotations[strimzi.io/reconciled] = 0.37.0` so they know it's reconciled to correct level

### Upgrade (with new mechansim)
This defines an example where it is imagined that 0.37.0 had this mechanism already implemented, and is now upgrading to a new version 0.38.0

- On an upgrade from 0.37.0, a user has a Kafka CR:
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.37.0
  ```
- The Kafka Reconciler watcher picks up the CR and does an initial status update
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.38.0
  ```
  and updates `StrimziPodSet` CR (not changing the annotations from previous, since Kafka reconciler does not own them) with:
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.37.0
  ```
- StrimziPodSet reconciler reconciles the SPS CR with annotations:
  ```
  strimzi.io/reconciled: 0.37.0
  strimzi.io/reconciling: 0.38.0
  ```
  and starts deploying Kafka brokers one at a time, and on completion updates the SPS CR to:
  ```
  strimzi.io/reconciled: 0.38.0
  strimzi.io/reconciling: 0.38.0
  ```
- Kafka Reconciler picks up that SPS is now `Ready` and finished, installing other components
  Once all other components are installed, the `Kafka` CR updates to:
  ```
  strimzi.io/reconciled: 0.38.0
  strimzi.io/reconciling: 0.38.0
  ```
- User can see `metadata.annotations[strimzi.io/reconciled] = 0.38.0` so they know the CR has been reconciled to correct level


## Kafka Version Annotation
As a follow-on from this proposal, the mechanism could be extended to cover Kafka versions as well, such as:
```
strimzi.io/kafkaVersion: 3.5.0
strimzi.io/kafkaVersionReconciling: 3.6.0
```
Where the `strimzi.io/kafkaVersion` annotation is updated only once the SPS has rolled all brokers to the new version.
And `strimzi.io/kafkaVersionReconciling` signals that a Kafka version update is in progress if `kafkaVersionReconciling` and `kafkaVersion` don't match.

## Affected/not affected projects
To the best of my knowledge only the `strimzi-kafka-operator` would be effected by this mechanism, including the code changes to the reconcilers, the doc changes, and the updates to the system tests.

## Compatibility

This mechanism is backwards compatible, in the sense that it works even if the annotations were not set on prior versions.
No compatability issues, unless this feature was later removed in which case a user would no longer be able to rely on this annotation.

## Rejected alternatives

The original implementation was going to use fields in the CR status, i.e. `Kafka.status.reconciled` but this would involve API changes, which after discussion with others seemed less preferably to simply using labels/annotations.
