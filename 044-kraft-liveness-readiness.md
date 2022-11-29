# Liveness and Readiness probes in a KRaft Kafka cluster

This proposal describes the liveness and readiness probes that should be put in place 
for a Kafka cluster that is using KRaft rather than ZooKeeper. This includes a KRaft 
cluster with combined nodes and one that has separate controller and broker nodes.

## Current situation

These are the current probes used in Strimzi when ZooKeeper is present:

|Component| Liveness | Readiness |
|---|---|---|
| ZooKeeper | "imok" from 127.0.0.1:12181 | "imok" from 127.0.0.1:12181 |
| ZK-based broker | if (ready) { listening on 9091 } else { have ZK session } | if (ready) { return true } |

The "ready" check in both cases is based on the existence of the file `../kafka-ready` which is 
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

### Current state metric and quorum

Some useful things to be aware of with the current-state metric and how the controller quorum affects the broker nodes:

* The [current-state](https://kafka.apache.org/documentation/#kraft_quorum_monitoring) metric indicates whether a controller is currently a leader or follower.
If this metric shows one of these two states, this means the leader election has happened and a controller quorum has been successfully formed.
* The current-state metric is not accessible as a Yammer metric, so cannot be accessed by the KafkaAgent in the same way it currently reads the BrokerState metric.
* The current-state metric could perhaps be read by the KafkaAgent if it called the MBean server within the JVM directly or read it from `$LOG_DIR/__cluster_metadata-0/quorum-state`.
* The broker nodes will not move out of the STARTING state until the quorum leader election has happened.

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

### Controller only mode

The proposed probes are:

* Liveness: controller is listening on 9090
* Readiness: it is possible to make a TCP connection to port 9090 on the controller
(i.e. client socket gets into the `ESTABLISHED` TCP socket state)

### Broker only mode

The proposed probes are:

* Liveness: broker is listening on 9091
* Readiness: the BrokerState metric >=3 && != 127

**Note:** This means the brokers will not become ready until all the controllers 
are up and running.
This is similar to the current behaviour of brokers when ZooKeeper is not ready.

### Combined mode

The proposed probes are:

* Liveness: node is listening on 9090
* Readiness: the BrokerState metric >=3 && != 127

**Note:** This means the nodes will not become ready until all the other nodes
are up and running.
This is acceptable because the Strimzi headless services use "publishNotReadyAddresses", which 
means the nodes will be able to communicate even if they are currently marked as not ready.

### Impact on the KafkaRoller

The existing KafkaRoller checks whether a pod is marked as ready to determine whether it needs to be rolled. 
It does not take into account the state of other broker pods.
The move to KRaft mode introduces some new requirements in terms of when a pod should or should not be rolled:

* If a broker pod has not become ready, KafkaRoller should check the controller quorum is formed before rolling the pod.
* If a combined pod has not become ready, KafkaRoller should check that all other combined pods have been scheduled 
(i.e. not in pending state) before rolling the pod or waiting for it to become ready.

The proposed readiness checks for combined mode will not work without changes to the KafkaRoller.
This is because the brokers do not move to RUNNING until a quorum has been formed.
For example in combined mode during normal startup the following would happen:

* All the pods are started at the same time
* The controller processes in each pod form a quorum
* The broker processes start up and start talking to the controller quorum so the controller unfences them and they move to STARTED and are marked as ready
* All pods are now ready

However, if the pods are started initially with affinity constraints so they are all in pending, then the constraint is removed, the following would happen:

* The first pod is scheduled
* The controller process starts up but cannot form a quorum because the other controllers are missing
* The broker process starts up but since there is no controller quorum it does not move to STARTED, it stays in STARTING. This means the pod is not marked as ready
* Because this pod doesn't become ready the other pods aren't scheduled, so the pod just sits in crash loop backoff

## Proposed order of tasks to complete this proposal

Since changes to the KafkaRoller and support for non-combined mode is needed to fully implement this proposal, the following phases of development are proposed:

### Phase 1 - Strimzi only supports combined mode
* Combined mode pods are marked as both alive and ready when the pod is listening on 9090. 
This is an improvement on the current state where the pod is marked as alive and ready as soon as it starts up.
* Update the KafkaAgent to check for a broker state of >= 3 && != 127.
This will be used for the broker only and combined readiness checks later and will improve the existing ZooKeeper based broker check.

### Phase 2 - Strimzi adds support for broker only and controller only modes
* Controller only pod liveness and readiness checks are fully implemented to match this proposal.
* Broker only pods are marked as both alive and ready if listening on port 9091.
* Combined mode pods are marked as both alive and ready when the pod is listening on 9090 (no change from phase 1).

### Phase 3 - KafkaRoller is updated
* KafkaRoller is updated to take the existence of controller pods and status of controller quorum into account when deciding whether to roll pods.

### Phase 4
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
