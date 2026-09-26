# Gateway API-based `type: tcproute` listener

This proposal suggests adding a new listener type (`type: tcproute`) based on the Gateway API and its `TCPRoute` resources.
It is an alternative to the `type: tlsroute` listener introduced in [SEP-136](https://github.com/strimzi/proposals/blob/main/136-tls-route-listener.md): both expose Apache Kafka through a single shared gateway, but `type: tcproute` distinguishes brokers by port rather than by hostname.

## Current situation

Strimzi currently supports five types of external listeners:

- Load balancers (`type: loadbalancer`)
- Node ports (`type: nodeport`)
- OpenShift Routes (OpenShift only) (`type: route`)
- Ingress with TLS passthrough (`type: ingress`, deprecated)
- Gateway API TLS routes (`type: tlsroute`)

Because the Kafka protocol requires clients to connect to every broker individually, each of these listener types has to expose N+1 distinct addresses for a cluster with N brokers.
They differ in _how_ the addresses are made distinct:

- `type: loadbalancer` gives each broker its own load balancer, so each address is a different IP address or hostname.
- `type: nodeport` gives each broker a different port on every Kubernetes node.
- `type: route`, `type: ingress`, and `type: tlsroute` give each broker a different hostname on a shared address and use TLS-SNI to demultiplex the traffic.

The first option is expensive: a 10-broker cluster provisions 11 cloud load balancers.
The second option requires exposing the Kubernetes nodes themselves.
The third option requires one resolvable DNS name per broker, and a certificate that covers every broker hostname.
Scaling a node pool then means the new DNS records must exist before the new broker is reachable, either through wildcard DNS or through another moving part such as external-dns and its propagation delay.

The `TCPRoute` API is the missing combination: a single shared gateway, and therefore a single cloud load balancer, where the brokers are distinguished by port rather than by hostname or by IP address.
Scaling adds a port on an address that already exists.
It does not add a DNS name, and the certificate already covers that name.

## Motivation

The operational problem that motivates this listener is the same class of problem as [strimzi-kafka-operator#11123](https://github.com/strimzi/strimzi-kafka-operator/issues/11123): a newly added broker is unreachable until configuration that Strimzi does not create is already in place.

With `type: tlsroute`, that configuration is a DNS record for the new broker's hostname, and a certificate that includes it as a SAN.
Every broker needs its own resolvable hostname.
With `type: tcproute`, brokers share one address and are distinguished by port.
A node pool can grow without new DNS records, without wildcard DNS, and without waiting for an external DNS controller.

The other properties follow from routing on port rather than on hostname:

- The cluster uses one gateway, and therefore one cloud load balancer, instead of the N+1 load balancers of `type: loadbalancer`.
  Gateway API implementations back a `Gateway` with a single load balancer that exposes one port per gateway listener.
  Scaling from 3 to 30 brokers adds 27 ports to an existing load balancer instead of provisioning 27 new ones.
- TLS is orthogonal to the routing.
  A gateway listener with `protocol: TCP` forwards the connection untouched, so `tls: true` still gives end-to-end TLS terminated at the broker, and mTLS is available.
  `tls: false` works as well, in the same way it already does for `type: loadbalancer` and `type: nodeport`, but it is not the reason to add the listener.

Users can already achieve this manually with a `type: cluster-ip` listener, self-managed `TCPRoute` resources, and hand-maintained `advertisedHost` and `advertisedPort` overrides. However, the routing resources have to be kept in sync with the node IDs that Strimzi assigns, and brokers that come up before their routes exist break producers and consumers.

This proposal automates the N+1 `TCPRoute` resources and the advertised addresses.
It does not provision the ports on the gateway.
Users create the gateway listeners.
One way to keep scale-up from racing with that step is to pre-provision a range of listeners with headroom, as in the example below.

`TCPRoute` resources graduated to the _Standard_ channel and to the `v1` API version in Gateway API 1.6.0.

## Choosing between `type: tcproute` and `type: tlsroute`

The `type: tcproute` and `type: tlsroute` listeners compete in the same way that `type: loadbalancer` and `type: nodeport` already do.
A given client-facing listener uses one of them, not both in parallel for the same purpose.
They make opposite trade-offs.

|                          | `type: tlsroute`               | `type: tcproute`                                         |
| ------------------------ | ------------------------------ | -------------------------------------------------------- |
| Brokers distinguished by | Hostname (TLS-SNI)             | Port                                                     |
| Gateway listeners needed | One, shared by all brokers     | One per broker plus one for the bootstrap                |
| DNS records              | One per broker, or a wildcard  | One, shared                                              |
| Certificate SANs         | One per broker, or a wildcard  | One, shared                                              |
| Scale-up requires        | New DNS names, or wildcard DNS | A gateway listener that already exists on the new port   |
| TLS                      | Required on the wire (SNI)     | Orthogonal; `tls: true` is passthrough to the broker     |
| mTLS authentication      | With TLS passthrough           | With `tls: true`                                         |
| Practical cluster size   | Unbounded                      | Bounded by the gateway and load balancer listener limits |

Because a `TLSRoute` listener is shared, `type: tlsroute` scales to any number of brokers on a single gateway listener.
Because `type: tcproute` consumes one listener per broker, it runs into the limits described below.

The documentation will treat this as a choice between listener types, not as a pair that should be combined.
`type: tlsroute` is the better fit when unbounded scale on one gateway listener matters more than orchestrating the creation of per-broker DNS records and certificates.
`type: tcproute` is the better fit when one load balancer for the whole cluster is preferable to one per broker, and when brokers can share a hostname so adding a broker does not require a new DNS record or certificate SAN.

## Proposal

Strimzi will implement a new listener type `type: tcproute` based on `TCPRoute` resources from the `gateway.networking.k8s.io/v1` API.

The listener follows the same architecture as the `type: tlsroute`, `type: ingress`, and `type: route` listeners:

- One per-listener bootstrap `Service` pointing to all Kafka brokers
- One `Service` per broker
- One bootstrap `TCPRoute` and one `TCPRoute` per broker, each pointing to the corresponding service

The important difference is how a route is bound to the gateway.
A `TLSRoute` is matched by hostname, so all TLS routes can share one gateway listener.
A `TCPRoute` has nothing to match on, so each route needs its own gateway listener.
The Gateway API specification is explicit about this: if several `TCPRoute` resources attach to the same listener, all of them are `Accepted` but only the oldest one receives traffic.
If a `TCPRoute` sets neither `sectionName` nor `port` on its parent reference, it attaches to every TCP listener on the gateway.
Every route Strimzi creates must therefore attach to a distinct TCP listener by port, and each of those listeners occupies a distinct port on the gateway.
If no TCP listener on the Gateway or ListenerSet uses that port, the route is not accepted.
Reconciliation then fails after the Cluster Operator reconciliation timeout.

### User-managed gateway listeners

Strimzi does not manage `Gateway` resources, and it will not manage the TCP listeners on them either.
Gateways are usually owned by an infrastructure team, live in a different namespace, and are shared between many applications.
Strimzi cannot know which ports are free on a shared gateway, or how much listener capacity is left.
Users may also prefer to define the listeners on the `Gateway` itself, or to contribute them through a `ListenerSet` they manage, and the operator should not pick that for them.

The user is therefore responsible for making sure the parent gateway has a TCP listener on every port the routes attach to, allowing `TCPRoute` attachment from the Kafka namespace.
Strimzi creates only the `TCPRoute` resources and the advertised addresses.
When a broker is added whose port has no gateway listener yet, that broker is not reachable until the listener exists, which is the same class of race as in the original issue.

The following example pre-provisions a range of gateway listeners with headroom, and lets Strimzi attach and detach routes as brokers come and go:

1. Choose a bootstrap port and a per-broker port formula that covers the node IDs the cluster will use, for example `9199` for the bootstrap and `9200 + {nodeId}` for node IDs `0` through `49`.
2. Create those TCP listeners on the `Gateway`, or on a `ListenerSet` that attaches to it, including spare ports for the next scale-up.
3. Configure `.configuration.bootstrap.networkPort` and `networkPortTemplate` to match those ports.
4. Scale brokers within the pre-provisioned range.
   Strimzi creates and deletes `TCPRoute` resources; the gateway listeners stay in place.
5. To grow beyond the range, add the new gateway listeners first, then scale.

This keeps port provisioning under the user's control, who is expected to know which ports are free, and still avoids a manual gateway change on every broker add.

Users who want to contribute listeners to a shared `Gateway` without editing the `Gateway` itself can use a `ListenerSet` ([GEP-1713](https://gateway-api.sigs.k8s.io/geps/gep-1713/)).
That resource is namespaced, reached the _Standard_ channel as `v1` in Gateway API 1.5.0, and is how an application namespace adds listeners to a gateway that someone else owns.
This proposal does not have Strimzi create or manage `ListenerSet` resources.
`parentRefs` can still point at a user-managed `ListenerSet` in the same way it can point at a `Gateway`.

### Cluster size limits

One `type: tcproute` listener maps to exactly one gateway, and therefore to one load balancer.
A cluster with N brokers consumes N+1 listeners on that gateway, and both the Gateway API and the underlying infrastructure impose limits on how many listeners a gateway can have:

- A `Gateway` resource is limited to 64 listeners.
  Users who need more can contribute extra listeners through a `ListenerSet` they manage.
- Cloud load balancers often apply their own listener quotas as well.
  Any listeners the parent `Gateway` already defines, if they have routes attached, count against the same budget.

These limits apply per listener and per gateway, not per cluster.
It is important to be clear about what does _not_ work around them.
Defining several `type: tcproute` listeners does not help, because every Strimzi listener covers every broker in the cluster, so each additional listener creates another complete set of N+1 routes rather than partitioning the existing ones.
Listing several parent references in one listener does not help either, because the references are applied to every route, which makes each broker reachable through every gateway rather than distributing the brokers between them.
For the same reason, extra parent references are of limited value for Kafka in general: `advertised.listeners` carries exactly one host and port per listener, so a broker can advertise only one of those gateways to its clients.

A cluster that outgrows these limits should use a `type: tlsroute` listener, which needs only a single gateway listener regardless of the number of brokers.
Sharding one `type: tcproute` listener across several gateways is out of scope, and is discussed below.

Because Strimzi does not manage the gateway listeners, it also cannot keep these limits from being hit.
Conflicts and exhausted capacity surface as routes that are never accepted, and therefore as a failed reconciliation after the reconciliation timeout.

### Implementation support

`TCPRoute` is an _Extended_ support feature of the Gateway API.
The Gateway implementation must support `TCPRoute`.
Strimzi will not model implementation-specific policy resources.

### Strimzi API

The `type: tcproute` listener reuses the `parentRefs` and advertised-address fields that already exist.
It adds a `networkPort` field, at bootstrap and per broker, together with a `networkPortTemplate`, and `advertisedHost` and `advertisedPort` fields on the bootstrap configuration.

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
      bootstrap:
        networkPort: 9199
        advertisedHost: kafka.example.com
      networkPortTemplate: "9200 + {nodeId}"
      advertisedHostTemplate: kafka.example.com
```

The matching gateway listeners are user-managed.
A `Gateway` that covers this example, with headroom for a few more brokers, looks like this:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kafka-gateway
  namespace: infra
spec:
  gatewayClassName: my-gateway-class
  listeners:
    - name: kafka-bootstrap
      protocol: TCP
      port: 9199
      allowedRoutes:
        kinds:
          - kind: TCPRoute
        namespaces:
          from: Selector
          selector:
            matchLabels:
              kubernetes.io/metadata.name: myproject
    - name: kafka-broker-0
      protocol: TCP
      port: 9200
      allowedRoutes:
        kinds:
          - kind: TCPRoute
        namespaces:
          from: Selector
          selector:
            matchLabels:
              kubernetes.io/metadata.name: myproject
    # ... one TCP listener per pre-provisioned broker port ...
    - name: kafka-broker-9
      protocol: TCP
      port: 9209
      allowedRoutes:
        kinds:
          - kind: TCPRoute
        namespaces:
          from: Selector
          selector:
            matchLabels:
              kubernetes.io/metadata.name: myproject
```

#### `parentRefs`

The `parentRefs` field is the same field, with the same schema, as the one used by `type: tlsroute` listeners.
The references are copied into the `.spec.parentRefs` of every generated `TCPRoute`.
Strimzi sets the `port` field of each reference to the network port of that route, so the route attaches to the matching TCP listener and not to every TCP listener on the gateway.
Any `port` value in the configured parent references is overwritten.
`sectionName` should be omitted, because a single section name cannot select N+1 different listeners.

Configuring more than one parent reference is allowed for consistency with `type: tlsroute`, but as explained above it creates additional paths to the same brokers rather than distributing them.

#### Network ports

A `TCPRoute` attaches to a gateway listener by port, so every route needs a port to attach to.
That port is a property of the gateway, not of the address a broker publishes to its clients, so it gets its own fields:

- `.configuration.bootstrap.networkPort` is required and sets the port of the gateway listener the bootstrap `TCPRoute` attaches to.
- `.configuration.networkPortTemplate` and `.configuration.brokers[].networkPort` set the same thing per broker, using the template syntax of [SEP-135](https://github.com/strimzi/proposals/blob/main/135-templating-advertised-port-fields.md).
  Either the template, or a `networkPort` for every broker, must be configured.

Strimzi copies these values into the `port` field of the parent references of the corresponding route.
They are unrelated to the listener `port` in `.spec.kafka.listeners`, which stays what it is for every other listener type: the port the brokers listen on, and therefore the port in the `backendRefs` of the generated routes.

The name is deliberately generic.
`gatewayListenerPort` would tie the field to the Gateway API, in the same way `nodePort` is tied to node ports and cannot be reused by any other listener type.

#### Addresses

`TCPRoute` resources contain no hostname, so the `host` and `hostTemplate` fields used by `type: route`, `type: ingress`, and `type: tlsroute` are not used.
Those fields configure the hostname in the generated route, and there is no such hostname here.
The address a client uses is the gateway's own address, which Strimzi does not know, so it has to be configured:

- `.configuration.bootstrap.advertisedHost` is required and sets the host of the bootstrap address, which is reported in the `Kafka` CR status and added to the broker certificates.
  This is a new field on the bootstrap configuration, named after the existing `.configuration.brokers[].advertisedHost`, and `.configuration.bootstrap.advertisedPort` is added next to it for the same reason.
- `.configuration.advertisedHostTemplate` and `.configuration.brokers[].advertisedHost` are required as well, either as the template or as a per-broker value for every broker.
  They are not defaulted from the bootstrap configuration, because the two addresses answer different questions and one should not silently stand in for the other.
  They will usually be set to the same host, since all brokers are reached through the same gateway.
- `.configuration.advertisedPortTemplate` and `.configuration.brokers[].advertisedPort` default to the network port of that broker, and `.configuration.bootstrap.advertisedPort` defaults to `.configuration.bootstrap.networkPort`.
  Configuring them explicitly is only useful when something in front of the load balancer translates ports.
- `.configuration.bootstrap.alternativeNames` adds any further names to the bootstrap certificate, as it does for the other listener types.

Certificates carry names, not ports, so a cluster whose brokers share a host needs that one name in the SANs however many brokers it has.

#### TLS and authentication

A gateway listener with `protocol: TCP` forwards the connection unchanged, so there is no TLS termination in the gateway and no equivalent of the TLS termination mode discussed for `type: tlsroute` listeners.

`type: tcproute` listeners will therefore be supported both with and without TLS encryption:

- With `tls: true`, the TLS session is established directly between the client and the Kafka broker, using the broker certificate, and mTLS authentication is available.
- With `tls: false`, the connection is unencrypted, in the same way as an unencrypted `type: loadbalancer` or `type: nodeport` listener today.

All authentication types supported by other external listeners remain available, with mTLS authentication allowed only when TLS encryption is enabled.

#### Templates

No new template fields are added.
The existing `externalBootstrapRoute` and `perPodRoute` template fields in `.spec.kafka.template`, and `perPodRoute` in `.spec.template` of the `KafkaNodePool` CR, will also apply to the generated `TCPRoute` resources, in the same way they were reused for `TLSRoute` resources.

### Generated resources

For a cluster `my-cluster` with a node pool `brokers` containing nodes 0, 1, and 2, and the configuration shown above, Strimzi generates the following resources in addition to the bootstrap and per-broker services.
This cluster uses 4 of the gateway's listeners.

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
      kind: Gateway
      name: kafka-gateway
      namespace: infra
      port: 9199
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
      kind: Gateway
      name: kafka-gateway
      namespace: infra
      port: 9200
  rules:
    - backendRefs:
        - name: my-cluster-brokers-0
          port: 9094
```

The naming of all generated resources follows the same rules as for the existing route and ingress based listeners.

With this configuration, the brokers advertise `kafka.example.com:9200`, `kafka.example.com:9201`, and `kafka.example.com:9202`, and the `Kafka` CR status reports the bootstrap address `kafka.example.com:9199`.
The advertised ports are not configured in the example, so they follow the network ports.

### Readiness

After creating the resources, Strimzi will wait for the `.status` section of every generated `TCPRoute` to contain at least one parent reference, exactly as it does for `TLSRoute` resources.
This indicates that the route was accepted.
Strimzi will not evaluate individual conditions and will not try to detect warnings, errors, or failures, because implementations report them differently.
If a route has no parent references after the Cluster Operator _reconciliation timeout_, the reconciliation fails with a corresponding error.

If a gateway listener is missing, if a port is already used by another listener, or if the gateway has run out of listener capacity, the routes attached to that port will not be accepted either.

### Validation

The listener validation will be extended to check that:

- `parentRefs` is configured
- `.configuration.bootstrap.networkPort` and `.configuration.bootstrap.advertisedHost` are configured
- Either `networkPortTemplate` is configured, or every broker has a `networkPort`
- Either `advertisedHostTemplate` is configured, or every broker has an `advertisedHost`
- All network ports within one listener are unique, including the bootstrap port, and within the valid port range
- `host` and `hostTemplate` are not configured, because they do not apply to `TCPRoute` resources

Strimzi cannot validate that the configured ports are free on the gateway, or that the gateway has capacity for them, because the gateway may be shared with other applications and other Kafka clusters.
Conflicts and exhausted capacity surface as routes that are never accepted, and therefore as a failed reconciliation after the reconciliation timeout.

### Prerequisites and dependencies

Using the listener requires:

- Gateway API 1.6.0 or newer, for the `v1` version of the `TCPRoute` API, and an implementation that supports `TCPRoute`
- A `Gateway` or user-managed `ListenerSet` with a TCP listener for every network port, each allowing `TCPRoute` attachment from the Kafka namespace

The `TCPRoute` resources and the services they point to all live in the Kafka namespace, so no `ReferenceGrant` is needed.
Attachment to a `Gateway` in another namespace is governed by `.spec.listeners[].allowedRoutes` on that gateway.
If the user manages a `ListenerSet`, attachment of that `ListenerSet` to the `Gateway` is governed by `.spec.allowedListeners` on the gateway.

Like the `type: tlsroute` listener, the operator will detect whether the `TCPRoute` API is available in the cluster through `PlatformFeaturesAvailability` and fail the reconciliation with a clear error when a `type: tcproute` listener is configured on a cluster without it.

The Cluster Operator `ClusterRole` will be extended with the `tcproutes` resource in the `gateway.networking.k8s.io` API group, with the full set of verbs.
The operator does not read or write `Gateway` or `ListenerSet` resources.

The implementation also needs a Java model for the `v1` version of the `TCPRoute` API.
Fabric8 7.9.0, which Strimzi main already uses, bundles Gateway API 1.6.1 and includes `v1.TCPRoute`.

### Testing strategy

The listener will be covered by unit tests.
System tests are not covered by this proposal, for the same reasons given in SEP-136: the value of system tests is limited when the behaviour depends on which Gateway API implementation is installed.

## Out of scope

### Managing `Gateway` resources

Strimzi will not create, modify, or delete `Gateway` resources.
Users bring their own gateway and reference it from the listener configuration.

### Managing `ListenerSet` resources

Strimzi will not create or manage `ListenerSet` resources.
Users who want to contribute listeners to a shared `Gateway` without editing the `Gateway` itself can create a `ListenerSet` and point `parentRefs` at it.
Operator-managed `ListenerSet` resources could be added later as an additive change if there is demand.

### Sharding one listener across multiple gateways

As described in the limits section, a `type: tcproute` listener is bounded by the number of listeners its gateway supports.
Spreading the brokers of one listener across several gateways, so that a cluster can exceed that limit, is out of scope.

The natural unit for such sharding in Strimzi would be the node pool, with different pools attaching to different gateways, and the bootstrap living on any one of them.
Kafka does not require the bootstrap and the brokers to share an address, so this would work from a protocol point of view.
It would, however, mean moving part of the listener configuration into the `KafkaNodePool` CR, which is a larger API change than this proposal wants to make, and it should be evaluated on its own merits in a future proposal.
Until then, clusters that outgrow a single gateway should use a `type: tlsroute` listener.
Users can still spread brokers across gateways themselves, with a `type: cluster-ip` listener and their own `TCPRoute` resources.

### `UDPRoute` and east-west traffic

`UDPRoute` resources graduated to the _Standard_ channel together with `TCPRoute`, but they are not useful for the Kafka protocol and are out of scope.
Any use of the Gateway API for routing internal Kubernetes traffic or for service mesh integration is out of scope as well.

### Automatic port allocation

Strimzi will not pick free ports on the gateway automatically.
The operator has no reliable way to know which ports are already taken by other applications sharing the gateway, and unpredictable ports would break firewall rules and any client configuration that pins the broker addresses.
Ports are always taken from the advertised-port configuration.

### Migration between listener types

Migrating an existing `type: loadbalancer`, `type: ingress`, or `type: tlsroute` listener to `type: tcproute` is out of scope, as it depends on the user's infrastructure and DNS setup.
Users can always add a new `type: tcproute` listener, reconfigure their clients, and then remove the old listener.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator, together with the installation files, the Helm chart, and the documentation.
No other Strimzi project is affected.

## Compatibility

This proposal is fully backwards compatible.
It adds a new optional listener type.
Existing listeners and existing custom resources are not affected in any way.

The new listener type is an additional value of an existing enum, so no CRD API version change is required.

## Rejected alternatives

### Adding a TCP mode to the `type: tlsroute` listener

Rather than a new listener type, `type: tlsroute` could get a flag that switches it to `TCPRoute` resources.
This was rejected because the two listeners have different required configuration, ports instead of hostnames, different validation rules, different resources, and different constraints on cluster size.
Separate listener types is also the established pattern in Strimzi and keeps the configuration and the documentation understandable.

### Strimzi adding listeners to the `Gateway` resource

Strimzi could patch the listener list of the `Gateway` itself.
This was rejected because it would mean writing to a resource that is usually owned by a different team in a different namespace, it would make the operator a co-owner of a shared resource it does not otherwise manage, and it would risk fighting with whatever else manages that gateway.

### Reusing the advertised ports as the gateway ports

The gateway port and the advertised port are almost always the same value, so the advertised-port fields could have carried both meanings, with the listener `port` acting as the bootstrap gateway port.
This was rejected because the two are different things: one selects a gateway listener, the other is what a broker tells its clients.
Overloading the advertised ports would rule out any port translation in front of the load balancer, and it would give the listener `port` a second meaning it does not have for any other listener type.
Separate `networkPort` fields cost API surface but keep each field with one job, and the advertised ports default to them, so the common case configures the ports once.

### Naming the new field `gatewayListenerPort`

The field could be named after the thing it selects, a listener on a `Gateway`.
This was rejected because it would tie a generic concept to one implementation, which is the same mistake as `nodePort`: that field cannot be reused by any listener type that is not based on node ports, even though the underlying idea is the same.
`networkPort` describes the role without naming the mechanism.

### Using `host` and `hostTemplate` for the advertised address

The other route-based listeners use `host` and `hostTemplate` for the hostname that lands in the generated route, and then `advertisedHost` as an override of what the brokers advertise.
`TCPRoute` has no hostname, so reusing `host` would give that field a different meaning than everywhere else.
The advertised-address fields are the ones that match the job: Kafka needs a name to advertise and a SAN on the certificate, and the Gateway API does not.

### Defaulting the broker addresses to the bootstrap address

With `TCPRoute` all brokers are reached through the same gateway, so the per-broker advertised host could default to the bootstrap host.
This was rejected because it makes one required field silently supply another, and a misconfigured bootstrap address would then produce brokers that advertise the wrong host instead of a validation error.
Both are required and validated.

### Constructing the bootstrap address from the broker addresses

Kafka clients accept a list of brokers as their bootstrap servers, so the `Kafka` CR status could report the broker addresses instead of a separate bootstrap address, and the listener would need no bootstrap host and no bootstrap route.
This was rejected because it changes what the status means for this one listener type, and it removes the single stable address that clients are configured with: every client configuration would have to be revisited whenever the set of brokers changes.
A bootstrap `TCPRoute` on its own gateway listener costs one port and keeps the listener consistent with every other type.

### Discovering the advertised address from the `Gateway` status

When `bootstrap.advertisedHost` was omitted, Strimzi could read the first address from the parent `Gateway`'s `.status.addresses`.
This was rejected because Strimzi does not otherwise touch `Gateway` resources, it would require read permission on them in the Cluster Operator `ClusterRole`, and reading gateway status is a common source of compatibility issues across implementations.
The advertised hostname is always taken from the listener configuration.

### Sharing a single gateway listener between all routes

Attaching the bootstrap and all broker routes to one gateway port would be the simplest configuration, and would remove the listener limits entirely, but it does not work.
A TCP listener has no hostname, SNI, or path to distinguish connections, so the Gateway API specification accepts all attached routes but sends traffic only to the oldest one.
Distinguishing brokers on a shared port is exactly what `type: tlsroute` does, using SNI.

### Distributing brokers across a list of gateways automatically

Strimzi could accept several gateways and assign brokers to them, so that a cluster could exceed a single gateway's listener limit without any user involvement.
This was rejected because the assignment has to remain stable for the lifetime of each node, or client-visible addresses move underneath running clients, and Strimzi's node IDs are sparse across node pools.
Neither an index-based nor a capacity-based mapping stays both stable and balanced as pools are added, scaled, and removed, and Strimzi has no way to discover how much listener capacity a gateway has left.

### Per-broker ports without a template

Ports could be configured only through `.configuration.brokers[].networkPort`.
This was rejected as the only option because it is verbose and, more importantly, it breaks on scale-out: a broker added without a matching entry would have no port.
The template makes the port a pure function of the node ID, so new brokers are handled automatically as long as the matching gateway listener already exists.
Explicit per-broker ports are still supported for users who need a specific mapping.
