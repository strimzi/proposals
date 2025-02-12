# Adding progress updates for Cruise Control rebalances

This proposal introduces a new feature to monitor the progression of an ongoing partition rebalance executed by a Strimzi-managed Cruise Control instance via a `KafkaRebalance` custom resource. 

## Current Situation

At this time, Strimzi users are able to execute partition rebalances via `KafkaRebalance` custom resources but can only monitor the progression of those partition rebalances in two ways:

- Manually querying the Cruise Control REST API endpoint directly (which requires creating a custom Cruise Control REST API user or disabling HTTP Basic authentication).
- Inspecting the logs of the Cruise Control instance. 

Unfortunately, neither of these methods are particularly user-friendly.

## Motivation

Information surrounding the progress of an executing partition rebalance is useful for planning future cluster operations, such as worker node maintenance, scaling, or upgrading brokers.
Knowing details like the duration and data left to transfer for an ongoing partition rebalance can help users schedule maintenance windows effectively and access the impact of continuing versus canceling a rebalance.
While it is ideal to complete a rebalance before performing certain operations, such as draining nodes, having this information allows users to schedule these operations more optimally.

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
    type: ProposalReady | Rebalancing | Stopped | Ready
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
- **estimatedTimeToCompletionInMinutes**: The estimated amount time it will take in minutes until the partition rebalance is complete.
- **completedByteMovementPercentage**: The percentage of the byte movement of the partition rebalance that is completed as a rounded down integer value in the range [0-100].
- **executorState**: The “non-verbose” JSON payload from the [`/kafkacruisecontrol/state?substates=executor`](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint, providing details about the executor's current status, including partition movement progress, concurrency limits, and total data to move.

Since the progress information is constant, we can safely add it to the existing `ConfigMap` maintained for and tied to the `KafkaRebalance` resource.
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
  completedByteMovementPercentage: 80 [2]
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

[2] The percentage of byte movement of the partition balance that is complete as a rounded down integer in the range [0-100].

[3] The “non-verbose” JSON payload from [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint.

[4] The broker load from the optimization proposal as a JSON string that already maintained in the `ConfigMap`.

### Field values per `KafkaRebalance` State

We will provide the progress information for the `KafkaRebalance` states where it is relevant:


| State/Field     | `estimatedTimeToCompletionInMinutes`                                   | `completedByteMovementPercentage`                                                                                                    | `executorState`                                                                                                                                                                                      |
|-----------------|------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ProposalReady` | -                                                                      | 0                                                                                                                                    | -                                                                                                                                                                                                    |
| `Rebalancing`   | The estimated time it will take for the ongoing rebalance to complete. | The percentage of byte movement of the ongoing partition rebalance that is complete as a rounded down integer in the range  [0-100]. | JSON object from  [/kafkacruisecontrol/state?substates=executor] ( https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) endpoint of Cruise Control REST API. |
| `Stopped`       | -                                                                      | The value of `completedByteMovementPercentage` from previous update.                                                                 | JSON object from previous update.                                                                                                                                                                    |
| `NotReady`      | -                                                                      | The value of `completedByteMovementPercentage` from previous update.                                                                 | JSON object from previous update.                                                                                                                                                                    |
| `Ready`         | 0                                                                      | 100                                                                                                                                  | -                                                                                                                                                                                                    |                                                                                                                                            |   |

**Notes:**
- The "`-`" symbol indicates the states where the field will not be displayed.

All the information required for the Cluster Operator to estimate the values of `estimatedTimeToCompletionInMinutes` and `completedByteMovementPercentage` fields can be derived from the Cruise Control server configurations and the [/kafkacruisecontrol/state?substates=executor](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) REST API endpoint.
However, the actual formulas used to produce values for these fields depend on the state of the `KafkaRebalance` resource.
Checkout the example in the [Field: executorState](#field-executorstate) section to see where the fields used in the formulas below come from.

### Field: `estimatedTimeToCompletionInMinutes`

The estimated time it will take in minutes for a rebalance to complete.

The formulas used to calculate the field value differ for each applicable `KafkaRebalance` state:

#### State: `ProposalReady`

Since the rebalance has not started at this point, we do not have the data needed to easily provide an accurate value for this field.
Therefore, we omit this field in the `ConfigMap` of the `KafkaRebalance` resource for this state.

#### State: `Rebalancing`

The estimated time it will take for the ongoing rebalance to complete based on the average rate of data transfer.

$$
r = \frac{D_{moved}}{T_{\text{current}} - T_{\text{start}}}
$$

$$
ETC = \frac{D_{total} - D_{moved}}{r \cdot 60}
$$

**Notes:**
- $r$: The average rate of data transfer in megabytes per second, calculated as the ratio of finished data movement to the elapsed time.
- $D_{moved}$: The number of megabytes already moved by the rebalance, provided by `finishedDataMovement` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.
- $T_{\text{current}}$: The current time when the estimate is being calculated.
- $T_{\text{start}}$: The time when the rebalance task was triggered, extracted from the `triggeredTaskReason` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.
- $D_{total}$: The total number of megabytes planned to be moved for the rebalance, provided by `totalDataToMove` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.
- $ETC$: The estimated time to completion in minutes based on the average rate of data transfer, the value of the `estimatedTimeToCompletionInMinutes` field.

#### State: `Stopped`

Once a rebalance has been stopped, it cannot be resumed. 
Therefore, there is no `estimatedTimeToCompletionInMinutes` for a stopped rebalance so we omit this field in the `ConfigMap` of the `KafkaRebalance` resource for this state.
To move from the `Stopped` state, a user must refresh the `KafkaRebalance` resource, the `rebalanceProgressConfigMap` will then be updated by the operator upon the next state change.

#### State: `NotReady`

Once a rebalance has been interrupted or completed with errors, it cannot be resumed.
Therefore, there is no `estimatedTimeToCompletionInMinutes` for the rebalance so we omit this field in the `ConfigMap` of the `KafkaRebalance` resource for this state.
To move from the `NotReady` state, a user must refresh the `KafkaRebalance` resource, the `rebalanceProgressConfigMap` will then be updated by the operator upon the next state change.

#### State: `Ready`

The rebalance is complete so we hardcode the value to `0`
This emphasizes that the rebalance is complete and helps clear up ambiguity surrounding what the `Ready` state means in the `KafkaRebalance` resource.

### Field: `completedByteMovementPercentage`

The percentage of the byte movement of the partition rebalance that is completed as a rounded down integer in the range [0-100].

The formulas used to calculate field value per  `KafkaRebalance` state:

#### State: `ProposalReady`

The rebalance has not started yet so no data has been moved, so we hardcode the value to `0`.
This emphasizes that the rebalance has not started and helps clear up ambiguity surrounding what the `optimizationResult` field means in the `KafkaRebalance` resource.

#### State: `Rebalancing`

The percentage of the byte movement of the partition rebalance that is completed as a rounded down integer in the range [0-100].

$$
\text{DMP} = \left\lfloor \frac{D_{moved}}{D_{total}} \times 100 \right\rfloor
$$

**Notes:**
- $DMP$: The percentage of byte data that has been moved as a rounded down integer in the range [0-100], the value of the `completedByteMovementPercentage` field.
- $D_{moved}$: The number of megabytes already moved by the rebalance, provided by `finishedDataMovement` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.
- $D_{total}$: The total number of megabytes planned to be moved for the rebalance, provided by `totalDataToMove` field from the [/kafkacruisecontrol/state?substates=executor](#field-executorstate) REST API endpoint.

#### State: `Stopped`

Once a rebalance has been stopped, it cannot be resumed. 
However, the `completedByteMovementPercentage` information from when before the rebalance was stopped may still be of value to users. 
Therefore, we reuse the same value of `completedByteMovementPercentage` from the previous update.

#### State: `NotReady`

Once a rebalance has been interrupted or completed with errors, it cannot be resumed.
However, the `completedByteMovementPercentage` information from when before the rebalance was interrupted or completed may still be of value to users.
Therefore, we reuse the same value of `completedByteMovementPercentage` from the previous update.

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

#### State: `ProposalReady`

The rebalance has not started yet so no data surrounding partition movement is available yet, so we omit this field in the `ConfigMap` of the `KafkaRebalance` resource for this state.

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

The rebalance is complete in this state so there is no data surrounding ongoing partition movement available, so we omit this field in the `ConfigMap` of the `KafkaRebalance` resource for this state.

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
    rebalanceProgressConfigMap: my-rebalance
```
[1] Error message from failed Cruise Control REST API call.

### Accessing progress fields using Kubernetes CLI

The progress information will be stored in a `ConfigMap` with the same name as the `KafkaRebalance` resource.
Using the name of the `ConfigMap` we can view its data from the command line using the Kubernetes CLI.

Example accessing `estimatedTimeToCompletionInMinutes` field.
```shell
kubectl get configmaps <my_rebalance_configmap_name> -o json | jq '.["data"]["estimatedTimeToCompletionInMinutes"]'
```

Example accessing `completedByteMovementPercentage` field.
```shell
kubectl get configmaps <my_rebalance_configmap_name> -o json | jq '.["data"]["completedByteMovementPercentage"]'
```

Example accessing `executorState` field.
```shell
kubectl get configmaps <my_rebalance_configmap_name> -o json | jq '.["data"]["executorState"] | fromjson | .'
```

### Affected/not affected projects

This change will affect the Strimzi Cluster Operators project and mostly the `KafkaRebalanceAssemblyOperator` class in the `cluster-operator` module.

### Future Improvements

#### Providing theoretical minimum estimate for `estimatedTimeToCompletionInMinutes` field for `ProposalReady` state

In addition to having an estimate of how long a rebalance will take _during_ a rebalance, it would also be useful to have a theoretical minimum estimate of how long a rebalance would take _before_ a rebalance has started.
However, providing such estimation is non-trivial.
It would be better addressed in its own proposal where we can have deeper discussion and analysis.

Leveraging the Cruise Control configurations and user-provided network capacity settings, we could provide a rough estimate for the `estimatedTimeToCompletion` field but there are a couple of challenges.
For inter-broker rebalances, the accuracy of the estimation is dependent on the accuracy of the user-defined network capacity settings.
For intra-broker rebalances, the accuracy of the estimation is dependent on broker disk read/write bandwidth.
These are only a couple of the challenges and complications of providing such an estimate, in a future proposal we can discuss in more detail.

** Rough estimation for inter-broker rebalances:**

We could only provide an accurate estimate for inter-broker rebalances with an accurate estimate of the network capacity of the brokers.
Therefore, the calculation would be dependent on how well users configure their network capacity settings.

$$
ETC = \frac{D_{total} \cdot 1024 \cdot 1024}{\min\left(NB, R_t\right) \cdot 60}
$$

Notes:
- $ETC$: Estimated time to completion in minutes, the value of the `estimatedTimeToCompletionInMinutes` field.
- $D_{total}$: The total number of megabytes planned to be moved for the rebalance, provided by `dataToMoveMB` field in `status.optimizationResult` section of `KafkaRebalance` resource.
- $NB$: The average network bandwidth per broker in the cluster in bytes per second, derived from the `spec.cruiseControl.brokerCapacity.inboundNetwork` and `spec.cruiseControl.brokerCapacity.outboundNetwork` fields in the `Kafka` custom resource.
- $R_t$: The replication throttle applied to replicas being moved in and out per broker in bytes per second, provided by Cruise Control  [`default.replication.throttle`](https://github.com/linkedin/cruise-control/wiki/Configurations) configuration.

** Rough estimation for intra-broker rebalances:**

It is challenging to provide an accurate estimate for intra-broker rebalances without an estimate for disk read/write bandwidth and getting disk bandwidth is non-trivial for Strimzi.

$$ETC = \frac{D_{total} \cdot 1024 \cdot 1024}{\text{DB} \cdot 60}$$

Notes
- $ETC$: The estimated time to completion in minutes, the value of the `estimatedTimeToCompletionInMinutes` field.
- $D_{total}$: The total number of megabytes planned to be moved for the rebalance, provided by `dataToMoveMB` field in `status.optimizationResult` section of `KafkaRebalance` resource.
- $DB$: The estimate disk read/write bandwidth in bytes per second, however, we don't have a method of providing this value.

### Rejected Alternatives

#### Maintaining progress fields in `KafkaRebalance` resource status

Given that the progress information included in a `KafkaRebalance` resource: 
- Contains information irrelevant to user driven partition rebalances (e.g. triggeredSelfHealingTaskId, triggeredTaskReason)
- Can be quite verbose (e.g. pendingPartitionMovement list)
- Requires tracking a "last updated" timestamp and special logic in the rebalance operator to avoid triggering recursive operator reconciliations

it is best if we maintain the progress information somewhere else.
