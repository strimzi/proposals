# Support for `type: custom` client authentication

When configuring authentication, Strimzi differentiates between two different types of authentication:
* Server-side authentication configured in the `.spec.kafka.listeners.authentication` section of the `Kafka` custom resource
* Client-side authentication configured in the `.spec.authentication` sections of the `KafkaConnect` and `KafkaBridge` resources, and in the `.spec.clusters.authentication` section of the `KafkaMirrorMaker2` resource.

On the server side, we currently support 4 different authentication types:
* mTLS authentication based on TLS certificates
* SASL SCRAM-SHA-512 authentication based on username and password with the users managed directly inside the Kafka cluster
* OAuth authentication based on the SASL OAUTHBEARER mechanism using the Strimzi OAuth library
* Custom authentication that gives users the flexibility to configure custom authentication mechanisms

On the client side, we currently support 5 different authentication types:
* mTLS authentication based on TLS certificates
* SASL PLAIN, SCRAM-SHA-256, and SCRAM-SHA-512 authentication based on username and password
* OAuth authentication based on the SASL OAUTHBEARER mechanism using the Strimzi OAuth library

There is currently no _custom_ authentication supported on the client side.
This proposal aims to add support for custom authentication to the client side as well, based on the same principles used to configure the custom authentication on the server side.

## Motivation

The main demand for supporting custom client-side authentication comes from users who need to integrate with 3rd party Apache Kafka services.
These services often use custom authentication mechanisms to better integrate with other services provided by a given service provider.
The most common example of such a service is the Amazon MSK service, which uses a custom authentication mechanism that integrates with the Amazon IAM service that is used to manage identity and access in the Amazon AWS cloud.
Another example is the Microsoft Event Hubs service, which has custom OAuth-based authentication using the Microsoft Entra ID service.

Some of these services provide an alternative authentication mechanism which is often based on one of the more common Kafka authentication mechanisms.
For example, Microsoft Event Hubs supports SASL PLAIN-based authentication as an alternative mechanism.
However, even when such an alternative mechanism is available, it is often not as secure as the custom authentication mechanism, and does not integrate so well with the native identity and access management services of a given provider.

Strimzi users typically integrate with these services in two different situations:
* Users run a managed Apache Kafka service for their Kafka cluster (brokers and controllers) but use Strimzi to self-manage other components like Kafka Connect or the HTTP Bridge.
* Users employ MirrorMaker 2 to synchronize data between different managed Kafka services or between a Strimzi-based cluster and a managed one.
  This can be for a one-off migration or a long-term hybrid cloud architecture.

Adding support for custom client-side authentication would give these users a way to use the custom authentication mechanisms.
It would allow them to improve their security (by using better mechanisms and better integration with other services).
And it should also make it easier to configure their operands as it should be more obvious how to configure the authentication.

## Proposal

A new `type: custom` authentication type will be added to all places where we use client-side authentication today:
* Kafka Connect
* Strimzi HTTP Bridge
* Kafka MirrorMaker 2

The API for the new `custom` type client-side authentication would follow the existing server-side API.
For more information, see [`KafkaListenerAuthenticationCustom` schema reference](https://strimzi.io/docs/operators/in-development/configuring#type-KafkaListenerAuthenticationCustom-reference)
It will have a flag indicating whether the authentication mechanism uses SASL.
And it will have a `config` section for adding additional configuration options such as `sasl.mechanism` or JAAS configuration.

The `secrets` field that is present in the server-side `custom` authentication but is already deprecated will not be used in the client-side API.
Users can use the _additional volumes_ feature to add custom secrets required by the authentication mechanism.

Users can also use configuration providers for values such as passwords to load them from files or environment variables without having them hardcoded and visible in the Strimzi custom resources.

The `config` field will allow configuration of the fields starting with the following prefixes:
* `ssl.keystore.`
* `sasl.`

Any other options will be filtered out with a warning and not used.

These options are not permissible with other types of authentication and cannot be set in the regular _configuration_ sections (i.e. in `.spec.config` or `.spec.clusters.config`).
Other options which might be needed by the custom authentication mechanisms can be configured in the regular _configuration_ sections.
That way, we make sure that users are able to configure the custom mechanisms, but cannot use it to override the other configuration options blocked by Strimzi (e.g. the REST API configuration for Connect) by mistake or intention.

Use of TLS and the configuration of trusted certificates will continue to be done through the corresponding `tls` sections.
It will not be configurable through the custom authentication mechanism (as only options with `ssl.keystore.` and `sasl.` prefixes will be allowed).

### Examples

The following examples show how the new `custom` type authentication might be used for various use cases.

* Amazon IAM authentication for connecting to Amazon MSK:
  ```yaml
  authentication:
    type: custom
    sasl: true
    config:
      sasl.mechanism: AWS_MSK_IAM
      sasl.jaas.config: software.amazon.msk.auth.iam.IAMLoginModule required;
      sasl.client.callback.handler.class: software.amazon.msk.auth.iam.IAMClientCallbackHandler
  ```
* Microsoft Event Hubs authentication using Microsoft Entra ID:
  ```yaml
  authentication:
    type: custom
    sasl: true
    config:
      sasl.mechanism: OAUTHBEARER
      sasl.jaas.config: org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
      sasl.login.callback.handler.class: CustomAuthenticateCallbackHandler
  ```
* Custom mTLS authentication (when the key is injected by an external mechanism rather than loaded from a Kubernetes `Secret`):
  ```yaml
  authentication:
    type: custom
    sasl: false
    config:
      ssl.keystore.location: /mnt/my-custom-tls-cert/keystore.p12
      ssl.keystore.password: changeme
      ssl.keystore.type: PKCS12
  ```

### Custom authentication plugins

Strimzi will not include any custom authentication plugins in its container images.
Users will be responsible for adding them on their own.
They can extend the Strimzi container images or use some of the more advanced features such as [Image Volumes](https://github.com/strimzi/proposals/blob/main/102-using-image-volumes-to-improve-extensibility-of-Strimzi-operands.md) as documented in our [docs](https://strimzi.io/docs/operators/latest/full/configuring.html#con-common-configuration-volumes-reference).

### Implementation details

The implementation of the `type: custom` client-side authentication will affect the following classes:
* `KafkaConnectConfigurationBuilder`
* `KafkaBridgeConfigurationBuilder`
* `KafkaMirrorMaker2Connectors`

These classes already handle the existing authentication methods.
Configuring custom authentication will be added to these classes as yet another possible authentication type.

### Testing strategy

Unit tests will be used to ensure the custom authentication can handle the various configurations users might use.

The Strimzi part of this feature does not do any authentication; it just passes the configuration in the correct format to the operands.
Therefore there should be no need for dedicated system tests and unit tests should be sufficient.

Should we decide at a later stage to add dedicated system tests for this feature, we should use only one of the server-side mechanisms already supported by Strimzi for the test - for example SCRAM-SHA-512.
As we do not plan to ship any custom authentication plugins, there should be no need for any system tests covering the integration with 3rd party services.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

This proposal adds a new authentication type and does not have any impact on the existing types.
As such, it is fully backwards-compatible.

## Out of scope

We should consider replacing the existing `type: oauth` authentication with `type: custom` in the future (both on server and client side).
The `type: oauth` authentication has a large number of different options and configuration types which allow many different configurations.
They introduce unnecessary interdependencies between our OAuth library and the Strimzi Cluster Operator.
Using `type: custom` for OAuth authentication would provide the following advantages:
* Give users more flexibility to configure OAuth according to their needs
* Provide better decoupling between the OAuth library and Strimzi Cluster Operator (updating the OAuth library bundled in Strimzi will not require any more changes to the Strimzi API and Strimzi Cluster Operator)
* Simplify testing of the different OAuth configurations (Strimzi Cluster Operator will no longer have the responsibility for the different configuration combinations of the OAuth library and would not need to explicitly test them. They will need to be tested in the OAuth library itself only.)
* Significantly reduce our API and related code simplification in the Strimzi API and Cluster Operator

However, this is not part of this proposal and should be considered later in a separate proposal.

## Rejected alternatives

There are currently no rejected alternatives.
