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

The Strimzi Cluster Operator will be able to create the new controllers from scratch (with the updated number of replicas) and it will restart the brokers with the new controller quorum configured (via the `controller.quorum.voters` property).
Of course, right after the controllers deletion step and until the controller quorum runs again, the cluster won't be available because brokers cannot work without the KRaft controllers running.

This is not parity features with ZooKeeper-based cluster where the ZooKeeper ensemble can be scaled up or down without cluster downtime.

## Motivation

KRaft dynamic controller quorum allows to enable the controllers scaling without downtime.
In general this operation can be useful for increasing the number of controllers when needed (or, of course, decreasing them).
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
* Add support for controllers scaling when the dynamic controller quorum is used.
* Add migration from static to dynamic controller quorum.

It also take into account the main operational changes when moving from static to dynamic quorum:

* Node storage formatting is impacted by this change.
* The `controller.quorum.voters` property is now replaced by the `controller.quorum.bootstrap.servers` property on both broker and controller nodes.
* The `kraft.version` feature is now `1` when using dynamic quorum. It's `0` when using static quorum. 

By the way, regarding the last point, there is no need to set the `kraft.version` feature upfront (to `0` or `1`).
It's being set when the controller storage is properly formatted and the quorum is configured as dynamic or static.
The only case needing the feature to be changed is when migrating from static to dynamic quorum (so updating the value from `0` to `1`).

The following sections describes what's the idea behind the proposal and how to approach the above steps.
As a referece, [here](http://kafka.apache.org/41/operations/kraft/#provisioning-nodes)'s the link to the official Apache Kafka documentation about using dynamic quorum.

#### Understanding Static vs Dynamic Quorum

The main difference between static and dynamic quorum lies in where the voter set membership information is stored and how it can be changed.

With **static quorum** (`kraft.version=0`), the quorum membership is hardcoded in the node configuratrion file by using the `controller.quorum.voters` property listing all the controllers which take part at the KRaft quorum.
This membership configuration never changes during the cluster's lifetime and exists only in each node's configuration files.
The voter set is immutable and there is no `VotersRecord` stored in the metadata log.
When controllers start up, they read the voter set directly from their local configuration and the leader discovery/election can start immediately.

With **dynamic quorum** (`kraft.version=1`), the quorum membership is stored in the replicated metadata log itself by using a `VotersRecord`.
The node configuration file uses the `controller.quorum.bootstrap.servers` property instead, which only provides contact points for initial discovery.
The voter set is mutable and can change through Raft consensus operations.
When controllers start up, they discover the voter set by reading the `VotersRecord` from either the bootstrap snapshot (for initial controllers) or by replicating it from the leader (for newly added controllers).

The key differences can be summarized as:

| Aspect | Static Quorum | Dynamic Quorum |
|--------|---------------|----------------|
| Voter set location | Configuration files | Replicated metadata log |
| Configuration property | `controller.quorum.voters` | `controller.quorum.bootstrap.servers` |
| Persistence | Each node's local config | Raft-replicated snapshot/log |
| Mutability | Immutable | Mutable via Raft consensus |
| Formatting requirement | Just cluster ID and config | Bootstrap snapshot with `VotersRecord` required |
| Membership changes | Requires downtime and reconfiguration | Dynamic add/remove operations |

This difference in where membership is stored has significant implications for storage formatting.
In dynamic quorum, membership is mutable state that must be replicated through Raft, which means it must exist in the replicated log from the beginning.
This is why initial controllers must be formatted with a bootstrap snapshot containing the `VotersRecord`, while newly added controllers discover membership by replicating from the existing quorum.
They use the `controller.quorum.bootstrap.servers` configuration to contact an existing controller from where fetching the metadata log containing the `VotersRecord` for the voter set.
The `VotersRecord` contains critical information including voter IDs, directory IDs (unique UUIDs), endpoints, and supported `kraft.version` ranges.

Without the bootstrap snapshot containing the `VotersRecord`, controllers face a kind of deadlock: they cannot participate in elections or become candidates unless they know who the voters are, but the voter set information comes from the `VotersRecord` which must be written to the replicated log by a leader.
However, no leader can exist without successful elections, and elections cannot happen without controllers knowing the voter set.
This circular dependency means that if all controllers start without a pre-written `VotersRecord` in their bootstrap snapshot, they would all have an empty voter set, preventing any of them from becoming candidates or holding elections, leaving the cluster unable to bootstrap.
The only way to break this deadlock is to pre-write the `VotersRecord` into the bootstrap snapshot during initial formatting using the `--initial-controllers` parameter, ensuring all initial controllers know the voter set from the beginning before the cluster even starts.

In conclusion, for the dynamic quorum to work correctly, all nodes must have a consistent view of the initial voter set from the replicated log, not from configuration files.
This ensures consistency and allows the quorum to evolve dynamically while maintaining the strong consistency guarantees provided by the Raft consensus protocol.

An alternative approach to bootstrapping a dynamic quorum cluster is to bootstrap a standalone controller first.
It means that, instead of formatting multiple controllers with the full initial controllers list, only a single controller is formatted and started initially using the `--standalone` flag.
This creates a quorum of one without requiring the initial controllers list but still writing a bootstrap snapshot file with a corresponding `VotersRecord`.
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

With the `STRIMZI_CLUSTER_ID` and `METADATA_VERSION` variables loaded from the corresponding fields within the ConfigMap which is generated for each node by the Strimzi Cluster Operator and mounted on the pod volume.

This works when the Apache Kafka cluster is using the static quorum during a new cluster creation, when all nodes need to be formatted from scratch at the same time, as well as on cluster scaling, when a new broker node is added but the formatting is done the same way (of course controllers scaling is not supported).

The dynamic quorum works differently from the formatting perspective and it makes a clear distintion on the `kafka-storage.sh` usage when it's a new cluster creation or a node addition (both broker or controller).
When the cluster is created, each broker can be formatted the same way we do today but the controller formatting needs the new `-I` (or `--initial-controllers`) option which specifies the initial controllers list.
The initial controllers list is a comma separated list of controllers, as `node-id@dns-name:port:directory-id`, which contains the initial controllers building the KRaft quorum in the new cluster we are going to create.

The formatting tool is then used the following way:

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -I="$INITIAL_CONTROLLERS"
```

The issue on cluster creation is about differentiating between broker and controller to use the right options for the formatting.
The broker storage formatting doesn't use the `-I` option.

But when it comes to scaling up, so adding a new broker or a controller on an existing cluster, in both cases the formatting tool needs the `-N` (or `--no-initial-controllers`) option instead, which replace the `-I` in case of a controller.

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -N
```

So it means that when using the dynamic quorum, the script executed on node startup needs a way to differentiate between a new cluster creation or a node scaling in order to run the formatting with the appropriate options.

The above scenarios can be summarized this way:

* Static quorum:
  * Broker and controllers are formatted with no additional new options.
* Dynamic quorum:
  * New cluster creation:
    * broker doesn't need any additional options (but it works well with `-N` as well to simplify the proposal, see later).
    * controller needs the initial controllers list via `-I`.
  * Existing cluster, scale up:
    * broker and controller needs the `-N` option.

The proposed idea is about having the Strimzi Cluster Operator generating the initial controllers list, which includes the directory IDs, that can be passed through the mounted ConfigMap, via a new `initial.controllers` field, to the run script to be used during the formatting, via a corresponding `INITIAL_CONTROLLERS` variable.
The initial controller list would be used as a differentiator between using static quorum (the list is empty) or dynamic quorum (the list is not empty).
The script can also get the role of the current node, broker and/or controller, by reading it from the `process.roles` field within the `/tmp/strimzi.properties` file.

The initial controllers list is also stored in a new `initialControllers` field within the `Kafka` custom resource `status` which is set on cluster creation with dynamic quorum.
It will store the expected string for formatting the controllers in a newly created cluster:

```yaml
# ...
status:
    # ...
    initialControllers: 3@my-cluster-controller-3.my-cluster-kafka-brokers.myproject.svc.cluster.local:9090:r9VGTiw2QUCoDUadCt6q2g,4@my-cluster-controller-4.my-cluster-kafka-brokers.myproject.svc.cluster.local:9090:r8E7BKRbR0SQxSIFro6wjg,5@my-cluster-controller-5.my-cluster-kafka-brokers.myproject.svc.cluster.local:9090:Anjr8banTey_LqVZsppzYg
    # ...
```

Of course, this field is not set when the cluster uses static quorum instead.
Having this field set (or not) is also a way to distinguish the reconciliation of an existing Kafka cluster using the new dynamic quorum (the field is set) or still using the static quorum (the field is missing).
This way the Strimzi Cluster Operator, together with the run script on the node, can handle both static and dynamic quorum based clusters and proper node formatting.

In a sort of metalanguage we could state:

```shell
quorum := Kafka.status.initialControllers IS NOT EMPTY ? "dynamic" : "static"
```

It's worth noticing that this field is going to be created on the cluster creation (with dynamic quorum) but not going to change for the entire life of the cluster.
It's a no goal for this proposal to remove the static quorum support.
We could consider to remove the above new status field if we'll consider to remove support for static quorum in the future.

More details about the initial controllers list creation in the following sections.

### New cluster creation and reconciliation with dynamic quorum

The initial cluster creation is going to use the [bootstrap with multiple controllers](https://kafka.apache.org/41/operations/kraft/#bootstrap-with-multiple-controllers) approach.
The Strimzi Cluster Operator:

* builds the broker and controller configuration by setting the `controller.quorum.bootstrap.servers` field (instead of the `controller.quorum.voters` one).
* generates a random directory ID, as a UUID (like the cluster ID), for each controller.
* builds the initial controllers list, as `initial.controllers` field within the node ConfigMap, to be loaded by the `kafka_run.sh` script where it's needed for formatting the storage properly.
* saves the `initialControllers` field within the `Kafka` custom resource status by storing the expected string representing the initial controllers list.

On node start up, within the `kafka_run.sh` script, the `initial.controllers` field is loaded from the node ConfigMap into a `INITIAL_CONTROLLERS` variable:

* if the variable is empty, the cluster is static quorum based, so using the usual formatting as today.
* if the variable is not empty, the cluster is dynamic quorum based, and the script gets the node role into a `PROCESS_ROLES` variable.
  * if it's a broker, formatting with the `-N` option (which works for both new cluster creation or brokers scale up).
  * if it's a controller, the script check if it's part of the initial controllers list or not:
    * part of the initial controller list, formatting with the `-I` option, because it could be a new cluster but if it's just the controller rolling for any other reason, the formatting error will be ignored as today (see `-g` option).
    * not part of the initial controller list, it could be a controller scale up so formatting with the `-N` option but if it's just the controller rolling for any other reason, the formatting error will be ignored as today (see `-g` option).

Of course when we have a combined node, because the `process.roles` includes being a controller, the run script follows the corresponding controller path for formatting.

The reconciliation of a dynamic quorum based cluster will go through the same path, by detecting that an existing cluster is using dynamic quorum because the `initialControllers` status field is present within the `Kafka` custom resources.

### Reconcile an existing cluster with static quorum

When the Strimzi Cluster Operator is upgraded with the latest version supporting dynamic quorum, it's going to reconcile existing `Kafka` custom resources for clusters which are using the static quorum.
The operator detects it's not the creation of a new cluster and also that the `initialControllers` status field is not present within the `Kafka` custom resources.
This is a way to understand that the existing cluster is not new and it's using the static quorum.
In such a case the reconciliation proceed as usual:

* builds the broker and controller configuration by setting the `controller.quorum.voters` field.
* doesn't build the `initialControllers` field for the node ConfigMap (and so doesn't save it into the `Kafka` status).

It means that there is no automatic migration from static to dynamic quorum.
The Strimzi Cluster Operator will be able to detect which type of quorum (static vs dynamic) an existing cluster is using without modifying it but just going through the proper reconciliation process.
Users who wish to migrate to the dynamic quorum can trigger the process using the annotation-based approach described in the ["Migration from static to dynamic quorum"](#migration-from-static-to-dynamic-quorum) section.

### Scaling controllers: (Un)registration within the reconciliation process

Scaling controllers up or down within a KRaft quorum involves more than just spinning up new nodes or shutting down existing ones.
When a new controller is added to the Apache Kafka cluster, it has to be registered in order to join the KRaft quorum.
When a running controller is shut down, it has to be unregistered in order to leave the KRaft quorum.

As one of the operations at the beginning of the `KafkaReconciler` reconciliation process, but before handling the scale down, the Strimzi Cluster Operator:

* gets the `QuorumInfo` by using the Kafka Admin API `describeMetadataQuorum` method in order to retrieve the current Quorum status within the cluster along with its corresponding voters (and observers).
* compares the "desired" controllers with voters and observers and identifies the controllers that need to be unregistered. They are in the voters but not "desired" controllers as result of a scale down operation.

As one of the operations at the end of the `KafkaReconciler` reconciliation process, but after handling the scale up, the Strimzi Cluster Operator: 

* gets the `QuorumInfo` by using the Kafka Admin API `describeMetadataQuorum` method in order to retrieve the current Quorum status within the cluster along with its corresponding voters (and observers).
* compares the "desired" controllers with voters and observers and identifies the controllers that need to be registered. They are "desired" controllers that are part of the observers but not yet in voters as a result of a scale up operation.

In both cases, when the controllers to be registered and unregistered are identified, the Kafka Admin API, `addRaftVoter` and `removeRaftVoter`, are used in order to register and unregister such controllers.

This approach is similar to what is already used today for brokers unregistration which compares, at the end of each reconciliation, the list of registered brokers retrieved via the Kafka Admin API with the "desired" brokers running in the cluster, in order to determine which ones are missing and need to be unregistered.
In the controllers scaling scenario, the operator handles the registration as well (not only the unregistration).

It also works in case the operator crashes, for example between scaling up controllers and registering them.
On start up, by listing voters and checking the "desired" controllers, it's able to run the registering operation.

In addition to handling the registration and unregistration of controllers, the Strimzi Cluster Operator has to check the health of the KRaft quorum before allowing the scale down of the controllers.
At the beginning of the `KafkaReconciler` reconciliation process, if scaling down controllers could brake the quorum, it should be blocked and reverted back by the operator.
A similar approach is already in place when attempting to scale down brokers that are hosting partitions, which could break HA.

The above mentioned operations would fit into the `KafkaReconciler` reconciliation process in this way:

* BEGIN RECONCILIATION
  * ... (other reconcile operations) ...
  * check the KRaft quorum health, revert back any attempt of scaling down controllers which could break the KRaft quorum.
  * retrieve the current KRaft quorum voters from the cluster, compare with "desired" controllers, unregister the ones which are going to be scaled down.
  * scale down controllers.
  * ... (other reconcile operations) ...
  * scale up new controllers.
  * retrieve the current KRaft quorum voters from the cluster, compare with "desired" controllers, register the new ones.
  * ... (other reconcile operations) ...
* END RECONCILIATION

The following sections provide additional details about how the controllers registration and unregistration is executed.

#### Adding new controllers (scale up)

When the controllers are scaled up, the Strimzi Cluster Operator:

* builds the new controllers configuration by setting the `controller.quorum.bootstrap.servers` field.
* starts up all the newly added controllers in parallel (but not running their registration explicitly when they are ready).

On node start up, within the `kafka_run.sh` script:

* the controller node is formatted by using the `-N` option.

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

The suggestion here is to build the controller endpoint within the operator logic as it's already done withing the `KafkaBrokerConfigurationBuilder`, because in the end it's the operator itself configuring the new controller so creating the advertised listeners list.

Within the same reconciliation, multiple controllers can be scaled up simultaneously.
When ready, these new controllers start as observers and begin catching up with the metadata from the existing quorum voters, but they don't join the quorum yet.
The operator then runs the registrations sequentially one by one, meaning it can only issue a new `addRaftVoter` request after the previous one completes.
Apache Kafka's internal logic only allows controllers to be promoted to voters once they are fully synchronized with the quorum.
When Apache Kafka receives a registration request, it writes a `VotersRecord` to the metadata log, and the controllers handle the quorum membership change.
This change can take time because the controller needs to catch up, and if the operator issues another `addRaftVoter` request within the same reconciliation, before the previous registration request completes, it will be refused.
In such cases, the operator stops further registrations, allowing the reconciliation to end, and the remaining registrations will proceed in subsequent reconciliations.
This means that while all controllers start up in a single reconciliation, registering them all as voters may require multiple reconciliation cycles.

Furthermore, monitoring that the controller has caught up with the active ones is not necessary.
If the controller hasn't caught up yet and the operator runs the registration, the active controller returns an error but the reconciliation doesn't fail and the registration can be re-tried on the next one.
The same happens if getting the quorum information or registering the controllers fails (the Apache Kafka returns an error), the reconciliation doesn't fail but just continues and the operator will re-try the operation on the next reconciliation.

Scaling up the KRaft quorum can be done in the following ways:

* by increasing the number of replicas for an existing `KafkaNodePool` with the `controller` role (both combined or dedicated nodes).
* by adding a new `KafkaNodePool` with `controller` role.
* by adding the `controller` role to an existing `KafkaNodePool` hosting `broker`-only nodes.

#### Removing controllers (scale down)

When the controllers are scaled down, the Strimzi Cluster Operator:

* removes/unregisters the controllers from the quorum by using the `removeRaftVoter` method in the Kafka Admin API.
* scales down the controllers by deleting the corresponding pods.

If getting the quorum information or unregistering the controllers fails (the Apache Kafka returns an error), the reconciliation will fail to avoid the controllers, not unregistered correctly, to be shutten down.
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

### Migration from static to dynamic quorum

Migration from static to dynamic quorum is supported starting from Apache Kafka 4.1.0.

The migration procedure is available on the official Kafka documentation [here](https://kafka.apache.org/documentation/#kraft_upgrade) and it covers the following main steps:

* the `kraft.version` has to be set to `1` (greater than `0` anyway).
* the `controller.quorum.bootstrap.servers` field should be used instead of the `controller.quorum.voters`.

The Strimzi Cluster Operator should be also able to build the `initialControllers` field:

* by reading the controller IDs from the `controller.quorum.voters` (they are going to be the initial controllers).
* getting the corresponding directory IDs by using Kafka Admin Client API `describeMetadataQuorum` method, and extract from the `QuorumInfo` (the `ReplicaState.replicaDirectoryId()` for each node within the `voters` field).

The `initialControllers` list is then saved within the `Kafka` custom resource status.
This will trigger nodes rolling and the operator will reconfigure them with the `controller.quorum.bootstrap.servers` field for using the dynamic quorum.

It's worth adding that, retrieving the controller directory IDs as described above should be done only after upgrading the `kraft.version` to `1`.
If this step is not done, the Kafka Admin Client API returns the `AAAAAAAAAAAAAAAAAAAAAA` value instead of the actual directory ID (from the `meta.properties` file).
It's the Base64 encoding of UUID all zeroes used within Apache Kafka when the voters are tracked within a static quorum.

Taking into account what explained above, the migration process could be automated the following way:

* the user applies a `strimzi.io/kraft-quorum-migration: true` annotation on the `Kafka` custom resource of the cluster to be migrated.
* if the cluster is already using dynamic quorum (the `initialControllers` field is available in the `Kafka` custom resource status), the annotation is ignored (it is up to the user to remove it).
* if the cluster is using static quorum, the cluster operator:
    * uses the Kafka Admin API to set the `kraft.version` feature to `1`.
    * gets the controllers directory IDs by using Kafka Admin Client API `describeMetadataQuorum` method, and extract from the `QuorumInfo` (the `ReplicaState.replicaDirectoryId()` for each node within the `voters` field).
    * builds the initial controllers list and patch the `Kafka` status by adding the `initialControllers` field.
    * remove the annotation.

## Affected/not affected projects

Only the Strimzi Cluster Operator.

## Compatibility

When upgrading the Strimzi operator to a release supporting the dynamic quorum, already existing cluster using the static quorum will be reconciled as usual with no changes or distruption.
No automatic migration to dynamic quorum is going to happen.
The user will run the migration whenever they want with the described procedure.
Any new Apache Kafka cluster will be deployed by using the dynamic quorum instead.

## Rejected alternatives

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

* Finally, patch the corresponding `Kafka` status by adding the `initialControllers` field:

```shell
kubectl patch kafka my-cluster -n myproject --type=merge --subresource=status -p '{"status":{"initialControllers":"3@my-cluster-controller-3.my-cluster-kafka-brokers.myproject.svc.cluster.local:9090:r9VGTiw2QUCoDUadCt6q2g,4@my-cluster-controller-4.my-cluster-kafka-brokers.myproject.svc.cluster.local:9090:r8E7BKRbR0SQxSIFro6wjg,5@my-cluster-controller-5.my-cluster-kafka-brokers.myproject.svc.cluster.local:9090:Anjr8banTey_LqVZsppzYg"}}'
```

Patching the `Kafka` custom resource status will trigger nodes rolling and the operator will reconfigure them with the `controller.quorum.bootstrap.servers` field for using the dynamic quorum.

### Fully automated migration from static to dynamic quorum

The operator recognizes that the reconciled `Kafka` custom resource is related to a cluster still using static quorum (the `initialControllers` list in the status is not set) and run the migration to dynamic quorum automatically (as described in the dedicated section) without the need for the user to trigger it by applying a dedicated annotation.
This approach was rejected because the check would run on every reconciliation instead of being triggered when needed.
It would also be better giving the user the power to make the decision when the migration is desired instead of having all their clusters being updated right after the upgrade to the new operator.
