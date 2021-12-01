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

The `restListeners` field is designed for potential future extension, and is a list of items that contains a `protocol` field and fields for providing certificates if required `certChainAndKey`, `trustedCertificates`

For this proposal, the `protocol` field only supports two values: `http` and `https`. If the `restListeners` list is empty, or not specified at all, the `KafkaConnect` and `KafkaMirrorMaker2` 
runtimes will be created with a single unencrypted listener on 8083 to match the existing behaviour.

The `certChainAndKey` field will be required if the `https` protocol is selected. The `trustedCertificates` will be required if the `https` protocol is selected and the provided 
certificate is not signed by a well-known CA.

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
              certChainAndKey:
                type: object
                properties:
                  certificate:
                    type: string
                    description: The name of the certificate file in the Secret.
                  key:
                    type: string
                    description: The name of the private key file in the Secret.
                  secretName:
                    type: string
                    description: The name of the Secret containing the certificate.
                  required:
                    - key
                    - certificate
                    - secretName
                description: Certificate for Kafka Connect to use to create a keystore for the rest listener.
              trustedCertificates:
                type: object
                properties:
                  certificate:
                    type: string
                    description: The name of the certificate file in the Secret.
                  secretName:
                    type: string
                    description: The name of the Secret containing the certificate.
                  required:
                    - certificate
                    - secretName
                description: Certificate for Strimzi to trust when making calls to the Connect rest listener.
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

If the user decides to enable an encrypted HTTPS listener, they must have the `certChainAndKey` 
property present in the `KafkaConnect` CR. The property must point to an existing secret that contains the 
certificate and private key that they want Kafka Connect to use. The certificate must contain the correct 
Subject Alternative Names (SANs) similarly to when a user provides their own certificates for listeners:
https://strimzi.io/docs/operators/latest/using.html#ref-alternative-subjects-certs-for-listeners-str

If the user has enabled an encrypted HTTPS listener and the certificate they referenced in the `certChainAndKey` field 
is not signed by a CA root certificate that is trusted by the Java runtime by default (i.e. signed by a well known CA like 
Verisign or Let's Encrypt), they must also configure the `trustedCertificates`. This property must point to an 
existing secret that contains the certificate that was used to sign the certificate provided in the `certChainAndKey` fields. 
This certificate will be used by Strimzi to generate a truststore. The resulting truststore will 
be used by the Strimzi operators, e.g. KafkaConnector, MirrorMaker2 when communicating with Kafka Connect.

## Affected/not affected projects

This proposal affects the `strimzi-kafka-operator` project.

## Compatibility

This proposal does not change the default behaviour of the Kafka Connect or MirrorMaker2 clusters so any existing clusters will not be affected.

## Rejected alternatives

### User provides the CA for the Kafka Connect REST API

In an earlier version of this proposal the user defined only the CA certificate and Strimzi used that to generate the actual certificate 
that Connect would host. The disadvantages of this proposal are Strimzi is forced to manage the lifecycle of the CA certificate and 
it does not allow users to use their own CA.

The advantage is that it would allow users to reuse the Strimzi cluster CA if they didn't want to manage their own CA.

This approach was rejected because the certificate management in Strimzi is not ideal, so picking an approach where the certificates 
are managed by the user is preferable. If the user does not want to manage their own certificates they can leave the Connect rest API 
as unencrypted.

### TLS encrypting the Kafka Connect REST API by default

[Proposal #8](008-tls-encrypt-the-kafka-connect-rest-api.md) proposed enabling a TLS encrypted listener be default. The problem with this approach is it is 
not clear what CA should be used to generate a self-signed certificate. The `KafkaConnect` CR does not provide an option to indicate the Strimzi cluster that 
it is connecting to. This is because the `KafkaConnect` CR and operator are often used with a Kafka cluster that isn't managed by Strimzi. Instead, the CR provides 
specific properties such as `bootstrapServers` to provide the listener address and `tls` to provide the certificate to trust.

### Annotation alternative to enable a plain HTTP REST API listener

Alternatively, a new annotation `strimzi.io/enable-plain-rest-listener` could be added, to avoid changing the spec.
Setting the `strimzi.io/enable-plain-rest-listener` annotation value to `true` would add the additional unencrypted HTTP listener on port 8083.
This was rejected as it is generally better to have configuration in the `spec`, and the proposed `restListeners` structure could be extended in future enhancements.
