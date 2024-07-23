# Deprecation and removal of Storage overrides

Currently, when configuring persistent-volume-claim storage, users can use per-broker overrides to override the storage class used by the individual brokers.
This feature should not be needed anymore with Kafka Node Pools and should be deprecated and removed.

## Current situation

Currently, users can override the storage class used by persistent-volume-claim storage on a per-broker basis.
The following example:

```yaml
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        deleteClaim: false
        class: my-storage-class
        overrides:
        - broker: 0
          class: my-storage-class-zone-1a
        - broker: 1
          class: my-storage-class-zone-1b
        - broker: 2
          class: my-storage-class-zone-1c
```

Would create a cluster where:
* Broker with node ID 0 uses the storage class `my-storage-class-zone-1a`
* Broker with node ID 1 uses the storage class `my-storage-class-zone-1b`
* Broker with node ID 2 uses the storage class `my-storage-class-zone-1c`
* All other brokers use storage class `my-storage-class`

Using a different storage class per broker can be useful in some special use-cases.
For example when you need to guide the scheduling of the different brokers to different zones / nodes.
But this kind of configuration has some challenges as well.
Unless the broker ID is listed in the overrides, it will use the default storage class.
So when you need to scale-up your Kafka cluster, you have to make sure it has the entries for the node IDs that will be added.

## Motivation

The overrides were very useful while all nodes were configured in the `Kafka` custom resource.
But the Kafka Node Pools features was designed with the idea of using different configurations for different sets of nodes from the beginning.
Users using node pools can have multiple different node pools - each representing a subset of Kafka nodes - with different configuration.
This includes a different storage class.
It also does need the user to specify exact node IDs as the storage configuration will be used for all nodes belonging to a given node pool.
So it simplifies the configuration.

In addition to simplifying the configuration for the users, it will also allow us to streamline the code and the API.
In particular, the _storage diffing_ needed to avoid undesired changes will be simplified.

## Proposal

The persistent-volume-claim storage overrides will be deprecated immediately in Strimzi 0.43.
The will remain supported and used, but users would be encouraged to move away from them and use Kafka Node Pools instead.
This also falls into the time period when we expect most users to migrate to node pools and to KRaft which provides a good opportunity to move to per-node-pool storage class instead of using the overrides.

Later, in the Strimzi version where support for ZooKeeper-based Kafka clusters is removed, the support for the storage overrides will be dropped and the overrides will be ignored.
The overrides will remain in the CRD API for backwards compatibility but will not be used by the operator code.
Finally, when moving the new `v1` CRD API in the future, the fields will be completely removed.

Kubernetes do not allow changing storage class for existing persistent volumes. 
If any user doesn't migrate from the overrides in time, the existing Kafka nodes and persistent volumes will not be affected.
Only when the PVC/PV is deleted or when new nodes will be added during a scale-up, the overrides will be ignored and the default storage class will be used.
So the impact on existing users should be minimal.

### Warnings

With the deprecation of the storage overrides, the operator will be updated to issue warnings when the overrides are used.
The warnings will be printed in regular logs from the Cluster Operator.
And they will be also added to the conditions in the `Kafka` CR status.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

This proposal removes the support for storage overrides in Kafka clusters.
It will impact all users using it and they will have to migrate to node pools with different storage classes.
Kafka clusters not using the overrides and any other operands supported by Strimzi will not be impacted in any way.

## Rejected alternatives

### Different timeline

The timeline between the deprecation of the overrides and the removal of their support is relatively short (likely 3-4 Strimzi versions).
If we think this is a problem, we can choose a different timeline.
For example, we can deprecate the overrides right now, but postpone when we drop the support for them:
* Drop the support only at a later version (e.g. first Strimzi release after June 2025)
* Drop the support only when we migrate to the `v1` CRD API

This might give users more time to handle it.
However, while the migration time window is relatively short, I believe it is sufficient for most existing users, as it includes the migration to KRaft and node pools, and provides a good opportunity to remove this legacy feature.
