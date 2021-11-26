# Service Binding Proposal for Strimzi

This document is intended to progress discussion about how to enable Strimzi to work well with the Service Binding Operators that implement the [Service Binding spec](https://github.com/servicebinding/spec). It includes some suggested enhancements to Strimzi to make it easier for Service Binding Operators to bind an application to Strimzi. It would be good to settle on an agreement about how these two technologies can best work together so we can begin the technical work to deliver.

The Service Binding specification defines how connection information (e.g. endpoints, username, password) for services such as databases and message brokers are made available to runtime applications in Kubernetes. Version 1 of the specification is due to be released shortly.

Today, Strimzi does not fit very nicely with service binding, it both lacks the status field required by the Service Binding operator and does not make credentials available in a single Secret which is what the Service Binding Operator expects from services that implement the spec.

Contents:

 - [Current situation](#current-situation)
 - [Proposal](#proposal)
 - [Rejected Alternatives](#rejected-alternatives)
 - [Overview of Service Binding specification](#overview-of-service-binding-specification)
 - [Connecting to Strimzi today](#connecting-to-strimzi-today)

## Current situation

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
    namespace: my-namespace # optional, defaults to same namespace as the KafkaConnection
    listener: # optional, when not specified Strimzi will choose an appropriate listener (see implementation below)
      name: tls
  user: # optional
    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaUser
    name: my-barista
    namespace: my-namespace # optional, defaults to same namespace as the KafkaConnection
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
type: servicebinding.io/kafka
data:
    type: kafka
    provider: strimzi
    bootstrap.servers: # comma separated list of host:port (this matches the expected format of Kafka clients)
    security.protocol: # one of PLAINTEXT, SASL_PLAINTEXT, SASL_SSL or SSL
    # Provided if TLS enabled:
    ssl.truststore.crt: #  Strimzi cluster CA certificate (copied from ca.crt in <clustername>-cluster-ca-cert secret)
    ssl.truststore.p12: #  PKCS #12 archive file for Strimzi cluster CA certificate (copied from ca.p12 in <clustername>-cluster-ca-cert secret)
    ssl.truststore.password: # Password for protecting the Strimzi cluster CA certificate PKCS #12 archive file (copied from ca.password in <clustername>-cluster-ca-cert secret)
    # Provided if selected user is SCRAM auth:
    username: # SCRAM username
    password: # SCRAM password
    sasl.jaas.config: # sasl jaas config string for use by Java applications
    sasl.mechanism: SCRAM-SHA-512
    # Provided if selected user is mTLS:
    ssl.keystore.p12: # client certificate for the consuming client PKCS #12 archive file for storing certificates and keys (copied from user.p12 in KafkaUser secret)
    ssl.keystore.password: # password for protecting the client certificate PKCS #12 archive file (copied from user.password in KafkaUser secret)
    ssl.keystore.crt: # certificate for the consuming client signed by the clients' CA (copied from user.crt in KafkaUser secret)
    ssl.keystore.key: # private key for the consuming client (copied from user.key in KafkaUser secret)
```

Naming of secret keys:

This proposal previously considered choosing secret keys that could be reused by other (non-Kafka) operators that use mutualTLS or SASL SCRAM authentication. However, given 
that Kafka applications generally have a well known list of configuration options they provide, the proposal instead uses those well-known keys. This will make it clearer 
to application developers what the different keys should be used for.

### Implementation

This proposal will result in a new `KafkaConnectionAssemblyOperator` class being added to the `cluster-operator` directory in the `strimzi-kafka-operator` repo. The class will be 
deployed as a separate Verticle and have its own reconcile loop.

The actions taken during the reconcile loop will be:
1. The operator can be deployed at cluster-scope where it will watch resources across all namespaces, or in a single namespace, where it will only watch resources in that namespace.
2. On start-up the operator will watch all resources of Kind `KafkaConnection`.
3. For any `KafkaConnection` resource present it will add a status of NotReady to that CR.
4. For each `KafkaConnection` resource it will perform a GET request on the Kafka cluster that is 
referenced in the `KafkaConnection` and pick out the configured listeners.
   1. If this or any subsequent GET requests fail due to a 401 (unauthorised) error, the status reason will be updated to reflect this.
5. If there are multiple listeners listed in the `Kafka` CR with the type `internal`, the operator 
will filter the list by comparing the `tls` and `authentication` properties in the `Kafka` and `KafkaUser` CRs.
6. If there are still multiple listeners it will pick one at random.
7. The operator will add the name of the listener chosen will be added to the `KafkaConnection` status so that in future reconciles 
the operator can take this into account in step 4 and avoid cycling through listeners.
8. If there are no listeners marked as type `internal`, the reconcile loop will fail.
9. The operator will then create a secret and add the bootstrapServers for the chosen listener to that secret.
10. If the chosen listener uses TLS, the operator will then perform a GET request on the `cluster-ca-cert` secret and put the certificate and truststore files in the new secret.
11. If the `KafkaConnection` references a `KafkaUser`, the operator will perform a GET request on that user to find the name of the secret containing the credentials and the username 
from the status if it is provided. The operator will also perform a GET request on the user secret to get any credentials. 
12. The operator will add the cluster certificate and any user related credentials to the secret it created earlier.
13. The operator will add a `status.binding` to the `KafkaConnection` resource with the name of the secret. 
14. The operator will then create `Watch` requests for the `Kafka` CR and, if used earlier, the `cluster-ca-cert` secret, the `KafkaUser` resource and the secret created by the `KafkaUser` operator. 
15. The operator will mark the `KafkaConnection` CR as ready.
16. If the `KafkaConnection` CR or any of the other watched resources change the operator will make the matching updates to the secret it created.

## Compatibility

This proposal does not change any of the existing CRDs or the secrets that are being created. Instead it adds a new CRD and secret.

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

## Connecting to Strimzi today

This section shows what and where the current set of credentials are exposed by Strimzi as an easy reference for discussions. 

To connect to Strimzi, a service binding needs the following:

* **host** - hostname or IP address
* **port** - port number
* **userName** - username, optional
* **password** - password or token, optional
* **certificates** - one or more certificates, optional

Currently some of this comes from the `Kafka` CR, some from a certificate secret and some from `KafkaUser`.

## Host and port

The host and port for clients to use is currently contained in the bootstrapServers address in the `status` of the `Kafka CR`:

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

## Kafka cluster certificate

If TLS is enabled for an endpoint, Kafka clients need access to the CA certificate that signed the broker's server certificate.

The CA certificate is most easily obtained from the `<cluster>-cluster-ca-cert` secret:

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-cluster-cluster-ca-cert
type: Opaque
data:
  ca.p12: # CA certificate PKCS #12 archive file for storing certificates and keys
  ca.password: # password for protecting the CA certificate PKCS #12 archive file
  ca.crt: # CA certificate for the cluster
```

## User specific credentials

Strimzi provides the `KafkaUser` custom resource as a way of managing users and credentials. There are two authentication options, using mutual TLS or SASL SCRAM.

The `KafkaUser` CR status includes the username and name of the secret containing credentials:

``` yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: # scram-sha-512 or tls
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

The referenced secret contains the password and sasl.jaas.config in the SASL case, and certificate files in the mTLS case:

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
# Provided if selected user is SCRAM auth:
  password: Z2VuZXJhdGVkcGFzc3dvcmQ=
  # value when base64 decoded looks like -> org.apache.kafka.common.security.scram.ScramLoginModule required username="my-user" password="generatedpassword";
  sasl.jaas.config: b3JnLmFwYWNoZS5rYWZrYS5jb21tb24uc2VjdXJpdHkuc2NyYW0uU2NyYW1Mb2dpbk1vZHVsZSByZXF1aXJlZCB1c2VybmFtZT0ibXktdXNlciIgcGFzc3dvcmQ9ImdlbmVyYXRlZHBhc3N3b3JkIjs=
# Provided if selected user is mTLS:
  user.p12:      # User PKCS #12 archive file for storing certificates and keys
  user.password: # Password for protecting the user certificate PKCS #12 archive file
  user.crt:      # Public key of the user
  user.key:      # Private key of the user
```
