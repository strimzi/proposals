# Cluster Wide Volume Usage Quota Management

## Current situation

The [kafka-quota-plugin](https://github.com/strimzi/kafka-quotas-plugin) applies rate limits to connections to
individual brokers, slowing producers down when consumed storage is over the soft limit (but below the hard limit) and
effectively pausing publication when at or over the hard limit. This largely prevents out of disk scenarios when topics
are replicated to all brokers thus there is largely similar disk usage between all brokers. Assuming all the brokers
have similar disk usage levels they will all apply rate limits at similar times and levels this effectively gives
cluster-wide out of storage protection.

However, as clusters scale up the likelihood of even topic distribution drops. When topics are not evenly distributed it
is possible for replication from broker `A` to broker `B` will cause broker `B` to consume all available disk space
without triggering throttling of clients, as broker `A` disk usage is remains acceptable.

[KIP-73](https://cwiki.apache.org/confluence/display/KAFKA/KIP-73+Replication+Quotas) replication quotas are designed to
manage the additional network load of migrating replicas between brokers rather and does not address client publication
leading to out of disk conditions through replication.

Currently, the quota-plugin considers the total quantity of storage and how much of that is consumed (
see [issue#2](https://github.com/strimzi/kafka-quotas-plugin/issues/2)) when considering whether to apply throttling to
clients. This is problematic with respect
to [KIP-112](https://cwiki.apache.org/confluence/display/KAFKA/KIP-112%3A+Handle+disk+failure+for+JBOD) as the broker
will take partitions offline when the volume they are stored on runs out of space. Which in the case of unbalanced usage
between volumes can lead to a volume running out of storage without throttling being applied. The broker will go-offline
if all volumes are unavailable.

Configuring the soft and hard limit thresholds is currently done in terms of bytes consumed, which forces users
deploying the plugin to calculate a threshold for throttling and would require separate thresholds per volume to avoid
out of space issues.

## Proposal

### Leverage kafka to distribute volume usage metrics throughout the cluster.

By publishing per volume usage metrics to a compacted topic keyed by the broker ID each instance of the plugin will be able
to start and quickly determine the state of the cluster and consistently apply throttling regardless of where the client
is connected.

[KIP-257](https://cwiki.apache.org/confluence/display/KAFKA/KIP-257+-+Configurable+Quota+Management) Defines the quota
API and defines quotas in terms of how much of a delay to apply to a given produce request. This leads to defining the
possible states of the plugin as:
`OPEN`, `THROTTLE` and `PAUSE` to accurately reflect the effects of transitioning between states.

#### Limit Types

1. Consumed space limit: Triggers if the consumed space of a volume breaches the value. Candidate for deprecation.
2. Minimum Free Bytes: Triggers if the amount of free space on a volume drops below the configured level. Particularly
   useful for hard limits as an absolute minimum of free space.
3. Minimum Free Percentage. Triggers if the amount of free space on the volume drops below the configured proportion of
   the Volumes total capacity.

#### The quota plugin data publisher should:

- Collect local volume usage
- Publish usage to an internal compacted topic keyed by broker ID.
- Collect and publish usage on start up and periodically after that.
- Connect as an internal service using TLS and client certificates to connect to the Control Plane listener.
- Publish metrics showing its connection status and a counter of the number of publication errors - to allow the
  detection of connectivity issues.
- Publish Volume Information with the following [schema](#Message-schema)

#### The quota plugin consumer should:

- Should apply the largest delay caused by each disk which breaches the soft limit.
    - The amount of throttling should be proportional to how much of the difference between the soft and hard limit
      remains for a given Volume. Applying more throttle the closer to the hard limit things are.
- Should apply a `PAUSE` level throttle to client produce requests if *any* Volume in the cluster breaches the hard limit.
- Connect as an internal service using TLS and client certificates to connect to the Control Plane listener.
- Use the Kafka Admin client to discover the list of currently active brokers
- On startup and periodically thereafter read the Volume usage stats from the topic
- Ensure that the quota plugin Volume updates are not throttled
- Should take a configured action when it can not determine the state of all known brokers.
    - The action should be one of `PAUSE` or `OPEN`.
    - Defaulting to `PAUSE` as the fail-safe option.
- Publish a metric showing when it is applying throttling due to a lack of data.
- Apply a freshness check to the volume usage it reads and ignore stale state [3]. The threshold for staleness
  should be a configuration parameter

#### Message schema

For simplicity and debug ability the message should be encoded as JSON, but could be converted to a more space efficient format later on if justified. 

##### Message

| Field      | Type                                    | Description                                                                   |
|------------|-----------------------------------------|-------------------------------------------------------------------------------|
| SnapshotAt | `ISO 8601 Date Time`                    | Timestamp when the snapshot of volume usage was generated [1]                 |
| Hard limit | [Limit Definition](#limit-definition)   | Defines when the source broker believes the producers should be paused [2]    |
| Soft limit | `Limit Definition`                      | Defines when the source broker believes the producers should be throttled [2] |
| Volumes    | Set of [VolumeDetails](#volume-details) | Details of the storage volumes attached to the broker                         |

##### Limit Definition

| Filed | Type   | Description                                                   |
|-------|--------|---------------------------------------------------------------|
| type  | Enum   | One of `ConsumedSpace`, `MinFreeBytes` or `MinFreePercentage` |
| level | Number | Defines the level at which the limit applies                  |

##### Volume Details

| Filed      | Type   | Description                                                      |
|------------|--------|------------------------------------------------------------------|
| VolumeName | String | An identifier to uniquely distinguish the disk. e.g. `/dev/sda1` |
| Capacity   | long   | The capacity of the volume in bytes                              |
| Consumed   | long   | The number of bytes currently in use on the volume               |

Notes:

1. Explicitly mark when the volume usage was captured rather than depending on message publication timestamps.
2. One could view the limits defined at the message level as being broker wide defaults, they could then optionally be
   overridden at the VolumeDetails level with a volume specific limit.
3. Separating freshness from topic eviction as freshness is a "business" concern of the plugin.

## Rejected Alternatives

### Using JMX metrics

Using JMX metrics directly would require a web of connections between brokers and the exposing of the JMX port to the
rest of the cluster. Using JMX is also problematic for tracking state across restarts of brokers as each broker would
lose state across restarts and thus lose track of any broker which is temporarily offline.

### External metrics store

Would require the quota plugin to:

- understand the API of the external metrics system
- The metrics system expose an endpoint to the broker for consuming metrics
- to have a predictable and consistent naming convention

It would also make the deployment of an external metrics a requirement for the quota-plugin to function.
