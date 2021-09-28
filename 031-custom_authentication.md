# Custom Authentication

This proposal focuses on supporting custom authentication in the Strimzi operator.

## Current situation

Currently, numerous authentication methods are [supported](https://github.com/strimzi/strimzi-kafka-operator/tree/0.25.0/api/src/main/java/io/strimzi/api/kafka/model/authentication) in Strimzi, and allows the user to configure which one to use. In addition, in the case a custom *authorizer* needs to be used, this is also currently [supported](https://github.com/strimzi/strimzi-kafka-operator/blob/0.25.0/cluster-operator/src/main/java/io/strimzi/operator/cluster/model/KafkaBrokerConfigurationBuilder.java#L541-L549). 

However, there is no ability to specify custom authentication, which is what this proposal will focus on.

## Motivation

Supporting custom authentication allows greater flexibility for users of Strimzi who may have bespoke/proprietary authentication requirements. This would greatly broaden user adoption of the operator, as it would also support more common authentication schemes, such as [Amazon MSK](https://docs.aws.amazon.com/msk/latest/developerguide/security_iam_service-with-iam.html), and attract users who have much more stringent requirements (i.e. proprietary auth). 

The “cost” of supporting such feature should be minimal as well, as the expectation for custom authentication passes the complexity onto the user. Strimzi should strictly support the passing of configuration values, and nothing else. It’s up to the user to ensure their authn/z schemes can be integrated with Strimzi. Considering this also requires custom built Strimzi Kafka images in order to load custom classes on to the classpath, it’s also expected that will only really apply to highly technical/pro users.

## Proposal

Add a CustomAuthenticationKafkaListener class, which supports an example of the following properties

```
listener.name.<listener-name>.oauthbearer.sasl.client.callback.handler.class=
listener.name.<listener-name>.oauthbearer.sasl.server.callback.handler.class=
listener.name.<listener-name>.oauthbearer.sasl.login.callback.handler.class=
listener.name.<listener-name>.oauthbearer.connections.max.reauth.ms=
listener.name.<listener-name>.sasl.enabled.mechanisms=
listener.name.<listener-name>.oauthbearer.sasl.jaas.config=
listener.name.<listener-name>.AWS_MSK_IAM.sasl.jaas.config=
listener.name.<listener-name>.AWS_MSK_IAM.sasl.client.callback.handler.class=
listener.security.protocol.map=
principal.builder.class=
```

Really, this can be boiled down to: 

```
listener.name.<lister-name>.<auth-type>.*
listener.name.<listener-name>.sasl.enabled.mechanisms
security.protocol
principal.builder.class
```

Taking this further, the desired yaml should look like: 

```
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: custom
          principal.builder.class: SimplePrincipal.class
          security.protocol: SASL_SSL
          sasl.enabled.mechanisms:
            - OAUTHBEARER
          listener-config:
            oauthbearer.sasl.client.callback.handler.class: client.class
            oauthbearer.sasl.server.callback.handler.class: server.class
            oauthbearer.sasl.login.callback.handler.class: login.class
            oauthbearer.connections.max.reauth.ms: 999999999
          tlsTrustedCertificates: ...
```

Then, when constructing the broker config:

* `principal.builder.class` is set cluster-wide for all authentication methods (this is the status quo for OAuth authentication). This can only be set once, and we will fail hard (i.e. reject the config) if there are multiple principals defined. 
    
    It is expected that the provided principal builder *must* support Strimzi authentication (i.e. ssl-based auth). This is expected to be identical, or very similar to [Strimzi's OAuth library](https://github.com/strimzi/strimzi-kafka-oauth/blob/main/oauth-server/src/main/java/io/strimzi/kafka/oauth/server/OAuthKafkaPrincipalBuilder.java). In particular, it leverages [Kafka's default principal builder class](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/common/security/authenticator/DefaultKafkaPrincipalBuilder.java#L73-L79), which allows the building of a principal based upon the name of the of peer certificate. Thus, it is expected that the custom principal builder provides a principal of type "user" with the name being that of the ssl peer certificate. 

    In addition, it also expected that the custom authorizer supports principals of type "user", and loads the `super.users` property to know which user's (i.e. Strimzi services) are authorized. 


* We parse out `security.protocol` and pass that value into the `listener.security.protocol.map` ** as we currently do for OAuth authentication types. 
* `name` determines the prefix for all entries in `listener-config`
    * I.e., `listener.name.<listener-name>.oauthbearer.sasl.client.callback.handler.class=`

The render config, given the above example, then should look like:

```
listener.name.tls-9093.oauthbearer.sasl.client.callback.handler.class=client.class
listener.name.tls-9093.oauthbearer.sasl.server.callback.handler.class=server.class
listener.name.tls-9093.oauthbearer.sasl.login.callback.handler.class=login.class
listener.name.tls-9093.oauthbearer.connections.max.reauth.ms=999999999
principal.builder.class=SimplePrincipal.class
listener.security.protocol.map=CONTROLPLANE-9090:SSL,REPLICATION-9091:SSL,TLS-9093:SASL_SSL
```

## Affected/not affected projects

Affected: [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)

## Compatibility

Backwards compatibility shouldn’t be an issue, as current configuration should not affect custom authentication. However, users who try to have multiple auth types (i.e. OAuth and Custom) could run into issues if the custom auth requires `principal.builder.class` (as OAuth setting requires that this be set).

Forwards compatibility will need to be mindful of the custom authentication setting, and how to handle the case of multiple listeners/multiple auth types. Explicitly, understand that: 
- `listener.security.protocol.map` is partially built by the custom authentication setting
- `principal.builder.class` can be set in custom auth, and therefore can only be set once
- listener-specific overrides occur here, and therefore `listener.name.<listener-name>.*` should not be overriden anywhere by other config settings

## Rejected alternatives


