# Service Binding Proposal for Strimzi

This document is intended to progress discussion about how to enable Strimzi to work well with the Service Binding Operators that implement the [Service Binding spec](https://github.com/servicebinding/spec). It includes some suggested enhancements to Strimzi to make it easier for Service Binding Operators to bind an application to Strimzi. It would be good to settle on an agreement about how these two technologies can best work together so we can begin the technical work to deliver.

The Service Binding specification defines how connection information (e.g. endpoints, username, password) for services such as databases and message brokers are made available to runtime applications in Kubernetes. Version 1 of the specification is due to be released shortly.

Today, Strimzi does not fit very nicely with service binding, it both lacks the status field required by the Service Binding operator and does not make credentials available in a single Secret which is what the Service Binding Operator expects from services that implement the spec.

Contents:

 - [Current problems with integrating Strimzi and a Service Binding Operator](#current-problems-with-integrating-strimzi-and-a-service-binding-operator)
 - [Proposal](#proposal)
 - [Rejected Alternatives](#rejected-alternatives)
 - [Overview of Service Binding specification](#overview-of-service-binding-specification)
 - [Connecting to Strimzi](#connecting-to-strimzi)

## Current problems with integrating Strimzi and a Service Binding Operator

At a high level these are the current problems that make integrating Strimzi with a Service Binding Operator hard:

1. Bootstrap Server information is not contained in any secret
2. Information is spread across multiple `Secret` resources
3. The user has to determine which `KafkaUser` can access which listener

## Proposal

This proposal introduces a new custom resource type called `KafkaConnection` and the application creates a `ServiceBinding` that binds to this service.

Required changes:

 - Add a new Custom Resource called `KafkaConnection` that refers to the `Kafka`, the specific listener and the `KafkaUser` that should be used
 - The Strimzi operator adds a `status.binding` to the `KafkaConnection` status that points to a Kubernetes `Secret`
 - The Strimzi operator creates a `Secret` in the same namespace as the `KafkaConnection` containing all the required information

`KafkaConnection` CR spec:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnection
metadata:
  name: barista-kafka
spec:
  kafka:
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    name: my-cluster
    namespace: my-namespace # optional
    listener: # optional
      name: tls # optional
  user: # optional
    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaUser
    name: my-barista
    namespace: my-namespace # optional
status:
  binding:
    name: barista-kafka
```

`Secret` created by Strimzi:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: barista-kafka
  namespace: my-namespace
type: Opaque
  data:
    bootstrapServer: # comma separated list of host:port
  # Provided if TLS enabled:
    ca.crt: #  Strimzi cluster CA certificate
    ca.p12: #  PKCS #12 archive file for Strimzi cluster CA certificate
    ca.password: # Password for protecting the Strimzi cluster CA certificate PKCS #12 archive file
  # Provided if selected user is SCRAM auth:
    username: # SCRAM username
    password: # SCRAM password
    sasl.jaas.config: # sasl jaas config string for use by Java applications
  # Provided if selected user is mTLS:
    user.p12: # client certificate for the consuming client PKCS #12 archive file for storing certificates and keys
    user.password: # password for protecting the client certificate PKCS #12 archive file
    user.crt: # certificate for the consuming client signed by the clients' CA
    user.key: # private key for the consuming client
```

Naming of secret keys:

The names for the keys given above match what is currently provided in other secrets created by Strimzi. The service binding specification does included some suggested fields for certificates, but they do not satisfy the requirements of an application connecting to Strimzi. These fields could be proposed back to the Service Binding specification.

An alternative approach for naming is to name the fields to match the use, for example `ca` -> `truststore` (`truststore.crt`, `truststore.p12`, `truststore.password`), `user` -> `keystore` (`keystore.p12`, `keystore.password`, `keystore.crt`, `keystore.key`)
 
Considerations:

- This approach results in a new Custom Resource type that is just for convenience
- Someone, whether it is the application developer or owner of the Kafka cluster, has to decide which KafkaConnection resources to create and which cluster, user and listener to choose
- A lot more changes required than other approaches, but perhaps fits best with the Service Binding Operator view of the world

## Rejected Alternatives

### 1 - Application binds to KafkaUser only

In this approach the application creates a `ServiceBinding` for a specific `KafkaUser` CR. The credential information is bound to the application and the cluster certificate and bootstrap server address is obtained by the application in another way.

Required changes:

- Update the `KafkaUser` CR to include a `status.binding` field where the `name` matches the current field `secret` in the status
- Update the secret created for the `KafkaUser` to include the username when the `KafkaUser` is using type SASL SCRAM

Considerations:

- In this approach a lot of work is still left up to the application developer to make sure their application has the correct cluster certificate and bootstrap server address.
    - Either they would have to look up the cluster certificate and bootstrap server address and hard code it into their application or a secret in their cluster, or they would have to write custom logic into their app so that it can query the Kafka CR.

### 2 - Application uses multiple ServiceBinding resources

In this approach the application creates two `ServiceBinding` resources, one that will bind to the `Kafka` CR and one that will bind to the `KafkaUser` CR.

Required changes:

- Update the `KafkaUser` CR to include a `status.binding` field where the `name` matches the current field `secret` in the status
- Update the secret created for the `KafkaUser` to include the username when the `KafkaUser` is using type SASL SCRAM
- Update the `Kafka` CR to include a `status.binding` field, where the `name` refers to a new secret containing both the cluster certificate and bootstrapServers
- Add a new secret that contains the bootstrap servers for each listener and the cluster certificate information
    - e.g. for listeners, the secret data could be formatted to have one string with all listeners:
      ```yaml
      data:
        listeners: aW50ZXJuYWxfa2Fma2Euc3ZjOjkwOTIsZXh0ZXJuYWxfbXlob3N0LmNvbTo0NDM= # when base64 decoded something like internal_kafka.svc:9092,external_myhost.com:443
        ca.crt: # Strimzi cluster CA certificate
        ca.p12: # PKCS #12 archive file for Strimzi cluster CA certificate
        ca.password: # Password for protecting the Strimzi cluster CA certificate PKCS #12 archive file
      ```
    - e.g. the secret could contain a separate entry for each listener:
      ```yaml
      data:
      listener.internal: aW50ZXJuYWxfbG9jYWxob3N0OjkwODA= # when base64 decoded something like internal_kafka.svc:9092
      listener.external: ZXh0ZXJuYWxfbXlob3N0LmNvbTo0NDM= # when base64 decoded something like external_myhost.com:443
      ca.crt: # Strimzi cluster CA certificate
      ca.p12: # PKCS #12 archive file for Strimzi cluster CA certificate
      ca.password: # Password for protecting the Strimzi cluster CA certificate PKCS #12 archive file
      ```

**Note:** The Service Binding spec does contain some suggested secret fields for certificates, but they will not suffice to encapsulate all the certificate related information that an application needs. Hence, the suggestion of the separate fields above.

Considerations:

- This approach results in a new secret being required that contains copies of existing secrets
- It does not help the user to decide which listener to use. The application would need to parse the secret to pick out the correct listener from the list

### 3 - Application binds to KafkaUser resource that contains all details

In this approach the application only binds to the `KafkaUser` and the secret created for the `KafkaUser` contains all the required details.

Required changes:

- Update the `KafkaUser` CR to include a `status.binding` field where the `name` matches the current field `secret` in the status
- Update the secret created for the `KafkaUser` to include the username when the `KafkaUser` is using type SASL SCRAM
- Update the secret created for the `KafkaUser` to include the bootstrap servers for each listener
- Update the secret created for the `KafkaUser` to include the cluster certificate information

Considerations:

- This approach results in a lot more information being added to each `KafkaUser` secret
- It does not help the user to decide which listener to use. The application would need to parse the secret to pick out the correct listener from the list

Possible extensions to allow only one bootstrap server to be listed:

- Could only add bootstrap server addresses for listeners that support the authentication type that the `KafkaUser` supports
- Could update the `KafkaUser` CR to include `spec.binding.listener` field which would then determine the listener bootstrap server address that is added to the secret

## Overview of Service Binding specification

This gives a short introduction to the Service Binding specification to help demonstrate the requirements they would place on Strimzi. See full spec on [GitHub](https://github.com/servicebinding/spec).

### Binding Secret in Status

The Service Binding specification requires that a [provisioned service](https://github.com/servicebinding/spec#provisioned-service) includes in its status a field called `binding`. Where this field contains a single field `name` that refers to the name of a secret that should be mounted to any application that requests to bind to the service. The specification then further details recommended formats for that secret.

```yaml
status:
  binding:
    name: # secret name
```

### Secret reference in ServiceBinding resource

The Service Binding specification requires that an application is bound to a service using a Custom Resource of kind `ServiceBinding`. The [ServiceBinding](https://github.com/servicebinding/spec#service-binding) includes a `workload` section that defines the application that is to be bound and a `service` section that describes the service the application is bound to.

``` yaml
apiVersion: servicebinding.io/v1alpha3
kind: ServiceBinding
metadata:
  name: barista-kafka
spec:
  workload:
    apiVersion: apps/v1
    kind: Deployments
    name: barista-kafka
  service:
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    name: my-cluster
```

If the service being bound does not contain a `status.binding` field the `ServiceBinding` can directly reference the secret to use in one of two ways:

- Including a `spec.env` section that refers to a `Secret` that is made available by the service
- Replacing the `spec.service` section with a direct reference to a `Secret` that is made available by the service

#### Env Mapping Example

```yaml
apiVersion: servicebinding.io/v1alpha3
kind: ServiceBinding
metadata:
  name: barista-kafka
spec:
  workload:
    apiVersion: apps/v1
    kind: Deployments
    name: barista-kafka
  service:
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    name: my-cluster
  env:
    name: my-user
    key: password
```

#### Direct Secret Reference Example

```yaml
apiVersion: servicebinding.io/v1alpha3
kind: ServiceBinding
metadata:
  name: barista-kafka
spec:
  workload:
    apiVersion: apps/v1
    kind: Deployments
    name: barista-kafka
  service:
    apiVersion: v1
    kind: Secret
    name: my-user
```

## Connecting to Strimzi

To connect to Strimzi, a service binding needs the following:

* **host** - hostname or IP address
* **port** - port number
* **userName** - username, optional
* **password** - password or token, optional
* **certificate** - certificate, optional

Some of this comes from the `Kafka` CR and some from `KafkaUser`. Below is detailed the requirements for different configurations of Strimzi and why the current behaviour does not conform to the Service Binding spec.

## Binding to a Kafka cluster with no TLS or authentication

The bootstrapServers address for clients to use is currently in the `status` of the `Kafka CR`:

``` yaml
status:
  listeners:
  - type: plain
    addresses:
    - host: my-cluster-kafka-bootstrap.my-namespace.svc
      port: 9092
    bootstrapServers: my-cluster-kafka-bootstrap.my-namespace.svc:9092
  - type: tls
    addresses:
    - host: my-cluster-kafka-bootstrap.my-namespace.svc
      port: 9093
    bootstrapServers: my-cluster-kafka-bootstrap.my-namespace.svc:9093
```

The problem with the location of this data is that it is not contained in a `Secret` and is therefore not accessible to a Service Binding Operator. Since there are also multiple listeners to choose from it is not obvious which should be placed in a `Secret` or how the application chooses the correct listener to call.

## Binding to a Kafka cluster with TLS but no authentication

The addition with this scenario is that Kafka clients need access to the CA certificate that signed the broker's server certificate.

The CA certificate is most easily obtained from the `<cluster>-cluster-ca-cert` secret. While this name is predictable, it is not known to the Service Binding Operator. In addition the application also needs the bootstrapServer information, but that is not contained in the currently provided `Secret`.

The consuming client needs to know the following binding information:

* **bootstrapServer** - bootstrap server information for the listener it is contacting
* **ca.p12** - CA certificate PKCS #12 archive file for storing certificates and keys
* **ca.password** - password for protecting the CA certificate PKCS #12 archive file
* **ca.crt** - CA certificate for the cluster

This information is spread across the `Kafka` CR and the `Secret`.

## Binding to a Kafka cluster with username/password authentication

Strimzi provides the `KafkaUser` custom resource as a way of managing users and credentials. Using the SASL SCRAM mechanism, the consuming client's credentials are made available in a combination of the CR's status and a secret.

The `KafkaUser` CR contains information about the user and the secret with the password. For example:

``` yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
    - resource:
        type: topic
        name: my-topic
        patternType: literal
      operation: Read
status:
  username: my-user-name
  secret: my-user
```

The secret looks like this:

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-user
  labels:
    strimzi.io/kind: KafkaUser
    strimzi.io/cluster: my-cluster
type: Opaque
data:
  password: Z2VuZXJhdGVkcGFzc3dvcmQ=
  # value when base64 decoded looks like -> org.apache.kafka.common.security.scram.ScramLoginModule required username="my-user" password="generatedpassword";
  sasl.jaas.config: b3JnLmFwYWNoZS5rYWZrYS5jb21tb24uc2VjdXJpdHkuc2NyYW0uU2NyYW1Mb2dpbk1vZHVsZSByZXF1aXJlZCB1c2VybmFtZT0ibXktdXNlciIgcGFzc3dvcmQ9ImdlbmVyYXRlZHBhc3N3b3JkIjs=
```

However, currently the status of the `KafkaUser` does not contain the `status.binding` information and the username is missing from the secret.

When you combine this with the use of TLS that means the following information is required:

* **bootstrapServer** - bootstrap server information for the listener the application is calling
* **ca.p12** - CA certificate PKCS #12 archive file for storing certificates and keys
* **ca.password** - password for protecting the CA certificate PKCS #12 archive file
* **ca.crt** - CA certificate for the cluster
* **username** - username for the consuming client
* **password** - password for the consuming client

This information is spread across the `Kafka` CR, the `KafkaUser` CR and two `Secret` resources.


## Binding to a Kafka cluster with mutual TLS authentication

The `KafkaUser` CR contains the name of the secret that has the certificate information. For example:

``` yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
    - resource:
        type: topic
        name: my-topic
        patternType: literal
      operation: Read
status:
  username: my-user-name
  secret: my-user
```

The secret looks like this:

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-user
  labels:
    strimzi.io/kind: KafkaUser
    strimzi.io/cluster: my-cluster
type: Opaque
data:
  user.p12:      # User PKCS #12 archive file for storing certificates and keys
  user.password: # Password for protecting the user certificate PKCS #12 archive file
  user.crt:      # Public key of the user
  user.key:      # Private key of the user
```

The binding application needs the following information:

* **bootstrapServer** - bootstrap server information for the listener the application is calling
* **ca.p12** - CA certificate PKCS #12 archive file for storing certificates and keys
* **ca.password** - password for protecting the CA certificate PKCS #12 archive file
* **ca.crt** - CA certificate for the cluster
* **user.p12** - client certificate for the consuming client PKCS #12 archive file for storing certificates and keys
* **user.password** - password for protecting the client certificate PKCS #12 archive file
* **user.crt** - certificate for the consuming client signed by the clients' CA
* **user.key** - private key for the consuming client

This information is spread across the `Kafka` CR, the `KafkaUser` CR and two `Secret` resources.
