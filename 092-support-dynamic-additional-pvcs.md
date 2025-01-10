# Support for dynamically creating and mounting Persistent Volume Claims

Proposal [75 - Support for additional volumes](https://github.com/strimzi/proposals/blob/main/075-additional-volumes-support.md) introduced the possibility to mount additional volumes into Strimzi operand Pods. Persistent Volume Claim is one of the supported volume types. However, the Persistent Volume Claim resource itself has to be created manually prior to creating the Kafka resource. 

Persistent Volume Claims can also be used as Kafka and Zookeeper storage. However, in that case - in contrast to additional volumes - Strimzi is responsible for creating the resources dynamically based on the user-given parameters in [PersistentClaimStorage](https://strimzi.io/docs/operators/latest/configuring#type-PersistentClaimStorage-reference).
This proposal suggests adding support for Persistent Volume Claims as additional volumes, which are created by Strimzi dynamically based on user defined parameters (similarly to data Persistent Volume Claims).

## Motivation

Mounting Persistent Volume Claims is useful for providing additional data volumes, for example, to store logs or for tiered storage. If a volume is shared among the brokers, a single Persistent Volume Claim needs to be created beforehand and added to the additional volume list in the corresponding resource (e.g. Kafka, KafkaNodePool).

However, in other cases, when each broker needs a unique persistent volume, each Persistent Volume Claim must have a unique name - and since there is no possibility of per-broker override for this name - a separate KafkaNodePool must be created for each broker. This limitation defeats the purpose of KafkaNodePools in this specific use case (unique and persistent additional volume).

For large clusters it might not be the best user experience to define a high number of Persistent Volume Claims and a different KafkaNodePool for each broker.

Recently, proposal [88 - Support for mounting CSI volumes](https://github.com/strimzi/proposals/blob/main/088-support-mounting-of-CSI-volumes.md) enabled the usage of CSI volumes. However, the created volume type is [CSI ephemeral volume](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volumes), which does not cover the above use case (persistent volume).

## Proposal

In order to support Strimzi managed Persistent Volume Claims, a new field named `dynamicPersistentVolumeClaim` will be added to the [`AdditionalVolume` class](https://github.com/strimzi/strimzi-kafka-operator/blob/ac319a6ba597cd320980d157f947ae4d5686270a/api/src/main/java/io/strimzi/api/kafka/model/common/template/AdditionalVolume.java#L43).
The type of this field will will be the already existing [`PersistentClaimStorage` class](https://github.com/strimzi/strimzi-kafka-operator/blob/ac319a6ba597cd320980d157f947ae4d5686270a/api/src/main/java/io/strimzi/api/kafka/model/kafka/PersistentClaimStorage.java#L32).
This enhancement will allow users to define Persistent Volume Claim templates in the pod template fields.
For example, users can use the following YAML to create and mount a Persistent Volume Claim for each broker:

```yaml
  template:
    kafkaContainer:
      volumeMounts:
        - name: unique-persistent-vol
          mountPath: /mnt/unique-persistent-vol/
    pod:
      volumes:
        - name: unique-persistent-vol
          dynamicPersistentVolumeClaim:
            size: 100Gi
            storageClass: custom-storage-class
            deleteClaim: false
```

Similarly to data volumes, Strimzi will be responsible for creating the Persistent Volume Claim for the above volume.
The same fields can be provided under `dynamicPersistentVolumeClaim` as in case of other [PersistentClaimStorages](https://strimzi.io/docs/operators/latest/configuring#type-PersistentClaimStorage-reference).

Strimzi will use the following convention for naming the new Persistent Volume Claims: `volume-<kafka_cluster_name>-kafka-<pod_id>-<volume_name>` where volume_name would be `unique-persistent-vol` in the above example.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

There is no impact on backwards compatibility.

## Rejected alternatives

1. The already existing `persistentVolumeClaim` field could be extended in the [AdditionalVolume schema](https://strimzi.io/docs/operators/latest/configuring#type-AdditionalVolume-reference). In that case, users could define already existing as well as dynamically created Persistent Volume Claims with this field. This solution would be more elegant than defining another field for the same volume type, but it would also require changing the type of the existing field from [PersistentVolumeClaimVolumeSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#persistentvolumeclaimvolumesource-v1-core) to probably an aggregating type of the fields in `PersistentVolumeClaimVolumeSource` and `PersistentClaimStorage` in order not to affect backward compatibility. The implementation seems to be more complex and difficult to reason about and not sure at this point if feasible.
