# Add support for TLS/SSL on the HTTP interface 

This proposal adds an option to enable TLS encryption on HTTP interface so that requests from clients to the Kafka Bridge can be sent with encryption.

## Current situation

Strimzi does not support TLS/SSL for HTTP Bridge, so client connections to HTTP Bridge are not encrypted at all. There is a single listener for all the endpoints that are used for Kafka operations as well as admin operations such as healthcheck and monitoring and it runs on port 8080 by default.

## Motivation

The connection between HTTP Bridge and Kafka cluster can be secured however, currently there is no support for securing the connections to HTTP Bridge from clients. The current recommendation is to use firewalls or API gateways to secure the client connections. This requires users to make additional configurations and does not provide the best user experience. 

Moreover, some of the endpoints are for internal use only, such as `/healthy`, `/ready` and `/metrics` but they are exposed on the same listener and the same port as the other endpoints that are used by external clients. This means the endpoints for internal use only are also exposed to external clients. And if SSL is enabled, it will introduce more operational complexity as these internal endpoints would need to be configured with TLS and certificates.

## Proposal
This proposal also adds new configurations to enable SSL server and load keystore certificate and key in PEM format so that client connections to HTTP Bridge can be encrypted. The new configurations will be added to the current `http` prefixed configurations:
- http.ssl.enable
- http.ssl.keystore.certificate.location
- http.ssl.keystore.key.location
- http.ssl.keystore.certificate.chain
- http.ssl.keystore.key
- http.ssl.enabled.protocols
- http.ssl.enabled.cipher.suites
- http.management.port

When `http.ssl.enable` is set to true, HTTP Bridge server will be started with SSL enabled and the new configurations will allow users to define locations of the keystore files or certificate chain and key in PEM format. In the future, we can support more formats such as `PKCS12` and `JKS`, if we see requirements from users. 

`http.ssl.enabled.protocols` can be configured with a comma separated list of enabled secure transport protocols that the server will accept from connecting clients. If not set, the server will use the same list of protocols that [Kafka use by default](https://kafka.apache.org/documentation/#brokerconfigs_ssl.enabled.protocols), which is `TLSv1.2,TLSv1.3`. 

`https.ssl.enabled.cipher.suites` can be configured with a comma separated list of cipher suites that the server will support. If not set, the default list of cipher suites provided by the underlying JDK SSL/TLS engine will be used.

If the existing `http.port` is not set and `http.ssl.enable` is set to true, the server port will be set to `443` by default. This will allow users to make TLS encrypted connections from their clients for all the HTTP Bridge endpoints for client operations.

This proposal also adds a separate listener for the endpoints that are used by internal clients such as
- `/healthy`
- `/ready`
- `/metrics`

The port for this listener will be set by the new configuration `http.management.port` or to 8081 by default. These endpoints will no longer be available on the existing default port defined by `http.port` or 8080. 
The rest of the endpoints will stay on the existing listener with the default port 8080 and 443 if SSL is enabled. 

This brings a clear separation of internal and external use for the endpoints and makes it simpler to enable SSL for the listener that is used by external clients. It also brings a better performance and security because admin type of operations such as collecting metrics are isolated from traffic for Kafka operations.

Once SSL is enabled, HTTP Bridge server will no longer accept unencrypted connections from external clients. If user wants to allow both HTTP and HTTPS connections, the recommendation would be to deploy 2 separate HTTP Bridge instances, one is enabled with SSL and the other one is not. That way external clients can make both encrypted and unencrypted connections for Kafka operations.


### HTTP Bridge managed by Strimzi operator

When deploying KafkaBridge custom resources with the Strimzi operator, users can set these configurations via `spec.http` in their KafkaBridge CR. Users can enable SSL by setting the new field `spec.http.sslEnable` which is set to false by default. When it is set to true, users have to provide the certificate and key in PEM format via Kubernetes secret that is referenced in the `certificateAndKey` field.

Let's take a look at an example of KafkaBridge CR with the new fields to enable SSL:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 8443
    sslEnable: true #NEW FIELD
    certificateAndKey: #NEW FIELD
      secretName: my-secret
      certificate: public.crt
      key: private.key
...
```
The liveness and readiness probes will connect to port 8081 (instead `http.port` or 8080). If users have metrics set up, they would need to use port 8081 to connect to the `metrics` endpoint and cannot reconfigure this port number as it's only for internal use.

An example of KafkaBridge CR with other possible SSL configurations:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 8443
    sslEnable: true 
    certificateAndKey: 
      secretName: my-secret
      certificate: public.crt
      key: private.key
    config: #NEW FIELD
      ssl.enabled.protocols: "TLSv1.1, "TLSv1.2", "TLSv1.3"
      ssl.enabled.cipher.suites: "TLS_DHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
...
```

## Future Improvements

### Support more formats for the keystore
We can support formats such as `PKSC12` and `JKS` as they are common for Java applications. If we support these formats, then we would need to add the following configurations:
- http.ssl.keystore.type
- http.ssl.keystore.password

### Support mTLS
In the future, we can support mutual TLS for the client connections to HTTP Bridge as well. This could be implemented by adding a field `trustedCertificates`, for example:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 8443
    sslEnable: true
    certificateAndKey:
      secretName: my-secret
      certificate: public.crt
      key: private.key
    trustedCertificates: 
      - secretName: my-cluster-cluster-cert
        pattern: "*.crt"
      - secretName: my-cluster-cluster-cert
        pattern: "*.crt"    
...
```
This is not in the scope of this proposal, as we will likely to support multiple authentication mechanisms in the future.

In the future, we are also likely to implement authentication and authorization for HTTP Bridge. By having separate listeners, the internal endpoints will not be affected by such changes and will make it easier to implement them.

### Automating certificate management

Users can choose to allow the operator to manage the certificate instead of providing their own certificate and key. This can be implemented by adding another boolean field on the KafkaBridge CR such as `generateCertificate`. The operator then could issue the certificate using the same CA for their Kafka cluster as we do today for internal components such as Cruise Control and Kafka Exporter, however with the upcoming support for cert-manager, the way the operator manages certificates will change. This is the reason we are leaving this feature for future improvement.

## Affected/not affected projects

- strimzi-kafka-bridge
- strimzi-cluster-operator

## Compatibility

This will introduce a breaking change for users who collect metrics on port 8080 or the port defined by `http.port` configuration. Also users who run Bridge on baremetals and do their own manual healtchecks using `/healthy` and `/ready` endpoints will be affected as well. These users have to reconfigure their healthcheck and monitoring system to use port 8081 or reconfigure their Bridge server with `http.management.port`.

## Rejected alternatives

### Automating certificate management as part of this proposal

- Currently the operator does not manage certificates for other custom resources, as they are designed to be independent, it only manages certificates for its internal components such as Cruise Control and Kafka Exporter, that are deployed as part of the Kafka custom resource. This will introduce a new behaviour to the operator and would require taking other custom resources such as KafkaConnect into consideration as well, because we would potentially need to manage certificates for it too. We are currently in the process of refactoring how the operator manages certificates to support external certificate manager as proposed [here](https://github.com/strimzi/proposals/pull/135), therefore it is not the right time to introduce such new behaviour, as it would not be future proof. Instead, it was decided to leave it for future improvement. 

- Continue using a single interface and only use separate listeners when SSL is enabled so that we don't introduce a breaking change to existing users. Given that we will be introducing the first major version of Bridge, introducing a breaking change is acceptable and it makes much more sense to have separate listeners for better separation of concerns regardless of TLS. In the future, we are likely to implement authentication and authorization for HTTP Bridge. By having separate listeners, th admin endpoints will not be affected and implementation will be simpler. 
