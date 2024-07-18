# Kafka Roller 2.0

## Current situation

The Kafka Roller is an internal Cluster Operator component that's responsible for coordinating the rolling restart or reconfiguration of Kafka pods when:
- non-dynamic reconfigurations needs to be applied
- update in Kafka CR is detected
- a TLS certificate is renewed
- pods have been [manually annotated](https://strimzi.io/docs/operators/latest/full/deploying#rolling_pods_manually_alternative_to_drain_cleaner) by the user for controlled restarts
- pod is stuck and has a pending update (e.g. not running with the desired version or configuration)
- Kafka broker is unresponsive to Kafka Admin connections

A pod is considered stuck if it is in one of following states:
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `ContainerCreating`
- `Pending` and `Unschedulable`

### Known Issues

The existing KafkaRoller suffers from the following shortcomings:
- Although it is safe and straightforward to restart one broker at a time, this process is slow in large clusters ([related issue](https://github.com/strimzi/strimzi-kafka-operator/issues/8547)).
- It does not account for partition preferred leadership. As a result, there may be more leadership changes than necessary during a rolling restart, consequently impacting clients because they would need to reconnect everytime.
- Hard to reason about when things go wrong. The code is complex to understand and it's not easy to determine why a pod was restarted from logs that tend to be noisy.
- Potential race condition between Cruise Control rebalance and KafkaRoller that could cause partitions under minimum in sync replica. This issue is described in more detail in the `Future Improvements` section.
- The current code for KafkaRoller does not easily allow growth and adding new functionality due to its complexity.


The following non-trivial fixes and changes are missing from the current KafkaRoller's KRaft implementation:
- KafkaRoller takes a long time to reconcile mixed nodes if they are all in `Pending` state. This is because a mixed node does not become ready until the quorum is formed and KafkaRoller waits for a pod to become ready before it attempts to restart other nodes. In order for the quorum to form, at least the majority of controller nodes need to be running at the same time. This is not easy to solve in the current KafkaRoller without introducing some major changes because it processes each node individually and there is no mechanism to restart multiple nodes in parallel. More information can be found [here](https://github.com/strimzi/strimzi-kafka-operator/issues/9426).

- The quorum health check relies on the `controller.quorum.fetch.timeout.ms` configuration, which is determined by the desired configuration values. However, during certificate reconciliation or manual rolling updates, KafkaRoller doesn't have access to these desired configuration values since they shouldn't prompt any configuration changes. As a result, the quorum health check defaults to using the hard-coded default value of `controller.quorum.fetch.timeout.ms` instead of the correct configuration value during manual rolling updates or when rolling nodes for certificate renewal.


## Motivation

Strimzi users have been reporting some of the issues mentioned above and would benefit from a new KafkaRoller that is designed to address the shortcomings of the current KafkaRoller.

The current KafkaRoller has complex and nested conditions therefore makes it challenging for users to debug and understand actions taken on their brokers when things go wrong and configure it correctly for their use cases. It is also not particularly easy to unit test which results in insufficient test coverage for many edge cases, making it challenging to refactor safely. Therefore, refactoring becomes essential to enhance test coverage effectively. A new KafkaRoller that is redesigned to be simpler would help users to easily understand the code and configure it to their needs.

As you can see above, the current KafkaRoller still needs various changes and potentially more as we get more experience with KRaft and discover more issues. Adding these non trivial changes to a component that is very complex and hard to reason, is expensive and poses potential risks of introducing bugs because of tightly coupled logics and lack of testability.

## Proposal

The objective of this proposal is to introduce a new KafkaRoller with more structured design resembling a finite state machine. Given the number of new features and changes related to KRaft, it is easiest to rewrite it from scratch rather than refactoring the existing component. With a more structured design, the process for evaluating pods in various states such as not running, unready, or lacking a connection; and deciding whether to restart them would become more defined and easier to follow.

KafkaRoller decisions would be informed by observations coming from different sources (e.g. Kubernetes API, KafkaAgent, Kafka Admin API). These sources will be abstracted so that KafkaRoller is not dependent on their specifics as long as it's getting the information it needs. The abstractions also enable much better unit testing.

Nodes would be categorized based on the observed states, the roller will perform specific actions on nodes in each category. Those actions should cause a subsequent observation to cause a state transition. This iterative process continues until each node's state aligns with the desired state.

In addition, the new KafkaRoller will introduce an algorithm to restart brokers in parallel when safety conditions are met. These conditions ensure Kafka producer availability and minimize the impact on controllers and overall cluster stability. It will also wait for partitions to be reassigned to their preferred leaders to avoid triggering unnecessary partition leader elections.


### Node State
When a new reconciliation starts up, a context object is created for each node to store the state and other useful information used by the roller. It will have the following fields:

 - <i>nodeRef</i>: NodeRef object that contains Node ID.
 - <i>currentNodeRole</i>: Currently assigned process roles for this node (e.g. controller, broker).
 - <i>lastKnownState</i>: It contains the last known state of the node based on information collected from the abstracted sources (Kubernetes API, KafkaAgent and Kafka Admin API). The table below describes the possible states.
 - <i>restartReason</i>: It is updated based on the current predicate logic passed from the `KafkaReconciler` class. For example, an update in the Kafka CR is detected.
 - <i>numRestartAttempts</i>: The value is incremented each time the node has been restarted or attempted to be restarted.
 - <i>numReconfigAttempts</i>: The value is incremented each time the node has been reconfigured or attempted to be reconfigured.
 - <i>numRetries</i>: The value is incremented each time the node is evaluated/processed but was not restarted/reconfigured due to not meeting safety conditions for example, availability check failed, log recovery or timed out waiting for pod to become ready.
 - <i>lastTransitionTime</i>: System.nanoTime of last observed state transition.

 The following table illustrates possible states for `lastKnownState` field and the next states it can transition into:
 | State            | Description      | Next possible transitions |
 | :--------------- | :--------------- | :----------- |
 | UNKNOWN               | The initial state when creating `Context` for a node or state just after the node gets restarted/reconfigured. We expect to transition from this state fairly quickly.  | `NOT_RUNNING` `NOT_READY` `RECOVERING` `READY` |
 | NOT_RUNNING           | Node is not running (Kafka process is not running). This is determined via Kubernetes API, more details for it below. | `READY` `UNKNOWN` `NOT_READY` `RECOVERING` | 
 | NOT_READY             | Node is running but not ready to serve requests which is determined by Kubernetes readiness probe (broker state is not RUNNING OR controller is not listening on port). | `READY` `UNKNOWN` `NOT_RUNNING` `RECOVERING` |
 | RECOVERING            | Node has started but is in log recovery (broker state == 2). This is determined via the KafkaAgent. | `READY` `NOT_RUNNING` `NOT_READY` |
 | READY               | Node is in running state and ready to serve requests which is determined by Kubernetes readiness probe (broker state is RUNNING OR controller is listening on port). | `LEADING_ALL_PREFERRED` `UNKNOWN` |
 | LEADING_ALL_PREFERRED | Node is leading all the partitions that it is the preferred leader for. Node's state can transition into this only from `READY` state.  | This is the final state we expect

Context about broker states and restart reasons:
- To determine if the node is ready or performing a log recovery, we use the [Broker States](https://github.com/apache/kafka/blob/3.7/metadata/src/main/java/org/apache/kafka/metadata/BrokerState.java) metric emitted by Kafka. KafkaAgent collects and exposes this metric via REST Endpoint. This is what the current KafkaRoller does already, and the new roller will use it the same way.

- If Kafka pod is ready, the restart reasons is checked to determine whether it needs to be restarted. The definitions of the possible restart reasons can be found via the following link: [Restart Reasons](https://github.com/strimzi/strimzi-kafka-operator/blob/0.40.0/cluster-operator/src/main/java/io/strimzi/operator/cluster/model/RestartReason.java). This is also what the current KafkaRoller roller does and the new roller will use it the same way.

#### NOT_RUNNING state

If one of the following is true, then node's state is `NOT_RUNNING`:
- no pod exists for this node
- unable to get the `Pod Status` for the pod 
- the pod has `Pending` status with `Unschedulable` reason
- the pod has container status `ContainerStateWaiting` with `CrashLoopBackOff` or `ImagePullBackOff` reason
If none of the above is true but the node is not ready, then its state would be `NOT_READY`.

#### High level flow diagram describing the flow of the states
![The new roller flow](./images/06x-new-roller-flow.png)



### Configurability
The following are the configuration options for the new KafkaRoller. If exposed to user, the user can configure it via `STRIMZI_` environment variables. Otherwise, the operator will set them to the default values (which are similar to what the current roller has):

| Configuration          | Default value | Exposed to user | Description                                                                                                                                                                                                                                                   |
|:-----------------------|:--------------|:----------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| maxRestartAttempts     | 3             | No              | The maximum number of restart attempts per node before failing the reconciliation. This is checked against node's `numRestartAttempts`.                                                                                                               |
| maxReconfigAttempts    | 3             | No              | The maximum number of dynamic reconfiguration attempts per node before restarting the node. This is checked against node's `numReconfigAttempts`.                                                                                                                        |
| maxRetries             | 10            | No              | The maximum number of times a node can be retried after not meeting the safety conditions e.g. availability check failed.  This is checked against the node's `numRetries`.                                                                                                                      |
| operationTimeoutMs | 60 seconds    | Yes             | The maximum amount of time we will wait for nodes to transition to `READY` state after an operation in each retry. This is already exposed to the user via environment variable `STRIMZI_OPERATION_TIMEOUT_MS`. |
| maxRestartParallelism  | 1             | Yes             | The maximum number of broker nodes that can be restarted in parallel. This will be exposed to the user via the new environment variable `STRIMZI_MAX_RESTART_BATCH_SIZE`. However, if there are multiple brokers in `NOT_RUNNING` state, they may get restarted in parallel despite this configuration for a faster recovery.    
| postRestartDelay  | 0             | Yes             | Delay between restarts of nodes or batches. It's set to 0 by default, but can be adjusted by users to slow down the restarts. This will also help JIT to reach a steady state and to reduce impact on clients.   
| restartAndPreferredLeaderElectionDelay  | 10 seconds             | No             | Delay between restart and triggering partition leader election so that just-rolled broker is leading all the partitions it is the preferred leader for. This is to avoid situations where leaders moving to a newly started node that does not yet have established networking to some outside networks, e.g. through load balancers.

### Algorithm

1. **Initialize Context for Each Node:**
   Create a context object with the following data:
   ```
   Context: {
       nodeRef: <NodeRef from KafkaReconciler>,
       nodeRoles: <Set using pod labels `strimzi.io/controller-role` and `strimzi.io/broker-role`>,
       state: UNKNOWN,
       lastTransition: <SYSTEM_TIME>,
       restartReason: <Result of predicate function from KafkaReconciler>,
       numRestartAttempts: 0,
       numReconfigAttempts: 0,
       numRetries: 0
   }
   ```
   Contexts are recreated in each reconciliation with the above initial data.

2. **Transition Node States:**
   Update each node's state based on information from abstracted sources. If failed to retrieve information, the current reconciliation immediately fails. When the next reconciliation is triggered, it will restart from step 1.

3. **Handle `NOT_READY` Nodes:**
   Wait for `NOT_READY` nodes to become `READY` within `operationTimeoutMs`.

4. **Categorize Nodes:**
   Group nodes based on their state and connectivity:
   - `RESTART_NOT_RUNNING`: Nodes in `NOT_READY` state.
   - `WAIT_FOR_LOG_RECOVERY`: Nodes in `RECOVERING` state.
   - `RESTART_UNRESPONSIVE`: Nodes unresponsive via Admin API.
   - `MAYBE_RECONFIGURE_OR_RESTART`: Broker nodes with empty reason lists and no previous restarts/reconfigurations.
   - `RESTART`: Nodes with reasons for restart and no previous restarts.
   - `NOP`: Nodes needing no operation.

5. **Wait for Log Recovery:**
   Wait for `WAIT_FOR_LOG_RECOVERY` nodes to become `READY` within `operationTimeoutMs`. If timeout is reached and `numRetries` exceeds `maxRetries`, throw `UnrestartableNodesException`. Otherwise, increment `numRetries` and repeat from step 2.

6. **Restart `RESTART_NOT_RUNNING` Nodes:**
   Restart nodes in `NOT_RUNNING` state, considering special conditions:
   - If all controller nodes are `NOT_RUNNING`, restart them in parallel to form a quorum.
      > This is to address the issue described in https://github.com/strimzi/strimzi-kafka-operator/issues/9426.
   - Restart `NOT_RUNNING` nodes with `POD_HAS_OLD_REVISION` in parallel. This is because, if the node is not running at all, then restarting it likely won't make any difference unless the node is out of date.
      > For example, if a pod is in pending state due to misconfigured affinity rule, there is no point restarting this pod again or restarting other pods, because that would leave them in pending state as well. If the user then fixed the misconfigured affinity rule, then we should detect that the pod has an old revision, therefore should restart it so that pod is scheduled correctly and runs.
   - Wait for each node's state to transition to `READY` within `operationTimeoutMs`. If timeout is reached, increment `numRetries` and repeat from step 2.

7. **Restart `RESTART_UNRESPONSIVE` Nodes:**
   Restart unresponsive nodes one by one in the order: pure controller, mixed, and broker nodes. Wait for each node's state to transition to `READY` within `operationTimeoutMs`. If timeout is reached, increment `numRetries` and repeat from step 2.

8. **Refine `MAYBE_RECONFIGURE_OR_RESTART` Nodes:**
   Describe Kafka configurations via Admin API:
   - Nodes with dynamic config changes are added to `RECONFIGURE` group.
   - Nodes with non dynamic config changes are added `RESTART` group.
   - Nodes with no config changes are added to `NOP` group.

9. **Reconfigure Nodes:**
   Reconfigure nodes in the `RECONFIGURE` group:
   - Check if `numReconfigAttempts` exceeds `maxReconfigAttempts`. If exceeded, add a restart reason and repeat from step 2. Otherwise, continue.
   - Send `incrementalAlterConfig` request, transition state to `UNKNOWN`, and increment `numReconfigAttempts`.
   - Wait for each node's state to transition to `READY` within `operationTimeoutMs`. If timeout is reached, repeat from step 2, otherwise continue.

10. **Check for `NOT_READY` Nodes:**
   If `RESTART` group is empty and no nodes are `NOT_READY`, reconciliation is successful. Otherwise, wait for `NOT_READY` nodes' state to transition to `READY` within `operationTimeoutMs`. If timeout is reached, increment `numRetries` and repeat from step 2. Otherwise, continue.

11. **Categorize and Batch Nodes:**
   Categorize and batch nodes for restart:
   - Ensure controllers are restarted sequentially in an order of pure controllers, mixed nodes and the active controller to maintain quorum.
   - Group broker nodes without common partitions for parallel restart to maintain availability.
   - If no safe nodes to restart, check `numRetries`. If exceeded, throw `UnrestartableNodesException`. Otherwise, increment `numRetries` and repeat from step 2.  More on safety conditions below.

12. **Restart Nodes in Parallel:**
   Restart broker nodes in the batch:
   - If `numRestartAttempts` exceeds `maxRestartAttempts`, throw `MaxRestartsExceededException`.
   - Restart nodes, transition state to `UNKNOWN`, and increment `numRestartAttempts`.
   - Wait for each node's state to transition to `READY` within `operationTimeoutMs`. If timeout is reached, increment `numRetries` and repeat from step 2.
   - After nodes are `READY`, apply `restartAndPreferredLeaderElectionDelay` and trigger preferred leader elections. Wait for nodes to transition to `LEADING_ALL_PREFERRED` state within `operationTimeoutMs`.

13. **Handle Exceptions:**
   If no exceptions are thrown, reconciliation is successful. If exceptions occur, reconciliation fails.

14. **Repeat Reconciliation:**
   Start the next reconciliation from step 1.

#### Quorum health check

The quorum health logic is similar to the current KafkaRoller except for a couple of differences. The current KafkaRoller uses the `controller.quorum.fetch.timeout.ms` config value from the desired configurations passed from the reconciler or uses the hard-coded default value if the reconciler pass null for desired configurations. The new roller will use the configuration value of the active controller. This will mean that the quorum health check is done from the active controller's point of view.

Also the current KafkaRoller does not connect to the controller via Admin API to get the quorum health information. By the time, we implement this proposal, Strimzi should support Kafka 3.7 which includes [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration). Therefore new KafkaRoller will be able to connect to the controller directly for quorum information and active controller's configuration.

#### Availability check

The availibility check logic similar to the current KafkaRoller. The ISRs that the broker is part of is checked against the configured under minimum ISR size. If `size(ISR containing the broker) - minISR > 0`, the broker can be considered safe to restart. If it equals to 0, restarting the broker could cause under minimum ISR partition. If it's less than 0, it means the partition is already under minimum ISR and restarting it would either not make a difference or make things worse. In both cases, the broker should not be restarted. 

However, if `size(Replicas containing the broker) - minISR <= 0` but the topic partition is configured with replication size less than minISR, the check will pass to proceed with the broker restart.

#### An example of rolling update

Here is an example of the new roller performing rolling restarts on a cluster with 12 nodes: 3 controllers, 3 mixed nodes and 6 brokers. The nodes are:
- controller-0
- controller-1
- controller-2
- mixed-3
- mixed-4
- mixed-5
- broker-6
- broker-7
- broker-8
- broker-9
- broker-10
- broker-11

1. The roller observes nodes and update their contexts based on the observation outcome:

All the nodes except `mixed-3` have the following Context with `nodeRef` being their `podname/node-id`, and `nodeRoles` having either `controller`, `broker` or both.
```
         nodeRef: controller-0/0
         nodeRoles: controller
         state: READY
         lastTransition: 0123456
         restartReason: MANUAL_ROLLING_UPDATE
         numRestartAttempts: 0
         numReconfigAttempts: 0
         numRetries: 0
```
The `mixed-3` node has the following context because the operator could not establish an admin connection to it even though it's ready from Kubernetes and KafkaAgent perspective:
```
         nodeRef: mixed-3/3
         nodeRoles: controller,broker
         state: NOT_READY
         lastTransition: 0123456
         restartReason: POD_UNRESPONSIVE
         numRestartAttempts: 0
         numReconfigAttempts: 0
         numRetries: 0
```
2. The roller checks if all of the controller nodes are in `NOT_RUNNING` state. Since they are not and `mixed-3` node has `POD_UNRESPONSIVE` reason, it is restarted and waited to have `READY` state. The `mixed-3`'s context becomes:
```
         nodeRef: mixed-3/3
         nodeRoles: controller,broker
         state: UNKNOWN
         lastTransition: 654987
         restartReason: POD_UNRESPONSIVE
         numRestartAttempts: 1
         numReconfigAttempts: 0
         numRetries: 0
```
3. `mixed-3` state becomes `READY` and since its `numRestartAttempts` is greater than 1, the roller checks the rest of the nodes. 
4. The roller checks which node is the active controller and finds that `controller-0` is. It then sends a request to the active controller via AdminClient to describe its `controller.quorum.fetch.timeout` config value.
5. It then considers restarting `controller-1` and checks if the quorum health would be impacted. The operator sends a request to the active controller to describe the quorum replication state. It finds that majority of the follower controllers have caught up with the quorum leader within the `controller.quorum.fetch.timeout.ms`. 
6. The roller restarts `controller-1` as it has no impact on the quorum health. When it has `READY` state, the roller repeats the quorum check and restarts `controller-2` and then `controller-0`. 
7. It then considers restarting `mixed-4`, so it performs quorum healthcheck and then availability check. Both check passes therefore `mixed-4` is restarted. The same is repeated for `mixed-5`.
8. All the controller and mixed nodes have `READY` state and `numRestartAttempts` set to greater than 1. This means, they have been successfuly restarted, therefore the roller considers restarting the broker nodes.
9. It sends requests to describe all the topic partitions and their `min.insync.replicas` configuration, and the following list of topics is returned:
```
topic("topic-A"), Replicas(9, 10, 11), ISR(9, 10), MinISR(2)
topic("topic-B"), Replicas(6, 7, 8), ISR(6, 7, 8), MinISR(2)
topic("topic-C"), Replicas(10, 8, 6), ISR(10, 8, 6), MinISR(2)
topic("topic-D"), Replicas(7, 9, 11), ISR(7, 9, 11), MinISR(2)
topic("topic-E"), Replicas(6, 10, 11), ISR(6, 10, 11), MinISR(2)
```

10. The roller batches the nodes that do not have any topic partition in common and the following batches are created: 
- (11, 8) - `broker-11` and `broker-8` do not share any topic partitions.
- (7) - `broker-7` and `broker-10` do not share any topic partitions, however topic-A is at min ISR, therefore 10 cannot be restarted and is removed from the batch.
- (6) - `broker-6` and `broker-9` do not share any topic partitions, however topic-A is at min ISR, therefore 9 cannot be restarted and is removed from the batch.

11. The roller picks the largest batch containing `broker-11` and `broker-8` and restarts them together. It waits for the nodes to have `READY` and then `LEADING_ALL_PREFERRED` state.
12. It then restarts the batch containing only `broker-7`. It waits for it to have `READY` and then `LEADING_ALL_PREFERRED` state.
13. It then restarts the batch containing only `broker-6`. It times out waiting for it to have `READY` state because it's still performing log recovery.
14. The roller retries waiting for `broker-6` to have `READY` state for a number of times and results in the following context:
```
         nodeRef: broker-6/6
         nodeRoles: broker
         state: RECOVERING
         lastTransition: 987456
         restartReason:
         numRestartAttempts: 1
         numReconfigAttempts: 0
         numRetries: 10
```
15. The `maxRetries` of 10 is reached for `broker-6`, therefore the roller throws `UnrestartableNodesException` and the reconciliation fails. The operator logs the number of remaining segments and logs to recover.
16. When the next reconciliation starts, all the nodes are observed and their contexts are updated. `broker-6` node has finished performing log recovery therefore have `READY` state. All nodes have `READY` state and no reason to restart except `broker-9` and `broker-10`.
17. Broker nodes that have no reason to restart are checked if their configurations have been updated. The `min.insync.replicas` has been updated to 1 therefore the roller sends a request containing the configuration update to the brokers and then transitions nodes' state to `RECONFIGURED`. 
18. Observe the broker nodes that have configuration updated, and wait until they have `LEADING_ALL_PREFERRED` state.
19. The roller considers restarting `broker-10` and `broker-9` as they still have `MANUAL_ROLLING_UPDATE` reason. 
20. It sends requests to describe all the topic partitions and their `min.insync.replicas` configuration and finds that all topic partitions are fully replicated. 
21. The roller create 2 batches with a single node in each because `broker-10` and `broker-9` share topic partition, "topic-A": 
22. It then restarts the batch containing `broker-10`. It waits for it to have `READY` and then `LEADING_ALL_PREFERRED` state. The same is repeated for the batch containing `broker-9`.
23. All nodes have `READY` or `LEADING_ALL_PREFERRED` and no exception was thrown therefore the reconciliation completes successfully.

### Switching from the old KafkaRoller to the new KafkaRoller

The new KafkaRoller will only work with KRaft clusters therefore when running in Zookeeper mode, the current KafkaRoller will be used. Kafka CR's `KafkaMetadataState` represents where the metadata is stored for the cluster. It is set to `KRaft` when a cluster is fully migrated to KRaft or was created in KRaft mode. `KafkaReconciler` class will be updated to switch to the new roller based on this state. This means the old KafkaRoller will be used during migration of existing clusters from Zookeeper to KRaft mode and the new roller is used only after the migration is completed and for new clusters created in KRaft mode.

### Future improvement

- We are not looking to solve the potential race condition between KafkaRoller and Cruise Control rebalance activity right away but this is something we can solve in the future. An example scenario that could cause this race:
   Let's say we have a 5 brokers cluster, `min.insync.replicas` for topic partition foo-0 is set to 2. The possible sequence of events that could happen is:
   - Broker 0 is down due to an issue and the ISR of foo-0 partition changes from [0, 1, 2] to [1 , 2]. In this case producers with acks-all still can produce to this partition.
   - Cruise Control sends `addingReplicas` request to reassign partition foo-0 to broker 4 instead of broker 2 in order to achieve its configured goal.
   - The reassignment request is processed and foo-0 partition now has ISR [1, 2, 4].
   - Cruise Control sends `removingReplicas` request to un-assign the partition from broker 2.
   - KafkaRoller is performing a rolling update to the cluster. It checks the availability impact for foo-0 partition before rolling broker 1. Since partition foo-0 has ISR [1, 2, 4], KafkaRoller decides that it is safe to restart broker 1. It is unaware of the `removingReplicas` request that is about to be processed.
   - The reassignment request is processed and foo-0 partition now has ISR [1, 4].
   - KafkaRoller restarts broker 1 and foo-0 partition now has ISR [4] which is below the configured minimum in sync replica of 2 resulting in producers with acks-all no longer being able to produce to this partition.
This would likely need its own proposal. 

## Affected/not affected projects

This proposal affects only
the [`strimzi-kafka-operator` GitHub repository](https://github.com/strimzi/strimzi-kafka-operator).

## Compatibility

The new KafkaRoller introduced by this proposal will used only for KRaft based clusters.
This proposal should have no impact on any existing Kafka clusters deployed with ZooKeeper.

## Rejected

- Why not use rack information when batching brokers that can be restarted at the same time?
When all replicas of all partitions have been assigned in a rack-aware way then brokers in the same rack trivially share no partitions, and so racks provide a safe partitioning. However nothing in a broker, controller or cruise control is able to enforce the rack-aware property therefore assuming this property is unsafe. Even if CC is being used and rack aware replicas is a hard goal we can't be certain that other tooling hasn't reassigned some replicas since the last rebalance, or that no topics have been created in a rack-unaware way.
