# Deprecate `secrets` field in `type: custom` authentication in `Kafka` CR

This proposal suggests to deprecate and later remove the `secrets` field of the `type: custom` authentication in the Kafka custom resource.

## Motivation

When using [`type: custom` authentication](https://strimzi.io/docs/operators/latest/full/configuring.html#type-KafkaListenerAuthenticationCustom-reference) (introduced in Strimzi 0.28) in Kafka brokers, users can mount any Secret resources that are used by their custom authentication mechanism.
These Secrets are mounted into the `/opt/kafka/custom-authn-secrets/custom-listener-<listener_name>-<port>/<secret_name>` directory and users can reference them by file path in their configuration.
Strimzi does not do any validation or sanitization of their content and does not have any special handling for these Secrets.
It just mounts them into the container.

Recently, we introduced support for mounting custom volumes to any Strimzi container.
This feature was introduced by [SP075 - Support for additional volumes](https://github.com/strimzi/proposals/blob/main/075-additional-volumes-support.md) in Strimzi 0.43.
It allows to mount Secrets, but also Config Maps, PVCs or CSI volumes.
Similarly to the Secrets from the `type: custom` authentication, Strimzi does not do any validation or sanitization of their content and does not have any special handling for these Secrets.
They are just mounted into the `/mnt` path where they can be used by the user.

The additional volumes feature can be used to replace the `secrets` field in the `type: custom` authentication.
Having two different features to cover the same thing seems to be unnecessary
* It creates more complex API and bigger CRD(s)
* It causes more complex / duplicate code for adding the volumes to Pods
* It causes more complex / duplicate code for adding the volume mounts to containers

## Proposal

This proposal suggests to immediately:
* Deprecate the `secrets` field in `type: custom` authentication object
* Update the documentation to not use this deprecated field and use the additional volumes instead
* Update the `CHANGELOG.md` file and documentation to inform users about this deprecation
* Have warnings raised by the Cluster Operator when the deprecated field is used

While deprecated, the `secrets` field will continue to work as before deprecation.

Later, the `secrets` field will be removed in the Strimzi `v1` CRD API.
And once the field is completely removed from the Strimzi API (i.e. after we drop support for `v1beta2` API), the code using the field will be removed as well.

### Why deprecate only this field?

This proposal targets the deprecation and removal of the `secrets` field in the `type: custom` authentication because it is used to mount opaque Secrets without any validation or special handling and as such it provides the same functionality as the additional volumes.
This is similar to how we already deprecated the external configuration volumes in Kafka Connect.

This proposal does not aim to deprecate any fields used to mount specific Secrets such as:
* Server certificates
* Trusted certificates
* Client certificates
* OAuth client IDs or Secrets
* Passwords

In these places we usually do additional validations, handle reloading of these Secrets etc.
This cannot be replaced by the additional volumes functionality.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

There is no impact on backwards compatibility.

## Rejected alternatives

There are currently no rejected alternatives.
