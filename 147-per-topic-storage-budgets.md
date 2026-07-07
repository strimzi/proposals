# Per-topic storage budgets with principal-level enforcement

This proposal extends the [kafka-quotas-plugin](https://github.com/strimzi/kafka-quotas-plugin) so that operators can assign storage budgets to individual topics.
When a topic exceeds its budget, only the principals writing to that topic are throttled, while every other tenant continues at full rate.
The existing cluster-wide protection remains in place as a backstop.

## Current situation

Since [proposal 047](https://github.com/strimzi/proposals/blob/main/047-cluster-wide-volume-usage-quota-management.md), the kafka-quotas-plugin provides cluster-wide out-of-disk protection.
`VolumeSource` polls `Admin#describeLogDirs` (KIP-827) every `storage.check.interval` to collect the total and available bytes of each volume.
If any volume breaches `storage.per.volume.limit.min.available.{bytes,ratio}`, the throttle factor drops to 0.0 and `StaticQuotaCallback` cuts the produce rate of all non-excluded principals to roughly 1 byte/s (Kafka cannot represent a zero quota).

This design is intentional: proposal 047 scoped the plugin as a last line of defense against running out of disk, and it fills that role well.
As a consequence, the enforcement decision is binary and global: the plugin does not track which topic is consuming the disk and has no way to act on a subset of clients.
The per-principal control it offers is a static produce/fetch rate that is identical for all principals and independent of storage state.

## Motivation

Consider a large multi-tenant cluster with hundreds of brokers, multi-terabyte volumes, short retention, and high per-broker ingest, protected only by the cluster-wide volume limit.

The cluster-wide protection reliably prevents the disk from filling up, which is exactly its job, but the blast radius of enforcement is the whole cluster.
A single tenant ramping up a new topic can cross a volume threshold quickly, at which point produce freezes for every tenant on every broker.
A single-tenant incident becomes a full-cluster producer outage.

Several existing mechanisms come close, but each was designed for a different problem:

- Per-user quotas (`KafkaUser.spec.quotas` or the built-in callback) cap request rates, but they cannot be combined with this plugin, because Kafka allows only one `client.quota.callback.class` per broker.
  They are also static rather than driven by disk pressure, and a rate cap does not bound storage, since the footprint grows with time and broker count.
- `retention.bytes` bounds the size of each partition rather than the topic, cannot reclaim the active segment, and gives the producer no backpressure (see Rejected alternatives).
- Cruise Control disk goals balance replicas across volumes, which helps with skew, but the total bytes stay the same.
- Tiered storage (KIP-405) lowers steady-state local usage, but it cannot stop ingest that outruns offload and does not fit short-retention clusters.

This proposal builds on the foundation these mechanisms provide, adding the missing piece: detect the topic that is over its storage allowance, throttle the writers of that topic, and keep everyone else at full rate.

## Proposal

1. Operators assign storage budgets to topics, either as absolute bytes or as a fraction of total cluster storage, with pattern-based defaults and per-topic overrides.
2. `VolumeSource` additionally aggregates per-topic sizes from the `replicaInfos` map that is already present in the `describeLogDirs` response it fetches today, so no new admin calls are needed.
   A topic's usage is the sum of its replica sizes across all log dirs, excluding future replicas; replicas are counted because budgets bound physical disk consumption.
3. When a topic exceeds its budget, the plugin resolves the principals mapped to that topic and drops their produce quota to a configured floor (1 byte/s by default), leaving all other principals untouched.
4. A topic must fall below `budget × (1 − hysteresis)` before its principals are restored, so enforcement does not flap around the threshold.

Only topics matching a budget entry are tracked, which bounds memory usage and metric cardinality.

### Why enforcement is principal-level

`ClientQuotaCallback` only sees the quota type, principal, and client ID, and produce throttling is applied per connection before the broker routes partitions.
A true per-topic rate quota is therefore not achievable within the quota callback contract and would require a new interception point in Apache Kafka, which means a KIP.
The callback API is also moving away from topic awareness: `updateClusterMetadata(Cluster)`, its only topic-aware hook, was never supported in KRaft mode and is deprecated in Kafka 4.4 for removal in 5.0 ([KIP-1200](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=373887083)).
This proposal therefore detects per topic using the log dir data, but enforces per principal using the existing quota path, and does not depend on `updateClusterMetadata`.

### Principal-to-topic mapping

The plugin needs to know which principals write to a topic, and supports two sources for this mapping:

- `config` (the initial mode): the operator declares `topicPattern → principals` explicitly.
  This is deterministic and auditable, and provisioning systems (as well as Strimzi `KafkaUser`/`KafkaTopic` ownership) already hold this data.
- `acl` (optional, additive): the plugin resolves the principals with `WRITE` access on the topic via `describeAcls`, on the same poll cadence.
  This tracks reality automatically, but it over-approximates the writer set and requires an authorizer.

Deriving the mapping from observed traffic is rejected for the first version, because the plugin has no visibility into the produce path (see Rejected alternatives).
Principals on `excluded.principal.name.list` are never budget-throttled.

### Configuration

New properties are added under the existing `client.quota.callback.static.` prefix:

| Property | Meaning |
|---|---|
| `storage.per.topic.budget.default.bytes` \| `.ratio` | Default budget for tracked topics (mutually exclusive) |
| `storage.per.topic.budgets` | `pattern=budget;...` overrides, first match wins |
| `storage.per.topic.throttle.floor` | Produce rate for restricted principals (default 1) |
| `storage.per.topic.release.hysteresis.ratio` | Release margin (default 0.1) |
| `storage.per.topic.principal.mapping` | `config`, `acl`, or `config+acl` |
| `storage.per.topic.principals` | `topicPattern=User:p1,User:p2;...` |

The `Kafka` CR (`spec.kafka.quotas`, `type: strimzi`) gains an optional `topicBudgets` section that the Cluster Operator renders into these properties:

```yaml
quotas:
  type: strimzi
  minAvailableRatioPerVolume: 0.05          # unchanged cluster-wide backstop
  topicBudgets:
    defaultBudget:
      ratio: 0.02
    principalMapping: config+acl
    budgets:
      - topicPattern: "my-events-.*"
        bytes: 21990232555520                # 20 TiB
        principals: [User:my-producer]
```

### Enforcement semantics

- On each poll, the plugin computes the restricted set: the union of the mapped principals of all over-budget topics, minus excluded principals.
  `quotaMetricTags()` adds a `restricted` tag so that Kafka allocates a distinct sensor, `quotaLimit()` returns the floor for restricted PRODUCE sensors, and changes to the set trigger the existing `quotaResetRequired` path.
- Each broker computes the restricted set independently from the same cluster-wide data, just like the global throttle factor today, so no coordination is added and brokers can diverge for at most one poll interval.
- The effective produce limit for a principal is `min(static quota × global throttle factor, floor if restricted)`.
- If the per-topic data goes stale past `throttle.factor.validity.duration`, budget enforcement disengages and the cluster-wide `minAvailable*` limit remains as the fail-safe.
- Internal topics such as `__consumer_offsets` and the transaction state topic are never budgeted.

### Metrics

The plugin exposes new metrics following the existing `io.strimzi.kafka.quotas` conventions: `TopicUsedBytes`, `TopicBudgetBytes`, and `TopicOverBudget` gauges tagged by topic, `TopicBudgetViolations` and `PrincipalRestricted` counters, and `RestrictedPrincipals`, `TrackedTopics`, and `TopicUsageStale` gauges.
Comparing used bytes against the budget enables tenant burn-down dashboards and alerting before the throttle engages, for example at 80% of budget.

### Known limitations

- Enforcement granularity is the principal, so a restricted principal's writes to healthy topics are throttled too.
  This is inherent to the callback API and can be mitigated by using one principal per tenant or pipeline.
- Budgets bound topics, not volumes: a topic within its budget can still fill a skewed volume, and detection lags by up to one poll interval.
  The cluster-wide limit remains the hard guarantee against running out of disk; budgets reduce how often it fires and how many tenants it hits.
- Over-budget topics with no mapped principals only produce metrics, which doubles as a dry-run mode.

## Affected/not affected projects

- Affected: `strimzi/kafka-quotas-plugin` and `strimzi/strimzi-kafka-operator` (extending the `QuotasPluginStrimzi` CRD type, config rendering, and documentation).
- Not affected: bridge, canary, drain-cleaner, and the behavior of the Topic Operator, User Operator, and Cruise Control, whose principals stay excluded.

## Compatibility

The change is purely additive.
Clusters that set no `storage.per.topic.*` property keep today's behavior, and no existing property, metric, or behavior changes.
There are no new Kafka API requirements, since `replicaInfos` predates KIP-827.
ACL mode needs `describeAcls` permission, which is only required when that mode is enabled.

## Rejected alternatives

- True per-topic produce rate quotas: the quota callback contract does not support this, since it only sees the principal and client ID, enforces per connection, and has no topic-aware hook after KIP-1200.
  This would require an Apache Kafka KIP, and such a KIP could later supersede the enforcement half of this proposal while keeping the budget model.
- Using `retention.bytes` as enforcement: it is per partition, so partition skew and partition-count changes make the topic-level bound loose, it cannot reclaim the active segment, and it deletes committed data while the producer keeps writing at full rate.
  It works well for tenants that prefer data expiry over throttling, so it is complementary rather than an alternative.
- Tiered storage: a good fit for reducing steady-state local usage, but it cannot stop ingest that outruns offload, and short-retention clusters gain little from it.
- An external watchdog using metrics and `AlterClientQuotas`: with the plugin installed, `StaticQuotaCallback` does not apply dynamic quota updates, so the watchdog cannot act through the native path.
  Its scrape-and-reconcile reaction time of minutes is also slow against the fill rates of high-throughput clusters, and in-broker enforcement on data that is already being polled avoids operating an additional system.
- Mapping principals from observed traffic: there is no supported produce-path hook that exposes the principal-topic pair, and scraping per-client JMX metrics across a large cluster would be hard to keep reliable.
  The mapping abstraction leaves room for this mode if Kafka ever exposes write attribution.
