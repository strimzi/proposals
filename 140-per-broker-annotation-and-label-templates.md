# Per-broker annotation and label templates for Kafka listeners

This proposal adds listener-level templates for generating per-broker annotations and labels on Services, Routes, and Ingresses created for Kafka listeners.

## Current situation

Strimzi already allows users to configure per-broker metadata for external and cluster-ip style listeners through `configuration.brokers[].annotations` and `configuration.brokers[].labels`.
This is useful for cases such as:

* configuring ExternalDNS annotations on per-broker `Service` resources
* attaching custom labels required by ingress controllers or internal platform tooling
* customizing per-broker resources for `loadbalancer`, `nodeport`, `route`, `ingress`, and `cluster-ip` listeners

A typical configuration today looks like this:

```yaml
spec:
  kafka:
    listeners:
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
        configuration:
          bootstrap:
            annotations:
              external-dns.alpha.kubernetes.io/hostname: kafka-bootstrap.example.com.
          brokers:
            - broker: 0
              annotations:
                external-dns.alpha.kubernetes.io/hostname: kafka-0.example.com.
              labels:
                dns.zone: primary
            - broker: 1
              annotations:
                external-dns.alpha.kubernetes.io/hostname: kafka-1.example.com.
              labels:
                dns.zone: primary
            - broker: 2
              annotations:
                external-dns.alpha.kubernetes.io/hostname: kafka-2.example.com.
              labels:
                dns.zone: primary
```

This works, but it becomes repetitive and error-prone as clusters grow or when node IDs are not simple `0,1,2` sequences.
That is especially visible in KRaft clusters using node pools, where users often need to think in terms of node IDs and pod names rather than replica ordinals.

## Motivation

In most real deployments, per-broker annotations and labels follow a predictable pattern.
Examples include:

* `external-dns.alpha.kubernetes.io/hostname: kafka-{nodeId}.example.com.`
* `my-company.io/pod-name: {nodePodName}`
* `topology.strimzi.io/broker-id: {nodeId}`

Requiring users to repeat these values broker by broker has several drawbacks:

* the `Kafka` custom resource becomes unnecessarily long
* copy-paste mistakes are easy to make
* scaling up requires editing the listener configuration first
* gaps in broker IDs make manual maintenance harder
* users cannot express “same pattern for all brokers, override only one broker if needed” cleanly

Strimzi already solved this problem for hostnames through `hostTemplate` and `advertisedHostTemplate`.
The same approach fits per-broker annotations and labels well.

## Proposal

This proposal suggests adding two new listener-level fields under `spec.kafka.listeners[].configuration`:

* `annotationsTemplate`
* `labelsTemplate`

Both fields will be maps of string keys to string values.
They define default per-broker metadata that Strimzi renders for each broker resource.

### Template variables

The template values support the same variables already used by listener host templating:

* `{nodeId}` → Kafka node ID
* `{nodePodName}` → Kubernetes pod name for the Kafka node

Only values are templated.
Keys are copied as-is.

For example:

```yaml
spec:
  kafka:
    listeners:
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
        configuration:
          annotationsTemplate:
            external-dns.alpha.kubernetes.io/hostname: kafka-{nodeId}.example.com.
            my-company.io/pod-name: "{nodePodName}"
          labelsTemplate:
            strimzi.io/node-id: "{nodeId}"
            strimzi.io/pod-name: "{nodePodName}"
```

For a node with ID `12` and pod name `my-cluster-brokers-12`, Strimzi would render:

```yaml
annotations:
  external-dns.alpha.kubernetes.io/hostname: kafka-12.example.com.
  my-company.io/pod-name: my-cluster-brokers-12
labels:
  strimzi.io/node-id: "12"
  strimzi.io/pod-name: my-cluster-brokers-12
```

### Where the templates apply

The new template fields apply only to per-broker resources.
They do not affect bootstrap resources.

They should be used everywhere Strimzi currently consumes per-broker annotations and labels from listener configuration, namely:

* per-broker `Service` resources
* per-broker `Route` resources
* per-broker `Ingress` resources

Bootstrap metadata remains configured explicitly through `configuration.bootstrap.annotations` and `configuration.bootstrap.labels`.

### Conflict handling and precedence

Users should be able to set a default pattern and still override specific brokers.
So the precedence should be:

1. render `annotationsTemplate` / `labelsTemplate`
2. overlay `configuration.brokers[].annotations` / `configuration.brokers[].labels`
3. use the merged result on the generated resource

This means broker-specific values always win over template-generated values.

For example:

```yaml
spec:
  kafka:
    listeners:
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
        configuration:
          annotationsTemplate:
            external-dns.alpha.kubernetes.io/hostname: kafka-{nodeId}.example.com.
            external-dns.alpha.kubernetes.io/ttl: "60"
          labelsTemplate:
            dns.zone: primary
            strimzi.io/node-id: "{nodeId}"
          brokers:
            - broker: 1
              annotations:
                external-dns.alpha.kubernetes.io/hostname: special-broker.example.com.
              labels:
                dns.zone: secondary
```

The resulting metadata would be:

* broker `0`

```yaml
annotations:
  external-dns.alpha.kubernetes.io/hostname: kafka-0.example.com.
  external-dns.alpha.kubernetes.io/ttl: "60"
labels:
  dns.zone: primary
  strimzi.io/node-id: "0"
```

* broker `1`

```yaml
annotations:
  external-dns.alpha.kubernetes.io/hostname: special-broker.example.com.
  external-dns.alpha.kubernetes.io/ttl: "60"
labels:
  dns.zone: secondary
  strimzi.io/node-id: "1"
```

This behavior is more flexible than using the template only when the per-broker map is completely absent.

### API and implementation details

The implementation would be additive and aligned with the existing listener templating model.

#### API model changes

In `GenericKafkaListenerConfiguration`, add:

* `Map<String, String> annotationsTemplate`
* `Map<String, String> labelsTemplate`

These fields should be documented similarly to `hostTemplate` and `advertisedHostTemplate`, including the supported placeholders.

#### Listener utility changes

`ListenersUtils` currently resolves per-broker metadata through helper methods such as `brokerAnnotations(listener, nodeId)` and `brokerLabels(listener, nodeId)`.
These helpers should be extended to:

* render the template maps for the current `NodeRef`
* merge rendered template values with explicit broker values
* keep explicit broker values as the winner on key conflicts

A practical implementation would:

* introduce a helper to render a templated string value using `{nodeId}` and `{nodePodName}`
* introduce a helper to render all values in a `Map<String, String>`
* add `brokerAnnotations(listener, node)` and `brokerLabels(listener, node)` overloads
* migrate the existing Kafka resource generation code to use the `NodeRef` overloads

#### Kafka cluster generation changes

The per-broker resource generation in `KafkaCluster` already routes all per-broker listener metadata through `ListenersUtils`.
Therefore the feature can be implemented centrally without duplicating logic in each resource builder.

The affected generated resources are:

* per-broker external services
* per-broker routes
* per-broker ingresses

#### Validation changes

The new template fields should be allowed on the same listener types where per-broker annotations and labels are already valid.
That means the validator behavior should mirror existing support for `brokers[].annotations` and `brokers[].labels`.

No additional syntax validation is needed beyond the existing simple placeholder replacement model.
Unknown placeholder text can remain unchanged.

### Examples

#### Example 1: LoadBalancer listener with annotation and label templates

```yaml
spec:
  kafka:
    listeners:
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
        configuration:
          bootstrap:
            annotations:
              external-dns.alpha.kubernetes.io/hostname: kafka-bootstrap.example.com.
          annotationsTemplate:
            external-dns.alpha.kubernetes.io/hostname: kafka-{nodeId}.example.com.
            external-dns.alpha.kubernetes.io/ttl: "60"
          labelsTemplate:
            app.kubernetes.io/component: kafka-broker-lb
            strimzi.io/node-id: "{nodeId}"
```

This removes the need to list every broker separately while keeping bootstrap configuration explicit.

#### Example 2: ClusterIP listener for Gateway API style access

```yaml
spec:
  kafka:
    listeners:
      - name: gateway
        port: 9092
        type: cluster-ip
        tls: true
        configuration:
          brokerCertChainAndKey:
            secretName: my-certificate
            certificate: tls.crt
            key: tls.key
          annotationsTemplate:
            gateway.strimzi.io/backend-pod: "{nodePodName}"
          labelsTemplate:
            gateway.strimzi.io/node-id: "{nodeId}"
```

This is useful when another networking layer consumes per-broker service metadata.

#### Example 3: Use template defaults and override one broker

```yaml
spec:
  kafka:
    listeners:
      - name: external
        port: 9094
        type: nodeport
        tls: true
        configuration:
          annotationsTemplate:
            external-dns.alpha.kubernetes.io/hostname: kafka-{nodeId}.example.com.
          labelsTemplate:
            exposure-type: standard
          brokers:
            - broker: 5
              annotations:
                external-dns.alpha.kubernetes.io/hostname: vip-broker.example.com.
              labels:
                exposure-type: premium
```

This keeps the common case short while preserving precise control where needed.

## Affected/not affected projects

Affected:

* `strimzi-kafka-operator` API model
* `strimzi-kafka-operator` Cluster Operator listener validation and resource generation
* generated CRD documentation for listener configuration
* user-facing listener documentation

Not affected:

* Topic Operator behavior
* User Operator behavior
* Kafka protocol behavior itself
* bootstrap listener metadata behavior

## Compatibility

This proposal is fully backwards compatible.

* Existing configurations using only `brokers[].annotations` and `brokers[].labels` continue to work unchanged.
* Existing bootstrap metadata configuration is unchanged.
* The change is additive and introduces no breaking API behavior.
* Users can adopt the templates gradually.

## Rejected alternatives

### Apply templates only when `brokers[].annotations` or `brokers[].labels` are absent

This was rejected because it makes partial overrides awkward.
Users often want a common default plus a targeted override for one broker.
Merging template values first and then overlaying explicit broker metadata is more useful.

### Support templating of metadata keys as well as values

This was rejected to keep the feature simple and predictable.
Templating only values matches the common use cases and avoids making resource metadata generation harder to understand.
