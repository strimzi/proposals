# Controller readiness probe based on KRaft raft state

This proposal adds a new `/v1/controller-ready` endpoint to the Kafka Agent that reports whether a KRaft controller is attached to the metadata quorum, by reading the `current-state` attribute of the `kafka.server:type=raft-metrics` JMX MBean.
The controller-only branch of `kafka_readiness.sh` is updated to call this endpoint so that the existing readiness probe reflects raft attachment state rather than only whether the controller listener port is bound.

## Current situation

For Kafka pods in controller-only mode, the readiness probe used today (`docker-images/kafka-based/kafka/scripts/kafka_readiness.sh`) only checks that port 9090 (`CONTROLPLANE-9090`) is in `LISTEN` state via `netstat`:

```bash
netstat -lnt | grep -Eq 'tcp6?[[:space:]]+[0-9]+[[:space:]]+[0-9]+[[:space:]]+[^ ]+:9090.*LISTEN[[:space:]]*'
```

This succeeds the moment the listener binds, which happens before `ControllerRegistrationManager` registers the node with the rest of the quorum and well before the local raft state machine has converged to `leader` or `follower`.

A controller that wedges during startup — for example because its `NodeToController` channel manager times out while peer FQDNs are temporarily unresolvable (the cold-boot CoreDNS race observed in [issue #12760](https://github.com/strimzi/strimzi-kafka-operator/issues/12760)) — therefore stays at `1/1 Ready`. The kubelet never restarts it, the cluster operator's `KafkaRoller` reconciles indefinitely with `"An error while trying to determine the active controller"` because `describeMetadataQuorum` can't complete, and recovery requires a manual roll of each controller one-at-a-time with the leader last.

The Strimzi Cluster Operator already consumes JMX raft metrics elsewhere: the KRaft Grafana dashboards shipped under `packaging/examples/metrics/strimzi-metrics-reporter/grafana-dashboards/strimzi-kraft.json` graph `kafka_server_raft_metrics_current_state_info`, `kafka_server_raft_metrics_current_leader`, `kafka_server_raft_metrics_high_watermark`, etc. The data the probe needs to do better is already exposed and stable; it just isn't read by the agent.

## Motivation

A reliable controller readiness signal:

* Closes the silent-wedge failure mode in [strimzi-kafka-operator#12760](https://github.com/strimzi/strimzi-kafka-operator/issues/12760), so a controller that can't attach to the quorum is detected by the kubelet and restarted within `initialDelaySeconds + failureThreshold * periodSeconds` — by which point a transient cause (DNS propagation, network blip, restart of a peer) has typically resolved itself.
* Catches a broader class of unattached-controller failures than the original incident, including misconfigured TLS, peer partition, future Kafka regressions, and any other path that prevents the local raft layer from staying in `leader`/`follower`. The cost of the existing probe is paid not only on the specific incident filed.
* Aligns with the broker-side `/v1/ready` endpoint already on the Kafka Agent, which reports broker readiness based on the in-process `BrokerState` Yammer metric. Symmetric handling of broker and controller probes via the agent is more maintainable than the controller-side `netstat` fallback.

## Proposal

### Endpoint contract

Add a new endpoint to the Kafka Agent:

* `GET http://localhost:8080/v1/controller-ready`
  * **204 No Content** when the value of the `current-state` attribute on the `kafka.server:type=raft-metrics` JMX MBean is one of `{leader, follower}`. The local node is attached to the metadata quorum.
  * **503 Service Unavailable** for any other `current-state` value (`unattached`, `candidate`, `prospective`, `voted`, `resigned`, `observer`). The response body includes the actual state name as a JSON object for diagnostics, e.g. `{"error":"controller not ready, current raft state: prospective"}`.
  * **404 Not Found** when the `kafka.server:type=raft-metrics` MBean is not registered on the local JVM. This is expected on broker-only nodes (no controller process), and possible during the early-startup window before the raft layer initializes. Returning 404 (rather than 503) lets the calling script preserve the existing port-listening behavior in those cases.

The endpoint listens on the existing localhost HTTP port (`8080`) the Kafka Agent already uses for `/v1/ready` and `/v1/broker-state`. No new ports, no new RBAC, no new authentication surface — the MBean is read on the local JVM via `ManagementFactory.getPlatformMBeanServer()`.

### Script behavior

The controller-only branch of `kafka_readiness.sh` is updated to call the new endpoint after the existing port check:

```bash
if [[ "$roles" =~ "controller" ]] && [[ ! "$roles" =~ "broker" ]]; then
  netstat -lnt | grep -Eq 'tcp6?...:9090.*LISTEN'

  controller_status=$(curl -sS -o /dev/null -w "%{http_code}" --max-time 5 \
    http://localhost:8080/v1/controller-ready/ || echo "000")
  case "$controller_status" in
    204|404) ;;   # ready, or MBean not yet present — fall through
    *) exit 1 ;;
  esac
fi
```

`404` and `204` are both treated as ready. This keeps behavior identical to today for any deployment where the MBean is not registered for whatever reason (e.g. an older Kafka version, very early startup), so the change is strictly additive.

The broker-only and combined-mode branches of the script are unchanged.

### Implementation

* The `KafkaAgent` class gains a new `ContextHandler` registered for `/v1/controller-ready` alongside the existing `/v1/broker-state` and `/v1/ready` handlers. The handler factory returns `Handler.Abstract` that:
  * Reads the MBean attribute via `getPlatformMBeanServer().getAttribute(new ObjectName("kafka.server:type=raft-metrics"), "current-state")`.
  * Returns 204 / 503 / 404 per the contract above, with state name in the 503 response body.
* The MBean read is wrapped in a `Supplier<String>` field on the class. The production constructor wires this to a static `queryRaftCurrentStateFromJmx()` method. A test-only constructor accepts a custom `Supplier<String>` so unit tests can drive every raft state deterministically without standing up a real Kafka JVM — mirroring the existing pattern for broker `Gauge` injection.
* `RAFT_READY_STATES = Set.of("leader", "follower")` is held as a constant; new states added by future Kafka versions default to 503 (the safe direction).

### Unit testing

Strimzi's existing `KafkaAgentTest` is extended with one case per raft state observed in Kafka 4.x:

| Test | Injected `current-state` | Expected HTTP |
|---|---|---|
| `testControllerReadinessSuccessLeader` | `leader` | 204 |
| `testControllerReadinessSuccessFollower` | `follower` | 204 |
| `testControllerReadinessFailUnattached` | `unattached` (wedge state from the cold-boot DNS race) | 503 |
| `testControllerReadinessFailCandidate` | `candidate` | 503 |
| `testControllerReadinessFailResigned` | `resigned` | 503 |
| `testControllerReadinessFailObserver` | `observer` | 503 |
| `testControllerReadinessFailProspective` | `prospective` | 503 |
| `testControllerReadinessNotFound` | `null` (MBean absent) | 404 |

Pre-existing tests (`testReadinessSuccess`, `testReadinessFail`, `testReadinessFailWithBrokerUnknownState`, `testBrokerRunningState`, `testBrokerRecoveryState`, `testBrokerMetricNotFound`) continue to pass — regression coverage on the broker path.

### Reference implementation

A proof-of-concept implementation of the above is available at [strimzi-kafka-operator#12768](https://github.com/strimzi/strimzi-kafka-operator/pull/12768) (closed pending this proposal). It includes the agent endpoint, the script update, the unit test matrix, and CHANGELOG entry. It was validated end-to-end on both `kind` (Strimzi 0.51.0 + Kafka 4.2.0) and an AKS cluster, including a `hostAliases`-induced wedge that captured `HTTP 503 {"error":"controller not ready, current raft state: prospective"}` from the new endpoint — confirming the JMX read works and the endpoint distinguishes wedged from healthy in a real cluster. The PR can be re-opened (or re-raised on a fresh branch) once this proposal is accepted.

## Affected/not affected projects

* **Affected**: `kafka-agent` (new endpoint + Javadoc + tests); `docker-images/kafka-based/kafka/scripts/kafka_readiness.sh` (calling the new endpoint in the controller-only branch); `CHANGELOG.md`.
* **Not affected**: the Cluster Operator, the Topic Operator, the User Operator, the HTTP Bridge, the brokers' readiness path, RBAC for any operator role, CRD APIs, Helm chart values, public service contracts.

## Compatibility

* **Strictly additive on existing deployments.** The new endpoint is a new path; existing `/v1/ready` and `/v1/broker-state` are unchanged. The script's controller-only branch keeps the original `netstat` check and only adds the new endpoint call; if the endpoint returns 404 (e.g. because the agent jar shipped without the new handler, or the MBean isn't registered yet), behavior matches today exactly.
* **No CR schema changes.**
* **No new probe timing changes** — kubelet's default `initialDelaySeconds=15, periodSeconds=10, failureThreshold=3` is sufficient for the steady-state contract. Cluster operators that wish to tune may already do so via `spec.kafka.readinessProbe`.
* **Forward-compatibility with newer Kafka raft states.** Any value emitted by Kafka other than `leader`/`follower` is treated as not-ready, with the literal state name surfaced in the 503 body. If a future Kafka version adds, say, `bootstrap` as an additional valid steady state, that requires a follow-up here to extend `RAFT_READY_STATES`. The safe direction (default-not-ready) preserves correctness in the meantime.

## Open question — `ControllerRegistrationManager`-layer wedges

The proposal as written catches the broad class of failures where the local raft layer is not attached to the quorum. The on-disk evidence captured during the [strimzi-kafka-operator#12760](https://github.com/strimzi/strimzi-kafka-operator/issues/12760) incident — `quorum-state` with `leaderId: 3, leaderEpoch: 1` persistent for 8 hours — suggests raft *did* elect a leader during that wedge, and the actual failure was one layer above (in `ControllerRegistrationManager`). If that interpretation is correct, `current-state` would have been `leader` on the active controller and `follower` on the others during the wedge, and this proposal's check would have returned 204.

A second check covering this layer is possible — for example, "if `current-state` is `leader`/`follower` but the metadata-apply offset (`broker-metadata-metrics.last-applied-record-offset` on the broker side, or the equivalent controller-side metric) hasn't advanced in the last N seconds, return 503". That introduces stateful checking (compare-to-previous-probe) and a tuning parameter (N), and is best discussed as a follow-up to this proposal once the basic JMX-state probe lands.

I'd appreciate maintainer steer on whether to include this layer in the same proposal or treat it as a follow-up.

## Rejected alternatives

### File-based check on the on-disk `quorum-state` file

The on-disk `__cluster_metadata-0/quorum-state` JSON file is straightforward to read from a shell script and was the first approach considered. It was rejected because the `appliedOffset` field in that file is an internal raft state-machine offset that stays at `0` on healthy nodes too (it is not a metadata-apply offset). The file therefore cannot reliably distinguish a wedged controller from a healthy one without additional signals. The JMX `current-state` attribute is the correct signal at this layer and is API-stable.

### `AdminClient.describeMetadataQuorum` from a localhost probe

A probe that calls `describeMetadataQuorum` via a Kafka `AdminClient` would semantically be the most direct check ("am I serving admin RPCs?"). It was rejected because the `KafkaRoller` already issues that exact call from the Cluster Operator and that call hangs during the wedge described in the issue — the probe would inherit the same failure mode it is meant to detect.

### Skipping the agent and reading JMX directly from the readiness script

In principle, the readiness script could query JMX over the JMX-Prometheus exporter that already runs in each Kafka pod (it listens on port 9404), then `grep` the resulting Prometheus exposition for `kafka_server_raft_metrics_current_state_info`. This was rejected because (a) it depends on the metrics-config ConfigMap including the right exporter regex; (b) it tightly couples the readiness probe to the Prometheus exposition format; (c) the existing Kafka Agent already owns the broker-side equivalent (`/v1/ready` reading `BrokerState`) and adding the symmetric controller endpoint there is more cohesive.

### Adding the check to the controller broker-state path in the Cluster Operator

A different design would emit a broker-state-like field for controllers and have the operator gate readiness on it directly, bypassing kubelet's HTTP probe. This was rejected because (a) it requires changes in the Cluster Operator pod-template generation, expanding the surface of the change; (b) it loses the kubelet's automatic pod-restart behavior, which is the primary value of fixing the readiness probe; (c) the existing kafka-agent HTTP-probe pattern is the established Strimzi convention for this layer.
