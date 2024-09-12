# Templating `host` and `advertisedHost` fields in the `Kafka` custom resource

This proposal discusses the possibility to template the `host` and `advertisedHost` fields in the Listener configuration in the `Kafka` custom resources.

## Current situation

Users using the `type: ingress` listener to expose their Apache Kafka cluster outside of its Kubernetes cluster currently need to configure the hostname that will be used for each of the Kafka brokers as well as the one used for bootstrapping the client connections.
These hostnames (configured in the `host` field) are used in the Ingress resources created by Strimzi Cluster Operator.
The following example shows how a `type: ingress` listener configuration for a Kafka cluster with 3 brokers looks like:

```yaml
listeners:
  # ...
  - name: external
    port: 9094
    type: ingress
    tls: true
    authentication:
      type: tls
    configuration:
      bootstrap:
        host: my-cluster-bootstrap.myingress.com
      brokers:
        - broker: 0
          host: my-cluster-kafka-0.myingress.com
        - broker: 1
          host: my-cluster-kafka-1.myingress.com
        - broker: 2
          host: my-cluster-kafka-2.myingress.com
```

Notice the configuration section listing all 3 brokers and configuring a different `host` for each of them.
If the `host` configuration would be missing for any of the brokers, an error will be raised by Strimzi and the custom resource will not be reconciled because Strimzi would not know what should be the address of the missing broker.

A very similar situation happens with the `advertisedHost` field when the users want to customize their listeners.
While in this case it is not required by Strimzi to configure the `advertisedHost` for each broker, it is often necessary for the listener to work.
One such example is in the recent [blog post about using the Gateway API](https://strimzi.io/blog/2024/08/16/accessing-kafka-with-gateway-api/):

```yaml
listeners:
  - name: obiwan
    port: 9092
    tls: true
    type: cluster-ip
    configuration:
      brokerCertChainAndKey:
        certificate: tls.crt
        key: tls.key
        secretName: my-certificate
      brokers:
        - advertisedHost: broker-10.strimzi.gateway.api.test
          advertisedPort: 9092
          broker: 10
        - advertisedHost: broker-11.strimzi.gateway.api.test
          advertisedPort: 9092
          broker: 11
        - advertisedHost: broker-12.strimzi.gateway.api.test
          advertisedPort: 9092
          broker: 12
```

## Motivation

Being required to specify these configurations for each broker has some negative side-effects:
* The `Kafka` custom resource gets unnecessarily complicated, especially for bigger clusters.
  This also increases the likelihood of typos and other configuration issues.
* With the introduction of KRaft and node pools, the broker IDs are not always just 0, 1, 2 etc.
  The brokers might now have different numbers and might include gaps.
  The controller nodes share the same node ID space, but their hostnames do not need to be configured.
  All of these easily lead to confusion of how should this configuration look like.
* When adding new brokers, users need to make sure the `host` or `advertisedHost` fields will be added for the new brokers before scaling the cluster up.

But in most cases, the (advertised) hostnames often follow the same pattern.
In the example above, you can see that the Ingress based listener uses the hostname pattern `my-cluster-kafka-<NODE-ID>.myingress.com`.
And the customized listener uses the advertised hostname pattern `broker-<NODE-ID>.strimzi.gateway.api.test`.

While there might be users using more random hostnames, for many Strimzi users, having an easy way to template the hostnames as suggested in this proposal would provide a huge simplification:
* Users will be able to scale-up without changing the configuration first
* The `Kafka` CR will be much smaller, simpler and easier to read
* The likelihood of typos and misconfigurations will be smaller

## Proposal

This proposal suggests adding two new fields to the `configuration` section of the listener:
* `advertisedHostTemplate`
* `hostTemplate`

The fields will be strings used to define templates for generating hostnames.
These templates will support variables, allowing for flexible configurations:
* The `{nodeId}` variable will be replaced with the ID of the Kafka node to which the template is applied.
* The `{nodePodName}` variable will be replaced with the Kubernetes pod name for the Kafka node where the template is applied.

Strimzi will take these template fields and replace the variables with the corresponding value for each Kafka node.
And the resulting value would be used for the (advertised) hostnames.

The templates will only be applied to per-broker values. 
The hostname for the bootstrap Ingress resource must be specified manually and will not be generated from the template. 
The advertised hostnames are used only for per-broker configurations and cannot be configured for bootstrap.

### Handling conflicts

It is possible that the user specifies both the `advertisedHostTemplate` / `hostTemplate` as well as the `advertisedHost` / `host` fields for one or more Kafka brokers.
In such a case, Strimzi will always use the `advertisedHost` / `host` fields first.
Only when it is missing, it will use the template to generate the value.

### Examples

The following examples show how the YAMLs from the introduction of this proposal would be simplified with the use of the template.
The Ingress type listener configuration based on the template would look like this:

```yaml
listeners:
  # ...
  - name: external
    port: 9094
    type: ingress
    tls: true
    authentication:
      type: tls
    configuration:
      bootstrap:
        host: my-cluster-bootstrap.myingress.com
      hostTemplate: "{nodePodName}.myingress.com"
```

The YAML from the Gateway API blog post could be simplified as follows:

```yaml
listeners:
  - name: obiwan
    port: 9092
    tls: true
    type: cluster-ip
    configuration:
      brokerCertChainAndKey:
        certificate: tls.crt
        key: tls.key
        secretName: my-certificate
      advertisedHostTemplate: broker-{nodeId}.strimzi.gateway.api.test
```

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

This proposal is fully backwards compatible.
The existing configurations with the (advertised) hostnames configured per-broker will continue to work without any change.

## Rejected alternatives

### Supporting additional variables

Originally, I considered supporting some additional variables in the template fields:
* `{clusterName}` will be replaced with the name of the Kafka cluster (name of the `Kafka` CR)
* `{namespace}` will be replaced with the namespace of the Kafka cluster (namespace where the `Kafka` CR exists)

But at the end I decided to not include them in the proposal.
These values do not differ between the different Kafka nodes and can be easily handled by the user.
If the `Kafka` CR is generated from some kind of template (e.g. Helm), the cluster name / namespace can be easily templated outside of Strimzi.
If the `Kafka` CR is written manually, these can be easily specified by the user.

If we see sufficient demand for these variables, we can easily add them later.
Removing them later after we add them initially would be much more complicated.
