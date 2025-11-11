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


### Known bugs

- KafkaRoller takes a long time to reconcile combined nodes if they are all in Pending stateMore information can be found [here](https://github.com/strimzi/strimzi-kafka-operator/issues/9426).
- The quorum health check relies on the `controller.quorum.fetch.timeout.ms` configuration, which is determined by the desired configuration values. However, during certificate reconciliation or manual rolling updates, KafkaRoller doesn't have access to these desired configuration values since they shouldn't prompt any configuration changes. As a result, the quorum health check defaults to using the hard-coded default value.

### Shortcomings of the existing code:

- Hard to reason about what's going on. 
The code is complex to understand and it's not easy to determine why a pod was restarted from logs that tend to be noisy.
- Hard to fix bugs, add features or generally refactor without introducing bugs.
The code for KafkaRoller does not easily allow growth and adding new functionality such as batch rolling due to its complexity, nested conditions and insufficient test coverage for many edge cases.
- Hard to add new tests.
The way unit tests are currently structured and tightly coupled with sources of information used to determine whether to roll nodes makes it challenging to improve and extend the test to increase the coverage.

### Potential improvements:

- Rolling brokers in parallel.
Although it is safe and straightforward to restart one broker at a time, this process is slow in large clusters ([related issue](https://github.com/strimzi/strimzi-kafka-operator/issues/8547)).
- Taking account of partition preferred leadership.
This would cause less leadership changes, consequently impacting clients less because they would not need to reconnect every time.
- Slowing down rolling update.
Being able to configure how long to wait between restart of brokers is useful for clusters that are extra sensitive to rolling updates.


## Motivation

The rolling of Kafka pods is a key role of the Strimzi cluster operator and therefore any changes to it should be made with as little risk as possible.
Today refactoring the KafkaRoller is an inherently risky thing to do and has resulted in bugs being introduced while other changes were made.
At the same time, since it is a key part of Strimzi we should expect that beyond the ones listed in this proposal it's likely that other new bugs or new features might come up.

We need to evolve the KafkaRoller in some way so that in future we can fix bugs and add new features with a higher level of certainty that we aren't introducing new bugs.

While we can address some of the known bugs and potential new features without changing the way the KafkaRoller works, there are some changes that require a fundamental change to the KafkaRoller so that it no longer considers Kafka pods one by one.

## Proposal

The objective of this proposal is to introduce a new KafkaRoller with a more structured design resembling a finite state machine.
The primary motivation for this is to make it easier to reason about and rigorously test the code, allowing it to continue to be evolved in future with lower levels of risk than what we have today.
The finite station machine design will also be implemented keeping in mind the following features:
- Ability to roll brokers in parallel to speed up updates in large clusters.
- After restarting a broker, allow it to lead the partitions it is the preferred leader for.
This will help reduce the impact on clients.
- Ability to restart controller nodes in parallel when needed, to help recovering controller quorum.
- Add configurable wait time between restarts to allow slowing down rolling updates.

The new roller will have the same behaviour of the current roller but with the additional features above, however, the implementation will be different following the finite state machine design.

In this proposal term `node` refers to the Kafka process that is either controller, broker or combined.
When the term `pod` is used, it refers to the Kubernetes `Pod` resource where a Kafka process is running.

## Implementation details

In the existing KafkaRoller a lot of the complexity comes from handling nodes that are in states other than Ready.
The Finite State Machine simplifies this by associating state with each node based on three sources (Kubernetes API, KafkaAgent, and Kafka Admin API) and then having explicit actions for each set of observed states. The KafkaRoller executes the actions until each node reaches the desired state, or the maximum number of attempts is reached.

The state sources will be abstracted so that the state machine is not dependent on their specifics as long as it's getting the information it needs. These abstractions will enable much better unit testing.

### States
- UNKNOWN (initial/default)
- NOT_RUNNING
- NOT_READY
- RECOVERING
- READY (desired state)

### Observation sources and information collected
- Kubernetes API
  - Pod is not Running and is one of CrashLoopBackOff, ImagePullBackOff, ContainerCreating and PendingAndUnschedulable
  - Pod is Running but lacking Ready status
  - Pod is Running and Ready

- KafkaAgent: It collects and expose Kafka metric [Broker State](https://github.com/apache/kafka/blob/3.7/metadata/src/main/java/org/apache/kafka/metadata/BrokerState.java)
  - Broker state is 2 (RECOVERY)
  - Broker state is not 2 (RECOVERY)

- Kafka Admin API
  - Leading all preferred partitions
  - Not leading all preferred partitions

### Actions
- `Observe` - This is a function to transition a node's state.
- `Wait and observe` - This is to repeat the `Observe` function until the desired state or timeout is reached.
- `Restart` - Delete a pod via Kubernetes API and then `Wait and Observe`. This action is followed by a preferred leader election trigger via Kafka Admin API for a broker node if it is not leading its preferred replicas yet.
- `Reconfigure` - Apply configuration updates via Kafka Admin API and then `Wait and Observe`.
- `No action` - This means we reached the desired state after taking one of the above actions or no action is needed.

### Observations -> States Map
| KubeAPI | KafkaAgent | States |
| :--------------- | :--------------- | :---------------
| - | - | UNKNOWN
| Pod is not Running | - | NOT_RUNNING
| Pod is Running but lacking Ready status | Broker state != 2 | NOT_READY
| Pod is Running but lacking Ready status | Broker state == 2 | RECOVERING
| Pod is Running and Ready | - | READY

### States -> Actions Map
| States | Actions
| :--------------- | :---------------
| UNKNOWN | `Observe`
| NOT_RUNNING | `Restart` OR `Wait and observe`
| RECOVERING | `Wait and observe`
| NOT_READY | `Restart` OR `Wait and observe`
| READY | `Restart` OR `Reconfigure` OR `No action`

Some states map to multiple possible actions, but only one of them is taken based on the other conditions.

`UNKNOWN` nodes will be observed.
This is the initial/default state before observation.

`NOT_RUNNING` nodes will restart only if the pod has an old revision (is out of date).
This is because, if the node is not running at all, then restarting it likely won't make any difference unless the node is out of date.
For example, if a pod is in pending state due to misconfigured affinity rule, there is no point restarting this pod again or restarting other pods, because that would leave them in pending state as well.
If the user then fixes the misconfigured affinity rule, then we should detect that the pod has an old revision, therefore should restart it so that the pod is scheduled correctly and runs.

`RECOVERING` nodes will be waited and observed only.
A Kafka node can take a long time to become ready while performing log recovery and it's not easy to determine how long it might take.
Therefore, it's important to avoid restarting the node during this process, as doing so would restart the entire log recovery, potentially causing the node to enter a loop of continuous restarts without becoming ready.
Moreover, while a node is in recovery, no other node should be restarted, as this could impact cluster availability and affect the client.

`NOT_READY` nodes will be restarted if they have a restart reason and have not been restarted yet.
If it is not ready after being restarted already, we don't want to restart any other nodes to avoid taking down more nodes.

`READY` nodes will be restarted if they have a restart reason.
If they don't have a restart reason but need to be reconfigured, they will be reconfigured.
If no reconfiguration is needed, then no action will be taken on these nodes.

If all nodes reach the desired state, the reconciliation will succeed, otherwise it will fail.
There are also some conditions that could result in early termination of the process and immediately failing the reconciliation.
This will be explained more later.

### High level flow diagram describing the flow of the states
![The new roller flow](./images/06x-new-roller-flow.png)

### State machine cycles

As previously mentioned, the process for nodes will be repeated unless the maximum attempt is reached, in which case the reconciliation fails.
The maximum number of attempts is hard-coded to 10 in the current roller.
It will be the same for the new roller, however this value will be further broken down.
The new roller will add 2 other hard-coded maximum attempt values.
One is for the maximum number of restarts and one is for the maximum number of reconfigurations that can be taken on each node.
This is because we want to limit how many times a node can be restarted in each reconciliation because restarting a node 10 times is not productive.
Also, in the current roller, if we failed to reconfigure a node, we immediately restart it.
Reconfiguration can fail sometimes due to transitive error so it would be useful to retry the reconfiguration a few times before we decide to restart a node.

In each reconciliation, the number of attempts is tracked for each node.
The number of attempts is how many times the overall process is repeated per node because of not reaching the desired state and the number of restarts is how many times a node is actually restarted.
If any node has reached the maximum number of attempts or restarts, the reconciliation will fail.
If the maximum number of reconfiguration is reached, then the node will be marked to restart but will not fail the reconciliation.
When a new reconciliation starts, the tracked number of actions taken on nodes will be reset.

The current roller also fails the reconciliation in the following situation to avoid continuing the process on more nodes:
- Cannot connect to KafkaAgent to check broker state metrics (KafkaAgent is crucial when determining safety before rolling nodes, so there is no point to try to progress further, if we cannot connect to the KafkaAgent).
- Pod is not Running and does not have an old revision (this will prevent the roller from restarting more nodes, which could result in making them stuck as well and bring down the entire cluster).

### Batch rolling

Batch rolling is one of the major features that the new roller is introducing.
The proposed algorithm is to group broker nodes without common partitions together for parallel restart while maintaining availability.

The proposed algorithm does not take rack information into account and the reason for this is explained in the `Rejected Alternatives` section.
When batching brokers without rack awareness is that batch size would likely decrease as the roller progresses and potentially result in brokers that cannot be batched together so they would be restarted one by one.
However, the majority of brokers would likely to get restarted in parallel and that would still speed the rolling in large clusters significantly.

There is also an interesting future improvement that can make the batch rolling more effective optimizing with Cruise Control.
This improvement will not be in the scope of this proposal but included in `Future Improvements` section.

The batching algorithm only applies to broker nodes, however, the capability to roll nodes in parallel will be used for controller nodes as well.
This is needed when a controller quorum is in a bad state.
[#9426](https://github.com/strimzi/strimzi-kafka-operator/issues/9426) mentioned in the `Known Issues` section, is an example of why this feature is important for controller nodes.
The new roller will check if there are multiple controller nodes not working affecting the quorum, and restart them in parallel to help recover it.

### Configurability

The following are the configuration options for the new roller.
Some of them are existing configurations that are used in the same way as the current roller.
The new configurations are `highlighted`.
If exposed to the user, the user can configure it via `STRIMZI_` environment variables.
Otherwise, the operator will hard code them to the default values:

| Configuration | Default value | Exposed to user | Description |
|:--------------|:--------------|:----------------| :-----------|
| maxAttempts | 10 | No | Maximum number of times a node can be attempted after not reaching the desired state.  This is checked against the node's `numAttempts`.                                                                                                                      |
| operationTimeoutMs | 60 seconds | Yes | Maximum amount of time we will wait for nodes to transition to `READY` state after an action. This is already exposed to the user via environment variable `STRIMZI_OPERATION_TIMEOUT_MS`. |
| `maxRestartAttempts` | 3 | No | Maximum number of restart attempts per node before failing the reconciliation. This is checked against node's `numRestartAttempts`.                                                                                                               |
| `maxReconfigAttempts`| 3 | No | Maximum number of dynamic reconfiguration attempts per node before restarting the node. This is checked against node's `numReconfigAttempts`. |
| `maxBrokerBatchSize` | 1 | Yes | Maximum number of broker nodes that can be restarted in parallel. It will be exposed to the user via the new environment variable `STRIMZI_MAX_RESTART_BATCH_SIZE`. |
| `dryRunForBatchRolling`| true | Yes | When set to true along with `maxBrokerBatchSize` set to greater than 1, the batched nodes will only be logged but not restarted. This configuration will be exposed to the user via the new environment variable `STRIMZI_DRY_RUN_BATCH_ROLLING`. |
| `postRestartDelayMs` | 0 | Yes | Delay to apply between node(s) restarts to slow down the rolling update. It will be exposed to the user via the new environment variable `STRIMZI_POST_RESTART_DELAY_SECONDS`.|
| `preferredLeaderElectionDelayMs` | 10 seconds | No | Delay to apply after a node restart and before triggering partition leader election. This is to avoid situations where leaders are moving to a newly started node that does not yet have established networking to some outside networks, e.g. through load balancers. |

### Testing

Currently both unit tests and system tests for the roller are limited in terms of the edge cases it covers. There is a plan to improve system tests so that both old and new roller can be tested more rigorously (refers to this issue here). There will also be new system tests added for batch rolling scenarios as part of this proposal implementation.
The way unit tests are currently structured and tightly coupled with sources of information used to determine whether to roll nodes makes it challenging to improve and extend the test to increase the coverage.
The new roller will have completely newly structured unit tests that will cover the current test scenarios as well as more using the abstracted sources of information.

## Feature Gate

The switch from the old roller to the new roller should be controlled by a new feature gate called `UseNewKafkaRoller`.
With this feature gate disabled, the operator will continue using the old KafkaRoller.
With it enabled, the new roller will be used.
The following table shows the expected graduation of the feature gate:
| Phase |  Default state |
|:------|:---------------|
| Alpha | Disabled by default |
| Beta  | Enabled by default  |
| GA    | Enabled by default (without possibility to disable it) |

We expect to remain in Alpha for 2 releases, then in Beta for at least 2 releases, if not more until we are happy that issues are ironed out and the roller is running stable.

Reddit is one of the Strimzi vendors that offered to test the new roller while it's in Alpha and Beta phase.

### Future improvement

In the future, we can optimize Cruise Control's `BrokerSetAwareGoal` to make the batch rolling more efficient when the user enables Cruise Control in their clusters.
This goal operates at the level of broker sets, which may correspond to physical or logical boundaries like racks, data centres, availability zones or custom logical groupings defined by administrators.
It ensures that replicas of a partition are not assigned to the same broker set by spreading them across sets as evenly as possible.
The new roller could then restart brokers in the same set in parallel while maintaining availability of partitions.
This goal can be used for both rack aware and non rack aware clusters, compared to `RackAwareGoal` which relies on Kafka's built-in `broker.rack` property.


This solution still has the limitation mentioned in the `Rejected Alternatives` section, that we can't be certain that other tooling hasn't reassigned some replicas since the last rebalance.
In this case, the proposed algorithm can be used to check that brokers in the same set have no common partitions.
This can be discussed further in the future.

## Affected/not affected projects

This proposal affects only
the [`strimzi-kafka-operator`](https://github.com/strimzi/strimzi-kafka-operator).

## Compatibility

The new KafkaRoller introduced by this proposal will be used only for KRaft based clusters.
The purpose of this proposal is to maintain the same behaviour of the old roller with a finite state machine design while adding the new features.

## Rejected

- Why not use rack information when batching brokers that can be restarted at the same time?
When all replicas of all partitions have been assigned in a rack-aware way then brokers in the same rack trivially share no partitions, and so racks provide a safe partitioning.
However nothing in a broker, controller or cruise control is able to enforce the rack-aware property therefore assuming this property is unsafe.
Even if CC is being used and rack aware replicas is a hard goal we can't be certain that other tooling hasn't reassigned some replicas since the last rebalance, or that no topics have been created in a rack-unaware way.

- Why not refactor the existing KafkaRoller rather than rewriting as a new component?
It is also not particularly easy to unit test which resulted in insufficient test coverage for many edge cases, making it challenging to refactor safely.  The past experience of adding non trivial changes to it has proven to be expensive and more risky of introducing bugs because of tightly coupled logics and lack of testability.
Rewriting with a more structured design that is easier to follow and test and then released under a `Feature gate` is safer and allows the community to try and test it until we are confident.