# Adopt the Kafka Static Quota plugin

## Current situation

One of the common problems when running Apache Kafka is running out of disk space.
Strimzi currently does not offer any protection against this.
Once the disk gets full, usually the only way how to recover from full disk is to either delete some files (log segments) or to increase the disk size.

## Motivation

Having some solution which would protect from the disks getting full would be a good improvement for Strimzi users.

## Proposal

Strimzi should adopt the [Kafka Static Quota plugin](https://github.com/lulf/kafka-static-quota-plugin).
It provides two types of quotas:

### Per-broker produce and fetch quotas

Apache Kafka itself offers only quotas per client / user.
The Kafka Static Quota plugin allows configuring of an overall produce and/or fetch quota per-broker.
The quota will be distributed between the clients connected to the broker.
The produce / fetch quotas currently don't support per-listener quotas.
There is always only one quota for all listeners.
The replication traffic is not counted into the quota.

### Storage quotas

The Kafka Static Quota plugin allows users to configure two storage limits: soft and hard.
After the soft limit is breached, it will start throttling the producers.
The allowed throughput is linearly decreased until the hard limit is reached.
Once the hard limit is reached, the allowed produce throughput will be 0.
The storage quotas currently don't support JBOD storage, only brokers with single log directory.

### Plugin adoption

In order to include the plugin into Strimzi images, we would need it to be available in Maven repositories.
This proposal suggests to fork the plugin under Strimzi and continue its development.
The name repository should be named `kafka-quotas-plugin`.
We can set up Azure Pipelines build for it and push it to Maven repositories.

We will add the plugin to our Kafka container images via the third-part libraries mechanism.
At this time, we should not add any integration into the Strimzi Kafka CRD.
The plugin and its quotas can be configured through `.spec.kafka.config`.
But we should document it and include it in our system tests.

We should also work on additional features.
For example:

* Support for JBOD storage
* Support for per-listener configuration

## Risks

Despite GitHub trying to make it as easy as possible, renaming the default branch will still cause some disruptions:

* Users / Developers will need to get used to the new default branch name (for example when using commonly used commands such as `git rebase master` etc.)
* Users / Developers will need to update their local repositories to use the new branch

## Affected / not affected projects

This proposal affects only the plugin and the operators repository where it will be added to the Apache Kafka images..

## Compatibility

The quota plugin will not be enabled by default, so there should be no impact on backwards compatibility.
