# Gateway API-based `type: tcproute` listener

This proposal suggests adding a new listener type (`type: tcproute`) based on the Gateway API and its `TCPRoute` resources.
It complements the `type: tlsroute` listener introduced in [SEP-136](https://github.com/strimzi/proposals/blob/main/136-tls-route-listener.md) by exposing Apache Kafka through a single shared gateway using one port per node instead of one hostname per node.

## Current situation

Strimzi currently supports five types of external listeners:

* Load balancers (`type: loadbalancer`)
* Node ports (`type: nodeport`)
* OpenShift Routes (OpenShift only) (`type: route`)
* Ingress with TLS passthrough (`type: ingress`, deprecated)
* Gateway API TLS routes (`type: tlsroute`)

Because the Kafka protocol requires clients to connect to every broker individually, each of these listener types has to expose N+1 distinct addresses for a cluster with N brokers.
They differ in _how_ the addresses are made distinct:

* `type: loadbalancer` gives each broker its own load balancer, so each address is a different IP address or hostname.
* `type: nodeport` gives each broker a different port on every Kubernetes node.
* `type: route`, `type: ingress`, and `type: tlsroute` give each broker a different hostname on a shared address and use TLS-SNI or HTTP host headers to demultiplex the traffic.

The first option is expensive: a 10-broker cluster provisions 11 cloud load balancers.
The second option requires exposing the Kubernetes nodes themselves.
The third option requires TLS on the wire, one DNS name per broker, and certificates that cover every broker hostname.

The `TCPRoute` API is the missing combination: a single shared gateway, and therefore a single cloud load balancer, where the brokers are distinguished by port rather than by hostname or by IP address.

## Motivation

`TCPRoute` resources route raw TCP connections from a gateway listener to a backend service based on protocol and port alone, with no L7 or TLS awareness.
They graduated to the _Standard_ channel and to the `v1` API version in Gateway API 1.6.0.

Adding a `type: tcproute` listener brings three benefits over the listener types Strimzi has today:

1. **One load balancer instead of N+1.**
   All bootstrap and per-broker `TCPRoute` resources attach to the same `Gateway`.
   Gateway API implementations back a `Gateway` with a single load balancer, for example one AWS NLB, that exposes one port per gateway listener.
   Scaling a cluster from 3 to 30 brokers adds 27 ports to an existing load balancer instead of provisioning 27 new ones.
   This reduces cost, avoids cloud load balancer quotas on the number of load balancers, and removes the provisioning latency that makes broker scale-up slow with `type: loadbalancer` listeners.
2. **No dependency on TLS-SNI.**
   `type: tlsroute` listeners can only carry TLS traffic, because SNI is what tells the gateway which broker a connection belongs to.
   `TCPRoute` resources have no such requirement, so a `type: tcproute` listener works with `tls: true` for TLS passthrough all the way to the broker, and with `tls: false` for example for a `SASL_PLAINTEXT` listener inside a trusted network.
3. **A single DNS name and a single certificate SAN.**
   Because the brokers are distinguished by port, all of them share one address.
   Users do not need wildcard DNS or one DNS record per broker, and the broker certificates need to cover only one name.

Users can already achieve this manually with a `type: cluster-ip` listener, self-managed `TCPRoute` resources, and hand-maintained `advertisedHost` and `advertisedPort` overrides.
That is exactly the situation described in [strimzi-kafka-operator#11123](https://github.com/strimzi/strimzi-kafka-operator/issues/11123): the routing resources have to be kept in sync with the node IDs that Strimzi assigns, and brokers that come up before their routes exist break producers and consumers.

## Relationship to the `type: tlsroute` listener

The `type: tcproute` and `type: tlsroute` listeners are complementary rather than competing, and neither replaces the other.
They make opposite trade-offs, and the deciding factors are the size of the cluster and whether TLS-SNI can be required from clients.

|  | `type: tlsroute` | `type: tcproute` |
|---|---|---|
| Brokers distinguished by | Hostname (TLS-SNI) | Port |
| Gateway listeners needed | One, shared by all brokers | One per broker plus one for the bootstrap |
| TLS | Required on the wire | Optional |
| mTLS authentication | With TLS passthrough | With `tls: true` |
| DNS records | One per broker, or a wildcard | One, shared |
| Certificate SANs | One per broker | One, shared |
| Practical cluster size | Unbounded | Bounded by the gateway and load balancer listener limits |

Because a `TLSRoute` listener is shared, `type: tlsroute` scales to any number of brokers on a single gateway listener.
Because `type: tcproute` consumes one listener per broker, it runs into the limits described below.
The documentation will recommend `type: tlsroute` as the default for large clusters, and `type: tcproute` when clients cannot use TLS-SNI, when the listener should not require TLS at all, or when a single DNS name and certificate is worth more than unbounded scale.

## Proposal

Strimzi will implement a new listener type `type: tcproute` based on `TCPRoute` resources from the `gateway.networking.k8s.io/v1` API.

The listener follows the same architecture as the `type: tlsroute`, `type: ingress`, and `type: route` listeners:

* One per-listener bootstrap `Service` pointing to all Kafka brokers
* One `Service` per broker
* One bootstrap `TCPRoute` and one `TCPRoute` per broker, each pointing to the corresponding service

The important difference is how a route is bound to the gateway.
A `TLSRoute` is matched by hostname, so all TLS routes can share one gateway listener.
A `TCPRoute` has nothing to match on, so each route needs its own gateway listener.
The Gateway API specification is explicit about this: if several `TCPRoute` resources attach to the same listener, all of them are `Accepted` but only the oldest one receives traffic.
Every route Strimzi creates must therefore attach to a distinct TCP listener, and each of those listeners occupies a distinct port on the gateway.

This raises the central design question of this proposal: who creates the N+1 gateway listeners?

### Managing gateway listeners with `ListenerSet` resources

Strimzi does not manage `Gateway` resources, and it should not start now.
Gateways are usually owned by an infrastructure team, live in a different namespace, and are shared between many applications.

The Gateway API solves this with the `ListenerSet` resource ([GEP-1713](https://gateway-api.sigs.k8s.io/geps/gep-1713/)), which reached the _Standard_ channel as `v1` in Gateway API 1.5.0.
A `ListenerSet` is a namespaced resource that contributes listeners to a `Gateway` owned by someone else.
The gateway owner opts in by setting `.spec.allowedListeners` on the `Gateway`, and from then on the application namespace can add and remove listeners on its own.
Listeners contributed through a `ListenerSet` share the parent gateway's address and infrastructure, which is precisely the "one load balancer, many ports" model this proposal is after.

This proposal suggests that Strimzi manages one `ListenerSet` per `type: tcproute` listener, containing one TCP listener entry for the bootstrap and one for each broker.
Strimzi then attaches the `TCPRoute` resources to those entries by `sectionName`.
When brokers are added or removed, the operator adds or removes both the listener entry and the route in the same reconciliation, so no manual step is needed on scale-up.

Requiring `ListenerSet` support is a real constraint, because both `TCPRoute` and `ListenerSet` are _Extended_ support features that implementations may choose not to implement.
To avoid making the listener unusable on implementations that support `TCPRoute` but not `ListenerSet`, and for users whose gateway does not allow `ListenerSet` attachment, the listener will also support a mode where Strimzi creates only the `TCPRoute` resources and the user pre-creates the gateway listeners.
This mode is controlled by the `createListenerSet` flag described below.

### Cluster size limits

One `type: tcproute` listener maps to exactly one gateway, and therefore to one load balancer.
A cluster with N brokers consumes N+1 listeners on that gateway, and both the Gateway API and the underlying infrastructure impose limits on how many listeners a gateway can have:

* A `Gateway` resource is limited to 64 listeners.
  Lifting this limit is one of the reasons `ListenerSet` exists, and it is another argument for preferring `createListenerSet: true`.
* Cloud providers apply their own limits.
  An AWS NLB supports at most 50 listeners and that quota is not adjustable, so `1 + N <= 50` and a single gateway supports at most 49 brokers.
  Any listeners the parent `Gateway` defines itself, if they have routes attached, count against the same budget.

These limits apply per listener and per gateway, not per cluster.
It is important to be clear about what does _not_ work around them.
Defining several `type: tcproute` listeners does not help, because every Strimzi listener covers every broker in the cluster, so each additional listener creates another complete set of N+1 routes rather than partitioning the existing ones.
Listing several parent references in one listener does not help either, because the references are applied to every route, which makes each broker reachable through every gateway rather than distributing the brokers between them.
For the same reason, extra parent references are of limited value for Kafka in general: `advertised.listeners` carries exactly one host and port per listener, so a broker can advertise only one of those gateways to its clients.

A cluster that outgrows these limits should use a `type: tlsroute` listener, which needs only a single gateway listener regardless of the number of brokers.
Sharding one `type: tcproute` listener across several gateways is out of scope, and is discussed below.

### Implementation support

Both `TCPRoute` and `ListenerSet` are _Extended_ support features, so support has to be verified per implementation.
At the time of writing, the AWS Load Balancer Controller supports everything this proposal needs:

* Version 3.5.0 requires Gateway API 1.6 CRDs, serves `TCPRoute` at `gateway.networking.k8s.io/v1`, and passes Gateway API 1.6.0 conformance.
* The `gateway.k8s.aws/nlb` GatewayClass provisions one NLB per `Gateway` and materialises one NLB listener for each gateway listener that has a route attached, which is exactly the model described above.
* `ListenerSet` is supported on both the NLB and ALB gateway controllers since version 3.2.0.

Two implementation-specific details are worth noting for users of that controller, and will be mentioned in the documentation rather than modelled in the Strimzi API.
Mixing protocol layers on one `Gateway` is not supported, so a gateway carrying Kafka traffic cannot also carry `HTTPRoute` resources.
Target group settings such as `targetType`, TCP health checks, and deregistration delay can be set once as a gateway-level default through `LoadBalancerConfiguration.defaultTargetGroupConfiguration`, and every broker target group inherits them.
That last point matters for the concern raised in the original issue, where the Envoy Gateway implementation needed a `BackendTrafficPolicy` resource per route: implementations differ in whether such tuning is per-route or inheritable, and Strimzi does not need to model implementation-specific policy resources in either case.

### Strimzi API

The `type: tcproute` listener reuses the `parentRefs` field introduced for `type: tlsroute` and adds three new fields.

The following YAML shows an example of the `type: tcproute` listener configuration in a `Kafka` CR:

```yaml
listeners:
  - name: external
    port: 9094
    type: tcproute
    tls: true
    authentication:
      type: tls
    configuration:
      parentRefs:
        - name: kafka-gateway
          namespace: infra
      createListenerSet: true
      bootstrap:
        host: kafka.example.com
        gatewayPort: 9192
      gatewayPortTemplate: "9200 + {nodeId}"
```

#### `parentRefs`

The `parentRefs` field is the same field, with the same schema, as the one used by `type: tlsroute` listeners.
Its meaning depends on `createListenerSet`:

* With `createListenerSet: true`, it identifies the `Gateway` that the generated `ListenerSet` attaches to.
  Because `ListenerSet` has a single `.spec.parentRef`, exactly one parent reference must be configured, it must refer to a `Gateway`, and it must not set `sectionName` or `port`.
* With `createListenerSet: false`, the references are copied into the `.spec.parentRefs` of every generated `TCPRoute`, and Strimzi sets the `port` field of each reference to the port assigned to that route.
  Configuring more than one reference is allowed for consistency with `type: tlsroute`, but as explained above it creates additional paths to the same brokers rather than distributing them.

#### `createListenerSet`

A new boolean field `createListenerSet` in the listener configuration, defaulting to `false`.

When set to `true`, Strimzi creates and manages a `ListenerSet` resource with the TCP listener entries for this listener.
When left at `false`, Strimzi creates only the `TCPRoute` resources, and the user is responsible for making sure the parent gateway has a TCP listener on every configured port that allows `TCPRoute` attachment from the Kafka namespace.

The default is `false` because it is the mode with the fewest requirements on the Gateway API implementation.
The documentation will recommend `createListenerSet: true` wherever it is supported, since it is the only mode where adding a broker does not require a change to the gateway configuration.

#### `gatewayPort` and `gatewayPortTemplate`

Two new fields configure the port that each route uses on the gateway:

* `gatewayPort` in the per-listener bootstrap configuration (`.configuration.bootstrap.gatewayPort`) and in the per-broker configuration (`.configuration.brokers[].gatewayPort`)
* `gatewayPortTemplate` in the listener configuration (`.configuration.gatewayPortTemplate`)

`gatewayPortTemplate` uses the same simple arithmetic syntax as the existing `advertisedPortTemplate` field from [SEP-135](https://github.com/strimzi/proposals/blob/main/135-templating-advertised-port-fields.md), with `{nodeId}` as the only placeholder.
For example, `9200 + {nodeId}` assigns port `9200` to node 0, port `9201` to node 1, and so on.
Reusing that syntax means the implementation can reuse the existing template rendering code, and it makes the ports deterministic: a given node ID always maps to the same port, so DNS, firewall rules, and client configuration stay stable across restarts and rescheduling.

The bootstrap `gatewayPort` is mandatory.
For the brokers, either `gatewayPortTemplate` or a `gatewayPort` for every broker must be configured, following the same pattern as `host` and `hostTemplate` for the other route-based listeners.

#### Addresses

`TCPRoute` resources contain no hostname, so the `host` fields have a slightly different meaning than for the other route-based listeners:
they do not influence the generated resources at all and are only used as the address that Strimzi advertises to clients, publishes in the `Kafka` CR status, and adds to the broker certificates.

* `.configuration.bootstrap.host` sets the bootstrap address.
* `.configuration.hostTemplate` and `.configuration.brokers[].host` set the per-broker addresses.
  When neither is configured, the brokers use the bootstrap host, since with `TCPRoute` all brokers are reached through the same gateway address.
* `advertisedHost`, `advertisedHostTemplate`, `advertisedPort`, and `advertisedPortTemplate` keep their usual meaning as overrides of what the brokers put into `advertised.listeners`.
* `.configuration.bootstrap.alternativeNames` keeps its usual meaning and can be used to add further names to the bootstrap certificate.

Unlike for the other route-based listeners, the `host` fields are optional.
When `.configuration.bootstrap.host` is not set and the parent reference points to a `Gateway`, Strimzi reads the first address from the gateway's `.status.addresses` and uses it as the address for the bootstrap and for all brokers.
This mirrors how `type: loadbalancer` listeners use the address assigned to the `Service`, and it makes the simplest possible configuration work without any DNS setup.
Most production users will want to put their own DNS name in front of the load balancer and will configure `host` explicitly.

The advertised port defaults to the gateway port assigned to the route rather than to a fixed value such as 443, because with `TCPRoute` the port is known to Strimzi.
`advertisedPort` and `advertisedPortTemplate` can still override it for setups with port translation in front of the gateway.

#### TLS and authentication

A gateway listener with `protocol: TCP` forwards the connection unchanged, so there is no TLS termination in the gateway and no equivalent of the TLS termination mode discussed for `type: tlsroute` listeners.

`type: tcproute` listeners will therefore be supported both with and without TLS encryption:

* With `tls: true`, the TLS session is established directly between the client and the Kafka broker, using the broker certificate, and mTLS authentication is available.
* With `tls: false`, the connection is unencrypted, in the same way as an unencrypted `type: loadbalancer` or `type: nodeport` listener today.

All authentication types supported by other external listeners remain available, with mTLS authentication allowed only when TLS encryption is enabled.

#### Templates

No new template fields are added.
The existing `externalBootstrapRoute` and `perPodRoute` template fields in `.spec.kafka.template`, and `perPodRoute` in `.spec.template` of the `KafkaNodePool` CR, will also apply to the generated `TCPRoute` resources, in the same way they were reused for `TLSRoute` resources.
The generated `ListenerSet` will use the `externalBootstrapRoute` template, since there is one `ListenerSet` per listener rather than one per broker.

### Generated resources

For a cluster `my-cluster` with a node pool `brokers` containing nodes 0, 1, and 2, and the configuration shown above, Strimzi generates the following resources in addition to the bootstrap and per-broker services.
This cluster uses 4 of the gateway's listeners.

The `ListenerSet` (only when `createListenerSet: true`):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: ListenerSet
metadata:
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/component-type: kafka
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  name: my-cluster-kafka-external
  namespace: myproject
  ownerReferences:
    - apiVersion: kafka.strimzi.io/v1
      blockOwnerDeletion: true
      controller: false
      kind: Kafka
      name: my-cluster
      uid: c111cc18-9056-4888-a426-c5c701b0ae90
spec:
  parentRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: kafka-gateway
    namespace: infra
  listeners:
    - name: bootstrap
      protocol: TCP
      port: 9192
      allowedRoutes:
        kinds:
          - kind: TCPRoute
    - name: broker-0
      protocol: TCP
      port: 9200
      allowedRoutes:
        kinds:
          - kind: TCPRoute
    - name: broker-1
      protocol: TCP
      port: 9201
      allowedRoutes:
        kinds:
          - kind: TCPRoute
    - name: broker-2
      protocol: TCP
      port: 9202
      allowedRoutes:
        kinds:
          - kind: TCPRoute
```

The bootstrap `TCPRoute`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: TCPRoute
metadata:
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/component-type: kafka
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  name: my-cluster-kafka-external-bootstrap
  namespace: myproject
  ownerReferences:
    - apiVersion: kafka.strimzi.io/v1
      blockOwnerDeletion: true
      controller: false
      kind: Kafka
      name: my-cluster
      uid: c111cc18-9056-4888-a426-c5c701b0ae90
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: ListenerSet
      name: my-cluster-kafka-external
      sectionName: bootstrap
  rules:
    - backendRefs:
        - name: my-cluster-kafka-external-bootstrap
          port: 9094
```

And one `TCPRoute` per broker:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: TCPRoute
metadata:
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/component-type: kafka
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
    strimzi.io/pool-name: brokers
  name: my-cluster-brokers-0
  namespace: myproject
  ownerReferences:
    - apiVersion: kafka.strimzi.io/v1
      blockOwnerDeletion: true
      controller: false
      kind: KafkaNodePool
      name: brokers
      uid: c111cc18-9056-4888-a426-c5c701b0ae90
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: ListenerSet
      name: my-cluster-kafka-external
      sectionName: broker-0
  rules:
    - backendRefs:
        - name: my-cluster-brokers-0
          port: 9094
```

With `createListenerSet: false`, no `ListenerSet` is created and the routes attach directly to the gateway by port:

```yaml
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: kafka-gateway
      namespace: infra
      port: 9200
```

The naming of all generated resources follows the same rules as for the existing route and ingress based listeners.
The `ListenerSet` listener entries are named `bootstrap` and `broker-<nodeId>`, which is unique within the `ListenerSet`.

With this configuration, the brokers advertise `kafka.example.com:9200`, `kafka.example.com:9201`, and `kafka.example.com:9202`, and the `Kafka` CR status reports the bootstrap address `kafka.example.com:9192`.

### Readiness

After creating the resources, Strimzi will wait for the `.status` section of every generated `TCPRoute` to contain at least one parent reference, exactly as it does for `TLSRoute` resources.
This indicates that the route was accepted.
Strimzi will not evaluate individual conditions and will not try to detect warnings, errors, or failures, because implementations report them differently.
If a route has no parent references after the Cluster Operator _reconciliation timeout_, the reconciliation fails with a corresponding error.

Waiting on the route status is sufficient to cover the `ListenerSet` as well: if the `ListenerSet` is rejected by the gateway, if one of its listener entries conflicts with a port already used by another listener, or if the gateway has run out of listener capacity, the routes attached to it will not be accepted either.

When the bootstrap address is discovered from the gateway instead of being configured, Strimzi also waits for the `Gateway` to report an address in its status.

### Validation

The listener validation will be extended to check that:

* `parentRefs` is configured, and with `createListenerSet: true` that it contains exactly one reference to a `Gateway` without `sectionName` or `port`
* `.configuration.bootstrap.gatewayPort` is configured
* Either `gatewayPortTemplate` is configured, or every broker has a `gatewayPort`
* All gateway ports within one listener are unique and within the valid port range
* `gatewayPort`, `gatewayPortTemplate`, and `createListenerSet` are used only with `type: tcproute` listeners

Strimzi cannot validate that the configured ports are free on the gateway, or that the gateway has capacity for them, because the gateway may be shared with other applications and other Kafka clusters.
Conflicts and exhausted capacity surface as routes that are never accepted, and therefore as a failed reconciliation after the reconciliation timeout.

### Prerequisites and dependencies

Using the listener requires:

* Gateway API 1.6.0 or newer, for the `v1` version of the `TCPRoute` API, and an implementation that supports `TCPRoute`
* For `createListenerSet: true`, Gateway API 1.5.0 or newer, an implementation that supports `ListenerSet`, and a `Gateway` with `.spec.allowedListeners` configured to accept `ListenerSet` resources from the Kafka namespace
* For `createListenerSet: false`, a `Gateway` with a TCP listener for every configured port, each allowing `TCPRoute` attachment from the Kafka namespace

Note that in the `ListenerSet` mode only the `ListenerSet` crosses the namespace boundary, governed by `allowedListeners` on the `Gateway`.
The `TCPRoute` resources and the services they point to all live in the Kafka namespace, so no `ReferenceGrant` is needed.

Like the `type: tlsroute` listener, the operator will detect whether the `TCPRoute` and `ListenerSet` APIs are available in the cluster through `PlatformFeaturesAvailability` and fail the reconciliation with a clear error when a `type: tcproute` listener is configured on a cluster without them.

The Cluster Operator `ClusterRole` will be extended with the `tcproutes` and `listenersets` resources in the `gateway.networking.k8s.io` API group, with the full set of verbs, and with read-only access to `gateways` for address discovery.

The implementation also needs a Java model for the `v1` version of the `TCPRoute` API.
Fabric8 generates its Gateway API model per kind and per API version from a specific Gateway API release, so the `v1` `TLSRoute` support added in Fabric8 7.7.0 does not carry over.
Fabric8 7.8.0 was released one day before Gateway API 1.6.0, and still pins `sigs.k8s.io/gateway-api` at 1.5.1, where `TCPRoute` exists only as the now-deprecated `v1alpha2`.
This is being addressed upstream in [fabric8io/kubernetes-client#8032](https://github.com/fabric8io/kubernetes-client/issues/8032) and [fabric8io/kubernetes-client#8033](https://github.com/fabric8io/kubernetes-client/pull/8033), which bump the pin to 1.6.1 and regenerate the model, adding `v1.TCPRoute` and `v1.UDPRoute` while keeping the `v1alpha2` types.
Once that is released, the implementation needs only a Fabric8 version bump, in the same way the `type: tlsroute` implementation followed the bump to 7.7.0.
Building on `v1alpha2` instead is rejected, because that version was deprecated in Gateway API 1.6 and will be removed.
If the Fabric8 release lags, the fallback is for Strimzi to carry the four `TCPRoute` model classes itself, reusing the existing Fabric8 `v1.ParentReference` and `v1.BackendRef` types, since the `v1alpha2` and `v1` schemas are identical.

### Testing strategy

The listener will be covered by unit tests and manual testing.
System tests are not covered by this proposal, for the same reasons given in SEP-136: several existing listener types rely on unit and manual testing only, and the value of system tests is limited when the behaviour depends on which Gateway API implementation is installed.

## Out of scope

### Managing `Gateway` resources

Strimzi will not create, modify, or delete `Gateway` resources.
Users bring their own gateway and reference it from the listener configuration.

### Sharding one listener across multiple gateways

As described in the limits section, a `type: tcproute` listener is bounded by the number of listeners its gateway supports.
Spreading the brokers of one listener across several gateways, so that a cluster can exceed that limit, is out of scope.

The natural unit for such sharding in Strimzi would be the node pool, with different pools attaching to different gateways, and the bootstrap living on any one of them.
Kafka does not require the bootstrap and the brokers to share an address, so this would work from a protocol point of view.
It would, however, mean moving part of the listener configuration into the `KafkaNodePool` CR, which is a larger API change than this proposal wants to make, and it should be evaluated on its own merits in a future proposal.
Until then, clusters that outgrow a single gateway should use a `type: tlsroute` listener.

### `UDPRoute` and east-west traffic

`UDPRoute` resources graduated to the _Standard_ channel together with `TCPRoute`, but they are not useful for the Kafka protocol and are out of scope.
Any use of the Gateway API for routing internal Kubernetes traffic or for service mesh integration is out of scope as well.

### Automatic port allocation

Strimzi will not pick free ports on the gateway automatically.
The operator has no reliable way to know which ports are already taken by other applications sharing the gateway, and unpredictable ports would break firewall rules and any client configuration that pins the broker addresses.
Ports are always derived from the user's configuration.

### Migration between listener types

Migrating an existing `type: loadbalancer`, `type: ingress`, or `type: tlsroute` listener to `type: tcproute` is out of scope, as it depends on the user's infrastructure and DNS setup.
Users can always add a new `type: tcproute` listener, reconfigure their clients, and then remove the old listener.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator, together with the installation files, the Helm chart, and the documentation.
No other Strimzi project is affected.

## Compatibility

This proposal is fully backwards compatible.
It adds a new optional listener type and new optional listener configuration fields.
Existing listeners and existing custom resources are not affected in any way.

The new fields are additive to the `Kafka` CRD, and the new listener type is an additional value of an existing enum, so no CRD API version change is required.

## Rejected alternatives

### Adding a TCP mode to the `type: tlsroute` listener

Rather than a new listener type, `type: tlsroute` could get a flag that switches it to `TCPRoute` resources.
This was rejected because the two listeners have different required configuration, ports instead of hostnames, different validation rules, different resources, and different constraints on TLS and on cluster size.
Separate listener types is also the established pattern in Strimzi and keeps the configuration and the documentation understandable.

### Strimzi adding listeners to the `Gateway` resource

Instead of creating a `ListenerSet`, Strimzi could patch the listener list of the `Gateway` itself.
This was rejected because it would mean writing to a resource that is usually owned by a different team in a different namespace, it would make the operator a co-owner of a shared resource it does not otherwise manage, and it would risk fighting with whatever else manages that gateway.
`ListenerSet` was designed for exactly this delegation, so there is no reason to invent an alternative.

### Requiring users to pre-create all gateway listeners

The listener could support only the mode where the user maintains the gateway listeners.
This was rejected as the sole option because it reproduces the problem this feature is meant to solve: every scale-up of a node pool would require a matching manual change to the gateway before the new broker becomes reachable, which is the race condition reported in the original issue.
It is kept as the non-default mode for implementations without `ListenerSet` support.

### Sharing a single gateway listener between all routes

Attaching the bootstrap and all broker routes to one gateway port would be the simplest configuration, and would remove the listener limits entirely, but it does not work.
A TCP listener has no hostname, SNI, or path to distinguish connections, so the Gateway API specification accepts all attached routes but sends traffic only to the oldest one.
Distinguishing brokers on a shared port is exactly what `type: tlsroute` does, using SNI.

### Distributing brokers across a list of gateways automatically

Strimzi could accept several gateways and assign brokers to them, so that a cluster could exceed a single gateway's listener limit without any user involvement.
This was rejected because the assignment has to remain stable for the lifetime of each node, or client-visible addresses move underneath running clients, and Strimzi's node IDs are sparse across node pools.
Neither an index-based nor a capacity-based mapping stays both stable and balanced as pools are added, scaled, and removed, and Strimzi has no way to discover how much listener capacity a gateway has left.

### Per-broker ports without a template

Ports could be configured only through `.configuration.brokers[].gatewayPort`.
This was rejected as the only option because it is verbose and, more importantly, it breaks on scale-out: a broker added without a matching entry would have no port.
The template makes the port a pure function of the node ID, so new brokers are handled automatically.
Explicit per-broker ports are still supported for users who need a specific mapping.

### Deriving the gateway port from `advertisedPort`

The gateway port and the advertised port are almost always the same value, so `advertisedPortTemplate` could be reused for both.
This was rejected because it conflates two different things: what the gateway listens on, and what the brokers tell clients to connect to.
Keeping them separate preserves the ability to override the advertised address for setups with port translation or an additional proxy in front of the gateway, which is how `advertisedPort` already behaves for every other listener type.
