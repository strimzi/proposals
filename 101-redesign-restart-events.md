# Redesign Restart Events

Update the Kubernetes events that are emitted by Strimzi cluster operator when rolling Pods to list the Kafka, Connect or MM2 resource as the `involvedObject`.

## Current situation

Currently, when the Kafka Pods are rolled, we issue Kubernetes Events describing the reason for the restart.
This is only done for the Kafka, Connect and MM2 Pod restarts.
The events are issued with the Pods as the `involvedObject`, for example:

```yaml
  action: StrimziInitiatedPodRestart
  # ...
  involvedObject:
    kind: Pod
    name: my-cluster-dual-role-0
    namespace: kafka
  kind: Event
  # ...
  message: Pod was manually annotated to be rolled
  # ...
  reason: ManualRollingUpdate
  reportingComponent: strimzi.io/cluster-operator
  reportingInstance: strimzi-cluster-operator-8d7bb7477-2dmxj
  # ...
```

Users can filter for events emitted by the Strimzi cluster operator using:
```shell
kubectl get events -n kafka --field-selector reportingComponent=strimzi.io/cluster-operator
```

## Motivation

Pods have many events when they are restarted, so although the restart reason event is issued by Strimzi cluster operator, it is still easily lost among other kubelet events.

## Proposal

The restart events emitted by the Strimzi cluster operator will be updated to have the `involvedObject` being the Kafka, Connect or MM2 resource, rather than the Pod being restarted.
This would mean an example event would now look like:

```yaml
  action: StrimziInitiatedPodRestart
  # ...
  involvedObject: # (1)
    kind: Kafka
    name: my-cluster
    namespace: kafka
  kind: Event
  # ...
  message: Pod my-cluster-dual-role-0 was manually annotated to be rolled # (2)
  # ...
  reason: ManualRollingUpdate
  related: # (3)
    kind: Pod
    name: my-cluster-dual-role-0
    namespace: kafka
  reportingComponent: strimzi.io/cluster-operator
  reportingInstance: strimzi-cluster-operator-55d66bf7bd-htjtd
  # ...
```

1. The `regarding` field in the [Event API](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/#Event) is changed to the Kafka (or Connect or MM2) resource.
  The `regarding` field maps to `involvedObject` in the output.
2. The restart message will be updated to include the Pod name to make it easier to identify the affected Pod when listing events.
3. The `related` field is added pointing to the Pod that is being rolled.

## Affected/not affected projects

This only affects the Strimzi cluster operator.

## Compatibility

This will be a change for users, however since the events being emitted aren't versioned there is no clear way to indicate this change.
The 0.46 release already includes several major changes like ZooKeeper and MirrorMaker 1 removal.
As a result it is reasonable to assume for this release users will be closely reviewing the changelog, so we should aim to include this change in that release.
If it misses the deadline we can wait for the v1 API as discussed in `Rejected alternatives`.

## Rejected alternatives

### Removing the restart events

We could fully remove the restart events.
There seem to be some users using them, but as far as we know it is a relatively small number of users.

If we remove the events entirely users must view the logs to see why a Pod was restarted.
Since the logs are fairly busy this can be hard to find, unless using a dedicated logging collection tool.
Even though the events do add some overhead, it isn't a great deal and they are a useful way to check why a Pod is restarting without having to trawl through logs.

### Waiting until the v1 API

A previous version of this proposal suggested releasing this change at the same time as the v1 API.
When the v1 API lands users are likely to be reviewing the changelog more thoroughly than for other releases, increasing the likelihood that they will not be caught out by the change.
The same statement is true for the 0.46 release since that release has other major changes like ZooKeeper and MirrorMaker 1 removal.
