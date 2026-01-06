# Node Pool Rack IDs

Rack awareness in Strimzi Kafka can be achieved without broker access to cluster-level Kubernetes APIs
by configuring a rack ID in each node pool for use by all brokers within that pool.

## Current situation

In Kubernetes clusters spanning multiple availability zones, Kafka can tolerate the loss of an
entire zone by ensuring partition replicas are spread across brokers in multiple zones.

This is achieved by configuring the [`broker.rack`](https://kafka.apache.org/documentation/#brokerconfigs_broker.rack)
property in Kafka to enable rack-aware replication assignment, where a rack represents an availability zone.

Strimzi currently provides rack awareness through the usage of an init container which queries the
Kubernetes API for the value of a specified topology label on the Kubernetes node where that
broker is running.
This requires broker pods to access the Kubernetes API and to use cluster-scoped RBAC.

## Motivation

Today, Strimzi requires access to cluster-scoped Kubernetes resources for rack-awareness, NodePort
listener configuration, and reading StorageClasses for volume resizing.
But some Kubernetes cluster administrators may restrict access to cluster-scoped Kubernetes
resources to ensure an application and the user managing it are contained within a limited set of namespaces.
In such situations, Strimzi users were not able to use the rack-awareness feature.
With this proposal, they will be able to configure and use rack-awareness even when Strimzi does not have the access rights to create `ClusterRolesBinding` resources.

This feature might also be useful in other situations such as:
* In testing, where it allows to configure rack-awareness independently of the underlying infrastructure (it, for example, allows to test rack-awareness related features on a single-node Kubernetes cluster by manually defining different racks for different node pools)
* In future stretch-cluster environments, where automatically configured rack-awareness might not be desired (for example, when moving Kafka nodes between two Kubernetes clusters)

Implementing the proposed pool-based approach makes cluster-scoped RBAC optional for rack awareness.

## Proposal

The `Kafka` CR will be updated to include a `type` sub-field under `rack` which configures the source
for broker rack IDs.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  annotations:
    strimzi.io/node-pools: enabled
  name: my-cluster
spec:
  kafka:
    rack:
      type: envvar
```

The field will be an optional enumeration type with the following values:

* `node-label`
  * This rack type maintains the existing behavior where rack IDs are configured using a node label
  * The node label is determined using the existing `topologyKey` field
* `envvar`
  * This rack type configures the `STRIMZI_RACK` environment variable in the broker container to populate the rack ID

When a `topologyKey` is defined, the default rack type will be `node-label` to maintain existing behavior.

The `KafkaNodePool` CR already provides the option to define environment variables which can be used
for the `envvar` rack type.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: my-zone0-pool
  labels:
    strimzi.io/cluster: my-cluster
spec:
  template:
    kafkaContainer:
      env:
        - name: STRIMZI_RACK
          value: zone0
```

When using rack awareness in general, users should configure affinity or topology spread constraints
to ensure the proper distribution of pods.
This proposal relies on distribution of node pools across zones, so users should configure affinity
or topology spread constraints in the Kafka pod template to ensure:

* Brokers within pools with the same rack ID are scheduled in the same availability zone
* Brokers in pools with different rack IDs are scheduled in different availability zones

This affinity configuration must be specified in the `KafkaNodePool` resource:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: my-zone0-pool
  labels:
    strimzi.io/cluster: my-cluster
spec:
  rackId: zone0
  template:
    pod:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    strimzi.io/cluster: my-cluster
                    strimzi.io/pool-name: my-zone0-pool
                topologyKey: topology.kubernetes.io/zone
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 90
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: strimzi.io/cluster
                      operator: In
                      values:
                        - my-cluster
                    - key: strimzi.io/pool-name
                      operator: NotIn
                      values:
                        - my-zone0-pool
                topologyKey: topology.kubernetes.io/zone
```

Although configuring affinity or topology spread constraints is required for proper availability-driven
data distribution, one benefit of this proposal is that users do not necessarily need to define these
rules when controlling the rack ID for testing or other non-production scenarios.

## Affected/not affected projects

Only the [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator/) would be affected:

* Changes to the `Kafka` API type
  * New optional `type` sub-field under `rack`
  * Change `topologyKey` sub-field under `rack` from required to optional
* Changes to the Cluster Operator to:
  * Modify the `KafkaBrokerConfigurationBuilder` to use the `STRIMZI_RACK` environment variable for the rack ID if the rack type is `envvar`
  * Only create the ClusterRoleBinding when using the `node-label` rack ID type or when using a NodePort listener
  * Only create the rack configuration init container when using the `node-label` rack ID type

## Compatibility

This proposal maintains CRD compatibility by introducing a new, optional field.
All existing configurations would continue to be valid and maintain their existing behavior.

## Available alternatives

As outlined in [strimzi-kafka-operator#11504](https://github.com/strimzi/strimzi-kafka-operator/issues/11504),
Kubernetes 1.33 provides a new alpha feature with which topology node labels are available via the downwardAPI.
This new feature could be used as another mechanism for configuring rack awareness without cluster-scoped
RBAC. The proposal here provides a few benefits over this potential future solution:

1. Supports versions of Kubernetes available today
2. Provides additional flexibility to configure rack IDs that do not align to topology labels
3. Protects brokers from changes to node labels

Benefit 3 can be achieved using a similar configuration to the example described [in the proposal section](#proposal).
The rack ID configured for each broker is a zone identifier, e.g. `zone0`, and not the name of the
actual topology node label. 
When rack IDs in brokers are configured to a specific topology node label, e.g. `us-east-2`, any changes to this label would require restarting the brokers.
For example, changing a set of nodes to use label `us-east-2a` may not warrant an immediate restart of broker pods.

## Rejected alternatives

* Continue with existing rack awareness support using Kubernetes API access in a broker init container
  * This alternative was rejected due to the heightened security requirements of some users, as outlined in the [Motivation section](#motivation).
