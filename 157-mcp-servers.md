# Donate StreamsHub MCP to the Strimzi Organisation

This proposal covers the donation of the [streamshub-mcp](https://github.com/streamshub/streamshub-mcp) repository to the Strimzi organisation as a new `strimzi/mcp-servers` repository.
The donation was formally approved by the StreamsHub maintainers in [streamshub/proposals#11](https://github.com/streamshub/proposals/pull/11).

## Current situation

Strimzi simplifies Kafka management on Kubernetes, but diagnosing issues or answering operational questions still requires running many `kubectl` commands and correlating results from custom resource statuses, operator logs, pod logs, metrics, and network configuration.
There is currently no structured way for AI assistants to access Strimzi-specific operational data.

The StreamsHub organisation created [streamshub-mcp](https://github.com/streamshub/streamshub-mcp) to fill this gap.
The project implements a [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server for Strimzi that provides read-only structured access to all Strimzi custom resources, pod state, logs, and metrics.
It is approaching its `0.3.0` release and has been validated against real-world Strimzi deployments.

## Motivation

Strimzi is the natural home for a Strimzi-specific MCP server.
Hosting it under the Strimzi organisation provides better alignment with the primary domain, opens the project to the broader Strimzi contributor base, and enables tighter integration with Strimzi's development and release processes.

The MCP server addresses a real operational problem.
When something goes wrong, engineers need to correlate information from many places — custom resource conditions, operator logs, broker pod logs, and metrics.
Doing this manually takes time and is error-prone, especially under pressure.
The MCP server gives LLM clients structured, read-only access to all of this data through Strimzi-specific tools.
The server itself does not diagnose issues.
The LLM, guided by prompt templates that encode Strimzi debugging expertise, interprets the data and produces a diagnosis.

## Proposal

The entire `streamshub-mcp` repository will be transferred to the Strimzi GitHub organisation under the name `mcp-servers`.
The last release under the StreamsHub organisation will be `0.3.0`.
The first release under the Strimzi organisation will be `0.4.0`, continuing the existing version sequence.

### Repository structure

The repository is a Maven multi-module project.
Currently, it contains the following parts:

```
strimzi-mcp/                       # Repository root
├── pom.xml                        # Parent POM with shared dependencies
├── common/                        # Shared SPI interfaces and utilities
├── metrics-prometheus/            # Prometheus/Thanos/VictoriaMetrics metrics provider
├── loki-log-provider/             # Grafana Loki log provider
├── elasticsearch-log-provider/    # Elasticsearch and OpenSearch log provider
├── strimzi-mcp-server/            # Strimzi MCP server
└── systemtest/                    # System test suite
```

Future MCP servers can be added as new modules alongside the existing ones.
A Kafka MCP server is one such candidate — it would complement the Strimzi MCP by providing access to Kafka cluster data via the Kafka Admin API (topics, consumer groups, offsets, configs) that is not available through Kubernetes resources.
Both servers would run side by side and an LLM client can coordinate them in a single conversation.
This is out of scope for the current proposal.

### MCP capabilities

The Strimzi MCP server exposes three categories of capabilities to LLM clients.

**MCP Tools** provide read-only structured access to Strimzi infrastructure.
The tools cover all supported Strimzi custom resources — Kafka, KafkaNodePool, KafkaTopic, KafkaUser, KafkaConnect, KafkaConnector, KafkaBridge, KafkaMirrorMaker2, and KafkaRebalance — as well as the Strimzi Cluster Operator and the Strimzi Drain Cleaner.
Every resource type has list and get tools.
Resource types that own a workload additionally have pod and log retrieval tools.
Metrics tools expose Prometheus-format metrics from Kafka broker and controller pods, KafkaConnect, KafkaBridge, KafkaExporter, and the Strimzi operators.
Composite diagnostic tools use MCP Sampling and Elicitation to orchestrate multi-step investigations in a single tool call.

**MCP Resources** expose live Kubernetes state as structured context that clients can attach directly to conversations without explicit tool calls.
Resource templates follow the Kubernetes API URI hierarchy and cover Kafka cluster status, cluster topology, KafkaNodePool status, KafkaTopic status, KafkaUser status, KafkaConnector status, KafkaRebalance status, and Strimzi operators status.
Resource subscriptions use Kubernetes watches to push real-time notifications to subscribed clients when resource state changes.

**MCP Prompt Templates** encode the diagnostic expertise of an experienced Strimzi engineer as structured multi-step workflows.
Each template generates a step-by-step instruction prompt that tells the LLM which tools to call, what to look for in the results, and how to correlate findings.
The initial set of thirteen prompt templates is listed below.

| Template name                      | Description                                                                                                                                                                                                 |
|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `diagnose-cluster-issue`           | Guides the LLM through a structured diagnosis of a Kafka cluster that is not ready or behaving unexpectedly — checks CR conditions, KafkaNodePool status, operator logs, pod health, and pod logs in order. |
| `troubleshoot-connectivity`        | Diagnoses client connectivity problems by inspecting listener configuration, bootstrap addresses, TLS certificate validity, and authentication settings for a given cluster and listener.                   |
| `troubleshoot-topic`               | Investigates a KafkaTopic that is not ready or has a configuration mismatch — checks topic status, partition and replica counts, and related broker conditions.                                             |
| `troubleshoot-connect`             | Diagnoses a KafkaConnect cluster that is not ready or failing to run connectors — checks Connect CR status, pod health, and Connect worker logs.                                                            |
| `troubleshoot-connector`           | Investigates a specific KafkaConnector that is in a failed or paused state — checks connector status, task failures, and the parent KafkaConnect worker logs.                                               |
| `troubleshoot-bridge`              | Diagnoses a KafkaBridge deployment that is not ready or returning errors — checks Bridge CR status, pod health, and Bridge pod logs.                                                                        |
| `troubleshoot-mirror-maker`        | Investigates a KafkaMirrorMaker2 instance with replication problems — checks MM2 CR status, connector states, pod health, and MM2 pod logs.                                                                 |
| `analyze-capacity`                 | Assesses whether a Kafka cluster is approaching resource limits by examining CPU and memory requests and limits, storage usage, topic scale, and throughput metrics.                                        |
| `analyze-kafka-metrics`            | Examines Prometheus metrics from Kafka broker and controller pods by category (replication, throughput, performance, or resources) and reports on anomalies and trends.                                     |
| `analyze-strimzi-operator-metrics` | Examines Prometheus metrics from the Strimzi Cluster Operator to identify reconciliation bottlenecks, error rates, or queue backlogs.                                                                       |
| `assess-upgrade-readiness`         | Checks whether a Kafka cluster is in a safe state to upgrade — verifies cluster health, under-replicated partitions, active rebalances, resource headroom, and certificate validity.                        |
| `compare-cluster-configs`          | Compares the effective configuration of two Kafka clusters side by side and highlights differences in listeners, storage, resource limits, and Kafka broker config.                                         |
| `audit-security`                   | Reviews the security posture of a Kafka cluster — checks listener authentication and encryption settings, TLS certificate expiry, and KafkaUser ACL configuration.                                          |

### Technology stack

- **Java 21** and **Quarkus** — cloud-native Java framework with fast startup and low memory footprint
- **Fabric8 Kubernetes Client** with the **Strimzi API** library for typed access to Strimzi custom resources
- **Quarkus MCP Server** (Quarkiverse) — MCP protocol implementation supporting both Streamable HTTP and SSE transports
- **Apache License 2.0** — consistent with the rest of the Strimzi ecosystem

### Pluggable provider architecture

The `common` module defines stable SPI contracts that allow alternative implementations to be selected via configuration without forking the repository:

- `MetricsProvider` — selected via `mcp.metrics.provider`.
  The default implementation scrapes Prometheus metrics directly from pod HTTP endpoints.
  The bundled `metrics-prometheus` module provides an alternative that queries an existing Prometheus, Thanos, or VictoriaMetrics instance.
- `LogCollectorProvider` — selected via `mcp.log.provider`.
  The default implementation reads logs directly from Kubernetes pod logs via the Fabric8 client.
  The bundled `loki-log-provider` and `elasticsearch-log-provider` modules provide alternatives for Grafana Loki and Elasticsearch or OpenSearch respectively.
- `GuardrailFilter` — a CDI interceptor chain triggered by the `@Guarded` annotation on tool classes.
  Multiple filter beans run in priority order before and after each tool executes: rate limiting per tool category (disabled by default), input sanitisation to strip control characters, secret redaction to remove tokens and passwords from output, response size limiting to prevent oversized context, and internal metrics recording.
  Each concern is an isolated, independently testable bean.
  New guardrails can be added without modifying existing tools.

### Security and RBAC

The MCP server uses a dedicated ServiceAccount with a minimal ClusterRole that grants only `get`, `list`, and `watch` on Strimzi custom resources and the additional plain Kubernetes resources the server needs: operator Deployments, Pods, Services, pod logs, Events, ConfigMaps, Routes, Ingresses, leader-election Leases, and ValidatingWebhookConfigurations.
An opt-in per-namespace Role for sensitive resources (Secrets for certificate metadata, and `pods/proxy` for direct pod metrics scraping) is provided separately and only needs to be applied in namespaces where Kafka clusters run.
If the sensitive Role is not applied in a given namespace, the certificate and pod-scraping features fail closed for that namespace and all other functionality continues to work.

The initial authorization model relies entirely on Kubernetes RBAC.
MCP-level authentication and authorization is explicitly out of scope for the initial release.
It is intended as follow-up work under the Strimzi organisation and will be covered by a separate proposal.

### Naming and Maven coordinates

The repository name changes from `streamshub-mcp` to `mcp-servers`.
The Maven `groupId` changes from `io.streamshub` to `io.strimzi.mcp`.
The artefact IDs change accordingly:

| Module                     | Old `artifactId`                        | New `artifactId`                 |
|----------------------------|-----------------------------------------|----------------------------------|
| Parent POM                 | `streamshub-mcp`                        | `strimzi-mcp`                    |
| Shared SPI                 | `streamshub-mcp-common`                 | `common`                         |
| Prometheus metrics         | `streamshub-metrics-prometheus`         | `metrics-prometheus-provider`    |
| Loki log provider          | `streamshub-loki-log-provider`          | `loki-log-provider`              |
| Elasticsearch log provider | `streamshub-elasticsearch-log-provider` | `elasticsearch-log-provider`     |
| Strimzi MCP server         | `strimzi-mcp`                           | `strimzi-mcp-server`             |

Container images move from `quay.io/streamshub/strimzi-mcp` to `quay.io/strimzi/strimzi-mcp-server`.

### Provider configuration values

The pluggable providers are activated by setting a string value via `mcp.log.provider` or `mcp.metrics.provider` in `application.properties`, or via the corresponding `MCP_LOG_PROVIDER` and `MCP_METRICS_PROVIDER` environment variables.
The values currently carry a `streamshub-` prefix, for example `mcp.log.provider=streamshub-kubernetes` and `mcp.metrics.provider=streamshub-pod-scraping`, with `streamshub-loki`, `streamshub-prometheus`, and `streamshub-elasticsearch` selecting the bundled alternatives.

All selector values drop the `streamshub-` prefix in favour of `strimzi-` in `0.4.0`, so that `streamshub-kubernetes` becomes `strimzi-kubernetes`, `streamshub-pod-scraping` becomes `strimzi-pod-scraping`, and so on.
The property and environment variable names themselves are already neutral and do not change.

### Governance and CI/CD

The `mcp-servers` repository adopts the standard Strimzi governance model, including the same maintainer and approver structure, code of conduct, and contribution process.
The repository also adopts Strimzi CI/CD tooling with the minimal necessary changes to accommodate the Maven multi-module build and the system tests that require a Kubernetes cluster.

### Documentation and website

The `mcp-servers` repository will maintain its own documentation covering installation, configuration, RBAC setup, and the available tools, resources, and prompt templates.
This documentation will be published to the Strimzi website under a dedicated MCP section and linked from the main Strimzi documentation.
Release notes for each version will follow the same format as other Strimzi components.
The StreamsHub site will be updated to redirect users to the Strimzi documentation once the transfer is complete.

### Commitments

The StreamsHub maintainers approved the donation in [streamshub/proposals#11](https://github.com/streamshub/proposals/pull/11).
This proposal accepts the following commitments on behalf of the Strimzi organisation:

- All artifacts released under the StreamsHub organisation remain publicly available at their original coordinates.
  Future releases published under the Strimzi organisation will be publicly available, in whatever form and location the Strimzi maintainers decide upon.
- Publishing the `io.strimzi.mcp:common` SPI module to Maven Central is the intended goal, so that third-party provider implementations can be built without forking the repository, subject to the Strimzi organisation's standard publishing infrastructure and policies.
- David Kornel will be nominated as a component owner for the Strimzi MCP component following the standard Strimzi maintainer vote process.

### Transition plan

The transition is carried out in the following order:

1. StreamsHub releases `0.3.0` as the final release under the StreamsHub organisation.
2. The repository is transferred to the Strimzi organisation and renamed to `mcp-servers`.
3. The Strimzi organisation onboards the repository to its CI/CD, Maven Central publishing, and `quay.io/strimzi` image push credentials.
4. The rename of the Maven coordinates, provider selector values, and container image repository lands on `main`.
5. Strimzi releases `0.4.0` as the first release under the Strimzi organisation once we finish the transition

A GitHub repository transfer preserves the full history together with issues, pull requests, releases, stars, and forks, and leaves a redirect at the old path.
Existing clone and import URLs therefore keep working, so contributors with local checkouts and users referencing the old URL are not broken by the move.

The move is announced through the `0.3.0` release notes, the StreamsHub site, and the Strimzi community channels, so that existing users know where subsequent releases are published.

## Affected/not affected projects

### Affected

- `streamshub/streamshub-mcp` — transferred to `strimzi/mcp-servers`.
  Nothing remains under the StreamsHub organisation apart from the GitHub redirect created by the transfer.
- `streamshub/streamshub-site` — upcoming releases will be hosted under the Strimzi organisation.
  The StreamsHub site will be updated to link to the Strimzi release page.
- Strimzi Maven Central publishing pipeline — the new `io.strimzi` SPI and server artefacts must be onboarded.

### Not affected

- `strimzi/strimzi-kafka-operator` — no changes to the operator itself.
- All other Strimzi repositories.
- All other StreamsHub repositories.

## Compatibility

Users who depend on the Maven SPI modules to build custom provider implementations will need to update their `groupId` to `io.strimzi.mcp` and their `artifactId` coordinates to the new values when upgrading to `0.4.0`.
The interfaces themselves do not change as part of the code donation, but they might change in the future as part of development.

Users who deploy the server will need to update the container image repository from `quay.io/streamshub/strimzi-mcp` to `quay.io/strimzi/strimzi-mcp-server`.

Users who explicitly configure a log or metrics provider will need to update the selector value, because the `streamshub-` prefix will be replaced by `strimzi-` in `0.4.0`.
This applies both to `mcp.log.provider` and `mcp.metrics.provider` in `application.properties` and to the `MCP_LOG_PROVIDER` and `MCP_METRICS_PROVIDER` environment variables.
Deployments that rely on the defaults are not affected, because the defaults change together with the accepted values.
This is a deliberate breaking change, taken once at the point where the project changes organisation rather than spread over later releases.
