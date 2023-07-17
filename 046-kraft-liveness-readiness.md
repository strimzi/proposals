# Liveness and Readiness probes in a KRaft Kafka cluster
> Note: This proposal has been updated since its initial merge.

This proposal describes the liveness and readiness probes that should be put in place 
for a Kafka cluster that is using KRaft rather than ZooKeeper. This includes a KRaft 
cluster with combined nodes and one that has separate controller and broker nodes.

## Current situation

These are the current probes used in Strimzi when ZooKeeper is present:

|Component| Liveness | Readiness |
|---|---|---|
| ZooKeeper | "imok" from 127.0.0.1:12181 | "imok" from 127.0.0.1:12181 |
| ZK-based broker | if (ready) { listening on 9091 } else { have ZK session } | if (ready) { return true } |

The "ready" check in both the liveness and readiness checks for a ZK-based broker is based on the existence of the file `../kafka-ready` which is 
created by the KafkaAgent when the broker state metric has value 3 (RUNNING).

## Background

This section contains some background information about the ports and metrics we could use for checks.

### Ports

In KRaft mode there are two different ports that the node could be listening on.
If it has a controller process it should be listening on the port defined as the `controller.listener.names` which in Strimzi is hard-coded to 9090.
If it has a broker process it should be listening on the replication port specified as the `inter.broker.listener.name` which is hard-coded to 9091.
If the node is running as a combined mode, it should be listening on both ports

### BrokerState

The expected [BrokerState](https://github.com/apache/kafka/blob/trunk/metadata/src/main/java/org/apache/kafka/metadata/BrokerState.java) transitions in the "happy path" during a normal broker lifecycle are:

NOT_RUNNING (0) -> STARTING (1) -> RECOVERY (2) -> RUNNING (3) -> PENDING_CONTROLLED_SHUTDOWN (6) -> SHUTTING_DOWN (7)

The other possible value is 127 which represents UNKNOWN.

You can see how the state transitions in the [BrokerLifecycleManager](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/server/BrokerLifecycleManager.scala#L379), specifically that 
the broker transitions from STARTING to RECOVERY once it has caught up with the cluster metadata, then it transitions to RUNNING as long as it is not fenced.

The broker nodes cannot catch up with the cluster metadata until the controller quorum has been formed, so they will not move out of the STARTING state until the quorum leader election has happened.

## Proposal

There are three different types of liveness/readiness probe that we need to consider, first for a controller only, then 
for a broker only, and finally for a combined node that has both roles.

The following statements describe the intent of the proposed probes:

* A controller is:
    * "alive" if it has a process running.
    * "ready" when it is ready to join the controller quorum.
* A broker is:
    * "alive" if it has a process running.
    * "ready" when it is ready to start accepting producer/consumer requests.
* A combined node is:
    * "alive" if it has a process running.
    * "ready" when it is ready to start accepting producer/consumer requests.
        * This implies that the controller quorum will have been formed while the nodes are not marked as ready, since it cannot start accepting producer/consumer requests until it can talk to the leader.
        * This relies on the node still accepting incoming connections from other controller nodes even if it isn't actually marked as "ready" yet.

### Controller only mode

The proposed probes are:

* Liveness: the Java process in the container is running
* Readiness: controller is listening on the port 9090 which is configured in `controller.listener.names`

### Broker only mode

The proposed probes are:

* Liveness: the Java process in the container is running
* Readiness: the BrokerState metric >=3 && != 127

**Note:** This means the brokers will not become ready until a majority of the controllers 
are up and running.
This is similar to the current behaviour of brokers when ZooKeeper is not ready.

### Combined mode

The proposed probes are:

* Liveness: the Java process in the container is running
* Readiness: the BrokerState metric >=3 && != 127

**Note:** This means the nodes will not become ready until a majority of the other controllers are up and running.
This is acceptable because the Strimzi headless services use "publishNotReadyAddresses", which 
means the nodes will be able to communicate even if they are currently marked as not ready.

### Implementation of the checks

To check that the Java process is running we will execute a command in the container. The proposed command is:
```shell
for proc in $(ls -1 /proc/ | grep [0-9]); do echo "$(ls -lh /proc/$proc/exe 2>/dev/null || true)" | grep -q java; done
```
This is preferable over using a tool like `ps` (`ps aux | grep -v grep | grep java`) since it does not require a change to the tools added to the container image.
In future this could be replaced with a call to an endpoint, since other proposals discuss adding endpoints, for example, to expose metrics.

To check that the server is listening on a particular port we will use `netstat`, e.g.:
```shell
netstat -lnt | grep -Eq 'tcp6?[[:space:]]+[0-9]+[[:space:]]+[0-9]+[[:space:]]+[^ ]+:9091.*LISTEN[[:space:]]*'
```

To check that the BrokerState metric has the correct value we will use the existing mechanism of checking for the existence of 
a file on disk that is created by the KafkaAgent.
In future this could be replaced with a call to an endpoint, since other proposals discuss adding an endpoint that returns this metric.

### Impact on the KafkaRoller

The existing KafkaRoller checks whether a pod is marked as ready to determine whether it needs to be rolled.
This relies on the fact that currently the pod being ready implies that the BrokerState is >= RUNNING.
This means any change to the readiness probes also impacts the KafkaRoller.
In future it would be better for the KafkaRoller to do its own checks, rather than relying on the readiness probes implementation.

The details for how the KafkaRoller should work going forwards should be discussed in a separate proposal.
This section describes some ideas for how the KafkaRoller should work in KRaft mode based on investigations done while writing this proposal.
It can be used as a starting point for the KafkaRoller in KRaft mode proposal.
The key requirement for this proposal to be fully implemented is that the KafkaRoller takes into account the state of controller pods when deciding whether to roll broker pods.

Based on investigations as part of writing this proposal, in KRaft mode:
* The KafkaRoller should observe the BrokerState metric and any other metrics it needs directly, rather than inferring the state based on readiness.
* If more than one controller pod has not become ready, KafkaRoller should try to roll the controllers that aren't the current active controller first.
* If a broker pod has not become ready, KafkaRoller should check the controller quorum is formed before rolling the pod.
* If a combined pod has not become ready, KafkaRoller should check that all other combined pods have been scheduled 
(i.e. not in pending state) before rolling the pod or waiting for it to become ready.

The reason for the final bullet (combined pod not being ready) might not be immediately obvious, so it is explained below.
The brokers do not move to RUNNING until a quorum has been formed.
For example in combined mode during normal startup the following would happen:

* All the pods are started at the same time
* The controller processes in each pod form a quorum
* The broker processes start up and start talking to the controller quorum so the controller unfences them and they move to RUNNING and are marked as ready
* All pods are now ready

However, if the pods are started with affinity constraints they are all put into a pending state by Kubernetes, since they cannot be scheduled. Then, if the constraint is removed or new nodes added that meet the requirements, the following happens:

* The first pod is scheduled
* The controller process starts up but cannot form a quorum because the other controllers are missing
* The broker process starts up but since there is no controller quorum it does not move to RUNNING, it stays in STARTING. This means the pod is not marked as ready
* Because this pod doesn't become ready, the KafkaRoller will not try to schedule the other pods, even though there are nodes available for them to be scheduled on

The result is that even though the cluster should be able to come up, one pod is in crash loop backoff and the other pods remain as pending, so the cluster never starts.

## Proposed order of tasks to complete this proposal

The KRaft liveness and readiness probes can be implemented in 3 different phases.
Phases 1 and 2 can be implemented solely based on this proposal.
Phase 3 can be implemented only after additional improvements of the `KafkaRoller` which will be covered by a separate proposal.

### Phase 1 - Strimzi only supports combined mode

* Combined mode pods are marked as both alive and ready when there is a Java process running. 
This is an improvement on the current state where the pod is marked as alive and ready as soon as the container starts up, regardless of whether the Java process started.
* Update the KafkaAgent to check for a broker state of >= 3 && != 127.
This will be used for the broker only and combined readiness checks later and will improve the existing ZooKeeper based broker check.

### Phase 2 - Strimzi adds support for broker only and controller only modes

* Controller only pod liveness and readiness checks are fully implemented to match this proposal.
* Broker only pods are marked as both alive and ready when there is a Java process running (no change from phase 1).
* Combined mode pods are marked as both alive and ready when there is a Java process running (no change from phase 1).

### Phase 3 - KafkaRoller is updated to take the existence of controller pods and status of controller quorum into account when deciding whether to roll pods

* Broker only readiness probe fully implemented to match this proposal.
* Combined mode readiness probe fully implemented to match this proposal.

## Affected/not affected projects

This change only affects the Cluster Operator and Kafka brokers.

## Compatibility

As part of this change the readiness probe for a Kafka broker in a ZooKeeper based cluster will be updated to 
continue being marked as ready when the BrokerState metric > 3 && != 127.
This means it will tolerate the PENDING_CONTROLLED_SHUTDOWN and SHUTTING_DOWN states.
This will not cause any compatibility problems because currently the KafkaAgent stops running once the metric reaches 
the RUNNING state.
The change will make the code between KRaft and ZooKeeper mode simpler and protect readiness if in future the agent is 
updated to continue running once the broker is "ready".

## Rejected alternatives

### Use the current-state metric for controller readiness

The [current-state](https://kafka.apache.org/documentation/#kraft_quorum_monitoring) metric indicates the current state of the quorum.
Previous versions of this proposal discussed whether we could check if this metric had a state of either follower or leader and use that to infer that the controller quorum had been formed and the controller should be marked as ready.
This approach had a couple of problems:
* The individual state of the controller doesn't always imply the full quorum state. 
For example, this particular controller might be offline, but the rest of the quorum is still healthy.
Also, during a leadership election the metric might show a state of `candidate` or `voted`.
* In general readiness checks in Kubernetes should be based on individual nodes, not the cluster as a whole.
* The current-state metric is not accessible as a Yammer metric, so cannot be accessed by the KafkaAgent in the same way it currently reads the BrokerState metric.
Although it could perhaps be read by the KafkaAgent if it called the MBean server within the JVM directly or read it from `$LOG_DIR/__cluster_metadata-0/quorum-state`.
* The broker nodes will not move out of the STARTING state until the quorum leader election has happened.

### Only mark controller nodes as ready when they are accepting incoming connections

We could only mark the controller node as ready when it is possible to make a TCP connection to the first port in `controller.listener.names` on the controller.
However, this would require us to make a client TCP connection to the controller, which in turn would create a lot of noise in the logs.
On further investigation into the [ControllerServer](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/server/ControllerServer.scala) class there seems to be very few actions taking place between the controller starting to listen and accepting connections.
As a result it was deemed there would be minimal benefit to having a check that is specific to making a connection, over just checking if the controller is listening on the required port.
