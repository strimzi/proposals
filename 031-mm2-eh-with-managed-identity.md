# EventHub MirrorMaker2 with Managed Identity Custom Authentication

Hello, as part of my team’s work at Microsoft, we would like to use Strimzi MirrorMaker2 to mirror between eventhubs both as source and target.
We managed to deploy Strimzi MirrorMaker2 using our event hubs and now, we would like to contribute an additional implementation of the AuthenticateCallbackHandler class.
This implementation will include authenticating with Azure Managed Identity (MSI), which is widely used in the Azure cloud community.

## Current situation

Currently, we can only use the Strimzi implementation specified in the 'JaasClientOauthLoginCallbackHandler' class.

## Motivation

This solution will have a huge business impact for the Azure cloud community because no solution exists today for mirroring between event hubs, and the more so for the problem of managed identity.

## Proposal

We already have implemented the AuthenticateCallbackHandler interface with MSI, our only request is to support configuration for 'sasl.login.callback.handler.class' which appears in two places in the code that are relevant to MM2;

(1) KafkaMirrorMaker2AssemblyOperator class
in the addClusterToMirrorMaker2ConnectorConfig method - make the 'SASL_LOGIN_CALLBACK_HANDLER_CLASS' configurable

(2) kafka_connect_config_generator.sh script
make the 'OAUTH_CALLBACK_CLASS' configurable

Once that is done, we can contribute our additional implementation under io.strimzi.kafka.oauth.client package.

As far as I saw, the other props are already configurable, please correct me if I'm wrong:
"sasl.mechanism": "OAUTHBEARER"
"security.protocol": "SASL_SSL"
"sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;"

## Affected/not affected projects

N/A

## Compatibility

N/A

## Rejected alternatives

I did go through the https://github.com/strimzi/proposals/pull/41 proposals,
I think that our case is more simple as we just need to add that one implementation and configuration.
Moreover, this can be another feature Strimzi has built-in, and it will be reusable for the Azure community.
Strimzi will now have not only the ability to mirror between EH, a solution that does not exists today, but also doing it with the commonly used authentication of azure identity.



Corrected with https://www.corrector.co/