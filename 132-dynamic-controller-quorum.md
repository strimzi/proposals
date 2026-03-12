# Support for dynamic controller quorum and controllers scaling

This proposal is about adding the support for the KRaft dynamic controller quorum within the Strimzi Cluster Operator, in order to support controllers scaling as well.
The dynamic controller quorum was introduced since Kafka 3.9.0 via [KIP-853](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes).

## Current situation

Currently, the Strimzi Cluster Operator supports the KRaft static controller quorum only, despite the Apache Kafka 4.x releases have the support for the dynamic one.
The static controller quorum doesn't allow scaling the controller nodes without downtime.
It means that after the Apache Kafka cluster creation, when the controller nodes are deployed, it is not possible to add or remove controllers to/from the quorum anymore, unless accepting some downtime for the Apache Kafka cluster.

A possible way for scaling controllers is:

* pause the cluster reconciliation.
* delete the controllers' `StrimziPodSet`(s) so that the controller pods are deleted.
* update the number of replicas within the `KafkaNodePool`(s) custom resource(s) related to controllers (increase or decrease).
* unpause the cluster reconciliation.

The Strimzi Cluster Operator will be able to create the new controllers from scratch (with the updated number of replicas) and it will restart the brokers with the new controller quorum configured via the `controller.quorum.voters` property.
Of course, right after the controllers deletion step and until the controller quorum runs again, the cluster won't be available because brokers cannot work without the KRaft controllers running.

This is not parity features with ZooKeeper-based cluster where the ZooKeeper ensemble can be scaled up or down without cluster downtime.

## Motivation

KRaft dynamic controller quorum enables the controllers scaling without downtime.
In general, this operation can be useful for increasing the number of controllers when needed (or, of course, decreasing them).
Also, sometimes, replacing a controller because of a disk or hardware failure is something that can be done by scaling.
Furthermore, even migration from an architecture with combined-nodes to controller dedicated nodes could be achieved by scaling the KRaft quorum.
Finally, this would provide parity features with a ZooKeeper-based cluster with ZooKeeper nodes can be added or removed to the quorum.

Beyond these operational benefits, dynamic quorum support is critical for Apache Kafka's strategic direction and production readiness.
With ZooKeeper support officially removed in Apache Kafka 4.0, KRaft is now the only metadata management option for Kafka clusters.
This makes dynamic quorum capabilities essential rather than optional and it's a fundamental requirement for KRaft to be truly production-ready and not a regression from ZooKeeper-based deployments.
However, it's important to note that while the core dynamic quorum functionality was introduced in Kafka 3.9.0 (via KIP-853), certain bug fixes, improvements and new features like quorum migration from static to dynamic are only available starting from Apache Kafka 4.1.0.
So it is now the right time to add support for it within Strimzi.

Not having such a support presents several challenges for production environments:

* **Operational Risk**: The manual workaround requiring controllers deletion and cluster downtime is error-prone and unacceptable for mission-critical Kafka deployments where availability is important.
* **Cloud-Native Expectations**: Modern Kubernetes-based deployments expect zero-downtime scaling as a standard capability. Static quorum limitations prevent Kafka from meeting these expectations.
* **Migration Barriers**: Organizations evaluating or migrating from ZooKeeper-based clusters to KRaft expect feature parity at minimum. The inability to scale controllers without downtime represents a regression that blocks adoption.
* **Incomplete KRaft Maturity**: While KRaft was introduced to eliminate ZooKeeper dependency and simplify Kafka's architecture, the lack of dynamic quorum support in Strimzi means users cannot fully leverage KRaft's capabilities as designed by the Apache Kafka project.

By implementing dynamic quorum support, Strimzi aligns with Apache Kafka's roadmap and enables users to operate KRaft-based clusters with the same (or better) operational flexibility they had with ZooKeeper, removing a major barrier to KRaft adoption in production environments.

## Proposal

This proposal is about supporting the dynamic controller quorum, thus the controllers scaling, within the Strimzi Cluster Operator, by going through three main steps:

* Add support for dynamic controller quorum to the Strimzi Cluster Operator and using it by default for any newly created Apache Kafka cluster.
* Add support for controllers scaling by leveraging the dynamic controller quorum.
* Add migration from static to dynamic controller quorum.

It also take into account the main operational changes when moving from static to dynamic quorum:

* Node storage formatting is impacted by this change.
* The `controller.quorum.voters` property is now replaced by the `controller.quorum.bootstrap.servers` property on both broker and controller nodes.
* The `kraft.version` feature is now `1` when using dynamic quorum. It's `0` when using static quorum. 

By the way, regarding the last point, there is no need to set the `kraft.version` feature upfront (to `0` or `1`).
It's being set when the controller storage is properly formatted and the quorum is configured as dynamic or static.
The only case needing the feature to be changed is when migrating from static to dynamic quorum (so updating the value from `0` to `1`) for an existing Apache Kafka cluster, when coming from an older operator release.

The following sections describes what's the idea behind the proposal and how to approach the above steps.
As a referece, [here](http://kafka.apache.org/41/operations/kraft/#provisioning-nodes)'s the link to the official Apache Kafka documentation about using dynamic quorum.

### Understanding Static vs Dynamic Quorum

The main difference between static and dynamic quorum lies in where the voters set membership information is stored and how it can be changed.

With **static quorum** (`kraft.version=0`), the quorum membership is hardcoded in the node configuratrion file by using the `controller.quorum.voters` property listing all the controllers which take part at the KRaft quorum.
This membership configuration never changes during the cluster's lifetime and exists only in each node's configuration file.
The voters set is immutable and there is no `VotersRecord` stored in the metadata log.
When controllers start up, they read the voters set directly from their local configuration and the leader discovery/election can start immediately.

With **dynamic quorum** (`kraft.version=1`), the quorum membership is stored in the replicated metadata log itself by using a `VotersRecord`.
The node configuration file uses the `controller.quorum.bootstrap.servers` property instead, which only provides contact points for initial discovery.
The voters set is mutable and can change through Raft consensus operations.
When controllers start up, they discover the voters set by reading the `VotersRecord` from either the bootstrap snapshot (for initial controllers) or by fetching it from the leader metadata log (for newly added controllers, during a scale up).

The key differences can be summarized as:

| Aspect | Static Quorum | Dynamic Quorum |
|--------|---------------|----------------|
| Voters set location | Configuration files | Replicated metadata log |
| Configuration property | `controller.quorum.voters` | `controller.quorum.bootstrap.servers` |
| Persistence | Each node's local config | Raft-replicated snapshot/log |
| Mutability | Immutable | Mutable via Raft consensus |
| Formatting requirement | Just cluster ID and config | Bootstrap snapshot with `VotersRecord` required (for initial controllers) |
| Membership changes | Requires downtime and reconfiguration | Dynamic add/remove operations |

This difference in where membership is stored has significant implications for storage formatting.
In dynamic quorum, membership is mutable state that must be replicated through Raft, which means it must exist in the replicated log from the beginning.
This is why initial controllers must be formatted with a bootstrap snapshot containing the `VotersRecord`, while newly added controllers discover membership by replicating from the existing quorum.
They use the `controller.quorum.bootstrap.servers` configuration to contact an existing controller from which they fetch the metadata log containing the `VotersRecord` for the voters set.
The `VotersRecord` contains critical information including voter IDs, directory IDs (unique UUIDs), endpoints, and supported `kraft.version` ranges.

Without the bootstrap snapshot containing the `VotersRecord`, initial controllers face a deadlock: they cannot participate in elections (becoming candidates unless they know who the voters are, but the this information comes from the `VotersRecord` which must be written to the replicated log by an elected leader.
However, no leader can exist without successful elections, and elections cannot happen without controllers knowing the voters set.
This circular dependency means that if all controllers start without a pre-written `VotersRecord` in their bootstrap snapshot, they would all have an empty voters set, preventing any of them from becoming candidates or holding elections, leaving the cluster unable to bootstrap.
The only way to break this deadlock is to pre-write the `VotersRecord` into the bootstrap snapshot during initial formatting using the `--initial-controllers` parameter, ensuring all initial controllers know the voters set from the beginning before the cluster even starts.

In conclusion, for the dynamic quorum to work correctly, all nodes must have a consistent view of the initial voters set from the replicated log, not from configuration files.
This ensures consistency and allows the quorum to evolve dynamically while maintaining the strong consistency guarantees provided by the Raft consensus protocol.

An alternative approach to bootstrapping a dynamic quorum cluster is to bootstrap a standalone controller first.
It means that, instead of formatting multiple controllers with the full controllers list, only a single controller is formatted and started initially using the `--standalone` flag.
This creates a quorum of one without requiring the controllers list but still writing a bootstrap snapshot file with a corresponding `VotersRecord`.
Once this standalone controller is running, additional controllers can be formatted with `--no-initial-controllers`, started as observers, and then dynamically added to the quorum using the standard add controller operations.
While this approach simplifies the initial bootstrap by avoiding the need to coordinate directory IDs for all controllers upfront, it means the cluster starts with no fault tolerance since a single-controller quorum cannot tolerate any failures.
However, once additional controllers are added and registered as voters, the cluster achieves the desired redundancy and fault tolerance.
This approach can't cope with how Strimzi starts up the cluster nodes all together and doesn't have the possibility to do a rolling start one by one on cluster creation. 

### The storage formatting "challenge"

Currently, when it comes to formatting the node storage (broker or controller) during the node startup, the `kafka-storage.sh` tool is used within the `kafka_run.sh` script with the following parameters:

* the cluster ID (using the `-t` option).
* the metadata version (using the `-r` option).
* the path to the configuration file (using the `-c` option and pointing to `/tmp/strimzi.properties` file within the container).

It also adds the `-g` option in order to ignore the formatting if the storage is already formatted.
Such option is needed because the same tool is executed every time on node startup (i.e. during a rolling) when the node storage is already formatted.
So the current command is the following:

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g
```

The `STRIMZI_CLUSTER_ID` and `METADATA_VERSION` variables are loaded from the corresponding fields within the ConfigMap which is generated for each node by the Strimzi Cluster Operator and mounted on the pod volume.

This works when the Apache Kafka cluster is using the static quorum during a new cluster creation, when all nodes need to be formatted from scratch at the same time, as well as on cluster scaling, when a new broker node is added but the formatting is done the same way (of course controllers scaling is not supported).

The dynamic quorum works differently from the formatting perspective and it makes a clear distintion on the `kafka-storage.sh` usage when it's a new cluster creation or a node addition (both broker or controller).
When the cluster is created, each broker can be formatted the same way we do today but the controller formatting needs the new `--initial-controllers` (or `-I`) option which specifies the controllers list.
The controllers list is a comma-separated list of controllers in the format `node-id@dns-name:port:directory-id`, representing the controllers that will form the KRaft quorum in the new cluster.

The formatting tool is then used the following way:

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -I="$CONTROLLERS"
```

The `CONTROLLERS` variable is the one containing such a controllers list.
The issue on cluster creation is about differentiating between broker and controller to use the right options for the formatting.
The broker storage formatting doesn't use the `--initial-controllers` option.

But when it comes to scaling up, so adding a new broker or a controller on an existing cluster, in both cases the formatting tool needs the `--no-initial-controllers` (or `-N`) option instead, which replace the `--initial-controllers` in case of a controller.

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -N
```

So it means that when using the dynamic quorum, the script executed on node startup needs a way to differentiate between a new cluster creation or a node scaling in order to run the formatting with the appropriate options.

The above scenarios can be summarized this way:

* Static quorum:
  * Broker and controllers are formatted with no additional new options.
* Dynamic quorum:
  * New cluster creation:
    * broker doesn't need any additional options (but it works well with `--no-initial-controllers` as well to simplify the proposal, see later).
    * controller needs the controllers list via `--initial-controllers`.
  * Existing cluster, scale up:
    * broker and controller needs the `--no-initial-controllers` option.

The proposed idea is about having the Strimzi Cluster Operator managing the list of current controllers, with their corresponding directory IDs, in the `Kafka` custom resource status.
This list is stored in a new `controllers` field which contains a list of `KafkaControllerStatus` objects, each having:

* `id`: The node ID of the controller.
* `directoryId`: The directory ID (UUID) of the controller used for KRaft quorum management.

For example:

```yaml
# ...
status:
  # ...
  controllers:
    - id: 3
      directoryId: "r9VGTiw2QUCoDUadCt6q2g"
    - id: 4
      directoryId: "r8E7BKRbR0SQxSIFro6wjg"
    - id: 5
      directoryId: "Anjr8banTey_LqVZsppzYg"
  # ...
```

This field is populated on cluster creation with dynamic quorum and maintained throughout the cluster lifecycle.
Of course, this field is not present when the cluster already exists and it uses the static quorum instead.

During each reconciliation, the Strimzi Cluster Operator builds a `controllers` string from the controllers list in the status and passes it through the mounted ConfigMap to the `kafka_run.sh` script.
The string format is: `node-id@dns-name:port:directory-id,node-id@dns-name:port:directory-id,...`

Importantly, the operator only includes current controllers with the corresponding `directoryId` set when building this string.
This means that during scale-up operations, new controllers being added (which don't have a directoryId in the status yet) are excluded from the controllers string.
These new controllers will format their storage using the `--no-initial-controllers` option and their directory IDs, automatically generated by Kafka, will be discovered later via the KRaft quorum reconciliation process (more details in the following sections).

The script can get the role of the current node, broker and/or controller, by reading it from the `process.roles` field within the `/tmp/strimzi.properties` file.

The `controllers` field in the status is always present for dynamic quorum clusters.
For new clusters, it is populated at creation time with operator generated directory IDs.
For existing static quorum clusters, it is populated during automatic migration after the operator retrieves directory IDs from the running quorum.

The Strimzi Cluster Operator automatically migrates all existing static quorum clusters to dynamic quorum during reconciliation by upgrading the `kraft.version` feature flag from `0` to `1`.
After migration, static quorum is no longer supported and all clusters operate with dynamic quorum.

The `controllers` field is created either at cluster creation (new clusters) or during migration (existing static clusters) and is maintained for the entire life of the cluster.

More details about the controllers list management, the controllers string building, the directory IDs generation and migration from static to dynamic quorum in the following sections.

### New cluster creation and reconciliation with dynamic quorum

The initial cluster creation is going to use the [bootstrap with multiple controllers](https://kafka.apache.org/41/operations/kraft/#bootstrap-with-multiple-controllers) approach.
The Strimzi Cluster Operator:

* builds the broker and controller configuration by setting the `controller.quorum.bootstrap.servers` field (instead of the `controller.quorum.voters` one).
* generates a random directory ID, as a UUID, for each controller.
* saves the `controllers` field (list of `KafkaControllerStatus` objects) within the `Kafka` custom resource status, containing the controller IDs and their corresponding directory IDs.
* builds the controllers string from the controllers status list, and stores it as `controllers` field within the node ConfigMap, to be loaded by the `kafka_run.sh` script where it's needed for formatting the storage properly.

On node start up, the `kafka_run.sh` script loads the `controllers` field from the node ConfigMap into a `CONTROLLERS` variable.
Since all clusters use dynamic quorum (either from creation or after automatic migration), this variable is always set.
The script determines the formatting options as follows:

* Reads the node role from the `process.roles` property in `/tmp/strimzi.properties` into a `PROCESS_ROLES` variable.
* If the node is a broker only, it formats with the `--no-initial-controllers` option (works for both new cluster creation and broker scale-up).
* If the node is a controller:
    * Extracts the controller IDs from the `CONTROLLERS` string.
    * Checks if the current controller's ID is in the controllers list:
      * **Controller in list**:
        * If metadata disk change is detected, formats with `--no-initial-controllers` to generate a new directory ID (handles disk failures or storage replacement scenarios).
        * Otherwise, formats with `--initial-controllers` to preserve the directory ID specified in the controllers string. This handles initial bootstrap, rolling restarts, PVC replacements, and disaster recovery scenarios.
      * **Controller not in list**:
        * Formats with `--no-initial-controllers` to generate a new random directory ID (new controller being added during scale-up).

All the formatting operations also have the `-g` (so `--ignore-formatted`) flag to ensure already-formatted storage is not reformatted.

When a node has the combined role (both broker and controller), the script follows the controller formatting path since the `process.roles` includes being a controller.

### Reconcile an existing cluster with static quorum

When the Strimzi Cluster Operator is upgraded with the latest version supporting dynamic quorum, it's going to reconcile existing `Kafka` custom resources for clusters which are using the static quorum.
The operator detects an existing cluster is using static quorum by querying the `kraft.version` feature flag via the Kafka Admin API.
If `kraft.version=0`, the cluster is using static quorum and the reconciliation proceeds by running an [automatic migration from static to dynamic](#migration-from-static-to-dynamic-quorum). The `controllers` field is also missing within the `Kafka` custom resource status.
If `kraft.version=1`, the cluster is already using dynamic quorum and no migration is needed.

### KRaft quorum reconciliation

The support for dynamic quorum and handling of scaling controllers are part of a broader process called "KRaft quorum reconciliation".
Scaling controllers up or down within a KRaft quorum involves more than just spinning up new nodes or shutting down existing ones.
When a new controller is added to the Apache Kafka cluster, it has to be registered in order to join the KRaft quorum.
When a running controller is shut down, it has to be unregistered in order to leave the KRaft quorum.
The KRaft quorum reconciliation is a critical process that ensures the KRaft controller quorum membership accurately reflects the desired state of the cluster.
The main goal is reconciling the current KRaft quorum gathered from the Apache Kafka cluster with the desired controllers state requested by the users.
This process handles controller registration and unregistration needed for scaling up/down operations, metadata disk changes and disaster recovery.
It also takes part within the migration from static to dynamic quorum process.
The proposal is to have a new dedicated `KRaftQuorumReconciler` class to manage the necessary reconciliation steps.

The KRaft quorum reconciliation fits into the `KafkaReconciler` reconciliation process in this way:

* BEGIN RECONCILIATION
  * ... (other reconcile operations) ...
  * **KRaft quorum reconciliation**: analyze current quorum state, unregister and register controllers as needed (typically unregisters controllers being scaled down).
  * scale down controllers.
  * ... (other reconcile operations) ...
  * **KRaft quorum reconciliation** (rolling): for each controller pod restart, `KafkaRoller` invokes a "single-controller reconciliation" to handle metadata disk changes immediately (unregister old voter with stale directory ID, register new observer with current directory ID).
  * scale up new controllers.
  * ... (other reconcile operations) ...
  * **KRaft quorum reconciliation**: analyze current quorum state, unregister and register controllers as needed (typically registers newly added controllers).
  * ... (other reconcile operations) ...
* END RECONCILIATION

The following sections provide additional details about how the KRaft quorum reconciliation works and controllers registration and unregistration are executed.

#### When it happens

The KRaft quorum reconciliation is executed at two key points during the `KafkaReconciler` reconciliation process.
Each invocation performs a full reconciliation: it analyzes the current quorum state and executes both unregistration and registration operations as needed.

At the beginning of reconciliation, and before scaling down controllers:
- Analyzes the current quorum state by querying the Kafka cluster for the current voters and observers.
- Performs unregistration and registration operations as needed. Typically unregisters controllers that are being scaled down, ensuring they are removed from the quorum before being shut down.

At the end of reconciliation, and after all rolling updates and scale up operations:
- Analyzes the quorum state again by querying the Kafka cluster.
- Performs unregistration and registration operations as needed. Typically registers newly added controllers that are running as observers and ready to join the quorum (i.e., caught up with the metadata log and synchronized with the active voters).

This two-phase approach ensures that:

- Controllers are unregistered before being deleted (during scale-down).
- Controllers are only registered after they have been rolled and are running with the correct configuration (during scale up or recovery).

This approach is similar to what is already used today for brokers unregistration which compares, at the end of each reconciliation, the list of registered brokers retrieved via the Kafka Admin API with the "desired" brokers running in the cluster, in order to determine which ones are missing and need to be unregistered.
In the controllers scaling scenario, the operator handles the registration as well (not only the unregistration).

In addition to handling the registration and unregistration of controllers, the Strimzi Cluster Operator has to check the health of the KRaft quorum before allowing the scale down of the controllers.
At the beginning of the `KafkaReconciler` reconciliation process, if scaling down controllers could break the quorum (by losing the majority of voters needed for consensus), it should be blocked and reverted back by the operator.
A similar approach is already in place when attempting to scale down brokers that are hosting partitions, which could break HA.

#### How it works

The KRaft quorum reconciliation process follows these steps:

Query the current quorum state: Uses the Kafka Admin API `describeMetadataQuorum()` method to retrieve the current list of voters and observers, along with their directory IDs provided by the `QuorumInfo` object.

Analyze desired vs. actual state, for each "desired" controller node:
- Checks if the controller pod has been rolled with the controller role (verified by the presence of the `strimzi.io/controller-role` label on the pod).
- Compares the controller's current state in the quorum (voter, observer, or absent) with the expected state based on the `controllers` status field.
- Determines if the controller needs to be registered, unregistered, or if it's already in the correct state.

Detect and handle metadata disk changes, when a controller's disk is replaced (e.g., new JBOD disk, changing metadata disk, PVC replacement, disaster recovery), the controller will have a new directory ID:
- The quorum will show multiple incarnations of the same node ID. The old voter with the old directory ID, and a new observer with the new directory ID.
- The reconciliation process reads the actual directory ID from the controller pod's `meta.properties` file using the Kafka Agent.
- Unregisters the old voter with the stale directory ID.
- Registers the new observer with the current directory ID.

Execute quorum changes in two phases:
- Unregistration: Removes controllers from the quorum using the Kafka Admin API `removeRaftVoter()` method. This handles scale-down and removal of stale voters after metadata disk changes. Unregistration happens first to allow metadata disk changes scenarios where a node already exists as a voter with an old directory ID and needs to be replaced with a new one.
- Registration: Adds controllers to the quorum using the Kafka Admin API `addRaftVoter()` method. This handles scale-up and re-registration after metadata disk changes.

Update the status: After successful reconciliation, the operator queries the quorum state one final time to retrieve the current voters and their directory IDs, then updates the `controllers` field in the `Kafka` custom resource status.

Retrieving the actual directory ID for a specific controller from the `meta.properties` file is a feature which is going to be added to the Kafka Agent and exposed through a dedicated `/directory-id` HTTP endpoint.
This endpoint accepts GET requests and returns the directory ID as a plain text UUID string read directly from the `meta.properties` file located in the metadata log directory.
The Kafka Agent is also modified to receive the KRaft metadata log directory path via an environment variable, enabling it to locate and read the `meta.properties` file.
Importantly, this endpoint is only invoked when the reconciliation detects multiple incarnations of a controller (same node ID with different directory IDs), and is not called during normal operations, minimizing performance impact.

#### Reconciliation algorithms

The KRaft quorum reconciliation operates through two distinct paths, each with specific responsibilities.

The `KRaftQuorumReconciler` class provides the following main API methods:
- `reconcileControllerQuorum()`: Entry point for full quorum reconciliation, used by `KafkaReconciler`.
- `reconcileSingleController()`: Entry point for single controller reconciliation, used by `KafkaRoller`.

These methods use the following helper methods:
- `analyzeQuorumChanges()`: Analyzes all desired controllers and builds lists of controllers to register/unregister (calls `analyzeControllerNode()` for each controller).
- `analyzeControllerNode()`: Analyzes a single controller node to determine if it needs registration/unregistration.
- `executeQuorumChanges()`: Executes the unregistration and registration operations in two phases.
- `buildControllerStatuses()`: Builds controller statuses from the final quorum state.
- `describeMetadataQuorum()`: Kafka Admin API method to query the current quorum state.

**Full Quorum Reconciliation** (executed during normal cluster reconciliation):

1. Describe current metadata quorum state using `describeMetadataQuorum()`
2. Analyze desired controllers via `analyzeQuorumChanges()` method which builds `toRegister` and `toUnregister` lists:
   - For each desired controller node, calls `analyzeControllerNode()` method:
     - Check if the pod has the `strimzi.io/controller-role` label (ensures pod has been rolled as controller)
     - Find if this node ID is part of voters and/or observers in the quorum
     - If multiple incarnations detected (same node ID with different directory IDs):
       - Read actual directory ID from pod via Kafka Agent HTTP endpoint
       - Add voter with old directory ID to `toUnregister` list
       - Add observer matching the new directory ID (read from the node) to `toRegister` list
     - Else if node has no voter but has observer:
       - Add the observer to `toRegister` list (handles scale-up or broker to combined node scenarios)
3. Analyze current voters and identify unwanted ones:
   - For each voter not in desired controllers:
     - Add to `toUnregister` list (handles scale-down scenarios)
4. Execute changes via `executeQuorumChanges()` method in two phases:
   - Phase 1: Unregister all controllers in the `toUnregister` list (stale/unwanted controllers). If one of the controller is the leader, it will be unregistered as the last one to avoid useless leader elections in between multiple controllers unregistrations.
   - Phase 2: Register all controllers in the `toRegister` list (new controllers)
5. Read final quorum state using `describeMetadataQuorum()`
6. Build controller statuses from voters via `buildControllerStatuses()` method and return for the `Kafka` custom resource status update

**Single Controller Reconciliation** (called by KafkaRoller after individual pod restart):

1. Describe current metadata quorum state using `describeMetadataQuorum()`
2. Analyze only the restarted controller via `analyzeControllerNode()` method which builds `toRegister` and `toUnregister` lists (using same logic as full reconciliation)
3. Execute changes via `executeQuorumChanges()` method in two phases (unregister first, then register)
4. Done (status building happens in the full reconciliation)

#### Integration with rolling updates

During rolling updates, the quorum reconciliation also operates at a finer granularity through the `KafkaRoller`:

- After each controller pod is restarted, the roller invokes a "single-controller reconciliation" to handle potential metadata disk changes immediately.
- This ensures that if a controller's disk was replaced during the restart, the old voter is unregistered and the new one is registered right away, without waiting for the full reconciliation cycle.
- This approach minimizes the time a controller with a stale directory ID remains in the quorum, reducing potential issues.

The single-controller reconciliation follows the same analysis and execution logic as the full KRaft quorum reconciliation, but focuses only on the specific controller that was just restarted.

#### Handling failures

The KRaft quorum reconciliation is designed to be resilient to failures:

- If querying the quorum state fails, the reconciliation will fail and retry in the next reconciliation cycle.
- If reading a controller's directory ID fails (e.g., pod not ready, agent unavailable), the reconciliation fails for that controller and retries later.
- If unregistration or registration operations fail (e.g., controller not caught up, quorum not healthy), the operation is retried in the next reconciliation cycle.
- If the operator crashes in the middle, the reconciliation will happen on start up again.
- The process is idempotent: running it multiple times with the same desired state will converge to the correct result without causing issues.

This reconciliation approach ensures that the KRaft quorum membership is always kept in sync with the desired cluster state, handling scale-up, scale-down, metadata disk changes and recovery scenarios transparently.

### Adding new controllers (scale up)

When the controllers are scaled up, the Strimzi Cluster Operator:

* builds the new controllers configuration by setting the `controller.quorum.bootstrap.servers` field.
* starts up all the newly added controllers in parallel (but not running their registration explicitly when they are ready).

On node start up, within the `kafka_run.sh` script:

* the controller node is formatted by using the `--no-initial-controllers` option.

The Strimzi Cluster Operator will also update the `controller.quorum.bootstrap.servers` configuration within the ConfigMap(s) for the existing brokers and controllers.
It will not be used immediately because all nodes will get the information about newly added controllers dynamically via the KRaft protocol.
The new `controller.quorum.bootstrap.servers` configuration will be used on next start up, loaded from the updated ConfigMap(s).

When the controller is up and running, the Strimzi Cluster Operator:

* adds/registers the controllers within the quorum, one by one in sequence, by using the `addRaftVoter` method in the Kafka Admin API.

In order to add a controller, its directory ID is needed together with the corresponding endpoints (advertised listeners).
The directory ID can be easily retrieved by using the Kafka Admin Client API `describeMetadataQuorum()` method, and extract the `ReplicaState.replicaDirectoryId()` for the specific controller (by using its ID).
But it also needs the advertised listeners for the new controller, via a `Set<RaftVoterEndpoint> endpoints` parameter.
The Kafka Admin Client API exposes such information via the `Node` class which is not provided by the `voters()` and `observers()` in the `QuorumInfo` instance but only by the `nodes()` which includes only controllers already in the quorum (the new ones are still observers).
If using the `describeCluster()`, it returns the brokers only instead.
This issue was also raised in this [discussion](https://lists.apache.org/thread/mq5n8j8w0ycg9fhd1m4zcc2ctrgfd0d4) on [KIP-1141](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=340037949).

The suggestion here is to build the controller endpoint within the operator logic as it's already done within the `KafkaBrokerConfigurationBuilder`, because in the end it's the operator itself configuring the new controller so creating the advertised listeners list.

Within the same reconciliation, multiple controllers can be scaled up simultaneously.
When ready, these new controllers start as observers and begin catching up with the metadata from the existing quorum voters, but they don't join the quorum yet.
The operator then runs the registrations sequentially one by one, meaning it can only issue a new `addRaftVoter` request after the previous one completes.
Apache Kafka's internal logic only allows controllers to be promoted to voters once they are fully synchronized with the quorum.
When Apache Kafka receives a registration request, it writes a `VotersRecord` to the metadata log, and the controllers handle the quorum membership change.
This change can take time because the controller needs to catch up, and if the operator issues another `addRaftVoter` request within the same reconciliation, before the previous registration request completes, it will be refused.
In such cases, the operator stops further registrations, allowing the reconciliation to end, and the remaining registrations will proceed in subsequent reconciliations.
This means that while all controllers start up in a single reconciliation, registering them all as voters may require multiple reconciliation cycles.

Furthermore, monitoring that the controller has caught up with the active ones is not necessary.
If the controller hasn't caught up yet and the operator runs the registration, the active controller returns an error and the reconciliation will fail, allowing the registration to be re-tried in the next reconciliation cycle.
The same happens if getting the quorum information or registering the controllers fails (Apache Kafka returns an error), the reconciliation fails and the operator will re-try the operation on the next reconciliation.

Scaling up the KRaft quorum can be done in the following ways:

* by increasing the number of replicas for an existing `KafkaNodePool` with the `controller` role (both combined or dedicated nodes).
* by adding a new `KafkaNodePool` with `controller` role.
* by adding the `controller` role to an existing `KafkaNodePool` hosting `broker`-only nodes.

#### Scale-up scenarios

The following scenarios demonstrate how the KRaft quorum reconciliation handles controller scale-up operations, including both the success path and failure recovery mechanisms.

> Each node got from a describe quorum operation, both voter or observer, is represented as a couple `<nodeId>:<directoryId>` (i.e. `3:A`).

> Each controller within the `Kafka` custom resource status is represented with `{<nodeId>:<directoryId>}` (i.e. `{3:A}`) to reflect the JSON nature of the the `KafkaControllerStatus` object.

**Scenario 1.A: Scale up controller (success path)**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- User scales up by adding controller 6
- Desired controllers become: [3, 4, 5, 6]

*Reconciliation:*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}]
- Analysis: Node 6 in desired but NOT in status
- Pod 6 starts, it's formatted with a new random UUID "D" (by Kafka tool) and becomes observer 6:D
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D]
- Analysis: Node 6 in desired but NOT in voters, needs registration
- Operator finds observer 6:D and registers it using `addRaftVoter`, **SUCCESS**

*Outcome:*
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Status syncs from quorum: [{3:A}, {4:B}, {5:C}, {6:D}]

**Scenario 1.B: Scale up, registration fails**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- User scales up by adding controller 6
- Desired controllers become: [3, 4, 5, 6]

*Reconciliation (first attempt):*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}]
- Analysis: Node 6 in desired but NOT in status
- Pod 6 starts, it's formatted with a new random UUID "D" (by Kafka tool) and becomes observer 6:D
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D]
- Analysis: Node 6 in desired but NOT in voters, needs registration
- Operator attempts to register 6:D, **FAILS**

*Outcome (first attempt):*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D]
- Reconciliation failed, status not updated, remains [{3:A}, {4:B}, {5:C}]

*Next reconciliation (automatic retry):*
- Desired controllers: [3, 4, 5, 6]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D]
- Analysis: Node 6 in desired but NOT in voters, retry registration
- Operator registers 6:D, **SUCCESS**

*Outcome (retry):*
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Status syncs from quorum: [{3:A}, {4:B}, {5:C}, {6:D}]

**Scenario 1.C: Scale up, operator crashes after successful registration**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- User scales up by adding controller 6
- Desired controllers become: [3, 4, 5, 6]

*Reconciliation:*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}]
- Analysis: Node 6 in desired but NOT in status
- Pod 6 starts, it's formatted with a new random UUID "D" (by Kafka tool) and becomes observer 6:D
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D]
- Analysis: Node 6 in desired but NOT in voters, needs registration
- Operator registers 6:D, **SUCCESS**
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Operator **CRASHES** before updating status

*Outcome (with crash):*
- Status: [{3:A}, {4:B}, {5:C}] (outdated, missing 6:D)

*Next reconciliation (automatic recovery):*
- Desired controllers: [3, 4, 5, 6]
- Status: [{3:A}, {4:B}, {5:C}] (outdated)
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Analysis: Node 6 in desired AND already in voters, no action needed

*Outcome (recovery):*
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Status syncs from quorum: [{3:A}, {4:B}, {5:C}, {6:D}]

These scenarios demonstrate the idempotent nature of the reconciliation process: it can safely retry across multiple cycles until convergence, and the status automatically self-heals by syncing from the actual quorum state.

### Removing controllers (scale down)

When the controllers are scaled down, the Strimzi Cluster Operator:

* removes/unregisters the controllers from the quorum by using the `removeRaftVoter` method in the Kafka Admin API.
* scales down the controllers by deleting the corresponding pods.

If getting the quorum information or unregistering the controllers fails (the Apache Kafka returns an error), the reconciliation will fail to avoid the controllers, not unregistered correctly, to be shut down.
The operator will re-try the operation on the next reconciliation.

The Strimzi Cluster Operator will also update the `controller.quorum.bootstrap.servers` configuration within the ConfigMap(s) for the existing brokers and remaining controllers.
It will not be used immediately because all nodes will get the information about removed controllers dynamically via the KRaft protocol.
The new `controller.quorum.bootstrap.servers` configuration will be used on next start up, loaded from the updated ConfigMap(s).

In order to remove a controller, its directory ID is needed.
It can be retrieved by using the Kafka Admin Client API `describeMetadataQuorum` method, and extract the `ReplicaState.replicaDirectoryId()` for each node within the `voters` field from the `QuorumInfo`.

Within the same reconciliation, multiple controllers can be scaled down simultaneously.
The operator then runs the unregistrations sequentially one by one, meaning it can only issue a new `removeRaftVoter` request after the previous one completes.
Apache Kafka's internal logic only allows controllers to be removed from voters if the quorum remains healthy and operational.
When Apache Kafka receives an unregistration request, it writes a `VotersRecord` to the metadata log, and the controllers handle the quorum membership change.
This change can take time, and if the operator issues another `removeRaftVoter` request within the same reconciliation, before the previous unregistration request completes, it will be refused.
In such cases, the operator stops further unregistrations and fails the reconciliation to prevent shutting down controllers that haven't been properly unregistered, allowing the remaining unregistrations to proceed in subsequent reconciliations.
This means that while all controllers may be marked for removal in a single reconciliation, unregistering them all from the quorum may require multiple reconciliation cycles.
During this process, all controllers remain running, but some may still be voters while others have been unregistered and are now observers.
Once all unregistrations complete successfully, the reconciliation can proceed with shutting down the controllers.

Scaling down the KRaft quorum can be done in the following ways:

* by decreasing the number of replicas for an existing `KafkaNodePool` with the `controller` role (both combined or dedicated nodes).
* by removing the `controller` role to an existing `KafkaNodePool` hosting combined nodes (both brokers and controllers).
* by deleting an existing `KafkaNodePool` with `controller` role.

In the last use case, the removed controllers are correctly unregistered but there is no protection against the possibility of having a non-healthy KRaft quorum because the `KafkaNodePool` custom resource deletion, done by the user, can't be blocked or reverted back.

#### Scale-down scenarios

The following scenarios demonstrate how the KRaft quorum reconciliation handles controller scale-down operations, including both the success path and failure recovery mechanisms.

> The notation `(garbage)` alongside an observer means that the node is waiting for the 5 minutes timeout to be removed from the list (Apache Kafka internals)

**Scenario 2.A: Scale down controller (success path)**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5, 6]
- Status: [{3:A}, {4:B}, {5:C}, {6:D}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]

*User action:*
- User scales down by removing controller 6
- Desired controllers become: [3, 4, 5]

*Reconciliation:*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}, {6:D}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Analysis: Node 6 NOT in desired but IS in voters, needs unregistration
- Operator unregisters 6:D using `removeRaftVoter`, **SUCCESS**
- Controller pod 6 is then safely deleted

*Outcome:*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D (garbage)]
- Status syncs from quorum: [{3:A}, {4:B}, {5:C}]

**Scenario 2.B: Scale down, unregistration fails**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5, 6]
- Status: [{3:A}, {4:B}, {5:C}, {6:D}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]

*User action:*
- User scales down by removing controller 6
- Desired controllers become: [3, 4, 5]

*Reconciliation (first attempt):*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}, {6:D}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Analysis: Node 6 NOT in desired but IS in voters, needs unregistration
- Operator attempts to unregister 6:D, **FAILS**

*Outcome (first attempt):*
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Reconciliation failed to prevent shutting down controller not properly unregistered
- Status not updated, remains [{3:A}, {4:B}, {5:C}, {6:D}]
- Pod 6 continues running as a voter

*Next reconciliation (automatic retry):*
- Desired controllers: [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}, {6:D}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Analysis: Node 6 NOT in desired but IS in voters, retry unregistration
- Operator unregisters 6:D, **SUCCESS**
- Pod 6 can be safely deleted

*Outcome (retry):*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D (garbage)]
- Status syncs from quorum: [{3:A}, {4:B}, {5:C}]

**Scenario 2.C: Scale down, operator crashes after successful unregistration**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5, 6]
- Status: [{3:A}, {4:B}, {5:C}, {6:D}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]

*User action:*
- User scales down by removing controller 6
- Desired controllers become: [3, 4, 5]

*Reconciliation:*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}, {6:D}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C, 6:D], observers=[0:E, 1:F, 2:G]
- Analysis: Node 6 NOT in desired but IS in voters, needs unregistration
- Operator unregisters 6:D, **SUCCESS**
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D (garbage)]
- Operator **CRASHES** before updating status

*Outcome (with crash):*
- Status: [{3:A}, {4:B}, {5:C}, {6:D}] (outdated, still has 6:D)

*Next reconciliation (automatic recovery):*
- Desired controllers: [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}, {6:D}] (outdated)
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D (garbage)]
- Analysis: Node 6 NOT in desired and NOT in voters, no action needed
- Pod 6 can be safely deleted (if not already deleted)

*Outcome (recovery):*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 6:D (garbage)]
- Status syncs from quorum: [{3:A}, {4:B}, {5:C}]

These scenarios demonstrate the idempotent nature of the reconciliation process and how it prevents controllers from being shut down before proper unregistration, while the status automatically self-heals by syncing from the actual quorum state.

### From dedicated to combined nodes (and viceversa)

The KRaft quorum reconciliation also handles scenarios where the cluster topology changes by adding or removing the controller role to/from existing nodes.

#### Adding controller role to broker-only nodes

When the controller role is added to an existing `KafkaNodePool` that was previously hosting broker-only nodes, the cluster effectively scales up the controller quorum by converting dedicated brokers into combined nodes.

The Strimzi Cluster Operator:

* detects the role change in the `KafkaNodePool` configuration.
* builds the node configuration by adding the controller-specific settings and updating `controller.quorum.bootstrap.servers` field for all nodes.
* triggers a rolling update of the affected nodes.

During the rolling update, when each affected node restarts:

* the `kafka_run.sh` script detects the controller role in the `process.roles` property.
* since this is an existing cluster and the node is not part of the controllers list, it formats the controller storage using the `--no-initial-controllers` option but it won't be actually re-formatted.
* the node starts as a combined node with both broker and controller roles, joining the quorum as an observer.
* the pod is labeled with the `strimzi.io/controller-role` label to indicate it has been successfully rolled as a controller.

After the rolling update completes, the KRaft quorum reconciliation (at the end of the reconciliation cycle):

* analyzes the quorum state and detects the newly added controllers running as observers.
* checks the `strimzi.io/controller-role` label on each pod to verify it has been rolled as a controller before attempting registration.
* registers them as voters using the `addRaftVoter` API, following the same sequential registration process described in the scale-up section.

The `strimzi.io/controller-role` label check is critical for resilience: if the operator crashes after some nodes have been rolled but before registration completes, on restart the operator can detect which nodes have already been rolled as controllers (by checking the label) and only register those, skipping nodes that haven't been rolled yet.
This prevents attempting to register nodes that are desired to be controllers but haven't yet been restarted as controllers.
The entire rolling cycle of all nodes need to complete before the KRaft quorum reconciliation attempts registering the new controllers.

The `controller.quorum.bootstrap.servers` configuration is updated on all existing nodes (both brokers and controllers) to include the newly added controllers, though this configuration will only be used on their next restart since nodes receive quorum membership updates dynamically via the KRaft protocol.

#### Removing controller role from combined nodes

When the controller role is removed from an existing `KafkaNodePool` hosting combined nodes, the cluster effectively scales down the controller quorum by converting combined nodes back to broker-only nodes.

The Strimzi Cluster Operator:

* detects the role change in the `KafkaNodePool` configuration.
* checks the KRaft quorum health to ensure removing these controllers won't break the quorum (losing the majority needed for consensus).

At the beginning of the reconciliation cycle, the KRaft quorum reconciliation:

* analyzes the quorum state and identifies controllers that need to be removed (those in the current voters but no longer having the controller role in the desired configuration).
* unregisters them from the quorum using the `removeRaftVoter` API, ensuring they are removed from the voter list before the rolling update begins.

After unregistration completes:

* the operator builds configurations of the affected nodes without controller-specific settings.
* triggers a rolling update of the affected nodes.
* updates the `controller.quorum.bootstrap.servers` configuration on all remaining nodes to exclude the removed controllers.

During the rolling update, when each affected node restarts:

* the `kafka_run.sh` script detects the node is now a broker-only (no controller role in `process.roles`).
* the node starts without initializing controller storage, operating solely as a broker.
* the `strimzi.io/controller-role` label is removed from the pod.

The metadata stored on the controller's storage directory from when it was a combined node remains on disk but is no longer used or maintained.

If the reconciliation fails during the unregistration phase (e.g., Kafka returns an error), the rolling update will not proceed, preventing broker-only nodes from being restarted while still registered as voters in the quorum.

#### Broker to combined node scenarios

The following scenarios demonstrate how the KRaft quorum reconciliation handles the conversion of broker-only nodes to combined nodes by adding the controller role.

**Scenario 3.A: Broker to combined node (success path)**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- Note: Brokers 0, 1, 2 are already observers (all KRaft nodes join as observers initially)

*User action:*
- User adds controller role to broker pool hosting nodes 0, 1, 2
- Desired controllers become: [0, 1, 2, 3, 4, 5]

*Reconciliation:*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}]
- Analysis: Nodes 0, 1, 2 in desired but NOT in status, new controllers
- Operator builds new configuration with controller-specific settings
- Rolling update triggered for nodes 0, 1, 2
- Pods 0, 1, 2 rolled and restarted with controller role
- Each node becomes combined node, already has directory ID (E, F, G respectively) in meta.properties
- Pods labeled with controller role label
- DescribeQuorum (after rolling): voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- Analysis: Nodes 0, 1, 2 in desired but NOT in voters, need registration
- Operator finds observers 0:E, 1:F, 2:G and registers them sequentially, **SUCCESS**

*Outcome:*
- DescribeQuorum: voters=[0:E, 1:F, 2:G, 3:A, 4:B, 5:C], observers=[]
- Status syncs from quorum: [{0:E}, {1:F}, {2:G}, {3:A}, {4:B}, {5:C}]

**Scenario 3.B: Broker to combined node, registration fails, operator crashes**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- User adds controller role to broker pool hosting nodes 0, 1, 2
- Desired controllers become: [0, 1, 2, 3, 4, 5]

*Reconciliation (first attempt):*
- Status before reconciliation: [{3:A}, {4:B}, {5:C}]
- Analysis: Nodes 0, 1, 2 in desired but NOT in status, new controllers
- Pods 0, 1, 2 rolled and restarted with controller role
- Pods labeled with controller role label, already have E, F, G in meta.properties
- DescribeQuorum (after rolling): voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- Analysis: Nodes 0, 1, 2 in desired but NOT in voters, need registration
- Operator registers 0:E, **SUCCESS**
- Operator attempts to register 1:F, **FAILS**
- DescribeQuorum: voters=[0:E, 3:A, 4:B, 5:C], observers=[1:F, 2:G]
- Operator **CRASHES** before updating status

*Outcome (with crash):*
- Status: [{3:A}, {4:B}, {5:C}] (outdated, missing 0, 1, 2)

*Next reconciliation (automatic recovery):*
- Desired controllers: [0, 1, 2, 3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}] (outdated)
- DescribeQuorum: voters=[0:E, 3:A, 4:B, 5:C], observers=[1:F, 2:G]
- Analysis:
  - Node 0 in desired AND already in voters, no action needed
  - Nodes 1, 2 in desired but NOT in voters, need registration
- Operator registers 1:F and 2:G, **SUCCESS**

*Outcome (recovery):*
- DescribeQuorum: voters=[0:E, 1:F, 2:G, 3:A, 4:B, 5:C], observers=[]
- Status syncs from quorum: [{0:E}, {1:F}, {2:G}, {3:A}, {4:B}, {5:C}]

These scenarios demonstrate how the controller role label check ensures that only nodes that have been successfully rolled with the controller role are registered, and how the reconciliation process handles partial registration failures with automatic recovery.

### Metadata disk changes

When a controller's metadata disk is replaced or changed (e.g., PVC replacement, disaster recovery, JBOD disk configuration changes, moving metadata to a different disk), the controller will have a new directory ID because each disk has its own unique directory ID stored in the `meta.properties` file.
The KRaft quorum reconciliation handles this scenario by detecting the directory ID mismatch and updating the quorum membership accordingly.

#### Immediate handling during rolling updates

When a controller pod restarts and the metadata disk has changed, the `KafkaRoller` detects and handles this immediately through a "single-controller reconciliation":

* After the controller pod successfully restarts, the `KafkaRoller` invokes a "single-controller reconciliation" for that specific controller.
* The reconciliation queries the current quorum state using `describeMetadataQuorum()` and detects multiple incarnations of the same node ID:
  * An old voter with the stale directory ID (from before the disk change).
  * A new observer with the current directory ID (the controller's actual current state).
* The reconciliation reads the actual directory ID from the controller pod's `meta.properties` file using the Kafka Agent's `/directory-id` HTTP endpoint.
* Compares the actual directory ID with the directory IDs found in the quorum:
  * Unregisters the old voter with the stale directory ID using the `removeRaftVoter` API.
  * Registers the new observer with the current directory ID using the `addRaftVoter` API.
* This happens immediately during the rolling update, minimizing the time the controller with a stale directory ID remains registered in the quorum.

The "single-controller reconciliation" follows the same two-phase approach as the full quorum reconciliation: unregistration first, then registration.
This ensures that even if the controller already exists as a voter with the old directory ID, it can be cleanly replaced.

#### Handling failures and recovery

If the single-controller reconciliation fails during the rolling update (e.g., unable to read directory ID from the pod, Kafka API errors, operator crash), the `KafkaRoller` will fail the rolling update for that controller:

* The rolling process throws an error, and the reconciliation is marked as failed.
* The controller remains running with the new directory ID but may still be registered with the old directory ID in the quorum, or not registered at all.
* No subsequent controllers in the rolling sequence are restarted until this issue is resolved.

On the next reconciliation cycle, the full KRaft quorum reconciliation (at the end of the reconciliation) recovers from this failure:

* Analyzes all controllers, including the one that failed during rolling.
* Detects the multiple incarnations (if the old voter wasn't unregistered) or missing voter (if unregistration succeeded but registration failed).
* Reads the actual directory ID from the controller pod's `meta.properties` file using the Kafka Agent.
* Performs the necessary unregistration and registration operations to bring the controller's quorum membership in sync with its actual state.
* If needed, the `KafkaRoller` proceeds with rolling other controllers (it depends when the error was raised and the reconciliation failed).
* Updates the `controllers` field in the `Kafka` custom resource status with the current directory IDs.

This recovery mechanism ensures that even if failures occur during rolling updates, the quorum membership will eventually converge to the correct state.
The idempotent nature of the reconciliation process means it can be safely retried across multiple reconciliation cycles until it succeeds.

The directory ID reading from the Kafka Agent is critical for this process.
If the agent is unavailable or the read fails, the reconciliation will fail and retry later, preventing incorrect assumptions about which directory ID is current.

#### Metadata disk change scenarios

The following scenarios demonstrate how the KRaft quorum reconciliation handles metadata disk changes during rolling updates, including both the success path and various failure recovery mechanisms.

**Scenario 4.A: Disk change, all operations succeed**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- Disk change on controller node pool (e.g., JBOD configuration change)
- Rolling update triggered

*Reconciliation (controller 3 rolls):*
- Pod 3 restarts with new disk, new directory ID: A'
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']
- Roller delegates to the KRaft quorum reconciler for single-controller reconciliation
- Multiple incarnations detected (voter 3:A + observer 3:A')
- Reads meta.properties from pod 3 via Kafka Agent: A'
- Unregisters 3:A, **SUCCESS**
- DescribeQuorum: voters=[4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 3:A']
- Registers 3:A', **SUCCESS**
- DescribeQuorum: voters=[3:A', 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage)]
- Proceed with controller 4 rolls
- Similar process, B becomes B'
- DescribeQuorum: voters=[3:A', 4:B', 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage)]
- Proceed with controller 5 rolls
- Similar process, C becomes C'
- DescribeQuorum: voters=[3:A', 4:B', 5:C'], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage), 5:C (garbage)]

*Outcome:*
- All nodes in desired AND in voters with correct directory IDs, no action needed
- Status syncs from quorum: [{3:A'}, {4:B'}, {5:C'}]

**Scenario 4.B: Disk change, registration fails**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- Disk change on controller node pool
- Rolling update triggered

*Reconciliation (controller 3 rolls - first attempt):*
- Pod 3 restarts with new disk, new directory ID: A'
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']
- Roller delegates to the KRaft quorum reconciler for single-controller reconciliation
- Multiple incarnations detected (voter 3:A + observer 3:A')
- Reads meta.properties from pod 3: A'
- Unregisters 3:A, **SUCCESS**
- DescribeQuorum: voters=[4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 3:A']
- Tries to register 3:A', **FAILS**
- Roller stops, reconciliation fails

*Outcome (first attempt):*
- Status: [{3:A}, {4:B}, {5:C}] (old, not updated because reconciliation failed)
- DescribeQuorum: voters=[4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 3:A']

*Next reconciliation (full quorum reconciliation - automatic recovery):*
- Desired controllers: [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 3:A']
- Analysis: Node 3 in desired but NOT in voters, has multiple observers
- Full reconciliation for node 3:
  - Multiple observers detected (3:A and 3:A')
  - Read meta.properties from pod 3: A'
  - Find observer matching A', register 3:A', **SUCCESS** (recovery)
- Proceed with controller 4 rolls
- Similar process, B becomes B'
- DescribeQuorum: voters=[3:A', 4:B', 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage)]
- Proceed with controller 5 rolls
- Similar process, C becomes C'
- DescribeQuorum: voters=[3:A', 4:B', 5:C'], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage), 5:C (garbage)]

*Outcome (recovery):*
- DescribeQuorum: voters=[3:A', 4:B, 5:C], observers=[3:A (garbage)]
- Status syncs from quorum: [{3:A'}, {4:B}, {5:C}]

**Scenario 4.C: Disk change, unregistration fails**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- Disk change on controller node pool
- Rolling update triggered

*Reconciliation (controller 3 rolls - first attempt):*
- Pod 3 restarts with new disk, new directory ID: A'
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']
- Roller delegates to the KRaft quorum reconciler for single-controller reconciliation
- Multiple incarnations detected (voter 3:A + observer 3:A')
- Reads meta.properties from pod 3: A'
- Tries to unregister 3:A, **FAILS**
- Roller stops, reconciliation fails

*Outcome (first attempt):*
- Status: [{3:A}, {4:B}, {5:C}] (old, not updated because reconciliation failed)
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']

*Next reconciliation (full quorum reconciliation - automatic recovery):*
- Desired controllers: [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']
- Analysis: Node 3 in desired AND in voters, but also has an observer (disk change scenario)
- Full reconciliation for node 3:
  - Has voter and observer, potential stale voter
  - Read meta.properties from pod 3: A'
  - Voter 3:A but actual directory ID is A', unregister stale 3:A, **SUCCESS** (recovery)
  - DescribeQuorum: voters=[4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 3:A']
  - Find observer matching A', register 3:A', **SUCCESS**
- Proceed with controller 4 rolls
- Similar process, B becomes B'
- DescribeQuorum: voters=[3:A', 4:B', 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage)]
- Proceed with controller 5 rolls
- Similar process, C becomes C'
- DescribeQuorum: voters=[3:A', 4:B', 5:C'], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage), 5:C (garbage)]  

*Outcome (recovery):*
- DescribeQuorum: voters=[3:A', 4:B', 5:C'], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage), 5:C (garbage)]
- Status syncs from quorum: [{3:A'}, {4:B'}, {5:C'}]

**Scenario 4.D: Disk change, unregistration fails, operator crashes**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]

*User action:*
- Disk change on controller node pool
- Rolling update triggered

*Reconciliation (controller 3 rolls):*
- Pod 3 restarts with new disk, new directory ID: A'
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']
- Roller delegates to the KRaft quorum reconciler for single-controller reconciliation
- Tries to unregister 3:A, **FAILS**
- Operator **CRASHES**

*Outcome (with crash):*
- Status: [{3:A}, {4:B}, {5:C}] (old, not updated because operator crashed)
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']

*Next reconciliation (full quorum reconciliation - automatic recovery):*
- Desired controllers: [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A']
- Analysis: Node 3 in desired AND in voters, but also has an observer (disk change scenario)
- Full reconciliation for node 3:
  - Has voter and observer, potential stale voter
  - Read meta.properties from pod 3: A'
  - Voter 3:A but actual directory ID is A', unregister stale 3:A, **SUCCESS** (recovery)
  - DescribeQuorum: voters=[4:B, 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 3:A']
  - Find observer matching A', register 3:A', **SUCCESS**
- Proceed with controller 4 rolls
- Similar process, B becomes B'
- DescribeQuorum: voters=[3:A', 4:B', 5:C], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage)]
- Proceed with controller 5 rolls
- Similar process, C becomes C'
- DescribeQuorum: voters=[3:A', 4:B', 5:C'], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage), 5:C (garbage)]

*Outcome (recovery):*
- DescribeQuorum: voters=[3:A', 4:B', 5:C'], observers=[0:E, 1:F, 2:G, 3:A (garbage), 4:B (garbage), 5:C (garbage)]
- Status syncs from quorum: [{3:A'}, {4:B'}, {5:C'}] (status self-heals)

These scenarios demonstrate the two-phase execution (unregister before register) which prevents race conditions during disk changes, and how the full quorum reconciliation automatically recovers from failures during rolling updates by detecting directory ID mismatches and updating the quorum membership accordingly.

### Disaster recovery

Strimzi already provides a procedure for [recovering a deleted Kafka cluster](https://strimzi.io/docs/operators/latest/deploying#proc-cluster-recovery-volume-str).
It is related to the scenario where the PVs are still in place but all Kafka related custom resources together with the PVCs were deleted.
Because the PVs are still in place, all the metadata information is still available including the `meta.properties` file.

For clusters using dynamic quorum, the disaster recovery process requires additional steps to restore the controller directory IDs.
Similar to how the `clusterId` must be manually retrieved through the PVCs and added to the `Kafka` custom resource status, users must also retrieve and provide the controller directory IDs.

The recovery process for dynamic quorum clusters:

1. Manually inspect the PVCs to retrieve the `clusterId` from the `meta.properties` file on any node.
2. For each controller, retrieve the directory ID (UUID) from the `meta.properties` file located in the metadata log directory.
3. Edit the `Kafka` custom resource with the status field pre-populated with both the `clusterId` and the `controllers` list containing each controller's ID and directory ID.
4. The operator will use these directory IDs to build the `controllers` string for formatting, ensuring controllers start with their existing directory IDs preserved. The formatting won't happen because storages are already formatted.
5. Pods start and use the PVCs with matching directory IDs already present in the `meta.properties` files.
6. The quorum forms successfully with the preserved directory IDs, and the status is confirmed by syncing from the actual quorum state.

Without the pre-populated `controllers` status field, the operator would generate new random directory IDs, causing a mismatch with the actual directory IDs stored on the PVCs, preventing the cluster from starting correctly.

#### Disaster recovery scenario

**Scenario 5.A: Disaster recovery**

The following scenario demonstrates how to recover a dynamic quorum cluster when all custom resources have been deleted but PVCs remain.

*Initial state:*
- All Kafka custom resources deleted (Kafka, KafkaNodePool, etc.)
- PVCs remain with existing data
- Controllers were 3, 4, 5 with directory IDs A, B, C

*User action:*
- User inspects PVCs manually to retrieve metadata
- Finds controller directory IDs: 3:A, 4:B, 5:C (from `meta.properties` files)
- Finds cluster ID: "xyz-cluster-id" (similar to how clusterId is retrieved in existing disaster recovery)

*Recovery:*
- User edits the Kafka CR with status pre-populated:
  ```yaml
  apiVersion: kafka.strimzi.io/v1beta2
  kind: Kafka
  metadata:
    name: my-cluster
  spec:
    # ... normal spec with same configuration as before ...
  status:
    clusterId: "xyz-cluster-id"  # From manual PVC inspection
    controllers:
      - id: 3
        directoryId: "A"
      - id: 4
        directoryId: "B"
      - id: 5
        directoryId: "C"
  ```

*Reconciliation:*
- Operator reads status: [{3:A}, {4:B}, {5:C}]
- Builds controllers string: "3@host:9090:A,4@host:9090:B,5@host:9090:C"
- Passes to pods via ConfigMap
- Pods start and format with `-I` option using the controllers list
- PVCs already have matching directory IDs in their `meta.properties` files
- Storage formatting recognizes existing format and preserves it (due to `-g` flag)

*Outcome:*
- DescribeQuorum: voters=[3:A, 4:B, 5:C] observers=[0:E, 1:F, 2:G]
- Cluster successfully recovered with original directory IDs preserved
- Status syncs from quorum: [{3:A}, {4:B}, {5:C}] (confirmed)

This scenario demonstrates that the `controllers` status field serves a similar purpose to `clusterId` for disaster recovery, both requiring manual retrieval from PVCs and pre-population in the Kafka CR status to ensure successful cluster recovery.

**Scenario 5.B: Disaster recovery, single controller pod and storage forcefully deleted**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- All controllers running normally

*Disaster event:*
- Controller pod 5 and its PVC are forcefully deleted
- Pod 5 stays "Pending" until operator creates the new PVC
- New empty PVC is created for pod 5

*Reconciliation:*
- Builds controllers string: "3@host:9090:A,4@host:9090:B,5@host:9090:C"
- Passes to pods via ConfigMap
- Pods start and format with `-I` option using the controllers list, same directory ID C
- Pod 5 starts successfully with preserved directory ID C
- While pod 5 is starting/catching up, quorum operates with 2 voters (majority still available)
- DescribeQuorum (during startup): voters=[3:A, 4:B, 5:C] where 5:C is catching up, observers=[0:E, 1:F, 2:G]
- Controller 5 catches up with metadata log and rejoins quorum as active voter

*Outcome:*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- Status remains: [{3:A}, {4:B}, {5:C}]

This scenario works fine but the cluster can experience a brief period with reduced fault tolerance while controller 5 catches up.

**Scenario 5.C: Disaster recovery, two controller pods and storage forcefully deleted**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- All controllers running normally

*Disaster event:*
- Controller pods 4 and 5 with their PVCs are forcefully deleted
- Pods stay "Pending" until operator creates new PVCs
- New empty PVCs are created for pods 4 and 5

*Reconciliation:*
- Pods 4 and 5 start with new empty storage
- Builds controllers string: "3@host:9090:A,4@host:9090:B,5@host:9090:C"
- Passes to pods via ConfigMap
- Pods start and format with `-I` option using the controllers list, same directory IDs B and C
- Pods 4 and 5 start successfully with preserved directory IDs
- While pods 4 and 5 are starting/catching up, quorum operates with 1 voter (no majority, cluster unavailable)
- Controllers 4 and 5 catch up with metadata log and rejoin quorum as active voters
- Once majority restored (at least 2 voters active), quorum becomes operational again

*Outcome:*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- Status remains: [{3:A}, {4:B}, {5:C}]

This scenario works but the cluster experiences downtime while waiting for controllers 4 and 5 to catch up and restore quorum majority.
Once quorum is restored, cluster becomes available again.

**Scenario 5.D: Disaster recovery, all controller pods and storage forcefully deleted**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5]
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- All controllers running normally

*Disaster event:*
- All controller pods (3, 4, 5) and their PVCs are forcefully deleted, slightly apart from each other
- All pods stay "Pending" until operator creates new PVCs
- New empty PVCs are created for all controllers

*Reconciliation:*
- All pods 3, 4, 5 start with new empty storage
- Builds controllers string: "3@host:9090:A,4@host:9090:B,5@host:9090:C"
- Passes to pods via ConfigMap
- All format with `-I` option using the same directory IDs A, B, C from controllers string
- Pods start successfully with preserved directory IDs

*Outcome:*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- Status remains: [{3:A}, {4:B}, {5:C}]

This scenario is similar to create a new cluster because of the KRaft quorum starting from scratch.
The current brokers cannot connect anymore, because their metadata (coming from older KRaft quorum) are ahead of the new quorum, in terms of offset.
The only way to recover such a cluster, but with data loss, is manually deleting the metadata on the brokers.

**Scenario 5.E: Disaster recovery, whole controller node pool deleted and replaced with new node pool**

*Initial state:*
- Desired controllers (from spec): [3, 4, 5] from KafkaNodePool "controller"
- Status: [{3:A}, {4:B}, {5:C}]
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- All controllers running normally

*Disaster event:*
- User deletes entire KafkaNodePool "controller" (with pods 3, 4, 5)
- User creates new KafkaNodePool "controller-new" with 3 replicas at (roughly) the same time
- All original controller pods and PVCs are deleted
- New controller pods created with same IDs (3, 4, 5)

*Reconciliation:*
- Operator detects KafkaNodePool deletion and creation
- Status still has the same controller IDs [{3:A}, {4:B}, {5:C}] but references old node pool
- Builds controllers string: "3@host:9090:A,4@host:9090:B,5@host:9090:C"
- Passes to new pods via ConfigMap
- All three new controllers format with `-I` option, using same directory IDs A, B, C from the controllers list
- Pods start with the same directory IDs as the deleted controllers

*Outcome:*
- DescribeQuorum: voters=[3:A, 4:B, 5:C], observers=[0:E, 1:F, 2:G]
- Status remains: [{3:A}, {4:B}, {5:C}]

This scenario is similar to the previous one like creating a new cluster with a new KRaft quorum.
The current brokers cannot connect anymore, because their metadata (coming from older KRaft quorum) are ahead, in terms of offset, from the new quorum.
The only way to recover such a cluster, but with data loss, is manually deleting the metadata on the brokers.

### Migration from static to dynamic quorum

Migration from static to dynamic quorum is supported starting from Apache Kafka 4.1.0.

The migration procedure is available on the official Kafka documentation [here](https://kafka.apache.org/documentation/#kraft_upgrade) which describes the key steps:

* setting `kraft.version` to `1`.
* switching from `controller.quorum.voters` to `controller.quorum.bootstrap.servers` configuration.

The Strimzi Cluster Operator automatically migrates clusters from static to dynamic quorum.
The migration happens during normal reconciliation and it is transparent to users.

During each reconciliation cycle, the operator:

* Checks the quorum type by querying the `kraft.version` feature flag.
* If `kraft.version=1`, the cluster is already using dynamic quorum, so no action is taken.
* If `kraft.version=0`, the cluster is using static quorum, so the migration can proceed:
    * Uses the Kafka Admin API to set the `kraft.version` feature to `1`, enabling dynamic quorum support in Kafka.
    * Retrieves the controllers directory IDs by using Kafka Admin Client API `describeMetadataQuorum()` method, extracting `ReplicaState.replicaDirectoryId()` for each controller in the `voters` field.
    * Triggers the rolling update for all nodes, controllers and brokers, with the new `controller.quorum.bootstrap.servers` configuration.
    * Builds the list of `KafkaControllerStatus` objects and adds the `controllers` field to the Kafka CR status.

It's worth adding that, retrieving the controller directory IDs as described above should be done only after upgrading the `kraft.version` to `1`.
If this step is not done, the Kafka Admin Client API returns the `AAAAAAAAAAAAAAAAAAAAAA` value instead of the actual directory ID (from the `meta.properties` file).
It's the Base64 encoding of UUID all zeroes used within Apache Kafka when the voters are tracked within a static quorum.

#### Failure handling during migration

The migration process is designed to be resilient to failures.

If setting `kraft.version` to `1` fails for any reason, the entire reconciliation fails immediately.
No other migration steps are attempted and the cluster remains in static quorum mode with `kraft.version=0`.
On the next reconciliation cycle, the operator will retry the upgrade.

If the operator successfully sets `kraft.version` to `1` but crashes or it is restarted before completing the migration (before updating the `controllers` field in the status or triggering the rolling update), the migration completes automatically on the next reconciliation.
This is possible because `kraft.version=1` is already persisted in Kafka.
When the operator restarts, it queries `kraft.version`, detects it is already set to `1`, and the `KRaftQuorumReconciler` proceeds to retrieve the controllers directory IDs.
Also the nodes configuration change about using `controller.quorum.bootstrap.servers` will trigger the rolling of nodes and at the end, the status will be updated as well.
The idempotent nature of the KRaft Quorum Reconciler ensures that the migration completes safely regardless of when the operator restart occurred.

## Affected/not affected projects

Only the Strimzi Cluster Operator.

## Compatibility

When upgrading the Strimzi operator to a release supporting the dynamic quorum, already existing cluster using the static quorum will be automatically migrated to use dynamic quorum.
Of course, any new Apache Kafka cluster will be also deployed by using the dynamic quorum.

### Downgrade

When a cluster is automatically migrated from static to dynamic quorum, Apache Kafka doesn't support to go back to static anymore.
If the user tries to downgrade the `kraft.version` from `1` to `0`, they will get an error.

What is the impact of it on downgrading the operator to a previous release where there is no support for dynamic quorum?

When downgraded to an older release, the operator reconfigures the nodes with the old `controller.quorum.voters` parameter and roll them.
The `controllers` field within the `Kafka` status is also removed.
The `kraft.version` stays at `1` and, as already mentioned above, it cannot be set to `0` anymore (the old operator releases doesn't have the logic to do so anyway).
In this condition, the cluster is actually using dynamic quorum, despite the nodes being configured with the `controller.quorum.voters` parameter.
This is the way that Apache Kafka uses for backward compatibility.

From a Strimzi perspective, nothing can stop the user to increase the number of controller replicas to scale up the quorum.
The new controllers start without any issues but they won't be registered as voters automatically, because the old operator doesn't have such logic.
Potentially, the user could do it manually with the dedicated Kafka tools.
The same could apply to scale down the KRaft quorum, on metadata disk changes and so on.

The cluster is actually working with dynamic quorum enabled but all the automation provided by the new operator is not available.
From an operator perspective there is nothing that can be really done.

## Rejected alternatives

The following sections don't describe totally different rejected solutions for the overall dynamic quorum support.
They mostly describe rejected approaches to solve specific problems within the overall dynamic quorum and controllers scaling support.

### Universal formatting via `--initial-controllers` option

The official Kafka documentation describes the differentiation between formatting initial controllers in a new cluster with the `--initial-controllers` option, and formatting scaled-up controllers in an existing cluster with the `--no-initial-controllers` option.

An alternative approach would be to simplify this logic by always using the `--initial-controllers` option with a dynamically generated controllers list.
On each reconciliation, the operator would build a complete list of all "desired" controllers (both existing and newly added) and use this list when formatting any controller's storage with the `--initial-controllers` option.
This would eliminate the need to differentiate between new cluster creation and scale-up scenarios in the formatting logic.

After investigation, this approach appears to work functionally.
A new controller formatted with `--initial-controllers` containing all current controllers can still join the quorum as an observer and be registered as a voter.
However, several issues make this approach unsuitable:

- Unnecessary checkpoint file: Formatting with `--initial-controllers` creates a bootstrap snapshot/checkpoint file on the scaled-up controller's disk. This file is redundant because new controllers don't use it to discover the quorum but they fetch the `VotersRecord` from the leader's metadata log instead. This can be considered a minor issue since the file is quite small and doesn't have any impact.
- Undocumented behavior: Most critically, this approach is not documented in the official Apache Kafka documentation or KIP-853. Relying on undocumented behavior creates a risk that future Kafka versions could change or break this functionality without notice.

For these reasons, the proposal follows the official documented approach: using `--initial-controllers` only for initial cluster bootstrap and `--no-initial-controllers` for scale-up operations.

### Manual controllers registration and unregistration

When scaling up controllers, instead of setting up a complex monitoring to check that the new controller caught up with the active ones or waiting for Apache Kafka 4.2.0, we could leave the user to register the controller manually.
The user could monitoring the lag of the newly added controller via the `kafka-metadata-quorum.sh` tool and wait for it to catch up.
Finally, "promoting" the controller from being an observer to join the quorum and become a voter, by manually running the following command on the controller pod itself:

```shell
bin/kafka-metadata-quorum.sh --command-config /tmp/strimzi.properties --bootstrap-server my-cluster-kafka-bootstrap:9092 add-controller
```

Then checking that the controller was "promoted" to voter by using the following command:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
```

When scaling down controllers, after having them removed (pod killed) by the Strimzi Cluster Operator, the user could use the `kafka-metadata-quorum.sh` to run the corresponding unregistration.
It would be making the controller to come back being an observer (not voter anymore) after scaling down with the following command to execute on any node:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 remove-controller --controller-id <id> --controller-directory-id <directory-id>
```

with `<id>` as the controller node id and the `<directory-id>` to be retrieved from the controller node in the `meta.properties` file.

After scaling down the controller, it will be still showed as observer for some time until a voters cache is refreshed.
Then checking that the controller was removed from the observers list by using the following command:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
```

This process is not user friendly and really error prone due to the users running several manual commands.
This solution was rejected because it's against the automation we want within the Strimzi Cluster operator.

### Using auto-join feature available with Apache Kafka 4.2

The [Automatic joining controllers](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes#KIP853:KRaftControllerMembershipChanges-Automaticjoiningcontrollers) feature, which is part of the KIP-853, is going to be available with Apache Kafka 4.2.0.
With auto-join enabled, a newly added controller starts initially as an observer and then it automatically joins the KRaft quorum by becoming a voter, when it's caught up with the quorum itself.
Apache Kafka handles the burden of checking that the new controller is in sync (or not) with the other voters.
Of course, it also means that the support for dynamic quorum and corresponding controllers scaling is available for an Apache Kafka cluster having 4.2.0 as the minimum version and such a check would be needed within the Strimzi Cluster Operator logic.

Leveraging this feature was explored but the current semantic of auto-join doesn't work pretty well when scaling down a controller in a Kubernetes-based environement.
The above KIP states that, in order to unregister a controller when auto-join is enabled, you have to:

* shutdown the controller.
* run the unregistration command.

If you run the unregistration command as the first step while the controller is still running, it will join the quorum immediately because of the auto-join.

It doesn't work well in a scenario where the Apache Kafka cluster has combined-nodes and we want to scale down controllers by removing the "controller" role from one or more of them.
In this case, the node is not actually shut down and removed forever but it's rolled with a new configuration (i.e. the "controller" role is removed).
But in a Kubernetes-based environment where the rolling of a pod is driven by the platform, there is no opportunity to execute a step (like the controller unregistration) in between the shut down and the start up.
To overcome this issue, the best approach would be unregister the controller first and then rolling the node as broker only, but as mentioned before it would join again right after the unregistration.

Changing the auto-join UX was subject of a long discussion in the Apache Kafka community and it could be changed in the future with a new KIP.
A few suggestions were proposed by the community like:

* avoiding that a node can auto-join after removal before a predefined timeout.
* allowing a node to auto-join only on start up.

But all the above presented issues which need more discussion.
For more details, look at the corresponding [KAFKA-19850](https://issues.apache.org/jira/browse/KAFKA-19850) issue as well as the corresponding PRs [#20859](https://github.com/apache/kafka/pull/20859) and [#21025](https://github.com/apache/kafka/pull/21025) where all the discussions happened.


### Manual migration from static to dynamic quorum

Running the migration from static to dynamic quorum could be also done with a manual approach with the following steps:

* Set the `kraft.version` feature to `1`, by running the following command on any node from the cluster:

```shell
bin/kafka-features.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 upgrade --feature kraft.version=1
```

* Then retrieve the controllers directory IDs by using the `kafka-metadata-quorum.sh` tool:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --replication
NodeId	DirectoryId           	LogEndOffset	Lag	LastFetchTimestamp	LastCaughtUpTimestamp	Status  	
3     	U3fHvCoMVWiCVYa2ri_K5w	875         	0  	1760635202277     	1760635202277        	Leader  	
4     	g3OMYG2gvmLCeE9Nv-Cz5Q	875         	0  	1760635201883     	1760635201883        	Follower	
5     	2K7pPIanujBKY1Tsxr-gWg	875         	0  	1760635201882     	1760635201882        	Follower	
0     	O4DOa5i6JbE-tKXvTnU9rA	875         	0  	1760635201884     	1760635201884        	Observer	
1     	qSXp8TpLVgJY11tWMZGsSw	875         	0  	1760635201884     	1760635201884        	Observer	
2     	ZGVp_10gzC6Pe5Its9-LFg	875         	0  	1760635201883     	1760635201883        	Observer
```

* Finally, patch the corresponding `Kafka` status by adding the `controllers` field:

```shell
kubectl patch kafka my-cluster -n myproject --type=merge --subresource=status -p '{"status":{"controllers":[{"id":3,"directoryId":"U3fHvCoMVWiCVYa2ri_K5w"},{"id":4,"directoryId":"g3OMYG2gvmLCeE9Nv-Cz5Q"},{"id":5,"directoryId":"2K7pPIanujBKY1Tsxr-gWg"}]}}'
```

Patching the `Kafka` custom resource status will trigger nodes rolling and the operator will reconfigure them with the `controller.quorum.bootstrap.servers` field for using the dynamic quorum.

This process is not user friendly and really error prone due to the users running several manual commands.
This solution was rejected because it's against the automation we want within the Strimzi Cluster operator.

### Semi-automated migration from static to dynamic quorum

In this approach, the operator detects static quorum (`kraft.version=0`) but does not automatically migrate.
The operator supports reconciling both static and dynamic quorum based clusters.
The user triggers the migration by annotating the `Kafka` custom resource with `strimzi.io/kraft-quorum-migration: true`.
At this point, the operator follows the same steps which are described in the proposed solution with the automatic migration.
This approach was rejected because we don't want to support both static and dynamic quorum and we should encourage users to have dynamic quorum by default to enable controllers scaling.
Furthermore, given the main goal of the Strimzi Cluster operator regarding the process automation of managing an Apache Kafka cluster, running the migration automatically makes more sense and copes with such a goal.
This way, the users don't need to deal with an additional migration process like it was for the KRaft based cluster in the past.