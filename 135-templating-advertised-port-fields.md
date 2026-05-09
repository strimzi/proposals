# Templating `advertisedPort` fields in the `Kafka` custom resource

## Current situation

In the [Strimzi Proposal #84](https://github.com/strimzi/proposals/blob/main/084-templating-host-and-advertisedHost-fields.md) we introduced the `hostTemplate` and `advertisedHostTemplate` fields.
These fields act as alternatives to the `host` and `advertisedHost` fields, which are configured on a per-broker basis.
They allow users to provide a single template that is used to generate the (advertised) host for all brokers.

However, the original proposal did not provide any simplification for configuring advertised ports.
They still have to be configured on a per-broker basis in `advertisedPort` fields.
This proposal aims to improve this and provide a similar solution for ports as we have for hosts and advertised hosts.

## Motivation

When using mechanisms such as:
  * Gateway API `TLSRoutes` or `TCPRoutes`
  * custom Ingress controllers
  * or others

Strimzi users often need to override the advertised host and port fields.
From the beginning, we have allowed these options to be overridden with configurations such as:

```yaml
listeners:
  - name: listener1
    port: 9094
    tls: true
    type: cluster-ip
    configuration:
      brokers:
        - advertisedHost: broker-10.strimzi.gateway.api.test
          advertisedPort: 8443
          broker: 10
        - advertisedHost: broker-11.strimzi.gateway.api.test
          advertisedPort: 8443
          broker: 11
        - advertisedHost: broker-12.strimzi.gateway.api.test
          advertisedPort: 8443
          broker: 12
```

[Strimzi Proposal #84](https://github.com/strimzi/proposals/blob/main/084-templating-host-and-advertisedHost-fields.md) further improved it by adding the `advertisedHostTemplate`.
So users can now use something like this:

```yaml
listeners:
  - name: listener1
    port: 9094
    tls: true
    type: cluster-ip
    configuration:
      advertisedHostTemplate: broker-{nodeId}.strimzi.gateway.api.test
      brokers:
        - advertisedPort: 8443
          broker: 10
        - advertisedPort: 8443
          broker: 11
        - advertisedPort: 8443
          broker: 12
```

However, the advertised ports still had to be configured on a per-broker level.
This has several disadvantages:
* More complex YAML configuration.
* Users need to know upfront what the node IDs will be in order to configure the ports for the correct nodes.
  This is especially true with KRaft and Node Pools when the node IDs are not as predictable as they used to be.
* There is an increased risk of typos and misconfiguration.
* Auto-scaling the Kafka cluster is harder because it relies on per-broker configuration.

This proposal removes this limitation by introducing `advertisedPortTemplate`.

## Proposal

This proposal suggests adding a new field named `advertisedPortTemplate` to the listener `configuration` section.
The field is a string template to generate port numbers.
Unlike `advertisedHostTemplate` and `hostTemplate`, the `advertisedPortTemplate` field cannot rely on simple string substitution.
For example, a string-substitution template such as `900{nodeId}` would work well for node IDs such as `0`, `1`, or `2`.
But it would not work well for node IDs such as `100`.

Instead, for the port number template, this proposal suggests supporting a mathematical formula.
For example:
* `9000+{nodeId}` _(without whitespaces)_
* `9000 + {nodeId}` _(with whitespaces)_
* `{nodeId} + 9000` _(order does not matter)_
* `9000 + ({nodeId} * 10)` _(because why not 🤷‍♂️)_

This approach works much better for different node IDs, where the resulting port number is consistent regardless of whether the node ID is `1`, `100`, or `1000`.

The formula is evaluated using the [Javaluator library](https://github.com/fathzer/javaluator).
To give users more flexibility, the supported operators will be `+`, `-` and `*`.
We will also support `(` and `)` brackets.
The formula will support a single variable `{nodeId}`, which will represent the ID of the Kafka node for which the `advertisedPortTemplate` will be rendered.

Users who do not need a formula because the advertised port is the same for all brokers can simply specify the port number in the template.
In that case, they can omit both the `{nodeId}` variable and any mathematical operations.

The advertised ports are used only for per-broker configurations and cannot be configured for bootstrap.

### Handling conflicts

It is possible that the user specifies both the `advertisedPortTemplate` and the `advertisedPort` field for one or more Kafka brokers.
In such a case, Strimzi will always use the `advertisedPort` field first.
Only when it is missing will it use the template to generate the value.

### Examples

The following examples show how the YAML snippets from the introduction of this proposal would be simplified by using the template.
The listener configuration based on the template would look like this:

```yaml
listeners:
  - name: listener1
    port: 9094
    tls: true
    type: cluster-ip
    configuration:
      advertisedHostTemplate: broker-{nodeId}.strimzi.gateway.api.test
      advertisedPortTemplate: 10000 + {nodeId}
```

Or with a single fixed advertised port number:

```yaml
listeners:
  - name: listener1
    port: 9094
    tls: true
    type: cluster-ip
    configuration:
      advertisedHostTemplate: broker-{nodeId}.strimzi.gateway.api.test
      advertisedPortTemplate: 8443
```

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

This proposal is fully backwards compatible.
The existing configurations with the advertised port configured per-broker will continue to work without any change.

## Rejected alternatives

### Using the Exp4j library

Using the [exp4j library](https://github.com/fasseg/exp4j) was considered instead of the Javaluator library.
Exp4j has more GitHub stars than Javaluator.
But the project is not under active development anymore.
It also does not allow limiting which operators can be used in formulas.

### Allowing only to use a static port configuration

Not supporting any kind of template or formula was also considered.
We could add a central advertisedPort field to configure a static advertised port override.
In other words, the port will be configured only once and not per-broker and the same value would be used for all brokers.
This might be sufficient in some use cases.
But it would not work, for example, with Gateway API TCPRoutes, where each broker often needs to use a different port number.
