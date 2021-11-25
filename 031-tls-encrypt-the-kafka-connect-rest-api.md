# TLS encrypting the Kafka Connect REST API

This proposal adds the option to encrypt communication with the Kafka Connect REST API by enabling TLS encryption on the Kafka Connect REST listener.

## Current situation

Currently, instances of Kafka Connect that are deployed by the Strimzi operators are configured with the default REST API endpoint settings.
This means that the Kafka Connect REST API endpoint uses HTTP on port 8083 and that the Strimzi `KafkaConnect`, `KafkaConnector` and `KafkaMirrorMaker2` operators make unsecured REST client calls based on this default configuration.

The default network policies created by the Strimzi operators restrict incoming REST API calls to only allow access from the operator pod.
Users can define further network policies to override the default policy and allow wider access.

## Motivation

There is currently no way to TLS encrypt the Kafka Connect REST API, leaving the Connect REST listener as perhaps the last key endpoint that cannot currently be encrypted using Strimzi.

## Proposal

This proposal adds REST api configuration options to the `KafkaMirrorMaker2` and `KafkaConnect` CRDs.

A new field is added to the `KafkaConnect` and `KafkaMirrorMaker2` specs named `restListeners`, which allows users to configure the Kafka Connect configuration to have:
  - a single HTTP listener on port 8083
  - a single encrypted HTTPS listener on port 8443
  - both an HTTP listener on port 8083 and an encrypted HTTPS listener on port 8443
  
This enables users to access the REST API with TLS if required.
The `KafkaConnector`, `KafkaConnect` and `KafkaMirrorMaker2` operators will use the TLS encrypted listener when it is enabled, even if an unencrypted listener is present.

The `restListeners` field is designed for potential future extension, and is a list of items that contains a `protocol` field and an optional `caCertChainAndKey` field.

For this proposal, the `protocol` field only supports two values: `http` and `https`. If the `restListeners` list is empty the `KafkaConnect` and `KafkaMirrorMaker2` 
runtimes will be created with a single unencrypted listener on 8083 to match the existing behaviour.

The `caCertChainAndKey` field will be required if the `https` protocol is selected.

For example, the following CR will enable the HTTP listener on port 8083 and the HTTPS listener on port 8443:
```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
spec:
  restListeners:
  - protocol: http
  - protocol: https
  ...
```

The schema for `restListeners` will be as follows:
```
openAPIV3Schema:
  type: object
  properties:
    spec:
      type: object
      properties:
        restListeners:
          type: array
          items:
            type: object
            properties:
              protocol:
                type: string
                enum:
                - http
                - https
                description: The protocol of the REST listener.
              caCertChainAndKey:
                type: object
                properties:
                  certificate:
                    type: string
                    description: The name of the ca certificate file in the Secret.
                  key:
                    type: string
                    description: The name of the ca private key file in the Secret.
                  secretName:
                    type: string
                    description: The name of the Secret containing the certificate.
                  required:
                    - key
                    - certificate
                    - secretName
                description: CA certificate to use to generate.
            required:
            - protocol
          description: List of additional REST listeners.
        ...
```

the above CR would result in the following properties in the generated connect configuration:

```
listeners: https://:8443,http://:8083
rest.advertised.listener: https
rest.advertised.port: 8443
listeners.https.ssl.client.auth: none
listeners.https.ssl.keystore.location: /tmp/kafka/kafka-connect-rest.keystore.p12
listeners.https.ssl.keystore.password: ***generated password***
listeners.https.ssl.keystore.type: PKCS12
```

### Certificates

If the user decides to enable an encrypted HTTPS listener, they must have the `caCertChainAndKey` 
property present in the `KafkaConnect` CR. The property must point to an existing secret that contains the 
CA certificate and CA private key that they want Kafka Connect to use. This secret can be created by the 
user, or it can be the [Strimzi cluster CA secret](#using-the-strimzi-cluster-ca).

The CA certificate and CA private key configured by the user in the `KafkaConnect` CR will be used by the Kafka Connect 
operator to generate a self-signed certificate. This self-signed certificate will be used to configure the keystore and 
truststore for Kafka Connect. The CA certificate will also be used to generate a truststore 
that the Strimzi operators, e.g. KafkaConnector, MirrorMaker2 can use when communicating with Kafka Connect.

#### Using the Strimzi cluster CA

If the Kafka Connect cluster is connecting to a Kafka instance that is managed by Strimzi, the user can configure the 
`caCertChainAndKey` property to point to the Strimzi cluster CA.

A `KafkaConnect` instance with a secure endpoint using the cluster CA will look like:

```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
spec:
  restListeners:
  - protocol: https
    caCertChainAndKey:
      secretName: <cluster-name>-cluster-ca
      key: ca.key
      certificate: ca.crt
  ...
```

As part of this proposal the `<cluster-name>-cluster-ca` secret will be updated to include the cluster CA certificate 
as currently it only includes the private key.

### Certificate renewals

The self-signed TLS certificate will have a 1-year expiration.
Thirty days before the self-signed TLS certificate expires, the operator will automatically renew the certificate, replace the old self-signed certificate, and restart the pods in a controlled manner to ensure the new certificates are in use by Kafka Connect without losing availability.

This would need to be a multi-phase process consisting of the following steps:

1. Generate new certificate
2. Distribute the new certificate to all pods and cluster operator truststores
3. Replace the key and roll all Connect pods
4. When the old certificate expires, remove it from the truststores and roll all Connect pods

## Affected/not affected projects

This proposal affects the `strimzi-kafka-operator` project.

## Compatibility

This proposal does not change the default behaviour of the Kafka Connect or MirrorMaker2 clusters so any existing clusters will not be affected.

## Rejected alternatives

### TLS encrypting the Kafka Connect REST API by default

[Proposal #8](008-tls-encrypt-the-kafka-connect-rest-api.md) proposed enabling a TLS encrypted listener be default. The problem with this approach is it is 
not clear what CA should be used to generate a self-signed certificate. The `KafkaConnect` CR does not provide an option to indicate the Strimzi cluster that 
it is connecting to. This is because the `KafkaConnect` CR and operator are often used with a Kafka cluster that isn't managed by Strimzi. Instead, the CR provides 
specific properties such as `bootstrapServers` to provide the listener address and `tls` to provide the certificate to trust.

### Annotation alternative to enable a plain HTTP REST API listener

Alternatively, a new annotation `strimzi.io/enable-plain-rest-listener` could be added, to avoid changing the spec.
Setting the `strimzi.io/enable-plain-rest-listener` annotation value to `true` would add the additional unencrypted HTTP listener on port 8083.
This was rejected as it is generally better to have configuration in the `spec`, and the proposed `restListeners` structure could be extended in future enhancements.
