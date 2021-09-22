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
    
    It is expected / will be documented that the provided principal builder *must* support Strimzi authentication (i.e. ssl-based auth). 
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

Backwards compatibility shouldn’t be an issue, as current configuration should not affect custom authentication. 

Forwards compatibility will need to be mindful of the custom authentication setting, and how to handle the case of multiple listeners/multiple auth types.  

## Rejected alternatives


