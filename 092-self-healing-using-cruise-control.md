# Self Healing using Cruise Control

This proposal is about integrating the self-healing feature of Cruise Control into the Strimzi operator.
The self-healing feature of Cruise Control allows us to heal the anomalies detected in the cluster, automatically. 
Anomalies are detected by Cruise Control using the anomaly detector manager.

## Current situation

During normal operations it's common for Kafka clusters to become unbalanced (through partition key skew, uneven partition placement etc.) or for problems to occur with disks or other underlying hardware which makes the cluster unhealthy.  
Currently, if we encounter any such scenario we need to fix these issues manually i.e. if there is some broker failure then we might move the partition replicas from that corrupted broker to some other healthy broker by using the `KafkaRebalance` custom resource in the [`remove-broker`](https://strimzi.io/docs/operators/latest/full/deploying.html#proc-generating-optimization-proposals-str) mode.
With smaller clusters, it is feasible to fix things manually. However, for larger ones it can be very time-consuming, or just not feasible, to fix all the issue on your own.

## Motivation

With the help of the self-healing feature we can resolve the issues like disk failure, broker failure and other issues automatically.
Cruise Control treats these issues as "anomalies", the detection of which is the responsibility of the anomaly detector mechanism.
Currently, users are [able to set]((https://strimzi.io/docs/operators/latest/full/deploying.html#setting_up_alerts_for_anomaly_detection)) the notifier to one of those included with Cruise Control (<list of included notifiers>).
However, any anomaly that is detected would then need to be fixed manually by using the `KafkaRebalance` custom resource.
It would be useful for users of Strimzi to be able to have these anomalies fixed automatically whenever they are detected.

## Proposal

### Introduction

![self-healing flow diagram](./images/090-self-healing-flow.png)

#### Anomaly Detector Manager

The anomaly detector manager helps in detecting the anomalies and also handling the detected anomalies.
It acts as a coordinator between the detector classes as well as the classes which will be handling and resolving the anomalies.
Various detector classes like `GoalViolationDetector`, `DiskFailureDetector`, `KafkaBrokerFailureDetector` etc. are used for the anomaly detection, which runs periodically to check if the cluster have their corresponding anomalies or not (This periodic check can be easily configured through the `anomaly.detection.interval.ms` configuration).
Every detector class works in a different way to detect their corresponding anomalies. A `KafkaBrokerFailureDetector` will require a deep usage of requesting metadata to the Kafka cluster (i.e. broker failure) while if we talk about `DiskFailureDetector` it will be more inclined towards using the admin client API. In the same way the `MetricAnomalyDetector` will be making use of metrics and other detector like `GoalViolationDetector` and `TopicAnomalyDetector` will have their own way to detect anomalies.
The detected anomalies can be of various types:
* Goal Violation - This happens if certain optimization goals are violated (e.g. DiskUsageDistributionGoal etc.). These goals can be configured independently (through the `self.healing.goals` config) to those used for manual rebalancing..
* Topic Anomaly - Can happen when one or more topics in cluster violates user-defined properties (e.g. some partitions are too large in disk).
* Broker Failure - It happens when a non-empty broker crashes or leaves a cluster.
* Disk Failure - This failure happens if one of the non-empty disks fails (related to a Kafka Cluster with JBOD disks).
* Metric anomaly - This failure happens if metrics collected by Cruise Control have some anomaly in their value (e.g. a sudden rise in the log flush time metrics).

The detected anomalies are inserted into a priority queue and the anomaly with the highest property is picked first to resolve.
The anomaly detector manager calls the notifier to get an action regarding, whether the anomaly should be fixed, delayed or ignored.
If the action is fix, then the anomaly detector manager calls the classes that are required to resolve the anomaly.

Anomaly detection also has several other [configurations](https://github.com/linkedin/cruise-control/wiki/Configurations#anomalydetector-configurations), such as the detection interval and the anomaly notifier class, which can effect the performance of the Cruise Control server and the latency of the anomaly detection.

#### Notifiers in Cruise Control

Whenever anomalies are detected, Cruise Control provides the ability to notify the user regarding the detected anomalies using optional notifier classes.
The notification sent by these classes increases the visibility of the operation that are taken by Cruise Control.
However, the notifier class used by Cruise Control is configurable and custom notifiers can be used by setting the `anomaly.notifier.class` property.
The notifier class returns the `action` that is going to be taken on the notified anomaly.
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
For example, the `BrokerFailures` class uses the `RemoveBrokersRunnable` class, the `DiskFailure` class use the `RemoveDisksRunnable` class and so on.
An optimization proposal will then be generated by these `Runnable` classes and that proposal will be applied on the cluster to fix the anomaly.
In case the anomaly detected is unfixable for e.g. violated hard goals that cannot be fixed typically due to lack of physical hardware(insufficient number of racks to satisfy rack awareness, insufficient number of brokers to satisfy Replica Capacity Goal, or insufficient number of resources to satisfy resource capacity goals), the anomaly wouldn't be fixed and the cruise control logs will be updated with `self-healing is not possible due to unfixable goals` warning.
When self-healing starts, you can check the anomaly status by polling the `state` endpoint with the `substate` configured as `anomaly_detector`. This would provide the anomaly details like `anomalyId`, `anomalyType` and the status of the anomaly etc.
For example, the detected anomaly would look like this:
```shell
#...
recentDiskFailures:[{anomalyId=4b1ed191-50d7-4aa1-806b-b8c5c0d679fe, 
detectionDate=2024-12-09T09:49:46Z, 
statusUpdateDate=2024-12-09T09:49:47Z, 
failedDisksByTimeMs={2={/var/lib/kafka/data-1/kafka-log2=1733737786603}}, 
status=FIX_STARTED}]}
```

The anomaly status would change based on how the healing progresses. The anomaly can transition to these possible status:
* DETECTED - The anomaly is just detected and no action is yet taken on it 
* IGNORED - Self healing is either disabled or the anomaly is unfixable
* FIX_STARTED - The anomaly has started getting fixed
* FIX_FAILED_TO_START - The proposal generation for the anomaly fix failed 
* CHECK_WITH_DELAY - The anomaly fix is delayed 
* LOAD_MONITOR_NOT_READY - Monitor which is monitoring the workload of a Kafka cluster is in loading or bootstrap state
* COMPLETENESS_NOT_READY - Completeness not met for some goals. The completeness describes the confidence level of the data in the metric sample aggregator. If this is low then some Goal classes may refuse to produce proposals.

We can also poll the `state` endpoint, with `substate` parameter as `executor`, to get details on the tasks that are currently being performed by Cruise Control.
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

### Enabling self-healing configuration in the Strimzi operator

Every configuration related to self-healing is currently disabled in the operator, and we will need to enable users altering some of those properties.
The setting of these properties will be placed behind a feature gate.

### Feature Gate

The new feature gate will be called `UseSelfHealing`.
It will be introduced in an alpha state and will be disabled by default.

We are using a feature gate for this feature due to various reasons:
* We want to allow users to try this feature and test it.
* We would like to know more about the positive/negatives of this feature by testing it in real deployments.
* Getting feedback from the community would allow us to understand how we can improve this even more.

### Enabling self-healing in the Kafka custom resource

When the feature gate is enabled, the operator will set the `self.healing.enabled` Cruise Control configuration to `true` and allow the users to set other appropriate `self-healing` related [configurations](https://github.com/linkedin/cruise-control/wiki/Configurations#selfhealingnotifier-configurations).

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
```

Where the user wants to enable self-healing for some specific anomalies, they can use configurations like `self.healing.broker.failure.enabled` for broker failure, `self.healing.goal.violation.enabled` for goal violation etc.

### Strimzi Notifier

Cruise Control provides `AnomalyNotifier` interface which has multiple abstract methods on what to do if certain anomalies are detected.
`SelfHealingNotifier` is the base class that contains the logic of self-healing and override the methods in by implementing the `AnomalyNotifier` interface. 
Some of those methods are:`onGoalViolation()`, `onBrokerFailure()`, `onDisk Failure`, `alert()` etc.
The `SelfHealingNotifier` class can then be further used to implement your own notifier
The other custom notifier offered by Cruise control are also based upon the `SelfHealingNotifier`.

We will be using our own notifier implementation called `StrimziNotifer`(set via the `anomaly.notifier.class` config).
Strimzi will configure Cruise Control to always use the `StrimziNotifier` to notify regarding the anomalies and take action on them.

For effective alerts and monitoring, we will be making use of Kubernetes events to signal to users what operations Cruise Control is performing automatically.
The `StrimziNotifier` will override the `alert()` method of the `SelfHealingNotifier` and will trigger and publish events whenever an anomaly is detected.
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

The `StrimziNotifier` class is going to reside in a separate module in the operator repository called the `strimzi-cruise-control-notifier` so that a seperate jar for the notifier can be built for inclusion in the Cruise Control container image.
We will also need to include the fabric8 client dependencies in the notifier JAR that will be used for generating the Kubernetes events as well as for reading the annotation on the Kafka resource from the notifier.

### Pausing the self-healing feature

The user should also have the option to pause the self-healing feature, in case they know or plan to do some rolling restart of the cluster or some rebalance they are going to perform which can cause issue with the self-healing itself.
For this, we will be having an annotation called `strimzi.io/self-healing-paused`.
We can set this annotation on the Kafka custom resource to pause the self-healing feature.
self-healing is not paused by default so missing annotation would mean that anomalies detected will keep getting healed automatically.
When self-healing is paused, it would mean that all the anomalies that are detected after the pause annotation is applied would be ignored and no fix would be done for them.
The anomalies that were already getting fixed would continue with the fix.

#### How the pausing works?

To pause self-healing, you will need to annotate the Kafka custom resource with the following annotation:

```sh
kubectl annotate kafka my-cluster `strimzi.io/self-healing-paused: "true"`
```

The Kafka custom resource will look like this:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  annotations:
     strimzi.io/self-healing-paused: "true"
  creationTimestamp: "2024-12-17T09:01:10Z"
  generation: 1
  name: my-cluster
  namespace: myproject
  resourceVersion: "109971"
  uid: 71928731-df71-4b9f-ac77-68961ac25a2f
```

We will have checks on methods like `onGoalViolation`, `onBrokerFailure`, `onTopicAnomaly` etc. in the `StrimziNotifier` to check if the `pause` annotation is applied on the Kafka custom resource or not.
If the annotation is applied then this check will set the `action` to be taken on the anomaly as `IGNORE` and the anomalies wouldn't be fixed.

### Strimzi Operations while self-healing is running/fixing an anomaly 

Self-healing is a very robust and resilient feature due to its ability to tackle issues which can appear while the healing of cluster is happening.

#### What happens when self-healing is running/fixing an anomaly and brokers are rolled

If a broker fails, self-healing will trigger and Cruise Control will try to move the partition replicas from the failed broker to other healthy brokers in the Kafka cluster.
If, during the process of moving partitions from the failed broker, another broker in the Kafka cluster gets deleted or rolled, then Cruise Control would finish the transfer from the failed broker but log that `self-healing finished successful with error`.
If the deleted or rolled broker comes back before the periodic anomaly check interval, then self-healing would start again only for the earlier failed broker with a new anomalyId since it was not fixed properly. 
In case the deleted broker doesn't come back healthy within the periodic anomaly check interval then another new anomaly for the deleted broker will be created and the earlier detected failed broker would also persist but will be updated with a new anomalyId.

#### What happens when self-healing is running/fixing an anomaly and Kafka Rebalance is applied

If self-healing is ongoing and a `KafkaRebalance` resource is posted in the middle of it then the `KafkaRebalance` resource will move to `NotReady` state, stating that there is already a rebalance ongoing. 
For example
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

## Affected/not affected projects

A new module name `strimzi-notifier` will be added to the operator repository.
The `strimzi-notifier` module should build as a jar and then be included in the cruise control image.

## Rejected Alternatives

### Alternative 1

This alternative is about using a Kubernetes custom resource to create a two-way interaction between the operator and Cruise Control.
The idea was to create a Kubernetes custom resource named `KafkaAnomaly` everytime an anomaly was detected.
The user will get updates regarding the anomaly fix through the generated `KafkaAnomaly` resource which would be updated by the operator by requesting the `state` endpoint with `anomaly_detector` and `executor` substates.

Pros:
* More hold on the self-healing process since everything is driven using the Kafka custom resource.

Cons:
* Very tight coupling with the operator.
* Would be hard to manage multiple `KafkaAnomaly` custom resources (for e.g. Deletion when anomaly is fixed etc.)

### Alternative 2

This alternative is similar to alternative 1 where we will use a Kubernetes custom resource to create a two-way interaction between the operator and Cruise Control.
The idea was to create a Kubernetes custom resource named `KafkaAnomaly` everytime an anomaly was detected and both the operator and the Notifier would watch the resource for updates.
But with this approach the operator will be responsible to make decision regarding the anomaly should be fixed or not

Pros:
* Allows the operator to ignore the anomalies is some task is already running in the cluster like rolling, rebalance etc.

Cons:
* Very tight coupling with the operator.
* Delaying the anomaly detector progress.
