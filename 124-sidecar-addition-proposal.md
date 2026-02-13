# Proposal: Native Sidecar Container Support for Strimzi Components

## Summary
Provide a brief summary of the feature you are proposing to add to Strimzi.

This proposal adds first‑class, **declarative sidecar container** support to Strimzi-managed workloads (starting with Kafka brokers), configured via CRDs under `spec.kafka.template.pod.sidecarContainers`. Users can attach monitoring, logging, security, and networking sidecars without webhooks or forks. The design introduces a stable CRD schema (`SidecarContainer`), a validation and conversion pipeline, and generic interfaces so other Strimzi components (KafkaConnect, KafkaBridge, MirrorMaker2, etc.) can adopt it.

## Current situation
Describe the current capability Strimzi has in this area.

Strimzi currently lacks native sidecar support. Users rely on:
1. Mutating webhooks/admission controllers
2. Forking and patching Strimzi code
3. Ephemeral containers for ad‑hoc troubleshooting

Limitations of these approaches include fragile upgrades, higher operational complexity, lack of declarative control, and weak integration with Strimzi lifecycle and security primitives.

## Motivation
Explain the motivation why this should be added, and what value it brings.

**Business value**
- Operational flexibility: Enable Flexibility for adding additional sidecars as per the custom need of the users
- Simplified ops: manage sidecars natively via Strimzi CRDs and GitOps
- Cloud‑native patterns: support log forwarders, APM agents, service‑mesh proxies

**Representative use cases**
- Monitoring/Observability: prometheus exporters, APM agents (Datadog, New Relic)
- Logging/Auditing: Fluent Bit/Filebeat; audit collectors
- Security/Compliance: Vault agent; cert rotation helpers
- Networking: traffic analyzers/proxies
- Data tooling: lightweight data checks or protocol helpers

## Proposal
Provide an introduction to the proposal. Use sub sections to call out considerations, possible delivery mechanisms etc.

This proposal introduces a **custom `SidecarContainer` CRD model** (stable, Fabric8‑independent) and a **component‑agnostic sidecar pipeline**:
1) Schema & API surface (`SidecarContainer`) kept minimal and serializable
2) Validation in CR processing; conversion to Fabric8 `Container` at runtime
3) Component interface (`SidecarInterface`) + `SidecarUtils` for reuse
4) PodTemplate placement for per‑pool (KRaft) configuration
5) Automatic NetworkPolicy enrichment for declared sidecar ports
6) Volume access model supporting Kafka data (read‑only), ConfigMaps/Secrets, and `emptyDir`

### Rationale for Custom `SidecarContainer` Abstraction
We use a custom `SidecarContainer` instead of Fabric8 `io.fabric8.kubernetes.api.model.Container` in CRDs.

**Technical justification**
- **Problem – CRD generation incompatibility**: Fabric8's `Container` exposes `IntOrString` fields (e.g., ports, probe handlers). Strimzi's CRD generator cannot reliably serialize `IntOrString` to OpenAPI v3, breaking validation and schema generation.
- **Solution – Custom abstraction**: `SidecarContainer` mirrors essential fields with simple types (String, List, standard objects). Probes use a Strimzi type (`SidecarProbe`) instead of Fabric8 `Probe`.

**Supported configuration (concise)**
- Core: `name`, `image`, `imagePullPolicy`, `command`, `args`
- Env & storage: `env` (ContainerEnvVar), `volumeMounts`
- Networking: `ports` (ContainerPort)
- Resources: `resources` (ResourceRequirements) **mandatory**
- Security: `securityContext` (SecurityContext)
- Health: `livenessProbe`, `readinessProbe` (custom `SidecarProbe`)

**Benefits**
- **CRD compatibility**: Ensures generated CRDs are valid and parseable by Kubernetes
- **Simplified API surface**: Exposes only the configuration options relevant to Strimzi sidecars
- **Maintainability**: Decouples Strimzi's API from potential breaking changes in Fabric8's container model
- **Type safety**: Avoids runtime serialization errors caused by ambiguous field types

### API design (small snippet)
```java
@Description("Sidecar container alongside main component")
public class SidecarContainer { 
  private String name;
  private String image;
  private List<ContainerPort> ports;
  private List<EnvVar> env;
  private ResourceRequirements resources; // mandatory
  private List<VolumeMount> volumeMounts;
  private SecurityContext securityContext;
}
```

**CRD location in PodTemplate (Kafka example)**
```yaml
spec:
  kafka:
    template:
      pod:
        sidecarContainers:
          - name: fluent-bit
            image: fluent/fluent-bit:2.2.0
            resources:
              limits: { cpu: "200m", memory: "256Mi" }
```

### Validation & conversion pipeline
**Hierarchy**: (1) CRD schema → (2) Strimzi API validation → (3) component rules → (4) pod creation.  
**Key rules**: unique names; no conflict with Strimzi-managed names; resource limits required; ports must not collide with component‑reserved ports; volume mounts must exist and avoid conflicts.

**Interface & utils (signatures only)**
```java
public interface SidecarInterface {
  void validateSidecarContainers(...);
  List<Container> createSidecarContainers(PodTemplate pod);
}

// Conversion
List<Container> convertSidecarContainers(PodTemplate pod);
```

### Network policies & volumes
- **NetworkPolicy**: Extract sidecar ports from all pools; add ingress rules for those ports by default. Users can further restrict via standard policies.
- **Volumes**: Support read‑only Kafka data mounts for log access, user‑defined `volumes` in `pod.template`, Secrets/ConfigMaps, and `emptyDir`.

### SidecarProbe (overview)
A Strimzi probe type used by sidecars to avoid `IntOrString` in CRDs and keep schemas simple.

**Compact class sketch**
```java
@JsonInclude(NON_NULL)
@JsonPropertyOrder({ "execCommand", "httpGetPath", "httpGetPort", "httpGetScheme",
  "tcpSocketPort", "initialDelaySeconds", "timeoutSeconds", "periodSeconds",
  "successThreshold", "failureThreshold" })
public class SidecarProbe implements UnknownPropertyPreserving { 
  private List<String> execCommand;
  private String httpGetPath, httpGetPort, httpGetScheme, tcpSocketPort;
  private int initialDelaySeconds = 15, timeoutSeconds = 5, periodSeconds = 10;
  private int successThreshold = 1, failureThreshold = 3;
  private Map<String,Object> additionalProperties;
}
```
**Highlights**
- Supports `exec`, `httpGet`, `tcpSocket` styles via simple String fields (no `IntOrString`).
- Sensible defaults: `initialDelaySeconds=15`, `timeoutSeconds=5`, `periodSeconds=10`.
- Min validations (e.g., timeout/period/success/failure thresholds ≥ 1).
- Preserves unknown properties for forward compatibility.

### Minimal examples
**Single sidecar (log forwarder)**
```yaml
spec:
  kafka:
    template:
      pod:
        sidecarContainers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2.0
          resources:
            limits: { cpu: "200m", memory: "256Mi" }
```
**Per‑pool configuration (KRaft)**
```yaml
kind: KafkaNodePool
spec:
  roles: [broker]
  template:
    pod:
      sidecarContainers:
      - name: log-forwarder
        image: fluent/fluent-bit:2.2.0
        resources:
          limits: { cpu: "200m", memory: "256Mi" }
```

## Affected/not affected projects
Call out the projects in the Strimzi organisation that are/are not affected by this proposal.

**Affected (Phase 1)**
- `strimzi-kafka-operator` (api: new `SidecarContainer`; operator: `SidecarInterface`, `SidecarUtils`, KafkaCluster integration; crd‑generator updates)

**Future (Phase 2+)**
- KafkaConnect, KafkaBridge, MirrorMaker2, EntityOperator (implement interface; add reserved ports/names)

**Not affected**
- strimzi-kafka-oauth, test-container, drain-cleaner, access-operator (no changes)

## Compatibility
Call out any future or backwards compatibility considerations this proposal has accounted for.

- **Backward compatible**: Field is optional; no change for existing CRs.
- **Forward compatible**: Stable custom model avoids Fabric8 `IntOrString` issues in schema generation.
- **Rolling behavior**: Add/modify/remove sidecars via standard rolling updates.
- **Versioning**: Introduce in Strimzi ≥ 0.50.0; other components can adopt incrementally.

## Rejected alternatives
Call out options that were considered while creating this proposal, but then later rejected, along with reasons why.

1) **Expose Fabric8 `Container` directly** — rejected due to OpenAPI/CRD stability and `IntOrString` schema issues.  
2) **Separate `KafkaSidecar` CRD** — rejected for not having a generic placeholder for future expandability.  
3) **Operator‑level auto‑injection** — rejected to keep explicit user control and avoid risky implicit coupling.  
4) **InitContainer/DaemonSet patterns** — do not satisfy long‑running, pod‑local sidecar use cases.  
5) **Helm‑only customization** — not portable; breaks on operator‑managed rollouts.

## References
References (informative)
- Strimzi Pod Templates (docs)
- Kubernetes multi‑container pods (docs)
- Implementation PR: strimzi/strimzi‑kafka‑operator#12121
