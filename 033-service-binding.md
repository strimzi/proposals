# Service Binding Proposal for Strimzi

This document is intended to progress discussion about how to enable Strimzi to work well with the Service Binding Operators that implement the [Service Binding spec](https://github.com/servicebinding/spec). It includes some suggested enhancements to Strimzi to make it easier for Service Binding Operators to bind an application to Strimzi. It would be good to settle on an agreement about how these two technologies can best work together so we can begin the technical work to deliver.

The Service Binding specification defines how connection information (e.g. endpoints, username, password) for services such as databases and message brokers are made available to runtime applications in Kubernetes. Version 1 of the specification is due to be released shortly.

Today, Strimzi does not fit very nicely with service binding, it both lacks the status field required by the Service Binding operator and does not make credentials available in a single Secret which is what the Service Binding Operator expects from services that implement the spec.

Contents:

 - [Current situation](#current-situation)
 - [Proposal](#proposal)
 - [Rejected Alternatives](#rejected-alternatives)
 - [Overview of Service Binding specification](#overview-of-service-binding-specification)

## Current situation

At a high level these are the current problems that make integrating Strimzi with a Service Binding Operator hard:

1. Bootstrap Server information is not contained in any secret
2. Information is spread across multiple `Secret` resources
3. The user has to determine which `KafkaUser` can access which listener

## Proposal

This proposal introduces a new custom resource type called `KafkaAccess` and the application creates a `ServiceBinding` that binds to this service.

Required changes:

 - Add a new Custom Resource called `KafkaAccess` that refers to the `Kafka`, the specific listener and the `KafkaUser` that should be used
 - The Strimzi operator adds a `status.binding` to the `KafkaAccess` status that points to a Kubernetes `Secret`
 - The Strimzi operator creates a `Secret` in the same namespace as the `KafkaAccess` containing all the required information

`KafkaAccess` CR spec:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaAccess
metadata:
  name: barista-kafka
spec:
  kafka:
    apiVersion: kafka.strimzi.io/v1beta2 # optional
    name: my-cluster
    namespace: my-namespace # optional, defaults to same namespace as the KafkaAccess
    listener: # optional, when not specified Strimzi will choose an appropriate listener (see implementation below)
      name: tls
  user: # optional
    apiVersion: kafka.strimzi.io/v1beta2 # optional
    kind: KafkaUser # this is required to allow possible extensions where the user specifies a secret instead of a KafkaUser
    name: my-barista
    namespace: my-namespace # optional, defaults to same namespace as the KafkaAccess
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

This proposal will introduce a new `KafkaAccess` operator. The operator will be separate from the existing `strimzi-kafka-operator` and live in its own GitHub repo in the `strimzi` organization.

The actions taken during the reconcile loop of the new operator will be:
1. The operator can be deployed at cluster-scope where it will watch resources across all namespaces, or in a single namespace, where it will only watch resources in that namespace.
2. On start-up the operator will watch all resources of Kind `KafkaAccess`.
3. For any `KafkaAccess` resource present it will add a status of NotReady to that CR.
4. For each `KafkaAccess` resource it will perform a GET request on the Kafka cluster that is 
referenced in the `KafkaAccess` and pick out the configured listeners.
   1. If this or any subsequent GET requests fail due to a 401 (unauthorised) error, the status reason will be updated to reflect this.
5. The operator will determine which listener to choose:
   1. If there are no listeners in the `Kafka` CR the reconcile loop will fail.
   2. If the `KafkaAccess` specifies a listener, the operator will choose this listener.
   3. If there are no listeners specified in the `KafkaAccess`, but only one listener in the `Kafka` CR, the operator will choose that listener.
   4. If there are no listeners specified in the `KafkaAccess`, but multiple listeners listed in the `Kafka` CR, the operator 
   will filter the list by comparing the `tls` and `authentication` properties in the `Kafka` and `KafkaUser` CRs.
   5. If there are still multiple listeners after filtering for `tls` and `authentication` the operator will choose the one that is of type `internal`.
   6. If there are still multiple listeners after filtering for `internal` the operator will sort the listeners alphabetically by name and choose the first one.
   7. If the operator fails to find a suitable listener after filtering the reconcile loop will fail.
8. The operator will then create a secret and add the bootstrapServers for the chosen listener to that secret.
9. If the chosen listener uses TLS, the operator will then perform a GET request on the `cluster-ca-cert` secret and put the certificate and truststore files in the new secret.
10. If the `KafkaAccess` references a `KafkaUser`, the operator will perform a GET request on that user to find the name of the secret containing the credentials and the username 
from the status if it is provided. The operator will also perform a GET request on the user secret to get any credentials. 
11. The operator will add any user related certificates and credentials to the secret it created earlier.
12. The operator will add a `status.binding` to the `KafkaAccess` resource with the name of the secret. 
13. The operator will then use a `Watch` or `Informer` to keep track of changes to the `Kafka` CR and, if used earlier, the `cluster-ca-cert` secret, the `KafkaUser` resource and the secret created by the `KafkaUser` operator,
14. The operator will mark the `KafkaAccess` CR as ready.
15. If the `KafkaAccess` CR or any of the other watched resources change the operator will make the matching updates to the secret it created.

## Compatibility

This proposal does not change any of the existing CRDs or the secrets that are being created. Instead, it adds a new CRD and secret.

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
