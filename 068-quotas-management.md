# Quotas management

This proposal introduces a new mechanism for managing the Apache Kafka quota plugins.
It supports both the Apache Kafka's built-in Quota plugin as well as the new (not yet released) version of the [Strimzi Quota plugin](https://github.com/strimzi/kafka-quotas-plugin).

## Current situation

### Quotas in Apache Kafka

Apache Kafka supports a pluggable quota mechanism that allows limiting the load that Kafka users and/or clients can place on the Kafka brokers.

Apache Kafka provides a built-in plugin that is enabled by default and can apply the following quotas:
* Maximum bytes per-second that each client group can publish to a broker
* Maximum bytes per-second that each client group can fetch from a broker
* Maximum CPU utilization as a percentage of network and I/O threads
* Rate per-second at which concurrent mutations are accepted from requests to create topics, create partitions, and delete topics.

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
It can be used to configure specific quotas for a given user using Kafka's built-in quota plugin.

## Motivation

The absence of a dedicated API for quota management in the `Kafka` CR currently creates two major limitations.
1. The built-in Apache Kafka quota plugin has an ability to configure _default quotas_.
   Default quotas apply to any user that has no specific quota set.
   However, the default quotas cannot be configured declaratively in the `.spec.kafka.config` section.
   And they cannot be configured in a `KafkaUser` resource either because they do not belong to any particular user.
   The only way to configure them right now is imperatively use the Kafka Admin API.
   While it can be used by users, it does not follow Strimzi's basic idea of declarative management.
   Having a dedicated API for quota configuration as part of the `Kafka` CR would make it much easier to configure the default quotas.
2. The initial releases of the Strimzi Quota Plugin were simple, preventing users from producing messages directly to a broker running out of disk space. 
   However, these versions did not address the issue of replicated messages produced on other brokers in the Kafka cluster, which could still fill up the disk space of the destination broker.
   They also didn't support many useful features, such as JBOD storage.
   Because the initial releases were so simple, it was possible to configure them in the `.spec.kafka.config` section of the `Kafka` CR.
   The next - not yet released - version of the Strimzi Quota plugin brings many improvements.
   More details about how the new version works can be found in the [Cluster Wide Volume Usage Quota Management](./047-cluster-wide-volume-usage-quota-management.md) proposal.
   It operates at the level of a whole Kafka cluster and it has improved capability in preventing replicated messages from filling in broker storage.
   It also supports JBOD storage.
   But it also has a much more complicated configuration.
   The plugin uses the Kafka Admin API to gather information about disk capacity from the Kafka brokers.
   So, it needs to connect to the Kafka cluster as a client.
   Having the user configure this directly in the `.spec.kafka.config` section would add a lot more complexity, as the user would need to configure the credentials for securing a connection,  such as keystores, truststores, passwords, bootstrap addresses, and so on.
   Having a dedicated API for quota configuration would make it much easier to have the operator configure the client automatically.

## Proposal

This proposal suggests adding a new `quotas` API to the `Kafka` custom resource.
It will be inside the `.spec.kafka` section.
And it will allow users to configure different types of quotas.
As with existing Strimzi APIs, it will use the `type` field to differentiate the quota plugin that will be used.

This proposal suggests to initially add support for two Quota plugins.
1. The built-in Kafka plugin
2. The Strimzi Quota plugin

The first supported plugin - Kafka's built-in plugin - will be configured as `type: kafka`.
The API will allow you to configure the default user quotas for this plugin in the same format as the `KafkaUser` API currently supports:

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

Based on this configuration, the operator will do the following:
* Configure the default quota plugin (make sure the `client.quota.callback.class` option is not set)
* Set the default user quotas
* When the `type: kafka` quotas API is set, but one or more of the properties with the individual quotas are missing, the operator will make sure that the given default quota is not set (set to `null`).
* Other configuration options for the built-in Kafka plugin can be configured by the user in the `.spec.kafka.config` section in the `Kafka` CR.

The second supported plugin - the Strimzi Quota plugin - will be configured as `type: strimzi`.
The API will allow you to configure the storage limits at which the plugin will start limiting the producers in order to prevent the disk from getting full.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    quotas:
      type: strimzi
      producerByteRate: 1000000
      consumerByteRate: 1000000
      minAvailableBytesPerVolume: 5368709120
      excludedPrincipals:
        - my-special-user
    # ... 
  # ...
```

The `type: strimzi` quotas API will support the following fields:
* `producerByteRate` for setting the default produce quota
* `consumerByteRate` for setting the default fetch quota
* `minAvailableBytesPerVolume` for setting the minimal required free space per volume as number of bytes (this field can be configured only when `minAvailableRatioPerVolume`is not set)
* `minAvailableRatioPerVolume` for setting the minimal required free space per volume as percentage of the overall volume size (this field can be configured only when `minAvailableBytesPerVolume`is not set)
* `excludedPrincipals` for setting a list of users to be excluded from the applied quotas

Other options might be configured directly by the user in `.spec.kafka.config` section of the `Kafka` CR.

Based on this configuration, the operator will:
* Set the `client.quota.callback.class` field in Kafka configuration to `io.strimzi.kafka.quotas.StaticQuotaCallback`.
* Set the configuration on how to connect to the Kafka broker as a client using the `client.quota.callback.static.kafka.admin.` options.
* Configure the default produce and fetch quota (`client.quota.callback.static.produce` and `client.quota.callback.static.fetch`).
* Configure the minimal available disk space per-volume using the `client.quota.callback.static.storage.per.volume.limit.min.available.bytes` or `client.quota.callback.static.storage.per.volume.limit.min.available.ratio` options.
* Set excluded users for whom quotas will not be applied using the `client.quota.callback.static.excluded.principal.name.list` option.
  Regardless of whether custom exclusions are specified, the list of excluded users will automatically include internal users employed by the Topic Operator and Cruise Control.
  This ensures these internal users are not obstructed, as they might be required for resolving disk space issues:
  * The Topic Operator modifies the retention settings of existing topics.
    Message production and fetching activities are required only for the Bidirectional Topic Operator. 
    Therefore, the Topic Operator user can be removed if support for the Bidirectional Topic Operator is discontinued.
  * Cruise Control rebalances topics among brokers to address disk space shortages.

If the new `.spec.kafka.quotas` API is not set at all, Strimzi will keep the default Kafka configuration (which is Apache Kafka's built-in Quota plugin).
Or - in case the user customized the quota plugin configuration in the `.spec.kafka.config` section - Strimzi will pass this configuration to Kafka.
That means that the Apache Kafka `client.quota.callback.class` will not be marked as a forbidden Apache Kafka configuration option.

That helps to ensure backwards compatibility.
Any users who currently use the default plugin or any custom plugin would not be required to start using the `.spec.kafka.quotas` section immediately.
They will be able to keep the configuration as is.

When the `.spec.kafka.quotas` API is used and the `.spec.kafka.config` section contains some custom quota plugin configuration, Strimzi will configure the quotas according to the `.spec.kafka.quotas` API.
It will also set a warning condition and log a message about this conflict.

## Affected projects

This proposal affects only the Strimzi Cluster Operator and the `Kafka` custom resource.
There is no change to the Strimzi User Operator or the `KafkaUser` custom resource API or any other projects.

## Backwards compatibility

This proposal is fully backwards compatible.
When the new `.spec.kafka.quotas` API is not set, the operator handles the quotas configuration exactly as in previous Strimzi versions.

## Rejected alternatives

Currently, there are no rejected alternatives.

