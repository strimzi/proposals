# Quotas management

This proposal introduces a new mechanism for managing the Apache Kafka quota plugins.
It supports both the Apache Kafka's built-in Quota plugin as well as the new (not yet released) version of the [Strimzi Quota plugin](https://github.com/strimzi/kafka-quotas-plugin).

## Current situation

### Quotas in Apache Kafka

Apache Kafka supports a pluggable quota mechanism that allows limiting the load that Kafka users and/or clients can place on the Kafka brokers.

Apache Kafka provides a built-in plugin that is enabled by default and can apply quotas to:
* Maximum bytes per-second that each client group can publish to a broker
* Maximum bytes per-second that each client group can fetch from a broker
* Maximum CPU utilization as a percentage of network and I/O threads
* Rate at which mutations are accepted for the create topics request, the create partitions request and the delete topics request

These quotas can be set at the user or client level and are useful to:
* Prevent a single client from overloading the broker
* Prevent noisy neighbor situations, where one very busy user or client negatively impacts others
* Ensure fairness in allocating resources to different users and clients

The built-in quota plugin is enabled when `client.quota.callback.class` is set to `null` (i.e. not set) in Apache Kafka configuration.

Instead of using the default plugin, users can also write and use their own quota plugins by implementing the `ClientQuotaCallback` Java interface and configuring it with the `client.quota.callback.class` in Kafka configuration.
The custom implementations can have different goals.
For example, the [Strimzi Quota plugin](https://github.com/strimzi/kafka-quotas-plugin) focuses on preventing the Kafka brokers from running out of disk space instead of limiting the individual clients or users.

### Quotas in Strimzi

Currently, Strimzi does not have any API support for quotas in the `Kafka` custom resource.
All quotas are configured directly by the users in `.spec.kafka.config` section of the `Kafka` custom resource.
That gives users a lot of freedom and flexibility to configure the quotas.
But it also does not provide them any support when the quota plugin requires a more complicated configuration.

The only place where Strimzi directly supports configuring quotas is in the `KafkaUser` resource.
It can be used to configure a specific quota for a given user using Kafka's built-in quota plugin.

## Motivation

The absence of a dedicated API for quota management in the `Kafka` CR currently creates two major limitations.
1. The built-in Apache Kafka quota plugin has an ability to configure _default quotas_.
   Default quotas apply to any user that has no specific quota set.
   However, the default quotas cannot be configured declaratively in the `.spec.kafka.config` section.
   And they cannot be configured in a `KafkaUser` resource either because they do not belong to any particular user.
   The only way to configure them right now is imperatively use the Kafka Admin API.
   While it can be used by users, it does not follow Strimzi's basic idea of declarative management.
   Having a dedicated API for quota configuration as part of the `Kafka` CR would make it much easier to configure the default quotas.
2. The initial releases of the Strimzi Quotas Plugin were very simple.
   It helps to prevent users producing directly to a broker that was running out of disk space.
   But it fails to prevent replication of messages produced to other brokers in the Kafka cluster from filling in the disk.
   It also doesn't support many additional features such as JBOD storage.
   Because the initial releases were very simple, they had also very simple configuration.
   And it was possible to configure them in the `.spec.kafka.config` section of the `Kafka` CR.
   The next - not yet released - version of the Strimzi Quotas plugin brings many improvements.
   More details about how the new version works can be found in the [Cluster Wide Volume Usage Quota Management](./047-cluster-wide-volume-usage-quota-management.md) proposal.
   It operates at the level of a whole Kafka cluster and it has improved ability to prevent replicated messages to fill in the broker storage.
   It also supports JBOD storage.
   But it has also much more complicated configuration.
   The plugin is using the Kafka Admin API to gather the information about the disk capacity from the Kafka brokers.
   And needs therefore to connect to the Kafka cluster as a client.
   Having the user configure this directly in the `.spec.kafka.config` section would be much more complicated, as the user would need to configure the keystores, truststores, their passwords, bootstrap address, etc.
   Having a dedicated API for quota configuration would make it much easier to have the operator configure the client automatically.

## Proposal

This proposal suggests to add a new `quotas` API to the `Kafka` custom resource.
It will be inside the `.spec.kafka` section.
And it will allow users to configure different types of quotas.
As many existing Strimzi APIs, it will use the `type` field to differentiate the quota plugin that will be used.

This proposal suggests to initially add support for two Quota plugins.
1. The built-in Kafka plugin
2. The Strimzi Quotas plugin

The first supported plugin - Kafka's built-in plugin - will be configured as `type: kafka`.
The API will allow to configure the default user quotas for this plugin in the same format as the `KafkaUser` API allows it today:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    quotas:
      type: kafka
      producerByteRate: 1048576
      consumerByteRate: 2097152
      requestPercentage: 55
      controllerMutationRate: 50
    # ... 
  # ...
```

Based on this configuration, the operator will:
* Configure the default quota plugin (make sure the `client.quota.callback.class` option is not set)
* Set the default user quotas
* When the `type: kafka` quotas API will be set, but one or more of the fields with the individual quotas are missing, the operator will make sure that the given default quota is not set (set to null).
* The other configuration options for the built-in Kafka plugin can be configured by the user in the `.spec.kafka.config` section in the `Kafka` CR.

The second supported plugin - the Strimzi Quotas plugin - will be configured as `type: strimzi`.
The API will allow to configure the storage limits at which the plugin will start limiting the producers in order to prevent the disk from getting full.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    quotas:
      type: strimzi
      produceQuota: 1000000
      fetchQuota: 1000000
      minAvailableBytesPerVolume: 5368709120
      excludedPrincipals:
        - my-special-user
    # ... 
  # ...
```

The `type: strimzi` quotas API will support the following fields:
* `produceQuota` for setting the default produce quota
* `fetchQuota` for setting the default fetch quota
* `minAvailableBytesPerVolume` for setting the minimal required free space per volume as number of bytes (this field can be configured only when `minAvailableRatioPerVolume`is not set)
* `minAvailableRatioPerVolume` for setting the minimal required free space per volume as percentage of the overall volume size (this field can be configured only when `minAvailableBytesPerVolume`is not set)
* `excludedPrincipals` for setting a list of users to be excluded from the applied quotas

Other options might be configured directly by the user in `.spec.kafka.config` section of the `Kafka` CR.

Based on this configuration, the operator will:
* Set the `client.quota.callback.class` field in Kafka configuration to `io.strimzi.kafka.quotas.StaticQuotaCallback`
* Set the configuration on how to connect to the Kafka broker as a client using the `client.quota.callback.static.kafka.admin.` options
* Configures the default produce and fetch quota (`client.quota.callback.static.produce` and `client.quota.callback.static.fetch`)
* Configure the minimal available disk space per-volume using the `client.quota.callback.static.storage.per.volume.limit.min.available.bytes` and `client.quota.callback.static.storage.per.volume.limit.min.available.ratio` options
* Set the excluded users for that the quotas will not be applied using the `client.quota.callback.static.excluded.principal.name.list` option

The quota plugin configuration will not be set a forbidden configuration.
When the new `.spec.kafka.quotas` API would be not set at all, Strimzi will keep the default Kafka configuration (which is the default built-in quotas plugin).
Or - in case the user customized the quota plugin configuration in the `.spec.kafka.config` section - Strimzi will pass this configuration to Kafka.

That helps to ensure backwards compatibility.
Any users who current use the default plugin or any custom plugin would not be required to start using the `.spec.kafka.quotas` section immediately.
They will be able to keep the configuration as is.

When the `.spec.kafka.quotas` API is used and the `.spec.kafka.config` section contains some custom quota plugin configuration, Strimzi will configure the quotas according to the `.spec.kafka.quotas` API.
It will also set a warning condition and log a message about this conflict.

## Affected projects

This proposal affects only the Strimzi Cluster Operator and the `Kafka` custom resource.
There is no change to the Strimzi User Operator or the `KafkaUser` custom resource API or any other projects.

## Backwards compatibility

This proposal is fully backwards compatible.
When the new `.spec.kafka.quotas` APi is not set, the operator handles the quotas configuration exactly as in previous Strimzi versions.

## Rejected alternatives

Currently, there are no rejected alternatives.

