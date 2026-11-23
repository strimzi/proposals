# Make the KafkaRoller retry window configurable

## Current situation

Consider a partition with replicas on brokers 0, 1 and 2 and `min.insync.replicas=2`.
Broker 0 has restarted and its pod is Ready, but the in-sync replica set (ISR) for this partition still contains only brokers 1 and 2.
Restarting broker 1 now would leave one in-sync replica, so `KafkaAvailability.canRoll(1)` returns `false`.
In the reported cluster, pod initialization alone could take about five minutes, with replica catch-up still to follow. [Original report](https://github.com/strimzi/strimzi-kafka-operator/issues/12649) · [Enhancement #13058](https://github.com/strimzi/strimzi-kafka-operator/issues/13058)

`KafkaRoller` uses `new BackOff(250, 2, 10)` per pod.
The first check is immediate, then nine delays double from 250 ms to 64,000 ms and add up to 127,750 ms.
If the tenth check still returns `false`, the roll fails even if broker 0 is about to catch up.

## Proposal

I propose one setting on the Cluster Operator deployment, `STRIMZI_KAFKA_ROLLER_AVAILABILITY_RETRY_BUDGET_MS`.
It limits the sum of the delays the roller schedules for a pod before giving up on a broker availability refusal during one roll.
Time spent querying Kafka, checking pod readiness or processing other pods is outside that sum.

For a cluster where replicas can take ten minutes to catch up, I would set the operator container environment as follows.

```yaml
env:
  - name: STRIMZI_KAFKA_ROLLER_AVAILABILITY_RETRY_BUDGET_MS
    value: "600000"
```

An unset or empty value selects `127750`.
Accept whole milliseconds from `1` to `3600000` and report an `InvalidConfigurationException` at operator startup for a malformed or out-of-range value.
An extra zero in the example would allow 100 minutes of scheduled waits while the reconciliation holds the lock for that `Kafka` resource.
I propose a one-hour ceiling to catch mistakes of that size.
At that limit a continuously refused broker gets at most 65 checks in one roll.

## Implementation

### Passing the setting to KafkaRoller

Register a `ConfigParameter<Long>` in `ClusterOperatorConfig.CONFIG_VALUES` with the default string `"127750"` and a parser that checks the range after calling `LONG.parse`.
`KafkaReconciler.maybeRollKafka()` and `CaReconciler.createKafkaRoller()` both pass the same hardcoded `BackOff` supplier today.
Read the budget from `ClusterOperatorConfig` in each reconciler's constructor and pass it alongside that supplier when creating the roller. [KafkaReconciler](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaReconciler.java#L470-L491) · [CaReconciler](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/CaReconciler.java#L490-L504)

### Identifying the failed check

`KafkaRoller.canRoll()` currently combines broker availability and controller quorum into one boolean.
I would have this private method return `ALLOWED`, `BROKER_REFUSED` or `QUORUM_REFUSED`.
In `restartIfNecessary()`, set a broker-veto flag on the `UnforceableProblem` only when the plan requires a restart (`needsRestart`) and the result is `BROKER_REFUSED`.
On a combined-role node, run both checks in their existing order and give a quorum refusal precedence if both return `false`. [KRaft rolling rules](./060-kafka-roller-kraft.md)

A failed Kafka API request is an exception, and a dynamic-only update has `needsReconfig` set.
Both use the ordinary retry path, as do controller-only refusals and log recovery.
The existing force-roll caller accepts `ALLOWED` and retains its handling of SSL errors on ordinary attempts.

### Scheduling the next check

Keep the existing `BackOff` in each `RestartContext` and record the milliseconds actually passed to the scheduler for that pod.
The constructor already consumes the initial zero delay.
`schedule()` can use the following calculation to queue the next check or fail the pod's future. [Current scheduling loop](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/KafkaRoller.java#L298-L368)

```java
record Retry(long delayMs, boolean extraAttempt) {}

Optional<Retry> nextRetry(boolean brokerVeto, long budgetMs) {
    boolean extraAttempt = backOff.done();
    long remainingMs = budgetMs - scheduledMs;
    if ((extraAttempt && !brokerVeto) || (brokerVeto && remainingMs <= 0)) {
        return Optional.empty();
    }
    long delayMs = extraAttempt ? 64_000L : backOff.nextDelayMs();
    if (brokerVeto) {
        delayMs = Math.min(delayMs, remainingMs);
    }
    scheduledMs += delayMs;
    return Optional.of(new Retry(delayMs, extraAttempt));
}
```

Queue the returned delay on the existing single-threaded scheduler so another eligible pod can run during the wait.
All scheduled delays count, including an earlier retry caused by a different error on the same pod.
With `600000`, the first ten attempts consume 127,750 ms, followed by seven 64,000 ms waits and a final 24,250 ms wait.
Run the check after that final wait and restart the broker if it is safe.
If the check still refuses the restart, complete the pod's future exceptionally.
Count attempts as their tasks start so the exhaustion log reports the actual 18 attempts and 600,000 ms in this example.

The `extraAttempt` flag matters because `backOff.done()` is already true during the tenth ordinary attempt and also controls force-roll handling.
Pass the flag into `restartIfNecessary()` when re-reading the pod and its restart reasons.
On an extra attempt, `checkIfRestartOrReconfigureRequired()` must propagate a failed `getConfig()` call.
If the plan sets `forceRestart`, reject that plan and leave the pod running.
Propagate a caught `ForceableProblem` before the `canRoll(..., true, ...)` fallback.
This way an extra attempt can restart the pod only after the normal checks pass on that attempt. [Force-roll path](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/KafkaRoller.java#L432-L479) · [Failed configuration read](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/KafkaRoller.java#L608-L622)

## Testing

`KafkaRollerTest.TestingKafkaRoller` already supplies `BackOff(10, 2, 4)`, which schedules 10, 20 and 40 ms.
Give its existing tests a matching 70 ms budget and use the following cases to exercise the change. [KafkaRollerTest](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/test/java/io/strimzi/operator/cluster/operator/resource/KafkaRollerTest.java#L370-L439)

| Setup | Expected result |
| --- | --- |
| Drive `nextRetry` directly with the production `BackOff` | Nine waits ending in 64,000 ms at `127750`, or 17 waits ending in 24,250 ms at `600000` |
| Spend 10 ms retrying another error, then get a broker veto with a 25 ms budget in the small fixture | One 15 ms wait before exhaustion, with the full 25 ms still available to a second pod |
| In `testBrokerNotInitiallyRollable`, refuse broker 1 four times with a 71 ms budget and allow it on check five | One extra 1 ms delay and restart order `0, 2, 3, 4, 1` |
| In `testBrokerNeverRollable`, use 71 ms and refuse broker 1 on every check | Five checks, a failed future with `UnforceableProblem` and zero restarts of broker 1 |
| Use 71 ms for a combined follower and stub quorum and broker availability separately, with quorum refusing | Four checks, 70 ms of ordinary waits and zero restarts of that node |
| Allow 71 ms and throw `ForceableProblem` from `getConfig()` on the fifth attempt after four broker refusals | The roll fails and the restart list excludes that pod |

In `ClusterOperatorConfigTest`, check that omitted and empty values select `127750`, `600000` is accepted, and `0`, `3600001` and `10m` fail during parsing.

## Affected projects and compatibility

Document the variable in the Cluster Operator configuration reference and show Helm users how to set it through `extraEnvs`.
Existing installations get the current ten attempts on upgrade, and each new roll starts with fresh per-pod counters.
On downgrade, the current operator filters unknown environment variables in `ClusterOperatorConfig.buildFromMap()` and uses its hardcoded backoff.
The ordinary rolling update propagates a failed roll to reconciliation.
The manual rolling update logs the failure and continues reconciliation, as specified in [proposal 70](./070-dont-fail-reconciliation-in-manual-rolling-update.md).

## Rejected alternatives

- A field on the `Kafka` resource would let two clusters under the same operator give up at different times.
  I would start with the deployment setting because a fast cluster still proceeds on its first successful check, and this avoids adding a CRD field for the reported case.
  The trade-off is that an unhealthy cluster under that operator can also spend the full ten minutes in scheduled waits before reporting failure.
- Increasing the attempt count on the shared `BackOff` would also postpone the force-roll decision after an Admin API failure.
  The next two delays would be 128 and 256 seconds, leaving a broker that has just caught up waiting several minutes for its next check.
- Reusing `STRIMZI_OPERATION_TIMEOUT_MS` would make administrators lengthen pod-readiness timeouts to tolerate slow replica catch-up after a pod is already Ready.
