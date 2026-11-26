# Make the KafkaRoller backoff configurable

## Current situation

Consider a partition with replicas on brokers 0, 1 and 2 and `min.insync.replicas=2`.
If broker 0 is Ready but still catching up, restarting broker 1 would leave only broker 2 in sync and stall writes with `acks=all`.
In the reported cluster, pod initialization alone could take five minutes before replica catch-up even started. [Issue #13058](https://github.com/strimzi/strimzi-kafka-operator/issues/13058)

`KafkaRoller` currently uses `new BackOff(250, 2, 10)` for each pod.
The first check is immediate and the nine scheduled waits double from 250 ms to 64 seconds, for a total of 127.75 seconds.
Elapsed time also includes work such as checking readiness and rolling other pods.
In the reporter’s log, broker 2 exhausted those waits 9 minutes and 19 seconds after its first deferral. [Incident log](https://github.com/strimzi/strimzi-kafka-operator/discussions/12650#discussioncomment-16608056)

## Proposal

I propose exposing the three inputs of the existing `BackOff` on the Cluster Operator deployment.
Each variable is optional and takes its own default if absent or empty.

| Environment variable | Default | Meaning |
| --- | --- | --- |
| `STRIMZI_KAFKA_ROLLER_BACKOFF_INITIAL_DELAY_MS` | `250` | Milliseconds before the first retry |
| `STRIMZI_KAFKA_ROLLER_BACKOFF_MULTIPLIER` | `2` | Integer factor applied to each successive wait (`1` gives a fixed interval) |
| `STRIMZI_KAFKA_ROLLER_BACKOFF_MAX_ATTEMPTS` | `10` | Total attempts per pod in one roll, including the first check (1 to 32) |

Setting only `STRIMZI_KAFKA_ROLLER_BACKOFF_INITIAL_DELAY_MS=1200` gives 613.2 seconds of scheduled waits over ten attempts.
The 307.2-second final wait means a replica could catch up five minutes before the roller checks again.
An operator who wants checks a minute apart for ten minutes can use `60000`, `1` and `11` respectively. [BackOff formula](https://github.com/strimzi/strimzi-kafka-operator/blob/268a06d6604a86db0e35ca72dd76c3dfe150add2/operator-common/src/main/java/io/strimzi/operator/common/BackOff.java#L88-L118)

The settings apply to the roller’s per-pod retries across all Kafka clusters managed by this Cluster Operator.
A broker availability refusal or a controller quorum refusal on the last attempt leaves the pod running.
The same counter also governs when the roller can consider a force restart after certain Kafka Admin API errors.
A longer schedule can postpone that decision, and a maximum of one attempt makes the first check the final one. [Restart decision](https://github.com/strimzi/strimzi-kafka-operator/blob/268a06d6604a86db0e35ca72dd76c3dfe150add2/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/KafkaRoller.java#L432-L479) · [KRaft rolling rules](./060-kafka-roller-kraft.md)

## Implementation

Register the delay as `strictlyPositive(LONG)` and the multiplier and attempt count as `strictlyPositive(INTEGER)` in `ClusterOperatorConfig`.
I propose the 32-attempt limit because the 33rd attempt overflows the `int` power in `BackOff.delay()` at the default multiplier of 2.
The same limit bounds the work in `totalDelayMs()` with a fixed interval.
After applying defaults, use checked arithmetic for the power (`int`), each delay and their sum (`long`), and reject invalid combinations at startup with `InvalidConfigurationException`. [BackOff arithmetic](https://github.com/strimzi/strimzi-kafka-operator/blob/268a06d6604a86db0e35ca72dd76c3dfe150add2/operator-common/src/main/java/io/strimzi/operator/common/BackOff.java#L105-L130)

Add `newKafkaRollerBackOff()` to `ClusterOperatorConfig` to return a fresh `new BackOff(initialDelayMs, multiplier, maxAttempts)`.
Use `config::newKafkaRollerBackOff` in both reconcilers to replace the hardcoded suppliers in `KafkaReconciler.maybeRollKafka()` and `CaReconciler.createKafkaRoller()`.
Each pod’s `RestartContext` then advances its own counter during ordinary and certificate rolls. [KafkaReconciler](https://github.com/strimzi/strimzi-kafka-operator/blob/268a06d6604a86db0e35ca72dd76c3dfe150add2/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaReconciler.java#L470-L491) · [CaReconciler](https://github.com/strimzi/strimzi-kafka-operator/blob/268a06d6604a86db0e35ca72dd76c3dfe150add2/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/CaReconciler.java#L490-L504) · [RestartContext](https://github.com/strimzi/strimzi-kafka-operator/blob/268a06d6604a86db0e35ca72dd76c3dfe150add2/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/KafkaRoller.java#L298-L363)

Document the variables and the Helm `extraEnvs` example in the Cluster Operator configuration reference.
An upgrade with no overrides or a downgrade to the previous operator uses the current ten-attempt schedule.
A failed ordinary roll fails the reconciliation, while a failed manual roll logs a warning and lets reconciliation continue. [Manual rolling updates](./070-dont-fail-reconciliation-in-manual-rolling-update.md)

## Testing

- In `ClusterOperatorConfigTest`, check absent and empty values, the 1,200 ms override and two backoffs from one configuration that can advance independently.
- Reject a zero delay, a fractional multiplier, 33 attempts, `(1, 3, 22)` where the power overflows and `(Long.MAX_VALUE, 1, 3)` where the sum of two valid waits overflows.
- Exercise the normal and CA roll construction paths with `(1, 2, 3)` in the operator configuration and one broker needing a restart.
  Two availability refusals followed by success should restart it once on check three, while three refusals should fail with `UnforceableProblem` and leave it running.
- With the same settings, make the config read fail on an otherwise rollable broker on every attempt and verify that the existing final-attempt handling restarts it only on the third check. [Existing roller cases](https://github.com/strimzi/strimzi-kafka-operator/blob/268a06d6604a86db0e35ca72dd76c3dfe150add2/cluster-operator/src/test/java/io/strimzi/operator/cluster/operator/resource/KafkaRollerTest.java#L370-L505)

## Alternatives

I initially proposed a time budget just for broker availability refusals.
Fitting an exact duration while keeping the scheduled waits short required capping the exponential delays, shortening the final wait and treating broker refusals differently from other retryable errors.
The report asks for more time for replicas to catch up, which the existing backoff can already provide.

A field on the `Kafka` resource would let clusters under one operator choose different schedules.
I would start with the deployment settings because the reported case can use them without a CRD change.
Reusing `STRIMZI_OPERATION_TIMEOUT_MS` would also change readiness timeouts for operators who only want to tune the intervals between checks.
