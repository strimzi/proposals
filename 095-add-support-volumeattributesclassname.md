# Add `volumeAttributesClassName` to the storage configuration

Volumes in Strimzi are configured using the `PersistentClaimStorage` model. This is part of the Strimzi API's `Kafka` and `KafkaNodePool` among others. This proposal goes over the ability to set `volumeAttributesClassName` in the configuration for `PersistentClaimStorage`. `VolumeAttributesClass` provides the ability to decouple storage parameters like IOPS, throughput, fstype or any other cloud specific ones from the `StorageClass`.

## Current situation

It is not possible to set the `volumeAttributesClassName` using [`PersistentClaimStorage`](https://github.com/strimzi/strimzi-kafka-operator/blob/c1b20f726dddbcd2a070c2eeb14fd30902027aec/api/src/main/java/io/strimzi/api/kafka/model/kafka/PersistentClaimStorage.java). To change any configuration parameters of the physical volume backing a `PersistentVolumeClaim`, users have to rely on the `StorageClass`'s `spec.parameters` field. This couples the parameters and the class.

## Motivation

Kubernetes v1.31 added a new method of configuring storage parameters for `PersistentVolumes` (PV) using [`VolumeAttributesClass`](https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/) (VAC). These VACs are a cluster-wide resource. `PersistentVolumeClaims` (PVC) can then refer to them using `volumeAttributesClassName` along with their corresponding `storageClassName`. This decouples storage parameters specification from the `StorageClass` (SC) into the VAC. In a PVC, the `storageClassName` field is immutable, whereas the `volumeAttributesClassName` isn't. This makes it possible to dynamically reconfigure the PV without losing data.

## Proposal

To accommodate this change, the `PersistentClaimStorage` (PCS) API needs an additional string field `volumeAttributesClass`. When this field changes in the PCS, the Cluster Operator (CO) can map it to the generated PVC's `volumeAttributesClassName` and let the CSI Driver take care of provisioning. There shouldn't be a need to validate the VAC since the parameter names depend on the cloud provider.

The [`external-provisioner`](https://github.com/kubernetes-csi/external-provisioner) `csi-provisioner` usually checks whether the `driverName` in the VAC is the same as the `provisioner` in the SC. The operator could check that pre-emptively, but cannot depend on the CSI driver to use that implementation of a csi-provisioner. The user would be responsible for making sure the VACs and the SCs are configured correctly before configuring the PCS.

An example configuration could be

```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: pool-a
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
  - broker
  resources:
    requests:
      cpu: 1
      memory: 512Gi
    limits:
      cpu: 500m
      memory: 256Gi
  storage:
    type: persistent-claim
    size: 16Gi
    class: ebs
    volumeAttributesClass: ebs-fast
    deleteClaim: true
```

which produces the following PVC

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <generated-name>
spec:
  accessMode:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    request:
      storage: 16Gi
  storageClassName: ebs
  volumeAttributesClassName: ebs-fast
```

Here the user is expected to have a VAC named ebs-fast which has the same `driverName` as the `provisioner` of the SC named ebs. They are also expected to have the right parameters for the corresponding cloud provider's CSI driver.

When a user decides to update the `volumeAttributesClassName` of a PVC, the CSI driver will apply the changes. This update will ideally happen via the PCS which in-turn updates the PVC. This might result in a cloud provider error being thrown as well. This error will be an event on the corresponding PVC's status. In that case, it is expected that the user deals with fixing any errors with the VAC and SC. For example, the AWS EBS CSI driver posts this error when changing the VAC right after provisioning it and the user is expected to fix it.

```
  Warning  VolumeModifyFailed     26s (x5 over 66s)      external-resizer ebs.csi.aws.com                                                          rpc error: code = Internal desc = Could not modify volume "vol-0fa7ff557865862c2": volume "vol-0fa7ff557865862c2" in OPTIMIZING state, cannot currently modify
```

## Affected/not affected projects

Cluster Operator is the only resource affected.

## Compatibility

No issues expected. The `volumeAttributesClassName` is optional in a PVC.

