# Make the KafkaRoller retry window configurable

Let a Cluster Operator administrator set how long KafkaRoller retries when restarting a broker would take a partition below its minimum number of in-sync replicas.

## Current situation

KafkaRoller tracks retries per node during a roll.
The first attempt is immediate and the nine subsequent delays double from 250 ms to 64,000 ms, for a total of 127,750 ms between attempts.
Checks, pod operations and work on other nodes can make the elapsed time longer.
Both Kafka reconciliation and certificate authority reconciliation construct KafkaRoller with this policy. [KafkaReconciler](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/KafkaReconciler.java#L477-L488) · [CaReconciler](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/assembly/CaReconciler.java#L490-L504)

## Motivation

In the reported nine-broker cluster, starting a broker could take about five minutes, with further time needed for replicas to catch up.
During a roll, KafkaRoller could exhaust its retries for the next broker while the previous one was recovering normally.
A later reconciliation can continue the roll, so a cluster with consistently slow recovery can report repeated failures during an otherwise healthy update. [Enhancement #13058](https://github.com/strimzi/strimzi-kafka-operator/issues/13058) · [Request for a proposal](https://github.com/strimzi/strimzi-kafka-operator/issues/13058#issuecomment-7084468959)

## Proposal

Add `STRIMZI_KAFKA_ROLLER_AVAILABILITY_RETRY_BUDGET_MS` to the Cluster Operator deployment.
The value limits the cumulative backoff KafkaRoller may schedule for one broker in one roll when the broker availability check refuses a restart.
Use 127,750 ms by default and accept integers from 1 to 3,600,000 ms, rejecting invalid values at operator startup.
The proposed one-hour ceiling bounds scheduled waiting per broker when recovery stalls.
For example, `600000` allows up to ten minutes of scheduled delays while replicas on a previously restarted broker catch up.

On an explicit refusal from the broker availability check, subtract the delays already scheduled for that node from its budget.
If time remains, schedule another attempt using the existing exponential sequence, with each delay capped at 64,000 ms and at the remaining budget.
Run the check on that final attempt even when its delay uses the last millisecond of the budget.
If the check still refuses the restart, leave the broker running and fail the reconciliation.
Each retry re-evaluates availability, so a broker that becomes safe can roll on the next successful check.

Apply this rule to normal broker restarts, including restarts for certificate authority changes.
On a node with both roles, extending the wait requires a successful controller quorum check and a refusal from the broker availability check.
Use the existing attempt limit for other retry causes, including errors querying Kafka, controller-only refusals and dynamic reconfiguration alone.
Count earlier scheduled delays on the same node across changes of failure reason so that alternating errors cannot renew the budget. [KafkaRoller](https://github.com/strimzi/strimzi-kafka-operator/blob/ffd4971b64ee213a7ea77ce1c1b720575c37d244/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/KafkaRoller.java#L332-L367)

## Configuration scope and alternatives

Start with one setting per Cluster Operator because a higher ceiling only adds waiting for brokers whose checks continue to fail.
An operator managing both slow and fast clusters can therefore give a slow broker time to recover, although a genuinely unhealthy broker in any of those clusters can also take longer to report failure.
A per-cluster override would address that case at the cost of another user-facing setting.
Feedback from administrators who need different ceilings under one operator would help decide whether to include an override.

Using `STRIMZI_OPERATION_TIMEOUT_MS` would also couple waits for individual operations to replica recovery across attempts.
Exposing an attempt count would require administrators to convert their observed recovery times into an exponential delay sequence.
The separate availability recheck proposal concerns a delay after a successful check. [Proposal PR #253](https://github.com/strimzi/proposals/pull/253)

## Compatibility and testing

Omitting the setting selects today's ten-attempt schedule.
Document the range, units and example in the Cluster Operator configuration reference.
Unit tests should record scheduled delays without sleeping and verify the default nine delays, a shortened last delay, recovery after the tenth attempt and failure on exhaustion.
Also cover invalid configuration, mixed failure reasons and a controller quorum refusal on a combined-role node.
Exercise both KafkaRoller construction paths so that certificate-driven and ordinary rolls use the same configuration.
