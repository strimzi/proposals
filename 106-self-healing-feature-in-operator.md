# Self Healing using Cruise Control

This proposal is about adding support for Cruise Control's self-healing capabilities in the Strimzi operator.
When enabled, the Strimzi operator should automatically resolve issues detected by the Anomaly Detector Manager without the need for manual intervention.
Anomalies are detected by Cruise Control using the anomaly detector manager.

## Current situation

Even under normal operation, it's common for Kafka clusters to encounter problems such as partition key skew leading to a uneven partition distribution, or hardware issues like as disk failures, which can degrade overall cluster health and performance.
Currently, if we encounter any such scenario we need to fix these issues manually i.e. if there is some goal violation and the cluster is imbalanced then we might instruct Cruise Control to move the partition replicas across the brokers in order to fix the violated goal by using the `KafkaRebalance` custom resource.

Currently, users can enable anomaly detection and can also [set](https://strimzi.io/docs/operators/latest/full/deploying.html#setting_up_alerts_for_anomaly_detection) the notifier to one of those included with Cruise Control (SelfHealingNotifier, AlertaSelfHealingNotifier, SlackSelfHealingNotifier etc.).
However, self-healing is currently disabled and disallowed in Strimzi.
The `self-healing` properties were disabled since there was investigation missing around how self-healing would act if pods roll in middle of it/rebalance starts in or how the operator would know if self-healing is running or not.

## Motivation

Currently, any anomaly that the user is notified about would need to be fixed manually by using the `KafkaRebalance` custom resource.
With smaller clusters, it is feasible to fix things manually. However, for larger ones it can be very time-consuming, or just not feasible, to fix all the anomalies on your own.
It would be useful for users of Strimzi to be able to have these anomalies fixed automatically whenever they are detected.

### Introduction to Self Healing

![self-healing flow diagram](images/105-self-healing-flow.png)

The above flow diagram depicts the self-healing process in Cruise Control.
The anomaly detector manager detects an anomaly (using the detector classes) and forwards it to the notifier.
The notifier classes like `SelfHealingNotifier`, `AlertaSelfHealingNotifier`, `SlackSelfHealingNotifier` etc. provides alerts to the users about the detected anomaly and also returns the action that needs to be taken on the anomaly i.e. whether to fix it, ignore it or delay it.

#### Anomaly Detector Manager

The anomaly detector manager helps in detecting the anomalies as well as handling them.
It acts as a coordinator between the detector classes and the classes which will handle resolving the anomalies.
Various detector classes like `GoalViolationDetector`, `DiskFailureDetector`, `KafkaBrokerFailureDetector` etc. are used for the anomaly detection, which runs periodically to check if the cluster has their corresponding anomalies or not.
The frequency of this check can be changed via the `anomaly.detection.interval.ms` configuration.
Detector classes have different mechanisms to detect their corresponding anomalies.
For example, `KafkaBrokerFailureDetector` utilises Kafka Metadata API whereas `DiskFailureDetector` and `TopicAnomalyDetector` utilises Kafka Admin API.
Furthermore, `MetricAnomalyDetector` and `GoalViolationDetector` use metrics to detect their anomalies.
The detected anomalies can be of various types:
* Goal Violation - This happens if certain optimization goals are violated (e.g. DiskUsageDistributionGoal etc.). These goals can be configured through the `self.healing.goals` configuration under the `spec.cruiseControl.config` section. These goals are independent of the manual rebalancing goals configured in the `KafkaRebalance` custom resource.
* Topic Anomaly - Where one or more topics in cluster violates user-defined properties (e.g. some partitions are too large in disk).
* Broker Failure - This happens when a non-empty broker crashes or leaves a cluster.
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
An optimization proposal will then be generated by these `Runnable` classes and that proposal will be applied on the cluster to fix the anomaly.
In case the anomaly detected is unfixable for e.g. violated hard goals that cannot be fixed typically due to lack of physical hardware (insufficient number of racks to satisfy rack awareness, insufficient number of brokers to satisfy Replica Capacity Goal, or insufficient number of resources to satisfy resource capacity goals), the anomaly wouldn't be fixed and the Cruise Control logs will be updated with `self-healing is not possible due to unfixable goals` warning.

### Proposal

This proposal allows the users to have their cluster balanced automatically whenever an anomaly is detected by Cruise Control.
To ensure that the operator drives the rebalance we will be leveraging the Cruise Control self-healing mechanism but not completely.
When using self-healing in Cruise Control, the rebalance are triggered automatically in the cluster which means that the operator wouldn't have the information about when a rebalance is happening.
To resolve this issue, we will only make use of the ability of Cruise Control to detect the anomalies and based on the detection, we will then notify the operator to run the rebalance.
Doing this will provide us with following merits:
* ensure that the operator know what is going on in the cluster.
* better debugging.
* to drive the rebalances/fixes from the operator.

#### `Full` mode in auto-rebalancing feature of Strimzi

To leverage the above self-healing mechanism, we will be introducing a new mode to the `auto-rebalancing` feature.
The new mode will be called `FULL`, which means that an anomaly was detected and the rebalancing should be applied to the whole cluster
The mode is defined by setting the `spec.cruiseControl.autoRebalance.mode` field as `full` and the corresponding rebalancing configuration is defined as a reference to a "template" `KafkaRebalance` custom resource (read later for more details), by using the `spec.cruiseControl.autoRebalance.template` field as a [LocalObjectReference](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/local-object-reference/).
This field is optional and if not specified, the auto-rebalancing runs with the default Cruise Control configuration.
Once the auto-rebalance with `Full` mode is enabled, self-healing will be enabled and a notifier `AnomalyDetectorNotifier` will be set(default notifier).
This notifier's job will be to update the operator regarding the anomalies being detected so that the operator can trigger a rebalance.
The user can change the default notifier if they want by setting the `anomaly.notifier.class` to point towards their own custom notifier.
With this proposal, we are only going to support the goal-violation anomaly. We plan to implement the same for topic and metrics anomalies as well but for that we will need to understand more on how those anomalies are actually resolved internally in Cruise Control.

Here is an example of what the configured `Kafka` custom resource could look:

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
      # using the full rebalance mode
      - mode: full
        template:
          name: my-full-rebalance-template
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
      - mode: full
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
  name: my-full-rebalance-template
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
The "actual" `KafkaRebalance` custom resource will be named as `<my-cluster-name>-auto-rebalancing-full` where the `<my-cluster-name>` part comes from the `metadata.name` in the `Kafka` custom resource, and `full` refers to applying rebalance on all the brokers.

#### AnomalyDetectorNotifier

Cruise Control provides the `AnomalyNotifier` interface, which has multiple abstract methods on what to do if certain anomalies are detected.
`SelfHealingNotifier` is the base class that contains the logic of self-healing and override the methods in by implementing the `AnomalyNotifier` interface.
Some of those methods are:`onGoalViolation()`, `onBrokerFailure()`, `onDiskFailure`, `alert()` etc.
The `AnomalyDetectorNotifier` will be based of the `SelfHealingNotifier` class.
The anomalies with smaller priority value and detected time will be considered priority and resolved first.
In case the anomaly is unfixable then they will be ignored.
The `AnomalyDetectorNotifier` will override the `alert()` method and other detector methods of the `SelfHealingNotifier` to ensure that it is able to alert the operator whenever an anomaly is detected.
Upon detection of an anomaly, the notifier would apply `strimzi.io/anomaly-detected: <anomaly-type>` annotation on the `Kafka` CR.
Based on the annotation applied from this notifier, the operator will the run the rebalance on the brokers.

The annotated Kafka resource would look like this:

```yaml
- apiVersion: kafka.strimzi.io/v1beta2
  kind: Kafka
  metadata:
    annotations:
      strimzi.io/anomaly-detected: "GOAL_VIOLATION"
# ...
```

#### Using other notifiers when using `Full` mode

Users can choose what type of notifier they want Cruise Control to use.
They can use the notifier provided by Cruise Control, such as the `SelfHealingNotifier`, `SlackSelfHealingNotifier`, `AlertaSelfHealingNotifier` and `MSTeamsSelfHealingNotifier` but then the operator wouldn't be handling the rebalance and Cruise Control will be handling the healing instead, since there is no way for the operator to know whether the anomaly was detected and a rebalance is required.

The users can also implement their own notifiers, but need to make sure that they have some way to let the operator know that an anomaly was detected and rebalance is required(like we do with default `AnomalyDetectorNotifier`), otherwise again the auto-rebalancing would be a no-op since no mode will be set.

### Auto-rebalancing execution for `Full` mode

### Auto-rebalancing Finite State Machine (FSM) for `Full` mode

The auto-rebalancing mechanism runs through a Finite State Machine (FSM) made by the following states:

* **Idle**: Initial state with a new auto-rebalancing initiated when self-healing was requested. This is also the ending state after an auto-rebalancing completed successfully or failed.
* **RebalanceOnAnomalyDetection**: a rebalancing related to a self-healing of cluster.

The FSM states' transitions are the following:

* from **Idle** to:
  * **RebalanceOnAnomalyDetection**: if the `strimzi.io/anomaly-detected: <anomaly-type>` annotation is applied on the Kafka CR.
* from **RebalanceOnAnomalyDetection**:
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is queued and will run if there is no other rebalancing scale down in queue. If a rebalancing scale down is in queue then it will be executed first.
  * **RebalanceOnScaleDown**: if a scale down operation was requested, it will run once the full rebalance is completed
  * **Idle**: if full rebalance was requested, it was executed and completed successfully or failed.

On each reconciliation, the logic will be like this:

1. The `KafkaClusterCreator` creates the `KafkaCluster` instance.
2. The `KafkaAutoRebalancingReconciler.reconcile()` will then check if the annotation `strimzi.io/anomaly-detected: <anomaly-type>` is applied on the Kafka CR, if applied then the `Full` rebalance would be performed.
   What happens for each reconciliation depends on the auto-rebalancing FSM as well as the "actual" `KafkaRebalance` custom resource status.

The `KafkaAutoRebalancingReconciler.reconcile()` loads the `Kafka.status.autoRebalance` content:

* `state`: is the FSM state.
* `lastTransitionTime`: when the transition to that state happened.
* `modes`: sets the mode as `Full`

Let's see what happens during the auto-rebalancing process when the FSM starts from the **Idle** state and tranistions to **RebalanceOnAnomalyDetection**

#### Idle

This state is set since the beginning when a `Kafka` custom resource is created with the `spec.cruiseControl.autoRebalance` field.
It is also the end state of a previous successfully completed or failed auto-rebalancing.
In this state, the operator removes the finalizer and deletes the corresponding "actual" `KafkaRebalance` custom resource.

#### RebalanceOnAnomalyDetection

In this state, an anomaly is detected and the annotation `strimzi.io/anomaly-detected: <anomaly-type>` is applied on `Kafka` CR

Check the current `KafkaRebalance` status:

* if `Ready`, the rebalance was successful.
  * if there is a queued rebalancing scale down (`Kafka.status.autoRebalance.modes[remove-brokers]` exists), start the rebalancing scale down and transition to **RebalanceOnScaleDown**.
  * if there is a queued rebalancing scale up (`Kafka.status.autoRebalance.modes[add-brokers]` exists), start the rebalancing scale up and transition to **RebalanceOnScaleUp**.
  * If no queued rebalancing scale down or scale up, just transition to **Idle**, clean `Kafka.status.autoRebalance.modes`, delete the "actual" `KafkaRebalance` custom resource and remove the annotation .
* if `PendingProposal`, `ProposalReady` or `Rebalancing`, the rebalancing scale up is still running.
  * No further actions required.
* if `NotReady`
  * the rebalancing failed, transition to **Idle** and also removing the corresponding mode from the status. The operator also deletes the "actual" `KafkaRebalance` custom resource.

If, during an ongoing auto-rebalancing, the `KafkaRebalance` custom resource is not there anymore on the next reconciliation, it could mean the user deleted it while the operator was stopped/crashed/not running.
In this case, the FSM will assume it as `NotReady` so falling in the last case above.

## Affected/not affected projects

This change will affect the Strimzi cluster operator and a new repository named `anomaly-detector-notifier` will be added under the Strimzi organisation.

## Rejected Alternatives

### Alternative 1

This alternative is about using a Kubernetes custom resource to create a two-way interaction between the operator and Cruise Control.
The idea was to create a Kubernetes custom resource named `KafkaAnomaly` everytime an anomaly was detected.
The user will get updates regarding the anomaly fix through the generated `KafkaAnomaly` resource which would be updated by the operator by requesting the `state` endpoint with `anomaly_detector` and `executor` substates.

Pros:
* More hold on the self-healing process since everything is driven using the Kafka custom resource.

Cons:
* Very tight coupling with the operator.
* Would be hard to manage multiple `KafkaAnomaly` custom resources (For example, deletion when anomaly is fixed etc.)

### Alternative 2

This alternative is similar to alternative 1 where we will use a Kubernetes custom resource to create a two-way interaction between the operator and Cruise Control.
The idea was to create a Kubernetes custom resource named `KafkaAnomaly` everytime an anomaly was detected and both the operator and the Notifier would watch the resource for updates.
But with this approach the operator will be responsible to make decision regarding the anomaly should be fixed or not

Pros:
* Allows the operator to ignore the anomalies is some task is already running in the cluster like rolling, rebalance etc.

Cons:
* Very tight coupling with the operator.
* Delaying the anomaly detector progress.
*
### Alternative 3

This alternative is to let Cruise Control handle the self-healing.
Whenever an anomaly is detected by Cruise Control, our notifier will generate an event to alert the operator that an anomaly was detected in the cluster
But the fix would be run by Cruise Control itself and not the operator

Pros:
* Loose coupling with the operator
* Faster decison making as Cruise Control runs the fix

Cons:
* Operator wouldn't play any role in the process
