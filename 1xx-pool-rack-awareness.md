# Pool-based Rack Awareness

Rack-awareness in Strimzi Kafka can be achieved without broker access to cluster-level Kubernetes APIs
by assigning one node pool per rack/availability zone.

## Current situation

In Kubernetes clusters spanning multiple availability zones, Kafka can tolerate the loss of an
entire zone by ensuring partition replicas are spread across brokers in multiple zones.

This is achieved by configuring the [`broker.rack`](https://kafka.apache.org/documentation/#brokerconfigs_broker.rack)
property in Kafka to enable rack-aware replication assignment, where a rack is analogous to an availability zone.

Strimzi currently provides rack-awareness through the usage of an init container which queries the
Kubernetes API for the value of a specified topology label on the Kubernetes node where that
broker is running.
This requires access to the Kubernetes API by the broker pods and requires cluster-scoped RBAC.

## Motivation

For users interested in a heightened security posture, the requirements of the current rack-awareness
implementation are prohibitive.
Many users may require adherence to the [separation of duty security principle](https://csrc.nist.gov/glossary/term/separation_of_duty)
under which application pods processing user data should not have access to the Kubernetes API.
All usage of the Kubernetes API must then be delegated to the operator.

Furthermore, many Kubernetes cluster administrators may restrict access to cluster-scoped Kubernetes
resources to ensure an application and the user managing it are contained within a limited set of namespaces.
Today, Strimzi only requires access to cluster-scoped Kubernetes resources for rack-awareness and NodePort
listener configuration.

Implementing the proposed method for pool-based rack awareness removes these potentially prohibitive
requirements while maintaining a simple deployment model.

## Proposal

The `Kafka` CR will be updated to include an `idType` field which signifies the type of rack ID
which will be configured.

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
      idType: pool-name
```

The field will be an optional enumeration type with the following values:

* `node-label` (default)
  * This rack ID type maintains the existing behavior where rack IDs are configured using a node label
  * The node label is determined using the existing `topologyKey` field
* `pool-name`
  * This rack ID type will use the node pool name as the rack ID

When using the `pool-name` rack ID type, users must configure pod affinity and anti-affinity in the Kafka
pod template to ensure:

* Brokers within a pool are scheduled in the same availability zone
* Brokers in different pools are scheduled in different availability zones

This affinity configuration would be configured in the KafkaNodePool:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: my-zone0-pool
  labels:
    strimzi.io/cluster: my-cluster
spec:
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

## Affected/not affected projects

Only the [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator/) would be affected:

* Changes to the `Rack` API type
  * New optional `idType` field
  * Change `topologyKey` field from required to optional
* Changes to the cluster operator to:
  * Only create the ClusterRoleBinding when using the `node-label` rack ID type or when using a NodePort listener
  * Modify the `KafkaBrokerConfigurationBuilder` to use the pool name as the rack ID when the rack ID type is `pool-name`

## Compatibility

This proposal maintains CRD compatibility by introducing a new, optional field.
All existing configurations would continue to be valid and maintain their existing behavior.

## Rejected alternatives

* Continue with existing rack awareness support using Kubernetes API access in a broker init container
  * This alternative was rejected due to the heightened security requirements of some users, as outlined in the [Motivation](#motivation)
* Use node topology labels via the downward API as suggested in [strimzi-kafka-operator#11504](https://github.com/strimzi/strimzi-kafka-operator/issues/11504)
  * This alternative was rejected due to a reliance on a Kubernetes feature that is only in alpha as of 1.33
