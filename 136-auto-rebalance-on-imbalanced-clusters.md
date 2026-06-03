# Auto-rebalance on imbalanced clusters

This proposal introduces support for automatic partition rebalancing whenever a Kafka cluster becomes imbalanced due to unevenly distributed replicas, overloaded brokers, or similar issues.

This feature is opt-in.
When enabled, it allows the Strimzi Operator to automatically execute partition rebalances through `KafkaRebalance` custom resources whenever cluster imbalances are detected by Cruise Control.

## Motivation

Currently, if the cluster is imbalanced, the user would need to manually rebalance the cluster by using the `KafkaRebalance` custom resource.
With Strimzi, if you want to rebalance due to cluster imbalance, the only available approach today is manual intervention - there is no automated mechanism for rebalancing on imbalance.
Automating this process would reduce operational overhead and benefit all users by eliminating the need for constant manual monitoring and rebalancing.

### Introduction to Self Healing in Cruise Control

In order to understand how we plan to automatically fix unbalanced Kafka clusters, the sections below will go over how Cruise Control's anomaly detection and self-healing features work to detect the anomalies in the cluster and fix them.

![self-healing flow diagram](images/136-self-healing-flow.png)

The above flow diagram depicts the self-healing process in Cruise Control.
The anomaly detector manager detects an anomaly (using the detector classes) and forwards it to the notifier.
The notifier then decides what action to take on the anomaly whether to fix it, ignore it, or delay it.
Cruise Control provides various notifiers to alert the users about the detected anomaly in several ways like Slack, Alerta, MS Teams etc.

#### Anomaly Detector Manager

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

#### Notifiers in Cruise Control

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

#### Self Healing

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

### Proposal

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

### Maintenance Time Windows

Auto-rebalance on imbalance supports maintenance time windows to allow users to control when rebalancing operations occur.

#### Implementation Design

Users can configure **dedicated maintenance time windows** specifically for auto-rebalance on imbalance. These windows are independent of the global maintenance windows used for certificate renewals.

**Recommended Configuration - Custom Maintenance Windows for Rebalancing:**

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
  maintenanceTimeWindows:
    - "* * 2-4 * * ?"     # Global: 2:00-4:59 UTC daily (for certificate renewals)
  cruiseControl:
    # ...
    autoRebalance:
      - mode: imbalance
        maintenanceTimeWindows:
          - "* * 8-10 * * ?"    # Custom window #1: 8:00-10:59 UTC daily
          - "* * 14-16 * * ?"   # Custom window #2: 14:00-16:59 UTC daily
          - "* * 20-22 * 5 ?"   # Custom window #3: 20:00-22:59 UTC on Fridays only
        template:
          name: my-imbalance-rebalance-template
```

**Key Features:**

- **Fully customizable**: Users define their own cron expressions for when rebalancing can occur
- **Multiple windows**: Support for multiple time windows per day/week (as shown above)
- **Independent from global windows**: Rebalancing windows are separate from certificate renewal windows
- **Standard cron format**: Uses the same cron expression format as `Kafka.spec.maintenanceTimeWindows` (in UTC timezone)
- **Optional field**: If not specified, different behavior applies (see options below)

**Configuration Options:**

1. **Custom maintenance windows for imbalance** (RECOMMENDED):
   ```yaml
   cruiseControl:
     autoRebalance:
       - mode: imbalance
         maintenanceTimeWindows:           # User-defined windows
           - "* * 8-10 * * ?"
           - "* * 14-16 * * ?"
   ```
   - ✅ Uses ONLY the windows defined here (ignores global windows)
   - ✅ Full user control over rebalancing schedule
   - ✅ Can be different from certificate renewal schedule
   - ✅ Best for production clusters with specific operational requirements

2. **Fallback to global windows**:
   ```yaml
   cruiseControl:
     autoRebalance:
       - mode: imbalance
         # maintenanceTimeWindows field not present
   # But Kafka.spec.maintenanceTimeWindows exists
   ```
   - Uses the global `Kafka.spec.maintenanceTimeWindows`
   - Simpler configuration when rebalancing and certificate renewal can share the same schedule
   - Less flexible but easier to manage

3. **No maintenance windows (immediate rebalancing)**:
   ```yaml
   cruiseControl:
     autoRebalance:
       - mode: imbalance
         # maintenanceTimeWindows field not present
   # AND Kafka.spec.maintenanceTimeWindows also not present
   ```
   - Rebalancing triggers immediately when anomalies are detected
   - No time-based restrictions
   - Suitable for development/test clusters or critical production clusters requiring immediate action

**Behavior:**

- **When maintenance windows are configured** (option 1 or 2):
  - When an anomaly is detected outside a maintenance window, the operator records the detection but does NOT trigger a rebalance
  - During the next maintenance window, the operator queries Cruise Control's `state` endpoint
  - If the anomaly still exists (based on `detectionDate` comparison), the rebalance is triggered
  - The anomaly detection continues every reconciliation, but rebalance triggering is gated by the maintenance window
  
- **When no maintenance windows are configured** (option 3):
  - Rebalance is triggered immediately when a fixable goal violation is detected
  - Useful for critical clusters where imbalance correction cannot wait

**Emergency Override:**

If users need immediate rebalancing despite configured maintenance windows (e.g., critical disk capacity issue), they can:
1. Temporarily remove or comment out the `maintenanceTimeWindows` field from the auto-rebalance configuration, or
2. Create a manual `KafkaRebalance` resource (which bypasses maintenance windows)

#### Interaction with Scale Operations

Scale-up and scale-down auto-rebalance operations will continue to **ignore** maintenance windows (executed immediately), while imbalance auto-rebalance respects them.

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
The operator will use this endpoint to understand if there are any goal violations so that the operator can trigger a rebalance (see section "KafkaAutoRebalanceReconciler using state endpoint" below).
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
If the user has not configured the anomaly detection goals in Cruise Control section of the Kafka CR then the operator will set the default goals to be used by the anomaly detector. 
The default anomaly detection goals set by the operator are `RACK_AWARENESS_GOAL`, `MIN_TOPIC_LEADERS_PER_BROKER_GOAL`, `REPLICA_CAPACITY_GOAL`, `DISK_CAPACITY_GOAL`. 
These are similar to the default goals used for `KafkaRebalance` if the users don't mention the rebalance goals.
**Template Goal Validation:**

If the user specifies a rebalance template, the operator validates that the goals in the template are a subset of the anomaly detection goals configured for Cruise Control.
This ensures that the resulting rebalance proposal will actually try to address the root cause of the anomaly that was detected. 
If the anomaly is due to a disk distribution goal violation but your rebalance optimization proposal does not use the DiskDistributionGoal, then it is wrong.

**When validation occurs:**
- During every `Kafka` CR reconciliation cycle, as part of the `KafkaAutoRebalanceReconciler.reconcile()` method
- The validation happens **after** Cruise Control is deployed but **before** any anomaly detection or rebalancing logic executes
- The operator first checks if the `imbalance` mode is configured in `spec.cruiseControl.autoRebalance`
- If `imbalance` mode is enabled and a template is specified, the operator validates that:
  - All goals in the template exist in the `anomaly.detection.goals` configuration (or defaults if not configured)
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
The `imbalance` mode will be mapped to the `full` mode in the generated KafkaRebalance resource which means that the generated `KafkaRebalance` custom resource will have the mode set as `full` which within the Strimzi rebalancing operator means calling the Cruise Control API to run a rebalancing taking all brokers into account.

The generated `KafkaRebalance` custom resource will be called `<my-cluster-name>-auto-rebalancing-goal-violation`, where the `<my-cluster-name>` part comes from the `metadata.name` in the `Kafka` custom resource, `goal-violation` means that a goal related imbalance was detected in the cluster.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-goal-violation
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

#### State endpoint in Cruise Control

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

#### Reconciliation logic using the state endpoint

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

If a scale operation interrupts an imbalance rebalance, the imbalance rebalance is dropped.
After the scale operation completes, if the cluster is still imbalanced, Cruise Control will detect the goal violation again in a future reconciliation, and a fresh imbalance rebalance will be triggered.
This prevents wasted work: scale operations change the cluster topology, making any in-progress imbalance rebalance based on outdated information.

#### Detecting Active Rebalances

To determine if a rebalance is already ongoing, the operator uses a unified approach that works for both manual and auto-generated `KafkaRebalance` resources.
Every reconciliation, the operator will:

1. **List all `KafkaRebalance` resources** for the cluster using `kafkaRebalanceOperator.listAsync(namespace, Labels.fromMap(Map.of(Labels.STRIMZI_CLUSTER_LABEL, clusterName)))`
2. **Extract the state** of each `KafkaRebalance` using the existing `KafkaRebalanceUtils.rebalanceState()` utility method
3. **Check for active states**: A rebalance is considered "active" if any `KafkaRebalance` resource is in one of these states:
   - `New` - Resource just created, waiting for operator processing
   - `PendingProposal` - Cruise Control is generating the rebalance proposal
   - `ProposalReady` - Proposal generated, waiting for user approval (for manual rebalances)
   - `Rebalancing` - Rebalance is actively executing partition movements

If any `KafkaRebalance` resource is in an active state, the operator will skip anomaly detection for imbalance mode and wait for the next reconciliation cycle.
This approach works uniformly for:
- **Auto-generated rebalances**: Created by the operator for scale-up, scale-down, or imbalance scenarios
- **Manual rebalances**: Created by users with arbitrary names

The implementation leverages the existing `KafkaRebalanceUtils` class which is already used throughout the Strimzi codebase for state extraction and validation.
All `KafkaRebalance` resources (manual and auto-generated) have the `strimzi.io/cluster` label, allowing the operator to discover them via label selector.

If there is no active rebalance, then we can move forward to check if there were any detected goal violations or not.
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
      "BROKER_FAILURE",
      "DISK_FAILURE",
      "GOAL_VIOLATION",
      "METRIC_ANOMALY",
      "TOPIC_ANOMALY",
      "MAINTENANCE_EVENT"
    ],
    "selfHealingDisabled": [],
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

#### Logic to trigger rebalance

To determine whether a detected anomaly should trigger a new rebalance, we need to compare the anomaly detection timestamp against the completion time of the most recent rebalance operation (whether manual or automatic).
This ensures we don't retrigger rebalances for goal violations that were already addressed.

**Tracking Rebalance Completion Times:**

Since auto-generated `KafkaRebalance` resources are deleted upon completion or failure, and manual `KafkaRebalance` resources can be deleted by users, we need a persistent location to track completion times.
The operator uses a **ConfigMap** to store this information.

**ConfigMap: `<cluster-name>-auto-rebalance-imbalance-tracker`**

This ConfigMap:
- Stores the timestamp of the most recent rebalance completion
- Tracks the current rebalancing phase for JBOD clusters (inter-broker vs. intra-broker)
- Is created in the same namespace as the Kafka cluster
- Persists even after `KafkaRebalance` resources are deleted
- Allows the operator to recover state after crashes or restarts
- Is managed exclusively by the `KafkaAssemblyOperator` (maintains proper ownership boundaries)

**ConfigMap Lifecycle:**

- **Creation**: Created automatically by the operator when the first rebalance (manual or auto-generated) completes
- **Updates**: Updated throughout the lifecycle whenever rebalances complete or JBOD rebalancing phases change
- **Persistence**: The ConfigMap is **not deleted** when rebalances complete or when the Kafka cluster is idle
- **Deletion**: Only deleted when the Kafka cluster itself is deleted (via owner reference to the Kafka CR)
- **Rationale**: Persisting the ConfigMap allows accurate timestamp comparison across reconciliations and operator restarts, preventing duplicate rebalancing operations

Example ConfigMap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-cluster-auto-rebalance-imbalance-tracker
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/kind: Kafka
data:
  lastRebalanceCompletionTime: "2026-06-03T14:23:45Z"
  # For JBOD clusters in RebalanceOnImbalance state, tracks which phase is active
  rebalancingPhase: "inter-broker"  # Values: "inter-broker", "intra-broker", or absent
```

**ConfigMap Data Fields:**

1. **`lastRebalanceCompletionTime`**: ISO-8601 timestamp of the most recent rebalance completion (manual or auto-generated)

2. **`rebalancingPhase`** (optional, only present during active RebalanceOnImbalance for JBOD clusters):
   - `"inter-broker"`: Inter-broker rebalancing is currently active
   - `"intra-broker"`: Intra-broker disk rebalancing is currently active
   - Field is absent when not in RebalanceOnImbalance state or for non-JBOD clusters

**Update Logic:**

To maintain proper operator ownership boundaries (where `KafkaAssemblyOperator` owns the `Kafka` CR and `KafkaRebalanceAssemblyOperator` owns `KafkaRebalance` CRs), the update mechanism works as follows:

The `KafkaAssemblyOperator` (which contains `KafkaAutoRebalanceReconciler`) maintains a watch on `KafkaRebalance` resources with the `strimzi.io/cluster` label matching Kafka clusters it manages.
When any `KafkaRebalance` resource (manual or auto-generated) transitions to a terminal state (Ready, NotReady, or Stopped):
1. The watch trigger fires in `KafkaAssemblyOperator`
2. The operator extracts the cluster name from the `KafkaRebalance` labels (`strimzi.io/cluster`)
3. The operator updates the ConfigMap `<cluster-name>-auto-rebalance-imbalance-tracker` with the current timestamp in the `lastRebalanceCompletionTime` field
4. This happens **before** the `KafkaRebalance` resource is deleted (for auto-generated ones)

This approach:
- Preserves operator ownership boundaries: only `KafkaAssemblyOperator` modifies the state ConfigMap
- Captures completion times for both manual and auto-generated rebalances
- Works even if users delete `KafkaRebalance` resources manually (the watch sees the terminal state before deletion)
- Provides a separate, queryable resource for tracking rebalance state without polluting the Kafka CR status

**Comparison Logic:**

A rebalance on imbalance will be triggered when:
```java
ConfigMap stateConfigMap = getConfigMap(namespace, clusterName + "-auto-rebalance-imbalance-tracker");
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

#### What happens if there are no rebalances running and an anomaly is detected (The Happy Path)

1. The operator retrieves the `lastRebalanceCompletionTime` from the `<cluster-name>-auto-rebalance-imbalance-tracker` ConfigMap and compares the anomaly `detectionDate` against this timestamp (see comparison logic in "Logic to trigger rebalance" section)
2. If the comparison indicates a new anomaly, the operator transitions to `RebalanceOnImbalance` and creates a new `KafkaRebalance` resource
3. Once the rebalance completes, the `KafkaAssemblyOperator` updates the `lastRebalanceCompletionTime` field in the ConfigMap, then the `KafkaAutoRebalancingReconciler` transitions back to `Idle` and deletes the resource

#### What happens if the anomaly is detected during a running rebalance

Using the detection approach described in "Detecting Active Rebalances" section, if any active rebalance is detected, we ignore the newly detected anomaly since it might be resolved by the ongoing operation.
If the anomaly persists after the rebalance completes, Cruise Control will detect it again in a subsequent cycle and a new rebalance will be triggered.

#### What happens if a rebalance fails

When an auto-rebalance on imbalance fails (the `KafkaRebalance` resource transitions to `NotReady` state):

**Step 1 - KafkaAssemblyOperator (via watch):**
1. Detects the NotReady state transition via the `KafkaRebalance` watch
2. Updates the `<cluster-name>-auto-rebalance-imbalance-tracker` ConfigMap with the current timestamp in the `lastRebalanceCompletionTime` field (before the resource is deleted)
3. This records the failure timestamp as the last completion time

**Step 2 - KafkaAutoRebalancingReconciler:**
1. **Propagates the error**: Sets a Warning condition on the Kafka CR with the error message from the `KafkaRebalance` status
2. **Transitions to Idle**: Moves the auto-rebalance state machine from `RebalanceOnImbalance` back to `Idle`
3. **Deletes the resource**: Removes the failed `KafkaRebalance` resource (safe since completion timestamp is in ConfigMap)

**Retry Behavior:**

The operator does NOT immediately retry a failed rebalance.
Instead:
- The ConfigMap's `lastRebalanceCompletionTime` now reflects the failure timestamp
- If the underlying issue persists, Cruise Control will detect the goal violation again in the next anomaly detection cycle
- The new detection will have a later `detectionDate` timestamp
- When compared against `lastRebalanceCompletionTime` (the failure timestamp), the new detection will trigger a fresh rebalance attempt
- This prevents repeated failed attempts while ensuring issues are eventually addressed once resolved

#### What happens if an unfixable goal violation happens

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
      message: |
        The detected goal violation contains unfixable goals: [DiskDistributionGoal]. 
        These violations typically require infrastructure-level changes such as adding more brokers, racks, or resources. 
        Auto-rebalance will not be triggered until the unfixable goals are resolved.
        Fixable goals detected: [NetworkInboundCapacityGoal]. These will be addressed once unfixable goals are resolved.
```

The warning message will include both the unfixable goals (requiring manual intervention) and any fixable goals (which will be addressed once the infrastructure constraints are resolved).

#### What happens if the rebalance template contains invalid goals

If the rebalance template specifies goals that are NOT included in the `anomaly.detection.goals` configuration, the validation will fail.
This prevents the operator from triggering rebalances for goals that Cruise Control is not actively monitoring for violations.

For example, if the anomaly detection goals are configured as `[RackAwareGoal, ReplicaCapacityGoal, DiskCapacityGoal]` but the rebalance template specifies `[RackAwareGoal, CpuCapacityGoal]`, the validation will fail because `CpuCapacityGoal` is not in the anomaly detection goals list.

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
      message: |
        The rebalance template 'my-imbalance-rebalance-template' contains goals that are not configured in anomaly.detection.goals.
        Invalid goals: [CpuCapacityGoal, NetworkInboundCapacityGoal]
        Configured anomaly detection goals: [RackAwareGoal, ReplicaCapacityGoal, DiskCapacityGoal]
        Please update the template to only include goals that are being monitored for anomalies.
```

Users must either:
- Update the rebalance template to only include goals from the anomaly detection configuration, or
- Update the Cruise Control `anomaly.detection.goals` configuration to include the additional goals

#### Stopping a running rebalance

To stop a running rebalance, users can apply the `strimzi.io/rebalance=stop` annotation on the generated `KafkaRebalance` resource.
This will stop the running rebalance and the stopped `KafkaRebalance` resource will be deleted.
Users will need to ensure that they disable the `imbalance` mode of auto-rebalance after stopping the rebalance; otherwise, the rebalance will be triggered again since the anomalies are still present in the cluster.

#### Handling Intra-Broker Disk Imbalances (JBOD Storage)

For Kafka clusters using **JBOD storage with multiple disks**, the `imbalance` mode automatically handles both **inter-broker** (replicas distributed across brokers) and **intra-broker disk imbalances** (disk usage between disks on the same broker).

**Storage Detection and Goal Configuration:**

The operator detects JBOD storage on startup and automatically extends Cruise Control's anomaly detection goals:

- **Non-JBOD storage**: Only inter-broker goals configured (default behavior)
- **JBOD storage**: Automatically adds `IntraBrokerDiskCapacityGoal` and `IntraBrokerDiskUsageDistributionGoal` to anomaly detection

This is **transparent to users** - no additional configuration needed.

**Mutual Exclusivity Constraint:**

Inter-broker and intra-broker rebalancing **cannot run simultaneously** (Cruise Control enforces this).
The operator handles violations sequentially:

**Priority order:**
1. Scale operations (highest)
2. Inter-broker rebalancing 
3. Intra-broker disk rebalancing (lowest)

**Execution Flow:**

When anomalies are detected, the operator categorizes violations by goal type:

1. **Both types detected**: 
   - First: Create `KafkaRebalance` with `rebalanceDisk=false` for inter-broker violations
   - After completion: Create `KafkaRebalance` with `rebalanceDisk=true` for intra-broker violations

2. **Only intra-broker violations**:
   - Create `KafkaRebalance` with `rebalanceDisk=true`
   - Goals automatically set by Cruise Control (template goals ignored)

3. **Only inter-broker violations**:
   - Standard inter-broker rebalance (existing behavior)

**State Machine Handling:**

Both inter-broker and intra-broker rebalancing use the same `RebalanceOnImbalance` state.
The operator tracks which type of rebalance is currently running and executes them sequentially within this state:

- `RebalanceOnImbalance` remains active while processing violations
- For JBOD clusters with both violations: stays in `RebalanceOnImbalance` and transitions from inter-broker → intra-broker rebalancing
- Transitions to `Idle` only after all queued rebalancing (both types) completes

**Generated `KafkaRebalance` Resources:**

Inter-broker:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-goal-violation
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
  name: my-cluster-auto-rebalancing-goal-violation
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

For JBOD clusters, the FSM state `RebalanceOnImbalance` alone is insufficient to recover after an operator crash because it doesn't indicate whether inter-broker or intra-broker rebalancing is active.
The operator uses the `<cluster-name>-auto-rebalance-imbalance-tracker` ConfigMap to track the current rebalancing phase.

The ConfigMap's `rebalancingPhase` field is updated as rebalancing progresses:
- Set to `"inter-broker"` when inter-broker rebalancing starts
- Updated to `"intra-broker"` when intra-broker rebalancing starts
- Removed when all rebalancing completes and FSM transitions to `Idle`

When the operator restarts during `RebalanceOnImbalance`:

1. Read the FSM state from `Kafka.status.autoRebalance.state` (finds `RebalanceOnImbalance`)
2. Read `rebalancingPhase` from the ConfigMap to determine which phase was active:
   - `"inter-broker"` → Inter-broker rebalancing was in progress
   - `"intra-broker"` → Intra-broker rebalancing was in progress
   - Field absent → Not a JBOD rebalance or already completed
3. Check for the existing `KafkaRebalance` resource and verify its `rebalanceDisk` field matches the expected phase
4. Resume monitoring the rebalance from its current state

If the `KafkaRebalance` resource is missing (deleted while operator was down), treat this as a failure: transition to `Idle` and remove the `rebalancingPhase` field from the ConfigMap.

#### Metrics for tracking the rebalance requests

If users want to track when auto-rebalances happen, they can access the Strimzi [metrics](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/grafana-dashboards/strimzi-operators.json#L712) for the `KafkaRebalance` custom resources.
This includes tracking when they were visible/created.
These metrics also cover the `KafkaRebalance` resources which were created automatically, so users can utilize them to understand when an auto-rebalance was triggered in their cluster.
We will add labels to the metrics to differentiate the created rebalances based on mode (i.e., whether the rebalance was triggered for `full` mode for imbalanced clusters, `add-broker` mode, or `remove-broker` mode).
Cruise Control currently does not have any metrics to depict what type of anomaly was detected and when.
Therefore, we will add a new counter metric named `strimzi_auto_rebalance_anomalies_detected_total` which will be incremented whenever an anomaly is detected by the anomaly detector.
We will also add labels to differentiate anomalies based on fixability (`fixable`, `unfixable`, `mixed`) and type of anomaly (`goal_violation`, `broker_failure`, `disk_failure`, etc.).
These metrics will be exposed by the operator and will only be available when the user has configured the `imbalance` mode of auto-rebalance.

#### Detecting Persisting Anomalies with Metrics and Alerts

An anomaly may persist after a rebalance if cluster constraints prevent full optimization, new load patterns emerge immediately, or the rebalance only partially addresses violations.
When this happens, Cruise Control keeps the same anomaly entry with the original `detectionDate`, preventing automatic retriggering (since `detectionDate < lastRebalanceCompletionTime`).

To avoid rebalance loops and cluster instability, the operator does not automatically retry.
Users can optionally monitor the exposed metrics and configure alerts to detect potential issues.
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
* **RebalanceOnImbalance**: Rebalancing triggered by goal violations is running. This state handles both inter-broker rebalancing (default) and intra-broker disk rebalancing (for JBOD storage with multiple volumes). Both types cannot run simultaneously due to Cruise Control constraints, so they are executed sequentially within this state.

With the new `imbalance` mode, the FSM state transitions will look like this:

![auto-rebalance state machine flow diagram](images/136-finite-state-machine.png)

* from **Idle** to:
  * **RebalanceOnScaleDown**: if a scale down operation was requested. This transition happens even if a scale up was requested at the same time but the rebalancing on scaling down has the precedence. The rebalancing on scale up is queued. They will run sequentially.
  * **RebalanceOnScaleUp**: if only a scale up operation was requested. There was no scale down operation requested.
  * **RebalanceOnImbalance**: if goal violations are detected (inter-broker, intra-broker, or both), and the `fixableViolatedGoals` list is not empty while the `unfixableViolatedGoals` list is empty. If there are queued scale up or scale down operations, then they will run first. For JBOD clusters, both inter-broker and intra-broker violations are handled sequentially within this state (inter-broker first, then intra-broker).

* from **RebalanceOnScaleDown** to:
  * **RebalanceOnScaleDown**: if a rebalancing on scale down is still running.
  * **RebalanceOnScaleUp**: if a scale down operation was requested together with a scale up and, because they run sequentially, the rebalance on scale down had the precedence, was executed first and completed successfully. We can now move on with rebalancing for the scale up.
  * **Idle**: if a scale down operation was requested, it was executed and completed successfully/failed or a full rebalance was asked due to an anomaly but since the scale-down rebalance is done, we can ignore the anomalies assuming they are fixed by the rebalance. In case, they are not fixed, Cruise Control will detect them again and a new rebalance would be requested.

* from **RebalanceOnScaleUp**:
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is still running.
  * **RebalanceOnScaleDown**: if a scale down operation was requested, so the current rebalancing scale up is stopped (and queued) and a new rebalancing scale down is started. The rebalancing scale up will be postponed.
  * **Idle**: if a scale up operation was requested, it was executed and completed successfully/failed or a full rebalance was asked due to an anomaly but since the scale-up rebalance is done, we can ignore the anomalies assuming they are fixed by the rebalance. In case, they are not fixed, Cruise Control will detect them again and a new rebalance would be requested.

* from **RebalanceOnImbalance**:
  * **RebalanceOnImbalance**: if a rebalancing on imbalance is still running. For JBOD clusters, this includes transitioning from inter-broker to intra-broker rebalancing within the same state (if both violations were detected).
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is in queue, then the rebalance on imbalance will be stopped and the scale up will happen first. If there is a rebalancing scale down in queue too, then it will be executed before both scale up and rebalance on imbalance.
  * **RebalanceOnScaleDown**: if a scale down operation was requested, then the rebalance on imbalance will be stopped and the scale down will be allowed to finish first and after that rebalance on imbalance will be executed.
  * **Idle**: if rebalancing was requested, it was executed and completed successfully or failed. For JBOD clusters with both violation types, this happens after both inter-broker and intra-broker rebalancing complete.

On each reconciliation, the following process will be used:

1. The operator creates the cluster model instance
2. The auto-rebalance reconciliation logic checks for ongoing rebalances (see "Detecting Active Rebalances" section)
3. If the state is `Idle` and no rebalance is running, query the `state` endpoint to get detected violations
4. Apply the comparison logic from "Logic to trigger rebalance" section to determine if a new rebalance should be triggered
5. For JBOD clusters, categorize and handle violations as described in "Handling Intra-Broker Disk Imbalances" section

The auto-rebalance reconciliation logic also loads the `Kafka.status.autoRebalance` content:

* `state`: is the FSM state
* `lastTransitionTime`: when the transition to that state happened.
* `modes`: enabled modes (`remove-brokers`, `imbalance` and/or `add-brokers`) if an auto-rebalancing requested.

The FSM is initialized based on the `state` field.

Let's see what happens during the auto-rebalancing process when the FSM starts from the **Idle** state and transitions to **RebalanceOnImbalance**.

#### Idle

This state is set initially when a `Kafka` custom resource is created with the `spec.cruiseControl.autoRebalance` field configured.
It is also the end state of a previously completed or failed auto-rebalancing operation.
In case of successful completion, once the rebalance moves to the `Ready` state, we will delete the generated `KafkaRebalance` and update the `auto-rebalance` state to `Idle`.
In case of failed auto-rebalancing, once the rebalance moves to the `NotReady` state, we will propagate the error to the Kafka CR, delete the generated `KafkaRebalance`, and set the state to `Idle`.
In this state, the operator removes the finalizer and deletes the corresponding generated `KafkaRebalance` custom resource.

#### RebalanceOnImbalance

In this state, goal violation anomalies were detected when we sent a request to the `state` endpoint with `anomaly_detector` substate.
A `KafkaRebalance` resource will now be applied to the cluster to fix the cluster imbalance.
This `KafkaRebalance` will be based on the template provided by the user; if no template is provided, then the `KafkaRebalance` will be created with default configurations.

For JBOD clusters, this state handles both inter-broker and intra-broker rebalancing sequentially (see "Handling Intra-Broker Disk Imbalances" section for details)

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
  * **For JBOD clusters**: if intra-broker rebalancing is queued (inter-broker rebalance just completed), create intra-broker `KafkaRebalance` and remain in **RebalanceOnImbalance** state.
  * If no queued operations, transition to **Idle**, clean `Kafka.status.autoRebalance.modes`, delete the generated `KafkaRebalance` custom resource.
* if `PendingProposal`, `ProposalReady` or `Rebalancing`, the rebalancing is still running.
  * No further actions required.
* if `NotReady`
  * The rebalancing failed. Transition to **Idle**, propagate the error to the `Kafka` CR status, and delete the generated `KafkaRebalance` custom resource.
  * The next anomaly detection cycle will detect if the issue persists and trigger a fresh rebalance.
* if `Stopped`
  * the rebalancing is stopped, transition to **Idle** and also removing the corresponding mode from the status. The operator deletes the generated `KafkaRebalance` custom resource. The users should then disable the `imbalance` mode of auto-rebalance else a new rebalance would be triggered.

If, during an ongoing auto-rebalancing, the `KafkaRebalance` custom resource is not there anymore on the next reconciliation, it could mean the user deleted it while the operator was stopped/crashed/not running.
In this case, the FSM will assume it as falling in the `NotReady` case above.

### `KafkaRebalance` Resource Lifecycle for Imbalance Mode

The lifecycle of auto-generated `KafkaRebalance` resources for the `imbalance` mode follows a deterministic pattern:

**Creation:**
A `KafkaRebalance` resource is created when:
- The auto-rebalance state machine is in `Idle` state
- No active `KafkaRebalance` resources exist (checked via `KafkaRebalanceUtils.rebalanceState()`)
- A goal violation is detected with the `detectionDate` timestamp greater than the `lastRebalanceCompletionTime` from the `<cluster-name>-auto-rebalance-imbalance-tracker` ConfigMap (or the ConfigMap doesn't exist / field is null, meaning no rebalance has ever completed)
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
   - Verify operator creates `<cluster>-auto-rebalancing-goal-violation` `KafkaRebalance` resource
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
   - Verify `<cluster-name>-auto-rebalance-imbalance-tracker` ConfigMap's `lastRebalanceCompletionTime` is updated
   - Verify operator transitions to `Idle` and deletes resource

## Affected/not affected projects

This change will affect the Strimzi cluster operator.

### Required API Changes

The following changes need to be made to support this feature:

1. **ConfigMap for state tracking**: Create a new ConfigMap `<cluster-name>-auto-rebalance-imbalance-tracker` to track the completion time of the most recent rebalance operation (manual or automatic). This ConfigMap contains a single data entry: `lastRebalanceCompletionTime`.

2. **Add `imbalance` mode**: Support automatic rebalancing on goal violations (handles both inter-broker and intra-broker for JBOD) in the `Kafka.spec.cruiseControl.autoRebalance` configuration.

3. **Add `RebalanceOnImbalance` state**: New FSM state when rebalancing (inter-broker or intra-broker) triggered by goal violations is running.

4. **Add intra-broker disk goals**: For JBOD storage support: `INTRA_BROKER_DISK_CAPACITY_GOAL` and `INTRA_BROKER_DISK_USAGE_DISTRIBUTION_GOAL`.

5. **Add `maintenanceTimeWindows` field**: Optional field in the auto-rebalance configuration to specify custom maintenance windows for imbalance rebalancing.

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
As this feature evolves, we can also explore ways to automatically fix issues like disk failures and broker failures, since the fix would be driven by the operator rather than Cruise Control directly.