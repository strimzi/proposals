# Support TLS/SSL on the HTTP interface 

This proposal adds an option to enable TLS encryption on HTTP interface so that requests from clients to the Kafka Bridge can be sent with encryption.

## Motivation

The connection between HTTP Bridge and Kafka cluster can be secured however, currently there is no support for securing the connections to HTTP Bridge from clients. The current recommendation is to use Network policies and firewalls, reverse proxies such as OAuth 2.0 or API gateways to secure the client connections. This requires users to make additional configurations but does not provide the best user experience.  

## Proposal

This proposal adds new configurations to enable SSL server and specify locations of certificate and key files in PEM format so that client connectionts to HTTP Bridge is encryped. The new configurations will be added to the current `http` prefixed configurations:
- http.ssl.enable
- http.ssl.keystore.location
- http.ssl.keystore.key.location
- http.ssl.protocol
- http.ssl.enabled.protocols

When `http.ssl.enable` is set to true, Kafka Bridge server would be started with ssl enabled and the key and certificate loaded from the locations defined by `http.ssl.keystore.location` and `http.ssl.keystore.key.location` configurations. If `http.port` configuration is not set, the server port will be set to `8443` by default. This will allow users to make HTTPS connections from their clients, and the server would not accept unsecured connections.

If `http.ssl.protocol` is configured, the protocol will be added to the current or default list of enabled protocols. If `http.ssl.enabled.protocols` is configured, the list will replace the default list of enabled secure transport protocols that Vertx sets.
 
When running KafkaBridge with Strimzi operator, users can choose to either provide these configurations via `spec.http` in their KafkaBridge CR or allow the operator to manage the certificate using the same CA for the Kafka cluster.

Users can enable SSL by setting the new field `spec.http.sslConfigurations`. Under this new field, they have to set `generateCertficiate` field that accepts boolean value and is set to false by default. If it is not set in the CR, that means the user chooses to provide their own certificate via Kubernetes secret, in which case, they have to set `certificateAndKey` to refer to the secret that contains the certificate and key. 

If `generateCertificate` is set true, the operator will issue the certificate using the same CA for their Kafka cluster. When operator is reconciling KafkaBridge resource, it is not aware of which Kafka cluster the HTTP Bridge is connected to. So if the Kafka cluster is running in a different namespace, therefore the CA secrets are not in the same namespace as the KafkaBridge resource, the operator needs to know where to fetch them. In this case users have to also set `caNamespace`. However, if their Kafka cluster and KafkaBridge resources are running in the same namespace, they don't need to set this field, as the operator would look for them in the namespace of the KafkaBridge resource by default.

Let's take a look at an example of an user providing their own certificate for the HTTP Bridge server:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 443
    sslConfigurations: #NEW FIELD
      certificateAndKey: 
        secretName: my-secret
        certificate: public.crt
        key: private.key
...
```
An example of allowing the operator to issue and manage the certificate where Kafka cluster and KafkaBridge resources are running in the same namespace:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 443
    sslConfigurations:
      generateCertficiate: true
...
```
An example of allowing the operator to issue and manage the certificate where Kafka cluster and KafkaBridge resources are running in different namespaces:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 443
    sslConfigurations:
      generateCertficiate: true
      caNamespace: kafka
...
```

Users can also configure the certificate validity days and renewal days instead of using CA's defaults by setting these optional fields:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  # HTTP configuration (required)
  http:
    port: 443
    sslConfigurations:
      generateCertficiate: true
      certificateValidityDays: 90 #CA default is 365
      certificateRenewalDays:  14 #CA default is 30
...
```

Users can also configure the secure transport protocols by setting these additional fields that are optional:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  http:
    port: 443
    sslConfigurations:
      sslProtocol: "TLSv1.3" #This will add the protocol to the default list of enabled protocols.
...
```

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 3
  bootstrapServers: <cluster_name>-cluster-kafka-bootstrap:9092
  # HTTP configuration (required)
  http:
    port: 443
    sslConfigurations:
      # This will override the default list of enabled protocols.
      # As of Vertx version 5.0.3, the default list is  "TLSv1, TLSv1.1, TLSv1.2, TLSv1.3".
      sslEnabledProtocols: "TLSv1.2,TLSv1.3" 
      
...
```

## Future Improvements

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
    port: 443
    sslConfigurations:
      ...
      trustedCertificates: 
        - secretName: my-cluster-cluster-cert
          pattern: "*.crt"
        - secretName: my-cluster-cluster-cert
          pattern: "*.crt"    
...
```
This is not in the scope of this proposal, as we will likely to support multiple authentication mechanisms in the future.

## Affected/not affected projects

strimzi-kafka-bridge
strimzi-cluster-operator

## Compatibility

## Rejected alternatives


