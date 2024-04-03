# Kafka Roller 2.0

## Current situation

The Kafka Roller is an internal Cluster Operator component that's responsible for coordinating the rolling restart or reconfiguration of Kafka pods when:
- non-dynamic reconfigurations needs to be applied
- update in Kafka CRD is detected
- a certificate is renewed
- pods have been manually annotated by the user for controlled restarts
- pod is stuck and is out of date
- Kafka broker is unresponsive to Kafka Admin connections

A pod is considered stuck if it is in one of following states:
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `ContainerCreating`
- `Pending` and `Unschedulable`

### Known Issues

The existing KafkaRoller suffers from the following shortcomings:
- While it is safe and simple to restart one broker at a time, it is slow in large clusters ([related issue](https://github.com/strimzi/strimzi-kafka-operator/issues/8547)).
- It doesn’t worry about partition preferred leadership. This means there can be more leadership changes than necessary during a rolling restart, with consequent impact on tail latency.
- Hard to reason about when things go wrong. The code is complex to understand and it's not easy to determine why a pod was restarted from logs that tend to be noisy.
- Potential race condition between Cruise Control rebalance and KafkaRoller that could cause partitions under minimum in sync replica. This issue is described in more detail in the `Future Improvements` section.
- The current code for KafkaRoller does not easily allow growth and adding new functionality due to its complexity.


The following non-trivial fixes and changes are missing from the current KafkaRoller's KRaft implementation:
- Currently KafkaRoller has to connect to brokers successfully in order to get KRaft quorum information and determine whether a controller node can be restarted. This is because it was not possible to directly talk to KRaft controllers at the time before [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration) was implemented. The issue is raised [here](https://github.com/strimzi/strimzi-kafka-operator/issues/9692).

- KafkaRoller takes a long time to reconcile combined nodes if they are all in `Pending` state. This is because the combined node does not become ready until the quorum is formed and KafkaRoller waits for a pod to become ready before it attempts to restart other nodes. In order for the quorum to form, at least the majority of controller nodes need to be running at the same time. This is not easy to solve in the current KafkaRoller without introducing some major changes because it processes each node individually and there is no mechanism to restart multiple nodes in parallel. More information can be found [here](https://github.com/strimzi/strimzi-kafka-operator/issues/9426).

- The quorum health check is based on the `controller.quorum.fetch.timeout.ms` configuration which it reads from the desired configurations passed from the Reconciler. However, `CAReconcilor` and manual rolling update pass null value for desired configurations because in both cases, the nodes don't need reconfigurations. This results in performing the quorum healthcheck based on the hard-coded default value of `controller.quorum.fetch.timeout.ms` rather than the accurate configuration value when doing manual rolling update and rolling nodes for certificate renewal.


## Motivation

Strimzi users have been reporting some of the issues mentioned above and would benefit from a new KafkaRoller that is designed to address the shortcomings of the current KafkaRoller.

The current KafkaRoller has complex and nested conditions therefore makes it challenging for users to debug and understand actions taken on their brokers when things go wrong and configure it correctly for their use cases. It is also not particularly easy to unit test which results in insufficient test coverage for many edge cases, making it challenging to refactor safely. Therefore, refactoring becomes essential to enhance test coverage effectively. A new KafkaRoller that is redesigned to be simpler would help users to easily understand the code and configure it to their needs.

As you can see above, the current KafkaRoller still needs various changes and potentially more as we get more experience with KRaft and discover more issues. Adding these non trivial changes to a component that is very complex and hard to reason, is expensive and poses potential risks of introducing bugs because of tightly coupled logics and lack of testability.

## Proposal

The objective of this proposal is to introduce a new KafkaRoller with simplified logic having a structured design resembling a finite state machine. KafkaRoller decisions are informed by observations coming from different sources (e.g. Kubernetes API, KafkaAgent, Kafka Admin API). These sources will be abstracted so that KafkaRoller is not dependent on their specifics as long as it's getting the information it needs. The abstractions also enable much better unit testing.

Depending on the observed states, the roller will perform specific actions. Those actions should cause a subsequent observation to cause a state transition. This iterative process continues until each node's state aligns with the desired state.

It will also introduce an algorithm that can restart brokers in parallel while applying safety conditions that can guarantee Kafka producer availability and causing minimal impact on controllers and overall stability of clusters.

### Node State
When a new reconciliation starts up, a context object is created for each node to store the state and other useful information used by the roller. It will have the following fields:

 - <i>nodeRef</i>: NodeRef object that contains Node ID.
 - <i>currentNodeRole</i>: Currently assigned process roles for this node (e.g. controller, broker).
 - <i>state</i>: It contains the current state of the node based on information collected from the abstracted sources (Kubernetes API, KafkaAgent and Kafka Admin API). The table below describes the possible states.
 - <i>reason</i>: It is updated based on the current predicate logic from the Reconciler. For example, an update in the Kafka CR is detected.
 - <i>numRestartAttempts</i>: The value is incremented each time the node has been attempted to restart.
 - <i>numReconfigAttempts</i>: The value is incremented each time the node has been attempted to reconfigure.
 - <i>numRetries</i>: The value is incremented each time the node cannot be restarted/reconfigured due to not meeting safety conditions (more on this later).
 - <i>lastTransitionTime</i>: System.currentTimeMillis of last observed state transition.

 <b>States</b>
 | State        | Description |
 | :-------------------- | :---------- |
 | UNKNOWN               | The initial state when creating `Context` for a node. We expect to transition from this state fairly quickly after creating the context for nodes.  |
 | NOT_RUNNING           | Node is not running (Kafka process is not running) |
 | NOT_READY             | Node is running but not ready to serve requests (broker state < 2 OR == 127) |
 | RESTARTED             | After successful `kubectl delete pod`. |
 | RECONFIGURED          | After successful Kafka node config update via Admin client. |
 | RECOVERING            | Node has started but is in log recovery (broker state == 2). |
 | SERVING               | Node is in running state and ready to serve requests (broker state >= 3 AND != 127). |
 | LEADING_ALL_PREFERRED | Node is in running state and leading all preferred replicas. |

The broker states are defined [here](https://github.com/apache/kafka/blob/58ddd693e69599b177d09c2e384f31e7f5e11171/metadata/src/main/java/org/apache/kafka/metadata/BrokerState.java#L46).

### Configurability
The following can be the configuration options for the new KafkaRoller:

| Configuration | Default value | Exposed to user | Description |
| :-------------| :-------------| :---------------| :---------- |
| maxRestartAttempts | 3 | No | The maximum number of times a node can be restarted before failing the reconciliation. This is checked against the node's `numRestartAttempts`.|
| maxReconfigAttempts | 3 | No | The maximum number of times a node can be reconfigured before restarting it. This is checked against the node's `numReconfigAttempts`.|
| maxRetries| 10 | No | The maximum number of times a node can retried after not meeting the safety conditions. This is checked against the node's `numRetries`.|
| postOperationTimeoutMs | 60 seconds | Yes | The maximum amount of time we will wait for nodes to transition to `SERVING` state after an operation in each retry. This will be based on the operation timeout that is already exposed to the user via environment variable `STRIMZI_OPERATION_TIMEOUT_MS`. |
| maxRestartParallelism | 1 | Yes | The maximum number of broker nodes that can be restarted in parallel. |


### Algorithm

1. Initialise a context object for each node with the following data:
```
Context: {
         nodeRef: <NodeRef object passed from KafkaReconciler>
         nodeRoles: <This will be set using the pod labels `strimzi.io/controller-role` and `strimzi.io/broker-role>
         state: UNKNOWN
         lastTransition: <SYSTEM_TIME>
         reason: <Result of the predicate function passed from KafkaReconciler>
         numRestartAttempts: 0
         numReconfigAttempts: 0
         numRetries: 0
}
```               
2. Observe and transition each node's state to the corresponding state based on the information collected from the abstracted sources.

3. If there are nodes in `NOT_READY` state, wait for them to have `SERVING` within the `postOperationalTimeoutMs`.  
   We want to give nodes chance to get ready before we try to connect to the or consider them for rolling. This is important especially for nodes which were just started.
   This is consistent with how the current roller handles unready nodes.
   - If the timeout is reached, proceed to the next step and check if any of the nodes need to be restarted.

4. Group the nodes into the following categories based on their state and connectivity:
   - `RESTART_FIRST` - Nodes that have `NOT_READY` or `NOT_RUNNING` state in their contexts. The group will also include nodes that we cannot connect to via Admin API.
   - `WAIT_FOR_LOG_RECOVERY` -  Nodes that have `RECOVERING` state.
   - `RESTART` - Nodes that have non-empty list of reasons from the predicate function and have not been restarted yet (Context.numRestartAttempts == 0).
   - `MAYBE_RECONFIGURE` - Broker nodes (including combined nodes) that have an empty list of reasons and not been reconfigured yet (Context.numReconfigAttempts == 0).
   - `NOP` - Nodes that have at least one restart or reconfiguration attempt (Context.numRestartAttempts > 0 || Context.numReconfigAttempts > 0 ) and have either
             `LEADING_ALL_PREFERRED` or `SERVING` state.

5. Wait for nodes in `WAIT_FOR_LOG_RECOVERY` group to finish performing log recovery. 
   - Wait for nodes to have `SERVING` within the `postOperationalTimeoutMs`. 
   - If the timeout is reached for a node and its `numRetries` is greater than or equal to `maxRetries`, throw `UnrestartableNodesException` with the log recovery progress (number of remaining logs and segments). Otherwise increment node's `numRetries` and repeat from step 2.

6. Restart nodes in `RESTART_FIRST` category:
   - if one or more nodes have `NOT_RUNNING` state, we first need to check 2 special conditions:
      - If all of the nodes are combined and are in `NOT_RUNNING` state, restart them in parallel to give the best chance of forming the quorum.
      > This is to address the issue described in https://github.com/strimzi/strimzi-kafka-operator/issues/9426.

      - If a node is in `NOT_RUNNING` state, the restart it only if it has `POD_HAS_OLD_REVISION` reason. This is because, if the node is not running at all, then restarting it likely won't make any difference unless the node is out of date.
      > For example, if a pod is in pending state due to misconfigured affinity rule, there is no point restarting this pod again or restarting other pods, because that would leave them in pending state as well. If the user then fixed the misconfigured affinity rule, then we should detect that the pod has an old revision, therefore should restart it so that pod is scheduled correctly and runs.

      - At this point either we started nodes or decided not to because nodes did not have `POD_HAS_OLD_REVISION` reason. Regardless, wait for nodes to have `SERVING` within `postOperationalTimeoutMs`. If the timeout is reached and the node's `numRetries` is greater than or equal to `maxRetries`, throw `TimeoutException`. Otherwise increment node's `numRetries` and repeat from step 2. 

   
   - Otherwise the nodes will be attempted to restart one by one in the following order:
      - Pure controller nodes
      - Combined nodes
      - Broker only nodes
      
   - Wait for the restarted node to have `SERVING` within `postOperationalTimeoutMs`. If the timeout is reached and the node's `numRetries` is greater than or equal to `maxRetries`, throw `TimeoutException`. Otherwise increment node's `numRetries` and repeat from step 2. 

7. Further refine the broker nodes in `MAYBE_RECONFIGURE` group:
   - Describe Kafka configurations for each node via Admin API and compare them against the desired configurations. This is essentially the same mechanism we use today for the current KafkaRoller.
   - If a node has configuration changes and they can be dynamically updated, add the node into another group called `RECONFIGURE`.
   - If a node has configuration changes but they cannot be dynamically updated, add the node into the `RESTART` group.
   - If a node has no configuration changes, put the node into the `NOP` group.

8. Reconfigure each node in `RECONFIGURE` group:
   - If `numReconfigAttempts` of a node is greater than the configured `maxReconfigAttempts`, add a restart reason to its context and repeat from step 2. Otherwise continue.
   - Send `incrementalAlterConfig` request with its config updates.
   - Transitions the node's state to `RECONFIGURED` and increment its `numReconfigAttempts`.
   - Wait for each node that got configurations updated until they have `LEADING_ALL_PREFERRED` within the `postOperationalTimeoutMs`.
   - If the `postOperationalTimeoutMs` is reached, repeat from step 2.

9. If at this point, the `RESTART` group is empty and if there is no nodes that is in `NOT_READY` state, the reconciliation will be completed successfully.
   - If there are nodes in `NOT_READY` state, wait for them to have `SERVING` within the `postOperationalTimeoutMs`. 
   - If the timeout is reached for a node and its `numRetries` is greater than or equal to `maxRetries`, throw `TimeoutException`. 
   - Otherwise increment node's `numRetries` and repeat from step 2.
   This is consistent with how the current roller handles unready nodes.

10. Otherwise, batch nodes in `RESTART` group and get the next batch to restart:
   - Further categorize nodes based on their roles so that the following restart order can be enforced:
       1. `NON_ACTIVE_CONTROLLER` - Pure controller that is not the active controller
       2. `ACTIVE_CONTROLLER` - Pure controller that is the active controller (the quorum leader)
       3. `COMBINED_AND_NOT_ACTIVE_CONTROLLER` - Combined node (both controller and broker) and is not the active controller
       4. `COMBINED_AND_ACTIVE_CONTROLLER` - Combined node (both controller and broker) and is the active controller (the quorum leader)
       5. `BROKER` - Pure broker
       
      > The batch returned will comprise only one node for all groups except 'BROKER', ensuring that controllers are restarted sequentially. This approach is taken to mitigate the risk of losing quorum when restarting multiple controller nodes simultaneously. A failure to establish quorum due to unhealthy controller nodes directly impacts the brokers and consequently the availability of the cluster. However, restarting broker nodes can be executed without affecting availability. If concurrently restarting brokers do not share any topic partitions, the in-sync replicas (ISRs) of topic partitions will lose no more than one replica, thus preserving availability.
  
   - If `NON_ACTIVE_PURE_CONTROLLER` group is non empty, return the first node that can be restarted without impacting the quorum health (more on this later).
   - If `ACTIVE_PURE_CONTROLLER` group is non empty, return the node if it can be restarted without impacting the quorum health. Otherwise return an empty set.
   - If `COMBINED_AND_NOT_ACTIVE_CONTROLLER` group is non empty, return the first node that can be restarted without impacting the quorum health and the availability.
   - If `COMBINED_AND_ACTIVE_CONTROLLER`  group is non empty, return the node if it can be restarted without impacting the quorum health and the availability. Otherwise return an empty set.
   - If `BROKER` group is non empty, batch the broker nodes:
       - build a map of nodes and their replicating partitions by sending describeTopics request to Admin API
       - batch the nodes that do not have any partitions in common therefore can be restarted together
       - remove nodes that have an impact on the availability from the batches (more on this later)
       - return the largest batch
   - If an empty batch is returned, that means none of the nodes met the safety conditions such as availability and qourum health impact. In this case, check their `numRetries` and if any of them is equal to or greater than `maxRetries`, throw `UnrestartableNodesException`. Otherwise increment their `numRetries` and repeat from step 2.

11. Restart the nodes from the returned batch in parallel:
   - If `numRestartAttempts` of a node is larger than `maxRestartAttempts`, throw `MaxRestartsExceededException`.
   - Otherwise, restart each node and transition its state to `RESTARTED` and increment its `numRestartAttempts`.
   - After restarting all the nodes in the batch, wait for their states to become `SERVING` until the configured `postOperationalTimeoutMs` is reached.
   - If the timeout is reached, throw `TimeoutException`. If a node's `numRetries` is greater than or equal to `maxRetries`. Otherwise increment their `numRetries` and repeat from step 2.
   - After all the nodes are in `SERVING` state, trigger preferred leader elections via Admin client. Wait for their states to become `LEADING_ALL_PREFERRED` until the configured `postOperationalTimeoutMs` is reached. If the timeout is reached, log a `WARN` message. 


12. If there are no exceptions thrown at this point, the reconciliation completes successfully. If there were `UnrestartableNodesException`, `TimeoutException`, `MaxRestartsExceededException` or any other unexpected exceptions throws, the reconciliation fails.

#### Quorum health check

The quorum health logic is similar to the current KafkaRoller except for a couple of differences. The current KafkaRoller uses the `controller.quorum.fetch.timeout.ms` config value from the desired configurations passed from the reconciler or uses the hard-coded default value if the reconciler pass null for desired configurations. The new roller will use the configuration value of the active controller. This will mean that the quorum health check is done from the active controller's point of view.

Also the current KafkaRoller does not connect to the controller via Admin API to get the quorum health information. By the time, we implement this proposal, Strimzi should support Kafka 3.7 which includes [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration). Therefore new KafkaRoller will be able to connect to the controller directly for quorum information and active controller's configuration.

#### Availability check

The availibility check logic similar to the current KafkaRoller. The ISRs that the broker is part of is checked against the configured under minimum ISR size. If `size(ISR containing the broker) - minISR > 0`, the broker can be considered safe to restart. If it equals to 0, restarting the broker could cause under minimum ISR partition. If it's less than 0, it means the partition is already under minimum ISR and restarting it would either not make a difference or make things worse. In both cases, the broker should not be restarted. 

However, if `size(ISR containing the broker) - minISR <= 0` but the topic partition is configured with replication size less than minISR, the check will pass to proceed with the broker restart.

### Switching from the old KafkaRoller to the new KafkaRoller

The new KafkaRoller will only work with KRaft clusters therefore when running in Zookeeper mode, the current KafkaRoller will be used.

Kafka CR's `KafkaMetadataState` represents where the metadata is stored for the cluster. It is set to `KRaft` when a cluster is fully migrated to KRaft or was created in KRaft mode. KafkaReconciler will be updated to switch to the new roller based on this state. This means the old KafkaRoller will be used during migration of existing clusters from Zookeeper to KRaft mode and the new roller is used only after the migration is completed and for new clusters created in KRaft mode.

### Future improvement

- We are not looking to solve the potential race condition between KafkaRoller and Cruise Control rebalance activity right away but this is something we can solve in the future. An example scenario that cause this race:
   Let's say we have 5 brokers cluster, minimum in sync replica for topic partition foo-0 is set to 2. The possible sequence of events that could happen:
   - Broker 0 is down due to an issue and the ISR of foo-0 partition changes from [0, 1, 2] to [1 , 2]. In this case producers with acks-all still can produce to this partition.
   - Cruise Control sends `addingReplicas` request to reassign partition foo-0 to broker 4 instead of broker 2 in order to achieve its configured goal.
   - The reassignment request is processed and foo-0 partition now has ISR [1, 2, 4].
   - Cruise Control sends `removingReplicas` request to un-assign the partition from broker 2.
   - KafkaRoller is performing a rolling update to the cluster. It checks the availability impact for foo-0 partition before rolling broker 1. Since partition foo-0 has ISR [1, 2, 4], KafkaRoller decides that it is safe to restart broker 1. It is unaware of the `removingReplicas` request that is about to be processed.
   - The reassignment request is processed and foo-0 partition now has ISR [1, 4].
   - KafkaRoller restarts broker 1 and foo-0 partition now has ISR [4] which is below the configured minimum in sync replica of 2 resulting in producers with acks-all no longer being able to produce to this partition.


## Affected/not affected projects

This proposal affects only
the [`strimzi-Kafka-operator` GitHub repository](https://github.com/strimzi/strimzi-Kafka-operator).

## Compatibility

The new KafkaRoller introduced by this proposal will used only for KRaft based clusters.
This proposal should have no impact on any existing Kafka clusters deployed with ZooKeeper.

## Rejected

- Why not use rack information when batching brokers that can be restarted at the same time?
When all replicas of all partitions have been assigned in a rack-aware way then brokers in the same rack trivially share no partitions, and so racks provide a safe partitioning. However nothing in a broker, controller or cruise control is able to enforce the rack-aware property therefore assuming this property is unsafe. Even if CC is being used and rack aware replicas is a hard goal we can't be certain that other tooling hasn't reassigned some replicas since the last rebalance, or that no topics have been created in a rack-unaware way.
