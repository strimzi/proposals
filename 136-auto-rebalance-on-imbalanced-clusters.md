# Auto-rebalance on imbalanced clusters

This proposal introduces support for automatic partition rebalancing whenever a Kafka cluster becomes imbalanced due to unevenly distributed replicas, overloaded brokers, or similar issues.

This feature is opt-in.
When enabled, it allows the Strimzi Operator to automatically execute partition rebalances through `KafkaRebalance` custom resources whenever cluster imbalances are detected by Cruise Control.

## Motivation

Currently, if the cluster is imbalanced, the user would need to manually rebalance the cluster by using the `KafkaRebalance` custom resource.
With Strimzi, if you want to rebalance due to cluster imbalance, the only available approach today is manual intervention - there is no automated mechanism for rebalancing on imbalance.
Automating this process would reduce operational overhead and benefit all users by eliminating the need for constant manual monitoring and rebalancing.

## Introduction to Self Healing in Cruise Control

In order to understand how we plan to automatically fix unbalanced Kafka clusters, the sections below will go over how Cruise Control's anomaly detection and self-healing features work to detect the anomalies in the cluster and fix them.

![self-healing flow diagram](images/136-self-healing-flow.png)

The above flow diagram depicts the self-healing process in Cruise Control.
The anomaly detector manager detects an anomaly (using the detector classes) and forwards it to the notifier.
The notifier then decides what action to take on the anomaly whether to fix it, ignore it, or delay it.
Cruise Control provides various notifiers to alert the users about the detected anomaly in several ways like Slack, Alerta, MS Teams etc.

### Anomaly Detector Manager

The anomaly detector manager helps in detecting the anomalies as well as handling them.
It acts as a coordinator between the detector classes and the classes which will handle resolving the anomalies.
Various detector classes like `GoalViolationDetector`, `DiskFailureDetector`, `KafkaBrokerFailureDetector` etc. are used for the anomaly detection, which runs periodically to check if the cluster has their corresponding anomalies or not.
The frequency of this check can be changed via the `anomaly.detection.interval.ms` configuration.
Detector classes use different mechanisms to detect their corresponding anomalies.
For example, broker, disk, and topic failure detection utilize the Kafka Admin API, metric anomaly detection uses collected metrics, and goal violation detection uses the load distribution calculated by Cruise Control itself.
The detected anomalies can be of various types:
* Goal Violation - This happens if certain [optimization goals](https://strimzi.io/docs/operators/in-development/deploying#optimization_goals) are violated (e.g. DiskUsageDistributionGoal etc.). These goals can be configured through the `anomaly.detection.goals` option in the Cruise Control configuration. However, this option is currently forbidden in the `spec.cruiseControl.config` section of the `Kafka` CR.
* Topic Anomaly - When one or more topics in the cluster violate user-defined properties (e.g. some partitions are too large on disk, the replication factor of a topic is different from a given default value etc).
* Broker Failure - This happens when a non-empty broker crashes or leaves a cluster for a long time.
* Disk Failure - This failure happens when one of the non-empty disks fails (in a Kafka cluster with JBOD disks).
* Metric anomaly - This failure happens if metrics collected by Cruise Control have some anomaly in their value (e.g. a sudden rise in the log flush time metrics).

The detected anomalies are inserted into a priority queue where the comparator is based upon the priority value.
The smaller the priority value is, the higher priority the anomaly type has.

The anomaly detector manager calls the notifier to get an action regarding whether the anomaly should be fixed, delayed, or ignored.
If self-healing is enabled and the action is `FIX`, then the anomaly detector manager calls the classes that are required to resolve the anomaly.

Anomaly detection also has various [configurations](https://github.com/linkedin/cruise-control/wiki/Configurations#anomalydetector-configurations), such as the detection interval and the anomaly notifier class, which can affect the performance of the Cruise Control server and the latency of the anomaly detection.

### Notifiers in Cruise Control

Whenever anomalies are detected, Cruise Control provides the ability to notify the user regarding the detected anomalies using optional notifier classes.
The notification sent by these classes increases the visibility of the operations that are taken by Cruise Control.
The notifier class used by Cruise Control is configurable and custom notifiers can be used by setting the `anomaly.notifier.class` property.
The notifier class returns the `action` that is going to be taken on the flagged anomaly.
These actions have three types:
* `FIX` - Fix the anomaly
* `CHECK` - Check the anomaly at a later time
* `IGNORE` - Ignore the anomaly

The default notifier enabled by Cruise Control is the `NoopNotifier`.

* This notifier always returns `IGNORE` which means that the anomaly detector manager won't use any class to fix the anomaly.
* This notifier doesn't implement any notification mechanism to the users.

Cruise Control also provides [custom notifiers](https://github.com/linkedin/cruise-control/wiki/Configure-notifications) like Slack Notifier, Alerta Notifier etc. for notifying users regarding the anomalies.

Users can develop their own notifiers by implementing Cruise Control's `AnomalyNotifier` interface and configuring it via the `anomaly.notifier.class` property in the Cruise Control configuration.
This pluggable mechanism provides flexibility for integrating with organization-specific alerting, ticketing, or approval systems.

There are multiple other [self-healing notifier](https://github.com/linkedin/cruise-control/wiki/Configurations#selfhealingnotifier-configurations) related configurations you can use to make notifiers more efficient as per the use case.

### Self Healing

If self-healing is enabled, Cruise Control will either fix or not fix the detected anomaly based on the action returned by the configured notifier.
If the notifier has returned `FIX` as the action then the classes which are responsible for resolving the anomaly would be called.
Each detectable anomaly is handled by a specific detector class which then uses another remediation class to run a fix.
For example, the `GoalViolations` class uses the `RebalanceRunnable` class, the `DiskFailure` class use the `RemoveDisksRunnable` class and so on.
An optimization proposal (a collection of replica reassignments and partition leadership changes) will then be generated by these `Runnable` classes and that proposal will be applied on the cluster to fix the anomaly.
In case the anomaly detected is unfixable for e.g. violated hard goals that cannot be fixed typically due to lack of physical hardware (insufficient number of racks to satisfy rack awareness, insufficient number of brokers to satisfy Replica Capacity Goal, or insufficient number of resources to satisfy resource capacity goals), the anomaly wouldn't be fixed and the Cruise Control will log a warning with the `self-healing is not possible due to unfixable goals` message.

## Current situation

Even under normal operation, it is common for Kafka clusters to encounter problems such as partition key skew leading to an uneven partition distribution, or hardware issues like disk failures, which can degrade the overall cluster's health and performance.
In Strimzi's current implementation of Cruise Control, fixing these issues needs to be triggered manually i.e. if the cluster is imbalanced then a user might instruct Cruise Control to move the partition replicas across the brokers in order to fix the imbalance using the `KafkaRebalance` custom resource.

Users can currently enable anomaly detection and can also [set](https://strimzi.io/docs/operators/latest/full/deploying.html#setting_up_alerts_for_anomaly_detection) the notifier to one of those included with Cruise Control (`SelfHealingNotifier`, `AlertaSelfHealingNotifier`, `SlackSelfHealingNotifier` etc.).
All the `self.healing` prefixed properties are currently disabled in Strimzi's Cruise Control integration because, initially, it was not clear how self-healing would act if pods were rolled in the middle of rebalances or how Strimzi triggered manual rebalances should interact with Cruise Control triggered self-healing ones.

## Proposal

This proposal is concerned with giving users the option to have their Kafka clusters balanced automatically whenever they become imbalanced due to overloaded brokers, high CPU usage, or similar constraints.
However, the proposal is not to rely on the Cruise Control self-healing implementation directly.
If we were to enable that functionality then, in response to detected anomalies, Cruise Control would issue partition reassignments without involving the Strimzi Cluster Operator.
This could cause potential conflicts with other administration operations and is the primary reason self-healing has been disabled until now.
To resolve this issue, we will only make use of Cruise Control's anomaly detection ability, the triggering of the partition reassignments (rebalance) will be the responsibility of the Strimzi Cluster Operator.
To enable this, we will use an approach based on the existing auto-rebalance for scaling feature (see the [documentation](https://strimzi.io/docs/operators/latest/deploying#proc-automating-rebalances-str) for more details).
We propose using only those anomaly detection classes, related to goal violations, that can be addressed by a partition rebalance.
We will not enable the other anomaly detection classes, related to goal violations, that would require manual interventions at the infrastructure level such as disk or broker failures.
For the latter case, it would be better to spin up new disks or to fix the issue with the broker(s) directly instead of just moving the partitions replicas away from them through rebalancing.
Therefore, given the interventions required, it would be non-trivial for the Strimzi Operator to fix these failures automatically.
Hence, these are not goals of this proposal and could be addressed later in separate proposals.
Following the above approach will provide several advantages:
* we ensure that the operator controls all rebalance and cluster remediation operations.
* using the existing `KafkaRebalance` CR system gives more visibility into what is happening and when, which (as we don't support the Cruise Control UI) enhances observability and will also aid in debugging.

### `imbalance` mode in Strimzi's auto-rebalancing feature

The [`auto-rebalancing`](https://strimzi.io/docs/operators/latest/deploying#proc-automating-rebalances-str) feature in Strimzi allows the operator to run a rebalance automatically when a Kafka cluster is scaled up (by adding brokers) or scaled down (by removing brokers).

Auto-rebalancing in Strimzi currently supports two modes:
* `add-brokers` - auto-rebalancing on scale up
* `remove-brokers` - auto-rebalancing on scale down

To leverage the automated rebalance on imbalanced clusters (those with detected goal violations), we will be introducing a new mode to the auto-rebalancing feature.
The new mode will be called `imbalance`, which means that cluster imbalance was detected and rebalancing should be applied to all the brokers.
The imbalance mode is configured by creating a new object in the `spec.cruiseControl.autoRebalance` list with its `mode` field set to `imbalance` and the corresponding rebalancing configuration is defined as a reference to a `KafkaRebalance` template custom resource, by using the `spec.cruiseControl.autoRebalance.template` field as a [LocalObjectReference](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/local-object-reference/).
The `template` field is optional and if not specified, the auto-rebalancing runs with the default Cruise Control configuration (i.e. the same used for unmodified manual `KafkaRebalance` invocations).
To provide users more flexibility, they only have to configure the auto-rebalance modes they wish to use, whether it be `add-brokers`, `remove-brokers`, or `imbalance`.
When the auto-rebalance configuration is set with `imbalance` mode enabled, the operator will trigger a partition rebalance whenever a fixable goal violation is detected by the anomaly detector.
For the operator to trigger the auto-rebalance, it must be aware that the cluster is imbalanced due to a goal violation anomaly.
We will make use of the `state` endpoint of Cruise Control to do this.
The operator will use this endpoint to understand if there are any goal violations so that the operator can trigger a rebalance (see section "Reconciliation logic using the state endpoint" below).
With this proposal, we will only support auto-rebalance for cluster imbalances caused by goal violations.
We also plan to implement the same for topic and metrics related issues, but these will be part of future work since their implementation requires different approaches.
For example, when dealing with topic related issues, it will require a coordination with topic operator and metrics issues will require coordination with the Kafka API.

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
      # using the imbalance rebalance mode
      - mode: imbalance
        template:
          name: my-imbalance-rebalance-template
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
      # using the default Cruise Control rebalancing configuration when doing rebalance on imbalance
      - mode: imbalance
```

The configuration for Cruise Control rebalance is provided through a `KafkaRebalance` custom resource which is specifically defined as a template and is referenced in the `template` field in the example above.
A `KafkaRebalance` custom resource with the `strimzi.io/rebalance-template: true` annotation set can be used as a template.
When a `KafkaRebalance` with this annotation is created, the cluster operator doesn't run any rebalancing.
This is not an actual rebalance request to get an optimization proposal; it is simply where the configuration for auto-rebalancing is defined.

Here is an example template:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-imbalance-rebalance-template
  annotations:
    strimzi.io/rebalance-template: "true" # specifies that this `KafkaRebalance` is a rebalance configuration template
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
When the `template` is set, the operator automatically creates (or updates) a corresponding `KafkaRebalance` custom resource (based on the template) when a fixable anomaly is detected and notified by the `state` endpoint.
The operator copies over goals and rebalancing options from the referenced `template` resource to the generated rebalancing one.
The copied over goals from the template are the goals that will be taken into consideration when the rebalance is running. 
In case we don't provide a template, then the default Cruise Control rebalance configuration and goals will be used when doing the rebalance.

The anomaly detection goals are the goals that will be used by the user to tell Cruise Control what type of anomalies it needs to track. For example:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: test-cluster
spec:
  kafka:
    version: 4.3.0
    metadataVersion: 4.3-IV0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2

  cruiseControl:
    config:
      anomaly.detection.goals: >
        com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.MinTopicLeadersPerBrokerGoal
```

The default anomaly detection goals set by the operator are `RackAwareGoal`, `MinTopicLeadersPerBrokerGoal`, `ReplicaCapacityGoal`, `DiskCapacityGoal`.
If the user has not configured the anomaly detection goals in Cruise Control section of the Kafka CR, then the operator will set the default anomaly detection goals configured by the operator to be used by the anomaly detector.

**Template Goal Validation:**

If the user specifies a rebalance template, the operator validates that the goals in the template are a superset of the anomaly detection goals configured for Cruise Control.
This ensures that the resulting rebalance proposal will actually try to address the root cause of the anomaly that was detected. 
If the anomaly is due to a disk distribution goal violation but your rebalance optimization proposal does not use the DiskDistributionGoal, then it is wrong.

**When validation occurs:**
- During every `Kafka` CR reconciliation cycle, as part of the `KafkaAutoRebalanceReconciler.reconcile()` method
- The validation happens **after** Cruise Control is deployed but **before** any anomaly detection or rebalancing logic executes
- The operator first checks if the `imbalance` mode is configured in `spec.cruiseControl.autoRebalance`
- If `imbalance` mode is enabled and a template is specified, the operator validates that:
  - All goals in the `anomaly.detection.goals` configuration (or defaults if not configured) exist in the template goals.
  - The `anomaly.detection.goals` configuration is currently ignored by the operator (which uses a hard coded default set), but with this proposal, the users will be able to configure the goals and have them applied to Cruise Control.
  - The template reference points to a valid `KafkaRebalance` resource with the `strimzi.io/rebalance-template: true` annotation
- This validation runs on **every reconciliation**, ensuring configuration changes are caught immediately
- Configuration errors are detected early, preventing invalid auto-rebalance attempts and providing immediate feedback to users

**How the operator retrieves anomaly detection goals:**

The `KafkaAutoRebalanceReconciler` retrieves the anomaly detection goals from the Kafka CR's Cruise Control configuration:
1. The operator accesses `kafkaCr.getSpec().getCruiseControl().getConfig()` which returns a `Map<String, Object>`
2. It looks up the `anomaly.detection.goals` key using `CruiseControlConfigurationParameters.ANOMALY_DETECTION_CONFIG_KEY.getValue()`
3. The value is a comma-separated string of goal names (e.g., `"RackAwareGoal,ReplicaCapacityGoal,DiskCapacityGoal"`)
4. If the key is not present or empty, the operator uses the default anomaly detection goals defined in `CruiseControlConfiguration.CRUISE_CONTROL_DEFAULT_ANOMALY_DETECTION_GOALS_LIST`.
   Currently, the list contains:
   - `RACK_AWARENESS_GOAL`
   - `MIN_TOPIC_LEADERS_PER_BROKER_GOAL`
   - `REPLICA_CAPACITY_GOAL`
   - `DISK_CAPACITY_GOAL`
5. The operator parses the goals string into a list and compares it against the template goals to ensure the template goals are a subset.

**Validation behavior:**
- If validation **passes**: Auto-rebalance proceeds normally when anomalies are detected
- If validation **fails**: 
  - A `Warning` condition is added to the `Kafka` CR status
  - Auto-rebalance for the `imbalance` mode is **suspended** (not triggered)
  - Other operations (manual rebalances, scale-up/down auto-rebalances) continue normally
  - Users must correct the template configuration to resume imbalance auto-rebalancing

The `KafkaRebalance` has 4 modes: `full`, `add-broker`, `remove-broker` and `remove-disks` mode.
The `imbalance` mode will use the `full` mode in the generated KafkaRebalance resource which means that the generated `KafkaRebalance` custom resource will have the mode set as `full` which within the Strimzi rebalancing operator means calling the Cruise Control API to run a rebalancing taking all brokers into account.

The generated `KafkaRebalance` custom resource will be called `<my-cluster-name>-auto-rebalancing-imbalance`, where the `<my-cluster-name>` part comes from the `metadata.name` in the `Kafka` custom resource, and the `-imbalance` suffix indicates that this rebalance was triggered by detected goal violations in the cluster.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-imbalance
  finalizers:
    - strimzi.io/auto-rebalancing
spec:
  mode: full
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

The operator sets a finalizer, named `strimzi.io/auto-rebalancing`, on the generated `KafkaRebalance` custom resource.
This finalizer mechanism is also used for the existing auto-scaling operations (scale-up and scale-down) to prevent the user, or any other tooling, from deleting the resource while the auto-rebalancing is still running.
The finalizer is removed when Cruise Control indicates that the partition reassignment (rebalance) process has finished, allowing the generated `KafkaRebalance` custom resource to be deleted by the operator itself.
In case the rebalance finishes with an error, the error message will be propagated to the Kafka custom resource just like we do for the `remove-broker` and `add-broker` operations, and the generated `KafkaRebalance` will be deleted.

### State endpoint in Cruise Control

Cruise Control provides the `state` endpoint to query the state of Cruise Control at any time by issuing a HTTP GET request.
The operator will use this endpoint with specific substates to coordinate auto-rebalancing operations.

**Anomaly Detector Substate (`state?substates=anomaly_detector`)**

The anomaly detector substate provides information about detected anomalies in the cluster, including goal violations.
The operator queries this substate to identify when cluster imbalances occur and determine whether an auto-rebalance should be triggered.
The response includes details about fixable and unfixable goal violations, anomaly timestamps, and the current status of detected anomalies.

For more information, refer to the [state endpoint documentation](https://github.com/linkedin/cruise-control/wiki/rest-apis#query-the-state-of-cruise-control).

### Flow diagram depicting process of auto-rebalance on imbalance

The following diagram depicts the complete auto-rebalance on imbalance process, showing how the operator queries the Cruise Control state endpoint, detects goal violations, and triggers rebalance operations:

![flow-diagram-auto-rebalance-on-imbalance](images/136-auto-rebalance-imbalance.png)

### Reconciliation logic using the state endpoint

The operator will interact with the Cruise Control `state` endpoint to understand when an anomaly is detected and when it should take any action.
The `KafkaAutoRebalanceReconciler` class will hold all the logic for the auto-rebalance triggered on scale up, scale down or imbalance.
The reconciler currently detects any sort of scale down or scale up automatically every reconciliation.
We will configure the KafkaAutoRebalanceReconciler to detect the imbalanced cluster every reconciliation too. 
But we also need to make sure that we don't detect any goal violations when a rebalance is already ongoing (manual or automatic rebalance).
An ongoing rebalance can mean that some goal violation which might be detected in that particular reconciliation will be fixed already by the rebalance and we don't need any extra rebalance.

On every reconciliation, the operator checks the cluster state in this order and executes the **first** operation that applies:

1. **Scale-down detected** → Execute scale-down rebalance immediately (if an imbalance rebalance is running, it is stopped)
2. **Scale-up detected** → Execute scale-up rebalance immediately (if an imbalance rebalance is running, it is stopped)
3. **Goal violation detected** → Execute imbalance rebalance (only if no scale operation is in progress, and maintenance time windows are satisfied if configured)

If a scale operation interrupts an imbalance rebalance, the imbalance rebalance will be stopped and the corresponding `KafkaRebalance` will be deleted to start the scaling operation.
After the scale operation completes, if the cluster is still imbalanced, Cruise Control will detect the goal violation again in a future reconciliation, and a fresh imbalance rebalance will be triggered.
This prevents wasted work: scale operations change the cluster topology, making any in-progress imbalance rebalance based on outdated information.

### Detecting Active Rebalances

To determine if a rebalance is already ongoing, the operator uses a unified approach that works for both manual and auto-generated `KafkaRebalance` resources.
Every reconciliation, the operator will:

1. **List all `KafkaRebalance` resources** for the cluster using `kafkaRebalanceOperator.listAsync(namespace, Labels.fromMap(Map.of(Labels.STRIMZI_CLUSTER_LABEL, clusterName)))`
2. **Extract the state** of each `KafkaRebalance` using the existing `KafkaRebalanceUtils.rebalanceState()` utility method
3. **Check for actively executing rebalances**: A rebalance is considered "active" if any `KafkaRebalance` resource is in the `Rebalancing` state (partition movements are currently executing)

#### Interaction with Manual Rebalances

The operator handles manual and auto-generated `KafkaRebalance` resources differently based on their state:

- **Manual rebalances in `New`, `PendingProposal`, or `ProposalReady` states**: 
  - These represent rebalances that were created by users but not yet approved
  - The operator will **ignore** these manual rebalances and proceed with auto-rebalance on imbalance
  - A log message will be emitted: `Manual KafkaRebalance {name} is in {state} state and will be ignored. Auto-rebalance on imbalance will proceed.`

- **Manual rebalances in `Rebalancing` state**:
  - The operator will **skip** auto-rebalance creation and wait for the manual rebalance to complete
  - A log message will be emitted: `Manual KafkaRebalance {name} is actively rebalancing. Auto-rebalance on imbalance will be skipped.`

- **Auto-generated rebalances** (identified by `strimzi.io/auto-rebalancing` finalizer):
  - These take precedence as they are managed by the auto-rebalance state machine
  - Scale-up/scale-down rebalances will interrupt imbalance rebalances as described in the FSM section

The implementation leverages the existing `KafkaRebalanceUtils` class which is already used throughout the Strimzi codebase for state extraction and validation.
All `KafkaRebalance` resources (manual and auto-generated) have the `strimzi.io/cluster` label, allowing the operator to discover them via label selector.

If there is no actively executing rebalance (i.e., no `Rebalancing` state detected), then we can move forward to check if there were any detected goal violations or not.
The `KafkaAutoRebalanceReconciler` will query the Cruise Control state endpoint with the anomaly detector substate during every reconciliation if no rebalance is already running.
The operator will query the Cruise Control service:
```
GET http://<cluster-name>-cruise-control:9090/kafkacruisecontrol/state?substates=anomaly_detector&json=true
```
Here is an example JSON returned by the state endpoint when called with the `anomaly_detector` substate:
```json
{
  "AnomalyDetectorState": {
    "selfHealingEnabled": [
    ],
    "selfHealingDisabled": [
      "BROKER_FAILURE",
      "DISK_FAILURE",
      "GOAL_VIOLATION",
      "METRIC_ANOMALY",
      "TOPIC_ANOMALY",
      "MAINTENANCE_EVENT"
    ],
    "selfHealingEnabledRatio": {
      "BROKER_FAILURE": 1.0,
      "DISK_FAILURE": 1.0,
      "GOAL_VIOLATION": 1.0,
      "METRIC_ANOMALY": 1.0,
      "TOPIC_ANOMALY": 1.0,
      "MAINTENANCE_EVENT": 1.0
    },
    "recentGoalViolations": [
      {
        "anomalyId": "c2071b83-b011-4924-8fa9-8d3cb0b2ebb9",
        "detectionDate": "2024-12-17T09:26:00Z",
        "statusUpdateDate": "2024-12-17T09:26:04Z",
        "fixableViolatedGoals": [
          "DiskUsageDistributionGoal"
        ],
        "unfixableViolatedGoals": [],
        "status": "FIX_STARTED"
      }
    ],
    "recentBrokerFailures": [],
    "recentMetricAnomalies": [],
    "recentDiskFailures": [],
    "recentTopicAnomalies": [],
    "recentMaintenanceEvents": [],
    "metrics": {
      "meanTimeBetweenAnomalies": {
        "GOAL_VIOLATION": 3.28,
        "BROKER_FAILURE": 0.0,
        "METRIC_ANOMALY": 0.0,
        "DISK_FAILURE": 0.0,
        "TOPIC_ANOMALY": 0.0
      },
      "meanTimeToStartFix": 2890.0,
      "numSelfHealingStarted": 1,
      "numSelfHealingFailedToStart": 0,
      "ongoingAnomalyDuration": 0.0
    },
    "ongoingSelfHealingAnomaly": "c2071b83-b011-4924-8fa9-8d3cb0b2ebb9",
    "balancednessScore": 88.45
  },
  "version": 1
}
```
The `recentGoalViolations` field in the above JSON depicts which goal violations happened in the cluster and when they occurred.
The `recentGoalViolations` list will grow as new violations are detected, but we will always be interested in the latest violated goal.
This is because the `recentGoalViolations` list contains both violations that were already fixed by previous rebalances as well as violations that still need to be addressed.
We will process only the most recent goal violation (the first element in the list when sorted by `detectionDate` in descending order) because earlier violations in the list may have already been resolved by previous rebalance operations.
We will parse the data from Cruise Control into a JSON object and retrieve the `detectionDate` from the most recent entry in the `recentGoalViolations` field.
This `detectionDate` timestamp will be used to determine whether a new auto-rebalance should be triggered.

### Logic to trigger rebalance

To determine whether a detected anomaly should trigger a new rebalance, we need to compare the anomaly detection timestamp against the completion time of the most recent rebalance operation (whether manual or automatic).
This ensures we don't retrigger rebalances for goal violations that were already addressed.

**Tracking Rebalance Completion Times:**

Since auto-generated `KafkaRebalance` resources are deleted upon completion or failure, and manual `KafkaRebalance` resources can be deleted by users, we need a persistent location to track completion times.
The operator will use a **ConfigMap** to store this information.

**ConfigMap: `<cluster-name>-rebalancing-tracker`**

This ConfigMap:
- Stores the timestamp of the most recent rebalance completion
- Is created in the same namespace as the Kafka cluster
- Persists even after `KafkaRebalance` resources are deleted
- Allows the operator to prevent duplicate rebalance triggers for already-fixed anomalies
- Is managed by the `KafkaRebalanceAssemblyOperator` (which already owns all `KafkaRebalance` reconciliation)

**ConfigMap Lifecycle:**

- **Creation**: Created automatically by the operator when the first rebalance (manual or auto-generated) completes. The ConfigMap is created with an **owner reference** to the Kafka CR, ensuring it is tied to the cluster lifecycle.
- **Updates**: Updated by `KafkaRebalanceAssemblyOperator` whenever any `KafkaRebalance` resource (manual or auto-generated) transitions to a terminal state (`Ready`, `NotReady`, or `Stopped`)
- **Persistence**: The ConfigMap is **not deleted** when rebalances complete or when the Kafka cluster is idle
- **Deletion**: Automatically deleted when the Kafka cluster itself is deleted (via owner reference to the Kafka CR) or when Cruise Control is deleted/disabled.
- **Rationale**: Persisting the ConfigMap allows accurate timestamp comparison across reconciliations and operator restarts, preventing duplicate rebalancing operations

Example ConfigMap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-cluster-rebalancing-tracker
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/kind: Kafka
  ownerReferences:
  - apiVersion: kafka.strimzi.io/v1
    kind: Kafka
    name: my-cluster
    uid: a1b2c3d4-e5f6-7890-abcd-1234567890ef
    blockOwnerDeletion: true
    controller: false
data:
  lastRebalanceCompletionTime: "2026-06-03T14:23:45Z"
```

**ConfigMap Data Fields:**

1. **`lastRebalanceCompletionTime`**: ISO-8601 timestamp of the most recent rebalance completion (manual or auto-generated)

**Update Logic:**

The operator leverages the **existing `KafkaRebalanceAssemblyOperator`** which already watches and reconciles all `KafkaRebalance` resources.
The ConfigMap update logic is added directly into `KafkaRebalanceAssemblyOperator.reconcileKafkaRebalance()` method, which processes every `KafkaRebalance` resource (whether auto-generated or manual).

**How it works:**

`KafkaRebalanceAssemblyOperator` already reconciles every `KafkaRebalance` resource and updates their status. We extend this to also update the ConfigMap when a rebalance reaches a terminal state:

```java
// In KafkaRebalanceAssemblyOperator.reconcileKafkaRebalance()
// After status update logic

KafkaRebalanceState currentState = KafkaRebalanceUtils.rebalanceState(kafkaRebalance.getStatus());

// Update ConfigMap for terminal states
if (currentState == Ready || currentState == NotReady || currentState == Stopped) {
    String clusterName = kafkaRebalance.getMetadata().getLabels().get(Labels.STRIMZI_CLUSTER_LABEL);
    if (clusterName != null) {
        // Update the tracker ConfigMap with the completion timestamp
        updateRebalanceCompletionTime(
            kafkaRebalance.getMetadata().getNamespace(), 
            clusterName, 
            Instant.now()
        );
    }
}

// Continue with existing reconciliation logic...
```

This approach:
- **Leverages existing infrastructure**: Uses the `KafkaRebalanceAssemblyOperator` watch that already processes all `KafkaRebalance` resources
- **Single source of truth**: Every rebalance (auto-generated or manual) is processed by the same operator
- **Automatic coverage**: The ConfigMap is updated for ANY rebalance that completes, regardless of how it was created
- **Consistent with ownership**: `KafkaRebalanceAssemblyOperator` owns `KafkaRebalance` reconciliation, so it's natural for it to track completion times
- **No duplication**: Avoids listing/checking rebalances in multiple places

**Comparison Logic:**

A rebalance on imbalance will be triggered when:
```java
ConfigMap stateConfigMap = getConfigMap(namespace, clusterName + "-rebalancing-tracker");
String lastCompletionTime = stateConfigMap != null ? stateConfigMap.getData().get("lastRebalanceCompletionTime") : null;

if (lastCompletionTime == null) {
    // No rebalance has ever completed - safe to trigger
    triggerRebalance();
} else if (anomalyDetectionDate.compareTo(Instant.parse(lastCompletionTime)) > 0) {
    // Anomaly detected after last rebalance completion - trigger new rebalance
    triggerRebalance();
} else {
    // Anomaly detected before last rebalance completion - already fixed, skip
}
```

This ensures anomalies fixed by either manual or auto-rebalances don't trigger duplicate operations, even if the `KafkaRebalance` resources have been deleted.

### What happens if there are no rebalances running and an anomaly is detected (The Happy Path)

1. The operator retrieves the `lastRebalanceCompletionTime` from the `<cluster-name>-rebalancing-tracker` ConfigMap and compares the anomaly `detectionDate` against this timestamp (see comparison logic in "Logic to trigger rebalance" section)
2. If the comparison indicates a new anomaly, the operator transitions to `RebalanceOnImbalance` and creates a new `KafkaRebalance` resource
3. Once the rebalance completes (transitions to `Ready` state), the `KafkaRebalanceAssemblyOperator` automatically updates the `lastRebalanceCompletionTime` field in the ConfigMap during its reconciliation
4. The `KafkaAutoRebalanceReconciler` then detects the completion, transitions back to `Idle`, and deletes the auto-generated resource

### What happens if the anomaly is detected during a running rebalance

Using the detection approach described in [Detecting Active Rebalances](#detecting-active-rebalances) section:
- If any `KafkaRebalance` resource in `Rebalancing` state is detected (actively executing partition movements), we skip the auto-rebalance creation since the ongoing operation might resolve the anomaly
- Manual rebalances in `New`, `PendingProposal`, or `ProposalReady` states are ignored (auto-rebalance proceeds as described in [Interaction with Manual Rebalances](#interaction-with-manual-rebalances))

If the anomaly persists after the rebalance completes, Cruise Control will detect it again in a subsequent cycle and a new rebalance will be triggered.

### What happens if a rebalance fails

When an auto-rebalance on imbalance fails (the `KafkaRebalance` resource transitions to `NotReady` state):

1. **ConfigMap updated automatically**: The `KafkaRebalanceAssemblyOperator` updates the `<cluster-name>-rebalancing-tracker` ConfigMap's `lastRebalanceCompletionTime` when it reconciles the `NotReady` state
2. **Failure logged**: The `KafkaAutoRebalanceReconciler` logs the failure at WARN level with the error message from the `KafkaRebalance` status
3. **State transition**: Moves the auto-rebalance state machine from `RebalanceOnImbalance` back to `Idle`
4. **Cleanup**: Removes the failed `KafkaRebalance` resource

**Retry Behavior:**

The operator does NOT immediately retry a failed rebalance.
Instead:
- The ConfigMap's `lastRebalanceCompletionTime` now reflects the failure timestamp
- If the underlying issue persists, Cruise Control will detect the goal violation again in the next anomaly detection cycle
- The new detection will have a later `detectionDate` timestamp
- When compared against `lastRebalanceCompletionTime` (the failure timestamp), the new detection will trigger a fresh rebalance attempt
- This prevents repeated failed attempts while ensuring issues are eventually addressed once resolved
- Users can monitor operator logs or set up log aggregation to track rebalance failures

### What happens if an unfixable goal violation happens

The `recentGoalViolation` property in the JSON returned by the `state` endpoint with `anomaly_detector` substate also provides information on fixable goals and unfixable goals under the `fixableViolatedGoals` and `unfixableViolatedGoals` lists in the JSON.
Here is an example JSON:

```json
    "recentGoalViolations": [
      {
        "anomalyId": "c2071b83-b011-4924-8fa9-8d3cb0b2ebb9",
        "detectionDate": "2024-12-17T09:26:00Z",
        "statusUpdateDate": "2024-12-17T09:26:04Z",
        "fixableViolatedGoals": [
          "DiskUsageDistributionGoal"
        ],
        "unfixableViolatedGoals": [],
        "status": "FIX_STARTED"
      }
    ]
```

**Important Edge Case:** If a goal violation contains BOTH fixable goals and unfixable goals (i.e., both `fixableViolatedGoals` and `unfixableViolatedGoals` lists are non-empty), the operator will NOT trigger a rebalance. This follows Cruise Control's design principle that unfixable goals typically indicate infrastructure constraints (e.g., insufficient racks for rack-awareness, insufficient brokers for replica capacity) that cannot be resolved through partition reassignment alone. Attempting to rebalance only the fixable goals while unfixable goals remain would result in an incomplete fix and could lead to repeated rebalance attempts.

The operator can retrieve the unfixable goals from the `unfixableViolatedGoals` list in the JSON response.
When unfixable goals are detected, the operator will set a warning condition in the status of the `Kafka` CR to alert users that manual intervention is required:

```yaml
status:
  autoRebalance:
    state: Idle 
    lastTransitionTime: "2025-09-22T16:04:00Z"
    modes:
      - mode: imbalance
  conditions:
    - type: Warning
      status: "True"
      reason: AutoRebalanceOnImbalanceFailure
      message: "Unfixable goals detected: [DiskDistributionGoal]. Auto-rebalance blocked until resolved. Fixable goals pending: [NetworkInboundCapacityGoal]."
```

The warning message will include both the unfixable goals (requiring manual intervention) and any fixable goals (which will be addressed once the infrastructure constraints are resolved).

**Manual Workaround**

When unfixable hard goals block auto-rebalance but fixable goals need urgent attention (e.g., `RackAwareGoal` is unfixable due to insufficient racks while `DiskUsageDistributionGoal` violations are getting worse), users can manually trigger a rebalance that skips the unfixable hard goals:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: manual-rebalance-skip-rack-goal
  namespace: kafka
spec:
  goals:
    - ReplicaCapacityGoal
    - DiskCapacityGoal
    - DiskUsageDistributionGoal
    # RackAwareGoal intentionally excluded
  skipHardGoalCheck: true  # Required to skip hard goals
```

This allows addressing fixable violations while infrastructure changes (adding racks, nodes, or resources) are in progress.

### What happens if the rebalance template contains invalid goals

If the `anomaly.detection.goals` configuration contains goals that are NOT included in the rebalance template, the validation will fail.
This ensures that when Cruise Control detects violations for monitored goals, the template includes those goals so the operator can fix the detected violations.

For example, if the anomaly detection goals are configured as `[RackAwareGoal, ReplicaCapacityGoal, DiskCapacityGoal]` but the rebalance template only specifies `[RackAwareGoal, DiskCapacityGoal]`, the validation will fail because `ReplicaCapacityGoal` is monitored for violations but missing from the template.

The template can include additional goals beyond what's monitored (e.g., template has `[RackAwareGoal, DiskCapacityGoal, CpuCapacityGoal]` while anomaly detection only monitors `[RackAwareGoal, DiskCapacityGoal]`) - this allows more comprehensive rebalancing.

When this validation fails, the operator will:
1. Add a warning condition to the `Kafka` CR status
2. NOT trigger any auto-rebalances for the `imbalance` mode until the configuration is corrected
3. Log a warning message indicating which goals failed validation

Example warning condition:

```yaml
status:
  autoRebalance:
    state: Idle
    lastTransitionTime: "2025-09-22T16:04:00Z"
    modes:
      - mode: imbalance
  conditions:
    - type: Warning
      status: "True"
      reason: InvalidRebalanceTemplateGoals
      message: "Anomaly detection goals [ReplicaCapacityGoal, NetworkInboundCapacityGoal] are missing from template 'my-imbalance-rebalance-template'. Add missing goals to template."
```

Users must either:
- Update the rebalance template to include all goals from the anomaly detection configuration, or
- Update the Cruise Control `anomaly.detection.goals` configuration to remove the goals that are missing from the template

### Stopping a running rebalance

To stop a running rebalance, users can apply the `strimzi.io/rebalance=stop` annotation on the generated `KafkaRebalance` resource.
This will stop the running rebalance and the stopped `KafkaRebalance` resource will be deleted.
Users will need to ensure that they disable the `imbalance` mode of auto-rebalance after stopping the rebalance; otherwise, the rebalance will be triggered again since the anomalies are still present in the cluster.

### Maintenance Time Windows

Auto-rebalance on imbalance will respect maintenance time windows to control when rebalancing operations occur, using the existing `Kafka.spec.maintenanceTimeWindows` configuration that is already used for certificate renewals in Strimzi.

#### Implementation Design

The operator will reuse the **global maintenance time windows** defined at `Kafka.spec.maintenanceTimeWindows` for auto-rebalance on imbalance operations.
This provides a unified approach to scheduling maintenance activities (certificate renewals and cluster rebalancing) within designated time windows.

**Configuration Example:**

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
  maintenanceTimeWindows:
    - "* * 2-4 * * ?"     # 2:00-4:59 UTC daily for both cert renewals AND auto-rebalancing
  cruiseControl:
    # ...
    autoRebalance:
      - mode: imbalance
        template:
          name: my-imbalance-rebalance-template
```

**Key Features:**

- **Reuses existing infrastructure**: Leverages the same `Util.isMaintenanceTimeWindowsSatisfied()` utility method already used for certificate renewals
- **Unified maintenance scheduling**: Both certificate renewals and imbalance rebalancing occur during the same time windows
- **Standard cron format**: Uses the same cron expression format as certificate renewal windows (in UTC timezone)
- **Optional field**: If not specified, rebalancing triggers immediately when anomalies are detected

**Configuration Options:**

1. **With maintenance windows configured**:
   ```yaml
   spec:
     maintenanceTimeWindows:
       - "* * 2-4 * * ?"      # 2:00-4:59 AM UTC daily
       - "* * 14-16 * 5 ?"    # 2:00-4:59 PM UTC on Fridays
     cruiseControl:
       autoRebalance:
         - mode: imbalance
           template:
             name: my-template
   ```
   - ✅ Rebalancing and certificate renewals respect the same time windows
   - ✅ Simpler configuration - one set of windows for all maintenance activities
   - ✅ Consistent maintenance scheduling across all operator operations
   - Note: In case a rebalance is ongoing during the maintenance window and the maintenance window gets over, the rebalance will still continue unless its done.

2. **Without maintenance windows (immediate rebalancing)**:
   ```yaml
   spec:
     # maintenanceTimeWindows not specified
     cruiseControl:
       autoRebalance:
         - mode: imbalance
   ```
   - Rebalancing triggers immediately when anomalies are detected
   - No time-based restrictions
   - Suitable for development/test clusters or critical production clusters requiring immediate action

**Behavior:**

- **When maintenance windows are configured**:
  - Cruise Control continuously detects anomalies based on the `anomaly.detection.interval.ms` configuration (default: 5 minutes, as described in [Anomaly Detector Manager](#anomaly-detector-manager) section)
  - When an anomaly is detected outside a maintenance window, Cruise Control raises the goal violation event periodically (every `anomaly.detection.interval.ms`), but the operator does NOT trigger a rebalance
  - During each reconciliation that falls within a maintenance window, the operator queries Cruise Control's `state` endpoint
  - If the anomaly still exists (based on `detectionDate` comparison against `lastRebalanceCompletionTime`), the rebalance is triggered
  - The anomaly detection by Cruise Control continues periodically regardless of maintenance windows, but rebalance triggering by the operator is gated by the maintenance window check using `Util.isMaintenanceTimeWindowsSatisfied(reconciliation, maintenanceWindows, clock.instant())`
  - **Key point**: Cruise Control will keep raising the same goal violation event periodically (with updated `detectionDate` and new `anomalyId`) until the issue is fixed. The auto-rebalance will only execute once the maintenance window is satisfied and Cruise Control raises the event during that window.

- **When no maintenance windows are configured**:
  - Rebalance is triggered immediately when a fixable goal violation is detected by Cruise Control
  - The operator still relies on Cruise Control's periodic anomaly detection (every `anomaly.detection.interval.ms`)

**Emergency Override:**

If users need immediate rebalancing despite configured maintenance windows (e.g., critical disk capacity issue), they can create a manual `KafkaRebalance` resource (which bypasses maintenance windows)

#### Interaction with Scale Operations

Scale-up and scale-down auto-rebalance operations will continue to **ignore** maintenance windows (executed immediately), while imbalance auto-rebalance respects them.

### Handling Intra-Broker Disk Imbalances (JBOD Storage)

For Kafka clusters using **JBOD storage with multiple disks**, the `imbalance` mode automatically handles both **inter-broker** (replicas distributed across brokers) and **intra-broker disk imbalances** (disk usage between disks on the same broker).

**Storage Detection and Goal Configuration:**

The operator detects JBOD storage on startup and automatically extends Cruise Control's anomaly detection goals:

- **Non-JBOD storage**: Only inter-broker goals configured (default behavior)
- **JBOD storage**: Automatically adds `IntraBrokerDiskCapacityGoal` and `IntraBrokerDiskUsageDistributionGoal` to anomaly detection

This is **transparent to users** - no additional configuration needed.

**Mutual Exclusivity Constraint:**

Inter-broker and intra-broker rebalancing **cannot run simultaneously** (Cruise Control enforces this).
The operator handles violations through separate anomaly detection cycles rather than sequential phases within a single cycle.

**Priority order:**
1. Scale operations (highest)
2. Inter-broker rebalancing 
3. Intra-broker disk rebalancing (lowest)

**Execution Flow:**

When anomalies are detected, the operator categorizes violations by examining the `fixableViolatedGoals` list from Cruise Control's state endpoint:

- **Inter-broker goals**: Any goal except `IntraBrokerDiskCapacityGoal` and `IntraBrokerDiskUsageDistributionGoal`
- **Intra-broker goals**: `IntraBrokerDiskCapacityGoal` or `IntraBrokerDiskUsageDistributionGoal`

Based on this categorization:

1. **Both types detected**: 
   - Create `KafkaRebalance` with `rebalanceDisk=false` for inter-broker violations only
   - After completion: Update ConfigMap with completion timestamp and transition to `Idle`
   - Wait for next anomaly detection cycle (default: 5 minutes, configurable via `anomaly.detection.interval.ms`)
   - If intra-broker violations still persist, Cruise Control detects them as a new anomaly
   - New anomaly timestamp > last completion timestamp → triggers new rebalance with `rebalanceDisk=true`
   - Inter-broker rebalancing moves partitions between brokers, which redistributes data across disks and may naturally resolve intra-broker imbalances without requiring a separate disk rebalance. Waiting for the next detection cycle avoids unnecessary intra-broker rebalancing.

2. **Only intra-broker violations**:
   - Create `KafkaRebalance` with `rebalanceDisk=true`
   - Goals automatically set by Cruise Control (template goals ignored)

3. **Only inter-broker violations**:
   - Create `KafkaRebalance` with `rebalanceDisk=false` (or field omitted, defaults to `false`)
   - Uses goals from template if configured, otherwise defaults

**State Machine Handling:**

Each rebalance type is handled in its own anomaly detection cycle:

- When both violations are detected, only inter-broker rebalancing is triggered
- After inter-broker completes: transitions to `Idle` and updates ConfigMap
- If intra-broker violations persist: Cruise Control detects them in the next cycle as a fresh anomaly
- The fresh anomaly triggers a new `RebalanceOnImbalance` transition with `rebalanceDisk=true`
- Each rebalance completes independently without queuing the next phase

**Generated `KafkaRebalance` Resources:**

Inter-broker:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-imbalance
spec:
  mode: full
  rebalanceDisk: false
  goals: [from template or defaults]
```

Intra-broker:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-imbalance
spec:
  mode: full
  rebalanceDisk: true
  concurrentIntraBrokerPartitionMovements: 2
  # No goals - automatically set by Cruise Control
```

**Template Handling:**
- Inter-broker: Uses goals from template
- Intra-broker: Ignores goals from template (auto-set), but uses `concurrentIntraBrokerPartitionMovements` if specified

**Operator Crash Recovery:**

When the operator crashes and restarts during `RebalanceOnImbalance`:

1. Read the FSM state from `Kafka.status.autoRebalance.state` (finds `RebalanceOnImbalance`)
2. Check if the `KafkaRebalance` resource `<cluster-name>-auto-rebalancing-imbalance` exists:
   - **If exists**: Continue monitoring it from its current state (`ProposalReady`, `Rebalancing`, `Ready`, etc.)
     - The `rebalanceDisk` field in the `KafkaRebalance` spec indicates whether this is inter-broker (`false`) or intra-broker (`true`) rebalancing
     - Process the rebalance normally based on its current state
   - **If missing**: The rebalance was either completed, failed, stopped, or manually deleted during the crash
     - Transition auto-rebalance state to `Idle`
     - Trust Cruise Control's anomaly detection to re-detect any remaining imbalances (runs every 5 minutes by default)
     - Next reconciliation will query Cruise Control and create a new `KafkaRebalance` if violations still exist

**Why This Works:**

This simplified recovery approach works because Cruise Control's anomaly detection acts as the persistent source of truth about cluster state.
The operator doesn't need to remember "what phase were we in" because Cruise Control will tell us what violations currently exist.

For JBOD clusters where inter-broker rebalancing may have left intra-broker violations:
- **Crash during inter-broker rebalance**: `KafkaRebalance` exists with `rebalanceDisk=false` → continue monitoring it from its current state
- **Crash after inter-broker completion**: 
  - Inter-broker `KafkaRebalance` was already deleted (transitioned to `Ready` and cleaned up)
  - Auto-rebalance state is `Idle`
  - If intra-broker violations persist: Cruise Control detects them in the next anomaly detection cycle
  - New anomaly with timestamp > ConfigMap's `lastRebalanceCompletionTime` → triggers new rebalance with `rebalanceDisk=true`
- **Crash during intra-broker rebalance**: `KafkaRebalance` exists with `rebalanceDisk=true` → continue monitoring it from its current state

### Metrics for tracking the rebalance requests

If users want to track when auto-rebalances happen, they can access the Strimzi [metrics](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/grafana-dashboards/strimzi-operators.json#L712) for the `KafkaRebalance` custom resources.
This includes tracking when they were visible/created.
These metrics also cover the `KafkaRebalance` resources which were created automatically, so users can utilize them to understand when an auto-rebalance was triggered in their cluster.
We will add labels to the metrics to differentiate the created rebalances based on mode (i.e., whether the rebalance was triggered for `full` mode for imbalanced clusters, `add-broker` mode, or `remove-broker` mode).
Cruise Control currently does not have any metrics to depict what type of anomaly was detected and when.
Therefore, we will add a new counter metric named `strimzi_auto_rebalance_anomalies_detected_total` which will be incremented whenever an anomaly is detected by the anomaly detector.
We will also add labels to differentiate anomalies based on fixability (`fixable`, `unfixable`, `mixed`) and type of anomaly (`goal_violation`, `broker_failure`, `disk_failure`, etc.).
These metrics will be exposed by the operator and will only be available when the user has configured the `imbalance` mode of auto-rebalance.

### Detecting Persisting Anomalies with Metrics and Alerts

An anomaly may persist after a rebalance if cluster constraints prevent full optimization, new load patterns emerge immediately, or the rebalance only partially addresses violations.
When this happens, Cruise Control keeps detecting the same anomaly entry with a different anomaly name and different detection time.
To monitor the rebalance loops and cluster instability, the users can make use of the metrics exposed by the operator.
Users can monitor the exposed metrics and configure alerts to detect potential issues.
Example alert conditions:

- **Frequent rebalances**: More than 3 auto-rebalances triggered within 1 hour may indicate a persisting anomaly
- **Repeated goal violations**: Same goal violation detected multiple times in a short time window (e.g., >2 times in 30 minutes)
- **Rebalance failures**: Unfixable goals detected or repeated NotReady states requiring manual intervention

When such patterns are detected, check Kafka CR status for warnings, review `KafkaRebalance` resources, query Cruise Control's state endpoint for `balancednessScore`, and verify cluster capacity matches anomaly detection goals.

**Shipped Alert Examples:**

The Strimzi project will not include pre-configured alerts for these metrics in the default shipped alerting rules.
This decision is intentional because:
- Alert thresholds are highly environment-specific (busy production clusters vs. development clusters have different normal behavior)
- Different organizations have different tolerances for rebalancing frequency
- Users should tune alerts based on their specific cluster characteristics and operational requirements

However, the Strimzi documentation will include **example alert configurations** for common scenarios:
- Example Prometheus alert for detecting frequent auto-rebalances
- Example alert for unfixable goal violations that persist beyond a threshold
- Example alert for rebalance failures

Users can copy and adapt these examples to their monitoring systems and customize thresholds to match their operational needs.

### Auto-rebalancing execution for `imbalance` mode

#### Auto-rebalancing Finite State Machine (FSM) for `imbalance` mode

Currently, the auto-rebalancing mechanism runs through a Finite State Machine (FSM) made by the following states:

* **Idle**: Initial state with a new auto-rebalancing initiated when scaling down/up operations were requested. This is also the ending state after an auto-rebalancing completed successfully or failed.
* **RebalanceOnScaleDown**: a rebalancing related to a scale down operation is running.
* **RebalanceOnScaleUp**: a rebalancing related to a scale up operation is running.

With the new `imbalance` mode, we will be introducing a new state to the FSM:
* **RebalanceOnImbalance**: Rebalancing triggered by goal violations is running. For JBOD clusters with both inter-broker and intra-broker violations, only the inter-broker rebalance is executed in this state. After completion, the operator transitions to `Idle` and waits for the next anomaly detection cycle to determine if intra-broker violations persist. This approach avoids unnecessary intra-broker rebalancing when inter-broker movements have already resolved disk imbalances.

With the new `imbalance` mode, the FSM state transitions will look like this:

![auto-rebalance state machine flow diagram](images/136-finite-state-machine.png)

* from **Idle** to:
  * **RebalanceOnScaleDown**: if a scale down operation was requested. This transition happens even if a scale up was requested at the same time but the rebalancing on scaling down has the precedence. The rebalancing on scale up is queued. They will run sequentially.
  * **RebalanceOnScaleUp**: if only a scale up operation was requested. There was no scale down operation requested.
  * **RebalanceOnImbalance**: if goal violations are detected (inter-broker, intra-broker, or both), and the `fixableViolatedGoals` list is not empty while the `unfixableViolatedGoals` list is empty. If there are queued scale up or scale down operations, then they will run first. For JBOD clusters with both violation types, only inter-broker rebalancing is triggered in this transition; intra-broker violations are addressed in a subsequent detection cycle if they persist.

* from **RebalanceOnScaleDown** to:
  * **RebalanceOnScaleDown**: if a rebalancing on scale down is still running.
  * **RebalanceOnScaleUp**: if a scale down operation was requested together with a scale up and, because they run sequentially, the rebalance on scale down had the precedence, was executed first and completed successfully. We can now move on with rebalancing for the scale up.
  * **Idle**: if a scale down operation was requested, it was executed and completed successfully/failed or a full rebalance was asked due to an anomaly but since the scale-down rebalance is done, we can ignore the anomalies assuming they are fixed by the rebalance. In case, they are not fixed, Cruise Control will detect them again and a new rebalance would be requested.

* from **RebalanceOnScaleUp**:
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is still running.
  * **RebalanceOnScaleDown**: if a scale down operation was requested, so the current rebalancing scale up is stopped (and queued) and a new rebalancing scale down is started. The rebalancing scale up will be postponed.
  * **Idle**: if a scale up operation was requested, it was executed and completed successfully/failed or a full rebalance was asked due to an anomaly but since the scale-up rebalance is done, we can ignore the anomalies assuming they are fixed by the rebalance. In case, they are not fixed, Cruise Control will detect them again and a new rebalance would be requested.

* from **RebalanceOnImbalance**:
  * **RebalanceOnImbalance**: if a rebalancing on imbalance is still running.
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is in queue, then the rebalance on imbalance will be stopped and the scale up will happen first. If there is a rebalancing scale down in queue too, then it will be executed before both scale up and rebalance on imbalance.
  * **RebalanceOnScaleDown**: if a scale down operation was requested, then the rebalance on imbalance will be stopped and the scale down will be allowed to finish first and after that rebalance on imbalance will be executed.
  * **Idle**: if rebalancing was requested, it was executed and completed successfully or failed. The ConfigMap is updated with the completion timestamp. For JBOD clusters where intra-broker violations persist, Cruise Control will detect them in the next anomaly detection cycle and trigger a new rebalance.

On each reconciliation, the following process will be used:

1. The operator creates the cluster model instance
2. The auto-rebalance reconciliation logic checks for ongoing rebalances (see "Detecting Active Rebalances" section)
3. If the state is `Idle` and no rebalance is running, query the `state` endpoint to get detected violations
4. Apply the comparison logic from "Logic to trigger rebalance" section to determine if a new rebalance should be triggered
5. For JBOD clusters, categorize violations and prioritize inter-broker over intra-broker as described in "Handling Intra-Broker Disk Imbalances" section. Each violation type is handled in its own anomaly detection cycle.

The auto-rebalance reconciliation logic also loads the `Kafka.status.autoRebalance` content:

* `state`: is the FSM state
* `lastTransitionTime`: when the transition to that state happened.
* `modes`: enabled modes (`remove-brokers`, `imbalance` and/or `add-brokers`) if an auto-rebalancing requested.

The FSM is initialized based on the `state` field.

Let's see what happens during the auto-rebalancing process when the FSM starts from the **Idle** state and transitions to **RebalanceOnImbalance**.

##### Idle

This state is set initially when a `Kafka` custom resource is created with the `spec.cruiseControl.autoRebalance` field configured.
It is also the end state of a previously completed or failed auto-rebalancing operation.
In case of successful completion, once the rebalance moves to the `Ready` state, we will delete the generated `KafkaRebalance` and update the `auto-rebalance` state to `Idle`.
In case of failed auto-rebalancing, once the rebalance moves to the `NotReady` state, we will propagate the error to the Kafka CR, delete the generated `KafkaRebalance`, and set the state to `Idle`.
In this state, the operator removes the finalizer and deletes the corresponding generated `KafkaRebalance` custom resource.

##### RebalanceOnImbalance

In this state, goal violation anomalies were detected when we sent a request to the `state` endpoint with `anomaly_detector` substate.
A `KafkaRebalance` resource will now be applied to the cluster to fix the cluster imbalance.
This `KafkaRebalance` will be based on the template provided by the user; if no template is provided, then the `KafkaRebalance` will be created with default configurations.

For JBOD clusters with both inter-broker and intra-broker violations, only the inter-broker rebalance is triggered in this cycle. After completion, the operator waits for the next anomaly detection cycle to determine if intra-broker violations persist (see "Handling Intra-Broker Disk Imbalances" section for details)

```mermaid
flowchart TB
  A[KafkaRebalanceState] --> B{if state}
  B --> C[Ready]
  C -- queued scale Down --> D[RebalanceOnScaleDown]
  C -- queued scale up --> E[RebalanceOnScaleUp]
  C -- queued rebalance --> F[RebalanceOnImbalance]
  B --> G[PendingProposal, ProposalReady or Rebalancing]
  G --> H{{rebalancing is running}}
  B --> I[NotReady]
  I --> J{{Transition to Idle state and delete rebalance resource}}
  B --> K[Stopped]
  K --> L{{delete rebalance and disable `imbalance` mode}}
```

Checking the current `KafkaRebalance` status:

* if `Ready`, the rebalance was successful.
  * if there is a queued rebalancing scale down (`Kafka.status.autoRebalance.modes[remove-brokers]` exists), start the rebalancing scale down and transition to **RebalanceOnScaleDown**.
  * if there is a queued rebalancing scale up (`Kafka.status.autoRebalance.modes[add-brokers]` exists), start the rebalancing scale up and transition to **RebalanceOnScaleUp**.
  * If no queued operations, transition to **Idle**, update ConfigMap with completion timestamp, clean `Kafka.status.autoRebalance.modes`, delete the generated `KafkaRebalance` custom resource.
  * **For JBOD clusters**: If intra-broker violations persist after this rebalance, Cruise Control will detect them in the next anomaly detection cycle and trigger a new rebalance.
* if `PendingProposal`, `ProposalReady` or `Rebalancing`, the rebalancing is still running.
  * No further actions required.
* if `NotReady`
  * The rebalancing failed. Transition to **Idle**, propagate the error to the `Kafka` CR status, and delete the generated `KafkaRebalance` custom resource.
  * The next anomaly detection cycle will detect if the issue persists and trigger a fresh rebalance.
* if `Stopped`
  * the rebalancing is stopped, transition to **Idle** and also removing the corresponding mode from the status. The operator deletes the generated `KafkaRebalance` custom resource. The users should then disable the `imbalance` mode of auto-rebalance else a new rebalance would be triggered.

If, during an ongoing auto-rebalancing, the `KafkaRebalance` custom resource is not there anymore on the next reconciliation, it could mean the user deleted it while the operator was stopped/crashed/not running.
In this case, the operator will transition the auto-rebalance state to `Idle`.
Cruise Control will re-detect any persisting goal violations in the next anomaly detection cycle, and a fresh rebalance will be triggered if needed.

### `KafkaRebalance` Resource Lifecycle for Imbalance Mode

The lifecycle of auto-generated `KafkaRebalance` resources for the `imbalance` mode follows a deterministic pattern:

**Creation:**
A `KafkaRebalance` resource is created when:
- The auto-rebalance state machine is in `Idle` state
- No active `KafkaRebalance` resources exist (checked via `KafkaRebalanceUtils.rebalanceState()`)
- A goal violation is detected with the `detectionDate` timestamp greater than the `lastRebalanceCompletionTime` from the `<cluster-name>-rebalancing-tracker` ConfigMap (or the ConfigMap doesn't exist / field is null, meaning no rebalance has ever completed)
- The `fixableViolatedGoals` list is non-empty and `unfixableViolatedGoals` list is empty

**State Progression:**
Once created, the `KafkaRebalance` resource progresses through these states:
1. `New` - Initial state after creation
2. `PendingProposal` - Waiting for Cruise Control to generate optimization proposal
3. `ProposalReady` - Optimization proposal is ready
4. `Rebalancing` - Partition reassignment is in progress
5. Terminal states: `Ready` (success), `NotReady` (failure), or `Stopped` (manually stopped)

**Auto-Approval:**
Unlike manual `KafkaRebalance` resources, auto-generated resources for imbalance mode are automatically approved.
The operator applies the rebalance without waiting for user approval when the proposal becomes ready.

**Deletion:**
The operator deletes the generated `KafkaRebalance` resource when:
- The rebalance completes successfully (`Ready` state) and the auto-rebalance state transitions to `Idle`
- The rebalance fails (`NotReady` state), the error is propagated to Kafka CR status, and the state transitions to `Idle`
- The rebalance is manually stopped (`Stopped` state) and the state transitions to `Idle`

The operator uses cascading deletion to ensure both the `KafkaRebalance` resource and its associated ConfigMap (storing rebalancing progress) are removed together.

**Finalizer Protection:**
The operator sets a `strimzi.io/auto-rebalancing` finalizer on generated resources to prevent premature deletion.
This finalizer is removed only when the operator is ready to delete the resource, ensuring that:
- Users or other tooling cannot accidentally delete the resource while rebalancing is active
- The resource cleanup is coordinated with the state machine transitions

## Testing Strategy

The testing approach will follow existing patterns used in `KafkaAutoRebalancingMockTest` for scale-up and scale-down modes.
Tests will use `MockCruiseControl` to simulate Cruise Control's `state` endpoint responses with anomaly detection data, and `MockKube3` to mock the Kubernetes API.

### Implementation Approach

- Extend `MockCruiseControl` to support mocking the `state` endpoint with `anomaly_detector` substate
- Use JSON response files to simulate different anomaly detection scenarios (fixable goals, unfixable goals, varying timestamps)
- Use `MockKube3` to mock Kubernetes resources and verify `KafkaRebalance` creation, updates, and deletion
- Simulate `KafkaRebalance` state progression by manually patching resource states (similar to `KafkaAutoRebalancingMockTest`)
- Validate state machine transitions through multiple reconciliation cycles
- Leverage `KafkaRebalanceUtils.rebalanceState()` to check for active rebalances (matches production code)

### Key Test Scenarios

1. **Anomaly Detection and Successful Rebalance**
   - Ensure no existing `KafkaRebalance` resources are in active states
   - Mock `state?substates=anomaly_detector` returning goal violation with `fixableViolatedGoals` and detection timestamp
   - Verify operator creates `<cluster>-auto-rebalancing-imbalance` `KafkaRebalance` resource
   - Simulate `KafkaRebalance` progressing to `Ready` state
   - Verify state transitions: `Idle` → `RebalanceOnImbalance` → `Idle`
   - Verify `KafkaRebalance` resource is deleted after completion

2. **Concurrent Manual Rebalance Running**
   - Create a manual `KafkaRebalance` resource in `Rebalancing` state
   - Mock anomaly detector returning goal violations
   - Verify operator does NOT create new `KafkaRebalance` (detects active rebalance via `KafkaRebalance`Utils)

3. **Auto-Rebalance Already Running**
   - Set Kafka CR's auto-rebalance status to `RebalanceOnScaleUp` or `RebalanceOnScaleDown`
   - Mock anomaly detector returning goal violations
   - Verify operator ignores detected anomalies until scale operations complete

4. **Scale Operations Take Priority**
   - Start with state `RebalanceOnImbalance` and active `KafkaRebalance`
   - Trigger scale-down by reducing broker replicas
   - Verify state transitions to `RebalanceOnScaleDown` (imbalance rebalance interrupted)

5. **Unfixable Goal Violations**
   - Mock anomaly response with non-empty `unfixableViolatedGoals` list
   - Verify operator does NOT create `KafkaRebalance` resource
   - Verify Warning condition is added to Kafka CR status listing unfixable goals

6. **Rebalance Failure Recovery**
   - Start rebalance, then simulate `KafkaRebalance` transitioning to `NotReady`
   - Verify operator transitions to `Idle` and deletes failed resource
   - In next reconciliation, mock new goal violation with later timestamp
   - Verify fresh rebalance attempt is triggered

7. **Stopped Rebalance**
   - Apply `strimzi.io/rebalance=stop` annotation on generated `KafkaRebalance`
   - Simulate transition to `Stopped` state
   - Verify `<cluster-name>-rebalancing-tracker` ConfigMap's `lastRebalanceCompletionTime` is updated
   - Verify operator transitions to `Idle` and deletes resource

### System Tests

The following system tests will be added to `CruiseControlST.java` following the pattern of `testAutoKafkaRebalanceScaleUpScaleDown()`:

#### 1. **Basic Auto-Rebalance on Imbalance** 

1. Deploy Kafka cluster with JBOD storage and auto-rebalance `imbalance` mode enabled
2. Configure fast anomaly detection (`anomaly.detection.interval.ms: 120000`)
3. Create KafkaRebalance template resource with `strimzi.io/rebalance-template: "true"` annotation
4. Create cluster imbalance (uneven partition distribution using AdminClient)
5. Wait for Cruise Control anomaly detection (up to 3 minutes)
6. Verify `<cluster>-auto-rebalancing-imbalance` KafkaRebalance resource is created
7. Verify state transitions: `Idle` → `RebalanceOnImbalance` → `Idle`
8. Verify KafkaRebalance resource is deleted after completion
9. Verify `<cluster>-auto-rebalance-imbalance-tracker` ConfigMap contains completion timestamp
10. Wait for next anomaly detection cycle (2+ minutes)
11. Verify no new rebalance is triggered (anomaly is fixed)

#### 2. **Maintenance Window Integration** 

Tests that auto-rebalance respects maintenance time windows.

**Steps:**
1. Deploy Kafka cluster with auto-rebalance `imbalance` mode and maintenance window set to future time
2. Create cluster imbalance
3. Wait for anomaly detection
4. Verify auto-rebalance does NOT trigger (outside maintenance window)
5. Verify state remains `Idle`
6. Verify no KafkaRebalance resource is created
7. Update maintenance window to current time
8. Verify auto-rebalance triggers within detection interval
9. Verify state transitions to `RebalanceOnImbalance`
10. Verify KafkaRebalance resource is created and completes
11. Verify state returns to `Idle`

#### 3. **Scale Operations Interrupt Imbalance Rebalancing**

Tests that scale operations take priority over imbalance rebalancing.

**Steps:**
1. Deploy Kafka cluster with auto-rebalance for all modes enabled
2. Trigger imbalance rebalance by creating uneven partition distribution
3. Wait for state `RebalanceOnImbalance` with active KafkaRebalance
4. Trigger scale-down by reducing broker replicas in KafkaNodePool
5. Verify imbalance KafkaRebalance is stopped (annotation `strimzi.io/rebalance=stop`)
6. Verify imbalance KafkaRebalance is deleted
7. Verify state transitions from `RebalanceOnImbalance` to `RebalanceOnScaleDown`
8. Verify scale-down KafkaRebalance resource `<cluster>-auto-rebalancing-remove-brokers` is created
9. Wait for scale-down rebalance completion
10. Verify state transitions to `Idle`

#### 4. **Rebalance Failure Handling**

Tests operator behavior when auto-triggered rebalance fails.

**Steps:**
1. Deploy Kafka cluster with auto-rebalance `imbalance` mode
2. Create cluster imbalance
3. Wait for auto-rebalance trigger and KafkaRebalance creation
4. Simulate rebalance failure by deleting Cruise Control pod while rebalancing
5. Wait for KafkaRebalance to transition to `NotReady` state
6. Verify operator logs warning about failure
7. Verify state transitions back to `Idle`
8. Verify failed KafkaRebalance resource is deleted
9. Verify ConfigMap `lastRebalanceCompletionTime` is updated with failure timestamp
10. Restart Cruise Control and wait for recovery
11. Wait for next anomaly detection cycle
12. Verify new rebalance attempt is triggered (detectionDate > failure timestamp)

#### 5. **JBOD Intra-Broker Rebalancing** - `testAutoRebalanceOnImbalanceJBODIntraBroker()`

Tests automatic intra-broker disk rebalancing for JBOD storage.

**Steps:**
1. Deploy Kafka cluster with JBOD storage (2+ volumes per broker)
2. Configure `anomaly.detection.goals` to include `IntraBrokerDiskCapacityGoal` and `IntraBrokerDiskUsageDistributionGoal`
3. Create uneven disk usage across volumes on same broker
4. Wait for Cruise Control to detect intra-broker violation
5. Verify inter-broker rebalance is triggered first (if inter-broker violations exist)
6. Wait for inter-broker rebalance completion
7. Verify intra-broker rebalance is triggered in next detection cycle
8. Verify generated KafkaRebalance has `rebalanceDisk: true`
9. Wait for intra-broker rebalance completion
10. Verify ConfigMap is updated
11. Verify state returns to `Idle`

#### 6. **Crash Recovery from RebalanceOnImbalance State** - `testCrashRecoveryDuringImbalanceRebalance()`

Tests operator crash recovery when in `RebalanceOnImbalance` state.

**Steps:**
1. Deploy Kafka cluster with auto-rebalance `imbalance` mode
2. Create imbalance and trigger auto-rebalance
3. Wait for state `RebalanceOnImbalance` and KafkaRebalance creation
4. Simulate operator crash by deleting operator pod
5. Wait for operator to restart
6. **If KafkaRebalance still exists:** Verify operator continues monitoring from current state
7. **If KafkaRebalance was deleted during crash:** Verify operator transitions to `Idle` and trusts Cruise Control to re-detect violations
8. Verify eventual completion or re-trigger based on cluster state
9. Verify ConfigMap is properly updated

## Affected/not affected projects

This change will affect the Strimzi cluster operator.

### Required API Changes

The following changes need to be made to support this feature:

**Add `imbalance` mode**: Support automatic rebalancing on goal violations (handles both inter-broker and intra-broker for JBOD) in the `Kafka.spec.cruiseControl.autoRebalance` configuration.

These API changes are required for the auto-rebalance on imbalance feature to function correctly.

## Backwards compatibility

This change will not cause any backwards compatibility issue.
The behavior stated in the proposal is only enabled if the user is making use of the `imbalance` mode of the auto-rebalancing feature.

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
The idea was to create a Kubernetes custom resource named `KafkaAnomaly` every time an anomaly was detected and both the operator and the Notifier would watch the resource for updates.
But with this approach the operator will be responsible to make decision regarding the anomaly should be fixed or not

### Using Cruise Control self-healing ability

If we were to enable the self-healing ability of Cruise Control then, in response to detected anomalies, Cruise Control would issue partition reassignments without involving the Strimzi Cluster Operator.
This could cause potential conflicts with other administration operations and is the primary reason self-healing has been disabled until now.

Pros:
* Allows the operator to ignore the anomalies if some task is already running in the cluster like rolling, rebalance etc.

Cons:
* Very tight coupling with the operator.
* Delaying the anomaly detector progress.

### Using Kubernetes Events but Cruise Control controls auto-rebalance

This alternative is to let Cruise Control handle the self-healing.
Whenever an anomaly is detected by Cruise Control, our notifier will generate an event to alert the operator that an anomaly was detected in the cluster
But the fix would be run by Cruise Control itself and not the operator

Pros:
* Loose coupling with the operator
* Faster decision-making as Cruise Control runs the fix

Cons:
* Operator wouldn't play any role in the process.

## Future Scope

In the future, we plan to introduce auto-rebalance for topic-related and metric-related imbalances.
Topic-related issues will require coordination with the Topic Operator, while metric-related issues will require coordination with the Kafka Admin API.
We can also enhance the operator to automatically proceed with rebalancing fixable goals even when unfixable hard goals are present, by adding the `skipHardGoalCheck` field in the template:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaRebalance
metadata:
  name: my-rebalance-template
  annotations:
    strimzi.io/rebalance-template: "true"
spec:
  goals:
    - ReplicaCapacityGoal
    - DiskCapacityGoal
    - DiskUsageDistributionGoal
    - RackAwareGoal  # May become unfixable
  skipHardGoalCheck: true  # Allow rebalance even if hard goals (like RackAwareGoal) are violated
```

Additionally, exponential backoff could be implemented for repeated rebalance failures by tracking failure history per goal type in the ConfigMap (consecutive failures, last failure time, next retry time), reducing load from persistent unfixable issues while still eventually retrying once resolved.

As this feature evolves, we can also explore ways to automatically fix issues like disk failures and broker failures, since the fix would be driven by the operator rather than Cruise Control directly.