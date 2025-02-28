# Deprecate and remove `type: opa` authorization in Kafka CR

This proposal suggests to deprecate and later remove the Open Policy Agent (OPA) authorization (`type: opa`).

## Current situation

Strimzi currently supports [Open Policy Agent authorization plugin](https://github.com/StyraInc/opa-kafka-plugin) for Kafka brokers.
The plugin is bundled as part of our Apache Kafka container images.
Users can configure it in the `Kafka` custom resource using the `type: opa` authorization.
For example:

```yaml
    authorization:
      type: opa
      url: http://opa:8181/v1/data/kafka/crd/authz/allow
      expireAfterMs: 60000
      superUsers:
        - my-super-user
```

## Motivation

Supporting the `type: opa` authorization and bundling the plugin in our images is not for free:
* We need to maintain the code in the Cluster Operator
* With every new Kafka release, we need to make sure the dependencies are aligned between Kafka, other plugins and the OPA Authorizer plugin
* We need to maintain the system tests and make sure we use an reasonably up-to-date OPA version

While some users appear to be using the `type: opa` authorization, it does not seem to be widely adopted.
For the users using the OPA authorizer, there is also a possible workaround.
They can continue using the OPA authorizer plugin even after we remove the dedicated support for it by following these steps:
1. Add the OPA authorizer plugin to the Kafka container image
2. Use the `type: custom` authorization to configure the OPA authorizer.
   The OPA authorizer class will be specified as part of the `authorization` section.
   The additional options can be specified in the `config` section.
   For example:
   ```yaml
   # ...
   kafka:
     # ...
     authorization:
       type: custom
       authorizerClass: org.openpolicyagent.kafka.OpaAuthorizer
       superUsers:
         - my-super-user
    config:
      opa.authorizer.url: http://opa:8181/v1/data/kafka/crd/authz/allow
      opa.authorizer.cache.expire.after.seconds: 60
    # ...
   ```

Given the available workaround and the maintenance effort, it seems reasonable to deprecate and remove the direct support for OPA authorizer from Strimzi.
It also helps to make Strimzi project leaner and rely more on pluggability instead.

## Proposal

This proposal suggests to immediately within Strimzi 0.46:
* Deprecate the `type: opa` authorization
* Update the documentation to not use this deprecated field and use the `type: custom` authorization instead
* Update the `CHANGELOG.md` file and documentation to inform users about this deprecation
* Have warnings raised by the Cluster Operator when the `type: opa` authorizer is used

While deprecated, we will still continue bundling the OPA authorizer plugin as part of Strimzi.

When the Strimzi `v1` CRD API is added, it will not support the `type: opa` anymore.
But as the `type: opa` authorization will be still part of the `v1beta2` API, the support in Cluster Operator and in container images has to remain.

Only in the first Strimzi version that drops the support for the `v1beta2` API and supports the `v1` API only, we will:
* Stop bundling the OPA authorizer plugin in the Strimzi container images
* Remove the production code for configuring the OPA authorization
* Remove the OPA system test
* Update the documentation to remove the `type: opa` authorization content.

From this version on, users will have to use a custom container image to add the OPA authorizer plugin and the `type: custom` authorization to use it.

## Affected projects

This proposal affects the Strimzi Cluster Operator, System Tests, and the documentation.

## Backwards compatibility

Users using the `type: opa` authorization will be impacted by this changes as they will need to start using custom container images and update the Kafka CR resources.
Other users will not be impacted.

## Rejected alternatives

As an alternative path, we could consider dropping the OPA support completely already before the `v1` CRD API.
For example drop the binaries and stop using the `type: opa` authorization already in an earlier Strimzi version such as Strimzi 0.48.
However, I decided to start the proposal with the OPA authorization removal as part of the `v1beta2` API version removal.
