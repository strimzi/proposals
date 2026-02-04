# Dynamic KRaft Controller Quorum Management

This proposal introduces support for dynamically managing KRaft controller quorum membership in Strimzi, enabling operators to scale controller nodes without cluster downtime.

> [!NOTE]
> This proposal builds on recent Strimzi work including [PR #12261](https://github.com/strimzi/strimzi-kafka-operator/pull/12261) (controller quorum voters configuration) and addresses issues [#11613](https://github.com/strimzi/strimzi-kafka-operator/issues/11613) (quorum instability) and [#11477](https://github.com/strimzi/strimzi-kafka-operator/issues/11477) (dual role migration).

## Current situation

Strimzi currently supports KRaft mode for Kafka clusters, eliminating the need for ZooKeeper. However, the **controller quorum is configured statically**:

1. **Static `controller.quorum.voters`**: The list of controller voters is set at cluster creation and stored in each controller's configuration
2. **No controller-only rolling updates**: Strimzi does not support rolling updates for nodes designated only as controllers
3. **Scaling limitations**: Node pools with controller roles cannot be scaled up or down
4. **Combined mode constraints**: Clusters using combined mode (controller + broker on same nodes) can only scale if a separate broker-only node pool exists

When operators need to change the controller quorum (e.g., scaling from 3 to 5 controllers for improved fault tolerance), they must:
1. Create a new cluster with the desired configuration
2. Migrate data using MirrorMaker 2
3. Switch traffic to the new cluster

This process is operationally expensive and introduces unnecessary risk and downtime.

## Motivation

### Business Drivers

| Scenario | Impact of Static Quorum |
|----------|-------------------------|
| **Scaling for HA** | Cannot add controllers for improved fault tolerance without cluster rebuild |
| **Hardware maintenance** | Cannot replace failed controller nodes gracefully |
| **Cost optimization** | Cannot reduce controllers during low-usage periods |
| **Cloud migration** | Cannot redistribute controllers across new availability zones |

### Kafka 3.9+ Enables Dynamic Quorums

**KIP-853** introduces dynamic reconfiguration of `controller.quorum.voters`:

- **Voter list stored in metadata log**: Changes replicate automatically to all controllers
- **New configuration**: `controller.quorum.bootstrap.servers` replaces static voters list
- **Operational safety**: Only one controller add/remove at a time to maintain quorum majority
- **Tooling**: `kafka-metadata-quorum.sh` provides add/remove commands

Strimzi must leverage KIP-853 to provide operational flexibility for KRaft deployments.

### Alignment with Strimzi Roadmap

This proposal aligns with:
- **Node Pools maturity**: Extends node pool capabilities to support controller scaling
- **KRaft as default**: Improves operational experience as ZooKeeper is phased out (UseKRaft feature gate is now GA)
- **Enterprise readiness**: Addresses critical operational gap for production deployments
- **Recent infrastructure work**: Leverages `controller.quorum.bootstrap.servers` configuration now properly handled in Strimzi (PR #12261)

## Proposal

### Overview

Add support for dynamic KRaft controller quorum management through:
1. **Feature gate** to enable dynamic quorum mode (following `UseKRaft` graduation pattern)
2. **New cluster provisioning** with dynamic quorum configuration
3. **KafkaNodePool integration** for controller scaling operations (extending `KafkaCluster.controllerNodes()` logic)
4. **Status reporting** for quorum visibility

### Feature Gate

A new feature gate controls dynamic quorum support:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strimzi-cluster-operator
spec:
  template:
    spec:
      containers:
        - name: strimzi-cluster-operator
          env:
            - name: STRIMZI_FEATURE_GATES
              value: "+DynamicControllerQuorum"
```

| Gate | Default | Graduation |
|------|---------|------------|
| `DynamicControllerQuorum` | `disabled` | Alpha → Beta → GA |

### Requirements

Dynamic quorum management requires:
- **Kafka version**: ≥ 3.9.0
- **KRaft feature version**: ≥ 1 (checked automatically by operator)

### New Cluster Provisioning

When `+DynamicControllerQuorum` is enabled, new KRaft clusters are provisioned with dynamic quorum:

#### Configuration Changes

| Static Quorum (Current) | Dynamic Quorum (Proposed) |
|-------------------------|---------------------------|
| `controller.quorum.voters=1@host1:9093,2@host2:9093,3@host3:9093` | `controller.quorum.bootstrap.servers=host1:9093,host2:9093,host3:9093` |

#### Bootstrap Process

1. Operator formats storage for first controller with single voter
2. First controller starts and becomes leader
3. Operator adds remaining controllers one-by-one via `kafka-metadata-quorum.sh --add-controller`
4. Each new controller joins as observer, then promoted to voter

```
╔════════════════════════════════════════════════════════════════════════════╗
║                     Dynamic Quorum Bootstrap Flow                          ║
╠════════════════════════════════════════════════════════════════════════════╣
║  1. Format first controller                                                ║
║     └─> voter: [controller-0]                                              ║
║                                                                            ║
║  2. Start controller-0 (single-node quorum)                                ║
║     └─> voter: [controller-0], leader: controller-0                        ║
║                                                                            ║
║  3. Add controller-1                                                       ║
║     └─> voter: [controller-0, controller-1]                                ║
║                                                                            ║
║  4. Add controller-2                                                       ║
║     └─> voter: [controller-0, controller-1, controller-2]                  ║
║                     ✓ Quorum ready                                         ║
╚════════════════════════════════════════════════════════════════════════════╝
```

### KafkaNodePool Integration

#### Scaling Up Controllers

When a `KafkaNodePool` with controller role is scaled up:

1. Operator detects replica count increase via `KafkaCluster.controllerNodes()` delta
2. New controller pod is created and started as observer
3. Operator waits for pod to be ready and caught up (using existing `RollingUpdateConfiguration`)
4. Operator calls `kafka-metadata-quorum.sh --add-controller <node-id>@<host>:<port>`
5. New controller promoted to voter
6. `KafkaNodePool.status` updated with new quorum membership
7. Controller added to `controller.quorum.bootstrap.servers` for future broker connections

#### Scaling Down Controllers

When a `KafkaNodePool` with controller role is scaled down:

1. Operator validates scale-down is safe (maintains quorum majority)
2. Operator calls `kafka-metadata-quorum.sh --remove-controller <node-id>`
3. Target controller demoted from voter to observer
4. Pod is deleted after demotion confirmed
5. `KafkaNodePool.status` updated

#### Safety Constraints

The operator enforces KIP-853 safety rules:

| Constraint | Enforcement |
|------------|-------------|
| One change at a time | Queue controller add/remove operations (sequential reconciliation) |
| Maintain quorum majority | Block scale-down that would lose majority (n/2 + 1 minimum) |
| Wait for catch-up | New controllers must sync before voting (observer → voter transition) |
| Avoid fencing issues | Respect `broker.session.timeout.ms` during rolling operations ([#11736](https://github.com/strimzi/strimzi-kafka-operator/issues/11736)) |

### Quorum Transition Behavior

```
                    Scale-up: 3 → 5 controllers
╔══════════════════════════════════════════════════════════════════════════╗
║ Time    Voters                              Action                       ║
╠══════════════════════════════════════════════════════════════════════════╣
║ T+0     [0, 1, 2]                           Detect scale-up to 5         ║
║ T+1     [0, 1, 2]                           Start controller-3 (observer)║
║ T+2     [0, 1, 2]                           controller-3 syncs metadata  ║
║ T+3     [0, 1, 2, 3]                        Add controller-3 as voter    ║
║ T+4     [0, 1, 2, 3]                        Start controller-4 (observer)║
║ T+5     [0, 1, 2, 3]                        controller-4 syncs metadata  ║
║ T+6     [0, 1, 2, 3, 4]                     Add controller-4 as voter    ║
║         ✓ Scale-up complete                                              ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### Status Reporting

New status fields provide visibility into quorum state:

#### Kafka.status.kraft

```yaml
status:
  kraft:
    quorumState: Stable  # Stable | Reconfiguring | Degraded
    quorumVoters:
      - nodeId: 0
        host: my-cluster-controllers-0.my-cluster-controllers.kafka.svc
        port: 9093
        state: Leader
      - nodeId: 1
        host: my-cluster-controllers-1.my-cluster-controllers.kafka.svc
        port: 9093
        state: Follower
      - nodeId: 2
        host: my-cluster-controllers-2.my-cluster-controllers.kafka.svc
        port: 9093
        state: Follower
    observers:
      - nodeId: 3
        host: my-cluster-controllers-3.my-cluster-controllers.kafka.svc
        port: 9093
        syncStatus: CaughtUp
```

#### KafkaNodePool.status

```yaml
status:
  roles:
    - controller
  replicas: 3
  readyReplicas: 3
  quorumMembership:
    - nodeId: 0
      votingStatus: Voter
    - nodeId: 1
      votingStatus: Voter
    - nodeId: 2
      votingStatus: Voter
```

### Migration Path

> [!IMPORTANT]
> **Existing static quorum clusters cannot be converted to dynamic quorum in-place.** This is a Kafka limitation (KIP-853), not a Strimzi limitation.

#### Recommended Migration

For clusters requiring dynamic quorum:

1. Deploy new cluster with `+DynamicControllerQuorum` enabled
2. Configure MirrorMaker 2 for data replication
3. Migrate consumers and producers to new cluster
4. Decommission old cluster

#### Documentation Requirements

Strimzi documentation must clearly state:
- Dynamic quorum is for **new clusters only**
- Existing clusters retain static quorum behavior
- Migration requires data replication, not in-place upgrade

## Affected/not affected projects

### Affected

| Project | Changes |
|---------|---------|
| `cluster-operator` | KafkaAssemblyOperator, NodePoolController, KRaft quorum management |
| `api` | New status fields for quorum visibility |
| `documentation` | Feature gate docs, migration guide, operational procedures |
| `helm-charts` | Feature gate configuration options |
| `systemtest` | Integration tests for quorum scaling |

### Not Affected

| Project | Reason |
|---------|--------|
| `topic-operator` | No interaction with controller quorum |
| `user-operator` | No interaction with controller quorum |
| `kafka-bridge` | Data plane component, no controller interaction |
| `drain-cleaner` | Works with existing pod eviction logic |

## Compatibility

### Backwards Compatibility

- **Opt-in feature gate**: Existing clusters unaffected
- **No CRD breaking changes**: New status fields are additive
- **Static quorum still supported**: Clusters without feature gate retain current behavior

### Kafka Version Requirements

| Kafka Version | Dynamic Quorum Support |
|---------------|------------------------|
| < 3.9.0 | Not supported (operator rejects) |
| ≥ 3.9.0 | Supported with KRaft feature version ≥ 1 |

### Forward Compatibility

- **Feature gate graduation**: Plan for Beta (stable API) and GA (default enabled)
- **Status field stability**: New status fields follow Kubernetes API conventions

### Upgrade Path

| Scenario | Behavior |
|----------|----------|
| Upgrade Strimzi, keep static quorum | No change, static quorum preserved |
| Enable `+DynamicControllerQuorum` on existing cluster | Operator logs warning, no effect |
| New cluster with `+DynamicControllerQuorum` | Dynamic quorum provisioned |

## Rejected alternatives

### 1. Custom CRD for Quorum Management

**Proposal**: Introduce `KafkaControllerQuorum` CRD for explicit quorum configuration.

**Rejected because**:
- Adds unnecessary CRD complexity
- Quorum is inherently tied to `KafkaNodePool` with controller role
- Violates principle of declarative node pool management

### 2. Automatic Static-to-Dynamic Migration

**Proposal**: Operator automatically converts existing static quorum to dynamic.

**Rejected because**:
- Kafka does not support this conversion (KIP-853 limitation)
- Would require cluster rebuild, misleading users about "automatic" behavior
- High risk of data loss if implemented incorrectly

### 3. Manual Quorum Management via Annotations

**Proposal**: Users annotate pods to trigger quorum add/remove operations.

**Rejected because**:
- Error-prone manual process
- Violates declarative Kubernetes patterns
- Inconsistent with Strimzi's automation philosophy

### 4. External Tooling Only (No Operator Integration)

**Proposal**: Document manual use of `kafka-metadata-quorum.sh` without operator support.

**Rejected because**:
- Bypasses operator's desired-state reconciliation
- Race conditions between operator and manual changes
- Poor user experience for core operational need

## References

### Kafka Enhancement Proposals

- [KIP-853: KRaft Controller Membership Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes) — Core spec for dynamic quorum
- [KIP-966: Eligible Leader Replicas](https://cwiki.apache.org/confluence/display/KAFKA/KIP-966%3A+Eligible+Leader+Replicas) — Related HA improvements
- [KIP-996: Pre-Vote](https://cwiki.apache.org/confluence/display/KAFKA/KIP-996%3A+Pre-Vote) — Quorum stability enhancement

### Strimzi Related Work

- [PR #12261](https://github.com/strimzi/strimzi-kafka-operator/pull/12261) — Controller quorum voters configuration fix
- [PR #11505](https://github.com/strimzi/strimzi-kafka-operator/pull/11505) — Docs clarifying static quorum limits
- [Issue #11613](https://github.com/strimzi/strimzi-kafka-operator/issues/11613) — Active controller instability
- [Issue #11477](https://github.com/strimzi/strimzi-kafka-operator/issues/11477) — Dual role migration broker unregistration
- [Issue #11736](https://github.com/strimzi/strimzi-kafka-operator/issues/11736) — Broker fencing during rolling restarts
- [Issue #11179](https://github.com/strimzi/strimzi-kafka-operator/issues/11179) — KRaft quorum state failures

