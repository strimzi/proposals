# Deprecate and remove `type: oauth` authentication and `type: keycloak` authorization

Strimzi has its own library for using [OAuth authentication](https://github.com/strimzi/strimzi-kafka-oauth) in Apache Kafka brokers and clients.
It also provides a custom authorizer based on OAuth authentication and Keycloak Authorization Services.
This library (both authentication and authorization) is bundled with Strimzi and our custom resources have an extensive API for configuring it.

This proposal suggests deprecating the `type: oauth` authentication and `type: keycloak` authorization from the Strimzi API and removing them from the [Strimzi `v1` CRD API](https://github.com/strimzi/proposals/pull/174).
The APIs will be replaced by `type: custom` authentication and authorization.

This proposal does not include deprecating the Strimzi OAuth library subproject or removing it from the Strimzi container images.
Its development and maintenance will continue as before.

## Motivation

The `type: oauth` authentication and `type: keycloak` authorization have a very extensive API with lots of different options.
These options have various interdependencies, where different fields should or should not be used with other fields.
However, they have only very weak validation.
So we typically just receive these options through the CRD API and pass them to the configuration in the various Kafka components.

These options need to be also processed in our source code.
We also need to prepare the certificates and Secrets.
Given the large number of options, this constitutes a sizable amount of code.
We also need to test all of these options — and that they are correctly processed — in our unit, integration, and system tests.
We must also document all the API options and their use in our documentation.

The documentation and tests are to a large extent duplicated between the OAuth library itself and the Strimzi Operators.
This is because the OAuth library is intended as a separate project.
So it has its own documentation and tests.

As a result, this seems to create a lot of maintenance effort without too much value gained out of it.
Reducing this effort is the main motivation for this proposal.

We already did the same to the Open Policy Agent authorization which was deprecated in [Strimzi Proposal #97](https://github.com/strimzi/proposals/blob/main/097-deprecate-OPA-authorization.md) and will be also removed in the `v1` CRD API.

## Proposal

This proposal recommends deprecating `type: oauth` authentication and `type: keycloak` authorization.
Deprecation is targeted for Strimzi 0.49, with support remaining while the `v1beta2` API is in use.
These APIs will not be present in the `v1` CRD API.
And the code will be removed from the Cluster Operator after we drop support for `v1beta2` API.

As a replacement, users can migrate to using `type: custom` authentication and authorization for OAuth configuration.
The `type: custom` API is supported for:
* Apache Kafka brokers server-authentication
* Apache Kafka brokers authorization
* Client-authentication in client based components such as Connect, MirrorMaker2, or Bridge

Custom authentication and authorization allow users to configure any authentication and authorization plugins.
In this case, Strimzi does not have detailed knowledge of the plugin being configured.
Therefore, configuration is not spread across multiple fields from which the operator constructs the Kafka configuration.
Instead, users configure the `sasl.jaas.config` directly in the Strimzi custom resources.
There is no validation of the configuration.
But as mentioned in the motivation section, that is similar to what we already have today with the existing APIs.

To configure sensitive options, such as TLS certificates, OAuth secrets, password, etc., users can use the existing template feature:
* Use the container template to define environment variables based on Secret and use configuration providers to load the sensitive data
* Mount the sensitive data from Secrets using the _additional volumes_ and reference their location in the container

The OAuth library already supports using PEM files directly or reference certificates inline to make it easier to use the data from the Secrets directly without the need for some preprocessing done by the Strimzi operator (such as converting PEM certificate to PKCS12 file).

The OAuth library will remain bundled with Strimzi container images even after the dedicated APIs are removed.
That way, users do not need to add the library manually by building new container images etc.

### Configuration examples

The following examples are included to give a better idea of what this change will mean for our users.

#### Server authentication in Apache Kafka brokers

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: custom
          sasl: true
          listenerConfig:
            sasl.enabled.mechanisms: OAUTHBEARER
            oauthbearer.sasl.server.callback.handler.class: io.strimzi.kafka.oauth.server.JaasServerOauthValidatorCallbackHandler
            oauthbearer.sasl.jaas.config: | 
                org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required unsecuredLoginStringClaim_sub="thePrincipalName" oauth.valid.issuer.uri="http://valid-issuer" oauth.jwks.endpoint.uri="http://jwks" oauth.jwks.expiry.seconds="500" oauth.jwks.refresh.seconds="400" oauth.username.claim="preferred_username" oauth.enable.metrics="true" oauth.ssl.truststore.location="/mnt/oauth-certs/tls.crt" oauth.ssl.truststore.type="PEM";
            principal.builder.class: io.strimzi.kafka.oauth.server.OAuthKafkaPrincipalBuilder
    config:
      # ...
    template:
      pod:
        volumes:
          - name: oauth-certs
            secret:
              name: oauth-secret
      kafkaContainer:
        volumeMounts:
          - name: oauth-certs
            mountPath: /mnt/oauth-certs
  # ...
```

#### Keycloak authorization in Apache Kafka brokers

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    authorization:
      type: custom
      authorizerClass: io.strimzi.kafka.oauth.server.authorizer.KeycloakAuthorizer
      superUsers:
        - CN=user-1
        - user-2
        - CN=user-3
    config:
      # ...
      principal.builder.class: io.strimzi.kafka.oauth.server.OAuthKafkaPrincipalBuilder
      strimzi.authorization.client.id: kafka
      strimzi.authorization.token.endpoint.uri: http://token
      strimzi.authorization.ssl.endpoint.identification.algorithm: ""
      strimzi.authorization.delegate.to.kafka.acl: "false"
      strimzi.authorization.ssl.truststore.location: /mnt/keycloak-certs/tls.crt
      strimzi.authorization.ssl.truststore.type: PEM
    template:
      pod:
        volumes:
          - name: keycloak-certs
            secret:
              name: keycloak-secret
      kafkaContainer:
        volumeMounts:
          - name: keycloak-certs
            mountPath: /mnt/keycloak-certs
  # ...
```

#### Client authentication in client-based components

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect
spec:
  # ...
  authentication:
    type: custom
    sasl: true
    config: 
      sasl.login.callback.handler.class: io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
      sasl.mechanism: OAUTHBEARER
      sasl.jaas.config: |
          org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required oauth.token.endpoint.uri="http://token" oauth.client.id=\"oauth-client-id\" oauth.client.secret="${strimzienv:OAUTH_CLIENT_SECRET}";
  template:
    connectContainer:
      env:
        - name: OAUTH_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: oauth-secret
              key: oauth-client-secret
  # ...
```

### Documentation

The examples of using the `type: custom` configuration will be added to the documentation.
The existing documentation using the OAuth APIs should be removed after its deprecation, while ensuring the things it covers are well documented in the OAuth library documentation/README.
We will also include basic instructions for how to migrate to the `type: custom` APIs, which will link to the Strimzi OAuth library docs for details about the different options.
The migration instructions will also include a mapping between the API options and the actual OAuth library configuration options.

### Examples

Existing OAuth examples in the Strimzi Operators repo will be updated to use the `type: custom` authentication and authorization.

### System tests

Basic tests of OAuth tests using the `type: custom` authorization should be added to the Strimzi Operators system tests.
After OAuth APIs are deprecated, we will start removing some of the related system tests.
But at least some of the basic tests will remain until the OAuth API is completely removed to ensure its functionality.

## Backwards compatibility

This proposal breaks backwards compatibility and forces users to change their custom resources at the latest when migrating to the `v1` CRD API version.

## Rejected alternatives

There are currently no rejected alternatives.