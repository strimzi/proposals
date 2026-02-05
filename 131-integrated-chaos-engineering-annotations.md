# Integrated Chaos Engineering Annotations for Strimzi

This proposal introduces native chaos engineering integration for Strimzi-managed Kafka clusters, enabling seamless compatibility with LitmusChaos, Chaos Mesh, and other cloud-native chaos tools.

## Current situation

Strimzi provides comprehensive label management for Kafka resources through the `Labels` class. Current labels include:

| Label | Purpose |
|-------|---------|
| `strimzi.io/cluster` | Cluster name association |
| `strimzi.io/kind` | CRD type (Kafka, KafkaConnect, etc.) |
| `strimzi.io/name` | Component name |
| `strimzi.io/component-type` | Component type (kafka, zookeeper, entity-operator) |
| `strimzi.io/pool-name` | Node pool name (KRaft) |
| `strimzi.io/broker-role` | Whether pod has broker role (true/false) |
| `strimzi.io/controller-role` | Whether pod has controller role (true/false) |
| `strimzi.io/controller` | Controller type (strimzipodset) |

However, users attempting to integrate chaos engineering tools face significant friction:

1. **StrimziPodSet Targeting Failure**: LitmusChaos `appinfo.appkind` expects `deployment`, `statefulset`, or `daemonset`. The `strimzipodset` controller type is not recognized, causing target selection failures.

2. **Manual Label Discovery**: Operators must manually inspect pods to find correct label combinations for chaos selectors.

3. **Controller Safety Gap**: No guidance exists to prevent accidental deletion of KRaft controller nodes, which can cause quorum loss.

4. **No Opt-in Signal**: Strimzi does not inject the `litmuschaos.io/chaos` annotation required when `annotationCheck: true` is configured.

## Motivation

Chaos engineering has become essential for validating Kafka cluster resilience. Organizations need to verify:

- **Broker failure recovery**: Pod deletion and recreation by StrimziPodSet
- **Network partition handling**: ISR shrinkage and leader election
- **Resource exhaustion**: CPU/memory stress impact on throughput
- **Storage failures**: Disk I/O delays and corruption scenarios

Current workarounds require deep knowledge of both Strimzi internals and chaos tool configurations:

```yaml
# Current workaround for LitmusChaos
spec:
  annotationCheck: 'false'
  # appinfo: {}  # Must be removed entirely
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: APP_NS
              value: 'kafka'
            - name: APP_LABEL
              value: 'strimzi.io/component-type=kafka'  # Manual discovery
```

This proposal eliminates these friction points through documentation, optional convenience labels, and status visibility.

## Proposal

### Overview

This proposal introduces three complementary enhancements:

1. **Convenience Labels**: New `chaos.strimzi.io/*` labels for simplified targeting
2. **Chaos Readiness Status**: Status field indicating cluster suitability for chaos experiments
3. **Integration Documentation**: Ready-to-use configurations for LitmusChaos and Chaos Mesh

### New Convenience Labels

Add chaos-oriented labels using the `chaos.strimzi.io/` prefix (avoiding the blocked `strimzi.io/*` namespace):

| Label | Values | Purpose |
|-------|--------|---------|
| `chaos.strimzi.io/target` | `true`, `false` | Indicates pod is a valid chaos target |
| `chaos.strimzi.io/safe` | `true`, `false` | Indicates pod can be safely disrupted |

These labels will be automatically applied to Kafka broker pods:

```
+---------------------------------------+------------------+---------------------+
| Label                                 | Broker-only Pod  | Controller-only Pod |
+---------------------------------------+------------------+---------------------+
| chaos.strimzi.io/target               | true             | true                |
| chaos.strimzi.io/safe                 | true             | false               |
+---------------------------------------+------------------+---------------------+
```

**Implementation in Labels.java**:

```java
public static final String CHAOS_DOMAIN = "chaos.strimzi.io/";
public static final String CHAOS_TARGET_LABEL = CHAOS_DOMAIN + "target";
public static final String CHAOS_SAFE_LABEL = CHAOS_DOMAIN + "safe";

public Labels withChaosTarget(boolean isTarget) {
    return with(CHAOS_TARGET_LABEL, isTarget ? "true" : "false");
}

public Labels withChaosSafe(boolean isSafe) {
    return with(CHAOS_SAFE_LABEL, isSafe ? "true" : "false");
}
```

### Chaos Readiness Status

Add a new `chaos` section to `Kafka.status` providing visibility into cluster chaos readiness:

```yaml
status:
  chaos:
    safeForChaos: true
    reason: "3/3 brokers ready, all partitions have sufficient ISR"
    lastEvaluated: "2026-02-05T01:30:00Z"
```

The `ChaosReadinessStatus` will evaluate:

- Minimum broker availability (configurable threshold)
- Partition ISR health
- Ongoing reconciliation status
- Controller quorum health (KRaft)

### Chaos Tool Integration Examples

#### LitmusChaos (Recommended Configuration)

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: kafka-broker-chaos
  namespace: kafka
spec:
  annotationCheck: 'false'
  engineState: active
  chaosServiceAccount: litmus-admin
  # NOTE: No appinfo block - required for StrimziPodSet compatibility
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TARGET_PODS_NAMESPACE
              value: kafka
            - name: APP_LABEL
              # Target brokers only, exclude controllers for safety
              value: 'strimzi.io/component-type=kafka,strimzi.io/broker-role=true,strimzi.io/controller-role=false'
            - name: PODS_AFFECTED_PERC
              value: '50'
            - name: FORCE
              value: 'false'
            - name: TOTAL_CHAOS_DURATION
              value: '60'
```

#### Chaos Mesh (Recommended Configuration)

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kafka-pod-failure
  namespace: kafka
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - kafka
    labelSelectors:
      strimzi.io/cluster: my-cluster
      strimzi.io/component-type: kafka
      strimzi.io/broker-role: "true"
      strimzi.io/controller-role: "false"
  duration: "60s"
```

#### Pool-Specific Targeting

Target a specific node pool for isolated testing:

```yaml
# LitmusChaos
- name: APP_LABEL
  value: 'strimzi.io/cluster=my-cluster,strimzi.io/pool-name=pool-alpha'

# Chaos Mesh
labelSelectors:
  strimzi.io/cluster: my-cluster
  strimzi.io/pool-name: pool-alpha
```

### User-Defined Chaos Annotations

Users can opt-in to chaos tool annotations via the existing template mechanism:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    template:
      pod:
        metadata:
          annotations:
            litmuschaos.io/chaos: "true"
            chaos-mesh.org/inject: "true"
```

## Affected projects

This proposal affects:

| Project | Changes |
|---------|---------|
| `operator-common` | `Labels.java` - add `chaos.strimzi.io/*` label constants and methods |
| `api` | `KafkaStatus.java` - add `ChaosReadinessStatus` class |
| `cluster-operator` | `KafkaCluster.java`, `KafkaPool.java` - emit chaos labels on pods |
| `documentation` | New chaos engineering integration guide |

## Compatibility

### Backward Compatibility

- **No breaking changes**: New labels are additive
- **Existing selectors**: Continue to work unchanged
- **Template annotations**: Already supported for user-defined annotations

### Forward Compatibility

- **Chaos tool updates**: Label-based targeting is stable across tool versions
- **New chaos tools**: Standard Kubernetes label selectors are universally supported

### Chaos Tool Compatibility

| Tool | Version | Compatibility |
|------|---------|---------------|
| LitmusChaos | 3.x | ✅ Verified |
| Chaos Mesh | 2.x | ✅ Verified |
| Gremlin | 3.x | ✅ Expected (label-based) |
| AWS FIS | Current | ✅ Expected (tag-based) |

## Rejected alternatives

### 1. Add `spec.chaos.enabled` CRD Field

**Rejected**: Existing labels already contain all targeting information. A new CRD field adds API surface area without clear benefit and requires schema migration.

### 2. Use `strimzi.io/chaos-*` Labels

**Rejected**: The `Labels.java` validation explicitly blocks user-defined labels with `strimzi.io/*` prefix (exception thrown at line 131). Using `chaos.strimzi.io/*` prefix avoids this constraint.

### 3. Automatically Inject `litmuschaos.io/chaos` Annotation

**Rejected**: Opt-in annotation should remain user-controlled per security principles. Users can already add via `template.pod.metadata.annotations`.

### 4. External Mutating Admission Controller

**Rejected**: Adds operational complexity. Labels and status should come from the authoritative source (Strimzi operator) rather than external components.

### 5. Support `appkind: strimzipodset` in LitmusChaos

**Rejected**: Would require upstream changes to LitmusChaos. Pure label targeting with `APP_LABEL` is the recommended approach and works today.
