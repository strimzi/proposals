# Auto-rebalancing on cluster scaling

This proposal is about enabling the Cluster Operator to rebalance the Apache Kafka cluster automatically when the user scales the cluster up/down by adding/removing brokers to/from the cluster.
Such a rebalancing can happen right after the scale up to move some topics' partitions to the newly added brokers or right before the scale down to move partition replicas off the brokers to be removed.

## Current situation

When a user scales a cluster up by adding new brokers, they would like to use the newly added brokers to host already existing topics' partitions and not only for the newly created ones.
At the same time, in order to scale the cluster down, the brokers to be removed must not host topics' partitions otherwise the operation is not allowed (more details on proposal [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md)).
The way the user has to move topics' partitions in the above scenarios is via the Cruise Control integration by using the `KafkaRebalance` custom resource.

Currently, there is no automatic rebalancing when the user scales the cluster up or down.
It means that:

* after scaling the cluster up, in order to rebalance the cluster and move some topics' partitions to the newly added brokers, the user has to manually create a `KafkaRebalance` custom resource using the `spec.mode: add-brokers` mode with the new brokers' IDs set on the `spec.brokers` field.
* before scaling the cluster down, in order to rebalance the cluster and move topics' partitions off the brokers to be removed, the user has to manually create a `KafkaRebalance` custom resource using the `spec.mode: remove-brokers` mode with the IDs of the brokers to be removed set on the `spec.brokers` field.

For more details, the proposal [035](https://github.com/strimzi/proposals/blob/main/035-rebalance-types-scaling-brokers.md) introduced the rebalancing modes mentioned above, which will be also referenced across this proposal.

## Motivation

The rebalancing operation during scaling up and down is a manual two-step process:

* Scaling up: 
   1. Scaling the cluster up by adding brokers
   2. Running a rebalance
* Scaling down: 
   1. Running a rebalance
   2. Scaling the cluster down by removing brokers

In order to simplify the process, it would be great for the user to deal with just scaling the cluster and then having the operator taking care of running a rebalancing operation automatically when needed.

## Proposal

This proposal is about providing the user the possibility to automatically run a rebalancing when they scale the cluster up (by adding brokers) or scale the cluster down (by removing brokers) by changing the `spec.replicas` within a `Kafka` or `KafkaNodePool` custom resource.

### Extending the `Kafka.spec.cruiseControl` field with auto-rebalancing related configuration

The `Kafka` custom resource is extended by adding a new `spec.cruiseControl.autoRebalance` field.
By default, without such field, no auto-rebalancing runs on scaling.

The `spec.cruiseControl.autoRebalance` field allows to specify a list of rebalancing modes with the corresponding configuration to be used.
For each entry in the list, the mode is defined by the `spec.cruiseControl.autoRebalance.mode` field (`add-brokers` or `remove-brokers`) and the corresponding rebalancing configuration is defined as a reference to a "template" `KafkaRebalance` custom resource (read later for more details), by using the `spec.cruiseControl.autoRebalance.template` field as a [LocalObjectReference](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/local-object-reference/).
This field is optional and if not specified, the auto-rebalancing runs with the default Cruise Control configuration.

In the following example, a user can decide to use the same rebalancing configuration when adding or removing brokers.

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
      # using the same rebalancing configuration when adding or removing brokers
      - mode: add-brokers
        template:
          name: my-add-remove-brokers-rebalancing-template
      - mode: remove-brokers
        template:
          name: my-add-remove-brokers-rebalancing-template
```

Furthermore, it is possible to use the default Cruise Control rebalancing configuration by omitting the corresponding field.

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
      # using the default Cruise Control rebalancing configuration when adding or removing brokers
      - mode: add-brokers
      - mode: remove-brokers
```

The user can even decide to use a different configuration when adding or removing brokers.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    # ...
  cruiseControl:
    # ...
    autoRebalance:
      # using specific configuration when adding brokers
      - mode: add-brokers
        template: 
          name: my-add-brokers-rebalancing-template
      # using different configuration when removing brokers  
      - mode: remove-brokers
        template: 
          name: my-remove-brokers-rebalancing-template
```

The user could even decide to have auto-rebalancing only when adding or removing brokers but not for both.
This way provides the greatest flexibility to describe in which case to run the auto-rebalancing and with which configuration.

The auto-rebalance configuration for the `spec.cruiseControl.autoRebalance.template` property in the `Kafka` custom resource is provided through a `KafkaRebalance` custom resource defined as a "template".
That is a `KafkaRebalance` custom resource with the `strimzi.io/rebalance: template` annotation set.
When it is created, the `KafkaRebalanceAssemblyOperator` doesn't run any rebalancing.
This is because it doesn't represent an "actual" rebalance request to get an optimization proposal, but it's just the place where configuration related to auto-rebalancing is defined.
The user can specify rebalancing goals and options, such as `skipHardGoalCheck`, within the resource.
When such a "template" `KafkaRebalance` custom resource is referenced by the `spec.cruiseControl.autoRebalance.template` field for one or more modes, the operator asks to Cruise Control for an optimization proposal with such configuration when scaling up/down happens.
If the `mode` and `brokers` fields are set, they will be just ignored because the resource doesn't represent an "actual" rebalancing.
Even the `rebalanceDisk` field will be ignored because it doesn't make sense to run an intra-broker related rebalancing on brokers being removed/added.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance-template
  annotations:
    strimzi.io/rebalance: template # specifies that this KafkaRebalance is a rebalance configuration template
spec:
  # NOTE: mode and brokers fields, if set, will be just ignored because they are
  #       automatically set on the corresponding KafkaRebalance by the operator
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

When the `spec.cruiseControl.autoRebalance` is set and a specific rebalancing mode refers to a "template" `KafkaRebalance` custom resource, the operator checks that it exists in the first place.
If it doesn't, the operator logs a warning about auto-rebalancing being ignored (as it wasn't set at all) because referring to a non existing "template" `KafkaRebalance` custom resource.
In this case, the scaling process proceeds as usual: the scale up just runs (without any following rebalance) while the scale down could be blocked (because no rebalance ran to get topics' partitions off the brokers to remove).

If the referenced "template" `KafkaRebalance` custom resource exists, the operator automatically creates (or updates) a corresponding "actual" `KafkaRebalance` custom resource when the user scales the cluster, by adding and/or removing brokers.
The operator copies over goals and rebalancing options from the referenced "template" resource to the "actual" rebalancing one and also adds the `spec.mode` and `spec.brokers` to it, based on the scaling operation the user asked for and on which brokers.
This is the reason why the same fields specified in the "template" `KafkaRebalance` custom resource are ignored.
The "actual" `KafkaRebalance` custom resource will be named as `<my-cluster-name>-auto-rebalancing-<mode>` where the `<my-cluster-name>` part comes from the `metadata.name` in the `Kafka` custom resource, and the `<mode>` is referring to which rebalancing has to run (add or remove brokers).

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-add-brokers
  finalizers:
    - strimzi.io/auto-rebalancing
spec:
  mode: add-brokers
  brokers: [3,4]
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

The operator also sets a finalizer, named `strimzi.io/auto-rebalancing`, on the "actual" `KafkaRebalance` custom resource.
This is needed to avoid the user, or any other tooling, to delete the resource while the auto-rebalancing is still running.
The finalizer is removed at the end of the auto-rebalancing process, with or without errors, allowing the "actual" `KafkaRebalance` custom resource deletion by the operator itself.

The user has the flexibility to specify the [goals and the other rebalancing related configurations](https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaRebalance-reference) the same way they already do today on a `KafkaRebalance` custom resource.
This allows the user to not rely on the default Cruise Control goals configuration which would be common for any rebalance without overriding goals.
They could define different goals based on the status of the Kafka cluster or skip hard goals in some scenarios for example.
As already mentioned, specifying goals but more in general customizing the optimization proposal request is already possible in the current usage of a `KafkaRebalance` custom resource so it should be allowed for the auto-rebalancing as well.

The idea of using a "template" `KafkaRebalance` custom resource could be also extended to the manual rebalancing operations started by the user when they create a new `KafkaRebalance` custom resource.
It would mean a change on the `KafkaRebalance` custom resource by adding a new `spec.template` field to reference the rebalancing "template" and deprecating all the other fields used today (i.e. `spec.goals`, ...).
Another option could be leaving both the ways living together.
If this proposal is considered valid, doing such a change on the `KafkaRebalance` custom resource would help the proposal itself because the operator should not copy over the entire configuration from the "template" resource to the actual rebalance resource but just set the configuration reference.
Of course, such a change would need a different proposal to be approved before this one.

It is also worth mentioning that a "template" `KafkaRebalance` custom resource doesn't add much complexity to the operator in terms of watching, reacting and processing.
The `KafkaRebalanceAssemblyOperator` is just going to ignore such a resource when it's created, updated or deleted.
It can be only referenced by an "actual" `KafkaRebalance` custom resource so its creation doesn't trigger any action by the operator.
The operator only reads the "template" `KafkaRebalance` custom resource and generates a new corresponding "actual" `KafkaRebalance` custom resource when the cluster is being scaled.

### Extending the `Kafka.status` field with auto-rebalancing related state

The auto-rebalancing, together with the scaling action, can take several reconciliations to finish.
For this reason, taking into account cluster operator might be restarted as well, we need to save some information about the state of the auto-rebalancing operation.
The `Kafka` custom resource is extended by adding a new `status.autoRebalance` field.

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
      # ...
status:
  autoRebalance:
    lastTransitionTime: # timestamp of last transition during the auto-rebalancing flow
    state: # if a scaling operation is running or what's the state of the current rebalancing
    modes:
      - mode: add-brokers
        brokers: # list with brokers' IDs to add (as part of the scaling up)
      - mode: remove-brokers
        brokers: # list with brokers' IDs to remove (as part of the scaling down)
```

The `state` field provides the current state of the rebalancing process.
Its value is defined by a Finite State Machine (FSM) described in the next sections which evolves even based on the status of the corresponding "actual" `KafkaRebalance` custom resource that was created for that purpose. 
The `lastTransitionTime` field provides the timestamp of the latest auto-rebalancing state update.
The `modes` field lists the enabled rebalancing modes, as defined through the `spec.cruiseControl.autoRebalance.mode` field, with the corresponding `brokers` list containing the brokers' IDs that are part of the scaling (up/down) actions to be useful across reconciliations.
The `modes` field is used to store a list of brokers' IDs that were part of the scaling (up/down) action that triggered an auto-rebalance to be used by the operator across reconciliations.
For each enabled rebalancing mode, as defined through the `spec.cruiseControl.autoRebalance.mode` field, it contains either:

* the brokers' IDs relevant to the current ongoing auto-rebalance, or
* the brokers' IDs relevant to a queued auto-rebalance (if a previous auto-rebalance is still in progress)

The brokers being added/removed are provided by the `KafkaClusterCreator` and stored within the `KafkaAssemblyOperator` for later use in the auto-rebalancing.

### Auto-rebalancing execution

Before describing the process in more details, it's worth mentioning how scaling up and down happens today within the Cluster Operator reconciliation flow and how the auto-rebalancing fits into it.

Both the operations involve the `StrimziPodSet` controller which takes care of adding (scale up) or removing (scale down) pods by communicating with the Kubernetes API.

The scaling up is triggered within the `KafkaReconciler.reconcile()` method by the `podSet()` call which updates the involved `StrimziPodSet`(s) so that the corresponding controller will take care of spinning up the newly added brokers.
The scaling operation ends during the same reconciliation, because after having the `StrimziPodSet` controller asking for new pods through the Kubernetes API, the subsequent calls to `podsReady()`, `serviceEndpointReady()` and `headlessServiceEndpointReady()` are waiting for the newly added pods, together with the corresponding services, to be ready.
It means that the auto-rebalancing can start in the same reconciliation, being sure that the newly added pods are available for the upcoming rebalancing operation.
Of course, the rebalancing will take time going through several reconciliations.

For the scaling down, if the brokers to be removed are hosting topics' partitions, the operation is reverted before calling the `KafkaReconciler.reconcile()` method which is called with the original replicas count as if the cluster was never scaled down.
In this case, the auto-rebalancing can start in the same reconciliation to take topics' partitions off but going through several reconciliations.
When the auto-rebalancing ends, the triggered pods deletion takes some time but it doesn't have any impact on the rebalancing already done.
If the brokers to be removed are not hosting any topics' partitions, the `KafkaReconciler.reconcile()` method will be called with the new replicas count and the scaling down is triggered by the `scaleDown()` call which updates the involved `StrimziPodSet`(s) so that the corresponding controller will take care of removing the brokers.
Of course, if the scale down check doesn't fail, it means that no topics' partitions are hosted on the brokers to be removed and no auto-rebalancing is needed.

The proposal is about having a new `KafkaAutoRebalancingReconciler` class with its own `reconcile()` method which is called by the `KafkaAssemblyOperator` as one of the last steps in the reconciliation loop, so that:

* we can get all the information about the brokers removed or added, which are available through the `KafkaClusterCreator` class (read later for more details).
* we are sure that, in case of scaling up, the Cruise Control pod was restarted and it's now up and running again so it can accept new rebalancing requests.
Of course the `KafkaAutoRebalancingReconciler.reconcile()` method also checks the status of an ongoing rebalancing across several reconciliations.

It's also important taking into account that a user, by using `KafkaNodePool`(s), could remove and add brokers at the same time, so actually triggering both scaling down and up.
In terms of scaling, the reconcile loop has the scale down triggered before the scaling up but it could be skipped because the brokers to be removed are still hosting topics' partitions. In such case the scaling up runs before.
In terms of rebalancing, after the scaling up happens, the rebalancing for brokers to be removed takes the precedence on the rebalancing for newly added brokers. They are executed sequentially and not in parallel.

### Auto-rebalancing Finite State Machine (FSM)

The auto-rebalancing mechanism runs through a Finite State Machine (FSM) made by the following states:

* **Idle**: Initial state with a new auto-rebalancing initiated when scaling down/up operations were requested. This is also the ending state after an auto-rebalancing completed successfully or failed.
* **RebalanceOnScaleDown**: a rebalancing related to a scale down operation is running.
* **RebalanceOnScaleUp**: a rebalancing related to a scale up operation is running.

The FSM states' transitions are the following:

* from **Idle** to:
  * **RebalanceOnScaleDown**: if a scale down operation was requested. This transition happens even if a scale up was requested at the same time but the rebalancing on scaling down has the precedence. The rebalancing on scale up is queued. They will run sequentially.
  * **RebalanceOnScaleUp**: if only a scale up operation was requested. There was no scale down operation requested.
* from **RebalanceOnScaleDown** to:
  * **RebalanceOnScaleDown**: if a rebalancing on scale down is still running or another one was requested while the first one ended.
  * **RebalanceOnScaleUp**: if a scale down operation was requested together with a scale up and, because they run sequentially, the rebalance on scale down had the precedence, was executed first and completed successfully. We can now move on with rebalancing for the scale up.
  * **Idle**: if only a scale down operation was requested, it was executed and completed successfully or failed.
* from **RebalanceOnScaleUp**:
  * **RebalanceOnScaleUp**: if a rebalancing on scale up is still running or another one was requested while the first one ended. 
  * **RebalanceOnScaleDown**: if a scale down operation was requested, so the current rebalancing scale up is stopped (and queued) and a new rebalancing scale down is started. The rebalancing scale up will be postponed.
  * **Idle**: if a scale up operation was requested, it was executed and completed successfully or failed.

On each reconciliation, the logic will be like this:

1. The `KafkaClusterCreator` creates the `KafkaCluster` instance and it includes:

   * checking the scale down (if brokers cannot be removed because they host topics' partitions) and reverting replicas count if it fails.
   * collecting the nodes reverted from scale down and the nodes added by scale up. Let's call these lists as `toBeRemovedNodes` and `addedNodes`.

2. The `KafkaReconciler.scaleDown()` scales the cluster down if the check didn't fail, or isn't aware of any scale down at all because it was just reverted within the `KafkaClusterCreator`.

3. The `KafkaReconciler.podSet()` scales the cluster up, ensuring newly added brokers are ready, together with all the corresponding services, before starting an auto-rebalancing.

4. Taking into account the changes needed for the capacity configuration due to newly added brokers (or even because of the removed ones, if the check didn't fail), the Cruise Control pod is rolled at this point.

5. The `KafkaAutoRebalancingReconciler.reconcile()` being involved and using the FSM to run rebalancing operations, if requested and in the following order:

   * rebalancing on scale down
   * rebalancing on scale up

What happens for each reconciliation depends on the auto-rebalancing FSM as well as the "actual" `KafkaRebalance` custom resource status.

The `KafkaAutoRebalancingReconciler.reconcile()` loads the `Kafka.status.autoRebalance` content:

* `state`: is the FSM state.
* `lastTransitionTime`: when the transition to that state happened.
* `modes`: enabled modes (`remove-brokers` and/or `add-brokers`) with corresponding brokers lists if an auto-rebalancing requested.

The FSM is initialized based on the `state` field.

The `Kafka.status.autoRebalance.modes[add-brokers|remove-brokers].brokers` list is filled or updated based on its current content and what's coming from `toBeRemovedNodes` and `addedNodes`.

Check the `toBeRemovedNodes`:

  * if empty
    * no further action and stay with the current `Kafka.status.autoRebalance.modes[remove-brokers].brokers` list (which could mean being empty/not existing too).
  * if not empty
    * update the `Kafka.status.autoRebalance.modes[remove-brokers].brokers` by using the full content from the `toBeRemovedNodes` list which always contains the nodes involved in a scale down operation. This covers the case that while rebalancing scale down running, the user requests an update on an already running one.

Check the `addedNodes`:

  * if empty
    * no further action and stay with the current `Kafka.status.autoRebalance.modes[add-brokers].brokers` list (which could mean being empty/not existing too)
  * if not empty
    * update the `Kafka.status.autoRebalance.modes[add-brokers].brokers` by producing a consistent list with its current content and what is in the `addedNodes` list. This covers the case that while rebalancing scale down running, the user requests a new scale up or an update on an already queued one. Or even the case that while rebalancing scale up running, the user requests an update on an already running one.

During the FSM processing, the `Kafka.status.autoRebalance.modes[add-brokers|remove-brokers].brokers` list is compared with the corresponding `KafkaRebalance.spec.brokers` list for the current rebalancing to determine if there were changes.
Those changes could mean several use cases: a new scale down/up requested while another one is running, a scale up requested while a scale down is running and so on.

Let's see what happens during the auto-rebalancing process when the FSM starts from a specific state and depending on the brokers list updates.
Remember that starting a rebalancing always means creating a corresponding "actual" `KafkaRebalance` customer resource from the "template" one and applying the `strimzi.io/rebalance-auto-approval: true` annotation.

#### Idle

This state is set since the beginning when a `Kafka` custom resource is created with the `spec.cruiseControl.autoRebalance` field for a future auto-rebalancing on scaling the cluster.
It is also the end state of a previous successfully completed or failed auto-rebalancing.
In this state, the operator removes the finalizer and deletes the corresponding "actual" `KafkaRebalance` custom resource.

From this state:

* if the previous auto-rebalancing was completed successfully, the user can trigger a new auto-rebalancing by scaling down/up the cluster again.
* if the previous auto-rebalancing failed, the operator logs the error and the user has to fix the issues which caused the failure. On the next reconciliation, in case of scale down, the operator will try to auto-rebalance again because the brokers were not removed and they still trigger an auto-rebalancing. In case of scale up, there won't be any further action by the operator itself. The user can deal with the rebalancing on their own with the usual `KafkaRebalance` custom resource manual creation.

Furthermore, in this state, it means no `Kafka.status.autoRebalance.modes[*].brokers` lists are available for any mode (they were removed when a previous auto-rebalancing completed/failed eventually).

Following the actions taken and corresponding transitions:

* if there is a queued rebalancing scale down (`Kafka.status.autoRebalance.modes[remove-brokers]` exists), start the rebalancing scale down and transition to **RebalanceOnScaleDown**.
* If no queued rebalancing scale down but there is a queued rebalancing scale up (`Kafka.status.autoRebalance.modes[add-brokers]` exists), start the rebalancing scale up and transition to **RebalanceOnScaleUp**.
* No queued rebalancing (so no scale down/up requested), stay in **Idle**.

#### RebalanceOnScaleDown

In this state, a rebalancing scale down is running.

Checking the current `KafkaRebalance` status:

* if `Ready`, the rebalancing scale down was successful.
  * check if `Kafka.status.autoRebalance.modes[remove-brokers].brokers` was updated compared to the current running rebalancing scale down. If different, start the rebalancing and stay in  **RebalanceOnScaleDown**.
  * If no changes, if there is a queued rebalancing scale up (`Kafka.status.autoRebalance.modes[add-brokers]` exists), start the rebalancing and transition to **RebalanceOnScaleUp**, or just transition to **Idle** if there is not, clean `Kafka.status.autoRebalance.modes` and delete the "actual" `KafkaRebalance` custom resource.
* if `PendingProposal`, `ProposalReady` or `Rebalancing`, the rebalancing scale down is still running.
  * check if `Kafka.status.autoRebalance.modes[remove-brokers].brokers` was updated compared to the current running rebalancing scale down. If different, update the corresponding `KafkaRebalance` in order to take into account the updated brokers list and refresh it by applying the `strimzi.io/rebalance: refresh` annotation. Stay in **RebalanceOnScaleDown**.
  * If no changes, no further action and stay in **RebalanceOnScaleDown**.
* if `NotReady`
  * the rebalancing scale down failed, transition to **Idle** and also removing the corresponding mode and brokers list from the status. The operator also deletes the "actual" `KafkaRebalance` custom resource.

If, during an ongoing auto-rebalancing, the `KafkaRebalance` custom resource is not there anymore on the next reconciliation, it could mean the user deleted it while the operator was stopped/crashed/not running.
In this case, the FSM will assume it as `NotReady` so falling in the last case above.

#### RebalanceOnScaleUp

In this state, a rebalancing scale up is running.

Check the current `KafkaRebalance` status:

* if `Ready`, the rebalancing scale up was successful.
  * if there is a queued rebalancing scale down (`Kafka.status.autoRebalance.modes[remove-brokers]` exists), start the rebalancing scale down and transition to **RebalanceOnScaleDown**.
  * If no queued rebalancing scale down, check if `Kafka.status.autoRebalance.modes[add-brokers].brokers` was updated compared to the current running rebalancing scale up. If no changes, no further actions but just transition to **Idle**, clean `Kafka.status.autoRebalance.modes` and delete the "actual" `KafkaRebalance` custom resource. If different, update the corresponding `KafkaRebalance` in order to take into account the updated brokers list and refresh it by applying the `strimzi.io/rebalance: refresh` annotation. Stay in **RebalanceOnScaleUp**.
* if `PendingProposal`, `ProposalReady` or `Rebalancing`, the rebalancing scale up is still running.
  * if there is a queued rebalancing scale down (`Kafka.status.autoRebalance.modes[remove-brokers]` exists), stop the current rebalancing scale up by applying the `strimzi.io/rebalance: stop` annotation on the corresponding `KafkaRebalance`. Start the rebalancing scale down and transition to **RebalanceOnScaleDown**.
  * If no queued rebalancing scale down, check if `Kafka.status.autoRebalance.modes[add-brokers].brokers` was updated compared to the current running rebalancing scale up. If no changes, no further actions. If different, update the corresponding `KafkaRebalance` in order to take into account the updated brokers list and refresh it by applying the `strimzi.io/rebalance: refresh` annotation. Stay in **RebalanceOnScaleUp**.
* if `NotReady`
  * the rebalancing scale up failed, transition to **Idle** and also removing the corresponding mode and brokers list from the status. The operator also deletes the "actual" `KafkaRebalance` custom resource.

If, during an ongoing auto-rebalancing, the `KafkaRebalance` custom resource is not there anymore on the next reconciliation, it could mean the user deleted it while the operator was stopped/crashed/not running.
In this case, the FSM will assume it as `NotReady` so falling in the last case above.

## Affected/not affected projects

This proposal involves the Cluster Operator only.
More specifically, the development is focused on adding a new `KafkaAutoRebalancingReconciler` class and using its logic within the `KafkaAssemblyOperator`.
Even the `KafkaClusterCreator` should be changed in order to collect the brokers to be removed, in case of scale down, in a dedicated list for the auto-rebalancing mechanism to proceed.

## Compatibility

There are no backward compatibility issues.
The auto-rebalancing on scaling up and down is optional.
The user has to define the `spec.cruiseControl.autoRebalance` field on the `Kafka` custom resource in order to have the operator running the rebalancing automatically.
Without that additional field, the scaling actions are not influenced by any rebalancing operations to take place.
If the user wants, the rebalancing can be still done with the manual two steps process as today.
The same way, the new `status.autoRebalance` field in the `Kafka` custom resource doesn't show up at all if the auto-rebalancing feature is not used.

## Rejected alternatives

### Alternative 1

Like the proposed solution, this alternative is still about leveraging the auto-creation of a `KafkaRebalance` custom resource and the `KafkaRebalanceAssemblyOperator` to take care of it without any further changes.
The main difference is related to the rebalancing configuration, because it just uses the Cruise Control defaults.

Pros:

* Simpler compared to the proposed solution because no "template" `KafkaRebalance` custom resource is involved.
  
Cons:

* No way to customize the configuration (options, goals, ...) for getting a rebalancing proposal.
* No way to differentiate rebalancing configuration between adding and removing brokers.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    # ...
  cruiseControl:
    # ...
    autoRebalance:
      modes:
      - add-brokers
      - remove-brokers
```

### Alternative 2

Like the proposed solution, this alternative is still about leveraging the auto-creation of a `KafkaRebalance` custom resource and the `KafkaRebalanceAssemblyOperator` to take care of it without any further changes.
The main difference is related to the rebalancing configuration, because it's possible to customize it by including it directly into the `Kafka` custom resource.

Pros:

* Simpler compared to the proposed solution because no "template" `KafkaRebalance` custom resource is involved.
* It allows to customize the configuration (options, goals, ...) for getting a rebalancing proposal.
* It allows to differentiate rebalancing configuration between adding and removing brokers.

Cons:

* Increasing the content size of the `Kafka` custom resource which could be a limit for future additions (even not related to rebalancing).

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
      # using the same rebalancing configuration when adding or removing brokers ... a lot of duplication
      - mode: add-brokers
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
      - mode: remove-brokers
        goals:
          - CpuCapacityGoal
          - NetworkInboundCapacityGoal
          - DiskCapacityGoal
          - RackAwareGoal
          - MinTopicLeadersPerBrokerGoal
          - NetworkOutboundCapacityGoal
          - ReplicaCapacityGoal
        skipHardGoalCheck: true
```

### Alternative 3

This alternative is about not leveraging a `KafkaRebalance` custom resource, automatically created by the Cluster Operator, but having a more tight integration between the Cruise Control rebalancing FSM (Finite State Machine) and the `KafkaAssemblyOperator`.
It means that we should factor out the Cruise Control rebalancing FSM (and all the corresponding interaction with the Cruise Control API) from the current `KafkaRebalanceAssemblyOperator`.
This way, the Cruise Control rebalancing FSM could be used directly by both the `KafkaRebalanceAssemblyOperator` for the user initiated rebalancing as today and the `KafkaAssemblyOperator` for the auto-rebalancing feature (a reference example could be the KRaft migration FSM).

In the first scenario, the `KafkaRebalanceAssemblyOperator` should be able to interact directly with the Cruise Control rebalancing FSM by getting any rebalancing parameter from the `KafkaRebalance` custom resource created by the user.
In the second scenario, the `KafkaAssemblyOperator` should be able to interact directly with the Cruise Control rebalancing FSM by getting any rebalancing parameter from a "template" `KafkaRebalance` custom resource or from the Cruise Control defaults, without any need for an auto-created `KafkaRebalance` custom resource which is the main goal of this alternative.

Pros:

* No automatic `KafkaRebalance` custom resource creation.
* No user confusion between auto-created and manually-created `KafkaRebalance` custom resources.

Cons:

* More complex and more work on factoring out the Cruise Control rebalancing FSM from the `KafkaRebalanceAssemblyOperator`.
* Not leveraging the already ready-to-use `KafkaRebalanceAssemblyOperator` through a `KafkaRebalance` custom resource.
* `KafkaAssemblyOperator` strongly tight to the Cruise Control rebalancing FSM with additional complexity for testing.
