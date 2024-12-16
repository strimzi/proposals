# Partition Rebalance Progress Status

This proposal introduces a new feature to monitor the progression of an ongoing partition rebalance executed by a Strimzi-managed Cruise Control instance via a `KafkaRebalance` custom resource. 

## Current Situation

At this time, Strimzi users are able to execute partition rebalances via `KafkaRebalance` custom resources but can only monitor the progression of those partition rebalances in two ways:

- Manually querying the Cruise Control REST API endpoint directly. 
- Inspecting the logs of the Cruise Control instance. 

Unfortunately, neither of these methods are particularly user-friendly.

## Motivation

Information surrounding the progress of an executing partition rebalance is useful for planning future cluster operations. 
Knowing things like how much time an ongoing partition rebalance has left to take and how much data an ongoing partition rebalance has left to transfer helps users understand the cost of an ongoing partition rebalance.
This information helps users decide whether they should continue or cancel an ongoing rebalance, and know when future operations will be able to be safely executed.

Further, having this information readily available and easily accessible via Kubernetes primitives, allows users and third-party tools like the Kubernetes CLI or Strimzi Console to easily track the progression of a partition rebalance.

## Proposal

This proposal extends the status section of the `KafkaRebalance` custom resource to include a `progress` section with a nested `rebalanceProgressConfigMap` field that references a `ConfigMap` that contains information related to an ongoing partition rebalance.

The `progress` section of the `KafkaRebalance` resource would look like the following:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
spec: {}
status:
  conditions:
  - lastTransitionTime: "2024-11-05T15:28:23.995129903Z"
    status: "True"
    type: Rebalancing | Stopped [1]
  observedGeneration: 1
  optimizationResult:
    afterBeforeLoadConfigMap: my-rebalance
    dataToMoveMB: 0
    excludedBrokersForLeadership: []
    excludedBrokersForReplicaMove: []
    excludedTopics: []
    intraBrokerDataToMoveMB: 0
    monitoredPartitionsPercentage: 100
    numIntraBrokerReplicaMovements: 0
    numLeaderMovements: 16
    numReplicaMovements: 0
    onDemandBalancednessScoreAfter: 95.4347095948149
    onDemandBalancednessScoreBefore: 89.4347095948149
    provisionRecommendation: ""
    provisionStatus: RIGHT_SIZED
    recentWindows: 1
  progress: [1]
    rebalanceProgressConfigMap: my-rebalance-progress [2]
```
[1] The `progress` section will be visible during the KafkaRebalance `Rebalancing` and  `Stopped` states.

[2] The `ConfigMap` containing information related to the ongoing partition rebalance, generated with the name "<kafka_rebalance_resource_name>-progress".

In the `ConfigMap`, we will include the following fields:
- **estimatedTimeToCompletion**: The estimated amount time it will take in minutes until partition rebalance is complete.
- **percentageDataMovementComplete**: The percentage of the data movement of the partition rebalance that is completed e.g. values in the range [0-100]%
- **executorState**: The “non-verbose” JSON payload from the `/kafkacruisecontrol/state?substates=executor` endpoint.

The progress `ConfigMap` of an inter-broker partition rebalance would look like the following:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-rebalance-progress
  …
data:
  estimatedTimeToCompletion: 5m [1]
  percentageDataMovementComplete: 80% [2]
  executorState: [3]
  {
   "abortingPartitions":0,
   "averageConcurrentPartitionMovementsPerBroker":5,
   "finishedDataMovement":0,
   "maximumConcurrentPartitionMovementsPerBroker":5,
   "minimumConcurrentPartitionMovementsPerBroker":5,
   "numFinishedPartitionMovements":0,
   "numInProgressPartitionMovements":0,
   "numPendingPartitionMovements":20,
   "numTotalPartitionMovements":20,
   "state":"INTER_BROKER_REPLICA_MOVEMENT_TASK_IN_PROGRESS",
   "totalDataToMove":0,
   "triggeredSelfHealingTaskId":"",
   "triggeredTaskReason":"No reason provided (Client: 172.17.0.1, Date: 2024-11-15T19:41:27Z)",
   "triggeredUserTaskId":"0230d401-6a36-430e-9858-fac8f2edde93"
  }

```
[1] The estimated time it will take the rebalance to complete based on the average rate of data transfer.

[2] The percentage complete of the ongoing rebalance in the range [0-100]%

[3] The “non-verbose” JSON payload from [/kafkacruisecontrol/state?substates=executor](#executor-state) endpoint.

### Field values per `KafkaRebalance` State

For the initial implementation, we will focus on providing the progress information in the following `KafkaRebalance` states:

- `Rebalancing`
- `Stopped`

These are the states where this progress information will be able to be most accurately calculated and most useful for users. 
We could provide this information for other states as well, such as the `ProposalReady` and `Ready` states, but it is not completely necessary, nor is it trivial. 
Further discussion on the inclusion of the progress information for these other states can be found in the [Future Improvements](#future-improvements) section near the bottom of this proposal.

All the information required for estimating the values of `estimatedTimeToCompletion` and `percentageDataMovementComplete` fields can be derived from either the Cruise Control server configurations or the [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) REST API endpoint.
However, the actual formula used to produce values for these fields depends on the state of the `KafkaRebalance` resource.
Checkout the example in the [Executor State](#executorState) section to see where the fields used in the formulas below come from.

#### estimatedTimeToCompletion
 
##### Rebalancing

$$
\text{rate} = \frac{\text{finishedDataMovement}}{\text{taskTriggerTime}^{[1]} - \text{currentTime}}
$$

$$
\text{estimatedTimeToCompletion} = \frac{\text{totalDataToMove} - \text{finishedDataMovement}}{\text{rate}}
$$

[1] `taskTriggerTime` is the time when the rebalance task was started, extracted from `triggeredTaskReason` field from the [Executor State](#executorState) for that task.

##### Stopped

Once a rebalance has been stopped, it cannot be resumed. 
Therefore, there is no `estimatedTimeToCompletion` for a stopped rebalance, so we set the field to `N/A` to emphasize this. 
To move from the `Stopped` state, a user must refresh the `KafkaRebalance` resource, the `rebalanceProgressConfigMap` will then be updated with the next state change.

$$
\text{estimatedTimeToCompletion} = \text{N/A}
$$

#### percentageDataMovementComplete

##### Rebalancing

$$
\text{percentageDataMovementComplete} = (\frac{\text{finishedDataMovement}}{\text{totalDataToMoveMB}} \times 100)\text{%}
$$

##### Stopped

Once a rebalance has been stopped, it cannot be resumed. 
However, the `percentageDataMovementComplete` information from when before the rebalance was stopped may still be of value to users. 
Therefore, we reuse the same value of `percentageDataMovementComplete` from the latest update.

#### executorState

##### Rebalancing

The `executorState` field will be populated with the "non-verbose" JSON payload of the [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint of the Cruise Control REST API.

Example of the Executor State JSON payload during an inter-broker balance:
```json
{
  "abortingPartitions": 0,
  "averageConcurrentPartitionMovementsPerBroker": 5,
  "finishedDataMovement": 0,
  "maximumConcurrentPartitionMovementsPerBroker": 5,
  "minimumConcurrentPartitionMovementsPerBroker": 5,
  "numFinishedPartitionMovements": 0,
  "numInProgressPartitionMovements": 0,
  "numPendingPartitionMovements": 20,
  "numTotalPartitionMovements": 20,
  "state": "INTER_BROKER_REPLICA_MOVEMENT_TASK_IN_PROGRESS",
  "totalDataToMove": 0,
  "triggeredSelfHealingTaskId": "",
  "triggeredTaskReason": "No reason provided (Client: 172.17.0.1, Date: 2024-11-15T19:41:27Z)",
  "triggeredUserTaskId": "0230d401-6a36-430e-9858-fac8f2edde93"
}
```
For determining which fields are included for different executor states (`NO_TASK_IN_PROGRESS`, `STARTING_EXECUTION`, `INTER_BROKER_REPLICA_MOVEMENT_TASK_IN_PROGRESS` etc) and whether the verbose parameter is provided or not, refer to the code [here](https://github.com/linkedin/cruise-control/blob/2.5.141/cruise-control/src/main/java/com/linkedin/kafka/cruisecontrol/executor/ExecutorState.java#L427).

For an exhaustive list of all the fields that could be included, see the Cruise Control’s OpenAPI spec [here](https://github.com/linkedin/cruise-control/blob/2.5.141/cruise-control/src/main/resources/yaml/responses/executorState.yaml)

##### Stopped

Once a rebalance has been stopped, it cannot be resumed.
However, the `executorState` information from when before the rebalance was stopped may still be of value to users.
Therefore, we reuse the same value of `executorState` from the latest update.

### Progress Update Cadence

For ease of implementation and minimizing the load on the CruiseControl REST API server, the operator will only query the `/kafkacruisecontrol/state?substates=executor` endpoint and update the progress `ConfigMap` upon `KafkaRebalance` resource reconciliation.

In the event that Cruise Control runs into an error when rebalancing, the operator will transition the `KafkaRebalance` resource to the `NotReady` state, remove the `progress` section, and delete the progress `ConfigMap`.
In the event that the Cruise Control REST API returns an error or fails to respond to the operator when querying the `/kafkacruisecontrol/state?substates=executor` endpoint during a rebalance, the operator will add a condition entry for the `Rebalancing` type with the message "Failed to retrieve rebalance progress" but leave the progress section and referenced progress `ConfigMap` as is.

When Cruise Control state retrievel failed, the `KafkaRebalance` resource would be updated like this:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
spec: {}
status:
  conditions:
  - lastTransitionTime: "2024-11-05T15:28:23.995129903Z"
    status: "True"
    type: Rebalancing
    message: "Failed to retrieve rebalance progress"   
  observedGeneration: 1
  optimizationResult:
    afterBeforeLoadConfigMap: my-rebalance
    dataToMoveMB: 0
    excludedBrokersForLeadership: []
    excludedBrokersForReplicaMove: []
    excludedTopics: []
    intraBrokerDataToMoveMB: 0
    monitoredPartitionsPercentage: 100
    numIntraBrokerReplicaMovements: 0
    numLeaderMovements: 16
    numReplicaMovements: 0
    onDemandBalancednessScoreAfter: 95.4347095948149
    onDemandBalancednessScoreBefore: 89.4347095948149
    provisionRecommendation: ""
    provisionStatus: RIGHT_SIZED
    recentWindows: 1
  progress:
    rebalanceProgressConfigMap: my-rebalance-progress
```

### Future Improvements

#### Support progress information for other KafkaRebalance states

In addition to the “progress” the `Rebalancing` and `Stopped` KafkaRebalance states, we could provide the `progress` section and `ConfigMap` for other states as well such as the `ProposalReady` and `Ready` states. 
Firstly, this would help emphasize that a rebalance had not started or had completed by having a `percentageDataMovementComplete`: 0% on `ProposalReady` and a `percentageDataMovementComplete`: 100% on `Ready`. 
This emphasis could help clear up ambiguity surrounding what the KafkaRebalance `Ready` state or `optimizationResult` field means. 
Secondly, but more importantly, it would provide an estimate for the minimum time a partition rebalance proposal would take to execute before even executing it. 
This feature would be of great value to users. 
However, providing an accurate estimation for this is non-trivial, namely the `estimatedTimeToCompletion` field for `ProposalReady` state, is non-trivial. 

Leveraging the Cruise Control configurations and user-provided network capacity settings, we could provide a rough estimate for `estimatedTimeToCompletetion` field for inter-broker movements. 
However, one challenge is coming up with a method of reliably determining a reasonable estimate for the disk read/write throughput. 
It is not so much of an issue for inter-broker rebalance estimates (assuming network is the bottleneck for inter-broker balances) but is certainly an issue for intra-broker rebalance estimates. 

Estimation for inter-broker partition rebalance time:

The maximum number of partition movements given CC partition movement cap

$$
\text{maxPartitionMovements} = \min\text{numberOfBrokers} \times \text{num.concurrent.partition.movements.per.broker}),\text{max.num.cluster.partition.movements})
$$

The network bandwidth given CC bandwidth throttle

$$
\text{bandwidth} = \min(\text{networkCapacity}, \text{replication.throttle})
$$

The throughput given the max allowed number of partition movements and network bandwidth

$$
\text{throughput} = \text{maxPartitionMovements} \times \text{bandwidth}
$$

$$
\text{estimatedTimeToCompletion} = \frac{\text{dataToMoveMB}}{\text{throughput}}
$$

However, without an estimate for disk read/write throughput, it is challenging to provide an accurate estimate for intra-broker rebalances but as mentioned above, getting disk throughput is non-trivial for Strimzi. 
We would either need some estimation of the disk throughput, make it user configurable, or hardcode the value ourselves.

The maximum number of partition movements given CC partition movement cap

$$
\text{maxPartitionMovements} = \min\left(\text{numberOfBrokers} \times \text{num.concurrent.intra.broker.partition.movements.per.broker}),\text{max.num.cluster.movements}\right)
$$

$$
\text{estimatedDiskThroughput} = \text{???}
$$

The throughput given the max allowed number of partition movements and disk throughput

$$
\text{throughput} = \text{maxPartitionMovements} \times \text{estimatedDiskThroughput}
$$

$$
\text{estimatedTimeToCompletion} = \frac{\text{intraBrokerDataToMoveMB}}{\text{throughput}}
$$


Given that its inclusion is not completely necessary and adds significant complexity to the proposal, it is out of scope for this proposal.

### Rejected Alternatives

#### Including “ExecutorState” in KafkaRebalance resource status

Given that some of the information in the Executor State is not relevant to user driven partition rebalances (e.g. triggeredSelfHealingTaskId and triggeredTaskReason) and can be quite verbose (e.g. pendingPartitionMovement list), it is best if we take what we take the high level details we need from the ExecutorState and store the rest somewhere else. 

#### Including “ExecutorState” in “afterBeforeLoadConfigmap”

Keeping the ExecutorState in its own ConfigMap as opposed to storing it in the existing “afterBeforeLoadConfigMap” (1) leaves more room for Executor state information should we decide to enable “verbosity” parameter in the future and (2) leaves more room for the broker load information in the “afterBeforeLoadConfigMap”. 
For smaller clusters, the space is not an issue but for larger production clusters with a larger number of brokers and partitions we run the risk of hitting the 1MB storage limit of the ConfigMap. 
The cost of another ConfigMap is worth avoiding the risk of hitting the limit of the other.

