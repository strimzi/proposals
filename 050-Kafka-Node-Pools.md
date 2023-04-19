# Kafka Node Pools

This proposal introduces a new custom resource called `KafkaNodePool` which would allow us to configure different _pools_ of Apache Kafka nodes with different configurations and properties.

## Current situation

Strimzi currently allows only one set of ZooKeeper nodes and one set of Kafka nodes.
All nodes within the given set have very similar configuration and differ only in small details:

* Each Pod might have a PVC using different storage class.
* The Kafka pods each have their own Config Map  (when using `StrimziPodSets`) with their Kafka configuration, but the configuration differs between the nodes only in the advertised addresses.

## Problem description

Strimzi does not currently support nodes that have:

* Different resources configuration (such as CPU or memory capacity)
* Different storage configuration (although nodes may have different storage classes, they should have the same storage size and number of disks)
* Different scheduling rules (such as affinity or tolerations)
* Different Kafka configuration (even if the option is configurable per-node or per-cluster in Kafka, Strimzi always manages it per-cluster)
* Different Kubernetes clusters

While from time to time these limitations were raised by the users, they were for a long time acceptable and not in general demand.
However, lately there seems to be increased demand for various features which require different configurations for different nodes.
This proposal does not aim to solve all of these but rather to provide a basis for solving them.

### ZooKeeper-less Kafka

Kafka is slowly finishing the implementation of the KRaft mode.
The KRaft mode removes the ZooKeeper dependency and replaces it with Kafka's own controller nodes based on the Raft protocol (_KRaft = Kafka + Raft_).
In the KRaft mode, the Kafka cluster will consist only from Kafka nodes.
But they have different roles - either controllers, or brokers, or a combination of both.
The controller role means the node is responsible for maintaining the cluster quorum, for the elections, discovery and for handling the cluster metadata instead of ZooKeeper.
The broker role means the node is responsible for handling messages, like a regular broker in the old ZooKeeper-based clusters.
And finally, the combined role means the node is responsible for both at the same time.

The KRaft based Kafka clusters will be typically used in 3 different architectures:

1. One or more nodes combining the controller and broker roles
2. One or more nodes with the controller roles and one or more nodes with the broker roles
3. One or more nodes with the combined role and one or more nodes with the broker roles

In Strimzi, we want to support all of these architectures.
But we also want to make it possible to transition between them.
Since Apache Kafka plans to remove ZooKeeper support in its 4.0 release, supporting KRaft mode in all its different forms is a requirement for us.

### Stretch clusters

There seems to be increased demand for Kafka clusters stretched across multiple Kubernetes clusters.
Apache Kafka itself has latency limitations and might not run well when stretched over distant clusters with big latency.
But there are many use cases where stretched clusters might be useful.
For example:

* When running closely co-located Kubernetes clusters
* When migrating from an old Kubernetes cluster to a new Kubernetes cluster

## Proposal

To resolve the issues inherent with the current implementation of Strimzi, this proposal suggests the introduction of Kafka _node pools_.
_Node pools_ are groups of Kafka nodes which will have the same configuration.
A Kafka cluster can consist of one or more pools.
Each node pool can belong to only one Kafka cluster.

### `KafkaNodePool` custom resource

A new custom resource called `KafkaNodePool` will be added to Strimzi.
One `KafkaNodePool` resource will represent one _node pool_.
It will take over some of the configuration from the `Kafka` CR, reuse other parts from it and add some new configuration options as well.
The `KafkaNodePool` resource will be a _public_ resource.
It will be created and managed directly by users (unlike for example the `StrimziPodSet` resource which is used only internally by Strimzi and should not be touched by the users).

The _cluster-wide_ configuration options will remain part of the `Kafka` CR.
While the _pool-specific_ options will be taken from `KafkaNodePool` resources

For the initial implementation, the following options are proposed for the `KafkaNodePool` resource and its `.spec` section:

* `replicas` indicating the number of nodes in the pool
* `storage` indicating the storage of the nodes in this pool
* `resources` indicating the amount of resources (requests / limits) assigned to the nodes in this pool
* `template` for customizing the pods and other resources which are part of the nodes in this pool
* `jvmOptions` for customizing the JVM configuration

For example:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: my-pool
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  resources:
    requests:
      memory: 8Gi
      cpu: 2
    limits:
      memory: 8Gi
      cpu: 4
  jvmOptions:
    -Xms: 2048m
    -Xmx: 2048m
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        deleteClaim: true
  template:
    podSet:
      metadata:
        labels:
          key1: label1
    pod:
      terminationGracePeriodSeconds: 60
```

The replicas and storage sections will be required in `KafkaNodePool`.
The other fields will be optional.

When the optional field is not present in `KafkaNodePool` and is present in `Kafka.spec.kafka`, it will default to the configured value from the `Kafka` CR.
So when `KafkaNodePool.spec.<field>` exists, `Kafka.spec.kafka.<field>` will be ignored.
This will be applied only on the highest level as demonstrated in the example below:

* If the `KafkaNodePool.spec.jvmOptions` does not exist but `Kafka.spec.kafka.jvmOptions` does exist, the values from `Kafka.spec.kafka.jvmOptions` will be used.
* When `KafkaNodePool.spec.jvmOptions` exists and has `-Xmx: 1024m` and `Kafka.spec.kafka.jvmOptions` exists as well with `-Xms: 512m`, the operator will completely ignore the value from `Kafka.spec.kafka.jvmOptions` and use only the value from `KafkaNodePool.spec.jvmOptions`.
* When `KafkaNodePool.spec.template` contains only the field `podSet.metadata.labels`, the operator will completely ignore the corresponding values from `Kafka.spec.kafka.template` (for example `podSet.metadata.annotations`, but also for example `pod.metadata.labels`).

In the future, additional fields might be moved from the `Kafka` resource to the `KafkaNodePool` resource and its `.spec` section according to the needs.
This proposal tries to minimize the effort and the scope and therefore picks up only some of the fields for the initial implementation.
Since the existing fields in the `Kafka` CR will be used as the _defaults_, new fields can be added to `KafkaNodePool` resource in the future without any backwards compatibility concerns.

### Impact on the `Kafka` custom resource

In the Kafka custom resource, `.spec.kafka.replicas` and `.spec.kafka.storage` values are required today.
In the future, these options will be defined in the `KafkaNodePool` resources and not in the `Kafka` resource.
With the introduction of the `KafkaNodePool` resource, we will make them optional and deprecated in the `Kafka` resource.
This will be done only after the `KafkaNodePool` feature gate moves to beta phase and is enabled by default.

Until then, `.spec.kafka.replicas` and `.spec.kafka.storage` values will be required in the `Kafka` resource so that regular users not using node pools have the proper validation of the custom resources at the Kubernetes level.
Users who want to test the node pools in the alpha phase can set dummy values in these fields.

The `.status` section of the `Kafka` custom resource will also contain a list of node pools used by this cluster:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: my-namespace
  annotations:
    strimzi.io/node-pools: enabled
spec:
  # ...
status:
  # ...
  nodePools:
    - name: <NodePool1Name>
    - name: <NodePool2Name>
```

The `name: <NodePoolName>` format was chosen instead of a simple list of strings to allow for easy extension in the future.
By using this format, additional fields can be added as needed.

### Resource naming

Each node pool will be represented by its own `StrimziPodSet` and pods running the actual nodes.
The `StrimziPodSet` will be created automatically by the operator based on the `KafkaNodePool` definition - it will not be created by the users.
The `StrimziPodSet` will be named `<ClusterName>-<PoolName>` - e.g. `my-cluster-big-nodes`.
The resulting pods will be named `<ClusterName>-<PoolName>-<ID>` - e.g. `my-cluster-big-nodes-5`.
The naming of the related resources such as Services, Config Maps, or PVCs will follow the pod names.
For example the services will be named `<ClusterName>-<PoolName>-<ListenerName>-<ID>` for per-broker and `<ClusterName>-<PoolName>-<ListenerName>-bootstrap` for the bootstrap service.

That means that when a node moves from one pool to another, all the resources will be recreated including the per-node services or storage.
It is not possible to re-use these things not just because of the naming concerns, but also because the different pools might have different configurations and the old resources might not be re-usable.
So a node which moves will start from _zero_ with empty disk and will need to re-sync all the data.

Combining the name of the `Kafka` custom resource and the `KafkaNodePool` resource creates a constraint on how long the names can be since the names of Kubernetes resources are limited to 63 characters and all the resource names derived from the `Kafka` and `KafkaNodePool` names have to fit into this limit.
But following this pattern allows us to provide backwards compatibility (`Kafka` cluster `my-cluster` with `KafkaNodePool` named `kafka` will result in pods named `my-cluster-kafka-<ID>` which is the same name as used today).
It also creates a consistency with the other already existing resources which are part of the Kafka cluster and which cannot be simply renamed (such as names of the existing services etc.).
So we should stick with it despite the limitations.

### Matching the node pools to the Kafka cluster

A particular `KafkaNodePool` resource can belong only to a single Kafka cluster.
The node pools will use the same mechanism to define to which cluster they belong as we use today for `KafkaConnector`, `KafkaTopic` or `KafkaUser` resources.
`strimzi.io/cluster` label will be set on the `KafkaNodePool` and its value will be the name of the `Kafka` custom resource to which it belongs.
For example, the label `strimzi.io/cluster: my-cluster` will mean that this node pool belongs to a Kafka cluster named `my-cluster`.
The `Kafka` and `KafkaNodePool` resources always have to be in the same namespace.

In order to use the `KafkaNodePool` resources users would also need to:
* Make sure the feature gate is enabled
* Add an annotation `strimzi.io/node-pools: enabled` to the `Kafka` custom resource

#### Example

The following example shows how the `Kafka` and `KafkaNodePool` resources might combine to setup a Kafka cluster with 6 different brokers:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: my-namespace
  annotations:
    strimzi.io/node-pools: enabled
spec:
  kafka:
    version: 3.4.0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.3"
  # ...
status:
  # ...
  nodePools:
    - name: big-nodes
    - name: small-nodes
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: big-nodes
  namespace: my-namespace
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  resources:
    requests:
      memory: 64Gi
      cpu: 8
    limits:
      memory: 64Gi
      cpu: 12
  storage:
    type: jbod
    volumes:
    - id: 0
      type: persistent-claim
      size: 1000Gi
      class: fast-storage
    - id: 1
      type: persistent-claim
      size: 1000Gi
      class: fast-storage
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: small-nodes
  namespace: my-namespace
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  resources:
    requests:
      memory: 4Gi
      cpu: 1
    limits:
      memory: 4Gi
      cpu: 2
  storage:
    type: jbod
    volumes:
    - id: 0
      type: persistent-claim
      size: 100Gi
      class: regular-storage
```

#### Protecting `KafkaNodePool` resources from being shared between multiple Kafka clusters

The `strimzi.io/cluster` label should guarantee that the node pool will match only one Kafka cluster.
Additional protection will be made against undesired moves between different Kafka clusters when the `strimzi.io/cluster` label is modified.
The `.status` section of the `KafkaNodePool` resource will contain the Kafka cluster ID of the cluster it belongs to.
For example:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: my-pool
  labels:
    strimzi.io/cluster: my-cluster
spec:
  # ...
status:
  # ...
  clusterId: cABpUKeDTqi7q_XOtBMgUv
```

The cluster ID is already part of the `Kafka` CR `.status` section.
The operator will check the cluster IDs in every reconciliation and if the cluster IDs don't match between the `Kafka` and `KafkaNodePool` CRs, it will expect that there is some kind of an misconfiguration.
It will throw an `InvalidResourceException` and wait for the user to fix the issue.

### Scaling the node pools

The `KafkaNodePool` resource will also support the `scale` sub-resource.
Support for the `scale` sub-resource should allow integration with Kubernetes Horizontal Pod Autoscaler.
That on its own doesn't really provide a fully functional Kafka auto-scaling mechanism.
For example, if the load is coming from a single partition which is handling too many messages, adding more nodes will not help.
So a proper auto-scaling mechanism would need to understand the configuration of topics and partitions and their load to properly scale the broker.
But the `scale` sub-resource would make it easier to develop such solutions in the future.

### KRaft support

To support the Kafka KRaft mode, a new `roles` field will be added to the `KafkaNodePool`.
It will contain a list of roles (`controller` or `broker`) assigned to nodes in a given pool.
This will allow to use different configurations.
For example, for a small cluster with shared broker and controller responsibilities:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: my-namespace
  annotations:
    strimzi.io/node-pools: enabled
spec:
  kafka:
    version: 3.4.0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.3"
  # ...
status:
  # ...
  nodePools:
    - name: mixed
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: mixed
  namespace: my-namespace
  labels:
    strimzi.io/cluster: my-cluster
spec:
  roles:
    - controller
    - broker
  replicas: 3
  resources:
    requests:
      memory: 8Gi
      cpu: 2
    limits:
      memory: 8Gi
      cpu: 4
  storage:
    type: jbod
    volumes:
    - id: 0
      type: persistent-claim
      size: 1000Gi
    - id: 1
      type: persistent-claim
      size: 1000Gi
```

Or for a cluster with separate controller and broker pools:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: my-namespace
  annotations:
    strimzi.io/node-pools: enabled
spec:
  kafka:
    version: 3.4.0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.3"
  # ...
status:
  # ...
  nodePools:
    - name: controllers
    - name: brokers
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controllers
  namespace: my-namespace
  labels:
    strimzi.io/cluster: my-cluster
spec:
  roles:
    - controller
  replicas: 3
  resources:
    requests:
      memory: 8Gi
      cpu: 2
    limits:
      memory: 8Gi
      cpu: 4
  storage:
    type: jbod
    volumes:
    - id: 0
      type: persistent-claim
      size: 100Gi
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: brokers
  namespace: my-namespace
  labels:
    strimzi.io/cluster: my-cluster
spec:
  roles
    - broker
  replicas: 6
  resources:
    requests:
      memory: 64Gi
      cpu: 8
    limits:
      memory: 64Gi
      cpu: 12
  storage:
    type: jbod
    volumes:
    - id: 0
      type: persistent-claim
      size: 1000Gi
    - id: 1
      type: persistent-claim
      size: 1000Gi
```

The KRaft mode will be supported only through the `KafkaNodePool` resources.

### Handling of node IDs

Node IDs (broker IDs) uniquely identify each of the Kafka cluster nodes.
Node IDs will be assigned sequentially:

* They will start with 0.
* By default, when a new node ID is needed, the lowest possible available number will be used regardless the pool where the node will be added:
    * In most cases, the node ID will be assigned from 0 up: 0, 1, 2, 3.
    * If any gap will exist in the existing node IDs, it will be filled in.
      For example when the existing nodes have IDs 0, 1, 4, 5, 6; the next node will get ID 2.
* By default, when scaling down a pool, a highest used node ID will be removed.
    * For example in Kafka cluster with two pools, where the first pool has IDs 0, 1 and 2 and the second pool has IDs 3, 4 and 5, when you scale down the first pool, the node with ID 2 will be removed and when you scale down the second pool node 5 will be removed.
* The pods will reflect the node IDs in their name in the same way as they do today.

The fact that the node IDs will not be assigned to the pools means the Pods will be mixed.
In a cluster with two pools, they might look like this:

* `my-cluster-big-nodes-0`
* `my-cluster-big-nodes-1`
* `my-cluster-small-nodes-2`
* `my-cluster-small-nodes-3`
* `my-cluster-big-nodes-4`
* `my-cluster-small-nodes-5`

This is anyway unavoidable if we want to allow nodes to move between the pools.

To allow the operator to keep track of the assigned node IDs, it will keep track of them in the `.status` section of the `KafkaNodePool` resource.
That way, it will be able to know at any point in time which IDs are used and which are available.
It will be also able to identify the scale-down or scale-up events using this information.

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: big-nodes
  # ...
spec:
  replicas: 3
  # ...
status:
  nodeIds:
    - 0
    - 1
    - 2
  clusterId: cABpUKeDTqi7q_XOtBMgUv
  # replicas and labelSelector fields are needed for the scale subresource
  replicas: 3
  labelSelector: >-
    strimzi.io/cluster=my-cluster,strimzi.io/name=m-cluster-big-nodes,strimzi.io/kind=Kafka
```

Users will be able to use annotations on the `KafkaNodePool` custom resources to indicate to the operator what should be the next ID when scaling up or what should be the node ID which should be removed when scaling down through annotations.
Setting an annotation `strimzi.io/next-node-ids` will tell the operator what should be the next ID used in scale-up.
The value might be a single number, list of numbers or one or more ranges.
E.g. `[3]`, `[3, 4, 5]` or `[1000-1010]`.
Similarly, setting an annotation `strimzi.io/remove-node-ids` will allow the user to configure the IDs nodes which should be removed.
This will support only an ordered list of nodes IDs without ranges.
When the operator is adding or removing a node from the pool, it will look at these annotations and if they are set, it will use them to override the default mechanism for picking up the next node ID or the node to remove.
These annotations should be added before changing the replica count and will be ignored when scaling is not requested by changing `.spec.replicas`.
If these annotations cannot be fulfilled (for example because the ID is already taken and already in use) they will be ignored and the next suitable ID will be picked up.
The operator will raise a warning in its log in such case.

The annotations will be used only when scaling up or down or moving nodes.
The operator will validate them only in such situations and not during every reconciliation.
It is expected that when using auto-scaling, the annotations will not be updated after every scale-up or scale-down event.
So when the user specifies with annotation for example that new nodes should come from a range `[1000-1010]`, such annotation is considered valid even when some of the IDs from the range are already in use.
This way, the node pool can auto-scale up and down within the range without changing the annotation.

#### Moving nodes between node pools

In some situations, it might be good to be able to move a node between different node pools.
The annotations can be used for that as well.
The process to move a node with ID 3 from a pool named `small-nodes` to a pool named `big-nodes` is as follows:
1) Annotate `KafkaNodePool` named `small-nodes` with annotation `strimzi.io/remove-node-ids: [3]`
2) Annotate `KafkaNodePool` named `big-nodes` with annotation `strimzi.io/next-node-ids: [3]`
3) Scale `KafkaNodePool` named `small-nodes` down by changing its `.spec.replicas` field
4) Scale `KafkaNodePool` named `big-nodes` up by changing its `.spec.replicas` field

The operator will proceed and remove the node with ID 3 from the `small-nodes` pool and create a new node with ID 3 in the `big-nodes` pool.
The moved node in its new pod will start from zero and will have to replicate all the data from other nodes in the cluster.

This process has several risks:
* The moved node starts from 0 with empty disk and will need to re-sync all data from other nodes.
  Kafka nodes might store in some environments TBs of data, so re-syncing the data might take a long time.
* While the node is re-syncing, the replicas hosted by it are not in-sync.
  So with typical configurations (such as replication factor set to 3 and minimal in-sync replicas 2) any problem with one of the other nodes would mean loss of availability and producers will not be able to produce messages.
* Since the replica will not be in-sync, it would also mean that it might not be possible to proceed with rolling updates and other operations.

Due to these risks and limitations, users should instead of moving a node consider moving the partition replicas.
They should:
1) Scale up the target node pool to have a new node created
2) Reassign the partition replicas from the old node to the new node
3) Once the reassignment is complete, scale down the node pool with the old node

While this will cause a change of the node ID, it will be a more graceful process.
The reassignment process will first create a new replica and start copying the data.
Only when the new replica on the new node is in-sync, the old replica will be deleted.
Thanks to that, the availability is still guaranteed and other changes such as rolling updates are not affected.
This process should be also recommended to the users in the documentation.

In the future, we should try to improve the Cruise Control support to make this process easier (maybe with a new rebalance mode `move-broker` or something similar).
This is however not part of this proposal.
Until this is implemented, the partition replicas can be moved also using the `kafka-reassign-partitions.sh` tool.

Despite the risks and limitations, having the ability to be able to move the nodes between node pools is still desired and might be useful.
So it should still be supported, even if it might not be part of the initial implementation.

### Deletion of the `KafkaNodePool` resource

When the `KafkaNodePool` resource belonging to an existing Kafka cluster is deleted, all the resources belonging to it and managed by it will be deleted as well.
This will also include the Pods and other resources.
The PVCs will be deleted only if the `deleteClaim` flag was set to `true` in the storage configuration (default value for `deleteClaim` is `false` - so it will not be deleted by default).
While it is possible that the user deleted the `KafkaNodePool` by mistake, once the resource is deleted, there is not much the operator can do about it.
This behavior does not differ from what happens if the `Kafka` custom resource is deleted, so it does not introduce any new behavior.
Users can use other mechanisms such as RBAC or finalizers to prevent deleting the custom resources by mistake.

### Implementation

#### Feature Gate

The node pools will be introduced gradually using a Feature Gate.
The new Feature gate will be named `KafkaNodePools`.
The following table shows the expected graduation of the `KafkaNodePools` feature gate:

| Phase | Strimzi versions       | Default state                                          |
|:------|:-----------------------|:-------------------------------------------------------|
| Alpha | 0.35 - 0.37            | Disabled by default                                    |
| Beta  | 0.38 - 0.39            | Enabled by default                                     |
| GA    | 0.40 and newer         | Enabled by default (without possibility to disable it) |

The main purpose of the feature gate is in this case to protect users from a feature which is not mature enough and might be removed in future versions.
Only small parts of the code will actually depend on the feature gate being enabled or disabled.

The annotation `strimzi.io/node-pools: enabled` on the `Kafka` custom resource will be also required in order to use the `KafkaNodePool` resources on a given Kafka cluster.

Since the KRaft mode will be supported only with `KafkaNodePool` resources, the `UseKRaft` feature gate will be configured as dependent on the `KafkaNodePools` feature gate.
It will not be possible to enable `UseKRaft` feature gate without also having `KafkaNodePools` feature gate enabled.

#### Virtual node pool

To avoid two parallel implementations in the Strimzi source code, the operator will use _virtual node pool_ when the feature gate is disabled.
It will create a node pool internally based on the `Kafka` custom resource and create the resources through this node pool.
The _virtual node pool_ will not exist as a Kubernetes resource.
The _virtual node pool_ will be named `kafka` which will result in the same names of the Kubernetes resources as we use today.
And since it will copy the configuration from the `Kafka` custom resource, the configuration (e.g. resources, storage etc.) will be the same as before as well.
This significantly simplifies the implementation and testing since the same code will be used all the time.

### Backwards compatibility and migration

When the `KafkaNodePool` feature gate is enabled, users do not need to change anything right away.
They can keep using their existing `Kafka` resources without any changes.
The operator will continue to use the _virtual node pool_ as described in the previous section.

When they decide to start using the actual `KafkaNodePool` resources, they can either:
* Deploy a brand new Kafka cluster with one or more `KafkaNodePool` resources
* Convert their existing Kafka cluster to use node pools

#### Converting existing Apache Kafka clusters

To convert an existing Kafka cluster to use node pools, the user has to do the following:

1. Create a new `KafkaNodePool` named `kafka` with the `.spec.replicas` and `.spec.storage` matching the configuration from their `Kafka` custom resource.
2. Add an annotation `strimzi.io/node-pools: enabled` to the `Kafka` custom resource

The following examples shows the important parts of the conversion.
An initial `Kafka` custom resource like this:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 3.3.1
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.3"
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 1000Gi
          class: fast-storage
  # ...
```

Is equivalent to the following combination of `Kafka` and `KafkaNodePool` resources:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  annotations:
    strimzi.io/node-pools: enabled
spec:
  kafka:
    version: 3.3.1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.3"
  # ...
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  storage:
    type: jbod
    volumes:
    - id: 0
      type: persistent-claim
      size: 1000Gi
      class: fast-storage
  # ...
```

Because the `KafkaNodePool` is named `kafka`, all the resources created by this configuration will be exactly the same and have the same names as today.
Using a name different from `kafka` would cause a new Kafka nodes to be created instead of migrating the existing nodes.
Once the initial migration is complete, users can continue to use the node pools without any limitation.
For example add additional node pools or further modify the `kafka` node pool.

#### Migration to KRaft

As mentioned earlier, KRaft will be supported only with the use of the `KafkaNodePool` resources. 
_(The detailed description of the migration to KRaft is not part of this proposal.)_
Thanks to that, latest when ZooKeeper support is removed from Strimzi and Apache Kafka, all users will have to migrate from using the _virtual node pool_ to the `KafkaNodePools`.
So once the ZooKeeper support is removed from Strimzi, we will also remove the support for the _virtual node pool_ and the `strimzi.io/node-pools` annotation.
And all Strimzi based Apache Kafka clusters will have to use the `KafkaNodePool` resources

## Risks

### Eventual consistency

Using multiple resources in Kubernetes is tricky as we have to rely on eventual consistency.
Users can store multiple YAML documents in a single file and apply them with a single command.
But their creation is not a single operation.
It can therefore happen that the operator first sees only the `KafkaNodePool` resources but not the `Kafka` resource or the other way around.
The basic situations can be handled by the operator code.
But it is not completely clear how many problems and various race conditions this might be causing when updating the node pools for existing clusters etc.

The eventual consistency also shows up in situations when you for example decide to scale two of your node pools.
The scale events will be independent and will almost never be caught by the same reconciliation.
So you will have one reconciliation to scale the first pool followed by the second to scale the second one.
The users would be responsible for taking this into account.
This will be also covered by the documentation.

## Not impacted

This proposal does not impact in any way how the ZooKeeper nodes, Kafka Exporter, Cruise Control or Entity Operator are configured.
It also does not impact configuration of the `KafkaConnect`, `KafkaBridge` or other custom resources used by Strimzi.
The _node pool_ concept is plan to be introduced only for Kafka broker nodes.
There are currently no plans to extend it further to Kafka Connect, Mirror Maker 2, or Bridge.

## Rejected alternatives

### KafkaNodePool selector in `.spec.kafka` section of the `Kafka` custom resource

One of the options considered was to have a label selector in the `Kafka` custom resource in `.spec.kafka`.
Using the `strimzi.io/cluster` label was preferred in the end.
It makes it easier to match the `KafkaNodePool` resources to the `Kafka` cluster.
This significantly simplifies the code - especially the watch which is triggered when changes are made.
It also helps to ensure that one node pool will always belong only to a single Kafka cluster since one node pool resource can have only one `strimzi.io/cluster` label.

### Configuring pools inside the `Kafka` custom resource

One of the considered options was configuring the node pools inside the `Kafka` custom resource.
For example:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    nodePoolSelector:
      matchLabels:
        cluster: my-cluster
    version: 3.3.1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.3"
  # ...
  nodePools: 
    - name: pool1
      replicas: 3
      resources:
        requests:
          memory: 64Gi
          cpu: 8
        limits:
          memory: 64Gi
          cpu: 12
      storage:
        type: jbod
        volumes:
          - id: 0
            type: persistent-claim
            size: 1000Gi
    - name: pool2
      replicas: 3
      resources:
        requests:
          memory: 64Gi
          cpu: 8
        limits:
          memory: 64Gi
          cpu: 12
      storage:
        type: jbod
        volumes:
          - id: 0
            type: persistent-claim
            size: 1000Gi
```

This would have some advantages:

* Having a single resource is easier to work with as it has a clear consistency.
  Using multiple resources means you have to deal with Kubernetes's eventual consistency when updating multiple resources at the same time

And of course also some disadvantages:
* It would not be possible to support the `scale` sub-resource and support autoscaling in the future.
* In the past, we had issues with the size of our CRDs.
  Bundling the node pools into the `Kafka` CRD would make it bigger.

## Not addressed

### Migration to KRaft / ZooKeeper-less Kafka

While this proposal defines how the node pools will provide support for running Apache Kafka in the KRaft mode, it does not focus on the migration of ZooKeeper-based Kafka clusters to KRaft-based Kafka clusters.
This should be part of a separate proposal.

### Future possibilities

As mentioned in the introduction, in the future, it should be possible to build additional features on top of the node pools.
One of the examples might be stretch clusters, where each pool will run on a different Kubernetes cluster.
These are not part of this proposal.

### `v1` version of the `Kafka` CRD API

While this proposal mentions some options for how the future `v1` API version of the `Kafka` CRD might look like, the final design should be discussed in a separate proposal.
