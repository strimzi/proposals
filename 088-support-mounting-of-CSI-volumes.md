# Support for mounting CSI volumes

Proposal [75 - Support for additional volumes](https://github.com/strimzi/proposals/blob/main/075-additional-volumes-support.md) introduced the possibility to mount additional volumes into Strimzi operand Pods.
It added support for the following volumes types:
* Secrets
* Config Maps
* EmptyDir volumes
* Persistent Volume Claims

This proposal follows up on it and proposes adding support for CSI volumes.

## Motivation

Mounting Persistent Volume Claims is useful to provide additional data volumes.
For example to store logs or for tiered storage.
EmptyDir volumes are useful as a temporary storage.
Finally, Kubernetes Secrets or Config Maps is useful to provide additional configuration data or credentials.

But in some cases, Kubernetes Secrets might not be the right way to store credentials.
And users might instead want to use other mechanisms to load them.
One of them might be special CSI drivers for mounting them directly.
For example:
* [cert-manager CSI Driver](https://cert-manager.io/docs/usage/csi/) can be used to mount certificates or SPIFFE identities
* [Secret Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/introduction) for mounting secrets from enterprise-grade secret stores such as Vault, AWS Secret Manager etc.

CSI volumes can be also used to directly mount data volumes without the use of the Persistent Volume Claims as the _intermediaries_.
While this might be useful in some cases, the main goal of this proposal are the special types of CSI drivers such as those mentioned above rather than data volumes.
But nothing will prevent users from using it to mount data volumes as well (and there is no reason to prevent such use).

## Proposal

In order to support the CSI volumes, a new field named `csi` will be added to the [`AdditionalVolume` class](https://github.com/strimzi/strimzi-kafka-operator/blob/87935da1fae794bab473a0470cbea214369ac985/api/src/main/java/io/strimzi/api/kafka/model/common/template/AdditionalVolume.java#L41).
This field will use the Fabric8 type `CSIVolumeSource` and map to the [Kubernetes `CSIVolumeSource` structure](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#csivolumesource-v1-core).
This would allow users to defined the CSI volumes in the container template fields.
For example, to mount a Cert Manager certificate in Kafka Connect, users can use the following YAML:

```yaml
  template:
    connectContainer:
      volumeMounts:
        - name: certificate
          mountPath: /mnt/certificate/
    pod:
      volumes:
        - name: certificate
          csi:
            driver: csi.cert-manager.io
            readOnly: true
            volumeAttributes:
              csi.cert-manager.io/issuer-name: my-ca
              csi.cert-manager.io/dns-names: ${POD_NAME}.${POD_NAMESPACE}.svc.cluster.local
```

This YAML will generate a new certificate using cert-manager and its Issue named `my-ca` and mount it in the `/mnt/certificate/` path.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

There is no impact on backwards compatibility.

## Rejected alternatives

There are currently no rejected alternatives.
