# Cluster Wide Volume Usage Quota Management

Extend the static quota mechanism to throttle message production in a broker if any active broker in the cluster is running out of disk.

Deprecate the current implementation that considers only aggregate local volume usage.

- [Current situation](#current-situation)
- [Motivation](#motivation)
- [Proposal](#proposal)
  - [High Level Changes](#high-level-changes)
    - [Volume](#volume)
    - [Volume Source](#volume-source)
    - [Quota Source](#quota-source)
    - [Throttle Factor](#throttle-factor)
    - [Cluster Volume Source](#cluster-volume-source)
      - [Admin Client Configuration](#admin-client-configuration)
      - [Limit Type Configuration](#limit-type-configuration)
    - [Fallback Throttle Factor](#fallback-throttle-factor)
- [Metrics](#metrics)
- [Rejected Alternatives](#rejected-alternatives)
- [Affected Projects](#affected-projects)
- [Compatibility](#compatibility)

## Current situation

The [kafka-static-quota-plugin](https://github.com/strimzi/kafka-quotas-plugin) applies byte-rate limits on client connections to
individual brokers (thresholds derived from storage limits), slowing producers down when used storage is over the soft limit (but below the hard limit) and
effectively pausing publication when reaching or exceeding the hard limit. This largely prevents out of disk scenarios when topics
are replicated to all brokers thus there is largely similar disk usage between all brokers. Assuming all the brokers
have similar disk usage levels, they will all apply rate limits at similar times and levels, effectively giving cluster-wide out-of-storage protection.

This however provides limited protection to clusters with un-even distribution of topics and thus storage usage.
Additionally, it ties the plug-in directly to storage usage, there are other factors which users may wish to respond to.

As clusters scale up the likelihood of even topic distribution drops. When topics are not evenly distributed it
is possible for replication from broker `1` to broker `2` will cause broker `2` to consume all available storage
without triggering throttling of clients, as broker `1` disk usage remains acceptable.

Addressing the effects of uneven topic distribution sounds like it should come under [KIP-73](https://cwiki.apache.org/confluence/display/KAFKA/KIP-73+Replication+Quotas). Unfortunately replication quotas
are designed to manage the additional network load of migrating replicas between brokers, which does not address client publication
leading to out of disk conditions through replication.

Currently, the kafka-quotas-plugin considers the total quantity of storage and how much of that is used (
see [issue#2](https://github.com/strimzi/kafka-quotas-plugin/issues/2)) when considering whether to apply throttling to
clients. This is problematic with respect to handling disk failure for Just a Bunch Of Disks (JBOD) deployments [(KIP-112)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-112%3A+Handle+disk+failure+for+JBOD) as the
broker will take partitions offline when the volume they are stored on runs out of space. Which in the case of unbalanced usage
between volumes can lead to a volume running out of storage without throttling being applied. The broker will go offline
if all volumes are unavailable.

Throttling down message production on all broker nodes will protect the cluster from
running out of disk due to replication.

## Motivation

Users need better protection against running out of disk on any of their nodes as it can degrade the service
in unpredictable ways (corrupted segment logs) which could impede the interventions required to recover.

## Proposal

1. Extend the kafka-quotas-plugin so that we can choose to observe the disk usage 
of all brokers in the cluster.
2. Deprecate the existing local disk observations.
3. Add new limit types, so that it is explicit what is being limited and better support heterogeneous disks
   1. throttle if used bytes exceeds threshold on any volume
   2. throttle if available bytes less than threshold on any volume
   3. throttle if available ratio less than threshold on any volume
4. Introduce an extension point in the quotas-plugin to support pluggable sources of quotas
5. Introduce a fallback throttle factor in case we cannot retrieve the cluster state

So every broker will make its own independent throttling decision based on knowledge of the volumes on all active broker nodes. If it detects that any volume in
the cluster is becoming too full it will throttle production of messages. The kafka quota API isn't rich enough to do anything 
smarter about only throttling writes to the brokers running out of space, so we fence the whole cluster.

### High level changes
To better support external sources for managing quotas this proposal introduces some new concepts to the plugin:

#### Quota Source

We propose adding a Quota Source concept to the plugin to provide an extension point where we could plug in
future sources of quotas, like pull them from an external system.

#### Throttle Factor

We introduce the concept of throttle factor. This is a factor in the range (0.0, 1.0) that is applied to a quota to
calculate the final byte rate limit. 

This is a concept that was already implicit in the current calculations, but we want to name it so that we can use it in 
metric names and provide an extension point in case we want to externalise the Throttle Factor.

#### Volume

The storage quotas operate on observations about a **Volume** with these characteristics:
1. `logDir`: the path of the logdir
2. `brokerId`
3. `totalBytes`
4. `availableBytes`
5. `usedBytes` (totalBytes - availableBytes)
6. `availableRatio` (availableBytes/totalBytes)

#### Volume Source

The quota plugin operates on observations about Volumes, the **Volume Source** determines where
we obtain those observations. Values are:
1. `local`: we inspect this brokers local log dirs to discover the state of their Volume
2. `cluster`: we ask kafka for the state of all log dirs for all currently active nodes

the source will be configurable with a property like:

1. `client.quota.callback.static.storage.volume.source=cluster`

Local will be the default if the property isn't provided but is deprecated from the beginning.

#### Cluster Volume Source

With the introduction of [KIP-827](https://cwiki.apache.org/confluence/display/KAFKA/KIP-827%3A+Expose+log+dirs+total+and+usable+space+via+Kafka+API) in
kafka 3.3 we can now obtain the total and usable (available) bytes per log dir as part of the DescribeLogDirsResponse.

Note: if a single disk contains multiple log dirs, it will be described multiple times through the kafka APIs. This
repetition is acceptable as our new limit types will be applied per-volume, so redundant volume descriptions don't
impact the outcome.

The Cluster Volume Source will use this API to discover volume information for the whole cluster.

##### Admin Client Configuration

To obtain log dir descriptions though the admin api we need to construct an admin client. We intend to
connect to the replication listener and will piggyback on the broker's existing SSL configuration to
obtain keystore/truststore properties.

If using cluster sourced volumes we will expose these optional properties to customise the admin connection:

1. `client.quota.callback.kafka.listener.name` (default `replication-9091`)
2. `client.quota.callback.kafka.listener.port` (default `9091`)
3. `client.quota.callback.kafka.listener.protocol` (default `SSL`)
4. `client.quota.callback.kafka.clientIdPrefix` (default `__strimzi`)

Setting `client.quota.callback.static.storage.hard` or `client.quota.callback.static.storage.soft` would be incompatible
with the cluster sourced volumes and fail configuration of the plugin.

The admin client will re-use the broker SSL configuration

##### Limit Type Configuration

When operating with cluster sourced volume data the existing limit types applied to the aggregate
used bytes of all volumes would be meaningless. Nodes enter and exit the active set as part of normal operation.
So we propose that the existing limiting method should be incompatible with cluster sourced volumes.

Instead, we propose introducing new limit types applied on a per-volume basis. Meaning that there is a single value
for each limit which we test against each volume. i.e. we do not support limiting based on a specific volume.

We retain the existing concept of a soft limit (where throttling begins) and a hard limit (where we effectively
stop message production) for all limit types.

The only requirement is a single hard-limit configuration, if no soft-limit is configured we will infer that soft-limit==hard-limit.

The different limit types can be combined, so you could combine a usedBytesExceeds soft limit with a availableRatioBelow
hard limit.

Defining multiple soft limits or multiple hard limits would be an invalid state.

The limits we want are:
1. throttle if [usedBytes](#volume) is greater-than-or-equal-to some threshold for any volume
2. throttle if [availableBytes](#volume) is less-than-or-equal-to some threshold for any volume
3. throttle if the [availableRatio](#volume) is less-than-or-equal-to some threshold for any volume

Using configuration properties like:

1. `client.quota.callback.static.storage.perVolumeLimit.usedBytesExceeds.soft=14000000000`
2. `client.quota.callback.static.storage.perVolumeLimit.usedBytesExceeds.hard=15000000000`
3. `client.quota.callback.static.storage.perVolumeLimit.availableBytesBelow.soft=5000000000`
4. `client.quota.callback.static.storage.perVolumeLimit.availableBytesBelow.hard=1000000000`
5. `client.quota.callback.static.storage.perVolumeLimit.availableRatioBelow.soft=0.05`
6. `client.quota.callback.static.storage.perVolumeLimit.availableRatioBelow.hard=0.01`

`availableRatioBelow` value can be in the range (0.0, 1.0)

#### Fallback Throttle Factor

We are going to depend on using the admin client to get volume information. This brings all the baggage of making a
connection and dealing with possible failures. Also, we can potentially have an inconsistent view of the world between
determining the active broker set and asking for a description of all log dirs. So we need to react somehow to cases
where we cannot get the volume data for all active brokers.

We propose introducing a configurable **fallback throttle factor** to be applied in situations where we don't have enough
information to act. With a default value of 0.0 ie effectively stopping all production to the cluster. This would allow
users to opt in to more dangerous behaviour like disabling this failsafe.

Fallback throttle factor can be in the range (0.0, 1.0)

Example configuration: `client.quota.callback.static.fallback.throttle.factor=0.0`

## Metrics

1. Add a `io.strimzi.kafka.quotas:type=LocalThrottleFactor,name=ThrottleFactor` gauge. This would emit the most recently calculated throttle factor.
2. Add a `io.strimzi.kafka.quotas:type=LocalThrottleFactor,name=FallbackThrottleFactorApplied,reason=xyz` counter. Where reason could be something like 'connection error'.
3. Add a `io.strimzi.kafka.quotas:type=LocalThrottleFactor,name=ThrottlingVolume,brokerId=1,volumeLogDir=/path` gauge. The volume that most recently and severely breached a limit.
4. Add a `io.strimzi.kafka.quotas:type=ClusterVolumeSouce,name=ActiveBrokers` gauge. To expose how many brokers this node considered most recently
5. Add a `io.strimzi.kafka.quotas:type=ClusterVolumeSouce,name=ActiveLogDirs` gauge. To expose how many log dirs this node considered most recently

## Rejected Alternatives

### Using a kafka topic to distribute metrics. 

See the original proposal PR [#51](https://github.com/strimzi/proposals/pull/51). We considered using a kafka topic to
push volume usage metrics out to all the brokers. KIP-827 made this redundant.

### Using JMX metrics

Using JMX metrics directly would require a web of connections between brokers and the exposing of the JMX port to the
rest of the cluster. Using JMX is also problematic for tracking state across restarts of brokers as each broker would
lose state across restarts and thus lose track of any broker which is temporarily offline.

### External metrics store

Would require the following:

- The quota plugin understands the API of the external metrics system
- A metrics system endpoint exposed to the broker for consuming metrics
- A predictable and consistent naming convention

It would also make the deployment of an external metrics store a requirement for the kafka-static-quota-plugin to function.

### KIP-73

KIP-73 is designed to protect client performance while cluster re-balancing exercises are taking place by limiting the
bandwidth available to the replication traffic. This is not suitable for use in preventing out of storage issues as the
bandwidth limit is configured as part of the partition re-assignment operation. As it applies a bandwidth limit it is
configured in  `units per second` which is problematic for the quota plugin to determine a sensible value for as it
should really be related to the expected rate at which data is purged from the tail of the partitions on the volume in
question. KIP-73 bandwidth limits are only applied to a specific set of `partition` & `replica` pairs which would
require the ability for the plugin to resolve the required pairs.

## Affected projects
* strimzi/kafka-quota-plugin

## Compatibility
Backwards compatibility would be maintained at first.

Configuration preserved for backwards compatability:
- `client.quota.callback.static.storage.hard`
- `client.quota.callback.static.storage.soft`
Controlling the number of aggregate used bytes **above** which throttling is applied. Compatible only with local volume source.

Attempting to use a `cluster` volume source with a kafka older than 3.3.0 will prevent the broker from starting up
and emit some sane log indicating the broker version is incompatible.

The existing metrics would also be deprecated but continue to be emitted if the plugin is using locally sourced volumes:
- `io.strimzi.kafka.quotas:type=StorageChecker,name=TotalStorageUsedBytes`
- `io.strimzi.kafka.quotas:type=StorageChecker,name=SoftLimitBytes`
- `io.strimzi.kafka.quotas:type=StorageChecker,name=HardLimitBytes`