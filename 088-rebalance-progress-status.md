# Adding progress updates for Cruise Control rebalances

This proposal introduces a new feature to monitor the progression of an ongoing partition rebalance executed by a Strimzi-managed Cruise Control instance via a `KafkaRebalance` custom resource. 

## Current Situation

At this time, Strimzi users are able to execute partition rebalances via `KafkaRebalance` custom resources but can only monitor the progression of those partition rebalances in two ways:

- Manually querying the Cruise Control REST API endpoint directly (which requires creating a custom Cruise Control REST API user or disabling HTTP Basic authentication).
- Inspecting the logs of the Cruise Control instance. 

Unfortunately, neither of these methods are particularly user-friendly.

## Motivation

Information surrounding the progress of an executing partition rebalance is useful for planning future cluster operations. 
Knowing things like how much time an ongoing partition rebalance has left to take and how much data an ongoing partition rebalance has left to transfer helps users understand the cost of an ongoing partition rebalance.
This information helps users decide whether they should continue or cancel an ongoing rebalance, and keep client teams informed on the expected duration of the rebalance.

Further, having this information readily available and easily accessible via Kubernetes primitives, allows users and third-party tools like the Kubernetes CLI or 3rd party tools to easily track the progression of a partition rebalance.

## Proposal
Currently, the `KafkaRebalance` custom resource references a `ConfigMap` in the `status.optimizationResult.afterBeforeLoadConfigMap` field which stores the estimated load on the brokers before and after the optimization proposal is executed.
This proposal extends the status section of the `KafkaRebalance` custom resource to include a `progress` section with a nested `rebalanceProgressConfigMap` field to reference that same `ConfigMap`.
This `ConfigMap` will also be enhanced to contain information related to an ongoing partition rebalance.

The `progress` section of the `KafkaRebalance` resource will look like the following:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
spec: {}
status:
  conditions:
  - lastTransitionTime: "2024-11-05T15:28:23.995129903Z"
    status: "True"
    type: ProposalReady | Rebalancing | Stopped | Ready [1]
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
    rebalanceProgressConfigMap: my-rebalance [2]
```
[1] The `progress` section will be visible during the `ProposalReady`, `Rebalancing`, `Stopped`, `NotReady` and `Ready` states.

[2] The `ConfigMap` containing information related to the ongoing partition rebalance.

In the `ConfigMap`, we will add the following fields:
- **estimatedTimeToCompletionInMinutes**: The estimated amount time it will take in minutes until partition rebalance is complete.
- **completedDataMovementPercentage**: The percentage of the data movement of the partition rebalance that is completed e.g. rounded down integer values in the range [0-100].
- **executorState**: The “non-verbose” JSON payload from the [`/kafkacruisecontrol/state?substates=executor`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint, providing details about the executor's current status, including partition movement progress, concurrency limits, and total data to move.

Since the progress information is constant, we can safely add it to the existing `ConfigMap` maintained for and tied to the `KafkadRebalance` resource.
This keeps `KafkaRebalance` information organized in one place, simplifies the proposal implementation, and has insignificant impact on the storage of the `ConfigMap`.

The enhanced `ConfigMap` of an inter-broker partition rebalance will look like the following:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-rebalance
  …
data:
  estimatedTimeToCompletionInMinutes: 5 [1]
  completedDataMovementPercentage: 80 [2]
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
  brokerLoad.json: {...} [4]
```
[1] The estimated time it will take in minutes for the rebalance to complete based on the average rate of data transfer.

[2] The percentage complete of the ongoing rebalance in the range [0-100].

[3] The “non-verbose” JSON payload from [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint.

[4] The broker load from the optimization proposal as a JSON string that already maintained in the `ConfigMap`.

### Field values per `KafkaRebalance` State

We will provide the progress information for the `KafkaRebalance` states where it is relevant:

- `ProposalReady`:
  - `estimatedTimeToCompletionInMinutes`: The theoretical minimum estimated time it would take the rebalance to complete if executed.
  - `completedDataMovementPercentage`: 0
  - `executorState`: Empty JSON object.
- `Rebalancing`:
  - `estimatedTimeToCompletionInMinutes`: The estimated time it will take for the ongoing rebalance to complete.
  - `completedDataMovementPercentage`: The percentage of data movement of the ongoing partition rebalance that is complete.
  - `executorState`: JSON object from [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint of Cruise Control REST API.
- `Stopped`:
  - `estimatedTimeToCompletionInMinutes`: "N/A" [1]
  - `completedDataMovementPercentage`: The value of `completedDataMovementPercentage` from previous update.
  - `executorState`: JSON object from previous update.
- `NotReady`:
  - `estimatedTimeToCompletionInMinutes`: "N/A" [1]
  - `completedDataMovementPercentage`: The value of `completedDataMovementPercentage` from previous update.
  - `executorState`: JSON object from previous update.
- `Ready`:
  - `estimatedTimeToCompletionInMinutes`: 0
  - `completedDataMovementPercentage`: 100
  - `executorState`: Empty JSON object.

Notes:
- [1] The abbreviation "N/A" stands for ""Not Available" or "Not Applicable".

All the information required for the Cluster Operator to estimate the values of `estimatedTimeToCompletionInMinutes` and `completedDataMovementPercentage` fields can be derived from the Cruise Control server configurations and the [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) REST API endpoint.
However, the actual formulas used to produce values for these fields depend on the state of the `KafkaRebalance` resource.
Checkout the example in the [Field: executorState](#field-executorstate) section to see where the fields used in the formulas below come from.

### Field: `estimatedTimeToCompletionInMinutes`

The estimated time it will take in minutes for a rebalance to complete.

The formulas used to calculate the field value differ for each applicable `KafkaRebalance` state:

#### State: `ProposalReady`

**Estimation for inter-broker rebalance:**

We can only provide an accurate estimate for inter-broker rebalances with an accurate estimate of the network capacity of the brokers.
Therefore, when users do not explicitly set the network capacity settings, `spec.cruiseControl.brokerCapacity.inboundNetwork` and `spec.cruiseControl.brokerCapacity.outboundNetwork` of the `Kafka` resource, `estimatedTimeToCompletionInMinutes` will be "not available" and set `N/A`:

When network capacity values are provided by the user the value of `estimatedTimeToCompletionInMinutes` will be a theoretical minimum derived from these capacity and throttle configurations.
This means that the cluster rebalance would take at least the estimated amount of time to complete.

$$\text{maxConcurrentPartitionMovements}_{[1]} = \min((\text{numberOfBrokers} \times \text{num.concurrent.partition.movements.per.broker}), \text{max.num.cluster.partition.movements})$$

$$\text{bandwidth}_{[2]} = \min(\frac{\text{networkCapacity}}{\text{maxConcurrentPartitionMovements}}, \text{replication.throttle})$$

$$\text{throughput}_{[3]} = \text{maxConcurrentPartitionMovements} \times \text{bandwidth}$$

$$\text{estimatedTimeToCompletionInMinutes} = \frac{\text{dataToMoveMB}}{\text{throughput}}$$

When network capacity configurations are not explicitly by the user, we cannot ensure an accurate value for `estimatedTimeToCompletionInMinutes`, so it will be set to `N/A`, an abbreviation meaning "Not Available" or "Not Applicable".

Notes:
- [1] The maximum number of concurrent partition movements given Cruise Control partition movement capacity.

- [2] The network bandwidth given Cruise Control bandwidth throttle.

- [3] The throughput given the max allowed number of concurrent partition movements and network bandwidth.

**Estimation for intra-broker rebalance:**

It is challenging to provide an accurate estimate for intra-broker rebalances without an estimate for disk read/write throughput and getting disk throughput is non-trivial for Strimzi.
Since we cannot accurately estimate `estimatedTimeToCompletionInMinutes` without knowing the disk throughput, we set `estimatedTimeToCompletionInMinutes` to `N/A`, an abbreviation meaning "Not Available" or "Not Applicable".

$$\text{maxPartitionMovements}_{[1]} = \min((\text{numberOfBrokers} \times \text{num.concurrent.intra.broker.partition.movements.per.broker}),\text{max.num.cluster.movements})$$

$$\text{estimatedDiskThroughput} = \text{???}_{[2]}$$

$$\text{throughput}_{[3]} = \text{maxPartitionMovements} \times  \text{estimatedDiskThroughput}
$$

$$\text{estimatedTimeToCompletionInMinutes} = \frac{\text{intraBrokerDataToMoveMB}}{\text{throughput}}$$

Since `estimatedDiskThroughput` is unavailable, the formula for `throughput` cannot be resolved, making `estimatedTimeToCompletionInMinutes` undefined or not applicable, `N/A`.

Notes
- [1] The maximum number of concurrent partition movements given Cruise Control partition movement capacity.
- [2] We don't have a method of determining disk throughput.
- [3] The throughput given the max allowed number of concurrent partition movements and disk throughput.
 
#### State: `Rebalancing`

The estimated time it will take for the ongoing rebalance to complete based on the average rate of data transfer.

$$
\text{rate} = \frac{\text{finishedDataMovement}\_{[1]}}{\text{currentTime} - \text{taskTriggerTime}\_{[2]}}
$$

$$
\text{estimatedTimeToCompletionInMinutes} = \frac{\text{totalDataToMove}\_{[3]} - \text{finishedDataMovement}\_{[1]}}{\text{rate}}
$$

Notes
- [1] The number of megabytes already moved by rebalance, provided by `finishedDataMovement` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.
- [2] The time when the rebalance task was started, extracted from the `triggeredTaskReason` field for that task from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.
- [3] The total number of megabytes planned to be moved for rebalance, provided by the `totalDataToMove` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.

#### State: `Stopped`

Once a rebalance has been stopped, it cannot be resumed. 
Therefore, there is no `estimatedTimeToCompletionInMinutes` for a stopped rebalance. We set the field to `N/A` to emphasize this. 
To move from the `Stopped` state, a user must refresh the `KafkaRebalance` resource, the `rebalanceProgressConfigMap` will then be updated by the operator upon the next state change.

#### State: `NotReady`

Once a rebalance has been interrupted or completed with errors, it cannot be resumed.
Therefore, we set `estimatedTimeToCompletionInMinutes` to `N/A` to emphasize this.
To move from the `NotReady` state, a user must refresh the `KafkaRebalance` resource, the `rebalanceProgressConfigMap` will then be updated by the operator upon the next state change.

#### State: `Ready`

The rebalance is complete so we hardcode the value to `0`
This emphasizes that the rebalance is complete and helps clear up ambiguity surrounding what the `Ready` state means in the `KafkaRebalance` resource.

### Field: `completedDataMovementPercentage`

The percentage of the data movement of the partition rebalance that is completed.

The formulas used to calculate field value per  `KafkaRebalance` state:

#### State: `ProposalReady`

The rebalance has not started yet so no data has been moved, so we hardcode the value to `0`.
This emphasizes that the rebalance has not started and helps clear up ambiguity surrounding what the `optimizationResult` field means in the `KafkaRebalance` resource.

#### State: `Rebalancing`

The percentage of the data movement of the partition rebalance that is completed.

$$
\text{completedDataMovementPercentage} = (\frac{\text{finishedDataMovement}\_{[1]}}{\text{totalDataToMove}\_{[2]}} \times 100)
$$

Notes
 - [1] The number of megabytes already moved by rebalance, provided by `finishedDataMovement` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.
 - [2] The total number of megabytes planned to be moved for rebalance, provided by `totalDataToMove` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.

#### State: `Stopped`

Once a rebalance has been stopped, it cannot be resumed. 
However, the `completedDataMovementPercentage` information from when before the rebalance was stopped may still be of value to users. 
Therefore, we reuse the same value of `completedDataMovementPercentage` from the previous update.

#### State: `NotReady`

Once a rebalance has been interrupted or completed with errors, it cannot be resumed.
However, the `completedDataMovementPercentage` information from when before the rebalance was interrupted or completed may still be of value to users.
Therefore, we reuse the same value of `completedDataMovementPercentage` from the previous update.

#### State: `Ready`

The rebalance is complete so we hardcode the value to `100`.
This emphasizes that the rebalance is complete and helps clear up ambiguity surrounding what the `Ready` state means in the `KafkaRebalance` resource.

### Field: `executorState`

The "non-verbose" JSON payload of the [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint of the Cruise Control REST API.

For determining which fields are included for different executor states (`NO_TASK_IN_PROGRESS`, `STARTING_EXECUTION`, `INTER_BROKER_REPLICA_MOVEMENT_TASK_IN_PROGRESS` etc), refer to the code [here](https://github.com/linkedin/cruise-control/blob/2.5.141/cruise-control/src/main/java/com/linkedin/kafka/cruisecontrol/executor/ExecutorState.java#L427).
For an exhaustive list of all the fields that could be included, see the Cruise Control’s OpenAPI spec [here](https://github.com/linkedin/cruise-control/blob/2.5.141/cruise-control/src/main/resources/yaml/responses/executorState.yaml)

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

The formulas used to calculate the field value differ for each applicable `KafkaRebalance` state:

#### State: `Ready`

The rebalance has not started yet so no data surrounding partition movement is available yet, so we set the field to an empty JSON object.

#### State: `Rebalancing`

The field will be assigned the "non-verbose" JSON payload of the [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint of the Cruise Control REST API.

#### State: `Stopped`

Once a rebalance has been stopped, it cannot be resumed.
However, the `executorState` information from when before the rebalance was stopped may still be of value to users.
Therefore, we reuse the same value of `executorState` from the previous update.

#### State: `NotReady`

Once a rebalance has been interrupted or completed with errors, it cannot be resumed.
However, the `executorState` information from when before the rebalance was interrupted or completed may still be of value to users.
Therefore, we reuse the same value of `executorState` from the previous update.

#### State: `Ready`

The rebalance is complete in this state so we hardcode the value to an empty JSON object.

### Progress Update Cadence

For ease of implementation and minimizing the load on the CruiseControl REST API server, the operator will only query the `/kafkacruisecontrol/state?substates=executor` endpoint and update the `ConfigMap` upon `KafkaRebalance` resource reconciliation.

In the event that Cruise Control runs into an error when rebalancing, when operator transitions the `KafkaRebalance` resource to the `NotReady` state, it will leave the `progress` section and `ConfigMap` as is.
In the event that the Cruise Control REST API returns an error or fails to respond to the operator when querying the `/kafkacruisecontrol/state?substates=executor` endpoint during a rebalance, the operator will add a "Warning" condition with error message from the failed REST API call but leave the existing progress section and progress `ConfigMap` as is.

When Cruise Control state retrieval fails, the `KafkaRebalance` resource will be updated as follows:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
  ...
spec: {}
status:
  conditions:
  - lastTransitionTime: "2024-11-05T15:28:23.995129903Z"
    status: "True"
    type: Rebalancing
  - lastTransitionTime: "2024-11-05T15:28:23.995129903Z"
    status: "True"
    type: Warning
    reason: CruiseControlRestException
    message: <error_message> [1]  
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
[1] Error message from failed Cruise Control REST API call.

### Accessing progress fields using Kubernetes CLI

The progress information will be stored in a `ConfigMap` with the same name as the `KafkaRebalance` resource.
Using the name of the `ConfigMap` we can view its data from the command line using the Kubernetes CLI.

Example accessing `estimatedTimeToCompletionInMinutes` field.
```shell
kubectl get configmaps <my_rebalance_configmap_name> -o json | jq '.["data"]["estimatedTimeToCompletionInMinutes"]'
```

Example accessing `completedDataMovementPercentage` field.
```shell
kubectl get configmaps <my_rebalance_configmap_name> -o json | jq '.["data"]["completedDataMovementPercentage"]'
```

Example accessing `executorState` field.
```shell
kubectl get configmaps <my_rebalance_configmap_name> -o json | jq '.["data"]["executorState"] | fromjson | .'
```

### Affected/not affected projects

This change will affect the Strimzi Cluster Operators project and mostly the `KafkaRebalanceAssemblyOperator` class in the `cluster-operator` module.

### Rejected Alternatives

#### Maintaining progress fields in `KafkaRebalance` resource status

Given that the progress information included in a `KafkaRebalance` resource: 
- Contains information irrelevant to user driven partition rebalances (e.g. triggeredSelfHealingTaskId, triggeredTaskReason)
- Can be quite verbose (e.g. pendingPartitionMovement list)
- Requires tracking a "last updated" timestamp and special logic in the rebalance operator to avoid triggering recursive operator reconciliations

it is best if we maintain the progress information somewhere else.
