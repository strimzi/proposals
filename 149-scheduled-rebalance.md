# Scheduled mode for auto-rebalance

Add a `schedule` mode to `spec.cruiseControl.autoRebalance` so the Cluster Operator can run `KafkaRebalance` on a recurring cron expression. Opt-in.

## Motivation

`autoRebalance` today has three trigger sources:

* manual `KafkaRebalance` CR;
* scaling (`add-brokers` / `remove-brokers`);
* detected imbalance (`imbalance` mode, proposal 136 / PR #211).

Time-driven prophylactic rebalances are still external: teams run a `CronJob` that templates `KafkaRebalance`, plus their own cleanup. This duplicates the template + finalizer + status machinery the other modes already provide. The proposal lifts it into the same list.

## Proposal

### `schedule` mode

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  cruiseControl:
    autoRebalance:
      - mode: schedule
        schedule: "0 3 * * SUN"
        timeZone: "Asia/Seoul"
        template:
          name: my-weekly-rebalance-template
```

* `schedule` — Quartz cron expression. Validated at admission.
* `timeZone` — IANA TZ name; defaults to UTC.
* `template` — same template-reference convention as `imbalance` / `add-brokers` / `remove-brokers`. Optional.

### Reconciliation

The Cluster Operator's existing `KafkaAutoRebalanceReconciler` adds one branch for `mode: schedule`:

1. Compute the next fire time from the cron expression each reconcile.
2. If `now >= nextFireTime` and no other auto-rebalance is running, generate `<cluster>-auto-rebalancing-schedule` from the template, with the `strimzi.io/auto-rebalancing` finalizer, mode `full`.
3. On completion, record outcome in `Kafka.status.autoRebalance`, remove the finalizer, delete the generated `KafkaRebalance`.
4. Re-arm.

The cron evaluation runs inside the existing reconcile loop. No separate scheduler thread. State (`lastScheduledRun`) survives operator restarts because it's derived from CR status.

### Interaction with other modes

Only one auto-rebalance runs at a time (existing rule).

* `schedule` + `imbalance`: if `imbalance` fires within `debounceWindow` (default `1h`) before the next scheduled tick, the scheduled run is skipped for that cycle.
* `schedule` + scaling: scheduled run defers until scaling completes. If still within `30m` of the nominal fire time, run; otherwise skip to next cron tick.

### Status

```yaml
status:
  autoRebalance:
    schedule:
      state: Idle              # Idle | Running | Failed | Skipped
      lastScheduledRun:        "2026-06-22T03:00:00Z"
      nextScheduledRun:        "2026-06-29T03:00:00Z"
      lastOutcome:             Success
      lastSkippedReason:       ""
```

### Out of scope

* Sub-minute granularity.
* Holiday / business-day calendars.
* Multiple `schedule` entries per `Kafka`.

## Alternatives considered

1. **Externalised cron** (`CronJob` + `kubectl apply`). Status + finalizer + template semantics get reinvented per team. Same argument that justified `imbalance` mode.
2. **Cruise Control anomaly detector tuned as a clock.** Abuses the anomaly framework; clock is not an anomaly.
3. **Embedded Quartz scheduler in the operator.** Adds a parallel scheduler thread that complicates leader election + state recovery; the reconcile loop is already a scheduler.

## Affected components

* `cluster-operator/` — new branch in `KafkaAutoRebalanceReconciler`, cron validation, status fields.
* `api/` — `KafkaAutoRebalanceMode` enum gains `schedule`; `KafkaAutoRebalance` gets `schedule` + `timeZone` fields.
* `documentation/` — section under "Auto-rebalancing".

## Compatibility

Additive. Users not configuring `mode: schedule` see no behaviour change. Existing modes are unaffected. New fields are additive in the CRD schema.

## References

* #136 / PR #211 — `imbalance` mode (sibling).
* #122 / PR #57 — original scaling auto-rebalance.
