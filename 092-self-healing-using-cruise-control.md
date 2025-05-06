# Self Healing using Cruise Control

This proposal is about enabling the self-healing properties of Cruise Control in the Strimzi operator.
The self-healing feature of Cruise Control allows us to heal the anomalies detected in the cluster, automatically. 
Anomalies are detected by Cruise Control using the anomaly detector manager.

## Current situation

During normal operations it's common for Kafka clusters to become unbalanced (through partition key skew, uneven partition placement etc.) or for problems to occur with disks or other underlying hardware which makes the cluster unhealthy.
Currently, if we encounter any such scenario we need to fix these issues manually i.e. if there is some goal violation and the cluster is imbalanced then we might instruct Cruise Control to move the partition replicas across the brokers in order to fix the violated goal by using the `KafkaRebalance` custom resource.
With smaller clusters, it is feasible to fix things manually. However, for larger ones it can be very time-consuming, or just not feasible, to fix all the anomalies on your own.

## Motivation

With the help of Cruise Control's self-healing feature we can resolve the anomalies causing unhealthy/imbalanced cluster automatically.
The detection of anomalies is the responsibility of the anomaly detector mechanism.
Notifiers in Cruise Control are used to send notification to user about the self-healing operations being performed in the cluster.
Notifiers can be useful as they provide visibility to the user regarding the actions that are being performed by Cruise Control regarding self-healing for e.g. anomaly being detected, anomaly fix being started etc.
Currently, users are [able to set](https://strimzi.io/docs/operators/latest/full/deploying.html#setting_up_alerts_for_anomaly_detection) the notifier to one of those included with Cruise Control (SelfHealingNotifier, AlertaSelfHealingNotifier, SlackSelfHealingNotifier etc.).
However, self-healing is currently disabled and disallowed in Strimzi. 
The `self-healing` properties were disabled since there was investigation missing around how self-healing would act if pods roll in middle of it/rebalance starts in  or how the operator would know if self-healing is running or not 
Any anomaly that the user is notified about would need to be fixed manually by using the `KafkaRebalance` custom resource.
It would be useful for users of Strimzi to be able to have these anomalies fixed automatically whenever they are detected.

## Proposal

### Introduction

![self-healing flow diagram](./images/090-self-healing-flow.png)

The above flow diagram depicts the self-healing process.
The anomaly detector manger detects the anomaly (using the detector classes) and forwards it to the notifier.
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
When self-healing starts, you can check the anomaly status by querying the [`state` endpoint](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control) with `substate` parameter set as `anomaly_detector`.
This will provide the anomaly details such as its ID number, the type of anomaly and the status of the anomaly etc.

For example, the user can query the endpoint using the `curl` command:

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

where: 
* `anomalyId` represents the task id assigned to the anomaly.
* `failedDisksByTimeMs` represents the broker id(which in this case is 2) as well the disk(/var/lib/kafka/data-1/kafka-log2) which failed.
* `detectionDate` and `statusUpdateDate` represent the date and time when the anomaly was detected and the status of the anomaly was updated.

The anomaly status changes based on how the healing is progressing.
The anomaly can transition to these possible states:
* DETECTED - The anomaly is just detected and no action is yet taken on it 
* IGNORED - Self healing is either disabled or the anomaly is unfixable
* FIX_STARTED - The anomaly fix has started
* FIX_FAILED_TO_START - The proposal generation for the anomaly fix failed 
* CHECK_WITH_DELAY - The anomaly fix is delayed 
* LOAD_MONITOR_NOT_READY - The Monitor which interprets the workload of a Kafka cluster is in loading or bootstrap state
* COMPLETENESS_NOT_READY - Completeness not met for some goals. The completeness describes the confidence level of the data in the metric sample aggregator. If this is low then some Goal classes may refuse to produce proposals.

We can again query the [`state` endpoint](https://github.com/linkedin/cruise-control/wiki/REST-APIs#query-the-state-of-cruise-control), with `substate` parameter as `executor`, to get details on the tasks that are currently being performed by Cruise Control.

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

where:
* `state` refers to the status of the replica movement (pending/in-progress/finished etc.)
* `triggeredSelfHealingTaskId` points to the anomalyId.
* `triggeredTaskReason` refers to the reason why self-healing was triggered.

#### Using metrics when using self-healing

Cruise control provides [self-healing related metrics](https://github.com/linkedin/cruise-control/wiki/Sensors#anomalydetector-sensors) which can help users understand what kind of anomalies were detected in their cluster, when self-healing started, the rate at which the anomalies are being detected in their cluster and much more.
These metrics can also provide a way for the users to debug self-healing related issues.
For example, the `number_of_self_healing_started` metric can provides users, the way to check when self-healing was started for an anomaly, and the number of self-healing actions that were performed in the cluster.
This information can be useful to at what point of time the self-healing occurred and the shape/replica assignment of the cluster was changed due to self-healing.
The user can also use the  metric like `number_of_self_healing_failed` to know if the self-healing failed to start to understand, and then they can check the reasons in the log for the failure. One example could be load monitor was ready at that point of time.
Cruise Control also have [metrics](https://github.com/linkedin/cruise-control/wiki/Sensors#executor-sensors) for tracking the inter/intra broker replica movement and also the ongoing replica movement rate/concurrency which can be useful to know if all the replicas movement was successful or if there were some replica movement which didn't complete.
[Work](https://github.com/linkedin/cruise-control/pull/2268) is currently ongoing in the upstream Cruise Control project to have some metrics which can give us the self-healing start and end timestamp based upon the type of anomaly.
To enable the metrics, the user can follow the Strimzi documentation on how we [expose the metrics](https://strimzi.io/docs/operators/latest/full/deploying.html#assembly-metrics-setup-str) through an HTTP endpoint.

### Proposal for enabling self-healing in Strimzi

In order to activate self-healing, users will need to set the `self.healing.enabled` property to `true` and specifying a notifier that the self-healing mechanism is going to use in the `spec.cruiseControl.config` section of their `Kafka` resource.
If the user forgets to set the notifier property then the `KubernetesEventNotifier`(custom notifier under Strimzi org) will be used by default and the user will also get a warning in the Kafka CR regarding missing `anomaly.notifier.class` property.
We will support most anomaly fixes however there are some fixes which we don't want self-healing to fix automatically in a Kubernetes context.
Broker failure or disk failure fix are disabled by default since we have better ways to fix them instead of moving the partition replicas across the brokers.
For example, brokers failures are generally due to lack of resources, so we can fix the issue at cluster level and in the same way failed disks can be easily replaced with a new one, so self-healing in these cases can be more resource consuming.

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
    config:
      # configuration to enable self-healing for all anomalies
      self.healing.enabled: true
      anomaly.notifer.class: <notifier-class> # can be set to `SelfHealingNotifier, SlackSelfHealingNotifier, AlertaSelfHealingNotifier etc. or you own custom notifier
      self.healing.broker.failure.enabled: false # disables self healing for broker failure
      self.healing.disk.failure.enabled: false # disables self healing for disk failure
```

The user can still enable self-healing for disk failures and broker failure if they want by setting the  `self.healing.broker.failure.enabled` for broker failure and `self.healing.disk.failure.enabled` for disk failure to `true`.
In the same way user can enable and disable the self-healing for other anomalies.

#### Using Notifiers when using Self-healing

Users can choose what type of notifier they want Cruise Control to use.
They can use the notifier provided by Cruise Control, such as the `SelfHealingNotifier`, `SlackSelfHealingNotifier`, `AlertaSelfHealingNotifier` and `MSTeamsSelfHealingNotifier` or they can implement their own custom notifier.
There can be cases where the users are not using Slack, MSTeams or Alerta and require a notifier for self-healing to work.
In such cases, the users can use our custom `KubernetesEventsNotifier` which will be available under the Strimzi GitHub organization.
It will provide them a cloud native way to see how anomalies are being detected/ the self-healing operations being performed on the cluster by making use of Kubernetes events.

##### KubernetesEventsNotifier 

Cruise Control provides the `AnomalyNotifier` interface, which has multiple abstract methods on what to do if certain anomalies are detected.
`SelfHealingNotifier` is the base class that contains the logic of self-healing and override the methods in by implementing the `AnomalyNotifier` interface.
Some of those methods are:`onGoalViolation()`, `onBrokerFailure()`, `onDiskFailure`, `alert()` etc.
The `KubernetesEventsNotifier` will be based of the `SelfHealingNotifier` class.
The `approve/ignore/delay` process for `KubernetesEventNotifier` will be similar to that of the `SelfHealingNotifier` class. 
The anomalies with smaller priority value and detected time will be considered priority and resolved first.
In case the anomaly is unfixable then they will be ignored.
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
    kind: Kafka
    name: <cluster-name>
    namespace: <project-namespace>
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
  related: 
    kind: Pod
    name: <cruise-control-pod-name>
    namespace: <project-namespace>
  reportingComponent: cruise-control
  reportingInstance: cruise-control
  source: {}
  type: Normal
```

### Strimzi Operations while self-healing is running/fixing an anomaly 

#### What happens when self-healing is running/fixing an anomaly and brokers are rolled

When an anomaly is detected, Cruise Control may try to fix/mitigate it by moving the partition replicas across the brokers in the Kafka cluster.
If, during the process of moving partitions replicas among the brokers in the Kafka cluster, some broker that was being used gets deleted or rolled, then Cruise Control would finish self-healing process and log that `self-healing finished successful with error`.
Since the self-healing process was not completed successfully, it would start again with a new anomalyId to be fixed properly.

#### Self-healing behaviour with KafkaRebalance 

* If self-healing is ongoing and a `KafkaRebalance` resource is created/applied in the middle of it then Cruise Control will still generate an optimization proposal.
The proposal should be based on the cluster shape at that particular timestamp.
* If self-healing is ongoing and the generated proposal is `approved` then the `KafkaRebalance` resource will move to `NotReady` state, stating that there is already a rebalance ongoing.

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

* In case, a manual rebalance is ongoing and self-healing is triggered, then the self-healing process waits till the rebalancing is over.
* If the KafkaRebalance is generated again using the `refresh` annotation in between of self-healing, then a proposal is still generated based on the cluster shape at that particular timestamp.

## Affected/not affected projects

This change will affect the Strimzi cluster operator and a new repository named `kubernetes-event-notifier` will be added under the Strimzi organisation. 

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
