# Node Pool Rack IDs

Rack awareness in Strimzi Kafka can be achieved without broker access to cluster-level Kubernetes APIs
by configuring a rack ID in each node pool for use by all brokers within that pool.

## Current situation

In Kubernetes clusters spanning multiple availability zones, Kafka can tolerate the loss of an
entire zone by ensuring partition replicas are spread across brokers in multiple zones.

This is achieved by configuring the [`broker.rack`](https://kafka.apache.org/41/configuration/broker-configs/#brokerconfigs_broker.rack)
property in Kafka to enable rack-aware replication assignment, where a rack represents an availability zone.

Strimzi currently provides rack awareness through the usage of an init container which queries the
Kubernetes API for the value of a specified topology label on the Kubernetes node where that
broker is running.
This requires broker pods to access the Kubernetes API and use cluster-scoped RBAC.

## Motivation

Today, Strimzi requires access to cluster-scoped Kubernetes resources for rack awareness, NodePort
listener configuration, and reading StorageClasses for volume resizing.
Some Kubernetes cluster administrators, however, may restrict access to cluster-scoped Kubernetes
resources to ensure an application and the user managing it are contained within a limited set of namespaces.
In these situations, Strimzi users are currently unable to use rack awareness.

With this proposal, users will be able to configure and use rack awareness when Strimzi does not
have access to create `ClusterRolesBinding` resources.

This feature might also be useful in other situations such as:

* In testing, where it allows to configuration of rack awareness independently of the underlying infrastructure
  * For example, it allows testing of rack awareness related features on a single-node Kubernetes cluster by manually defining different racks for different node pools
* In future stretch-cluster environments, where automatically configured rack awareness might not be desired
  * For example, when moving Kafka nodes between two Kubernetes clusters, rack IDs may need to be customized

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
      envvar: STRIMZI_RACK_ID
```

The `type` field will be an optional enumeration type with the following values:

* `node-label`
  * This rack type maintains the existing behavior where rack IDs are configured using a node label
  * The node label is determined using the existing `topologyKey` field
* `envvar`
  * This rack type uses an environment variable in the broker container to populate the rack ID

A new `envvar` optional field will also be added to configure the environment variable name used for the
`envvar` rack type. The default value of the `envvar` field will be `STRIMZI_RACK_ID`.

As a part of this change, the `topologyKey` sub-field under `rack` will become optional.

The new `type` field must also be optional to support existing deployments.

We want to be careful to avoid configuration such as the following:

```yaml
spec:
  kafka:
    rack: {}
```

To accomplish this, the following [CEL validation rules](https://kubernetes.io/docs/reference/using-api/cel/)
will be applied to the `rack` object:

* `self.type != "" || (self.type == "" && self.topologyKey != "")`
  * The `type` field can only be empty if the `topologyKey` is defined
* `self.type == "" || (self.type == "envvar" && self.topologyKey == "") || (self.type == "node-label" && self.topologyKey != "")`
  * The `type` field must be correctly defined for the `topologyKey`
* `self.type == "envvar" || (self.type == "" && self.envvar == "") || (self.type == "node-label" && self.envvar == "")`
  * The `envvar` field must only be defined for when `type` is `envvar`

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
        - name: STRIMZI_RACK_ID
          value: zone0
```

When using rack awareness in general, users must configure affinity or topology spread constraints
to ensure the proper distribution of pods.
This is the case for rack awareness configured with either the existing node label method or the
proposed environment variable option.

Although configuring affinity or topology spread constraints is required for proper availability-driven
data distribution, one benefit of this proposal is that users do not necessarily need to define these
rules when controlling the rack ID for testing or other non-production scenarios.

## Affected/not affected projects

Only the [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator/) would be affected:

* Changes to the `Kafka` API type
  * New optional `type` sub-field under `rack`
  * New optional `envvar` sub-field under `rack`
  * Change `topologyKey` sub-field under `rack` from required to optional
* Changes to the Cluster Operator to:
  * Modify the `KafkaBrokerConfigurationBuilder` to use an environment variable for the rack ID if the rack type is `envvar`
  * Only create the broker `ClusterRoleBinding` that grants access to cluster-scoped Kubernetes resources when using the `node-label` rack ID type or when using a NodePort listener
  * Only create the rack configuration init container when using the `node-label` rack ID type

These changes do not affect the `ClusterRoleBinding` resources required by the Cluster Operator itself.

## Compatibility

This proposal maintains CRD compatibility by introducing new, optional fields.
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

* Continue with existing rack awareness support using Kubernetes API access to cluster-scoped resources in a broker init container
  * This alternative was rejected due to the permissions limitations of some users, as outlined in the [Motivation section](#motivation).
