# Advertising Pod IP addresses in Kafka listeners

This proposal lets Kafka brokers advertise their Pod IP address on a listener, instead of a DNS name.
It addresses [strimzi/strimzi-kafka-operator#3984](https://github.com/strimzi/strimzi-kafka-operator/issues/3984).

## Current situation

Today, brokers on `internal` listeners advertise their Pod DNS names, such as `kafka-cluster-broker-0.kafka-cluster.myproject.svc`. These names only resolve inside the Kubernetes cluster.

`advertisedHostTemplate` supports only the `{nodeId}` and `{nodePodName}` placeholders.
The operator can't render the Pod IP into the broker configuration, because the IP is only known once the Pod starts.
The container template doesn't support the Downward API either, so users can't pass the IP in through an environment variable.

The only way to do this today is to inject the IP with a mutating admission webhook and put `${strimzienv:<VAR>}` into `advertisedHostTemplate`.
It works, but it depends on operator behavior that isn't documented and can change between releases.

## Motivation

In many environments, such as EKS with the VPC CNI or VPC-native GKE, Pod IPs are routable outside the Kubernetes cluster.
A client running in another cluster in a peered VPC can reach the brokers directly, but it can't resolve the `.svc` names they advertise.

The external listener types solve the naming problem, but they add a node port hop or a load balancer per broker to the data path, even though the Pod IPs are already reachable.
If the brokers advertise their Pod IPs, clients can talk to them directly.

## Proposal

### Principles

The proposal follows these principles:

- The operator never needs to know the IP.
The broker resolves config provider references in its configuration at startup, before it registers with the KRaft controllers.
The controllers, and the clients through the metadata, only ever see the real IP, and it is resolved again every time the Pod starts.
Node port listeners already work this way: they advertise `${strimzifile:/opt/kafka/init/external.address:...}`, which the broker resolves from a file written by the `kafka-init` container.
- The operator owns the plumbing.
Users only choose which address to advertise, in the listener configuration.
The operator adds whatever the broker needs to resolve it, so there is no Downward API in templates and nothing to keep in sync across node pools.
- Only the Pods that need it change.
The environment variable and the init container are added to broker nodes only when a listener uses the new placeholders.
- DNS names stay the default, and Pod IPs are opt-in.

### New placeholders

`advertisedHostTemplate` gets new placeholders, next to `{nodeId}` and `{nodePodName}`:


| Placeholder                                                            | Advertised address                                                                  | Resolved through       |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ---------------------- |
| `{nodePodIp}`                                                          | The primary Pod IP (`status.podIP`)                                                 | Environment variable   |
| `{nodePodIp:ipv4}`, `{nodePodIp:ipv6}`                                 | The Pod IP of the given family (from `status.podIPs`)                               | `kafka-init` container |
| `{nodePodInterfaceIp:<interface>}`, optionally with `:ipv4` or `:ipv6` | The IP on a network interface, for example one added by Multus (IPv4 if it has one) | `kafka-init` container |


The new placeholders are supported only on `internal` listeners.
Clients connect to the Pod IPs directly, so the other listener types add nothing except a bootstrap address: a load balancer for `loadbalancer`, or a node IP and node port for `nodeport`, which could be supported in a follow-up.
If a requested address doesn't exist, the init container fails, so a broker never starts with the wrong address.

#### Primary Pod IP

This covers most setups, such as EKS or GKE clusters with routable Pod IPs:

```yaml
listeners:
  - name: podip
    port: 9094
    type: internal
    tls: false
    configuration:
      advertisedHostTemplate: "{nodePodIp}"
```

The operator adds a `STRIMZI_POD_IP` environment variable to the `kafka` container of the brokers, and references it in the configuration:

```yaml
env:
  - name: STRIMZI_POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
```

```properties
advertised.listeners=...,PODIP-9094://${strimzienv:STRIMZI_POD_IP}:9094
```


#### IPv6 address in a dual-stack cluster

`status.podIP` only holds the address of the cluster's primary IP family.
To advertise the IPv6 address in a cluster where IPv4 is primary:

```yaml
configuration:
  advertisedHostTemplate: "{nodePodIp:ipv6}"
```

The operator passes `status.podIPs` to the `kafka-init` container, which picks the IPv6 address and writes it to a file:

```properties
# /opt/kafka/init/pod.address
pod.ipv6.address=fd00:10:0:1::12
```

The broker reads the address from that file, the same way node port listeners do:

```properties
advertised.listeners=...,PODIP-9094://${strimzifile:/opt/kafka/init/pod.address:pod.ipv6.address}:9094
```

Nothing changes on the broker side.
Since Strimzi 0.46, brokers use the JVM network defaults ([#11247](https://github.com/strimzi/strimzi-kafka-operator/pull/11247)), so listeners bound to `0.0.0.0` accept IPv6 connections as well.
In IPv6-only clusters, the primary Pod IP is already the IPv6 address, so `{nodePodIp}` is enough.

With DNS names, one listener serves both families: in a dual-stack cluster, the broker name resolves to both an IPv4 and an IPv6 address, and the client picks one.
A listener that advertises an IP has exactly one address, so it serves one family.
To serve clients of both families, use two listeners, one with `{nodePodIp:ipv4}` and one with `{nodePodIp:ipv6}`.

#### Address on a secondary network

Multi-network CNIs such as Multus add extra interfaces to the Pod.
They are named `net1`, `net2`, and so on, or as set in the `interface` field of the network selection:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: broker
  labels:
    strimzi.io/cluster: my-cluster
spec:
  roles:
    - broker
  template:
    pod:
      metadata:
        annotations:
          k8s.v1.cni.cncf.io/networks: '[{"name": "kafka-net", "interface": "net1"}]'
  # ...
---
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    listeners:
      - name: secondary
        port: 9095
        type: internal
        tls: false
        configuration:
          advertisedHostTemplate: "{nodePodInterfaceIp:net1}"
```

The `kafka-init` container shares the Pod's network, so it reads the address of `net1` directly and writes it to the same file, for example `interface.net1.address=192.168.50.12`.
The listener then advertises `${strimzifile:/opt/kafka/init/pod.address:interface.net1.address}`.
If the interface has an IPv4 address, it is used, otherwise its IPv6 address is used.
To always use the IPv6 address, set `{nodePodInterfaceIp:net1:ipv6}`.

We read the interface instead of the Multus `network-status` annotation.
The annotation is JSON, and neither the Downward API nor Kafka's config providers can extract a value from it.
It may also not be set yet when the container starts.
Reading the interface needs no Kubernetes API access and no JSON parsing.

#### How `kafka-init` resolves the addresses

The CNI, including Multus and the plugins it calls, creates the Pod's interfaces and assigns their addresses before any container starts.
All containers in a Pod, including init containers, share the same network, so `kafka-init` sees every interface when it runs.

No extra permissions are needed:

- No Kubernetes RBAC, because `kafka-init` doesn't call the Kubernetes API for these addresses.
- No Linux capabilities, because reading the Pod's own interfaces is unprivileged, just like running `ip addr`.
It works as non-root with all capabilities dropped.
- The Pod IPs come from `status.podIPs`, which the operator passes to `kafka-init` through the Downward API.

The operator configures `kafka-init` with environment variables, as it already does for node ports:


| Environment variable | Value                                                                    | Set when                                         |
| -------------------- | ------------------------------------------------------------------------ | ------------------------------------------------ |
| `POD_IPS`            | `status.podIPs`, for example `10.0.1.12,fd00:10:0:1::12`                 | `{nodePodIp:ipv4}` or `{nodePodIp:ipv6}` is used |
| `POD_ADDRESSES`      | The keys to write, for example `pod.ipv6.address,interface.net1.address` | Any `kafka-init` placeholder is used             |


Linux doesn't keep a label for each address.
It stores every address on an interface with its address family, which `ip addr` shows as `inet` (IPv4) or `inet6` (IPv6).
Java returns each address as an `Inet4Address` or an `Inet6Address`, and the addresses in `POD_IPS` are parsed the same way, so `kafka-init` knows the family from the type of the address.
The keys in the file are defined by Strimzi:


| Placeholder                      | Key                           |
| -------------------------------- | ----------------------------- |
| `{nodePodIp:ipv4}`               | `pod.ipv4.address`            |
| `{nodePodIp:ipv6}`               | `pod.ipv6.address`            |
| `{nodePodInterfaceIp:net1}`      | `interface.net1.address`      |
| `{nodePodInterfaceIp:net1:ipv4}` | `interface.net1.ipv4.address` |
| `{nodePodInterfaceIp:net1:ipv6}` | `interface.net1.ipv6.address` |


For interfaces, `kafka-init` first ignores loopback and link-local addresses (`169.254.0.0/16` and `fe80::/10`).
Every interface with IPv6 enabled has a `fe80::` address, and other hosts can't use it to reach the broker.
It then picks from the addresses that are left:


| Addresses on the interface | `{nodePodInterfaceIp:net1}` | `{nodePodInterfaceIp:net1:ipv4}` | `{nodePodInterfaceIp:net1:ipv6}` |
| -------------------------- | --------------------------- | -------------------------------- | -------------------------------- |
| One IPv4                   | The IPv4 address            | The IPv4 address                 | Fails                            |
| One IPv6                   | The IPv6 address            | Fails                            | The IPv6 address                 |
| One IPv4 and one IPv6      | The IPv4 address            | The IPv4 address                 | The IPv6 address                 |
| Several IPv4               | The first IPv4 address      | The first IPv4 address           | Fails                            |
| Several IPv6               | The first IPv6 address      | Fails                            | The first IPv6 address           |
| Several IPv4 and IPv6      | The first IPv4 address      | The first IPv4 address           | The first IPv6 address           |
| None                       | Fails                       | Fails                            | Fails                            |


"First" means the first address Java returns for the interface, and that order isn't defined.
Every address on the interface reaches the broker, but clients have to be able to route to the one that is picked.
For a predictable address, keep one address per family on the interface used by the listener.
The init container only fails when there is no address of the requested family, so a broker never starts without an address to advertise.

IPv6 addresses are written as plain literals, for example `pod.ipv6.address=fd00:10:0:1::12`.  
The `%<interface>` scope suffix that Java adds to interface addresses is removed before writing.

### TLS

We handle TLS the same way as for [node port listeners](https://strimzi.io/docs/operators/latest/deploying#con-hostname-varification-node-ports-str):

- The new placeholders are never added to the broker certificates, because the address isn't known when the certificate is issued.
Today, the operator tries to add a `${strimzienv:...}` template to the certificate, and the reconciliation fails with an `Invalid DNS name` error.
- `tls: true` still works: the traffic is encrypted, and clients validate the certificate chain against the cluster CA.
- Clients disable hostname verification with `ssl.endpoint.identification.algorithm=` (empty value), as already [documented for node ports](https://strimzi.io/docs/operators/latest/deploying#con-hostname-varification-node-ports-str).

Adding the IP to the certificate would need the operator to reissue it after every Pod start, and the broker to reload it without a restart, because a restart gives it a new IP.
That depends on dynamic certificate reloading, like the one discussed in [proposal PR #239](https://github.com/strimzi/proposals/pull/239), so it is out of scope here.

### Bootstrap

Pod IPs change, so clients outside the cluster still need a stable address to bootstrap from.
Any broker can answer the first metadata request, so this can be any Service or DNS record that reaches the brokers, created by Strimzi or outside of it.

### Client behavior on restarts

Strimzi used Pod IPs in its early days and moved to DNS names because changing IPs confused clients ([#50](https://github.com/strimzi/strimzi-kafka-operator/issues/50)).
That's why this feature is opt-in and DNS names stay the default.

Current Kafka clients recover from changed addresses on their own.
Strimzi rolls one broker at a time, so a client always has other brokers to refresh metadata from.
If a client connects to an old IP with nothing behind it, the connection fails and the client refreshes its metadata.
If another broker has taken over the old IP, it answers with errors like `NOT_LEADER_OR_FOLLOWER`, which also triggers a refresh.
And if every address the client knows is stale, for example after a full cluster restart, it goes back to the bootstrap address with `metadata.recovery.strategy=rebootstrap`.
This was added in Kafka 3.8 ([KIP-899](https://cwiki.apache.org/confluence/display/KAFKA/KIP-899%3A+Allow+producer+and+consumer+clients+to+rebootstrap)) and is the default since Kafka 4.0 ([KIP-1102](https://cwiki.apache.org/confluence/x/KYuMEw)).

### Implementation plan

1. Document the new placeholders in the `advertisedHostTemplate` description.
  The field is a plain string, so the CRD schema doesn't change.
2. Render the placeholders in `ListenersUtils.brokerAdvertisedHost`, as `${strimzienv:STRIMZI_POD_IP}` or `${strimzifile:/opt/kafka/init/pod.address:<key>}`.
  `ModelUtils.renderTemplate` is shared with volume templates, so the new placeholders stay out of it.
3. In `KafkaCluster`, when a listener uses the placeholders, add `STRIMZI_POD_IP` to the `kafka` container with `ContainerUtils.createEnvVarFromFieldRef`, or enable the `kafka-init` container on the broker nodes and set its `POD_IPS` and `POD_ADDRESSES` environment variables.
4. Add a `writePodAddresses` method to `InitWriter` in `kafka-init`, next to `writeExternalAddress`.
  It resolves the keys in `POD_ADDRESSES` from `POD_IPS` and `java.net.NetworkInterface` with the rules above, and writes them to `/opt/kafka/init/pod.address`.
   It needs no Kubernetes API access and no extra permissions.
5. Skip templates with the new placeholders when collecting certificate SANs in `KafkaListenersReconciler`.
6. Validate the placeholders in `ListenersValidator`: use only on `internal` listeners and only in `advertisedHostTemplate`, a known IP family, and a valid interface name.
7. Document Pod IP listeners, including the bootstrap address, TLS, and the recommended client settings.
8. Add unit tests, and a system test where clients keep producing and consuming through a `{nodePodIp}` listener during a rolling update of the brokers.


## Affected/not affected projects

Only `strimzi-kafka-operator` is affected: the `advertisedHostTemplate` API description, the Cluster Operator, the `kafka-init` container, and the documentation.
The Kafka container image doesn't change.

## Compatibility

The feature is opt-in, and existing listeners keep advertising DNS names.
The new placeholders don't mean anything today, so no existing configuration changes behavior.
Templates that use `${strimzienv:...}` directly keep passing through as they are.
Users running the webhook workaround can switch to `{nodePodIp}` and drop the webhook, which rolls the brokers once.

## Rejected alternatives

### Downward API in the container template

Adding `fieldRef` to the container template would let users define the environment variable themselves, and reference it with `${strimzienv:<NAME>}`.
But the Downward API in templates raises security concerns, and users would have to keep the variable name in sync between the node pool templates and the listener.
It also can't pick the IPv6 address or a secondary network address.
The Downward API returns `status.podIPs` as a list and the Multus `network-status` as JSON, and Kafka can't extract a single address from either.

### Always setting `STRIMZI_POD_IP`

The operator could add `STRIMZI_POD_IP` to every broker, and users would reference it with `${strimzienv:STRIMZI_POD_IP}`.
It only covers the primary Pod IP, and it changes every broker Pod, rolling all brokers after the operator upgrade even when nothing uses it.
