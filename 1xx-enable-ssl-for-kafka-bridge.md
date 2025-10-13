# Add support for TLS/SSL on the HTTP interface 

This proposal recommends adding an option that can enable TLS encryption on HTTP interface so that requests from clients to the Kafka Bridge can be sent with encryption.

## Current situation

Strimzi does not support TLS/SSL for the HTTP Bridge, so client connections to the HTTP Bridge are unencrypted. A single listener serves all endpoints for Kafka operations and admin operations such as health checks and monitoring. The default port is 8080.

## Motivation

The connection between HTTP Bridge and Kafka cluster can be secured. However, there is currently no support for securing client connections to the HTTP Bridge. The current recommendation is to use firewalls or API gateways to secure the client connections. This requires users to make additional configurations and does not provide the best user experience. 

Moreover, some of the endpoints are for internal use only, such as `/healthy`, `/ready` and `/metrics` but they are exposed on the same listener and the same port as the other endpoints that are used by external clients. This means the endpoints for internal use only are also exposed to external clients. And if SSL is enabled, it will introduce more operational complexity as these internal endpoints would need to be configured with TLS and certificates.

## Proposal

This proposal introduces adds new configurations to enable an SSL server and load a keystore certificate and key in PEM format so that client connections to HTTP Bridge can be encrypted. The new configurations will be added to the existing `http` prefixed configurations:
- http.ssl.enable
- http.ssl.keystore.certificate.location
- http.ssl.keystore.key.location
- http.ssl.keystore.certificate.chain
- http.ssl.keystore.key
- http.ssl.enabled.protocols
- http.ssl.enabled.cipher.suites
- http.management.port

When `http.ssl.enable` is set to `true`, the HTTP Bridge server starts with SSL enabled and the new configurations will allow users to define locations of the keystore files or certificate chain and key in `PEM` format. Additional formats such as `PKCS12` and `JKS` might be supported in the future if users request them.

`http.ssl.enabled.protocols` can be configured with a comma separated list of enabled secure transport protocols that the server will accept from connecting clients. If not set by the user, the server will use `TLSv1.2,TLSv1.3` as the default. 

`http.ssl.enabled.cipher.suites` can be configured with a comma separated list of cipher suites that the server will support. If not set, the default list of cipher suites provided by the underlying JDK SSL/TLS engine will be used.

If the existing `http.port` is not set and `http.ssl.enable` is set to `true`, the server port will be set to `443` by default. This will allow users to make TLS encrypted connections from their clients for all the HTTP Bridge endpoints for client operations.

This proposal also adds support for a separate listener for internal endpoints such as:
- `/healthy`
- `/ready`
- `/metrics`

The internal endpoints will use the port defined by the new configuration property `http.management.port`, defaulting to 8081 if unspecified. These endpoints will no longer be accessible via the default listener port defined by `http.port` or port 8080. All other endpoints will continue to use the existing listener on port 8080, or 443 if SSL is enabled.

This brings a clear separation of internal and external use for the endpoints and makes it simpler to enable SSL for the listener that is used by external clients. It should also improve performance and security because admin operations, such as collecting metrics, are isolated from traffic for Kafka operations.

Once SSL is enabled, HTTP Bridge server will reject unencrypted connections from external clients. To support both HTTP and HTTPS connections, the recommendation would be to deploy two separate HTTP Bridge instances: one with SSL enabled, and one without. That way external clients can make both encrypted and unencrypted connections for Kafka operations.

### HTTP Bridge managed by Strimzi operator

When deploying `KafkaBridge` custom resources with the Strimzi operator, users can set these configurations in the `spec.http` section of their `KafkaBridge` CR. Users can enable SSL by setting the new field `spec.http.sslEnable`, which is set to `false` by default. When it is set to `true`, users have to provide the certificate and key in PEM format using a Kubernetes `Secret` that is referenced in the `certificateAndKey` field.

Let's take a look at an example of the `KafkaBridge` CR with the new fields to enable SSL:
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
    tls: #NEW FIELD
      serverCertChainAndKey: #NEW FIELD
        secretName: my-secret
        certificate: public.crt
        key: private.key
...
```
The liveness and readiness probes will connect to port 8081 (instead of `http.port` or 8080). If users have metrics set up, they would need to use port 8081 to connect to the `metrics` endpoint and cannot reconfigure this port number as it's only for internal use.

An example of the `KafkaBridge` CR with other possible SSL configurations:
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
    tls: #NEW FIELD
      serverCertChainAndKey: #NEW FIELD
        secretName: my-secret
        certificate: public.crt
        key: private.key
      config: #NEW FIELD
        ssl.enabled.protocols: [TLSv1.1, TLSv1.2, TLSv1.3]
        ssl.enabled.cipher.suites: "TLS_DHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
...
```

### Documentation
(Securing the Kafka Bridge HTTP interface)[https://strimzi.io/docs/bridge/latest/#con-securing-http-interface-bridge] will be updated with an example of how to enable SSL and confugure it.
[Deploying Kafka Bridge](https://strimzi.io/docs/operators/latest/deploying#kafka-bridge-str) will also be updated with the new API options and an example of `KafkaBridge` custom resource configuration for enabling SSL.

### Testing
New integration tests will be added to `strimzi-kafka-bridge`, to test the new configurations. 
New system tests will be added to `strimzi-kafka-operator` to deploy HTTP Bridge with SSL enabled and ensure client connections work as expected. 

## Future Improvements

### Support more formats for the keystore
We can support formats such as `PKCS12` and `JKS` as they are common for Java applications. If we support these formats, then we would need to add the following configurations:
- http.ssl.keystore.type
- http.ssl.keystore.password

### Support mTLS
In the future, we can support mutual TLS for the client connections to HTTP Bridge as well. This could be implemented by adding a new `trustedCertificates` property in the `tls` section, for example:

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
    tls: 
      serverCertChainAndKey: 
        secretName: my-secret
        certificate: public.crt
        key: private.key
      trustedCertificates: #NEW FIELD
        - secretName: my-cluster-cluster-cert
          pattern: "*.crt"
        - secretName: my-cluster-cluster-cert
          pattern: "*.crt"    
...
```
This is not in the scope of this proposal, but we are likely to add support for mutual TLS with multiple authentication mechanisms in the future.

We also plan to implement authentication and authorization for the HTTP Bridge. By using separate listeners, internal endpoints are isolated from these changes, making implementation simpler and less disruptive.

### Automating certificate management

In the future, users might want the operator to manage the certificate instead of providing their own certificate and key. This could be implemented by adding another boolean field on the `KafkaBridge` CR such as `generateCertificate`. The operator could then issue certificates using the same CA for their Kafka cluster, as is currently done for internal components like Cruise Control and Kafka Exporter. However, with upcoming support for cert-manager, certificate management is expected to change. For this reason, we are deferring this feature for implementation in the future.

## Affected/not affected projects

- strimzi-kafka-bridge
- strimzi-cluster-operator

## Compatibility

This will introduce a breaking change for users who collect metrics on port 8080 or the port defined by `http.port` configuration. Also users who run the HTTP Bridge on bare metal and do their own manual health checks using `/healthy` and `/ready` endpoints will be affected as well. These users must reconfigure their health check and monitoring system to use port 8081 or reconfigure their Bridge server with `http.management.port`.

## Rejected alternatives

### Automating certificate management as part of this proposal
Currently, the Cluster Operator does not manage certificates for other custom resources, as they are designed to operate independently. It only handles certificates for internal components, such as Cruise Control and Kafka Exporter, which are deployed as part of the `Kafka` custom resource. Extending certificate management to other resources, like `KafkaConnect`, would introduce new behavior and require broader changes. Since we’re looking at refactoring certificate handling to support external managers (see the proposal for the [External Certificate Manager](https://github.com/strimzi/proposals/pull/135)), now is not the right time to introduce this change. To ensure future compatibility, we’ve deferred it as a potential improvement.

### Continue using a single interface when SSL is not enabled
Use separate listeners only when SSL is enabled so that we don't introduce a breaking change to existing users. Given that we will be introducing the first major version of Bridge, introducing a breaking change is acceptable and it makes much more sense to have separate listeners for better separation of concerns, whether or not TLS is enabled. In the future, we are likely to implement authentication and authorization for HTTP Bridge. By having separate listeners, the admin endpoints will not be affected and implementation will be simpler.