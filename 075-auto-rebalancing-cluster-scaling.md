# Auto-rebalancing on cluster scaling

This proposal is about enabling the Cluster Operator to rebalance the Apache Kafka cluster automatically when the user scales the cluster up/down by adding/removing brokers to/from the cluster.
Such a rebalancing can happen right after the scale up to move some partition replicas to the newly added brokers or right before the scale down to move partition replicas off the brokers to be removed.

## Current situation

When a user scales a cluster up by adding new brokers, they would like to use the newly added brokers to host already existing partition replicas and not only for the newly created ones.
At the same time, in order to scale the cluster down, the brokers to be removed must not host partition replicas otherwise the operation is not allowed (more details on proposal [049](https://github.com/strimzi/proposals/blob/main/049-prevent-broker-scale-down-if-it-contains-partition-replicas.md)).
The way the user has to move partition replicas in the above scenarios is via the Cruise Control integration by using the `KafkaRebalance` custom resource.

Currently, there is no automatic rebalancing when the user scales the cluster up or down.
It means that:

* after scaling the cluster up, in order to rebalance the cluster and move some partition replicas to the newly added brokers, the user has to manually create a `KafkaRebalance` custom resource using the `spec.mode: add-brokers` mode with the new brokers' IDs set on the `spec.brokers` field.
* before scaling the cluster down, in order to rebalance the cluster and move partition replicas off the brokers to be removed, the user has to manually create a `KafkaRebalance` custom resource using the `spec.mode: remove-brokers` mode with the IDs of the brokers to be removed set on the `spec.brokers` field.

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
For each entry in the list, the mode is defined by the `spec.cruiseControl.autoRebalance.mode` field (i.e. `add-brokers` or `remove-brokers`) and the corresponding rebalancing configuration is defined as a reference to a "template" `KafkaRebalance` custom resource (read later for more details), by using the `spec.cruiseControl.autoRebalance.config` field as a [LocalObjectReference](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/local-object-reference/).
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
        config: my-add-remove-brokers-rebalancing-config
      - mode: remove-brokers
        config: my-add-remove-brokers-rebalancing-config
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
        config: my-add-brokers-rebalancing-config
      # using different configuration when removing brokers  
      - mode: remove-brokers
        config: my-remove-brokers-rebalancing-config
```

The user could even decide to have auto-rebalancing only when adding or removing brokers but not for both.

This way provides the greatest flexibility to describe in which case to run the auto-rebalancing and with which configuration.

The auto-rebalance configuration for the `spec.cruiseControl.autoRebalance.config` property in the `Kafka` resource is provided through a `KafkaRebalance` custom resource defined as a "template".
That is a `KafkaRebalance` custom resource with the `strimzi.io/rebalance: template` annotation set.
When it is created, the `KafkaRebalanceAssemblyOperator` doesn't run any rebalancing.
This is because it doesn't represent an "actual" rebalance request to get an optimization proposal, but it's just the place where configuration related to auto-rebalancing is defined.
The user can specify rebalancing goals and options, such as `skipHardGoalCheck`, within the resource.
When such a "template" `KafkaRebalance` custom resource is referenced within the `spec.cruiseControl.autoRebalance.config` field for one or more modes, the operator asks to Cruise Control for an optimization proposal with such configuration.
If the `mode` and `brokers` fields are set, they will be just ignored because the resource doesn't represent an "actual" rebalancing.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance-config
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

When the `spec.cruiseControl.autoRebalance` is set and a specific rebalancing mode refers to a "template" `KafkaRebalance` custom resource, the operator automatically creates a corresponding `KafkaRebalance` custom resource when the user scales the cluster, by adding and/or removing brokers.
The operator copies over goals and rebalancing options from the referenced "template" resource to the "actual" rebalancing one and also adds the `spec.mode` and `spec.brokers` to it.
The "actual" `KafkaRebalance` custom resource could be named as `<my-cluster-name>-auto-rebalancing-<mode>` where the `<my-cluster-name>` part comes from the `metadata.name` in the `Kafka` custom resource, and the `<mode>` is referring to which rebalancing has to run (add or remove brokers).

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-cluster-auto-rebalancing-add-brokers
  finalizers:
    - strimzi.io/rebalance
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

The operator also sets a finalizer, named `strimzi.io/rebalance`, on the "actual" `KafkaRebalance` custom resource.
This is needed to avoid the user, or any other tooling, to delete the resource while the auto-rebalancing is still running.
The finalizer is removed at the end of the auto-rebalancing process, allowing the "actual" `KafkaRebalance` custom resource deletion by the operator itself.

The user has the flexibility to specify the [goals and the other rebalancing related configurations](https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaRebalance-reference) the same way they already do today on a `KafkaRebalance` custom resource.
This allows the user to not rely on the default Cruise Control goals configuration which would be common for any rebalance without overriding goals.
They could define different goals based on the status of the Kafka cluster or skip hard goals in some scenarios for example.
As already mentioned, specifying goals but more in general customizing the optimization proposal request is already possible in the current usage of a `KafkaRebalance` custom resource so it should be allowed for the auto-rebalancing as well.

The idea of using a "template" `KafkaRebalance` custom resource could be also extended to the manual rebalancing operations started by the user when they create a new `KafkaRebalance` custom resource.
It would mean a change on the `KafkaRebalance` custom resource by adding a new `spec.config` field to reference the rebalancing "template" and deprecating all the other fields used today (i.e. `spec.goals`, ...).
Another option could be leaving both the ways living together.
If this proposal is considered valid, doing such a change on the `KafkaRebalance` custom resource would help the proposal itself because the operator should not copy over the entire configuration from the "template" resource to the actual rebalance resource but just set the configuration reference.
Of course, such a change would need a different proposal to be approved before this one.

It is also worth mentioning that a "template" `KafkaRebalance` custom resource doesn't add much complexity to the operator in terms of watching, reacting and processing.
The `KafkaRebalanceAssemblyOperator` is just going to ignore such a resource when it's created, updated or deleted.
It can be only referenced by an "actual" `KafkaRebalance` custom resource so its creation doesn't trigger any action by the operator.
The operator only reads the "template" `KafkaRebalance` custom resource and generates a new corresponding "actual" `KafkaRebalance` custom resource when the cluster is being scaled.

### Extending the `Kafka.status` field with auto-rebalancing related state

The auto-rebalancing, together with the scaling action, can take several reconciliations to finish.
For this reason, taking into account cluster operator might crash as well, we need to save some information about the state of the auto-rebalancing operation.
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

The `state` field provides the current state of the overall operation: if scaling is going on or what's the phase of a rebalancing operation.
When the rebalancing is running, its value is got from the status of the corresponding "actual" `KafkaRebalance` custom resource that was created for the purpose.
The `lastTransitionTime` field provides the timestamp of the latest auto-rebalancing state update.
The `modes` field lists the enabled rebalancing modes, as defined through the `spec.cruiseControl.autoRebalance.mode` field, with the corresponding `brokers` list containing the brokers' IDs that are part of the scaling (up/down) actions to be useful across reconciliations, while scaling or the effective rebalancing is going on.
Their values are coming from the corresponding `spec.brokers` field in the "actual" `KafkaRebalance` custom resource created by the operator.

### Auto-rebalancing execution

Before describing the process in more details, it's worth mentioning how scaling up and down happens today within the Cluster Operator reconciliation flow.

Both the operations run asynchronously and they involve the `StrimziPodSet` controller which takes care of adding (scale up) or removing (scale down) pods by communicating with the Kubernetes API.

In the `KafkaReconciler.reconcile()` method, the scaling up is triggered by the `podSet()` method which updates the involved `StrimziPodSet`(s) so that the corresponding controller will take care of spinning up the newly added nodes while the current reconciliation just ends.
It means that, after the scaling up is triggered, the new pods will take time to be up, running and ready, so it's likely that the auto-rebalancing will happen on a future reconciliation.

In the `KafkaReconciler.reconcile()` method, the scaling down is triggered by the `scaleDown()` method which updates the involved `StrimziPodSet`(s) so that the corresponding controller will take care of removing the nodes.
Even in this case, the triggered deletion takes some time while the reconciliation ends.

The idea is about adding a new `autoRebalancing()` method in the `KafkaReconciler.reconcile()` flow, taking care of starting a rebalancing and checking the status of an ongoing one across reconciliations.
The `KafkaReconciler.reconcile()` flow, taking into account methods involved in the scaling and rebalancing, would be something like:

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
     * the `autoRebalancing()` method detects that a scaling up was requested because of `KafkaCluster.addedNodes()` returning a non empty list with the brokers' IDs to be added to the cluster. Because the `Kafka.status.autoRebalance.state` doesn't exist or it's `Ready` (from a previous auto-rebalancing), it's a new operation so the state is updated as `ScalingUp` and the `brokers` with the brokers' IDs. Of course, it doesn't trigger any rebalance because the scaling up is going on asynchronously.
2. From the **2nd** to the **(N-1)th** reconciliation (scaling up is going on, checks when it's done):
     * the `podSet()` does nothing because no new brokers were added this time.
     * the `autoRebalancing()` evaluates the `Kafka.status.autoRebalance.state: ScalingUp`, so a scaling up is running, and it has to check if it's finished or not. This can be achieved looking at the involved `StrimziPodSet`(s) and comparing `readyPods` and `pods` fields in the corresponding status. If they differ, the scaling up is still running. It should not take long time but it could go across multiple reconciliations, so this check repeats.
3. On the **Nth** reconciliation (scaling up is done, rebalancing can start):
     * the `podSet()` does nothing because no new brokers are added this time.
     * the `autoRebalancing()` evaluates the `Kafka.status.autoRebalance.state: ScalingUp`, so a scaling up is running, but finally looking at the involved `StrimziPodSet`(s), it gets `readyPods` equals to `pods` so the scaling up is done. A new `KafkaRebalance` custom resource is created, with the `strimzi.io/rebalance-auto-approval: true` annotation, and the `Kafka.status.autoRebalance.state` is updated with the corresponding state (should be `ProposalPending`). The `KafkaRebalanceAssemblyOperator` takes care of handling the rebalancing.
4. From the **(N+1)th** to the **(M-1)th** reconciliation (rebalancing is going on, checks when it's done):
     * the `podSet()` does nothing because no new brokers are added this time. 
     * the `autoRebalancing()` has to check the status of the rebalancing by looking at the corresponding `KafkaRebalance` custom resource status which will move through `ProposalPending`, `ProposalReady` to `Rebalancing` (we are assuming auto-approval). The rebalancing status is reflected into the `Kafka.status.autoRebalance.state` field.
5. On the **Mth** reconciliation (rebalancing is done):
     * the `podSet()` does nothing because no new brokers are added this time.
     * the `autoRebalancing()` detects that the `KafkaRebalance` custom resource status reports the `Ready` state (so the rebalancing is completed). The end of rebalancing is reflected into the `Kafka.status.autoRebalance.state` field and the `KafkaRebalance` is deleted.

During the above flow, errors can happen after the scaling up:

* There is an error when getting the proposal.
* After auto-approving the proposal, there is an error when running the rebalance.

In both scenarios, the reconciliation ends with the `KafkaRebalance` custom resource in the `NotReady` state which is reflected into the `Kafka.status.autoRebalance.state` field.

On the next reconciliation, the scaling up is done already, but the rebalancing cannot run because of the `KafkaRebalance` custom resource in the `NotReady` state.
The user can fix it by deleting the `KafkaRebalance` custom resource so that on the next reconciliation, the operator will be able to retry an auto-rebalancing, by recreating a new `KafkaRebalance` custom resource getting the added brokers from the `status.autoRebalancing.brokers` for the `add-brokers` mode.
Another approach the user can take is removing the `spec.cruiseControl.autoRebalance` (other than deleting the `KafkaRebalance` custom resource) in order to run a manual rebalancing later on.

### Auto-rebalancing on scaling down

The following flow is triggered when the user decreases the `replicas` field within a `Kafka` or `KafkaNodePool` custom resource in order to scale down the cluster by removing brokers.

1. On the **1st** reconciliation (scaling down is skipped, rebalancing started):
     * the check on scaling down the cluster fails, and it reverts the `replicas` back only in memory but not on the `Kafka` custom resource. The removed nodes are collected (for the auto-rebalancing mechanism) in a dedicate `removedNodes` list, which would be a copy of the one returned by `KafkaCluster.removedNodes()`.
     * the `scaleDown()` method skips doing the scale down because a rebalancing is needed first (and the check failed).
     * the `autoRebalancing()` method detects that a scaling down was requested because of `removedNodes` not being empty but containing the brokers' IDs to be removed from the cluster. Because the `Kafka.status.autoRebalance.state` doesn't exist or it's `Ready` (from a previous auto-rebalancing), it's a new operation so a new `KafkaRebalance` custom resource is created, with the `strimzi.io/rebalance-auto-approval: true` annotation, and the `Kafka.status.autoRebalance.state` is updated with the corresponding state (should be `ProposalPending`) and the `brokers` with the brokers' IDs.
2. From the **2nd** to the **(N-1)th** reconciliation (rebalancing is going on, checks when it's done):
     * the checks on scaling down the cluster keeps failing but it keeps reverting the scaling operation.
     * the `scaleDown()` method skips doing scale down because a rebalancing is needed first (and the check failed).
     * the `autoRebalancing()` has to check the status of the rebalancing by looking at the corresponding `KafkaRebalance` custom resource status which will move through `ProposalPending`, `ProposalReady` to `Rebalancing` (we are assuming auto-approval). The rebalancing status is reflected into the `Kafka.status.autoRebalance.state` field.
3. On the **Nth** reconciliation (rebalancing id done, scaling down can start):
     * the checks on scaling down the cluster don't fail, because the auto-rebalancing was completed and the nodes to be removed don't host any partition replicas anymore.
     * the `scaleDown()` method can run the scaling down.
     * the `autoRebalancing()` detects that the `KafkaRebalance` custom resource status reports the `Ready` state (so the rebalancing is completed). The end of rebalancing is reflected into the `Kafka.status.autoRebalance.state` field and the `KafkaRebalance` is deleted.

During the above flow, errors can happen before the scaling down:

* There is an error when getting the proposal.
* After auto-approving the proposal, there is an error when running the rebalance.

In both scenarios, the reconciliation ends with the `KafkaRebalance` custom resource in the `NotReady` state which is reflected into the `Kafka.status.autoRebalance.state` field.

The reconcile ends but the scale down cannot proceed because the brokers to be removed are still hosting some partition replicas.
On the next reconciliation, the cluster operator will try to do a scale down again, so going through the same flow.
The scale down check fails but the rebalancing cannot run because of the `KafkaRebalance` custom resource in the `NotReady` state.
The user can fix it by deleting the `KafkaRebalance` custom resource so that on the next reconciliation, the operator will be able to retry an auto-rebalancing, by recreating a new `KafkaRebalance` custom resource.
Another approach the user can take is removing the `spec.cruiseControl.autoRebalance` (other than deleting the `KafkaRebalance` custom resource) in order to run a manual rebalancing later on.

## Affected/not affected projects

This proposal involves the Cluster Operator only.
More specifically, the development is focused on the `KafkaReconciler` class where methods mentioned in the previous sections need to be added and/or updated.
Even the `KafkaClusterCreator` should be changed in order to collect the removed nodes, in case of scale down, in a dedicated list for the auto-rebalancing mechanism to proceed.

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
