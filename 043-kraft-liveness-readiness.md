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
If it has a controller process it should be listening on the port defined as the `controller.listener.names` which in Strimzi is currently hard-coded to 9090.
If it has a broker process it should be listening on the replication port which is hard-coded to 9091.
If the node is running as a combined mode, it should be listening on both ports

### BrokerState
The expected [BrokerState](https://github.com/apache/kafka/blob/trunk/metadata/src/main/java/org/apache/kafka/metadata/BrokerState.java) transitions are:

NOT_RUNNING (0) -> STARTING (1) -> RECOVERY (2) -> RUNNING (3) -> PENDING_CONTROLLED_SHUTDOWN (6) -> SHUTTING_DOWN (7)

The other possible value is 127 which represents UNKNOWN.

### Current state metric and quorum

Some useful things to be aware of with the current-state metric and how the controller quorum affects the broker nodes:

* The [current-state](https://kafka.apache.org/documentation/#kraft_quorum_monitoring) metric indicates whether a controller is currently a leader or follower.
If this metric shows one of these two states, this means the leader election has happened and a controller quorum has been successfully formed.
* The current-state metric is not accessible to the existing KafkaAgent because it is not a Yammer metric.
* The broker nodes will not move to a RUNNING state until the quorum leader election has happened.

## Proposal

There are three different types of liveness/readiness probe that we need to consider, first for a combined node, 
then for a broker and then for a controller.

### Combined mode

The proposed probes are:

* Liveness: node is listening on 9090 and 9091
* Readiness: node is listening on 9090 and 9091

### Controller only mode

The proposed probes are:

* Liveness: node is listening on 9090
* Readiness: node is listening on 9090

### Broker only mode

The proposed probes are:

* Liveness: the BrokerState metric < 3 || listening on 9091
* Readiness: the BrokerState metric >=3 && != 127

**Note:** This means the brokers will not become ready until all the controllers 
are up and running.
This is similar to the current behaviour of brokers when ZooKeeper is not ready.

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

### Check BrokerState metric in combined mode

Since the brokers do not move to RUNNING until a quorum has been formed, the BrokerState metric cannot be used 
in combined mode as it would prevent the nodes becoming ready under certain circumstances.
For example in combined mode during normal startup the following would happen:

* All the pods are started at the same time
* The controller processes start listening on 9090 so the first part of the readiness check passes
* The broker processes start up and start talking to the controller quorum so the controller unfences them and they move to STARTED and therefore the second part of the readiness check passes
* All pods are now ready

However, if the pods are started initially with affinity constraints so they are all in pending, then the constraint is removed, the following would happen:

* The first pod is scheduled
* The controller process starts up and is listening on 9090 so the first part of the readiness check passes
* The broker process starts up but since the other controllers are not started yet it does not move to STARTED, it stays in STARTING. This means the second part of the readiness check fails
* Because this pod doesn't become ready the other pods aren't scheduled, so the pod just sits in crash loop backoff

A possible workaround is to in combined mode use the current-state metric to determine if a quorum has been formed.
However since this metric is not a Yammer metric the existing KafkaAgent cannot see it.
Two ways to address this could be to either change the way the KafkaAgent works or to see if the metric can be read from `$LOG_DIR/__cluster_metadata-0/quorum-state`.
It is also not ideal for the readiness probe of a single node to be dependent on other nodes in the cluster.
