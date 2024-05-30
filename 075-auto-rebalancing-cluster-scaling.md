# Auto-rebalancing on cluster scaling

This proposal is about enabling the Cluster Operator to rebalance the Apache Kafka cluster automatically when the user scales the cluster up/down by adding/removing brokers to/from the cluster.
Such a rebalancing can happen right after the scale up to move some partition replicas to the newly added brokers or right before the scale down to move partition replicas off the brokers to be removed.

## Current situation

When a user scales a cluster up by adding new brokers, they would like to use the newly added brokers to host already existing partition replicas and not only for the newly created ones.
At the same time, in order to scale the cluster down, the brokers to be removed don't have to host partition replicas otherwise the operation is not allowed at all (more details on proposal [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md)).
The way the user has to move partition replicas in the above scenarios is via the Cruise Control integration by using the `KafkaRebalance` custom resource.

Currently, there is no automatic rebalancing when the user scales the cluster up or down.
It means that:

* after scaling the cluster up, in order to rebalance the cluster and move some partition replicas to the newly added brokers, the user has to manually create a `KafkaRebalance` custom resource using the `spec.mode: add-brokers` mode with the new brokers' IDs set on the `spec.brokers` field.
* before scaling the cluster down, in order to rebalance the cluster and move partition replicas off the brokers to be removed, the user has to manually create a `KafkaRebalance` custom resource using the `spec.mode: remove-brokers` mode with the IDs of the brokers to be removed set on the `spec.brokers` field.

For more details, the proposal [035](https://github.com/strimzi/proposals/blob/main/035-rebalance-types-scaling-brokers.md) introduced the rebalancing modes mentioned above, which will be also referenced across this proposal.

## Motivation

The rebalancing operation during scaling up and down is a manual two steps process.

* scaling the cluster up by adding brokers, then running a rebalance.
* running a rebalance, then scaling the cluster down by removing brokers.

In order to simplify the process, it would be great for the user to deal with just scaling the cluster and then having the operator taking care of running a rebalancing operation automatically when needed.

## Proposal

This proposal is about providing the user the possibility to automatically run a rebalancing when they scale the cluster up (by adding brokers) or scale the cluster down (by removing brokers) by changing the `spec.replicas` within a `Kafka` or `KafkaNodePool` custom resource.

### Extending the `Kafka.spec.cruiseControl` field with auto-rebalancing related configuration

The `Kafka` custom resource is extended by adding a new `spec.cruiseControl.autoRebalance` field.
By default, without such field, no auto-rebalancing runs on scaling.

The `spec.cruiseControl.autoRebalance` field allows to specify a list of rebalancing modes with the corresponding configuration to be used.
For each entry in the list, the mode is defined by the `spec.cruiseControl.autoRebalance.mode` field (i.e. `add-brokers` or `remove-brokers`) and the corresponding rebalancing configuration is defined as a reference to a `KafkaRebalanceSettings` custom resource by using the `spec.cruiseControl.autoRebalance.settings` field as a [LocalObjectReference](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/local-object-reference/).
This field is optional and if not specified, the auto-rebalancing runs with the default Cruise Control configuration.

In the following example, a user can decide to use the same rebalancing settings when adding or removing brokers.

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
      # using the same rebalancing settings when adding or removing brokers
      - mode: add-brokers
        settings: my-add-remove-brokers-rebalancing-settings
      - mode: remove-brokers
        settings: my-add-remove-brokers-rebalancing-settings
```

The user can even decide to use different settings when adding or removing brokers.

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
      # using specific settings when adding brokers
      - mode: add-brokers
        settings: my-add-brokers-rebalancing-settings
      # using different settings when removing brokers  
      - mode: remove-brokers
        settings: my-remove-brokers-rebalancing-settings
```

The user could even decide to have auto-rebalancing only when adding or removing brokers but not for both.

This way provides the greatest flexibility to describe in which case to run the auto-rebalancing and with which configuration.

The new `KafkaRebalanceSettings` custom resource is very similar to the `KafkaRebalance` one but it doesn't have information about the rebalancing mode and the brokers which are going to be involved in the rebalancing itself.
This is because it doesn't represent an actual rebalance request, to get an optimization proposal first, but just a place where rebalancing related configuration can be defined.
The user can specify rebalancing goals and options (i.e. the `skipHardGoalCheck` one) within such a resource.
When the `KafkaRebalanceSettings` custom resource is referenced within the `spec.cruiseControl.autoRebalance.settings` field for one or more modes, the settings are used to ask Cruise Control for an optimization proposal with such configuration.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalanceSettings
metadata:
  name: my-rebalance-settings
spec:
  # NOTE: mode and brokers fields are not available as in the KafkaRebalance,
  #       they are automatically set on the corresponding KafkaRebalance by the operator
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

When the `spec.cruiseControl.autoRebalance` is set and a specific rebalancing mode refers to a `KafkaRebalanceSettings` custom resource, the operator automatically creates a `KafkaRebalance` custom resource when the user scales the cluster, by adding and/or removing brokers.
The operator copies over goals and rebalancing options from the referenced settings resource to the actual rebalancing one and also adds the `spec.mode` and `spec.brokers` to it.
The `KafkaRebalance` custom resource could be named as `<my-cluster-name>-auto-rebalancing-<mode>` where the `<my-cluster-name>` part comes from the `metadata.name` in the `Kafka` custom resource, and the `<mode>` is referring to which rebalancing has to run (add or remove brokers).

The user has the flexibility to specify the [goals and the other rebalancing related configurations](https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaRebalance-reference) the same way they already do today on a `KafkaRebalance` custom resource.
This allows the user to not rely on the default Cruise Control goals configuration which would be common for any rebalance without overriding goals.
They could define different goals based on the status of the Kafka cluster or skip hard goals in some scenarios for example.
As already mentioned, specifying goals but more in general customizing the optimization proposal request is already possible in the current usage of a `KafkaRebalance` custom resource so it should be allowed for the auto-rebalancing as well.

The idea of using a `KafkaRebalanceSettings` custom resource could be also extended to the manual rebalancing operations started by the user when they create a new `KafkaRebalance` custom resource.
It would mean a change on the `KafkaRebalance` custom resource by adding a new `spec.settings` field to reference the rebalancing settings and deprecating all the other fields used today (i.e. `spec.goals`, ...).
If this proposal is considered valid, doing such a change on the `KafkaRebalance` custom resource would help the proposal itself because the operator should not copy over the entire configuration from the settings resource to the rebalance resource but just set the settings reference.
Of course, such a change would need a different proposal to be approved before this one.

It is also worth mentioning that the additional `KafkaRebalanceSettings` custom resource doesn't bring any additional complexity to the operator in terms of watching, reacting and processing.
The operator doesn't have any Kubernetes watch on the creation or update of such a resource.
It can be only referenced by a `KafkaRebalance` custom resource so its creation doesn't trigger any action by the operator.
The same happens when the `KafkaRebalanceSettings` custom resource is updated: no action is triggered for the operator, its updated content will be used only when a rebalancing is requested via a `KafkaRebalance` custom resource referencing those settings.

### Extending the `Kafka.status` field with auto-rebalancing related state

The auto-rebalancing, together with the scaling action, can take several reconciliations to finish.
For this reason, taking into account cluster operator crashes as well, there is the need to save some state about it.
The `Kafka` custom resource is extended by adding a new `status.cruiseControl.autoRebalance` field.

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
  cruiseControl:
    autoRebalance:
      state: # if a scaling operation is running or what's the state of the current rebalancing
      add-brokers: # list with brokers' IDs to add (as part of the scaling up)
      remove-brokers: # list with brokers' IDs to remove (as part of the scaling down)
```

The `state` field provides the current state of the overall operation: if scaling is going on or what's the phase of a rebalancing operation.
When the rebalancing is running, its value is got from the status of the corresponding `KafkaRebalance` custom resource that was created for the purpose.
The `add-brokers` and `remove-brokers` track the brokers' IDs that are part of the scaling (up/down) actions to be useful across reconciliations, while scaling or the effective rebalancing is going on.
Their values are got from the new `KafkaCluster.addedNodes` method (needs to be implemented) and the already existing `KafkaCluster.removedNodes` one.

### Auto-rebalancing execution

Before describing the process in more details, it's worth mentioning how scaling up and down happens today within the Cluster Operator reconciliation flow.

The scaling up is triggered asynchronously during a reconciliation which isn't blocked on waiting for the newly added nodes to be ready.
In the `KafkaReconcile.reconcile()` method, it is triggered via the `podSet()` method which updates the involved `StrimziPodSet`(s) so that the corresponding controller will take care of spinning up the newly added nodes while the current reconciliation just ends.
It means that, after the scaling up is triggered, the new pods will take time to be up, running and ready, so it's likely that the auto-rebalancing will happen on a future reconciliation.

The scaling down runs synchronously within a single reconciliation which is blocked on waiting for the nodes to be removed.
In the `KafkaReconcile.reconcile()` method, it is executed via the `scaleDown()` method which updates the involved `StrimziPodSet`(s) so that the corresponding controller will take care of removing the nodes while the current reconciliation waits for the end of the operation.

The idea is about adding a new `autoRebalancing()` method in the `KafkaReconcile.reconcile()` flow, taking care of starting a rebalancing and checking the status of an ongoing one across reconciliations.
The `KafkaReconcile.reconcile()` flow, taking into account methods involved in the scaling and rebalancing, would be something like:

* ... 
* `scaleDown()`
* ...
* `podSet()`
* ...
* `autoRebalancing()`
* ...

> In case of scaling down, the rebalancing has to happen before removing brokers and not after that as it looks like from the above flow. Despite that, some internal state will help to have the right actions order across reconciliations as explained in the next sections.

#### Auto-rebalancing on scaling up

The following flow is triggered when the user increases the `replicas` field within a `Kafka` or `KafkaNodePool` custom resource in order to scale up the cluster by adding more brokers.

1. On the **1st** reconciliation (scaling up is requested): 
     * the `podSet()` method triggers the scaling up, as described in the previous section, but the reconciliation is not blocked on waiting for the newly added pods to be ready.
     * the `autoRebalancing()` method detects that a scaling up was requested because of `KafkaCluster.addedNodes()` returning a non empty list with the brokers' IDs to be added to the cluster. Because the `Kafka.status.cruiseControl.autoRebalance.state` doesn't exist, it's a new operation so the state is updated as `ScalingUp` and the `add-brokers` with the brokers' IDs. Of course, it doesn't trigger any rebalance because the scaling up is going on asynchronously.
2. From the **2nd** to the **(N-1)th** reconciliation (scaling up is going on, checks when it's done):
     * the `podSet()` does nothing because no new brokers were added this time.
     * the `autoRebalancing()` evaluates the `Kafka.status.cruiseControl.autoRebalance.state: ScalingUp`, so a scaling up is running, and it has to check if it's finished or not. This can be achieved looking at the involved `StrimziPodSet`(s) and comparing `readyPods` and `pods` fields in the corresponding status. If they differ, the scaling up is still running. It should not take long time but it could go across multiple reconciliation, so this check repeats.
3. On the **Nth** reconciliation (scaling up is done, rebalancing can start):
     * the `podSet()` does nothing because no new brokers are added this time.
     * the `autoRebalancing()` evaluates the `Kafka.status.cruiseControl.autoRebalance.state: ScalingUp`, so a scaling up is running, but finally looking at the involved `StrimziPodSet`(s), it gets `readyPods` equals to `pods` so the scaling up is done. A new `KafkaRebalance` custom resource is created and the `Kafka.status.cruiseControl.autoRebalance.state` is updated with the corresponding state (should be `ProposalPending`). The `KafkaRebalanceAssemblyOperator` takes care of handling the rebalancing.
4. From the **(N+1)th** to the **(M-1)th** reconciliation (rebalancing is going on, checks when it's done):
     * the `podSet()` does nothing because no new brokers are added this time. 
     * the `autoRebalancing()` has to check the status of the rebalancing by looking at the corresponding `KafkaRebalance` custom resource status which will move through `ProposalPending`, `ProposalReady` to `Rebalancing` (we are assuming auto-approval). The rebalancing status is reflected into the `Kafka.status.cruiseControl.autoRebalance.state` field.
5. On the **Mth** reconciliation (rebalancing id done):
     * the `podSet()` does nothing because no new brokers are added this time.
     * the `autoRebalancing()` detects that the `KafkaRebalance` custom resource status reports the `Ready` state (so the rebalancing is completed).

During the above flow, errors can happen after the scaling up:

* There is an error when getting the proposal.
* After auto-approving the proposal, there is an error when running the rebalance.

In both scenarios, the reconciliation ends with the `KafkaRebalance` custom resource in the `NotReady` state which is reflected into the `Kafka.status.cruiseControl.autoRebalance.state` field.

It's the user who should deal with this scenario:

* Checking the reason why the `KafkaRebalance` is in the `NotReady` state and :
  * deleting the custom resource if they are not interested in the rebalancing anymore. Scaling up is done already.
  * setting the `strimzi.io/rebalance: refresh` annotation on the `KafkaRebalance` custom resource to try the rebalancing again.

### Auto-rebalancing on scaling down

The following flow is triggered when the user decreases the `replicas` field within a `Kafka` or `KafkaNodePool` custom resource in order to scale down the cluster by removing brokers.

1. On the **1st** reconciliation (scaling down is skipped, rebalancing started):
     * the check on scaling down the cluster fails, but it doesn't have to revert the `replicas` back because the `Kafka.spec.cruiseControl.autoRebalance` is provided. So the reconciliation doesn't fail.
     * the `scaleDown()` method skips doing the scale down because a rebalancing is needed first (and the check failed).
     * the `autoreRalancing()` method detects that a scaling down was requested because of `KafkaCluster.removedNodes()` returning a non empty list with the brokers' IDs to be removed from the cluster. Because the `Kafka.status.cruiseControl.autoRebalance.state` doesn't exist, it's a new operation so a new `KafkaRebalance` custom resource is created and the `Kafka.status.cruiseControl.autoRebalance.state` is updated with the corresponding state (should be `ProposalPending`) and the `remove-brokers` with the brokers' IDs.
2. From the **2nd** to the **(N-1)th** reconciliation (rebalancing is going on, checks when it's done):
     * the checks on scaling down the cluster keeps failing but it doesn't have to revert the `replicas` back because the `Kafka.spec.cruiseControl.autoRebalance` is provided. So the reconciliation doesn't fail.
     * the `scaleDown()` method skips doing scale down because a rebalancing is needed first (and the check failed).
     * the `autoRebalancing()` has to check the status of the rebalancing by looking at the corresponding `KafkaRebalance` custom resource status which will move through `ProposalPending`, `ProposalReady` to `Rebalancing` (we are assuming auto-approval). The rebalancing status is reflected into the `Kafka.status.cruiseControl.autoRebalance.state` field.
3. On the **Nth** reconciliation (rebalancing id done, scaling down can start):
     * the checks on scaling down the cluster doesn't fail, because the auto-rebalancing was completed and the nodes to be removed don't host any partition replicas anymore.
     * the `scaleDown()` method can run the scaling down.
     * the `autoRebalancing()` detects that the `KafkaRebalance` custom resource status reports the `Ready` state (so the rebalancing is completed).

During the above flow, errors can happen before the scaling down:

* There is an error when getting the proposal.
* After auto-approving the proposal, there is an error when running the rebalance.

In both scenarios, the reconciliation ends with the `KafkaRebalance` custom resource in the `NotReady` state which is reflected into the `Kafka.status.cruiseControl.autoRebalance.state` field.

The reconcile ends but the scale down cannot proceed because of the brokers to be removed are still hosting some partition replicas.

On the next reconciliation, the cluster operator will try to do a scale down again, so running the flow as before.
The previous `KafkaRebalance` custom resource in the `NotReady` state is overridden to generate a new optimization proposal.

## Affected/not affected projects

This proposal involves the Cluster Operator only.
More specifically, the development is focused on the `KafkaReconciler` class where methods mentioned in the previous sections need to be added and/or updated.
Even the `KafkaClusterCreator` should be changed in order to avoid reverting back a scaling down request when an auto-rebalance is configured in the `Kafka` custom resource.

## Compatibility

There are no backward compatibility issues.
The auto-rebalancing on scaling up and down is optional.
The user has to define the `spec.cruiseControl.autoRebalance` field on the `Kafka` custom resource in order to have the operator running the rebalancing automatically.
Without that additional field, the scaling actions are not influenced by any rebalancing operations to take place.
If the user wants, the rebalancing can be still done with the manual two steps process as today.
The same way, the new `status.cruiseControl.autoRebalance` field in the `Kafka` custom resource doesn't show up at all if the auto-rebalancing feature is not used.

## Rejected alternatives

### Alternative 1

Like the proposed solution, this alternative is still about leveraging the auto-creation of a `KafkaRebalance` custom resource and the `KafkaRebalanceAssemblyOperator` to take care of it without any further changes.
The main difference is related to the rebalancing configuration, because it just uses the Cruise Control defaults.

Pros:

* Simpler compared to the proposed solution because no additional `KafkaRebalanceSettings` custom resource is involved.
  
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

* Simpler compared to the proposed solution because no additional `KafkaRebalanceSettings` custom resource is involved.
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
In the second scenario, the `KafkaAssemblyOperator` should be able to interact directly with the Cruise Control rebalancing FSM by getting any rebalancing parameter from a settings resource or from the Cruise Control defaults, without any need for an auto-created `KafkaRebalance` custom resource which is the main goal of this alternative.

Pros:

* No automatic `KafkaRebalance` custom resource creation.
* No user confusion between auto-created and manually-created `KafkaRebalance` custom resources.

Cons:

* More complex and more work on factoring out the Cruise Control rebalancing FSM from the `KafkaRebalanceAssemblyOperator`.
* Not leveraging the already ready-to-use `KafkaRebalanceAssemblyOperator` through a `KafkaRebalance` custom resource.
* `KafkaAssemblyOperator` strongly tight to the Cruise Control rebalancing FSM with additional complexity for testing.



