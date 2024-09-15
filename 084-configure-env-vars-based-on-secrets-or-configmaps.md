# Configure environment variables in all containers from Secrets or Config Maps

This proposal introduces a new feature to allow configuring environment variables in any container deployed by Strimzi based on a value from a Secret or Config Map.

## Current situation

[Strimzi Proposal 75](https://github.com/strimzi/proposals/blob/main/075-additional-volumes-support.md) that was approved and implemented as part of Strimzi 0.43.0 gave Strimzi users the ability to mount custom volumes into any container deployed by Strimzi Cluster Operator.
This includes mounting volumes based on Secrets and Config Maps.

The [`ContainerEnvVar`](https://strimzi.io/docs/operators/latest/full/configuring.html#type-ContainerEnvVar-reference) API allows users to also specify environment variables for any container.
But it currently allows users to set the environment variables only to plain text values directly inlined in the Strimzi custom resources.  
The only exception to this are the Kafka Connect and MirrorMaker 2 containers, where additional environment variables based on Secret or Config Map can be added through the [external configuration](https://strimzi.io/docs/operators/0.43.0/full/configuring.html#type-ExternalConfiguration-reference) option.

## Motivation

The ability to specify environment variables for various containers based on Secret or Config Map comes up from time to time as a question or a feature request from our users.
Often it is part of some advanced customization.
For example passing credentials to various injected agents or passing decryption keys to files mounted from a Secret as a volume (for example when an encrypted certificate store is mounted as a volume and should be used).

In addition, we now have an asymmetry in the Strimzi API design:
* Users can now use Secrets and Config Maps as volumes in any container.
* The [external configuration](https://strimzi.io/docs/operators/0.43.0/full/configuring.html#type-ExternalConfiguration-reference) part related to volumes has been deprecated and is planned to be removed in the future (with the v1 CRD API).
* Environment variables in containers (except Kafka Connect and MirrorMaker 2 containers) can be defined only as plain values.
* Environment variables based on Secrets and Config Maps can be defined for Kafka Connect and MirrorMaker 2 containers in the part of [external configuration](https://strimzi.io/docs/operators/0.43.0/full/configuring.html#type-ExternalConfiguration-reference) related to environment variables.
  This part is currently not deprecated and not planned to be removed.

## Proposal

The [`ContainerEnvVar`](https://strimzi.io/docs/operators/latest/full/configuring.html#type-ContainerEnvVar-reference) will be extended to allow creating environment variables from a Secret or ConfigMap.
To do this, a new field `valueFrom` would be added to this API.
The `valueFrom` field would be configured together with the existing `value` field as `oneOf` alternatives, so that only one of them can be set.

The `valueFrom` field will allow to reference a key inside a Secret of Config Map.
This value under this key in the Secret or Config Map would be used as the environment variable in the container.
Strimzi itself will not directly extrapolate the content of the Secret or Config Map.
It will configure the container environment variables to reference the Secret or Config Map and leave the extrapolation to Kubernetes.

The enhanced `CointainerEnvVar` API would allow following options:

```yaml
    env:
      - name: PLAIN_TEXT_ENV_VAR
        value: "I'm plain text"
      - name: SECRET_ENV_VAR
        valueFrom:
          secretKeyRef:
            name: my-custom-secret
            key: my-password
      - name: CONFIG_ENV_VAR
        valueFrom:
          configMapKeyRef:
            name: my-custom-cm
            key: my-custom-option
```

In the same way as already done for the additional volumes, Strimzi will not validate the existence of the referenced Secret or Config Map.
This validation will be done by Kubernetes when creating the Pod / Container.
If the Secret or Config Map would not exist or would not contain the correct key, The Pod will not be started.
This is considered acceptable given this is an advanced option.

### Specifying environment variables based on other sources

In addition to Secrets and Config Maps, Kubernetes also allows to set environment variables in containers based on other sources such as the Downward API.
Strimzi will currently not support any additional sources other than Secrets and Config Maps.
These sources can be added later based on a separate proposal.

### Security risks

The ability to use values from Secrets and Config Maps as environment variables is not validated against the RBAC rules of the user (or a service account) who creates the Strimzi custom resource.
Following situation might occur and allow the user bypass the RBAC rights it has:
1. Secret `my-secret` exists in a namespace watched by the Strimzi Cluster Operator
2. User does not have the RBAC rights to read the Secret, but has the right to create Strimzi custom resources and exec into the Pods created by Strimzi
3. The user creates any Strimzi custom resource and maps values from the Secret to environment variables
4. Strimzi creates the Pod/container with the value from the Secret mapped to an environment variable
5. User execs into the container created by Strimzi and checks the value of the environment variable

This risk should be considered acceptable, mainly because:
* This is standard Kubernetes behavior as the same could be done with any Kubernetes Workload API.
* This can be already done while mounting the Secret or Config Map as a volume.
* This risk already exists in Kafka Connect or MirrorMaker 2 through the external configuration.
* The Secret or ConfigMap need to be in the same namespace os the Strimzi custom resource.
  It would be rare that a user can create custom resources and exec into Pods in an namespace but cannot access some of its Secrets or Config Maps.

### Deprecation of the external configuration API

Together with the implementation of this new feature, the rest of the [external configuration](https://strimzi.io/docs/operators/0.43.0/full/configuring.html#type-ExternalConfiguration-reference) will be marked as deprecated and replaced by the container template.
It will be removed as part of the CRD v1 API.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

This proposal adds new API fields that are planned to be supported in the long term.
The existing [external configuration](https://strimzi.io/docs/operators/0.43.0/full/configuring.html#type-ExternalConfiguration-reference) API will be deprecated and scheduled for removal in the CRD v1 API.
Until then, it will be supported and any environment variables configured through it will be used.

## Rejected alternatives

There are currently no rejected alternatives.
