# Kafka Roller 2.0

## Current situation

The Kafka Roller is a Cluster Operator component that's responsible for rolling Kafka pods when:
- non-dynamic reconfigurations needs to be applied
- update in Kafka CRD is detected
- a certificate is renewed
- pods have been manually annotated by the user for controlled restarts
- pod is stuck and is out of date
- Kafka broker is unresponsive to Kafka Admin connections

These are not the exhaustive list of possible triggers for rolling Kafka pods, but the main ones to highlight.

A pod is considered stuck if it is in one of following states:
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `ContainerCreating`
- `Pending` and `Unschedulable`

### Known Issues

The existing KafkaRoller has been suffering from the following shortcomings:
- While it is safe and simple to restart one broker at a time, it is slow in large clusters.
- It doesn’t worry about partition preferred leadership
- Hard to reason about when things go wrong. The code is complex to understand and it's not easy to determine why a pod was restarted from logs that tend to be noisy.
- Potential race condition between Cruise Control rebalance and KafkaRoller that could cause partitions under minimum in sync replica. This issue is described in more detail in the `Future Improvements` section.
- The current code for KafkaRoller does not easily allow growth and adding new functionality due to its complexity.


The following non-trivial fixes and changes are missing from the current KafkaRoller's KRaft implementation:
- Currently KafkaRoller has to connect to brokers successfully in order to get KRaft quorum information and determine whether a controller node can be restarted. This is because it was not possible to directly talk to KRaft controllers at the time before [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration) was implemented.

- KafkaRoller takes a long time to reconcile combined nodes if they are all in `Pending` state. This is because the combined node does not become ready until the quorum is formed and KafkaRoller waits for a pod to become ready before it attempts to restart other nodes. In order for the quorum to form, at least the majority of controller nodes need to be running at the same time. This is not easy to solve in the current KafkaRoller without introducing some major changes because it processes each node individually and there is no mechanism to restart multiple nodes in parallel. More information can be found [here](https://github.com/strimzi/strimzi-kafka-operator/issues/9426).

- The quorum health check is based on the `controller.quorum.fetch.timeout.ms` configuration which it reads from the desired configurations passed from the Reconciler. However, `CAReconcilor` and manual rolling update pass null value for desired configurations because in both cases, the nodes don't need reconfigurations. This results in performing the quorum healthcheck based on the hard-coded default value of `controller.quorum.fetch.timeout.ms` rather than the accurate configuration value when doing manual rolling update and rolling nodes for certificate renewal.

- The current roller's quorum health will be broken once we have scaling supported via [KIP-853](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes). KafkaRoller relies on the response from `describeQuorum` API to get the total number of configured controllers. During scale down, the nodes could return inconsistent number of controllers in their responses until all nodes are updated with the correct configuration. This could result in quorum healthcheck not passing therefore not able to restart nodes.

- KafkaRoller cannot transition nodes from controller only role to combined role. This is because the KRaft controller identifies the role of the node by `NodeRef` object which contains the desired role for the node rather than the currently assigned role. The current roller would have to be updated with a new mechanism to get the currently assigned role. More information can be found here [here](https://github.com/strimzi/strimzi-kafka-operator/issues/9434).


## Motivation

Strimzi users have been reporting some of the issues mentioned above and would benefit from a new KafkaRoller that is designed to address the shortcomings of the current KafkaRoller.

The current KafkaRoller has complex and nested conditions therefore makes it challenging for users to debug and understand actions taken on their brokers when things go wrong and configure it correctly for their use cases. A new KafkaRoller that is redesigned to be simpler would help users to easily understand the code and configure it to their needs.

As you can see above, the current KafkaRoller still needs various changes and potentially more as we get more experience with KRaft and discover more issues. Adding these non trivial changes to a component that is very complex and hard to reason, is expensive and poses potential risks of introducing bugs because of tightly coupled logics andlack of testability.

## Proposal

The objective of this proposal is to introduce a new KafkaRoller with simplified logic therefore more testable, and has structured design resembling a finite state machine. KafkaRoller desisions can become more accurate and better informed by observations coming from different sources (e.g. Kubernetes API, KafkaAgent, Kafka Admin API). These sources will be abstracted so that KafkaRoller is not dependent on their specifics as long as it's getting the information it needs. This will enable the KafkaRoller to run even if the underlying platform is different, for example, not Kubernetes. 

Depending on the observed states, the roller will perform specific actions, causing each node's state to transition to another state based on the corresponding action. This iterative process continues until each node's state aligns with the desired state.

It will also introduce an algorithm that can restart brokers in parallel while applying safety conditions that can guarantee Kafka producer availability and causing minimal impact on controllers and overall stability of clusters.

0. The following can be the configured for the new KafkaRoller:

| Configuration | Default value | Exposed to user | Description |
| :-------------| :-------------| :---------------| :---------- |
| maxRestartAttempts | 3 | No | The maximum number of times we attempt to restart a broker before failing the reconciliation. This is checked against `numRestarts` in the `ServerContext`.|
| maxReconfigAttempts | 3 | No | The maximum number of times we attempt to reconfigure a broker before restarting it. |
| postRestartTimeoutMs | 60 seconds | Yes | The maximum amount of time we will wait for brokers to transition to `SERVING` state after a restart. This will be based on the operational timeout that is already exposed to the user. |
| postReconfigureTimeoutMs | 60 seconds | Yes | The maximum amount of time we will wait for brokers to transition to `SERVING` state after a reconfiguration.
| maxBatchSize | 1 | Yes | The maximum number of broker nodes that can be restarted in parallel. |

1. When a new reconciliation starts up, `ServerContext` is created for each broker.
 ```
 ServerContext {
   nodeId: int
   nodeRole: String
   state: ServerState
   reason: String
   numRestarts: int
   lastTransitionTime: long
 }
 ```

 - <i>nodeId</i>: Node ID.
 - <i>nodeRoles</i>: Process roles of this node (e.g. controller, broker). This will be set using the pod labels `strimzi.io/controller-role` and `strimzi.io/broker-role` because these are currently assigned roles of the node.
 - <i>state</i>: It contains the current state of the node based on information collected from the abstracted sources. The table below describes the possible states.
 - <i>reason</i>: It is updated based on the current predicate logic from the Reconciler. For example, an update in the Kafka CR is detected.
 - <i>numRestarts</i>: The value is incremented each time the node has been attempted to restart.
 - <i>numReconfigs</i>: The value is incremented each time the node has been attempted to reconfigure.
 - <i>lastTransitionTime</i>: System.currentTimeMillis of last observed state transition.

 <b>States</b>
 | State        | Description |
 | :-------------------- | :---------- |
 | UNKNOWN               | The initial state when creating `ServerContext` for a node. We expect to transition from this state fairly quickly after creating the context for nodes.  |
 | NOT_RUNNING           | Node is not running (Kafka process is not running) |
 | NOT_READY             | Node is running but not ready to serve requests (broker state < 2 OR == 127) |
 | RESTARTED             | After successful `kubectl delete pod`. |
 | RECONFIGURED          | After successful Kafka node config update via Admin client. |
 | RECOVERING            | Node has started but is in log recovery (broker state == 2). |
 | SERVING               | Node is in running state and ready to serve requests (broker state >= 3 AND != 127). |
 | LEADING_ALL_PREFERRED | Node is in running state and leading all preferred replicas. |

2. The existing predicate function will be called for each of the nodes and those for which the function returns a non-empty list of reasons will be restarted.

2. Group the nodes into four categories:
   - `RESTART_FIRST` - Nodes that have `NOT_READY` or `NOT_RUNNING` state in their contexts. The group will also include nodes that
      we cannot connect to via Admin API.
   - `RESTART` - Nodes that have non-empty list of reasons from the predicate function and have not been restarted yet (ServerContext.numRestarts == 0).
   - `MAYBE_RECONFIGURE` - Nodes that have empty list of reasons and have not been reconfigured yet (ServerContext.numReconfigs == 0).
   - `NOP` - Nodes that have been restarted or reconfigured at least once (ServerContext.numRestarts > 0 || ServerContext.numReconfigs > 0 ) and have either
             `LEADING_ALL_PREFERRED` or `SERVING` state. Also nodes that have `RECOVERING` state.


3. Restart nodes in `RESTART_FIRST` category either one by one in the following order unless all nodes are combined
and are in `NOT_RUNNING` state:
   - Pure controller nodes
   - Combined nodes.
   - Broker only nodes

   If all controllers are combined and are in `NOT_RUNNING` state, restart all nodes in parallel and wait for them to have `SERVING`. Explained more in detail below.

   Wait until the restarted node to have `SERVING` and then `LEADING_ALL_PREFERRED` state within `postReconfigureTimeoutMs`.

4. Further refine the nodes in `MAYBE_RECONFIGURE` category:
   - Describe Kafka configurations for each node via Admin API and compare them against the desired configurations. This is essentially the same mechanism we use today for the current KafkaRoller.
   - If a node has configuration changes and they can be dynamically updated, add the node into another group called `RECONFIGURE`.
   - If a node has configuration changes but they cannot be dynamically updated, add nodes into the `RESTART` group.
   - If a node has no configuration changes, put the node into the `NOP` group.

5. Reconfigure each node in `RECONFIGURE` group:
   - If `numReconfigs` of a node is greater than the configured `maxReconfigAttempts`, add a restart reason to its context. Otherwise continue.
   - Send `incrementalAlterConfig` request with its config updates.
   - Transitions the node's state to `RECONFIGURED` and increment its `numReconfigs`.
   - Wait for each node that got configurations updated until they have `SERVING` and then `LEADING_ALL_PREFERRED` state within `postReconfigureTimeoutMs`.

6. If at this point, the `RESTART` group is empty, the reconciliation will be completed successfully.

7. Otherwise, batch nodes in `RESTART` group and get the next batch to restart:
   - Further categorize nodes based on their roles so that the following restart order can be enforced:
       1. `NON_ACTIVE_PURE_CONTROLLER` - Pure controller that is not the active controller
       2. `ACTIVE_PURE_CONTROLLER` - Pure controller is the active controller (the quorum leader)
       3. `BROKER_AND_NOT_ACTIVE_CONTROLLER` - Node that is at least a broker but also might be a controller (combined) and is not the active controller
       4. `BROKER_AND_ACTIVE_CONTROLLER` - Combined node that is the active controller (the quorum leader)
  
   - If `NON_ACTIVE_PURE_CONTROLLER` group is non empty, check which nodes can be restarted without impacting the quorum health (more on this later) and return a batch containing the first one.
   - If `ACTIVE_PURE_CONTROLLER` group is non empty, check if the node can be restarted without impacting the quorum health and return a batch containing the active controller node.
   - If `BROKER_AND_NOT_ACTIVE_CONTROLLER` group is non empty, batch the nodes:
       - build a map of brokers and their replicating partitions by sending describeTopics request to Admin API.
       - batch the nodes that do not have any partitions in common therefore can be restarted together.
       - remove nodes that have an impact on the availability from the batches (more on this later).
       - return the largest batch.

8. Restart the nodes in the returned batch in parallel:
   - If `numRestarts` of a node is larger than `maxRestarts`, return `MaxRestartsExceededException` , which will fail the reconciliation.
   - Otherwise, restart each node and transition its state to `RESTARTED` and increment its `numRestarts`.
   - After restarting all the nodes in the batch, wait for their states to become `SERVING` and then `LEADING_ALL_PREFERRED` until the configured `postRestartTimeoutMs` is reached.

9. If there are no exceptions thrown at this point, the reconciliation completes successfully.

#### Restarting not running combined nodes

When restarting not running combined nodes, we will apply a special logic to address the issue described in https://github.com/strimzi/strimzi-kafka-operator/issues/9426.

In step 3, we restart each node in the `RESTART_FIRST` group one by one. In this specific case, we will compare the total number of not running combined nodes against the total number of controller nodes in the cluster. This is to identify whether all of controllers nodes in this cluster are running in combined mode and all of them are in `NOT_RUNNING` state. In this case, we will restart all the nodes in parallel to give the best chance of forming the quorum. We will then wait for the nodes to have `SERVING` state and then `LEADING_ALL_PREFERRED`.

#### Quorum health check

The quorum health logic is similar to the current KafkaRoller except for a couple of differences. The current KafkaRoller uses the `controller.quorum.fetch.timeout.ms` config value from the desired configurations passed from the reconciler or uses the hard-coded default value if the reconciler pass null for desired configurations. The new roller will use the configuration of the active controller. This will mean that the quorum health check is done from the active controller's point of view.

Also the current KafkaRoller does not connect to the controller via Admin API to get the quorum health information. By the time, we implement this proposal, Strimzi should support Kafka 3.7 which includes [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration). Therefore new KafkaRoller will be able to connect to the controller directly for quorum information and active controller's configuration.

The total number of controllers used in the quorum healthcheck will based on the currently assigned roles of the nodes, rather the size of the `voters` list returned from `describeQuorum` API. That way, when we are scaling down, the quorum healthcheck does not fail due to inconsistent responses from the nodes. This will address the issue described in https://github.com/strimzi/strimzi-kafka-operator/issues/9434.

#### Availability check

At this point, we would have already built a map of brokers and their replicating partitions by sending describeTopics requests to the Admin API. Then ISRs that the broker is part of will be checked against the configured under minimum ISR size. If `size(ISR containing the broker) - minISR` > 0, the broker can be considered safe to restart. Otherwise, restarting the broker would cause under minimum ISR partition. If it's less than 0, it means the partition is already under minimum ISR and restarting it would either not make a difference or make things worse. In both cases, the broker should not be restarted.

### Switching from the current KafkaRoller to the new KafkaRoller

The new KafkaRoller will only work with KRaft clusters therefore when running in Zookeeper mode, the current KafkaRoller

The new KafkaRoller will be enabled by default for new KRaft clusters which means new KRaft clusters will always run with the new KafkaRoller.

The current KafkaRoller will be used during migration of existing clusters from Zookeeper to KRaft mode and will be switched to the new roller once the migration is completed.

### Future potentials

- We are not looking to solve the potential race condition between KafkaRoller and Cruise Control rebalance activity right away but this is something we can solve in the future. An example scenario that cause this race:
   Let's say we have 5 brokers cluster, minimum in sync replica for topic partition foo-0 is set to 2. The possible sequence of events that could happen:
   - Broker 0 is down due to an issue and the ISR of foo-0 partition changes from [0, 1, 2] to [1 , 2]. In this case producers with acks-all still can produce to this partition.
   - Cruise Control sends `addingReplicas` request to reassign partition foo-0 to 4 instead of 2 in order to achieve its configured goal.
   - The reassignment request is processed and foo-0 partition now has ISR [1, 2, 4].
   - Cruise Control sends `removingReplicas` request to un-assign the partition from broker 2.
   - KafkaRoller is performing a rolling update to the cluster. It checks the availability impact for foo-0 partition before rolling broker 1. Since partition foo-0 has ISR [1, 2, 4], KafkaRoller decides that it is safe to restart broker 1. It is unaware of the `removingReplicas` request that is about to be processed.
   - The reassignment request is processed and foo-0 partition now has ISR [1, 4].
   - KafkaRoller restarts broker 1 and foo-0 partition now has ISR [4] which is below the configured minimum in sync replica of 2 resulting in producers with acks-all no longer being able to produce to this partition.


## Affected/not affected projects

This proposal affects only
the [`strimzi-Kafka-operator` GitHub repository](https://github.com/strimzi/strimzi-Kafka-operator).

## Compatibility

The new KafkaRoller introduced by this proposal will be feature-gated.
This proposal should have no impact on any existing Kafka clusters deployed with ZooKeeper.

## Rejected

- Why not use rack information when batching brokers that can be restarted at the same time?
When all replicas of all partitions have been assigned in a rack-aware way then brokers in the same rack trivially share no partitions, and so racks provide a safe partitioning. However nothing in a broker, controller or cruise control is able to enforce the rack-aware property therefore assuming this property is unsafe. Even if CC is being used and rack aware replicas is a hard goal we can't be certain that other tooling hasn't reassigned some replicas since the last rebalance, or that no topics have been created in a rack-unaware way.
