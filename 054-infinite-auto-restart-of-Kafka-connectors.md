# Infinite auto-restart of Apache Kafka connectors

This proposal introduces changes to the connector auto-restart functionality which was introduced in [Proposal #7](https://github.com/strimzi/proposals/blob/main/007-restarting-kafka-connect-connectors-and-tasks.md) and implemented in [Strimzi 0.33.0](https://github.com/strimzi/strimzi-kafka-operator/pull/7500).

## Current situation

Strimzi 0.33.0 introduced support for automatically restarting connectors in Apache Kafka Connect clusters (when managed using the `KafkaConnector` resources). This feature is disabled by default, but can be enabled in the `KafkaConnectors` `.spec` section:

```yaml
apiVersion: strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: my-source-connector
spec:
  # ...
  autoRestart:
    enabled: false
  # ...
```

When enabled, it works like this:
* The connector is restarted up to 7 times
* The times of the auto-restarts are:
    * Immediate
    * 2+ minutes after previous restart
    * 6+ minutes after previous restart
    * 12+ minutes …
    * 20+ minutes …
    * 30+ minutes …
    * 42+ minutes …
    * _There is no exact time, so it will be restarted when the next reconciliation happens and the time condition is fulfilled (i.e. restart after 6+ minutes can happen for example after 7 minutes in the next suitable periodical reconciliation)._
* After 7th restart, the connector will not be automatically restarted anymore
* When the container runs successfully for at least the backoff interval for the restart counter to be reset.
  E.g. if it was restarted 4 times, it needs to run for 20 minutes for the restart counter to be reset to 0 and the restart sequence start from the beginning in case of the next failure.

## Motivation

The auto-restart functionality works fine. But in some cases, it would be useful to have more flexibility including a possibility to retry the restarts indefinitely.
Infinite restarts allow the auto-restarting to be used for even longer outages which for example take the whole weekend and it increases the value of this feature.

## Proposal

A new field `maxRestarts` will be added to the `autoRestart` section of the `KafkaConnector` CRD:

```yaml
apiVersion: strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: my-source-connector
spec:
  # ...
  autoRestart:
    enabled: false
    maxRestarts: 20
  # ...
```

The `maxRestarts` field will default to null (not set).
And when it is not set (set to null), the operator will attempt to restart the connector infinitely.
When a user sets `maxRestarts` to a specific value, the operator will attempt to restart the operator only for a given number of attempts and then give up on it.
In that case, the user would need to restart it later manually.

**The solution proposed by this proposal is not fully backwards compatible.**
**Please read the _Compatibility_ section for more details.**

The current timing will be unchanged.
It will be calculated based on the following formula where the `restartCount` means the number of restarts that have already happened.
The maximal value of the back-off time will be set to 60 minutes as maximum.

```
backoff_in_minutes = minimum((restartCount * restartCount) + restartCount, 60)
```

It means the operator will be restarted at following times:
* Immediate
* 2+ minutes after previous restart
* 6+ minutes after previous restart
* 12+ minutes …
* 20+ minutes …
* 30+ minutes …
* 42+ minutes …
* 56+ minutes …
* 60+ minutes …
* 60+ minutes …
* …

_There is no exact time, so it will be restarted when the next reconciliation happens and the time condition is fulfilled (i.e. restart after 6+ minutes can happen for example after 7 minutes in the next suitable periodical reconciliation)._

When the connector runs successfully for at least the backoff interval for the restart counter to be reset.
E.g. if it was restarted 4 times, it needs to run for 20 minutes for the restart counter to be reset to 0 and the restart sequence start from the beginning in case of the next failure.
This is unchanged compared to the current state.

## Affected/not affected projects

This affects only the Strimzi Kafka Operators repository and its Cluster Operator.

## Compatibility

### API Compatibility

This proposal maintains a full API (= the Kubernetes CRDs) compatibility.

### Changes to the semantics

**While the API is fully backwards compatible, the semantics of how the API is used and how the auto-restart works is different.**
In the previous versions, a Kafka Connector with the auto-restart enabled would see it restart up to 7 times after it fails:

```yaml
apiVersion: strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: my-source-connector
spec:
  # ...
  autoRestart:
    enabled: false
  # ...
```

But the same connector after this proposal is introduced would be restarting infinitely.
Existing users who might want to maintain the original behavior would need to change their `KafkaConnector` custom resources and add the `maxRestarts: 7` option :

```yaml
apiVersion: strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: my-source-connector
spec:
  # ...
  autoRestart:
    enabled: false
    maxRestarts: 7
  # ...
```

The infinite restarts will be happening at the maximal bak-off interval of 60 minutes.
So it should cause only a minimal disruption to users who have this activated by mistake.

While this change is not backwards compatible, it gives us a clean API for the long term future, where the `maxRestarts` field would have no defaults and the following configuration would mean infinite restarts.

```yaml
  autoRestart:
    enabled: false
```

This API is easier to understand, read, and provides better user experience.

The alternative would be to have the `maxRestarts` field default to 7.
That might be confusing for new users who start using Strimzi later and might not expect the restart limit to be 7 restarts when nothing is configured.

## Rejected alternatives

### Defaulting to 7 restarts

The `maxRestarts` field will be added as described in this proposal.
But its default value will be `7`.
Thanks to that, existing users will be not affected by this change and this change will provide both API as well as semantic backwards compatibility.
However:
* The default value of `7` will stay in Strimzi forever and might not provide the best user experience in the long term.
* The _infinite_ restarts would need to be represented by setting `maxRestarts` to some special value (for example `0`) or by setting it to some large number such as `1000000` which again is not the most user-friendly way in the long term.
