# Kubernetes Configuration Provider for Apache Kafka

This proposal suggests to create a configuration provider for reading data from Kubernetes Secrets and Config Maps.

## Current situation

Apache Kafka supports pluggable configuration providers which can load configuration data from external sources.
They can be used in the configuration of the different components.
By default, Kafka provides two configuration providers:

* FileConfigProvider for reading configuration records from properties files
* DirectoryConfigProvider for reading configuration from complete files

In addition, users can implement their own configuration providers as well.

## Motivation

Strimzi is using Kubernetes Secrets to store different information.
For example cluster or user certificates or user passwords.
Kafka clients (or other components) running in the same namespace can mount the secrets as volumes or environment variables and use them.
But this does not work for applications running in other namespaces or outside the Kubernetes cluster.
They have to either copy the secrets into their namespace or extract the files to use them.
Users also need to make sure that their data are kept in sync and update them when the original secrets change.

Having a configuration provider which can extract these data directly from the Kubernetes API would eliminate these issues.
It would allow to load the configuration data from Kubernetes API even from other namespaces or outside of the Kubernetes cluster.
Users would just need to use the configuration provider from their properties and have the data pulled from Kubernetes directly.
Loading the configuration data directly from Kubernetes API will also solve the issues with keeping the data up-to-date since they will be loaded directly from the source.

## Proposal

We should create a new configuration provider which would load data from Kubernetes Secrets and Config Maps.
Secrets and Config Maps are a key-value stores.
The configuration provider will get the Secret or Config Map and extract the desired keys from them.

It will use the Fabric8 Kubernetes client to communicate with the Kubernetes API.
There will be no special configuration for connecting to the Kubernetes cluster.
It will just use the default auto-detection of Kubernetes credentials from `kubeconfig` file, mounted tokens etc.
In addition, users can use the Fabric8 environment variables or Java system properties to configure the client.

The configuration provider only reads the requested Config Map or Secret.
The only RBAC rights it needs is the `get` access rights on given resource.
For example:

```yaml
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["my-cluster-cluster-ca-cert"]
  verbs: ["get"]
```

Thanks to that users should be able to give the configuration provider only very restricted access to just the selected resources.

To use the configuration provider, the user will need to configure their client like this:

```properties
config.providers=secrets
config.providers.secrets.class=io.strimzi.kafka.KubernetesSecretConfigProvider
security.protocol=SSL
ssl.keystore.type=PEM
ssl.keystore.certificate.chain=${secrets:myproject/my-user:user.crt}
ssl.keystore.key=${secrets:myproject/my-user:user.key}
ssl.truststore.type=PEM
ssl.truststore.certificates=${secrets:myproject/my-cluster-cluster-ca-cert:ca.crt}
```

The configuration provider should use a separate GitHub repository under the Strimzi GitHub organization.
The repository should be named `kafka-kubernetes-config-provider`.
It should be published to Maven Central as `io.strimzi:kafka-kubernetes-config-provider`.

## Compatibility

The configuration provider should be compatible with all recent Apache Kafka versions.

## Affected components

This is a separate component which should make it easier for users to use Strimzi based Apache Kafka clusters from their applications.
But it has currently no direct impact on the other resources.

## Rejected alternatives

No other alternatives were considered.
