# Self Healing using Cruise Control

This proposal is about enabling the self-healing properties of Cruise Control in the Strimzi operator.
The self-healing feature of Cruise Control allows us to heal the anomalies detected in the cluster, automatically. 
Anomalies are detected by Cruise Control using the anomaly detector manager.

## Current situation

During normal operations it's common for Kafka clusters to become unbalanced (through partition key skew, uneven partition placement etc.) or for problems to occur with disks or other underlying hardware which makes the cluster unhealthy.  
Currently, if we encounter any such scenario we need to fix these issues manually i.e. if there is some goal violation and the cluster is imbalanced then we might move the partition replicas across the brokers to fix the violated goal by using the `KafkaRebalance` custom resource in the [`rebalance`](https://strimzi.io/docs/operators/latest/full/deploying.html#proc-generating-optimization-proposals-str) mode.
With smaller clusters, it is feasible to fix things manually. However, for larger ones it can be very time-consuming, or just not feasible, to fix all the anomalies on your own.

## Motivation

With the help of the self-healing feature we can resolve the anomalies causing unhealthy/imbalanced cluster automatically.
The detection of anomalies is the responsibility of the anomaly detector mechanism.
Notifiers in Cruise Control are used to send notification to user about the self-healing operations being performed in the cluster.
Notifiers can be useful as they provide visibility to the user regarding the actions that are being performed by Cruise Control regarding self-healing for e.g. anomaly being detected, anomaly fix being started etc.
Currently, users are [able to set](https://strimzi.io/docs/operators/latest/full/deploying.html#setting_up_alerts_for_anomaly_detection) the notifier to one of those included with Cruise Control (SelfHealingNotifier, AlertaSelfHealingNotifier, SlackSelfHealingNotifier etc).
However, any anomaly that is detected would then need to be fixed manually by using the `KafkaRebalance` custom resource.
It would be useful for users of Strimzi to be able to have these anomalies fixed automatically whenever they are detected.

## Proposal

### Introduction

![self-healing flow diagram](./images/090-self-healing-flow.png)

#### Anomaly Detector Manager

The anomaly detector manager helps in detecting the anomalies as well as handling them.
It acts as a coordinator between the detector classes as well as the classes which will be handling and resolving the anomalies.
Various detector classes like `GoalViolationDetector`, `DiskFailureDetector`, `KafkaBrokerFailureDetector` etc. are used for the anomaly detection, which runs periodically to check if the cluster have their corresponding anomalies or not (The frequency of this check can be easily configured through the `anomaly.detection.interval.ms` configuration).
Detector classes have different mechanisms to detect their corresponding anomalies. For example, `KafkaBrokerFailureDetector` utilises Kafka Metadata API whereas `DiskFailureDetector` and `TopicAnomalyDetector` utilises Kafka Admin API. Furthermore, `MetricAnomalyDetector` and `GoalViolationDetector` uses metrics to detect the anomalies.
The detected anomalies can be of various types:
* Goal Violation - This happens if certain optimization goals are violated (e.g. DiskUsageDistributionGoal etc.). These goals can be configured through the `self.healing.goals` configuration under the `spec.cruiseControl.config` section. These goals are independent of the manual rebalancing goals configured in the KafkaRebalance custom resource.
* Topic Anomaly - Can happen when one or more topics in cluster violates user-defined properties (e.g. some partitions are too large in disk).
* Broker Failure - It happens when a non-empty broker crashes or leaves a cluster.
* Disk Failure - This failure happens if one of the non-empty disks fails (related to a Kafka Cluster with JBOD disks).
* Metric anomaly - This failure happens if metrics collected by Cruise Control have some anomaly in their value (e.g. a sudden rise in the log flush time metrics).

The detected anomalies are inserted into a priority queue and the anomaly with the highest property is picked first to resolve.
The anomaly detector manager calls the notifier to get an action regarding whether the anomaly should be fixed, delayed, or ignored.
If the action is `FIX`, then the anomaly detector manager calls the classes that are required to resolve the anomaly.

Anomaly detection also has various [configurations](https://github.com/linkedin/cruise-control/wiki/Configurations#anomalydetector-configurations), such as the detection interval and the anomaly notifier class, which can affect the performance of the Cruise Control server and the latency of the anomaly detection.

#### Notifiers in Cruise Control

Whenever anomalies are detected, Cruise Control provides the ability to notify the user regarding the detected anomalies using optional notifier classes.
The notification sent by these classes increases the visibility of the operations that are taken by Cruise Control.
However, the notifier class used by Cruise Control is configurable and custom notifiers can be used by setting the `anomaly.notifier.class` property.
The notifier class returns the `action` that is going to be taken on the flagged anomaly.
These actions can be classified into three types:
* FIX - Start the anomaly fix
* CHECK - Delay the anomaly fix
* IGNORE - Ignore the anomaly fix

The default `NoopNotifer` always sets the notifier action as `IGNORE` means that the detected anomaly will be silently ignored.

Cruise Control also provides [custom notifiers](https://github.com/linkedin/cruise-control/wiki/Configure-notifications) like Slack Notifier, Alerta Notifier etc. for notifying users regarding the anomalies. There are multiple other [self-healing notifier](https://github.com/linkedin/cruise-control/wiki/Configurations#selfhealingnotifier-configurations) related configurations you can use to make notifiers more efficient as per the use case.

#### Self Healing

If self-healing is enabled in Cruise Control and anomalies are detected, then a fix for these will be attempted automatically.
The action returned by the notifier would decide whether the anomaly should be fixed or not.
If the notifier has returned `FIX` as the action then the classes which are responsible for resolving the anomaly would be called.
Each detected anomaly is implemented by a specific class which leverages a corresponding other class to run a fix.
For example, the `GoalViolations` class uses the `RebalanceRunnable` class, the `DiskFailure` class use the `RemoveDisksRunnable` class and so on.
An optimization proposal will then be generated by these `Runnable` classes and that proposal will be applied on the cluster to fix the anomaly.
In case the anomaly detected is unfixable for e.g. violated hard goals that cannot be fixed typically due to lack of physical hardware(insufficient number of racks to satisfy rack awareness, insufficient number of brokers to satisfy Replica Capacity Goal, or insufficient number of resources to satisfy resource capacity goals), the anomaly wouldn't be fixed and the cruise control logs will be updated with `self-healing is not possible due to unfixable goals` warning.
When self-healing starts, you can check the anomaly status by polling the [`state` endpoint](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) with `substate` parameter set as `anomaly_detector`. This would provide the anomaly details like anomalyId, the type of anomaly and the status of the anomaly etc.
For example, the user can poll the endpoint using the `curl` command:
```shell
curl -vv -X GET "http://localhost:9090/kafkacruisecontrol/state?substates=anomaly_detector"
```
the detected anomaly would look something like this:
```shell
#...
recentDiskFailures:[{anomalyId=4b1ed191-50d7-4aa1-806b-b8c5c0d679fe, 
detectionDate=2024-12-09T09:49:46Z, 
statusUpdateDate=2024-12-09T09:49:47Z, 
failedDisksByTimeMs={2={/var/lib/kafka/data-1/kafka-log2=1733737786603}}, 
status=FIX_STARTED}]}
```
where the `anomalyID` represents the task id assigned to the anomaly, `failedDisksByTimeMs` represents the broker id(which in this case is 2) as well the disk(/var/lib/kafka/data-1/kafka-log2) which failed, `detectionDate` and `statusUpdateDate` represent the date and time when the anomaly was detected and the status of the anomaly was updated.

The anomaly status would change based on how the healing progresses. The anomaly can transition to these possible status:
* DETECTED - The anomaly is just detected and no action is yet taken on it 
* IGNORED - Self healing is either disabled or the anomaly is unfixable
* FIX_STARTED - The anomaly fix has started
* FIX_FAILED_TO_START - The proposal generation for the anomaly fix failed 
* CHECK_WITH_DELAY - The anomaly fix is delayed 
* LOAD_MONITOR_NOT_READY - Monitor for workload of a Kafka cluster is in loading or bootstrap state
* COMPLETENESS_NOT_READY - Completeness not met for some goals. The completeness describes the confidence level of the data in the metric sample aggregator. If this is low then some Goal classes may refuse to produce proposals.

We can again poll the [`state` endpoint](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control), with `substate` parameter as `executor`, to get details on the tasks that are currently being performed by Cruise Control.
For example:
```shell
ExecutorState: {state: INTER_BROKER_REPLICA_MOVEMENT_TASK_IN_PROGRESS,
 pending(20)/in-progress(0)/aborting(0)/finished(0)/total(20) inter-broker partition movements, 
 completed(0)/total(0) bytes in MBs: 0.00%, max/min/avg concurrent inter-broker partition movements per-broker: 5/5/5.00, 
 triggeredSelfHealingTaskId: 4b1ed191-50d7-4aa1-806b-b8c5c0d679fe, triggeredTaskReason:
 Self healing for DISK_FAILURE: {
	Disk /var/lib/kafka/data-1/kafka-log2 on broker 2 failed at 2024-12-09T09:49:46Z
 }}
```
which gives more information about inter broker replica movements tasks which are: in-progress , pending, aborted, finished etc.

### Self-healing configuration in the Strimzi operator

Users can activate self-healing by setting the `self.healing.enabled` property to `true` and specifying a notifier that the self-healing mechanism is going to use in the `spec.cruiseControl.config` section of their `Kafka` resource.
If the user don't set the notifier property then `NoopNotifer` will be used by default which means all the action on the anomalies would be set to `IGNORE`.
Self-healing corresponding to broker failure or disk failure is disabled by default since we have better ways to fix these anomalies instead of moving the partition replicas across the brokers.
For example, brokers failures are generally due to lack of resources, so we can fix the issue at cluster level and in the same way failed disks can be easily replaced with a new one, so self-healing in these cases can be more resource consuming.

Here is an example of how the configured Kafka custom resource could look:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
  cruiseControl:
    config:
      # configuration to enable self-healing for all anomalies
      self.healing.enabled: true
      anomaly.notifer.class: <notifier-class> # can be set to `SelfHealingNotifier, SlackSelfHealingNotifier, AlertaSelfHealingNotifier etc. or you own custom notifier
      self.healing.broker.failure.enabled: false # disables self healing for broker failure
      self.healing.disk.failure.enabled: false # disables self healing for disk failure
```

The user can still enable self-healing for disk failures and broker failure if they want by setting the  `self.healing.broker.failure.enabled` for broker failure and `self.healing.disk.failure.enabled` for disk failure to `true`.
In the same way user can enable and disable the self-healing for other anomalies.

### Using Notifiers when using Self-healing

Users can set the notifier as per their choice.
They can also use the notifier provided by Cruise Control like the `SelfHealingNotifier`, `SlackSelfHealingNotifier`, `AlertaSelfHealingNotifier`, `MSTeamsSelfHealingNotifier` or they can even build own custom notifier jar and use it.
In case, the user wants to have some Kubernetes way to see what anomalies are being detected or the self-healing operations are being performed on the cluster, they can use our custom `KubernetesEventsNotifier` which will be available under the Strimzi GitHub organization.

#### KubernetesEventsNotifier 

Cruise Control provides `AnomalyNotifier` interface which has multiple abstract methods on what to do if certain anomalies are detected.
`SelfHealingNotifier` is the base class that contains the logic of self-healing and override the methods in by implementing the `AnomalyNotifier` interface.
Some of those methods are:`onGoalViolation()`, `onBrokerFailure()`, `onDiskFailure`, `alert()` etc.
The `KubernetesEventsNotifier` is based of the `SelfHealingNotifier` class.
For effective alerts and monitoring, it makes use of Kubernetes events to signal to users what operations Cruise Control is performing automatically.
The `KubernetesEventsNotifier` will override the `alert()` method of the `SelfHealingNotifier` and will trigger and publish events whenever an anomaly is detected.
These events would publish information regarding the anomaly like the anomalyID, the type of anomaly which was detected etc.
The events would help the user to understand the anomalies that are present, when the anomalies appeared in their cluster and also would help them know what is going on in their clusters.

The event published would look like this:
```yaml
- action: <action>
  apiVersion: v1
  eventTime: "2024-12-19T08:06:21.314140Z"
  firstTimestamp: null
  involvedObject:
    kind: Pod
    name: <cruise-control-pod-name>
    namespace: myproject
  kind: Event
  lastTimestamp: null
  message: Cruise Control is fixing the detected anomaly
  metadata:
    creationTimestamp: "2024-12-19T08:06:21Z"
    generateName: cruise-control-event
    name: my-cluster-34c41626-0add-47a3-9b79-d6ee29d5d81f  # my-cluster-<Anomaly ID>
    namespace: myproject
    resourceVersion: "101070"
    uid: 80892c7d-e3a1-44ad-84dc-2d88e15a7ff0
  reason: Anomaly was detected in the cluster GOAL_VIOLATION # Detected Anomaly type
  reportingComponent: cruise-control
  reportingInstance: cruise-control
  source: {}
  type: Normal
```

The users can build a jar for the notifier for inclusion in the Kafka image to run on as part of Cruise Control.

### Strimzi Operations while self-healing is running/fixing an anomaly 

Self-healing is a very robust and resilient feature due to its ability to tackle issues while healing a cluster.

#### What happens when self-healing is running/fixing an anomaly and brokers are rolled

If some anomaly is getting healed , Cruise Control will then try to fix/mitigate the anomaly by moving the partition replicas across the brokers in the Kafka cluster.
If, during the process of moving partitions replicas among the brokers in the Kafka cluster, some broker that was being used gets deleted or rolled, then Cruise Control would finish self-healing process and log that `self-healing finished successful with error`.
If the deleted or rolled broker comes back before the periodic anomaly check interval, then self-healing would start again only for the earlier failed broker with a new anomalyId since it was not fixed properly. 
In case the deleted broker doesn't come back healthy within the periodic anomaly check interval then another new anomaly for the deleted broker will be created and the earlier detected failed broker would also persist but will be updated with a new anomalyId.

#### What happens when self-healing is running/fixing an anomaly and Kafka Rebalance is approved

If self-healing is ongoing and a `KafkaRebalance` resource is posted in the middle of it then the `KafkaRebalance` resource will move to `NotReady` state, stating that there is already a rebalance ongoing. 
For example:
```yaml
status:
  conditions:
  - lastTransitionTime: "2025-01-08T08:24:01.314273452Z"
    message: 'Error for request: my-cluster-cruise-control.myproject.svc:9090/kafkacruisecontrol/rebalance?json=true&dryrun=false&verbose=true&skip_hard_goal_check=false&rebalance_disk=false.
      Server returned: Error processing POST request ''/rebalance'' due to: ''java.lang.IllegalStateException:
      Cannot start a new execution while there is an ongoing execution. Please use
      stop_ongoing_execution=true to stop ongoing execution and start a new one.''.'
    reason: CruiseControlRestException
    status: "True"
    type: NotReady
  - lastTransitionTime: "2025-01-08T08:24:01.349653295Z"
    message: 'Wrong annotation value: approve. The rebalance resource supports the
      following annotations in NotReady state: [refresh]'
    reason: InvalidAnnotation
    status: "True"
    type: Warning
  observedGeneration: 1
```

#### What happens when self-healing is running/fixing an anomaly and Kafka Rebalance is applied to ask for a proposal

If self-healing is ongoing and a `KafkaRebalance` resource is applied in the middle of it then Cruise Control will still generate an optimization proposal. The proposal should be based on the cluster shape at that particular timestamp.

## Affected/not affected projects

A new repository named `kubernetes-event-notifier` will be added under the Strimzi organisation.

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
