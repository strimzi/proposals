# Deprecate `type: ingress` listeners

This proposal suggests deprecating the `type: ingress` listener API in the `Kafka` custom resources.
It does not suggest a deadline for removal of the underlying code.

## Motivation

The `type: ingress` listener API allows exposing the Kafka cluster using the Kubernetes Ingress API.
However, Kafka has specific non-standard requirements (such as TLS passthrough).
And that always limited the compatibility with different Ingress controllers.
So the `type: ingress` listener API was designed around the [Kubernetes Ingress NGINX controller](https://github.com/kubernetes/ingress-nginx).

However, the Kubernetes Ingress NGINX controller is now being retired.
And all maintenance effort will be stopped in March 2026.
You can read more about it in the related [Kubernetes blog post](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/).

In addition to the archiving of the Kubernetes Ingress NGINX controller, the Ingress API itself is being replaced by the new Gateway API.
So we should consider the future of the `type: ingress` listener in Strimzi.

## Proposal

This proposal suggests to deprecate the `type: ingress` listener API.
This will indicate to users that they should consider using another listener type.
And that we do not plan to rebase the `type: ingress` listener to any other Ingress controller.
The API is planned for removal when the `v2` API is introduced.

The implementation will remain in the Strimzi code base for the foreseeable future.
Any future removal would be proposed and agreed separately.
If appropriate, such a proposal could align the removal with the introduction of the v2 API.

_Note: Strimzi has recently moved to `v1` API. The `v2` API is not expected in the near term._

The documentation and release notes will be updated accordingly as well.
Deprecation warnings will also be automatically raised by the operator in its logs and in the `Kafka` CR `.status` section.

## Affected projects

This proposal affects the Strimzi Operators project only.
It impacts the following components:
* `api` module (deprecation of the API)
* `cluster-operator` module (suppress the Java compiler deprecation warnings)
* Documentation

## Backwards compatibility

This proposal is fully backwards compatible.
While `type: ingress` listeners will be marked as deprecated, this change will have no impact (other than the deprecation warnings) on the Kafka clusters using it.

## Rejected alternatives

### Removing the support for `type: ingress` listeners completely

We could remove the `type: ingress` support completely from the code base.
However, this alternative was rejected because some users might want to keep using the Kubernetes Ingress NGINX controller despite its archival.
For example in disconnected environments where the associated security risks are mitigated.
Given its open source nature, they can also maintain their own private fork with security fixes.
Or they might use one of the commercial offerings that provide security patches for it despite its archival.

### Supporting other Ingress controllers

We could rebase the `type: ingress` listener API to support one of the other open source Ingress controllers.
However, this alternative was rejected because that might break support for Kubernetes Ingress NGINX controller users.
And because the Ingress API is being replaced by the Gateway API (Strimzi might provide a dedicated Gateway APi support in the future, but that is not subject of this proposal).
Users who want to use a different Ingress controller can always do so using the `type: cluster-ip` listener and manually maintained Ingress resources.
