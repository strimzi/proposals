# Node Pool Rack IDs

Rack-awareness in Strimzi Kafka can be achieved without broker access to cluster-level Kubernetes APIs
by configuring a rack ID in each node pool for usage across all brokers within the pool.

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
Many Kubernetes cluster administrators may restrict access to cluster-scoped Kubernetes
resources to ensure an application and the user managing it are contained within a limited set of namespaces.
Today, Strimzi only requires access to cluster-scoped Kubernetes resources for rack-awareness and NodePort
listener configuration.

Implementing the proposed method for pool-based rack awareness removes these potentially prohibitive
requirements while maintaining a simple deployment model.

## Proposal

The `KafkaNodePool` CR will be updated to include an `rackId` field which configures the rack ID for
brokers in that node pool.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: my-zone0-pool
  labels:
    strimzi.io/cluster: my-cluster
spec:
  rackId: zone0
```

The field is a string and takes precedence over rack awareness configuration in the `Kafka` CR.

When using a pool rack ID, users must configure pod affinity and anti-affinity in the Kafka
pod template to ensure:

* Brokers within pools with the same rack ID are scheduled in the same availability zone
* Brokers in pools with different rack IDs are scheduled in different availability zones

This affinity configuration would be specified in the KafkaNodePool:

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

## Affected/not affected projects

Only the [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator/) would be affected:

* Changes to the `KafkaNodePool` API type
  * New optional `rackId` field
* Changes to the cluster operator to:
  * Modify the `KafkaBrokerConfigurationBuilder` to use the rack ID specified in the KafkaNodePool if defined

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

Item (3) can be achieved using a similar configuration to the one described [above](#proposal).
The rack ID configured for each broker is a zone identifier, e.g. `zone0`, and not the name of the
actual topology node label. When rack IDs in brokers are configured to a specific topology node label,
e.g. `us-east-2`, any changes to this label would require restarting the brokers. If I wanted to change
a set of my nodes to use label `us-east-2a`, I may not want my broker pods to immediately restart
with this change.

## Rejected alternatives

* Continue with existing rack awareness support using Kubernetes API access in a broker init container
  * This alternative was rejected due to the heightened security requirements of some users, as outlined in the [Motivation](#motivation)
