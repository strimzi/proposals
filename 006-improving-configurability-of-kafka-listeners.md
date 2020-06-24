# Improving configurability of Kafka listeners

## Current situation

Kafka listeners are configured in the `Kafka` custom resource in `.spec.kafka.listeners`.
Apart from the non-configurable internal replication listener, users can configure only 3 listeners:
* `plain` listener available within the Kubernetes cluster only.
The only configuration options are authentication and network policy rules.
* `tls` listener available within the Kubernetes cluster only.
The only configuration options are authentication, network policy rules and custom certificates.
* `external` listener available for use outside of the Kubernetes cluster. 
This listener has 4 different types (`nodeport`, `loadbalancer`, `ingress` and `route`).
Some of these types have extensive configuration options including overriding advertised hosts etc.

The listeners have fixed ports:
* 9092 for the `plain` listener
* 9093 for the `tls` listener
* 9094 for the `external` listener

The configuration in the Kafka custom resource can look for example like this:

```yaml
    listeners:
      plain:
        authentication:
          type: scram-sha-512
        networkPolicyPeers:
          - podSelector:
              matchLabels:
                app: kafka-plaintext-consumer
          - podSelector:
              matchLabels:
                app: kafka-plaintext-producer
      tls:
        authentication:
          type: tls
        networkPolicyPeers:
          - podSelector:
              matchLabels:
                app: kafka-consumer
          - podSelector:
              matchLabels:
                app: kafka-producer
      external:
        type: route
        authentication:
          type: tls
```

But in the simple form it might look like this when now configuration options are used, just the listeners are enabled:

```yaml
    listeners:
      plain: {}
      tls: {}
```

In the operator code, each listener has currently its own code path which implements it.
This works fine, but seems to offer some space for improvement since there are many similarities between the listeners and processing them by the same code would be more useful.
But our current implementation makes that hard because each listener has its own class in the `api` module with its own configs, settings etc.

## Motivation for change

The current implementation has many limitations
* Only 3 listeners can be configured
* 2 out of the 3 listeners have fixed roles in terms of encryption, so when you want to use for example encrypted listeners only, you have effectively only 2 available listeners.
* The `plain` and `tls` listeners do not offer some additional configuration options such as overriding advertised hosts etc.
* Only one listener can be used for outside of the Kubernetes cluster.
* Since Kafka listeners can have each only single authentication mechanism, possibilities for offering different authentication to different clients are limited.
* Port numbers are assigned and are not configurable.

Some of the use cases raised by out users in the past are listed here:

### Multiple external listeners

In some situations, having multiple external listener could be very useful.
Apart from the obvious such as multiple authentication mechanisms or listeners with and without encryption, multiple listeners can be also useful to handle access from different networks outside the Kubernetes cluster.

For example in AWS, you can have two types of load balancers: public and internal.
Public load balancers are exposed to the Internet and can be used by applications running anywhere online.
On the other hand internal load balancers are exposed only within the VPC (Virtual Private Cloud) - and accessible only to the applications running outside of Kubernetes but inside the private network of the user.

### Configurability of internal listeners

Some users heavily customize their Kubernetes network.
One of the examples is joining their Kubernetes network with their network outside of Kubernetes.
In such case, a common problem is using or not using the cluster service DNS suffix (`.cluster.local` by default) which is in some cases needed and in some not.
Having the ability to override the advertised hostnames on internal listeners similarly to how they can be treated on the external listener or being able to have one internal listeners could help in these situations.

## Proposed changes

This proposal tries to make the configuration of the listeners more flexible.
It should use array / list instead of an object with predefined listeners.
It should also open more configuration options to all of them.

_Note: The replication listener should be still kept non-configurable and hidden from users._

The new configuration can look like this:

```yaml
    listeners:
      - name: encrypted1
        port: 9092
        type: service
        tls: true
        authentication:
          type: tls
        overrides:
          brokers:
            - broker: 0
                advertisedHost: my-cluster-brokers-kafka-0.myns.cluster.local
            - broker: 1
                advertisedHost: my-cluster-brokers-kafka-1.myns.cluster.local
            - broker: 2
                advertisedHost: my-cluster-brokers-kafka-2.myns.cluster.local
      - name: encrypted2
        port: 9093
        type: service
        tls: true
        authentication:
          type: scram-sha-512
      - name: routes
        port: 9094
        type: route
        tls: true
        authentication:
          type: tls
      - name: nodeports
        port: 9095
        type: nodeport
        tls: true
        authentication:
          type: tls
```

The listeners will be configurable as array.
Each listener will have its own unique name and will specify a type and unique port as required values.
The port can be set to anything apart from `9091` (internal replication listener) and `9404` (Prometheus).
A new type service will be introduced for the internal listeners designed for apps running inside the same Kubernetes cluster.
Additionally, all the types from the existing external listeners will be supported as well.

Together with this change, I suggest to change the default value of the listener `tls` flag from `true` to `false`.
The current situation, when TLS is enabled by default without using the `tls` field and needed to explicitly use the `tls: false` to disable TLS seems unintuitive and confusing.

### Backwards compatibility

The old format can be easily converted into the new format without any information loss.
The example YAML below corresponds to the first example from the _Current situation_ section:

```yaml
    listeners:
      - name: plain
        port: 9092
        type: service
        tls: false
        authentication:
          type: scram-sha-512
        networkPolicyPeers:
          - podSelector:
              matchLabels:
                app: kafka-plaintext-consumer
          - podSelector:
              matchLabels:
                app: kafka-plaintext-producer
      - name: tls
        port: 9093
        type: service
        tls: true
        authentication:
          type: tls
        networkPolicyPeers:
          - podSelector:
              matchLabels:
                app: kafka-consumer
          - podSelector:
              matchLabels:
                app: kafka-producer
      - name: external
        port: 9094
        type: route
        tls: true
        authentication:
          type: tls
```

The second example could be converted like this:

```yaml
    listeners:
      - name: plain
        port: 9092
        type: service
      - name: tls
        port: 9093
        type: service
```

Being easily able to convert the old and new structures makes it easy to have single code in our operator while being able to easily work with the old format as well.

### Code changes

Since both the new and the old structure will use the same `listeners` property, the CRD generator will need to be enhanced to support the OpenAPI `oneOf` for different types.
(The CRD generator already supports `oneOf` for having allowed only one of multiple fields defined, but not for having one field with multiple different types)

```yaml
listeners:
  oneOf:
    - type: object
        properties:
          # ...
    - type: array
        items:
          # ...
```

In the `api` module, the setter and getter for the `listeners` field will use generic `JsonNode` type and inside the `api` module decide whether it should be decoded into the new or old format when the getter or setter is called.
The `api` module will keep both the old and new format to be able to also serialize back to the original CRD.
When the old format is used, it will be converted in the `fromCrd` method inside the `KafkaCluster` class in the `cluster-operator` module and from there on, only the new format will be used to configure the pods, services etc.
Since all listeners will be part of the same array, it should be possible to simplify the code configuring the listeners and just pass all of them through a single loop.

### Multiple external listeners

Having multiple external listeners would require to have multiple sets of external services.
For backwards compatibility, when the name of the external listener is `external`, the current names of the services will be used.
For any other external listeners, a separate set of services will be created and named after the name of the listener.

_Note: separate services are needed to create different external listeners with different configurations._

## Rejected alternatives

### Extending the current API

Some of the problems described in this proposal can be also handled just by extending the existing object.
But it will give less flexibility than the proposal described above and will keep the code harder to maintain.

### Using a different field for new format

The new listeners configuration could use a different field - for example `endpoints`.
That will simplify the CRD generator and the `api` module which will not need to be able to handle multiple types under the same property.
But it seems less elegant and the ability to handle multiple different types might be useful on other places as well.
