# Topic replication factor change

In Kafka, the unit of replication is the topic partition.
The total number of partition replicas, including the leader, constitutes the replication factor.

The replication factor is configured at the topic level, and can be used to enable strong durability and fault tolerance.
When the Kafka cluster is healthy and a broker is down, another one can serve the data of topics with replication factor greater than one.
A tradeoff is that each topic partition will take up an amount of storage equal to its size times the replication factor value.
The cluster `default.replication.factor` configuration is used when a new topic is created without specifying the replication factor.

When new messages arrives, they are first written into the operating system's page cache, and then flushed to disk asynchronously.
If the Kafka JVM crashes for whatever reason, recent messages are still in the page cache, and can be flushed by the operating system.
However, this doesn't protect from data loss when the machine crashes, and this is why enabling topic replication is important.
To further improve fault tolerance, a rack-aware Kafka cluster can be used to distribute topic replicas evenly across data centers in the same geographic region.

Once a topic is created, it is possible to change its replication factor using a command line tool.
This operation is quite complicated on busy clusters with thousands of partitions, because the user have to decide leader and partition movements without causing cluster unbalance.

The goal of this proposal is to allow Strimzi users to change the topic replication factor by simply updating the KafkaTopic resource.
Unless specified, we always refer to the Unidirectional Topic Operator implementation.

## Current situation

When the user change the `.spec.replicas` field of a KafkaTopic, the Topic Operator fails the reconciliation, and the resource becomes not ready. 
An error is reported in Topic Operator logs and KafkaTopic resource status.

> Replication factor change not supported, but required for partitions\
> [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21,\
> 22, 23, 24]

The only [documented solution](https://strimzi.io/docs/operators/latest/deploying#proc-changing-topic-replicas-str) is to use the kafka-reassign-partitions.sh tool by spinning up a separate interactive pod.
This is far from user friendly, and the user is in charge of doing the calculations to determine the best broker to host a new replica, or which replica needs to be dropped.

When data loss is not a problem, the user can simply delete and recreate the KafkaTopic resource with the desired replication factor.
This is not really an option for most use cases.

## Motivation

The topic replication factor change feature will provide a much better cloud-native experience for Strimzi users.
They will be able to change the topic replication factor by simply updating the KafkaTopic resource, which may also happen through a GitOps pipeline.

This is a non exhaustive list of use cases that will become significantly easier:

- Set a lower replication factor for non-critical topics to reduce the infrastructure costs (compute, storage and network).
- Set a lower replication factor to deal with resource shortage (trade-off between fault-tolerance and resource utilization).
- Set a higher replication factor for critical topics to improve durability and fault tolerance.

A tighter integration with Cruise Control is also one of the long term goals of the Strimzi project. 

## Proposal

The replication factor change feature can be implemented by integrating the Topic Operator with Cruise Control.
The Cluster Operator is already integrated with Cruise Control, so we can factor out the existing logic and classes.

Cruise Control creates a task for each operation, such as topic configuration and cluster rebalance.
A task is first created in the Active state, then moves to InExecution, and finally to Completed or CompletedWithError.
At most `max.active.user.tasks` tasks can be Active at the same time (5 by default), and only one task can be InExecution at any given time.
During the execution, a task is divided in one or more batches, which are then executed one after the other.
A task may include multiple topics and it is marked as completed once the whole task's status is Completed.

### Topic configuration

The [`topic_configuration`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#change-kafka-topic-configuration) endpoint is used to request a topic configuration update, and currently only supports replication factor changes.
Once accepted, the replication factor change is put into the Cruise Control's task queue and executed asynchronously.

The two required parameters for this endpoint are `topic` (regular expression to specify subject topics) and `replication_factor` (target replication factor).
In order to group multiple changes in the same request to Cruise Control, the Topic Operator uses a JSON payload containing topics grouped by their replication factor.

This is useful with bulk replicas change, which can happen in some edge cases.
A potentially big backlog of messages may be created when the Topic Operator is not running, or there is an issue with Cruise Control or Kafka.
Another example is a GitOps pipeline delivering many replicas changes at the same time to reduce the storage occupation of non-critical topics.

```sh
$ curl -vv -X POST -H "Content-Type: application/json" -d '{replication_factor:{topic_by_replication_factor:{2:topic1|topic3|topic4,3:topic2|topic5}}}' \
  "http://localhost:9090/kafkacruisecontrol/topic_configuration?skip_rack_awareness_check=true&dryrun=false&json=true" | jq
> POST /kafkacruisecontrol/topic_configuration?skip_rack_awareness_check=true&dryrun=false&json=true HTTP/1.1
> Host: localhost:9090
> User-Agent: curl/8.0.1
> Accept: */*
> 
} [91 bytes data]
< HTTP/1.1 200 OK
< Date: Thu, 07 Dec 2023 14:58:55 GMT
< Set-Cookie: JSESSIONID=node0lfkx183ijq4pjkkjahsegfmp30.node0; Path=/
< Expires: Thu, 01 Jan 1970 00:00:00 GMT
< User-Task-ID: 5344ca89-351f-4565-8d0f-9aade00e053d
< Content-Type: application/json;charset=utf-8
< Cruise-Control-Version: 2.5.77-SNAPSHOT
< Cruise-Control-Commit_Id: 535ea07e8847990cc02dde1f1be99b387dbeaf5b
< Content-Length: 17051
< Server: Jetty(9.4.52.v20230823)
< 
{ [17321 bytes data]
{
  "summary": {
    "numIntraBrokerReplicaMovements": 0,
    "numReplicaMovements": 125,
    "onDemandBalancednessScoreAfter": 100,
    "intraBrokerDataToMoveMB": 0,
    "monitoredPartitionsPercentage": 100,
    "provisionRecommendation": "",
    "excludedBrokersForReplicaMove": [],
    "excludedBrokersForLeadership": [],
    "provisionStatus": "RIGHT_SIZED",
    "onDemandBalancednessScoreBefore": 100,
    "recentWindows": 5,
    "dataToMoveMB": 0,
    "excludedTopics": [],
    "numLeaderMovements": 0
  }
  ...
```

The `skip_rack_awareness_check` parameter configures whether to skip the rack awareness sanity check or not (default to false).
In case of a single-rack deployment, the Topic Operator automatically enables `skip_rack_awareness_check` to allow replication factor changes.

The `goals` parameter is a comma separated list of goals used to generate the automatic cluster rebalance for the replication factor change.
This is configurable for all endpoints in `Kafka.spec.cruiseControl.config` using the `default.goals` property.

The `replication_throttle` parameter is an upper bound on the bandwidth used to move replicas (bytes per second).
This is configurable for all endpoints in `Kafka.spec.cruiseControl.config` using the `default.replication.throttle` property.

### User tasks

The [`user_tasks`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-user-request-result) endpoint is used to periodically check the replicas change tasks state.
The only required parameter is the `User-Task-ID`, which is returned by the `topic_configuration` response.
A completed task is retained for `completed.user.task.retention.time.ms`, which defaults to 24 hours.

```sh
$ curl -s "localhost:9090/kafkacruisecontrol/user_tasks?user_task_ids=5344ca89-351f-4565-8d0f-9aade00e053d&json=true" | jq
{
  "userTasks": [
    {
      "Status": "Completed",
      "ClientIdentity": "127.0.0.1",
      "RequestURL": "POST /kafkacruisecontrol/topic_configuration?topic=my-topic&replication_factor=2&skip_rack_awareness_check=true&dryrun=false&json=true",
      "UserTaskId": "5344ca89-351f-4565-8d0f-9aade00e053d",
      "StartMs": "1701961135978"
    }
  ],
  "version": 1
}
```

### Configuration parameters

The Topic Operator needs to authenticate and send HTTP requests to the Cruise Control's REST API.
The following environment variables are set by the Cluster Operator for consumption by the Topic Operator when Cruise Control is configured in Kafka resource, or manually by the user in case of standalone deployment:

| Name                                   | Type    | Default | Description                                                   |
|:---------------------------------------|---------|---------|---------------------------------------------------------------|
| STRIMZI_CRUISE_CONTROL_ENABLED         | Boolean | false   | Whether Cruise Control integration is enabled                 |
| STRIMZI_CRUISE_CONTROL_HOSTNAME        | String  | ""      | Cruise Control hostname                                       |
| STRIMZI_CRUISE_CONTROL_PORT            | Integer | 9090    | Cruise Control port                                           |
| STRIMZI_CRUISE_CONTROL_SSL_ENABLED     | Boolean | false   | Whether SSL encryption is enabled                             |
| STRIMZI_CRUISE_CONTROL_AUTH_ENABLED    | Boolean | false   | Whether Basic authentication is enabled                       |
| STRIMZI_CRUISE_CONTROL_RACK_ENABLED    | Boolean | false   | Whether rack awareness is enabled in the target Kafka cluster |
| STRIMZI_CRUISE_CONTROL_SECRET_NAME     | String  | ""      | Secret name containing the truststore                         |
| STRIMZI_CRUISE_CONTROL_API_SECRET_NAME | String  | ""      | Secret name containing the API credentials                    |

With Cluster Operator deployment, a dedicated admin user is automatically created for the Topic Operator.
With standalone deployment, the Cluster Operator expects two secrets that needs to be created by the user:

- A secret with `cruise-control.p12` key containing the truststore in PKCS12 format, and `cruise-control.password` key containing the truststore password.
- A secret with `cruise-control.apiToAdminUsername` key containing the API username with admin role, and `cruise-control.apiToAdminPassword` key containing the user password.

### KafkaTopic CRD changes

During the whole replicas change process the KafkaTopic remains in a ready state.
A replicas change is executed asynchronously, and takes more than one reconciliation to complete.

A new optional KafkaTopicStatus subsection called `replicasChange` is used to update the user, and keep track of the replicas change state.
The `replicasChange` can be in a pending or ongoing `state`, and it can contains a `message` in case of failure.
Other fields are `sessionId`, which maps to Cruise Control's `User-Task-ID`, and `replicas`, which reflects the target replicas value (this may be different from .spec.replicas when the state is ongoing).

- **Pending**: Not in Cruise Control's task queue (not yet sent or request error).
  Cruise Control's task states: None.
  ```sh
  status:
    conditions:
    - lastTransitionTime: "2024-01-18T16:13:50.490918232Z"
      status: "True"
      type: Ready
    observedGeneration: 2
    replicasChange:
      replicas: 2
      state: pending
    topicName: my-topic
  ```

- **Ongoing**: In Cruise Control's task queue, but execution not started, or not completed. 
  Cruise Control's task states: Active, InExecution.
  ```sh
  status:
    conditions:
    - lastTransitionTime: "2024-01-18T16:13:53.490918232Z"
      status: "True"
      type: Ready
    observedGeneration: 3
    replicasChange:
      replicas: 2
      sessionId: 1aa418ca-53ed-4b93-b0a4-58413c4fc0cb
      state: ongoing
    topicName: my-topic
  ```

- **Completed**: Cruise Control's task execution completed (target replication factor reconciled).
  Cruise Control's task states: Completed.
  ```sh
  status:
    conditions:
    - lastTransitionTime: "2024-01-18T16:13:58.490918232Z"
      status: "True"
      type: Ready
    observedGeneration: 4
    topicName: my-topic
  ```

- **Failed**: Cruise Control's request failed, the change remains pending, and it is periodically retried.
  Cruise Control's task states: CompletedWithError.
  ```sh
  status:
    conditions:
    - lastTransitionTime: "2024-01-18T16:13:58.490918232Z"
      status: "True"
      type: Ready
    observedGeneration: 4
    replicasChange:
      message: Change request failed, Cluster model not ready
      replicas: 2
      state: pending
    topicName: my-topic
  ```

### Reconciliation logic

At runtime, the Topic Operator watches and periodically reconciles the configuration of all managed and unpaused KafkaTopic resources.
Among the other configurations, the reconciliation logic detects `.spec.replicas` changes in KafkaTopic resources by comparing with the topic replication factor value in Kafka.

A ReplicasChangeClient is used to isolate the logic required to call the Cruise Control REST API from the rest of the reconciliation logic.
The ReplicasChangeClient supports sending multiple replicas changes with a single request, and checking multiple tasks with a single request.

On every reconciliation loop, when Cruise Control integration is enabled and replicas changes are detected, the BatchingTopicController uses the ReplicasChangeClient for the following operations:

1. Request pending replicas changes (uses `topic_configuration` endpoint, and returns topics with pending and ongoing changes).
2. Check ongoing replicas changes (uses `user_tasks` endpoint, and returns topics with completed changes).

The result of these operations is used to update the KafkaTopic status.

### Error handling

When there is an error, the replicas change remains pending or ongoing (no status update), and it is indefinitely retried by the periodic reconciliation.
All errors are written in the Topic Operator logs and in the KafkaTopic status.

In Cruise Control, there is no way to differentiate between temporary errors (resolve by themself) from permanent errors (require some action).
For example, the `NotEnoughValidWindowsException` can be raised when Cruise Control has just started and the cluster model is still building, but also when there is a configuration error which prevents broker metrics collection.
A similar example is the `OptimizationFailureException` raised when some hard goal is violated, which can be temporary if the configured network capacity is less than the real network capacity and there is a traffic peak.
Instead, the `OngoingExecutionException` is clearly temporary, and it means that a task execution was attempted while there is already one that is being executed by Cruise Control.
Another temporary example is the `RuntimeException` when Cruise Control's task queue is full, and the Topic Operator periodically retries until there is room for it.

Cruise Control allows to scale down the replication factor under the `min.insync.replicas` value, and this can cause disruption to producers with `acks=all`.
When this happens, the Topic Operator logs a warning but continues with the replicas change, because the KafkaRoller ignores topics with RF < minISR, and they don't even show up as under replicated in Kafka metrics.
The target replication factor should also be less than or equal to the number of available brokers, but this is enforced directly by Cruise Control.

When a managed KafkaTopic with replicas change is deleted in Kubernetes, the Topic Operator also deletes the topic in Kafka, but the task execution continues in Cruise Control.
There is little benefit in using the `stop_proposal_execution` endpoint, because most replicas change tasks will have only one batch that can't be stopped.
No task will remain hanging, the only problem is that Cruise Control does some unnecessary work executing a batch for a topic that has been deleted.

When a manged topic with replicas change is deleted in Kafka, the Topic Operator recreates the topic with the target replication factor.

When Topic Operator is restarted, both pending and ongoing replicas changes are recovered from KafkaTopic status.
The ongoing state includes the `sessionId`, which is the unique identifier of a Cruise Control task, which includes one or more replicas changes.

When Cruise Control is restarted, all active tasks are lost, as there is no persistent memory.
The Topic Operator detects this event when the ongoing replicas changes check returns no tasks for a known `sessionId`.
All pending and ongoing replicas changes are still stored in KafkaTopic status, and new tasks can be created as soon as Cruise Control becomes ready.

## Affected/not affected projects

The following components will be affected by this proposal:

- Topic Operator: this component will drive the whole replication factor change process.
- Cluster Operator: this component will be responsible to initialize the Cruise Control environment variables.

## Compatibility

There is no backward compatibility issue for this proposal.

## Rejected alternatives

- Use a ReplicasChangeManager object to drive the replicas change reconciliation, which is notified by the BatchingTopicController. 
  This makes the implementation way more complicated, as this new object needs to maintain an in-memory cache of ongoing changes.

---
