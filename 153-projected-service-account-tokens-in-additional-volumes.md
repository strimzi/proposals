# Projected Service Account Tokens in Additional Volumes

This proposal suggests adding support for projected Service Account token volumes to the additional volumes section of Pod templates.
This would allow users to use the same short-lived token-based authentication mechanism that was approved for internal communication in [SEP-150](https://github.com/strimzi/proposals/blob/main/150-configurable-security-of-internal-communication.md).

## Current Situation

Strimzi currently supports three authentication mechanisms for user-defined Kafka listeners and Kafka clients in client-based operands (Kafka Connect, MirrorMaker 2, and Kafka Bridge):

* mTLS authentication based on TLS client certificates
* Authentication based on usernames and passwords (SASL SCRAM-SHA-512)
* OAuth authentication

Each of these mechanisms has its own advantages and disadvantages.
For example:
* mTLS requires TLS encryption and is therefore unsuitable for environments where the infrastructure provides TLS encryption, such as through Istio or another service mesh.
* SCRAM-SHA-512 requires the management and distribution of passwords.
  It also uses long-lived credentials.

OAuth authentication also has drawbacks.
It provides secure token-based authentication, but a common obstacle is that it requires an OAuth server to manage credentials and issue tokens.
For users who do not already have OAuth expertise or an OAuth server, this requirement is a major drawback.

Using OAuth-style authentication with Kubernetes Service Account tokens provides an interesting alternative because it does not require users to manage their own OAuth server.
This approach allows users to rely on existing Kubernetes features to issue tokens for Kubernetes Service Accounts and verify them using the Kubernetes signing keys.
Service Account tokens therefore provide secure authentication based on short-lived credentials with minimal effort, especially for internal Kafka listeners and Kafka clients running in the same Kubernetes cluster as the Kafka cluster.

## Motivation

Strimzi users can already use Service Account-based authentication.
They can configure it in the listener configuration of a `Kafka` custom resource:

```yaml
      - name: plain
        port: 9092
        type: internal
        tls: false
        authentication:
          type: custom
          sasl: true
          listenerConfig:
            sasl.enabled.mechanisms: OAUTHBEARER
            oauthbearer.sasl.server.callback.handler.class: io.strimzi.kafka.oauth.server.JaasServerOauthValidatorCallbackHandler
            oauthbearer.sasl.jaas.config: >-
                  org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required
                  unsecuredLoginStringClaim_sub="unused"
                  oauth.check.access.token.type="false"
                  oauth.valid.issuer.uri="https://kubernetes.default.svc.cluster.local"
                  oauth.jwks.endpoint.uri="https://kubernetes.default.svc.cluster.local/openid/v1/jwks"
                  oauth.username.claim="sub"
                  oauth.ssl.truststore.location="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
                  oauth.ssl.truststore.type="PEM"
                  oauth.server.bearer.token.location="/var/run/secrets/kubernetes.io/serviceaccount/token"
                  oauth.jwks.refresh.seconds="300"
                  oauth.include.accept.header="false";
```

They can also configure it for operands such as Kafka Connect:

```yaml
  authentication:
    type: custom
    sasl: true
    config:
      sasl.mechanism: OAUTHBEARER
      sasl.jaas.config: >-
            org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required
            oauth.access.token.location="/var/run/secrets/kubernetes.io/serviceaccount/token";
      sasl.login.callback.handler.class: io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
```

However, client-based operands can use only the default Service Account token, which is mounted at `/var/run/secrets/kubernetes.io/serviceaccount/` and uses the default audience.
They cannot use a custom audience.
Without a custom audience, regular Service Account tokens available to other applications can also be used to authenticate to the Apache Kafka cluster.
This makes the authentication less secure.

To allow users to configure a custom audience, client-based operands need to support projected Service Account tokens for which users can specify an audience.
Users can then enforce the audience as part of Kafka broker authentication (`oauth.custom.claim.check="@.aud anyof ['my-internal-listener']"`):

```yaml
      - name: plain
        port: 9092
        type: internal
        tls: false
        authentication:
          type: custom
          sasl: true
          listenerConfig:
            sasl.enabled.mechanisms: OAUTHBEARER
            oauthbearer.sasl.server.callback.handler.class: io.strimzi.kafka.oauth.server.JaasServerOauthValidatorCallbackHandler
            oauthbearer.sasl.jaas.config: >-
                  org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required
                  unsecuredLoginStringClaim_sub="unused"
                  oauth.check.access.token.type="false"
                  oauth.custom.claim.check="@.aud anyof ['my-internal-listener']" # <=== Audience check
                  oauth.valid.issuer.uri="https://kubernetes.default.svc.cluster.local"
                  oauth.jwks.endpoint.uri="https://kubernetes.default.svc.cluster.local/openid/v1/jwks"
                  oauth.username.claim="sub"
                  oauth.ssl.truststore.location="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
                  oauth.ssl.truststore.type="PEM"
                  oauth.server.bearer.token.location="/var/run/secrets/kubernetes.io/serviceaccount/token"
                  oauth.jwks.refresh.seconds="300"
                  oauth.include.accept.header="false";
```

Once the audience is enforced, only tokens with the correct audience can connect.
Regular Kubernetes Service Account tokens that are available in many Pods for communication with the Kubernetes API would then fail authentication.

This proposal suggests adding support for Service Account token projections to additional volumes.
This will allow users to configure token audiences in client-based operands and enforce those audiences in the Kafka configuration.

_Note: [SEP-150](https://github.com/strimzi/proposals/blob/main/150-configurable-security-of-internal-communication.md) uses the same mechanism to provide Service Account-based authentication for internal cluster communication._

## Proposal

Kubernetes [projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/) allow several source types to be mapped into a single volume:

* Secrets
* ConfigMaps
* Downward API data
* Service Account tokens
* Cluster trust bundles
* Pod certificates

This proposal adds support only for Service Account token projections.
It does not add support for the other sources.
However, it follows the existing Kubernetes API structures so that support for the other sources can be added in the future if needed.

Projected volumes will be supported in the additional volumes section of the Pod template.
For example, users will be able to configure `KafkaConnect.spec.template.pod.volumes` as follows:

```yaml
spec:
  template:
    pod:
      volumes:
        - name: auth-token
          projected:
            sources:
              - serviceAccountToken:
                  audience: my-internal-listener
                  expirationSeconds: 3600
                  path: token
```

Other source types could be added to the `sources` list in the future.

Once a projected volume is added to the Pod, it can be mounted like any other volume.
Therefore, no changes will be needed to how volume mounts are configured for additional volumes.

### Implementation Details

The new API classes will be added to the `io.strimzi.api.kafka.model.common.template` package in the `api` module.
The first two levels, `projected` and `sources`, will be represented by dedicated Strimzi API classes, giving Strimzi control over the supported volume sources.
At this point, the API classes will support only Service Account token projections.
The `serviceAccountToken` level will use the existing Fabric8 class because there is no reason to create a dedicated Strimzi class.

```yaml
      volumes:
        - name: auth-token
          projected:                       # Strimzi class `ProjectedVolume`
            sources:                       # Strimzi class `ProjectedVolumeSource`
              - serviceAccountToken:       # Fabric8 class `ServiceAccountTokenProjection`
                  # ...
```

The existing `TemplateUtils.createVolumeFromConfig` method will be updated to handle projected volumes alongside the existing volume types.

#### Security Impact

A Service Account token projection mounts a token for the Service Account assigned to the Pod.
It cannot be used to get a token for another Service Account.
Therefore, this feature would allow Strimzi users with permission to manage Strimzi custom resources to mount tokens with different audiences for the Pod's Service Account.
However, it would not allow them to access tokens for other Service Accounts.

## Affected Projects

This proposal affects only the Strimzi Cluster Operator.

## Backwards Compatibility

This proposal is fully backward compatible.
The other volume types will continue to work without changes.

## Rejected Alternatives

There are no rejected alternatives.
