# Auto-rebalance on imbalanced clusters

This proposal is about adding support for auto-rebalancing the Kafka cluster in case it gets imbalanced due to some issues like unevenly distributed replicas or overloaded brokers e.t.c.
When enabled, the Strimzi operator should automatically resolve these issues detected by the Anomaly Detector Manager by running KafkaRebalance via Cruise Control using the KafkaRebalance resource.
Anomalies are detected by Cruise Control using the anomaly detector manager (see section [ Anomaly Detector Manager](./106-auto-rebalance-on-imbalanced-clusters.md#anomaly-detector-manager) below for a detailed description).

## Motivation

Currently, any anomaly that the user is notified about would need to be fixed manually by using the `KafkaRebalance` custom resource.
With smaller clusters, it is feasible to fix things manually. However, for larger ones it can be very time-consuming, or just not feasible, to fix all the anomalies on your own.
It would be useful for users of Strimzi to be able to have these anomalies fixed automatically whenever they are detected.

### Introduction to Self Healing

![self-healing flow diagram](images/106-self-healing-flow.png)

The above flow diagram depicts the self-healing process in Cruise Control.
The anomaly detector manager detects an anomaly (using the detector classes) and forwards it to the notifier.
The configured notifiers provides alerts to the users about the detected anomaly and also returns the action that needs to be taken on the anomaly i.e. whether to fix it, ignore it or delay it.

#### Anomaly Detector Manager

The anomaly detector manager helps in detecting the anomalies as well as handling them.
It acts as a coordinator between the detector classes and the classes which will handle resolving the anomalies.
Various detector classes like `GoalViolationDetector`, `DiskFailureDetector`, `KafkaBrokerFailureDetector` etc. are used for the anomaly detection, which runs periodically to check if the cluster has their corresponding anomalies or not.
The frequency of this check can be changed via the `anomaly.detection.interval.ms` configuration.
Detector classes have different mechanisms to detect their corresponding anomalies.
For example, `KafkaBrokerFailureDetector` utilises Kafka Metadata API whereas `DiskFailureDetector` and `TopicAnomalyDetector` utilises Kafka Admin API.
Furthermore, `MetricAnomalyDetector` use metrics and `GoalViolationDetector` uses the load distribution to detect their anomalies.
The detected anomalies can be of various types:
* Goal Violation - This happens if certain [optimization goals](https://strimzi.io/docs/operators/in-development/deploying#optimization_goals) are violated (e.g. DiskUsageDistributionGoal etc.). These goals can be configured through the `self.healing.goals` option in Cruise Control configuration.  However, this option is forbidden in the `spec.cruiseControl.config` section of the `Kafka` CR.
* Topic Anomaly - Where one or more topics in cluster violates user-defined properties (e.g. some partitions are too large in disk).
* Broker Failure - This happens when a non-empty broker crashes or leaves a cluster for time for a long time.
* Disk Failure - This failure happens if one of the non-empty disks fails (related to a Kafka Cluster with JBOD disks).
* Metric anomaly - This failure happens if metrics collected by Cruise Control have some anomaly in their value (e.g. a sudden rise in the log flush time metrics).

The detected anomalies are inserted into a priority queue where comparator is based upon the priority value and the detection time.
The smaller the priority value and detected time is, the higher priority the anomaly type has.

The anomaly detector manager calls the notifier to get an action regarding whether the anomaly should be fixed, delayed, or ignored.
If the action is `FIX`, then the anomaly detector manager calls the classes that are required to resolve the anomaly.

Anomaly detection also has various [configurations](https://github.com/linkedin/cruise-control/wiki/Configurations#anomalydetector-configurations), such as the detection interval and the anomaly notifier class, which can affect the performance of the Cruise Control server and the latency of the anomaly detection.

#### Notifiers in Cruise Control

Whenever anomalies are detected, Cruise Control provides the ability to notify the user regarding the detected anomalies using optional notifier classes.
The notification sent by these classes increases the visibility of the operations that are taken by Cruise Control.
The notifier class used by Cruise Control is configurable and custom notifiers can be used by setting the `anomaly.notifier.class` property.
The notifier class returns the `action` that is going to be taken on the flagged anomaly.
These actions have three types:
* FIX - Start the anomaly fix
* CHECK - Delay the anomaly fix
* IGNORE - Ignore the anomaly fix

The default `NoopNotifer` always sets the notifier action as `IGNORE`, which  means that the detected anomaly will be silently ignored and no notification is sent to the user.

Cruise Control also provides [custom notifiers](https://github.com/linkedin/cruise-control/wiki/Configure-notifications) like Slack Notifier, Alerta Notifier etc. for notifying users regarding the anomalies. There are multiple other [self-healing notifier](https://github.com/linkedin/cruise-control/wiki/Configurations#selfhealingnotifier-configurations) related configurations you can use to make notifiers more efficient as per the use case.

#### Self Healing

If self-healing is enabled, then an action is returned by the notifier to would decide whether the anomaly should be fixed or not.
If the notifier has returned `FIX` as the action then the classes which are responsible for resolving the anomaly would be called.
Each detectable anomaly is handled by a specific detector class which then uses another remediation class to run a fix.
For example, the `GoalViolations` class uses the `RebalanceRunnable` class, the `DiskFailure` class use the `RemoveDisksRunnable` class and so on.
An optimization proposal (a collection of replica reassignments and partition leadership changes) will then be generated by these `Runnable` classes and that proposal will be applied on the cluster to fix the anomaly.
In case the anomaly detected is unfixable for e.g. violated hard goals that cannot be fixed typically due to lack of physical hardware (insufficient number of racks to satisfy rack awareness, insufficient number of brokers to satisfy Replica Capacity Goal, or insufficient number of resources to satisfy resource capacity goals), the anomaly wouldn't be fixed and the Cruise Control logs will be updated with `self-healing is not possible due to unfixable goals` warning.

## Current situation

Even under normal operation, it's common for Kafka clusters to encounter problems such as partition key skew leading to an uneven partition distribution, or hardware issues like as disk failures, which can degrade overall cluster health and performance.
Currently, if we encounter any such scenario we need to fix these issues manually i.e. if the cluster is imbalanced then we might instruct Cruise Control to move the partition replicas across the brokers in order to fix the imbalance using the `KafkaRebalance` custom resource.

Users can currently enable anomaly detection and can also [set](https://strimzi.io/docs/operators/latest/full/deploying.html#setting_up_alerts_for_anomaly_detection) the notifier to one of those included with Cruise Control (SelfHealingNotifier, AlertaSelfHealingNotifier, SlackSelfHealingNotifier etc.).
However, self-healing is disabled and disallowed in Strimzi.
The `self-healing` properties were disabled in Strimzi's Cruise Control integration because, initially, it was not clear how self-healing would act if pods were rolled in middle of rebalances or how Strimzi triggered manual rebalances should interact with Cruise Control triggered self-healing ones.

### Proposal

This proposal allows the users to have their cluster balanced automatically whenever the cluster gets imbalanced due to overloaded broker, CPU usage e.t.c.
With self-healing ability of Cruise Control, the rebalances are triggered automatically in the cluster which means that the operator wouldn't have the information about when a rebalance is happening.
To resolve this issue, we will only make use of the ability of Cruise Control to detect the anomalies and based on the detection, we will then notify the operator to run the rebalance using an approach based on the existing auto-rebalance for scaling feature.
We will be using the goal violation anomaly detection related classes in Cruise Control to detect imbalanced cluster.
Doing this will provide us with the following advantages:
* we will ensure that the operator knows what is going on in the Kafka cluster.
* using the existing `KafkaRebalance` CR system make it easier for users to see what is happening and when, which aids in debugging.
* ensures the operators is in charge of rebalances.

### `skew` mode in Strimzi's auto-rebalancing feature

The [`auto-rebalancing`](https://strimzi.io/docs/operators/latest/deploying#proc-automating-rebalances-str) feature in Strimzi allows the operator to run a rebalance automatically when a Kafka cluster is scaled up (by adding brokers) or scaled down (by removing brokers).

Auto-rebalancing in Strimzi currently supports two modes:
* add-brokers - auto-rebalancing on scale up
* remove-brokers - auto-rebalancing on scale down

To leverage the automated rebalance on imbalanced cluster, we will be introducing a new mode to the auto-rebalancing feature.
The new mode will be called `skew`, which means that cluster imbalance was detected and rebalancing should be applied to the all the brokers.
The mode is defined by setting the `spec.cruiseControl.autoRebalance.mode` field as `skew` and the corresponding rebalancing configuration is defined as a reference to a "template" `KafkaRebalance` custom resource, by using the `spec.cruiseControl.autoRebalance.template` field as a [LocalObjectReference](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/local-object-reference/).
This field is optional and if not specified, the auto-rebalancing runs with the default Cruise Control configuration.
To provide user more flexibility, they don't have to configure all the modes for auto-rebalancing. 
They can configure auto-rebalance to enable only for their specific case i.e. setting only `skew` mode or other scaling related modes.
Once the auto-rebalance with `skew` mode is enabled, the operator will be ready to trigger auto-rebalance whenever the cluster becomes imbalanced.
To trigger the auto-rebalance, the operator must know that the cluster is imbalanced due to some goal violation anomaly. 
We will create our own custom notifier named `AnomalyDetectorNotifier` to do the same.
This notifier's job will be to update the operator regarding the goal violations so that the operator can trigger a rebalance(see section [AnomalyDetectorNotifier](./106-auto-rebalance-on-imbalanced-clusters.md#anomalydetectornotifier))
With this proposal, we are only going to support auto-rebalance on imbalanced cluster.
We also plan to implement the same for topic and metrics related issues, but it will be part of future work since their implementation require different approach.
For example, when dealing with topic related issues, it will require a coordination with topic operator and metrics issues will require coordination with the Kafka apis.

Here is an example of what the configured `Kafka` custom resource could look like:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
  # ...
  cruiseControl:
    # ...
    autoRebalance:
      # using the skew rebalance mode
      - mode: skew
        template:
          name: my-skew-rebalance-template
```

It is also possible to use the default Cruise Control rebalancing configuration by omitting the `template` field.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
  cruiseControl:
    # ...
    autoRebalance:
      # using the default Cruise Control rebalancing configuration 
      - mode: skew
```

The auto-rebalance configuration for the `spec.cruiseControl.autoRebalance.template` property in the `Kafka` custom resource is provided through a `KafkaRebalance` custom resource defined as a "template".
That is a `KafkaRebalance` custom resource with the `strimzi.io/rebalance-template: true` annotation set.
When it is created, the `KafkaRebalanceAssemblyOperator` doesn't run any rebalancing.
This is because it doesn't represent an "actual" rebalance request to get an optimization proposal, but it's just the place where configuration related to auto-rebalancing is defined.
The user can specify rebalancing goals and other configuration for rebalancing, within the resource.

Here is an example template:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-skew-rebalance-template
  annotations:
    strimzi.io/rebalance-template: "true" # specifies that this KafkaRebalance is a rebalance configuration template
spec:
  goals:
    - CpuCapacityGoal
    - NetworkInboundCapacityGoal
    - DiskCapacityGoal
    - RackAwareGoal
    - MinTopicLeadersPerBrokerGoal
    - NetworkOutboundCapacityGoal
    - ReplicaCapacityGoal
  skipHardGoalCheck: true
  # ... other rebalancing related configuration
```
When the "template" is set the operator automatically creates (or updates) a corresponding "actual" `KafkaRebalance` custom resource based on the "template" when an anomaly is detected and notified by the `AnomalyDetectorNotifier`
The operator copies over goals and rebalancing options from the referenced "template" resource to the "actual" rebalancing one and also adds the `spec.mode` to it.
The generated `KafkaRebalance` custom resource will be called `<my-cluster-name>-auto-rebalancing-skew-anomalyId`, where the `<my-cluster-name>` part comes from the `metadata.name` in the `Kafka` custom resource, and `skew` refers to applying the rebalance to all the brokers and the anomalyId would be retrieved from the notifier.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-add-brokers
  finalizers:
    - strimzi.io/auto-rebalancing
spec:
  mode: skew
  goals:
    - CpuCapacityGoal
    - NetworkInboundCapacityGoal
    - DiskCapacityGoal
    - RackAwareGoal
    - MinTopicLeadersPerBrokerGoal
    - NetworkOutboundCapacityGoal
    - ReplicaCapacityGoal
  skipHardGoalCheck: true
  # ... other rebalancing related configuration
```

The operator also sets a finalizer, named `strimzi.io/auto-rebalancing`, on the "actual" `KafkaRebalance` custom resource.
This is needed to avoid the user, or any other tooling, to delete the resource while the auto-rebalancing is still running.
The finalizer is removed at the end of the auto-rebalancing process, with or without errors, allowing the "actual" `KafkaRebalance` custom resource deletion by the operator itself.

#### AnomalyDetectorNotifier

Cruise Control provides the `AnomalyNotifier` interface, which has multiple abstract methods on what to do if certain anomalies are detected.
Some of those methods are:`onGoalViolation()`, `onBrokerFailure()`, `onDiskFailure`, `alert()` etc.
The `AnomalyDetectorNotifier` will be based of the `AnomalyNotifier` class.
The anomalies with smaller priority value and detected time will be considered priority and resolved first.
In case the anomaly is unfixable due to issues like  lack of physical hardware (e.g. insufficient number of racks to satisfy rack awareness, insufficient number of brokers to satisfy ReplicaCapacityGoal, or insufficient number of resources to satisfy resource capacity goals), then they will be ignored.
The `AnomalyDetectorNotifier` will override all the abstract methods provided by the `AnomalyNotifier` interface.
We will add an `alert` method which will alert the operator whenever an anomaly is detected by the operator.
Upon detection of an anomaly, the notifier would create a configmap with name set as `goal-violation` followed by `anomalyId`.

The ConfigMap would look like this:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: goal-violation-<anomalyID>
data:
  anomalyType: GOAL_VIOLATION
  anomalyId: "<anomaly-id>"
  detectionTime: "<time-of-detection>"
# ...
```

The operator will then check if any configmap with prefix `goal-violation` is created or not, if it finds one created then operator will trigger the rebalance.
Separate configmaps would be created for every goal violation such that on completion of the rebalance we can remove the particular configmap.
The advantages of using a separate configmap for every anomaly are:
1. Every anomaly would have a separate place to put their data in.
2. Better readability.  
3. Improves scope for future improvement when dealing with different violation, for example topic or metrics related violations.

#### Using other notifiers when using `skew` mode

Users cannot configure the notifier if they are utilising the auto-rebalance on imbalanced cluster.
This is because the operator is using our custom notifier for getting alerts about goal violations. 
If the users try to override the notifier while the `skew` mode is enabled, the auto-rebalance `skew` configuration then the operator would throw errors in the auto-rebalance status field

If the users really want to have their own way of dealing with the imbalanced clusters then they can just disable auto-rebalance in `skew` mode and use their own notifier.

#### What happens if some unfixable goal violation happens
In case, there is an unfixable goal violation then the notifier would simply ignore that anomaly and prompt the user about the unfixable violation in the auto-rebalancing status section.

#### What happens if same anomaly is detected again while the auto-rebalance is happening
Since the cluster operator has the knowledge regarding the detected violation, we will ignore the anomalies while the rebalancing is happening. In case the anomaly still exists after the rebalance, Cruise Control will detect it again and a new rebalance would be triggered

### Auto-rebalancing execution for `skew` mode

### Auto-rebalancing Finite State Machine (FSM) for `skew` mode

Currently, the auto-rebalancing mechanism runs through a Finite State Machine (FSM) made by the following states:

* **Idle**: Initial state with a new auto-rebalancing initiated when scaling down/up operations were requested. This is also the ending state after an auto-rebalancing completed successfully or failed.
* **RebalanceOnScaleDown**: a rebalancing related to a scale down operation is running.
* **RebalanceOnScaleUp**: a rebalancing related to a scale up operation is running.

With the new `skew` mode, we will be introducing a new state to the FSM called `RebalanceOnAnomalyDetection`. 
This state will be related to rebalances related to imbalanced cluster.

With the new `skew` mode, the FSM state transitions would look something like this:

```mermaid
flowchart TB
  A[Idle] --scaleDown requested--> B[RebalanceOnScaleDown]
  A[Idle] --scaleUp requested --> C[RebalanceOnScaleUp] 
  A[Idle] --imbalanced cluster detected --> D[RebalanceOnAnomalyDetection]
```
* from **Idle** to:
  * **RebalanceOnScaleDown**: if a scale down operation was requested. This transition happens even if a scale up was requested at the same time but the rebalancing on scaling down has the precedence. The rebalancing on scale up is queued. They will run sequentially.
  * **RebalanceOnScaleUp**: if only a scale up operation was requested. There was no scale down operation requested.
  * **RebalanceOnAnomalyDetection**: if a configmap related to goal violation was detected and a complete rebalance was requested.

```mermaid
sequenceDiagram
  autonumber
  RebalanceOnScaleDown->>RebalanceOnScaleDown: queued scale down
  RebalanceOnScaleDown->>RebalanceOnScaleUp: queued scale up
  RebalanceOnScaleDown->>RebalanceOnAnomalyDetection: queued skew rebalance
  critical queued scale down and skew rebalance
    RebalanceOnScaleDown->>RebalanceOnScaleDown: execute scale down
    RebalanceOnScaleDown-->>RebalanceOnAnomalyDetection: then skew rebalance
  end
  critical queued scale up and scale down
    RebalanceOnScaleDown->>RebalanceOnScaleDown: execute scale down
    RebalanceOnScaleDown-->>RebalanceOnScaleUp: then scale up
  end
  critical queued scale down and scale up and skew rebalance
    RebalanceOnScaleDown->>RebalanceOnScaleDown: execute scale down
    RebalanceOnScaleDown-->>RebalanceOnScaleUp: then scale up
    RebalanceOnScaleUp-->>RebalanceOnAnomalyDetection: then skew rebalance
  end
  RebalanceOnScaleDown->>Idle: scale down complete with no queued items
```

* from **RebalanceOnScaleDown** to:
  * **RebalanceOnScaleDown**: if a rebalancing on scale down is still running or another one was requested while the first one ended.
  * **RebalanceOnScaleUp**: if a scale down operation was requested together with a scale up and, because they run sequentially, the rebalance on scale down had the precedence, was executed first and completed successfully. We can now move on with rebalancing for the scale up.
  * **RebalanceOnAnomalyDetection**: if a configmap related to goal violation was detected. It will run once the queued scale down and scale up is completed
  * **Idle**: if only a scale down operation was requested, it was executed and completed successfully or failed.

```mermaid
sequenceDiagram
  autonumber
  RebalanceOnScaleUp->>RebalanceOnScaleDown: queued scale down
  RebalanceOnScaleUp->>RebalanceOnScaleUp: queued scale up
  RebalanceOnScaleUp->>RebalanceOnAnomalyDetection: queued skew rebalance
  critical queued scale up and skew rebalance
    RebalanceOnScaleUp->>RebalanceOnScaleUp: execute scale down
    RebalanceOnScaleUp-->>RebalanceOnAnomalyDetection: then skew rebalance
  end
  critical queued scale up and scale down
    RebalanceOnScaleUp->>RebalanceOnScaleDown: execute scale down
    RebalanceOnScaleUp-->>RebalanceOnScaleUp: then scale up
  end
  critical queued scale down and scale up and skew rebalance
    RebalanceOnScaleUp->>RebalanceOnScaleDown: execute scale down
    RebalanceOnScaleDown-->>RebalanceOnScaleUp: then scale up
    RebalanceOnScaleUp-->>RebalanceOnAnomalyDetection: then skew rebalance
  end
  RebalanceOnScaleUp->>Idle: scale up complete with no queued items
```

* from **RebalanceOnScaleUp**:
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is still running or another one was requested while the first one ended.
  * **RebalanceOnScaleDown**: if a scale down operation was requested, so the current rebalancing scale up is stopped (and queued) and a new rebalancing scale down is started. The rebalancing scale up will be postponed.
  * **RebalanceOnAnomalyDetection**: if a configmap related to goal violation was detected. It will run once the queued scale down and scale up is completed.
  * **Idle**: if a scale up operation was requested, it was executed and completed successfully or failed.

```mermaid
sequenceDiagram
  autonumber
  RebalanceOnAnomalyDetection->>RebalanceOnScaleDown: queued scale down
  RebalanceOnAnomalyDetection->>RebalanceOnScaleUp: queued scale up
  RebalanceOnAnomalyDetection->>RebalanceOnAnomalyDetection: queued skew rebalance
  critical queued scale up and skew rebalance
    RebalanceOnAnomalyDetection->>RebalanceOnScaleUp: execute scale up
    RebalanceOnScaleUp-->>RebalanceOnAnomalyDetection: then skew rebalance
  end
  critical queued scale down and skew rebalance
    RebalanceOnAnomalyDetection->>RebalanceOnScaleDown: execute scale down
    RebalanceOnScaleDown-->>RebalanceOnAnomalyDetection: then skew rebalance
  end
  critical queued scale down and scale up and skew rebalance
    RebalanceOnAnomalyDetection->>RebalanceOnScaleDown: execute scale down
    RebalanceOnScaleDown-->>RebalanceOnScaleUp: then scale up
    RebalanceOnScaleUp-->>RebalanceOnAnomalyDetection: then skew rebalance
  end
  RebalanceOnAnomalyDetection->>Idle: scale up complete with no queued items
```
* from **RebalanceOnAnomalyDetection**:
  * **RebalanceOnAnomalyDetection**: if another goal violation was detected while the first one ended. If a scale down and scale up is also queued up then they will execute first.
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is queued and will run if there is no other rebalancing scale down in queue. If a rebalancing scale down is in queue then it will be executed first.
  * **RebalanceOnScaleDown**: if a scale down operation was requested, it will run once the skew rebalance is completed
  * **Idle**: if skew rebalance was requested, it was executed and completed successfully or failed.

On each reconciliation, the following process will be used:

```mermaid
flowchart TB
  A[KafkaClusterCreator] --creates--> B[KafkaCluster]
  B -- calls --> D[KafkaAutoRebalancingReconciler.reconcile]
  D -- check for configmap with goal-violation prefix --> E{if config map present?}
  D -- if rebalance in progress --> F[ignore new configmaps and delete them]
  E -- yes --> G[Trigger auto-rebalance]
  E -- no --> H[No operation]
```

1. The `KafkaClusterCreator` creates the `KafkaCluster` instance.
2. The `KafkaAutoRebalancingReconciler.reconcile()` will then check if there was any configmap  created with `goal-violation` as prefix, if created, then the `skew` rebalance would be performed.
3. In case, a rebalance is already ongoing and another configmap related to goal violation is detected, then the operator will just ignore the configmap and delete it.

The `KafkaAutoRebalancingReconciler.reconcile()` loads the `Kafka.status.autoRebalance` content:

* `state`: is the FSM state.
* `lastTransitionTime`: when the transition to that state happened.
* `modes`: sets the mode as `skew`

The FSM is initialized based on the `state` field.

Let's see what happens during the auto-rebalancing process when the FSM starts from the **Idle** state and transitions to **RebalanceOnAnomalyDetection**

#### Idle

This state is set since the beginning when a `Kafka` custom resource is created with the `spec.cruiseControl.autoRebalance` field.
It is also the end state of a previous successfully completed or failed auto-rebalancing.
In case of successful completion, once the rebalance moves to `Ready` state, we will delete the KafkaRebalance and the configmap associated with the rebalance by matching the `anomalyId` suffix in their names and then update the `auto-rebalance` state to `Idle`.
In case of failed auto-rebalancing, once the rebalance moves to `NotReady` state, we will foloow the same procedure we used in successful completion.
In this state, the operator removes the finalizer and deletes the corresponding "actual" `KafkaRebalance` custom resource.

#### RebalanceOnAnomalyDetection

In this state, an anomaly was detected and a corresponding configmap was generated by the notifier.

 A KafkaRebalance resource will now be applied to the cluster to fix the imbalanced cluster. This kafka rebalance will be based on the template provided by the user, if no template is provided then the kafkaRebalance will be created with default configurations.

```mermaid
flowchart TB
  A[KafkaRebalanceState] --> B{if state}
  B --> C[Ready]
  C -- queued scale Down --> D[RebalanceOnScaleDown]
  C -- queued scale up --> E[RebalanceOnScaleUp]
  C -- queued rebalance --> F[RebalanceOnAnomalyDetection]
  B --> G[`PendingProposal`, `ProposalReady` or `Rebalancing`]
  G --> H{{rebalancing is running}}
  B --> I[NotReady]
  I --> J{{Transition to Idle state and delete rebalance resource and configmap}}
```

Checking the current `KafkaRebalance` status:

* if `Ready`, the rebalance was successful.
  * if there is a queued rebalancing scale down (`Kafka.status.autoRebalance.modes[remove-brokers]` exists), start the rebalancing scale down and transition to **RebalanceOnScaleDown**.
  * if there is a queued rebalancing scale up (`Kafka.status.autoRebalance.modes[add-brokers]` exists), start the rebalancing scale up and transition to **RebalanceOnScaleUp**.
  * If no queued rebalancing scale down or scale up, just transition to **Idle**, clean `Kafka.status.autoRebalance.modes`, delete the "actual" `KafkaRebalance` custom resource and also the configmap that triggered the rebalance by matching the `anomalyId` suffix in their names.
* if `PendingProposal`, `ProposalReady` or `Rebalancing`, the rebalancing is still running.
  * No further actions required.
* if `NotReady`
  * the rebalancing failed, transition to **Idle** and also removing the corresponding mode from the status. The operator also deletes the "actual" `KafkaRebalance` custom resource and the configmap associated with it.

If, during an ongoing auto-rebalancing, the `KafkaRebalance` custom resource is not there anymore on the next reconciliation, it could mean the user deleted it while the operator was stopped/crashed/not running.
In this case, the FSM will assume it as `NotReady` so falling in the last case above.

## Affected/not affected projects

This change will affect the Strimzi cluster operator and a new repository named `anomaly-detector-notifier` will be added under the Strimzi organisation.

## Rejected Alternatives

### Using custom resource for interaction between Operator and Cruise Control I

This alternative is about using a Kubernetes custom resource to create a two-way interaction between the operator and Cruise Control.
The idea was to create a Kubernetes custom resource named `KafkaAnomaly` everytime an anomaly was detected.
The user will get updates regarding the anomaly fix through the generated `KafkaAnomaly` resource which would be updated by the operator by requesting the `state` endpoint with `anomaly_detector` and `executor` substates.

Pros:
* More hold on the self-healing process since everything is driven using the Kafka custom resource.

Cons:
* Very tight coupling with the operator.
* Would be hard to manage multiple `KafkaAnomaly` custom resources (For example, deletion when anomaly is fixed etc.)

### Using custom resource for interaction between Operator and Cruise Control II

This alternative is similar to alternative 1 where we will use a Kubernetes custom resource to create a two-way interaction between the operator and Cruise Control.
The idea was to create a Kubernetes custom resource named `KafkaAnomaly` everytime an anomaly was detected and both the operator and the Notifier would watch the resource for updates.
But with this approach the operator will be responsible to make decision regarding the anomaly should be fixed or not

Pros:
* Allows the operator to ignore the anomalies is some task is already running in the cluster like rolling, rebalance etc.

Cons:
* Very tight coupling with the operator.
* Delaying the anomaly detector progress.

### Using Kubernetes Events but Cruise Control controls auto-rebalance

This alternative is to let Cruise Control handle the self-healing.
Whenever an anomaly is detected by Cruise Control, our notifier will generate an event to alert the operator that an anomaly was detected in the cluster
But the fix would be run by Cruise Control itself and not the operator

Pros:
* Loose coupling with the operator
* Faster decison making as Cruise Control runs the fix

Cons:
* Operator wouldn't play any role in the process