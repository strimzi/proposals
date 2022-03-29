# Cluster Wide Volume Usage Quota Management

## Current situation

The [kafka-static-quota-plugin](https://github.com/strimzi/kafka-quotas-plugin) applies byte-rate limits on connections to
individual brokers (thresholds derived from storage limits), slowing producers down when consumed storage is over the soft limit (but below the hard limit) and
effectively pausing publication when reaching or exceeding the hard limit. This largely prevents out of disk scenarios when topics
are replicated to all brokers thus there is largely similar disk usage between all brokers. Assuming all the brokers
have similar disk usage levels, they will all apply rate limits at similar times and levels, effectively giving cluster-wide out-of-storage protection.

However, as clusters scale up the likelihood of even topic distribution drops. When topics are not evenly distributed it
is possible for replication from broker `A` to broker `B` will cause broker `B` to consume all available disk space
without triggering throttling of clients, as broker `A` disk usage remains acceptable.

[KIP-73](https://cwiki.apache.org/confluence/display/KAFKA/KIP-73+Replication+Quotas) replication quotas are designed to
manage the additional network load of migrating replicas between brokers, which does not address client publication
leading to out of disk conditions through replication.

Currently, the kafka-static-quota-plugin considers the total quantity of storage and how much of that is consumed (
see [issue#2](https://github.com/strimzi/kafka-quotas-plugin/issues/2)) when considering whether to apply throttling to
clients. This is problematic with respect
to handling disk failure for JBODs [(KIP-112)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-112%3A+Handle+disk+failure+for+JBOD) as the broker
will take partitions offline when the volume they are stored on runs out of space. Which in the case of unbalanced usage
between volumes can lead to a volume running out of storage without throttling being applied. The broker will go offline
if all volumes are unavailable.

Configuring the soft and hard limit thresholds is currently done in terms of bytes consumed, which forces users
deploying the plugin to calculate a threshold for throttling and would require separate thresholds per volume to avoid
out of space issues.

## Proposal
### Leverage Kafka to distribute volume usage metrics throughout the cluster.
By publishing per volume usage metrics to a compacted topic keyed by the broker ID, each instance of the plugin will be able to start and quickly determine the state of the cluster and consistently apply throttling regardless of where the
client is connected.

[KIP-257](https://cwiki.apache.org/confluence/display/KAFKA/KIP-257+-+Configurable+Quota+Management) describes configurable quota management, where quotas are defined in terms of how much of a delay to apply to a given produce request. This leads to defining the
possible states of the plugin as:
`OPEN`, `THROTTLE` and `PAUSE` to accurately reflect the effects of transitioning between states.

This implies there are three roles.
1. **Data source** - Publish volume usage metrics about the volumes underlying the brokers log dirs.
2. **Data Sink** - Consume volume usage metrics and make those available for making Quota Policy decisions.
3. **Quota Policy** - Convert volume usage metrics for the entire cluster into throttling decisions.

This proposal envisages them all being implemented by the kafka-static-quota-plugin however by clearly distinguishing the roles we leave open future options to move each of the roles out of process.

#### Limit Types
1. Consumed space limit: Triggers if the consumed space of a volume breaches the value. Candidate for deprecation.
2. Minimum Free Bytes: Triggers if the amount of free space on a volume drops below the configured level. Particularly
   useful for hard limits as an absolute minimum of free space.
3. Minimum Free Percentage. Triggers if the amount of free space on the volume drops below the configured proportion of
   the volumes total capacity.

#### The data source should:
- Collect local volume usage
- Publish usage to an internal compacted topic keyed by broker ID.
- Collect and publish volume usage metrics (using this [message schema](#Message-schema)) on start up and periodically after that.
- Connect as an internal service using TLS and client certificates to connect to the Replication listener.

#### The data sink should:
- Connect as an internal service using TLS and client certificates to connect to the Replication listener.
- On startup and periodically thereafter read the volume usage stats from the topic
- Use the Kafka Admin client to discover the list of currently active brokers

#### The quota policy should:
- Should apply the largest delay caused by each disk which breaches the soft limit.
    - The amount of throttling should be proportional to how much of the difference between the soft and hard limit
      remains for a given volume. Applying more throttle the closer to the hard limit things are.
- Should apply a `PAUSE` level throttle to client produce requests if **any** volume in the cluster breaches the hard
  limit.
- Ensure that the quota plugin, and other Strimzi services, updates are not throttled.
- Should take a configured action when it can not determine the state of all known brokers.
    - The action should be one of `PAUSE` or `OPEN`.
    - Defaulting to `PAUSE` as the fail-safe option.
- Apply a freshness check to the volume usage it reads and ignore stale state [3]. The threshold for staleness should be a configuration parameter.

#### Implementation details
##### Internal API within the quota plugin

To make all this work the quota plugin will need to introduce some new interfaces:

1. A Quota Policy - To encapsulate the logic for evaluating if a volume `complies` or `breaches` a policy
2. A Quota Factor Source - To calculate the factor in the range `0..100`% to apply to the entries in the quota map
3. A Data Source - To publish the current volume usage for the broker in question.

One can envisage a model where the Quota policy was externalised to a separate service which calculated the quota factor
and thus the Quota Factor Source would just provide a wrapper around that external service. For the purposes of this
proposal all three interfaces would be implemented with the current quota plugin.

In this proposal the Factor Source implementation would add the kafka consumer and delegate to the quota policy to
determine what factor to apply.

The Data Source would periodically calculate and publish the usage metrics for the local broker.

##### Proposed interface details
1. QuotaPolicy
```java
/*
 * Copyright Strimzi authors.
 * License: Apache License 2.0 (see the file LICENSE or http://apache.org/licenses/LICENSE-2.0.html).
 */
package io.strimzi.kafka.quotas.policy;

import io.strimzi.kafka.quotas.VolumeDetails;

/**
 * Abstracts the decision-making around hard and soft limits and how to calculate the affect the impact breaching the limits has on the client request.
 */
public interface QuotaPolicy {

    /**
     * Does the particular volume breach this policy's defined hard limit.
     *
     * @param volumeDetails details of the disk in question.
     * @return <code>true</code> if this policy considers the volume to breach the limit otherwise <code>false</code>
     */
    boolean breachesHardLimit(VolumeDetails volumeDetails);

    /**
     * Does the particular volume breach this policy's defined soft limit.
     *
     * @param volumeDetails details of the disk in question.
     * @return <code>true</code> if this policy considers the volume to breach the limit otherwise <code>false</code>
     */
    boolean breachesSoftLimit(VolumeDetails volumeDetails);

    /**
     * Returns the fraction of the original quota this policy thinks is appropriate. Represented as percentage value between <code>0</code> and <code>1</code>
     * <p>
     * Where a fraction of <code>1.0</code> is un affected <br>
     * Breaching the hard limit implies a quota factor of <code>0.0</code>
     * @param volumeDetails details of the disk in question.
     * @return A value between <code>0</code> and <code>1</code>.
     */
    double quotaFactor(VolumeDetails volumeDetails);

    /**
     * At what level does this policy start applying a non-zero quota factor.
     * Primarily for metrics purposes.
     * Note: Returns <code>Number</code> to represent both fixed or relative usage levels. e.g. 5% free
     *
     * @return the level at which the quotaFactor becomes non-zero.
     */
    Number getSoftLimit();

    /**
     * At what level does this policy apply its maximum level of throttling.
     * Primarily for metrics purposes.
     * Note: Returns <code>Number</code> to represent both fixed or relative usage levels. e.g. 5% free
     *
     * @return the level at which the quotaFactor becomes non-zero.
     */
    Number getHardLimit();
}
```
2. QuotaFactorSource
```java
/*
 * Copyright Strimzi authors.
 * License: Apache License 2.0 (see the file LICENSE or http://apache.org/licenses/LICENSE-2.0.html).
 */

package io.strimzi.kafka.quotas;

import java.util.function.Supplier;

/**
 * Determines the current restriction factor to be applied to the client quota.
 * Values returned are required to be in the range <code>0.0..1.0</code> inclusive.
 * Where a value of `1.0` implies no additional restriction over and above the defined quota.
 * A value of `0.0` implies that there is no quota available regardless of the defined quota.
 */
public interface QuotaFactorSource extends Supplier<Double> {
}
```
3. DataSourceTask
```java
/*
 * Copyright Strimzi authors.
 * License: Apache License 2.0 (see the file LICENSE or http://apache.org/licenses/LICENSE-2.0.html).
 */

package io.strimzi.kafka.quotas;

import java.util.concurrent.TimeUnit;

public interface DataSourceTask extends Runnable {
    @Override
    void run();

    long getPeriod();

    TimeUnit getPeriodUnit();
}
```

##### Message schema
For simplicity and debug ability the message should be encoded as JSON, but could be converted to a more space efficient
format later on if justified.

###### Message

| Field      | Type                                    | Description                                                                   |
|------------|-----------------------------------------|-------------------------------------------------------------------------------|
| SnapshotAt | `ISO 8601 Date Time`                    | Timestamp when the snapshot of volume usage was generated [1]                 |
| Hard limit | [Limit Definition](#limit-definition)   | Defines when the source broker believes the producers should be paused [2]    |
| Soft limit | `Limit Definition`                      | Defines when the source broker believes the producers should be throttled [2] |
| Volumes    | Set of [VolumeDetails](#volume-details) | Details of the storage volumes attached to the broker                         |

###### Limit Definition

| Field | Type   | Description                                                   |
|-------|--------|---------------------------------------------------------------|
| type  | Enum   | One of `ConsumedSpace`, `MinFreeBytes` or `MinFreePercentage` |
| level | Number | Defines the level at which the limit applies                  |

###### Volume Details

| Field      | Type   | Description                                                      |
|------------|--------|------------------------------------------------------------------|
| VolumeName | String | An identifier to uniquely distinguish the disk. e.g. `/dev/sda1` |
| Capacity   | long   | The capacity of the volume in bytes                              |
| Consumed   | long   | The number of bytes currently in use on the volume               |

Notes:

1. Explicitly mark when the volume usage was captured rather than depending on message publication timestamps.
2. One could view the limits defined at the message level as being broker wide defaults, they could then optionally be
   overridden at the VolumeDetails level with a volume specific limit.
3. Separating freshness from topic eviction as freshness is a "business" concern of the plugin.

##### Metrics
- Throttling applied due lack of data. - Gauge with the value `0` not applied or `1` applied
- Connection status - Gauge with the value `0` disconnected or `1` Connected
- publication errors - Counter to allow the detection of connectivity issues.
- Metrics showing usage per local volume.

##### Configuring the plugin
###### Existing properties
Preserved for backwards compatability.
- `client.quota.callback.static.storage.hard`
- `client.quota.callback.static.storage.soft`

Controlling the number of consumed bytes **above** which throttling is applied. 

###### New properties
- `client.quota.callback.static.storage.hard.min-free-bytes`
- `client.quota.callback.static.storage.hard.min-free-percent` Expressed as `0.0..1.0`
- `client.quota.callback.static.storage.soft.min-free-bytes`
- `client.quota.callback.static.storage.soft.min-free-percent` Expressed as `0.0..1.0`    

Expressed as the number of available bytes, derived from the proportion of the total volume size, **below** which throttling is applied.

## Rejected Alternatives

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
