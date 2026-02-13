# OpenTelemetry Tracing for Strimzi Operator Reconciliation Loops

This proposal introduces native OpenTelemetry distributed tracing support for Strimzi operators (Cluster Operator, Topic Operator, and User Operator) to provide visibility into reconciliation performance and enable root-cause analysis of operational issues.

## Current situation

Strimzi currently supports distributed tracing for **Kafka components** (data plane):
- **Kafka Connect**: Traces messages consumed and produced by connectors
- **MirrorMaker 2**: Traces messages from source to target clusters
- **Kafka Bridge**: Traces HTTP-to-Kafka message flows

This is enabled via `spec.tracing.type: opentelemetry` in the respective custom resources, using the OTLP protocol by default.

However, **Strimzi operators** (control plane) do **not** support distributed tracing:
- **Cluster Operator**: No visibility into Kafka/Connect/Bridge reconciliation phases
- **Topic Operator**: No visibility into topic creation, configuration, or deletion latency
- **User Operator**: No visibility into user creation, credential generation, or ACL application

Current debugging relies on:
1. Log analysis (verbose but unstructured)
2. Prometheus metrics (aggregated, not request-scoped)
3. Kubernetes events (coarse-grained)

## Motivation

### Problem Statement

Operators often experience reconciliation failures or slowdowns that are difficult to diagnose:

1. **Slow Reconciliations**: Why did a Kafka reconciliation take 15 minutes instead of 2 minutes?
2. **Hidden Latency**: Which phase (rolling update, certificate renewal, Cruise Control call) caused delays?
3. **Cross-Component Visibility**: How do changes to `Kafka` CR propagate through to `KafkaNodePool`, `KafkaTopic`, and `KafkaUser` resources?
4. **Production Debugging**: When a reconciliation fails, what was the exact sequence of operations?

### Value Proposition

| Benefit | Description |
|---------|-------------|
| **Root Cause Analysis** | Identify exact phase causing reconciliation failures or slowdowns |
| **Performance Profiling** | Measure duration of rolling updates, CA renewals, topic operations |
| **Cross-Resource Correlation** | Trace how `Kafka` changes affect dependent resources |
| **SRE Observability** | Integrate with existing enterprise tracing infrastructure (Jaeger, Grafana Tempo, Datadog) |
| **Kafka 4.0 Alignment** | Complements KIP-938 (KRaft Performance Metrics) and KIP-1076 (Client Metrics) |

### Industry Alignment

OpenTelemetry is the CNCF standard for observability. Major Kubernetes operators already support tracing:
- **ArgoCD**: Built-in OTLP tracing for GitOps sync operations
- **Flux**: OpenTelemetry instrumentation for reconciliation loops
- **Crossplane**: Tracing for provider reconciliations

## Proposal

### Overview

Add optional OpenTelemetry tracing instrumentation to all Strimzi operators, with:
1. **OTLP export** by default (compatible with Jaeger, Grafana Tempo, etc.)
2. **Configurable via environment variables** (following OTel conventions)
3. **Opt-in feature gate** to minimize impact on existing deployments
4. **Semantic span naming** following OpenTelemetry conventions

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Strimzi Cluster Operator                         │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐       │
│  │ Kafka Assembler │   │ Connect Assembler│   │ Bridge Assembler│       │
│  │   [Traced]      │   │    [Traced]      │   │    [Traced]     │       │
│  └────────┬────────┘   └────────┬─────────┘   └────────┬────────┘       │
│           │                     │                      │                │
│  ┌────────▼─────────────────────▼──────────────────────▼───────┐        │
│  │                    OpenTelemetry SDK                        │        │
│  │  - Span creation for each reconciliation phase              │        │
│  │  - Context propagation between operators                    │        │
│  │  - Resource attributes (cluster name, namespace, version)   │        │
│  └────────────────────────────────┬────────────────────────────┘        │
└───────────────────────────────────┼─────────────────────────────────────┘
                                    │ OTLP/gRPC or OTLP/HTTP
                                    ▼
                    ┌──────────────────────────────────┐
                    │   OpenTelemetry Collector        │
                    │   (or direct to Jaeger/Tempo)    │
                    └──────────────────────────────────┘
```

### Span Hierarchy

Each reconciliation will create a trace with the following span structure:

```
ReconcileKafka (root span)
├── ValidateResource
├── ReconcileNodePools
│   ├── ReconcileNodePool[pool-a]
│   └── ReconcileNodePool[pool-b]
├── ReconcileBrokers
│   ├── GenerateConfigs
│   ├── CreateOrUpdateStatefulSets → CreateOrUpdateStrimziPodSets
│   └── RollingUpdate
│       ├── RollPod[my-cluster-kafka-0]
│       ├── RollPod[my-cluster-kafka-1]
│       └── RollPod[my-cluster-kafka-2]
├── ReconcileCruiseControl
│   └── CallCruiseControlAPI
├── ReconcileCertificates
│   ├── CheckCAExpiry
│   └── RenewCertificates
├── ReconcileUsers (propagated to User Operator)
└── ReconcileTopics (propagated to Topic Operator)
```

### Configuration

#### Environment Variables (Cluster Operator Deployment)

Following OpenTelemetry conventions:

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
            # Enable tracing (opt-in)
            - name: STRIMZI_TRACING_ENABLED
              value: "true"
            # OpenTelemetry service name
            - name: OTEL_SERVICE_NAME
              value: "strimzi-cluster-operator"
            # OTLP endpoint (gRPC by default)
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://jaeger-collector.observability:4317"
            # Optional: Export protocol (grpc or http/protobuf)
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "grpc"
            # Optional: Sampling ratio (0.0 to 1.0)
            - name: OTEL_TRACES_SAMPLER
              value: "parentbased_traceidratio"
            - name: OTEL_TRACES_SAMPLER_ARG
              value: "0.1"
```

#### Feature Gate

Tracing will be controlled by a new feature gate:

```yaml
env:
  - name: STRIMZI_FEATURE_GATES
    value: "+OperatorTracing"
```

| Gate | Default | Description |
|------|---------|-------------|
| `OperatorTracing` | `disabled` | Enable OpenTelemetry tracing for operators |

### Span Attributes

Each span will include semantic attributes following OpenTelemetry conventions:

| Attribute | Example Value | Description |
|-----------|---------------|-------------|
| `strimzi.resource.kind` | `Kafka` | Kind of resource being reconciled |
| `strimzi.resource.name` | `my-cluster` | Name of the resource |
| `strimzi.resource.namespace` | `kafka` | Namespace of the resource |
| `strimzi.resource.generation` | `5` | Generation of the resource |
| `strimzi.operator.version` | `0.51.0` | Strimzi operator version |
| `strimzi.kafka.version` | `4.1.1` | Kafka version being managed |
| `strimzi.reconciliation.trigger` | `periodic`, `watch`, `manual` | What triggered the reconciliation |

### Error Handling

Failed reconciliations will:
1. Set span status to `ERROR`
2. Record exception details via `span.recordException()`
3. Add `error.type` and `error.message` attributes

```java
try {
    reconcileKafka(kafka);
} catch (Exception e) {
    span.setStatus(StatusCode.ERROR, e.getMessage());
    span.recordException(e);
    throw e;
}
```

### Implementation Phases

#### Phase 1: Cluster Operator (Core)
- Add OpenTelemetry SDK dependency (`io.opentelemetry:opentelemetry-sdk`)
- Instrument `KafkaAssemblyOperator` reconciliation loop
- Add spans for major phases: validation, node pools, brokers, rolling updates
- Feature gate and environment variable configuration

#### Phase 2: Topic and User Operators
- Instrument `TopicOperator` batch processing
- Instrument `UserOperator` credential and ACL operations
- Context propagation from Cluster Operator via CR annotations

#### Phase 3: Cross-Operator Correlation
- Propagate trace context through `strimzi.io/trace-context` annotation
- Enable end-to-end traces from `Kafka` CR change to `KafkaUser` credential update

### Example Trace Visualization

In Jaeger/Grafana Tempo, a typical trace would show:

```
Trace ID: abc123...
Duration: 4m 32s

[4m 32s] ReconcileKafka (my-cluster)
    ├── [120ms] ValidateResource
    ├── [1.2s] ReconcileNodePools
    │   ├── [600ms] ReconcileNodePool (pool-a)
    │   └── [580ms] ReconcileNodePool (pool-b)
    ├── [3m 45s] ReconcileBrokers          ← Bottleneck identified!
    │   ├── [200ms] GenerateConfigs
    │   └── [3m 44s] RollingUpdate
    │       ├── [45s] RollPod (my-cluster-kafka-0)
    │       ├── [48s] RollPod (my-cluster-kafka-1)   ← Slow restart
    │       └── [2m 10s] RollPod (my-cluster-kafka-2) ← Very slow!
    ├── [12s] ReconcileCruiseControl
    └── [8s] ReconcileCertificates
```

## Affected/not affected projects

### Affected

| Project | Changes |
|---------|---------|
| `strimzi-kafka-operator` | Core implementation in Cluster, Topic, and User Operators |
| `strimzi-kafka-operator` (docs) | Documentation for enabling and configuring tracing |
| Helm charts | Add environment variable templates for tracing configuration |

### Not Affected

| Project | Reason |
|---------|--------|
| `strimzi-kafka-bridge` | Already has OpenTelemetry support |
| `strimzi-kafka-oauth` | Out of scope (authentication plugin) |
| `strimzi-drain-cleaner` | Minimal reconciliation, tracing not beneficial |
| `metrics-reporter` | Metrics-focused, not tracing |

## Compatibility

### Backwards Compatibility

- **Opt-in by default**: Tracing is disabled unless explicitly enabled via feature gate
- **No CRD changes**: Configuration via environment variables only
- **No behavioral changes**: Operators function identically with tracing disabled
- **Graceful degradation**: If OTLP endpoint is unavailable, tracing fails silently

### Forward Compatibility

- **OpenTelemetry SDK versioning**: Use BOM for consistent dependency versions
- **Semantic conventions**: Follow stable OTel semantic conventions to avoid breaking changes
- **Feature gate graduation**: Plan for Beta → GA promotion in future releases

### Upgrade Path

1. Upgrade Strimzi as normal
2. Optionally enable `+OperatorTracing` feature gate
3. Configure `OTEL_EXPORTER_OTLP_ENDPOINT` to point to tracing backend

## Rejected alternatives

### 1. Jaeger-only implementation

**Rejected because**: OpenTelemetry provides vendor-neutral support. Users can choose Jaeger, Grafana Tempo, Datadog, or other backends without code changes.

### 2. CRD-based configuration (e.g., `Kafka.spec.clusterOperator.tracing`)

**Rejected because**: 
- Cluster Operator runs independently of any specific `Kafka` CR
- Environment variables follow OTel conventions and are simpler
- Avoids chicken-and-egg problem for tracing operator startup

### 3. Java agent-based instrumentation

**Rejected because**:
- Less control over span structure and attributes
- Potential conflicts with other Java agents
- Manual instrumentation allows semantic span naming

### 4. Micrometer Tracing instead of OpenTelemetry SDK

**Rejected because**:
- Adds abstraction layer that complicates configuration
- OpenTelemetry alternatives are not CNCF standard with broader ecosystem support
- Direct SDK usage aligns with existing Strimzi component tracing
