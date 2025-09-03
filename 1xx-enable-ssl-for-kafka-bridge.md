# Support TLS/SSL on the HTTP interface 

This proposal adds an option to enable TLS encryption on HTTP interface so that requests from clients to the Kafka Bridge can be sent with encryption.

## Current situation

Strimzi does not support TLS/SSL for HTTP Bridge, so clients connections to HTTP Bridge is not encrypted at all.

## Motivation

The connection between HTTP Bridge and Kafka cluster can be secured however, currently there is no support for securing the connections to HTTP Bridge from clients. The current recommendation is to use firewalls or API gateways to secure the client connections. This requires users to make additional configurations and does not provide the best user experience.  

## Proposal

This proposal adds new configurations to enable SSL server and specify locations of certificate and key files in PEM format so that client connections to HTTP Bridge are encrypted. The new configurations will be added to the current `http` prefixed configurations:
- http.ssl.enable
- http.ssl.port
- http.ssl.keystore.location
- http.ssl.keystore.key.location
- http.ssl.enabled.protocols
- http.ssl.enabled.cipher.suites

When `http.ssl.enable` is set to true, HTTP Bridge server will be started with SSL enabled and the key and certificate loaded from the locations defined by `http.ssl.keystore.location` and `http.ssl.keystore.key.location` configurations. If `http.ssl.port` configuration is not set, the server port will be set to `8443` by default. This will allow users to make TLS encrypted connections from their clients for all the HTTP Bridge endpoints for client operations. However, connections to the following endpoints for monitoring will stay unencrypted:
- `/healthy`
- `/ready`
- `/metrics`
- `/openapi`
- `/openapiv2`
- `/openapiv3`
- `/info`

When `http.ssl.enable` is set, these endpoints will have a separate listener, that is always PLAINTEXT. It will use the port defined by the existing `http.port` that is set to `8080` by default. The purpose of these endpoints is to be exposed only internally for the health checks, getting OpenAPI specification and metrics collection. In the future, we are likely to implement authentication and authorization for HTTP Bridge. By having separate listeners, these monitoring endpoints will not be affected by such changes.

`http.ssl.enabled.protocols` can be configured with a comma separated list of enabled secure transport protocols that the server will accept from connecting clients. If not set, the server will use the [default list](https://vertx.io/docs/apidocs/io/vertx/core/net/SSLOptions.html#DEFAULT_ENABLED_SECURE_TRANSPORT_PROTOCOLS) Vertx sets. 

`http.ssl.enabled.cipher.suites` can be configured with a comma separated list of cipher suites that server will support. If not set, Vertx uses the default cipher suites provided by the underlying JDK SSL/TLS engine.

### HTTP Bridge managed by Strimzi operator

When deploying KafkaBridge custom resources with the Strimzi operator, users can set these configurations via `spec.http` in their KafkaBridge CR. Users can enable SSL by setting the new field `spec.http.sslEnable` which is set to false by default. When it is set to true, users have to provide the certificate and key via Kubernetes secret that is referenced in the `certificateAndKey` field.

Let's take a look at an example of KafkaBridge CR with the new fields:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 8080
    sslPort: 443 #NEW FIELD
    sslEnable: true #NEW FIELD
    certificateAndKey: #NEW FIELD
      secretName: my-secret
      certificate: public.crt
      key: private.key
...
```
The liveness and readiness probes will continue to check on the existing port, that is defined by `port` field (or 8080 if not set).  

## Future Improvements

### Supporting mTLS
In the future, we can support mutual TLS for the client connections to HTTP Bridge as well. This could be implemented by adding following fields under the `spec.http.sslConfigurations`:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    sslPort: 443
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

### Automating certificate management

Users can choose to allow the operator to manage the certificate instead of providing their own certificate and key. This can be implemented by adding another boolean field on the KafkaBridge CR such as `generateCertificate`. The operator then could issue the certificate using the same CA for their Kafka cluster as we do today for internal components such as Cruise Control and Kafka Exporter, however with the upcoming support for cert-manager, the way the operator manages certificates will change. This is the reason we are leaving this feature for future improvement.

## Affected/not affected projects

- strimzi-kafka-bridge
- strimzi-cluster-operator

## Compatibility

By default, HTTP Bridge will operate as before without encryption and have a single interface with default port of 8080, therefore there shouldn't be any compatibility issue.
When SSL is enabled, only then there will be separate interfaces for endpoints for client operations and endpoints for monitoring purpose. The `http.port` will still be used for the endpoints for monitoring purpose therefore should not affect the existing users that are collecting metrics and doing healthcheck on this port.

## Rejected alternatives

### Automating certificate management as part of this proposal

Currently the operator does not manage certificates for other custom resources, as they are designed to be independent, it only manages certificates for its internal components such as Cruise Control and Kafka Exporter, that are deployed as part of the Kafka custom resource. This will introduce a new behaviour to the operator and would require taking other custom resources such as KafkaConnect into consideration as well, because we would potentially need to manage certificates for it too. We are currently in the process of refactoring how the operator manages certificates to support external certificate manager as proposed [here](https://github.com/strimzi/proposals/pull/135), therefore it is not the right time to introduce such new behaviour, as it would not be future proof. Instead, it was decided to leave it for future improvement. 