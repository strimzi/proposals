# Custom Authentication in Kafka Brokers 

This proposal focuses on supporting custom authentication in Kafka Brokers for SASL/mTLS mechanisms via the Strimzi operator.

## Current situation

Currently, numerous authentication methods for brokers are [supported](https://github.com/strimzi/strimzi-kafka-operator/tree/0.25.0/api/src/main/java/io/strimzi/api/kafka/model/authentication) in Strimzi, and allows the user to configure which one to use. In addition, in the case a custom *authorizer* needs to be used, this is also currently [supported](https://github.com/strimzi/strimzi-kafka-operator/blob/0.25.0/cluster-operator/src/main/java/io/strimzi/operator/cluster/model/KafkaBrokerConfigurationBuilder.java#L541-L549).

However, there is no current ability to specify custom authentication for Kafka Brokers, which is what this proposal will focus on.

## Motivation

Supporting custom authentication allows greater flexibility for users of Strimzi who may have bespoke/proprietary authentication requirements. Ideally, this would  increase adoption of the operator, as it would allow for other popular third-party authentication schemes.

In addition, the cost of supporting custom authentication should be minimal as well, seeing as custom authn/z usually requires the end-user to build their own images. As a result, the level of expertise for such a user will be higher than the average adopter. It should also be noted that, it’s on the end-user to ensure that their custom authentication workflow works with Strimzi; the operator itself is strictly responsible for pushing down the necessary config to the broker pods. 

One down-side to note is that this may not incentive users of custom auth/n to contribute back to Strimzi with the images they have built, along with providing documentation for authentication scheme they’re using. 

## Proposal

### Workflow

Add a CustomAuthenticationKafkaListener class, which would support the following properties.

```
listener.name.<listener-name>.<sasl-mechanism>.sasl.client.callback.handler.class=
listener.name.<listener-name>.<sasl-mechanism>.sasl.server.callback.handler.class=
listener.name.<listener-name>.<sasl-mechanism>.sasl.login.callback.handler.class=
listener.name.<listener-name>.<sasl-mechanism>.connections.max.reauth.ms=
listener.name.<listener-name>.<sasl-mechanism>.sasl.jaas.config=
listener.name.<listener-name>.sasl.enabled.mechanisms=
listener.name.<listener-name>.ssl.client.auth=
listener.security.protocol.map=
principal.builder.class=
```

- `<listener-name>` would be the name for a given listener. 
- `<sasl-mechanism>` would be the sasl mechanism which you wish to configure. Seeing as users can configure/build their own sasl mechanisms, all values will be allowed for this field, rather than having an allowlist of the standard sasl mechanisms. 

To walkthrough an actual example, let's try configuring custom authentication for an oauthbearer workflow, where we would like to set the following properties:

```
listener.name.<listener-name>.ssl.keystore.location=/tmp/kafka/cluster.keystore.p12
listener.name.<listener-name>.ssl.keystore.password=<password>
listener.name.<listener-name>.ssl.keystore.type=PKCS12
listener.name.<listener-name>.oauthbearer.sasl.client.callback.handler.class=client.class
listener.name.<listener-name>.oauthbearer.sasl.server.callback.handler.class=server.class
listener.name.<listener-name>.oauthbearer.sasl.login.callback.handler.class=login.class
listener.name.<listener-name>.oauthbearer.connections.max.reauth.ms=999999999
listener.name.<listener-name>.sasl.enabled.mechanisms=oauthbearer
listener.name.<listener-name>.oauthbearer.sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required ;
principal.builder.class=SimplePrincipal.class
listener.security.protocol.map=CONTROLPLANE-9090:SSL,REPLICATION-9091:SSL,TLS-9093:SASL_SSL
```

To achieve this, we'd simply add the following listener config to our Kafka custom resource.

```
spec:
  kafka:
    config:
      principal.builder.class: SimplePrincipal.class
    listeners:
      - name: custom-auth-listener
        port: 9093
        type: internal
        tls: true
        configuration:
          brokerCertChainAndKey:
            secretName: my-secret
            certificate: my-certifciate.crt
            key: my-key.key
        authentication:
          type: custom
          sasl: true
          listener-config:
            oauthbearer.sasl.client.callback.handler.class: client.class
            oauthbearer.sasl.server.callback.handler.class: server.class
            oauthbearer.sasl.login.callback.handler.class: login.class
            oauthbearer.connections.max.reauth.ms: 999999999
            sasl.enabled.mechanisms: oauthbearer
            oauthbearer.sasl.jaas.config: |
              org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required ;
          tlsTrustedCertificates: ...
          secrets:
            - name: example
```

Then, when constructing the broker config, we’ll perform the following tasks:

* `Principal`, if set, is set cluster-wide for all authentication methods. This is a limitation of Kafka, which only allows one principal to be specified for the entire cluster. If set, this override will always be used, even if another authentication type, such as OAuth, purposefully overrides this. 
* The protocol for this listener would be derived from the `tls` and `sasl`, which then would be appended to `listener.security.protocol.map`. For example, if `tls: true` and `sasl: true`, then the protocol will be `SASL_SSL`. 
* Each configuration entry under `listener-config` would be pre-appended with `listener.name.<listener-name>`. 
* `brokerCertChainAndKey` can be specified if a custom certificate wishes to be served from this listener. This automatically creates/shoves the specified certifciate into a keystore automatically for the user, as it's done for other authentication mechanisms today.
* `tlsTrustedCertificates` functions identically to OAuth’s setting. This is needed for workflows which will use custom mTLS, or mTLS + SASL workflows, as they may require to add additional certificate authorities. Similar to `brokerCertChainAndKey`, a truststore is generated automatically for the user, with their provided certifciate authorities. 
* `secrets` allows to specify a list of secrets to mount to the pod to the directory `/mnt/strimzi/custom-auth-secrets/<listener-name>`. This is needed for workflows which need additional credentials locally, such as GSSAPI (Kerberos). 
    
In addition, this would allow for even more complex setups, where it would be desired to have multiple SASL mechanisms enabled. 
    
For example, take the scenario where a user would like to support both scram-sha-512 and their own custom SASL mechanism:

```
listener.name.<listener-name>.ssl.keystore.location=/tmp/kafka/cluster.keystore.p12
listener.name.<listener-name>.ssl.keystore.type=PKCS12
listener.name.<listener-name>.custom-sasl-protocol.sasl.jaas.config=my.custom.loginmodule.CustomLoginModule required;
listener.name.<listener-name>.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required;
listener.name.<listener-name>.sasl.enabled.mechanisms=CUSTOM_SASL_PROTOCOL,SCRAM-SHA-512
principal.builder.class=SimplePrincipal.class
```

All then we need to do simply configure a listener in our Kafka custom resource, with the following properties:

```
    listeners:
      - name: custom-auth-listener
        port: 9093
        type: internal
        tls: true
        configuration:
          brokerCertChainAndKey:
            secretName: my-secret
            certificate: my-certifciate.crt
            key: my-key.key
        authentication:
          type: custom
          sasl: true
          principal.builder.class: SimplePrincipal.class
          listener-config:
            custom-sasl-protocol.sasl.jaas.config: my.custom.loginmodule.CustomLoginModule required;
            scram-sha-512.sasl.jaas.config: org.apache.kafka.common.security.scram.ScramLoginModule required;
            sasl.enabled.mechanisms: CUSTOM_SASL_PROTOCOL,SCRAM-SHA-512
```

### Additional considerations 

**Principal** being set is a dangerous setting which cannot be verified or guaranteed by integration tests that it will work until deployment time, as this class is set and provided by the user. 

It is expected that the provided principal builder *must* support Strimzi authentication (i.e. ssl-based auth). This is expected to be identical, or very similar to [Strimzi's OAuth library](https://github.com/strimzi/strimzi-kafka-oauth/blob/main/oauth-server/src/main/java/io/strimzi/kafka/oauth/server/OAuthKafkaPrincipalBuilder.java). In particular, it leverages [Kafka's default principal builder class](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/common/security/authenticator/DefaultKafkaPrincipalBuilder.java#L73-L79), which allows the building of a principal based upon the name of the of peer certificate. Thus, it is expected that the custom principal builder provides a principal of type "user" with the name being that of the ssl peer certificate.

An example of such a principal builder which satisfies Strimzi’s auth requirements:

```
public final class CustomKafkaPrincipalBuilder implements KafkaPrincipalBuilder {

    public KafkaPrincipalBuilder() {}

    @Override
    public KafkaPrincipal build(AuthenticationContext context) {
        if (context instanceof SslAuthenticationContext) {
            SSLSession sslSession = ((SslAuthenticationContext) context).session();
            try {
                return new KafkaPrincipal(
                        KafkaPrincipal.USER_TYPE, sslSession.getPeerPrincipal().getName());
            } catch (SSLPeerUnverifiedException e) {
                throw new IllegalArgumentException("Cannot use an unverified peer for authentication", e);
            }
        }
        
        // Create your own KafkaPrincipal here
        ...
    }
}
```

Another thing to be mindful of, is to ensure your **CustomAuthorizer** supports `super.users`, and the default KafkaPrincipal to ensure seamless integration with Strimzi. This is relevant as in the case of using a custom principal builder, you’re most likely using a CustomAuthorizer as well. 

### Testing strategy

Custom authentication has a delicate testing path due to the nature of the properties being used. Therefore, it’s best to incorporate an integration test alongside unit tests.

Unit tests: (*not an exhaustive list by any means*) 

* Ensure that multiple principals are not specified, and if so, throw user friendly exception.
* Validate that the protocol map being generated contains the proper custom auth prefix (i.e., SASL_SSL)
* All properties specified in `listener-config` are propagated through 

Integration test:

* Custom authentication is specified with Strimzi’s OAuth settings. This is ideal as all of the necessary files are already on the classpath, and this setup is known to work. 
    
    As mentioned prior, the operator should strictly be responsible for propagating the necessary config values (this test asserts that), while the end users are actually responsible that their custom authentication integrates seamlessly with Strimzi. 

## Affected/not affected projects

Affected: [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)

## Compatibility

Backwards compatibility shouldn’t be an issue, as current configuration should not affect custom authentication. However, users who try to have multiple auth types (i.e. OAuth and Custom) could run into issues if the custom auth requires `principal.builder.class` (as OAuth setting requires that this be set).

Forwards compatibility will need to be mindful of the custom authentication setting, and how to handle the case of multiple listeners/multiple auth types. Explicitly, understand that:  

* `listener.security.protocol.map` is partially built by the custom authentication setting -
* `principal.builder.class` can be set in custom auth, and therefore can only be set once
* listener-specific overrides occur here, and therefore `listener.name.<listener-name>.*` should not be overriden anywhere by other config settings

## Rejected alternatives

Regular OAuth type authentication (via StrimziOAuth) is currently not an option due to the requirement of needing custom callback handlers. This is due to requirements of our internal security system, which need to perform extra steps when handling calls. It’s also ideal to provide a custom principal builder as well, as to allow us to distinguish between sources of authentication (aka, internal system or ssl peer certificates). 
