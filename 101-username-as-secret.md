# Username as secret in `KafkaClientAuthenticationPlain`

Add the ability to populate username from a Kubernetes secret in the `KafkaClientAuthenticationPlain` configuration.

## Current situation

Currently, in Strimzi, the `KafkaClientAuthenticationPlain` configuration requires the username to be provided as
plaintext within the Kafka resource. There is no mechanism to populate the username from a Kubernetes secret, even
though passwords can be stored securely using a Kubernetes secret.

## Motivation

The username is a sensitive credential that should not be stored in plaintext within the Kafka resource definition.
Currently, the password is already configurable via a Kubernetes secret, but the username must be provided as a
plaintext string. This proposal enhances security by allowing the username to be stored securely in a Kubernetes secret,
reducing the risk of accidental exposure.
Additionally, this feature simplifies credential rotation. By storing the username in a secret, users can update it
dynamically without modifying Kafka resources, reducing the operational overhead of managing authentication credentials.

## Proposal

To allow users to configure the username using a Kubernetes secret, a new field named `usernameSecret` will be
introduced in the `KafkaClientAuthenticationPlain` configuration.
This field will allow users to specify the name of the Kubernetes secret and the key within the secret that contains the
username.

Users will be able to configure the username using a Kubernetes secret as shown below:

```yaml
authentication:
  type: plain
  usernameSecret:
    secretName: my-connect-secret
    username: my-username-field-name
  passwordSecret:
    secretName: my-connect-secret
    password: my-password-field-name
```

### Behavior and Precedence

- If both `username` and `usernameSecret` are specified, `usernameSecret` takes precedence.
- The Strimzi operator will read the username from the secret and pass it to the Kafka client configurations.
- If `usernameSecret` is specified but the referenced secret or key does not exist, the authentication configuration
  should fail with a clear error message.

## Affected/not affected projects

The `KafkaClientAuthenticationPlain` is used in `KafkaBridgeSpec`, `KafkaConnectSpec`, `KafkaMirrorMaker2ClusterSpec`,
`KafkaMirrorMakerConsumerSpec`, `KafkaMirrorMakerProducerSpec`
and as such this change affects these components.

## Compatibility

For backwards compatibility, the `username` field will remain in the `KafkaClientAuthenticationPlain` configuration.
User will have the ability to configure the username as a plain string using the `username` field or the
`usernameSecret` field.

## Rejected alternatives

There are currently no rejected alternatives.
