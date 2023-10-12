# KafkaRoller KRaft Support

This proposal describes the actions that the KafkaRoller should take when operating 
against a Strimzi cluster in KRaft mode.
The proposal describes the checks the KafkaRoller should take, how it should perform 
those checks, and in what order, but does not discuss exactly how the KafkaRoller works.
This proposal is expected to apply to both the current KafkaRoller and a future iteration of the KafkaRoller.

This proposal assumes that liveness/readiness of nodes is as described in [proposal #46](https://github.com/strimzi/proposals/blob/main/046-kraft-liveness-readiness.md) 
and [PR #8892](https://github.com/strimzi/strimzi-kafka-operator/pull/8892).


## Current situation

When operating on a ZooKeeper based cluster the KafkaRoller has the behaviour 
described below.

### Order to restart nodes
Restart in order:
 1. Unready brokers
 2. Ready brokers
 3. Current controller

### Triggers
The following are some of the triggers that restart a broker:
- Pod or its StrimziPodSet is annotated for manual rolling update
- Pod is unready
- Broker's configuration has changed

### Restart conditions
The KafkaRoller considers the following in order to determine if it is ok to restart a pod:
 1. Does not restart a broker performing log recovery
 2. Attempts to connect an admin client to a broker and if it can't connect, restarts the broker
 3. Restarts a broker if the pod is in stuck state
 4. Restarts a broker if the configuration has changed and cannot be updated dynamically
 5. Does not restart a broker if doing so would take the in-sync replicas count below `min.insync.replicas` for any topic hosted on that broker

#### Unready pod
If pod is unready but not stuck, KafkaRoller awaits its readiness until the operational timeout is reached before doing anything. A pod is considered stuck if it is in one of following states:
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `ContainerCreating`
- `Pending` and `Unschedulable`

If a pod is stuck, KafkaRoller restarts it only if the pod is out of date. Otherwise, KafkaRoller fails the reconciliation process and does not restart other pods. If a pod is stuck, restarting the other pods might lead them to the same stuck state.
If pod is unready but not stuck, KafkaRoller considers the restart conditions that are based on configuration change and availability check (#4 and #5 of `Restart conditions` above) to restart the pod. 


#### Configuration changes
The KafkaRoller currently handles configuration updates to the brokers in the following way:
 - Retrieves the current Kafka configurations of the broker via admin client and compares it with the desired configurations specified in the Kafka CR. 
 - Performs dynamic configuration updates if possible

In KRaft mode the KafkaRoller currently skips controller only pods, but performs the above steps on any combined or broker only pods.
This is causing a problem in combined mode because if the quorum has not formed due to some of the pods not being ready 
the KafkaRoller will still try to contact the broker via the admin client.
This call fails because the quorum is not formed, so in some cases this results in the cluster being stuck with some pods 
in a pending state.

## Motivation

When running in the KRaft mode the controller pods need to be rolled if manually annotated, configuration changes occur or pod is unready. At the moment the 
existing logic is blocking the ZooKeeper to KRaft migration that is being proposed in [PR #90](https://github.com/strimzi/proposals/pull/90) as well as the full implementation of KRaft liveness and readiness checks as described in [proposal #46](https://github.com/strimzi/proposals/blob/main/046-kraft-liveness-readiness.md).

## Proposal

The KafkaRoller behaviour should be unchanged when operating against a ZooKeeper based cluster.

The proposed behaviour when operating against a KRaft cluster is described below.

### Order to restart nodes
Restart in order:
1. Unready controller/combined nodes
2. Ready controller/combined nodes in follower state
3. Active controller (applies to both pure controller and combined node)
4. Unready broker-only nodes
5. Ready broker-only nodes


### Triggers
The following are some of the triggers that would restart a KRaft controller or combined node:
- Pod or its StrimziPodSet is annotated for manual rolling update
- Pod is unready 
- Controller's configuration has changed

The triggers for broker remain the same as ZooKeeper mode.

### Restart conditions
The restart conditions that the KafkaRoller considers for different modes are described below.

#### The new quorum check
The restart conditions include a new check for controllers to verify that restarting the Kafka node does not affect the quorum health.
The proposed check ensures that a sufficient majority of controller nodes are caught up before allowing a restart:
- Create admin client connection to the brokers and call `describeMetadataQuorum` API.
- If failed to connect to the brokers, return `UnforceableProblem` which currently results in delay and retry for the pod until the maximum attempt is reached.
- From the quorum info returned from the admin API, read `lastCaughtUpTimestamp` of each controller. `lastCaughtUpTimestamp` is the last millisecond timestamp at which a replica controller was known to be caught up with the quorum leader.
- Check the quorum leader id using the quorum info and identify the `lastCaughtUpTimestamp` of the quorum leader.
- Retrieve value of the Kafka property `controller.quorum.fetch.timeout.ms` from the desired configurations specified in the Kafka CR. If this property does not exist in the desired configurations, then use the hard-coded default value for it which is `2000`. The reason for this is explained further in the **NOTE** below.
- Mark a KRaft controller node as caught up if `leaderLastCaughtUpTimestamp - replicaLastCaughtUpTimestamp < controllerQuorumFetchTimeoutMs`, or if it is the current quorum leader. This will exclude the controller KafkaRoller is currently considering to restart, because we are trying to check whether the quorum would stay healthy when this controller is restarted.
- Count each controller node that is caught up (`numOfCaughtUpControllers`).
- Can restart if: `numOfCaughtUpControllers >= ceil((double) (totalNumOfControllers + 1) / 2)`.

> NOTE: Until [KIP-919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum) is implemented, KafkaRoller cannot create an admin connection to the controller directly to describe its configuration or the quorum state. Therefore KafkaRoller checks the desired configurations to get the value of `controller.quorum.fetch.timeout.ms` and creates admin client connection to the brokers for the quorum check. If KafkaRoller cannot connect to any of the brokers after the maximum attempts to retry, the controllers will be marked as failed to reconcile because of not being able to determine the quorum health. KafkaRoller would then try to reconcile the brokers, which may help restoring admin client connections to them. In this scenario, the reconciliation will be completed with failure and reported to the operator. The controllers quorum check will be retried in the next round of reconciliation.


#### Separate brokers and controllers
For controller-only:
 1. Does not restart a controller performing log recovery
 2. Restarts a controller if the pod is in stuck state
 3. Attempts to connect an admin client to any of the brokers, if it cannot connect, returns `UnforceableProblem` which results in delay and retry for the pod until the maximum attempt is reached. 
When the maximum attempt is reached, the KafkaRoller will move on to restart broker pods in case this resolves connection issues but will still ultimately mark the reconciliation as failed.
 4. Restarts a controller if the controller configuration has changed
 5. Does not restart a controller if doing so would take the number of caught-up controllers (inc leader) to less than majority of the quorum

For broker-only:
 1. Does not restart a broker performing log recovery
 2. Restarts a broker if the pod is in stuck state
 3. Attempts to connect an admin client to a broker and if it can't connect restarts the broker
 > **NEW** Currently when pod is stuck and out of date, it gets restarted because admin client connection fails. With this change, we will not attempt admin client connection to the broker, if the pod is stuck. Instead the pod will be restarted without making the admin client connection. Therefore the resulting action that restarts the pod that is stuck stays the same for ZooKeeper mode. This resolves the issue mentioned at the beginning of the proposal that leaves some pods stuck in pending state.
 4. Restarts a broker if the broker configuration has changed and cannot be updated dynamically
 5. Does not restart a broker if doing so would take the in-sync replicas count below `min.insync.replicas` for any topic hosted on that broker


#### Combined mode
1. Does not restart a combined node performing log recovery
2. Restarts a combined node if the pod is in stuck state
3. Attempts to connect an admin client to a broker and if it can't connect restarts the broker
4. Restarts a combined node if the controller configuration has changed OR if the broker configuration has changed and cannot be updated dynamically
5. Does not restart a ready combined node if doing so would take the number of caught-up controllers (inc leader) to less than majority of the quorum
6. Does not restart a ready combined node if doing so would take the in-sync replicas count below `min.insync.replicas` for any topic hosted on that broker


#### Unready pod
Remains the same as ZooKeeper mode.


#### Configuration changes
As implemented in [PR #9125](https://github.com/strimzi/strimzi-kafka-operator/pull/9125) until [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration) is implemented:
 - For broker-only nodes, configuration changes are handled in the same way as brokers in the ZooKeeper case. If controller configurations changed, they will be ignored and brokers will not restart.
 - For controller-only nodes, a hash of the controller configs is calculated. A change to this hash causes a restart of the node, regardless of whether this configuration can be updated dynamically or not.
 - For combined nodes, both controller and broker configurations are checked and handled in the same way as brokers in the ZooKeeper case.

Once KIP 919 is implemented, the configurations of controller-only nodes will be diffed to see what values changed.
If the configurations that were updated are dynamic configurations, the KafkaRoller will call the Admin API to dynamically update 
these values. This will be similar to how dynamic configuration updates are handled in ZooKeeper mode.

## Affected/not affected projects

The only affected project is the Strimzi cluster operator.

## Compatibility

This proposal does not affect the ZooKeeper broker KafkaRoller behaviour.
This proposal does change the way that KRaft nodes are rolled, however since KRaft mode is not supported for production use 
and the existing logic is incomplete this is acceptable.

## Rejected alternatives

### Node rolling order
We considered rolling all unready pods, then all ready pods, regardless of whether they were controllers or brokers.
However, the problem with this approach is that for broker nodes to become ready the controller quorum must be formed.
