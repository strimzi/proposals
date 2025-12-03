# Stretch Kafka Cluster

The Strimzi Kafka Operator currently manages Kafka clusters within a single Kubernetes cluster.
This proposal extends its support to stretch Kafka clusters, where broker and controller Kafka pods are distributed across multiple Kubernetes clusters.

## Current situation

At present, the availability of Strimzi-managed Kafka clusters is constrained by the availability of the underlying Kubernetes cluster.
If a Kubernetes cluster experiences an outage, the entire Kafka cluster becomes unavailable, disrupting all connected Kafka clients.

While it is possible to enhance availability by running Kubernetes across multiple availability zones and configuring Strimzi with affinity rules, tolerations, or topology spread constraints, such configurations are still limited to a single Kubernetes control plane.
Any failure or disruption to this control plane or a broader infrastructure issue affecting the cluster can impact cluster operations and affect connected clients. 
In contrast, the stretch cluster distributes Kafka nodes across independent Kubernetes clusters, each with its own control plane and fault domain.
This enhances fault tolerance by enabling Kafka to continue operating even if one entire Kubernetes cluster becomes unavailable.

The intent is not to imply that Kubernetes control planes are unreliable, they are mature and widely used in HA configurations.
Rather, the stretch approach supports scenarios where users operate multiple isolated Kubernetes clusters for organizational, operational, or regulatory reasons, and want Kafka to span these boundaries while maintaining quorum and availability.

## Motivation

A stretch Kafka cluster allows Kafka nodes to be distributed across multiple Kubernetes clusters. 
This approach facilitates several valuable use cases:

- **High Availability**: In a stretch cluster deployment, Kafka broker and Controller pods are distributed across multiple, fully independent Kubernetes clusters.
Each cluster operates its own control plane and infrastructure.
This separation improves resilience because the failure of one Kubernetes cluster whether due to control plane issues, networking failures, or infrastructure outages does not affect the availability of Kafka brokers running in the other clusters.
Clients can continue producing and consuming data without interruption as long as a quorum of brokers and controllers remains operational.
Unlike MirrorMaker 2 (MM2), a stretch cluster provides strong data durability through synchronous replication and enables fast disaster recovery with automated client failover.

- **Migration Flexibility**: A stretch Kafka cluster enables seamless migration, whether it's moving the entire cluster across Kubernetes environments or cloud providers without downtime, or relocating individual Kafka nodes as needed. 
This flexibility helps with maintenance, scaling, and workload transitions between environments.

## Proposal

This proposal enhances the Strimzi Kafka operator to support stretch Kafka clusters, where broker, controller, and combined-role Kafka pods are distributed across multiple Kubernetes clusters.
The goal is to ensure high availability of Kafka client operations such as producing and consuming messages even in the event of a single cluster failure, including failure of the central cluster.
The proposal outlines the high level topology, design concepts, and detailed implementation considerations for such deployments.

### Limitations and Considerations

While a stretch Kafka cluster offers several advantages, it also introduces some challenges:

- **Increased Network Complexity and Costs**: Communication between brokers and controllers across clusters relies on network connectivity, which can be less reliable and more costly than intra-cluster communication.

- **Latency Requirements**: Stretch Kafka clusters are best suited for environments with low latency and high bandwidth network connections between Kubernetes clusters.
High latency can adversely affect performance and synchronization of Kafka nodes.

- **Optimal Use Case**: Stretch clusters are optimized for same datacenter multi availability zone deployments where inter-cluster latency is < 5ms. 
Testing validates excellent performance (50,000 msg/sec throughput with full availability) in this configuration. 

  **Important:** Latency recommendations below are based on conservative worst-case testing.
  Real world same datacenter multi-AZ deployments will perform significantly better than these thresholds suggest.
  See "Performance Characteristics and Operational Guidelines" section for detailed validation results and testing methodology.

  For cross region disaster recovery with higher latency (> 50ms), MirrorMaker 2 asynchronous replication is more appropriate.

### Prerequisites

- **Multiple Kubernetes clusters**: Stretch Kafka clusters require multiple Kubernetes clusters.
The recommended minimum number of clusters is 3 to simplify achieving quorum for Kafka controllers and enhance High Availability (HA).
However, the Cluster Operator does not enforce this as a hard requirement.
Stretch clusters can be deployed with fewer than 3 clusters to allow migration flexibility, resource optimization scenarios, or test environments.

- **Low Latency and High Bandwidth**: Kafka clusters require low-latency and high-bandwidth network connections between clusters.
  - **Recommended:** < 5ms inter-cluster latency (same datacenter, multi-AZ deployment)
  - **Maximum viable:** < 50ms (severe performance degradation expected)
  - **Bandwidth:** >= 10 Gbps recommended
  - **Packet loss:** < 0.1%
  - **Jitter:** < 2ms
  
  **Note:** These thresholds are based on conservative worst-case testing methodology. 
  Real world same datacenter deployments where only cross cluster traffic experiences latency will perform significantly better.
  Extensive validation with 27 brokers across 3 clusters achieved 50,000 msg/sec throughput with < 1ms inter-cluster latency.
  See "Performance Characteristics and Operational Guidelines" section for complete testing results.
  
  Users MUST validate network characteristics before deployment using latency tests, bandwidth tests, and optional Chaos Mesh validation (see "Performance Characteristics and Operational Guidelines" section).

- **cross cluster networking**: Enabling networking between Kubernetes clusters requires additional technology.
Users must configure a networking solution that enables pod-to-pod communication across cluster boundaries.
This proposal defines a Service Provider Interface (SPI) that allows multiple networking implementations to be used as plugins.


### Networking Architecture: Plugin-Based Design

#### Overview

The core challenge in stretch Kafka clusters is enabling cross cluster pod-to-pod communication for Kafka brokers and controllers.
Different deployment environments may require different networking approaches:
- Multi-Cluster Services (MCS) API for DNS-based service discovery
- NodePort services with fixed node IPs
- LoadBalancer services with external IPs
- Custom cloud specific solutions

Rather than implementing these mechanisms directly in the operator, this proposal introduces a **Service Provider Interface (SPI)** that allows networking implementations to be loaded as external plugins.

#### Stretch Networking Provider SPI

The `StretchNetworkingProvider` interface defines the contract that all networking plugins must implement:

```java
public interface StretchNetworkingProvider {
    // Initialize provider with configuration and cluster access
    Future<Void> init(Map<String, String> config,
                      ResourceOperatorSupplier centralSupplier,
                      RemoteResourceOperatorSupplier remoteSupplier);
    
    // Create networking resources (Services, ServiceExports, etc.)
    Future<List<HasMetadata>> createNetworkingResources(
        Reconciliation reconciliation,
        String namespace,
        String podName,
        String clusterId,
        Map<String, Integer> ports);
    
    // Discover the actual endpoint for a pod (critical for configuration)
    Future<String> discoverPodEndpoint(
        Reconciliation reconciliation,
        String namespace,
        String serviceName,
        String clusterId,
        String portName);
    
    // Generate advertised.listeners configuration
    Future<String> generateAdvertisedListeners(
        Reconciliation reconciliation,
        String namespace,
        String podName,
        String clusterId,
        Map<String, String> listeners);
    
    // Generate controller.quorum.voters configuration
    Future<String> generateQuorumVoters(
        Reconciliation reconciliation,
        String namespace,
        List<ControllerPodInfo> controllerPods,
        String replicationPortName);
    
    // Generate DNS names for services and pods
    String generateServiceDnsName(String namespace, String serviceName, String clusterId);
    String generatePodDnsName(String namespace, String serviceName, String podName, String clusterId);
    
    // Clean up networking resources
    Future<Void> deleteNetworkingResources(
        Reconciliation reconciliation,
        String namespace,
        String podName,
        String clusterId);
    
    String getProviderName();
}
```

#### Plugin Loading Mechanism

Networking providers are loaded dynamically at operator startup using Java's ServiceLoader mechanism.
The central cluster operator configuration specifies which plugin to use:

```yaml
env:
- name: STRIMZI_STRETCH_PLUGIN_CLASS_NAME
  value: io.strimzi.plugin.stretch.NodePortNetworkingProvider
- name: STRIMZI_STRETCH_PLUGIN_CLASS_PATH
  value: /opt/strimzi/plugins/*
```

The `StretchNetworkingProviderFactory` handles plugin discovery and initialization:
1. Creates a custom ClassLoader with the specified plugin classpath
2. Loads the provider class using reflection
3. Calls the `init()` method to initialize the provider
4. Returns the initialized provider for use by the operator

This approach provides:
- **Separation of concerns**: Core operator logic is decoupled from networking implementation details
- **Extensibility**: Users can implement custom providers without modifying operator code
- **Testing**: Providers can be tested independently and swapped for different environments

#### Reference Implementations

The Strimzi project provides reference implementations in separate repositories:

**NodePort Provider** (`strimzi-stretch-nodeport-plugin`)
- Creates NodePort services for each Kafka pod
- Caches one stable node IP per cluster (NodePort is accessible from any node)
- Returns endpoints in format: `nodeIP:nodePort` (e.g., `10.21.37.21:31001`)
- Best for: On-premises deployments, development environments

**LoadBalancer Provider** (`strimzi-stretch-loadbalancer-plugin`)
- Creates LoadBalancer services for each Kafka pod
- Waits for LoadBalancer IP assignment with exponential backoff retry
- Returns endpoints in format: `loadBalancerIP:port` (e.g., `10.21.50.10:9091`)
- Best for: Cloud environments with LoadBalancer support (AWS ELB, GCP, Azure)

**MCS Provider** (Third-party reference)
- Creates Service + ServiceExport resources per the Multi-Cluster Services API
- Returns DNS-based endpoints: `pod.clusterId.svc.clusterset.local:port`
- Best for: Environments with MCS implementation (Submariner, Cilium Cluster Mesh)

Each provider handles the specific requirements of its networking approach, including resource creation, endpoint discovery, and cleanup.


### High-level Architecture

#### Topology of a Stretch Cluster

<Add new Architecture diagram>

The diagram illustrates a topology comprising three Kubernetes clusters.

One cluster is designated as the "central cluster", while additional clusters are considered "remote".
The central cluster acts as the control plane where users create all custom resources for the Kafka cluster: Kafka, KafkaNodePool, KafkaUser, KafkaTopic, etc.

A KafkaNodePool definition specifies a target Kubernetes cluster (central or remote) as the deployment target.
The operator on the central cluster creates all necessary resources for the node pool on the target cluster.

#### Resource Distribution

The following table shows which resources are created in each cluster:

| Resource Type | Central Cluster | Remote Clusters |
|--------------|----------------|----------------|
| **Custom Resources (Kafka, KafkaNodePool)** | Created by user | Never created |
| **StrimziPodSet** | Created for central pools | Created by central operator |
| **Pods** | Managed by local operator | Managed by local operator |
| **Services** | Created for central pods | Created by central operator |
| **ConfigMaps** | Created for central pods | Created by central operator |
| **Secrets (CA, certs)** | Created once | Replicated by central operator |
| **PersistentVolumeClaims** | Created for central pools | Created by central operator |
| **ServiceExport/Routes/Ingress** | Per networking provider | Per networking provider |

The central operator has write access to remote clusters via kubeconfig and creates resources directly.
Remote operators are responsible only for reconciling StrimziPodSet resources (creating/updating pods).

### Configuration and Setup

#### Step 1: Deploy Cluster Operators

The cluster operator must be deployed to all Kubernetes clusters (central and remote).

The operator in the central cluster requires additional configuration to access remote clusters:

```yaml
- name: STRIMZI_CENTRAL_CLUSTER_ID
  value: "cluster-central"

- name: STRIMZI_REMOTE_KUBE_CONFIG
  value: |
    cluster-east.url=https://api.cluster-east.example.com:6443
    cluster-east.secret=kubeconfig-cluster-east
    cluster-west.url=https://api.cluster-west.example.com:6443
    cluster-west.secret=kubeconfig-cluster-west

- name: STRIMZI_STRETCH_PLUGIN_CLASS_NAME
  value: io.strimzi.plugin.stretch.NodePortNetworkingProvider

- name: STRIMZI_STRETCH_PLUGIN_CLASS_PATH
  value: /opt/strimzi/plugins/*
```

**Environment Variables:**

- `STRIMZI_CENTRAL_CLUSTER_ID` (Required): Unique identifier for the central cluster
- `STRIMZI_REMOTE_KUBE_CONFIG` (Required): Maps cluster IDs to API endpoints and credential secrets
- `STRIMZI_STRETCH_PLUGIN_CLASS_NAME` (Required): Fully qualified class name of the networking provider
- `STRIMZI_STRETCH_PLUGIN_CLASS_PATH` (Required): Classpath for loading the provider JAR

**Kubeconfig Secrets:**

For each remote cluster, create a Secret in the **central cluster** containing the kubeconfig for accessing that remote cluster:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kubeconfig-cluster-east
  namespace: kafka  # Created in CENTRAL cluster, not remote
type: Opaque
data:
  kubeconfig: <base64-encoded-kubeconfig>  # Credentials to access cluster-east API server
```

The central operator uses these secrets to authenticate to remote cluster API servers. 
The operator validates kubeconfig expiry and reports errors if credentials expire.

#### Step 2: Create Kafka and KafkaNodePool Resources

**Kafka CR:**

Enable stretch mode with an annotation:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  annotations:
    strimzi.io/enable-stretch-cluster: "true"
spec:
  kafka:
    version: 3.8.0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
```

**KafkaNodePool CRs:**

Each node pool specifies its target cluster via annotation:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: pool-central
  labels:
    strimzi.io/cluster: my-cluster
  annotations:
    strimzi.io/stretch-cluster-alias: "cluster-central"
spec:
  replicas: 2
  roles:
    - controller
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: pool-east
  labels:
    strimzi.io/cluster: my-cluster
  annotations:
    strimzi.io/stretch-cluster-alias: "cluster-east"
spec:
  replicas: 2
  roles:
    - controller
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: pool-west
  labels:
    strimzi.io/cluster: my-cluster
  annotations:
    strimzi.io/stretch-cluster-alias: "cluster-west"
spec:
  replicas: 1
  roles:
    - controller
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
```

This creates a 5-node Kafka cluster (2+2+1) with controllers and brokers distributed across 3 Kubernetes clusters.

#### Validation Rules

The operator enforces comprehensive validation through `StretchClusterValidator`:

##### Configuration Validation Matrix

| Operator Configured | Kafka CR Annotation | KNP Annotations | Result | Error Code |
|---------------------|---------------------|-----------------|--------|------------|
| ❌ No | ❌ No | ❌ None | ✅ Valid: Standard deployment | - |
| ❌ No | ❌ No | ✅ Some/All have alias | ❌ **INVALID** | `OperatorNotConfigured` |
| ❌ No | ✅ Yes | ❌ None | ❌ **INVALID** | `OperatorNotConfigured` |
| ❌ No | ✅ Yes | ✅ Some/All have alias | ❌ **INVALID** | `OperatorNotConfigured` |
| ✅ Yes | ❌ No | ❌ None | ✅ Valid: Stretch configured but not used | - |
| ✅ Yes | ❌ No | ✅ Some/All have alias | ❌ **INVALID** | `MissingStretchAnnotation` |
| ✅ Yes | ✅ Yes | ❌ None (all missing) | ❌ **INVALID** | `MissingTargetCluster` |
| ✅ Yes | ✅ Yes | ⚠️ Some have alias | ❌ **INVALID** | `MissingTargetCluster` |
| ✅ Yes | ✅ Yes | ✅ All have valid aliases | ✅ Valid: Stretch cluster | - |
| ✅ Yes | ✅ Yes | ✅ All have aliases (some invalid) | ❌ **INVALID** | `InvalidTargetCluster` |

##### Validation Checks Performed

**1. Operator Configuration Check:**
- Validates `STRIMZI_REMOTE_KUBE_CONFIG` is set
- Validates `STRIMZI_CENTRAL_CLUSTER_ID` is set
- Validates `STRIMZI_STRETCH_PLUGIN_CLASS_NAME` is set
- Validates `STRIMZI_STRETCH_PLUGIN_CLASS_PATH` is set

**2. Kafka CR Annotation Check:**
- Looks for `strimzi.io/enable-stretch-cluster: "true"` annotation
- Must be present when any KafkaNodePool has `strimzi.io/stretch-cluster-alias`

**3. KafkaNodePool Annotation Check:**
- Each pool must have `strimzi.io/stretch-cluster-alias` annotation when stretch mode is enabled
- Alias value must match either `STRIMZI_CENTRAL_CLUSTER_ID` or a cluster ID in `STRIMZI_REMOTE_KUBE_CONFIG`
- ALL pools must have the annotation (no mixing of stretch and non-stretch pools)

**4. Runtime Connectivity Check:**
- Validates API server reachability for each remote cluster
- Validates `strimzipodsets.core.strimzi.io` CRD exists in each cluster
- Validates kubeconfig credentials are valid

**Error Messages:**

```
OperatorNotConfigured: "Stretch mode is enabled for Kafka cluster 'my-cluster', 
but required environment variables are not properly configured. Required: 
STRIMZI_REMOTE_KUBE_CONFIG, STRIMZI_CENTRAL_CLUSTER_ID, 
STRIMZI_STRETCH_PLUGIN_CLASS_NAME, STRIMZI_STRETCH_PLUGIN_CLASS_PATH"

MissingStretchAnnotation: "Kafka CR 'my-cluster' is missing required annotation 
'strimzi.io/enable-stretch-cluster: true' but the following node pools have 
stretch cluster annotations: pool-east, pool-west. Either add the annotation 
to the Kafka CR or remove stretch annotations from all node pools."

MissingTargetCluster: "The following node pools are missing required annotation 
'strimzi.io/stretch-cluster-alias': pool-central. All node pools must specify 
a target cluster when stretch mode is enabled."

InvalidTargetCluster: "The following target cluster IDs are not configured: 
cluster-unknown. Valid cluster IDs are: cluster-central, cluster-east, cluster-west. 
Please check STRIMZI_CENTRAL_CLUSTER_ID and STRIMZI_REMOTE_KUBE_CONFIG."

ConnectivityError: "Cannot connect to cluster 'cluster-east': Connection refused. 
Please check kubeconfig and network connectivity."

MissingCRD: "Required CRD 'strimzipodsets.core.strimzi.io' not found in cluster 
'cluster-east'. Please ensure Strimzi CRDs are installed in all target clusters."
```

Reconciliation fails immediately with descriptive errors for invalid configurations.

### Low-Level Design and Implementation

#### Garbage Collection for Remote Resources

One of the key challenges in stretch clusters is ensuring proper cleanup of resources created in remote clusters.
Kubernetes garbage collection relies on `OwnerReferences`, but these do not work across cluster boundaries.

**Problem:**
- Central operator creates StrimziPodSets, ConfigMaps, Secrets, Services in remote clusters
- These resources cannot have OwnerReferences pointing to Kafka CR in central cluster
- Standard Kubernetes garbage collection cannot cascade delete these resources

**Solution: Garbage Collector ConfigMap**

The operator creates a special "garbage collector" ConfigMap in each remote cluster.
This ConfigMap has **NO ownerReferences** - it is a standalone resource that the operator explicitly manages:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-cluster-kafka-gc
  namespace: kafka
  labels:
    app.kubernetes.io/name: kafka
    app.kubernetes.io/instance: my-cluster
    strimzi.io/cluster: my-cluster
    strimzi.io/kind: Kafka
    strimzi.io/component-type: garbage-collector
  # NO ownerReferences - this is intentional!
  # Cross-cluster owner references don't work
data:
  cluster-id: "cluster-east"
  kafka-cluster: "my-cluster"
  namespace: "kafka"
  purpose: "Garbage collection anchor for remote cluster resources"
  managed-resources: "StrimziPodSet,ServiceAccount,Secret,ConfigMap,Service,ServiceExport,PersistentVolumeClaim"
```

**Why no ownerReferences?**

The Kafka CR exists in the central cluster, not in the remote cluster.
Kubernetes in the remote cluster cannot validate an owner that exists in a different cluster.
Setting an ownerReference to a cross-cluster resource would cause Kubernetes to reject it or mark it as orphaned.
The GC ConfigMap must be a standalone resource that the operator explicitly creates and deletes.

All remote cluster resources (StrimziPodSets, ConfigMaps, Services, etc.) set this GC ConfigMap as their owner:

```java
private <T extends HasMetadata> List<T> addGarbageCollectorOwnerReference(
        String targetClusterId, List<T> resources) {
    String gcConfigMapName = KafkaResources.kafkaComponentName(reconciliation.name()) + "-gc";
    String gcUid = gcConfigMapUids.get(targetClusterId);
    
    OwnerReference gcOwnerRef = new OwnerReferenceBuilder()
        .withApiVersion("v1")
        .withKind("ConfigMap")
        .withName(gcConfigMapName)
        .withUid(gcUid)
        .withController(false)
        .withBlockOwnerDeletion(false)
        .build();
    
    return resources.stream()
        .map(resource -> {
            resource.getMetadata().setOwnerReferences(
                List.of(gcOwnerRef));
            return resource;
        })
        .toList();
}
```

**Deletion Flow:**

1. User deletes Kafka CR from central cluster
2. Kubernetes deletes the Kafka CR immediately
3. Operator's watch detects the deletion and triggers reconciliation
4. Since the CR no longer exists, the operator's `delete()` method is invoked
5. The `delete()` method explicitly deletes GC ConfigMap from each remote cluster:
   ```java
   private Future<Void> deleteGarbageCollectorConfigMaps(Reconciliation reconciliation) {
       for (String clusterId : remoteClusterIds) {
           ConfigMapOperator configMapOp = getConfigMapOperatorForCluster(clusterId);
           configMapOp.reconcile(reconciliation, namespace, gcConfigMapName, null);
       }
   }
   ```
6. Kubernetes garbage collector in each remote cluster cascades deletion to all resources owned by the GC ConfigMap
7. All StrimziPodSets, ConfigMaps, Services, PVCs in remote clusters are automatically deleted

**Key Design Points:**

The GC ConfigMap itself has **no ownerReferences** because cross-cluster ownership doesn't work.
Remote resources (Services, ConfigMaps, Secrets) have the GC ConfigMap as their owner (same-cluster ownership).
The operator explicitly manages the GC ConfigMap lifecycle (create on reconcile, delete on Kafka CR deletion).
This two-level ownership pattern enables proper garbage collection across cluster boundaries.

#### Kafka Configuration Generation

The networking provider is responsible for generating Kafka configuration that enables cross cluster communication.

**advertised.listeners Configuration:**

For each broker, the operator calls the provider's `generateAdvertisedListeners()` method:

```java
// Example: NodePort provider returns
"REPLICATION-9091://10.21.37.21:31001,PLAIN-9092://10.21.37.21:31002,TLS-9093://10.21.37.21:31003"

// Example: MCS provider returns
"REPLICATION-9091://broker-0.cluster-east.my-cluster-kafka-brokers.kafka.svc.clusterset.local:9091,..."
```

The operator writes this directly to the broker's configuration file.

**controller.quorum.voters Configuration:**

The operator collects all controller pods from the Kafka model and calls the networking provider's `generateQuorumVoters()` method:

```java
// Operator builds list of controller pods across all clusters
List<ControllerPodInfo> controllerInfos = new ArrayList<>();
for (NodeRef node : kafka.controllerNodes()) {
    controllerInfos.add(new ControllerPodInfo(
        node.nodeId(),           // e.g., 0, 1, 2
        node.podName(),          // e.g., "my-cluster-pool-central-0"
        getClusterIdForNode(node) // e.g., "cluster-central"
    ));
}

// Provider generates quorum voters string
String quorumVoters = networkingProvider.generateQuorumVoters(
    reconciliation, namespace, controllerInfos, "replication"
);

// Example: NodePort provider returns
"0@10.21.37.21:31093,1@10.21.37.22:31093,2@10.21.50.10:31093,..."

// Example: MCS provider returns
"0@my-cluster-pool-central-0.cluster-central.my-cluster-kafka-brokers.kafka.svc.clusterset.local:9091,..."
```

This configuration is written to the KRaft configuration file for all brokers and controllers.

**Certificate SANs for TLS:**

When TLS is enabled, broker and controller certificates include Subject Alternative Names for cross cluster DNS resolution.

For stretch mode, additional SANs are generated based on the provider's `generatePodDnsName()` and `generateServiceDnsName()` methods:

```bash
# Regular SANs (all modes)
DNS:my-cluster-broker-0.my-cluster-kafka-brokers.kafka.svc.cluster.local
DNS:my-cluster-kafka-brokers.kafka.svc

# Additional SANs in stretch mode (example from MCS provider)
DNS:my-cluster-broker-0.cluster-east.my-cluster-kafka-brokers.kafka.svc.clusterset.local
DNS:cluster-east.my-cluster-kafka-brokers.kafka.svc.clusterset.local
```

This ensures TLS hostname verification succeeds when brokers/controllers connect across clusters.

#### Remote Resource Creation

The central operator creates resources in remote clusters using the `RemoteResourceOperatorSupplier`:

```java
// Get operator supplier for target cluster
ResourceOperatorSupplier supplier = 
    remoteResourceOperatorSupplier.getResourceOperatorSupplier(targetClusterId);

// Create ConfigMap in remote cluster
ConfigMapOperator configMapOp = supplier.configMapOperations;
configMapOp.reconcile(reconciliation, namespace, configMapName, configMap)
    .onComplete(result -> {
        if (result.succeeded()) {
            LOGGER.info("Created ConfigMap {} in cluster {}", configMapName, targetClusterId);
        }
    });
```

The `RemoteResourceOperatorSupplier` maintains a map of cluster IDs to `ResourceOperatorSupplier` instances, each configured with the appropriate kubeconfig for that cluster.

#### Status Reporting

The Kafka CR status is enhanced to show comprehensive information about the stretch cluster deployment:

**Example Kafka CR Status (using MCS provider):**

```yaml
status:
  observedGeneration: 1
  kafkaVersion: 3.8.0
  kafkaMetadataVersion: 3.8-IV0
  kafkaMetadataState: KRaft
  clusterId: fPb3DwyWQpabxMy0SzaxUg
  operatorLastSuccessfulVersion: 0.48.0
  
  # Lists all KafkaNodePools in the cluster
  kafkaNodePools:
    - name: pool-central
    - name: pool-east
    - name: pool-west
  
  # Stretch cluster indicator condition
  conditions:
    - type: StretchCluster
      status: "True"
      reason: StretchClusterActive
      message: "Stretch cluster active: provider=mcs, central=cluster-central, clusters=cluster-central,cluster-east,cluster-west"
      lastTransitionTime: "2025-11-29T10:00:00Z"
    - type: Ready
      status: "True"
      lastTransitionTime: "2025-11-29T10:00:15Z"
  
  # Listener addresses from ALL clusters
  listeners:
    # Internal plain listener - shows DNS names from all 3 clusters
    - name: plain
      addresses:
        - host: my-cluster-kafka-bootstrap-plain.cluster-central.kafka.svc.clusterset.local
          port: 9092
        - host: my-cluster-kafka-bootstrap-plain.cluster-east.kafka.svc.clusterset.local
          port: 9092
        - host: my-cluster-kafka-bootstrap-plain.cluster-west.kafka.svc.clusterset.local
          port: 9092
      bootstrapServers: >-
        my-cluster-kafka-bootstrap-plain.cluster-central.kafka.svc.clusterset.local:9092,
        my-cluster-kafka-bootstrap-plain.cluster-east.kafka.svc.clusterset.local:9092,
        my-cluster-kafka-bootstrap-plain.cluster-west.kafka.svc.clusterset.local:9092
    
    # Internal TLS listener with certificates
    - name: tls
      addresses:
        - host: my-cluster-kafka-bootstrap-tls.cluster-central.kafka.svc.clusterset.local
          port: 9093
        - host: my-cluster-kafka-bootstrap-tls.cluster-east.kafka.svc.clusterset.local
          port: 9093
        - host: my-cluster-kafka-bootstrap-tls.cluster-west.kafka.svc.clusterset.local
          port: 9093
      bootstrapServers: >-
        my-cluster-kafka-bootstrap-tls.cluster-central.kafka.svc.clusterset.local:9093,
        my-cluster-kafka-bootstrap-tls.cluster-east.kafka.svc.clusterset.local:9093,
        my-cluster-kafka-bootstrap-tls.cluster-west.kafka.svc.clusterset.local:9093
      certificates:
        - |
          -----BEGIN CERTIFICATE-----
          MIIFLTCCAxWgAwIBAgIU... [REDACTED] ...
          -----END CERTIFICATE-----
    
    # External Route listener (OpenShift) - shows actual route hostnames
    - name: external
      type: route
      addresses:
        - host: my-cluster-kafka-external-bootstrap.apps.cluster-central.example.com
          port: 443
        - host: my-cluster-kafka-external-bootstrap.apps.cluster-east.example.com
          port: 443
        - host: my-cluster-kafka-external-bootstrap.apps.cluster-west.example.com
          port: 443
      bootstrapServers: >-
        my-cluster-kafka-external-bootstrap.apps.cluster-central.example.com:443,
        my-cluster-kafka-external-bootstrap.apps.cluster-east.example.com:443,
        my-cluster-kafka-external-bootstrap.apps.cluster-west.example.com:443
      certificates:
        - |
          -----BEGIN CERTIFICATE-----
          MIIFLTCCAxWgAwIBAgIU... [REDACTED] ...
          -----END CERTIFICATE-----
```

**Key Status Features:**

1. **`kafkaNodePools` field**: Lists all node pools participating in the stretch cluster
2. **`StretchCluster` condition**: Indicates stretch mode is active, shows provider name and cluster list
3. **Multi-cluster listener addresses**: Each listener shows bootstrap addresses from all clusters
4. **Provider-specific DNS**: Format depends on networking provider:
   - MCS: `service.clusterID.namespace.svc.clusterset.local`
   - NodePort: `nodeIP:nodePort`
   - LoadBalancer: `loadBalancerIP:port`

**KafkaNodePool Status:**

KafkaNodePool CR status remains unchanged from standard Strimzi deployments. The status fields are identical to single cluster mode:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: pool-east
  annotations:
    strimzi.io/stretch-cluster-alias: "cluster-east"
status:
  observedGeneration: 1
  replicas: 2
  nodeIds:
  - 1
  - 3
  roles:
    - controller
    - broker
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2025-11-29T10:00:00Z"
```

The target cluster information is only present in the metadata annotation (`strimzi.io/stretch-cluster-alias`), not in the status.

**Degraded State Handling:**

If a remote cluster becomes unavailable:
- The KafkaNodePool for that cluster transitions to `Ready: False` with standard error conditions
- The Kafka CR status shows degraded state but Kafka cluster continues operating with remaining nodes
- Client applications can still connect using bootstrap addresses from available clusters

#### Rolling Updates

Kafka pods follow the standard rolling update process, with the central operator determining which pods need restart based on:
- Revision hash changes (configuration updates)
- Certificate updates
- CA renewals
- Storage resizing

The operator uses the `strimzi.io/stretch-cluster-alias` annotation on each pod to determine the target cluster and performs the rolling restart via the appropriate `ResourceOperatorSupplier`.

#### Reconciliation Flow

The reconciliation flow for stretch clusters extends the standard Strimzi reconciliation. The key stretch specific steps are:

1. **Validation**: Validate stretch configuration using `StretchClusterValidator` (cluster IDs, kubeconfig validity, annotation consistency)
2. **Garbage Collector ConfigMap**: Create GC ConfigMaps in remote clusters for owner reference tracking
3. **Listeners Reconciliation**: Reconcile Services, Routes, Ingresses for each cluster with cluster specific endpoints
4. **Networking Resources**: Call provider's `createNetworkingResources()` for cross cluster pod communication (ServiceExports, LoadBalancers, etc.)
5. **CA and Certificate Secrets**: Create/update CA and broker/controller certificate secrets in all clusters with cross cluster DNS SANs
6. **Configuration Generation**: Call provider's `generateAdvertisedListeners()` and `generateQuorumVoters()` to build Kafka broker configuration
7. **ConfigMaps**: Generate and distribute broker/controller ConfigMaps to all clusters with stretch specific configuration
8. **StrimziPodSets**: Create StrimziPodSets in target clusters with GC ConfigMap owner reference for garbage collection
9. **Rolling Updates**: Perform rolling updates if configuration or pod specs changed
10. **Pods Ready**: Wait for all pods across all clusters to become ready
11. **Listener Status**: Aggregate listener addresses from all clusters (Routes, Ingresses, LoadBalancers, internal DNS)
12. **Kafka Status**: Update Kafka CR status with stretch cluster condition, listener addresses, and node pool statuses

The actual implementation includes additional steps for PVCs, service accounts, network policies, quotas, and cleanup operations. 
Each step handles both central and remote clusters with proper error handling and continues reconciliation even if individual clusters are temporarily unavailable.


### Testing Strategy

#### Unit and Integration Tests

The stretch cluster implementation includes comprehensive test coverage:

**Plugin Loading Tests** (`StretchNetworkingProviderPluginLoadingTest`)
- Plugin discovery via ServiceLoader
- ClassLoader isolation
- Provider initialization with valid/invalid configurations
- Error handling for missing plugins or initialization failures

**Integration Tests** (`StretchNetworkingProviderIntegrationTest`)
- Full reconciliation flow with test provider
- Endpoint discovery and configuration generation
- Resource creation and deletion
- Multi cluster scenarios

**Reconciliation Flow Tests** (`StretchReconciliationFlowTest`)
- Kafka CR reconciliation with stretch mode enabled
- StrimziPodSet creation in remote clusters
- ConfigMap generation with custom configuration
- Garbage collection flow

**Test Provider** (`TestNetworkingProvider`)
- Fully functional reference implementation
- In-memory endpoint storage
- Configurable behavior for testing error scenarios
- Validates provider contract compliance



#### System Testing Approach

The feature design enables comprehensive system testing:
- Multi-cluster Kafka deployment validation
- Pod creation and lifecycle across all clusters
- Produce/consume functionality testing
- Rolling update procedures
- Cluster failure scenarios
- Garbage collection verification

#### Test Coverage

The stretch cluster feature includes comprehensive automated test coverage:

**Unit and Integration Tests:**
- **Plugin Loading Tests**: Validate plugin discovery, ClassLoader isolation, initialization, and error handling
- **SPI Contract Tests**: Full provider lifecycle testing using MockKube3 (3-cluster simulation)
- **Validation Tests**: Operator configuration, Kafka CR validation, error codes and messages
- **Reconciliation Tests**: Stretch mode annotations, scaling operations, configuration propagation
- **Test Provider**: Reference implementation demonstrating complete SPI contract

**System Testing:**

The plugin architecture enables flexible system testing approaches:

- End-to-end validation can be performed with any networking provider (NodePort, LoadBalancer, MCS)
- Multi-cluster test environments can use lightweight solutions (kind, minikube)
- Community members can validate the feature with their preferred networking solutions

Standard Strimzi system test infrastructure will validate basic stretch cluster functionality with NodePort provider.

#### Performance Testing Methodology

**Tools:**
- **OpenMessaging Benchmark (OMB):** Throughput and latency measurement
- **Chaos Mesh:** Network latency injection for sensitivity testing
- **Automation:** Python scripts for automated test execution and result collection

**Test Scenarios:**
1. **Baseline:** same datacenter deployment (validates full throughput)
2. **Latency Sensitivity:** Progressive latency injection (0ms, 10ms, 50ms, 100ms)
3. **Breaking Point:** Binary search to find Kafka stability threshold

**Conservative Test Methodology:**
Latency sensitivity tests use Chaos Mesh NetworkChaos CR to inject artificial delay between **all pods in all clusters**, including intra-cluster communication. 
This represents a **worst-case scenario** more pessimistic than real world deployments, where only cross cluster communication experiences added latency. 
Real production performance in same datacenter multi-AZ deployments will be significantly better than these conservative test results.

**Validation Criteria:**
- same datacenter deployment maintains >= 90% of single cluster throughput
- P99 latency remains < 500ms for same site deployments
- Zero pod crashes or restarts under normal conditions
- Graceful degradation under network latency (no crashes until 750ms threshold)

**Validation Results:**

Performance testing using OMB and Chaos Mesh validates the performance characteristics described in the "Performance Characteristics and Operational Guidelines" section. 
The testing methodology is documented and can be replicated by users to validate their own deployments.

#### Validation Approach

**Operator and Kafka Upgrades:**

Standard Strimzi upgrade testing applies to stretch clusters. 
The implementation includes validation that all operators run compatible versions, and standard rolling update mechanisms work across multiple clusters.

**Plugin Testing:**

Reference implementations (NodePort, LoadBalancer) maintained by Strimzi include comprehensive test coverage. 
Third party plugins (e.g., MCS) are maintained independently with their own testing approaches.
The SPI design allows plugin authors to validate their implementations without requiring changes to core Strimzi tests.

### Additional Considerations

#### External Access

External listener types (Route, Ingress, LoadBalancer, NodePort) work in stretch mode:
- Each cluster exposes its own external endpoints
- Kafka CR status lists all bootstrap addresses from all clusters
- Clients can connect to any cluster's bootstrap address
- Kafka handles client redirection to appropriate brokers

Example with NodePort external listener:

```yaml
status:
  listeners:
    - name: external
      type: nodeport
      addresses:
        - host: node1.cluster-central.example.com
          port: 32100
        - host: node1.cluster-east.example.com
          port: 32100
        - host: node1.cluster-west.example.com
          port: 32100
      bootstrapServers: >-
        node1.cluster-central.example.com:32100,
        node1.cluster-east.example.com:32100,
        node1.cluster-west.example.com:32100
```

#### Rack Awareness

Rack awareness works across clusters by using topology labels:
- Each cluster can have different zone labels
- The `strimzi.io/stretch-cluster-alias` annotation serves as the primary topology key
- Additional zone labels within each cluster provide finer-grained replica distribution

Example topology configuration:

```yaml
spec:
  kafka:
    rack:
      topologyKey: topology.kubernetes.io/zone
```

Kafka distributes replicas considering both cluster boundaries and zone labels within clusters.

#### Disaster Recovery

If the central cluster fails, Kafka brokers and controllers continue operating.
Administrative control can be restored using one of these approaches:

**Manual Recovery:** (I think we should verify this. I don’t recall how we tested it earlier, other than shutting down the Fyre (central) cluster and rebooting it after some time)
1. Deploy a new Kubernetes cluster or select an existing remote cluster
2. Deploy the cluster operator
3. Apply the Kafka and KafkaNodePool CRs
4. Update the `STRIMZI_CENTRAL_CLUSTER_ID` to the new cluster's ID
5. Operator resumes management of the stretch cluster

**GitOps Recovery:**
1. Store all CRs in a Git repository managed by ArgoCD or Flux
2. GitOps tool automatically applies CRs to the new central cluster
3. Operator resumes management

The key insight is that the Kafka cluster continues serving client requests during central cluster failure.
Only administrative operations (scaling, configuration changes) are temporarily unavailable.

### Performance Characteristics and Operational Guidelines

Extensive testing validates that stretch Kafka clusters deliver excellent performance for **same datacenter multi-availability-zone deployments**, the primary and recommended use case for this feature.

#### Validated Performance - same datacenter Deployment

Testing was performed across three OpenShift clusters deployed within the same data center, representing the optimal deployment scenario for stretch clusters.

**Test Configuration:**
- **Infrastructure:** 3 OpenShift clusters, same site (< 1ms inter-cluster latency)
- **Kafka Topology:** 5 controllers (2+2+1), 27 brokers (9+9+9)
- **Networking:** Cilium with Multi-Cluster Services (MCS) API
- **Testing Tool:** OpenMessaging Benchmark (OMB) with Kafka driver
- **Workload:** 1 topic, 16 partitions, 1KB messages
- **Latency Testing:** Chaos Mesh NetworkChaos CR for controlled latency injection

**Baseline Results (0ms Added Latency):**

| Metric | same site Stretch Cluster | Expected single cluster | Overhead |
|--------|--------------------------|-------------------------|----------|
| **Throughput** | **50,000 msg/sec** | ~50,000 msg/sec | **0%** ✅ |
| Avg Latency | 80ms | 45ms | +78% |
| P99 Latency | 494ms | 175ms | +182% |
| Error Rate | 0% | 0% | 0% |

**Key Finding:** Stretch clusters deployed in same datacenter configurations **maintain full throughput** while providing high availability across independent Kubernetes clusters and availability zones. The latency increase is acceptable for most production workloads and provides significant operational benefits through fault domain isolation.

**Testing Methodology Note:** Additional latency sensitivity tests (10ms, 50ms, 100ms) used Chaos Mesh to inject artificial latency between **all pods**, including those within the same cluster.
This represents a **worst-case scenario**, real world same datacenter deployments only experience latency on cross cluster communication, resulting in significantly better performance than these conservative test results suggest.

#### Use Case Suitability

Stretch Kafka clusters are **optimized for specific deployment scenarios**. The following guidance helps users determine when stretch clusters provide value:

##### ✅ Recommended: same datacenter Multi-AZ (Primary Use Case)

**Scenario:** Multiple Kubernetes clusters within the same data center, across different availability zones.

**Network Characteristics:**
- Inter-cluster latency: < 5ms (validated < 1ms in testing)
- Packet loss: < 0.1%
- Jitter: < 2ms
- Bandwidth: >= 10 Gbps

<Let's add the Graphs here before we submit the proposal>

**Expected Performance:**
- **Throughput:** 90-100% of single cluster baseline ✅
- **Latency:** +50-100% overhead (acceptable for HA benefit)
- **Availability:** Survives entire Kubernetes cluster failure

**Benefits:**
- **High Availability:** Independent control planes eliminate single point of failure
- **Fault Isolation:** Failure of one Kubernetes cluster (control plane, etcd, networking) does not impact Kafka
- **Compliance:** Meets requirements for separate failure domains
- **Performance:** Excellent - full throughput maintained

**real world Validation:** Testing demonstrated 50,000 msg/sec throughput with 80ms average latency across 3 clusters in same datacenter.

**Example Architecture:**
```
Datacenter: US-East-1
├── Kubernetes Cluster 1 (AZ-1a): 2 controllers, 9 brokers
├── Kubernetes Cluster 2 (AZ-1b): 2 controllers, 9 brokers
└── Kubernetes Cluster 3 (AZ-1c): 1 controller, 9 brokers

Network: Low-latency private network (< 1ms)
Benefit: Survives entire AZ failure or K8s control plane outage
```

##### Use with Caution: Metro Area

**Scenario:** Kubernetes clusters in nearby datacenters within metro area (< 50km).

**Network Characteristics:**
- Inter-cluster latency: 5-50ms
- Requires highly stable, dedicated network links

**Expected Performance:**
- **Throughput:** 10-30% of baseline (severe degradation)
- **Latency:** 1-10 seconds average
- **Stability:** Sensitive to network jitter

**Testing Data:** At 10ms added latency, throughput drops to 17,000 msg/sec (66% reduction). At 50ms, only 4,300 msg/sec remains (91% reduction).

**IMPORTANT - Conservative Test Methodology:** These latency sensitivity tests represent a **worst-case scenario**. The test methodology injected artificial latency between **ALL pods in ALL clusters**, including pods within the same Kubernetes cluster and namespace. In real world same datacenter deployments, only cross cluster communication would experience added latency (typically < 1ms for same site clusters), while intra-cluster communication remains at normal Kubernetes latency (< 0.5ms). This means **actual production performance in same datacenter multi-AZ deployments will be significantly better** than the degraded numbers shown above, approaching the 0ms baseline performance (50,000 msg/sec).


##### NOT Recommended: Regional or cross region

**Scenarios to Avoid:**
| Deployment | Latency | Reason |
|------------|---------|--------|
| Regional (> 200km) | > 100ms | 96%+ throughput loss, constant publish timeouts |
| cross region | > 150ms | System effectively unusable |
| Intercontinental | > 250ms | Constant failures |

**Alternative:** Use **MirrorMaker 2** for cross region disaster recovery. MM2 provides asynchronous replication optimized for high-latency scenarios and is the correct tool for geographic distribution.
----
#### Network Requirements and Pre-Deployment Validation

#### Optional: Advanced Network Validation

**Automated Validation:**

The Strimzi operator automatically validates network latency when you deploy a stretch
cluster. It deploys test pods, measures TCP latency, and blocks deployment if latency
exceeds thresholds. This validation runs automatically - no user action required.

**Optional Manual Pre-Validation:**

For high-confidence production deployments, users may optionally perform additional
validation before applying the Kafka CR:

**1. Extended Latency Test (Optional - 10-minute continuous ping):**

The operator samples latency 5 times. For additional confidence, run an extended test:

```bash
kubectl run -it --rm latency-test --image=nicolaka/netshoot --restart=Never -- \
  ping -c 600 <remote-cluster-service-ip> | tee latency-results.txt

# Analyze results
grep 'rtt min/avg/max/mdev' latency-results.txt
```
**2. Bandwidth Test (Recommended for production):**

The operator validates latency but not bandwidth. 
For production deployments, validate you have sufficient bandwidth:

```bash
# On cluster-1: start iperf3 server
kubectl run iperf-server --image=networkstatic/iperf3 -- -s

# On cluster-2: run client test
kubectl run -it --rm iperf-client --image=networkstatic/iperf3 -- \
  -c <iperf-server-ip> -t 60 -P 10
```

Expected: >= 10 Gbps for production deployments.

**3. Chaos Mesh Pre-Validation (Highly Recommended):**

Before production deployment, simulate expected latency and validate Kafka behavior under realistic conditions:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: stretch-cluster-validation
spec:
  action: delay
  delay:
    latency: "5ms"  # Use your measured inter-cluster latency
    correlation: "100"
    jitter: "1ms"
  selector:
    namespaces: [kafka-namespace]
    labelSelectors:
      app.kubernetes.io/instance: test-cluster
  mode: all
  duration: "10m"
```

Deploy a test Kafka cluster, apply NetworkChaos, run OpenMessaging Benchmark. 
If performance meets requirements, proceed with production deployment.

Note: These manual tests are optional. 
The operator's automated validation will block unsuitable deployments. 
However, for production environments, performing these additional tests provides extra confidence and helps user understand their network characteristics.



#### Summary: When to Use Stretch Clusters

**✅ Use Stretch Clusters When:**
- Deploying across multiple AZs in same datacenter (< 5ms latency)
- Requiring strong consistency (synchronous replication)
- Need automatic client failover without application changes
- Organizational requirements for separate Kubernetes control planes
- Compliance requirements for independent failure domains

**❌ Use MirrorMaker 2 Instead When:**
- Deploying across regions (> 50ms latency)
- Geographic disaster recovery is primary goal
- Can tolerate asynchronous replication
- Prefer simpler operational model

**Key Takeaway:** Stretch clusters excel at providing high availability within a single datacenter while maintaining full Kafka performance.
They are not a replacement for cross region disaster recovery, which is better served by MirrorMaker 2.

#### Authentication and Security

The operator is agnostic to the authentication method used for remote cluster access.
Common approaches include:

- **ServiceAccount tokens**: Long-lived token stored in Secret
- **mTLS certificates**: Client certificate authentication
- **OIDC**: Integration with identity provider
- **Cloud IAM**: AWS IAM, GCP IAM, Azure AD

Users must ensure kubeconfig credentials are properly secured and rotated.
The operator validates kubeconfig expiry and reports errors before credentials expire.

### Kafka Ecosystem Components

#### Deployment Strategy for Connect, Bridge, and MirrorMaker 2

**Recommendation:** All Kafka ecosystem components should be deployed in the **central cluster only**.

| Component | Deployment Location | Reason |
|-----------|---------------------|--------|
| **KafkaConnect** | Central cluster | Simplifies management, connects as Kafka client |
| **KafkaBridge** | Central cluster | HTTP/REST proxy, connects as Kafka client |
| **KafkaMirrorMaker2** | Central cluster | For replication to external (non-stretch) clusters |

**Rationale:**
- These components are Kafka **clients**, not part of the Kafka cluster itself
- Deploying in central cluster provides single control plane
- Operator in remote clusters ignores these CRs (validated and rejected)
- Simplifies lifecycle management and upgrades

**cross cluster Access:** (These are my thoughts we need to discuss internally whether this will be acceptable for the community)
If applications in remote clusters need Connect or Bridge access:
- Expose via Kubernetes Service (ClusterIP, LoadBalancer, or Ingress)
- Use standard Kubernetes cross cluster networking
- Consider locality: Deploy Connect workers in each cluster if needed for performance

**Consumer Locality Optimization:**
Connect workers can use `client.rack` consumer configuration to prefer local brokers:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect
spec:
  config:
    client.rack: cluster-east  # Matches strimzi.io/stretch-cluster-alias
```

This reduces cross cluster traffic for consuming by fetching from in-sync replicas in the same cluster.

**Validation:**
The operator validates that `KafkaConnect`, `KafkaBridge`, and `KafkaMirrorMaker2` CRs are only created in the namespace where the Kafka CR exists (central cluster).
Attempts to create these resources in remote clusters are rejected with validation errors.

#### Entity Operator

**Deployment:**
- Deployed only in the central cluster
- Manages KafkaTopic and KafkaUser resources
- No changes required for stretch cluster support

**Functionality:**
- Standard Strimzi Topic Operator for topic management
- Standard Strimzi User Operator for ACL and SCRAM user management
- Connects to Kafka using standard bootstrap addresses

#### Drain Cleaner

**Deployment:**
- Deployed to all clusters
- Functions independently in each cluster
- No changes required

**Behavior:**
- Evicts pods during node drain operations
- Each cluster's Drain Cleaner only affects pods in that cluster
- No cross cluster coordination needed

## Affected/Not Affected Projects

This proposal impacts only the `strimzi-kafka-operator` project.

Changes include:
- New `StretchNetworkingProvider` SPI in `cluster-operator` module
- New `StretchNetworkingProviderFactory` for plugin loading
- New `RemoteResourceOperatorSupplier` for multi-cluster access
- Enhanced `KafkaReconciler` with stretch cluster logic
- New `StretchClusterConfig` and `StretchClusterValidator` for configuration management
- Garbage collection ConfigMap implementation
- Enhanced status reporting

No changes to:
- `strimzi-topic-operator`
- `strimzi-user-operator`
- `kafka-agent`
- `strimzi-drain-cleaner`
- CRD definitions (annotations only)

## Rejected Alternatives

### Built-in Networking Providers

**Rejected:** Including NodePort, LoadBalancer, or MCS providers in the core operator.

**Reason:** 
- Increases operator complexity and binary size
- Creates maintenance burden for networking code
- Limits extensibility for cloud-specific solutions
- Makes testing more difficult

**Alternative:** All providers are external plugins, including Strimzi-maintained reference implementations.

### Automated Central Cluster Failover

**Rejected:** Automatic detection of central cluster failure and promotion of a remote cluster.

**Reason:**
- Requires distributed consensus mechanism (Raft, etcd, ZooKeeper)
- Complex state replication across clusters
- Risk of split-brain scenarios
- Significant implementation complexity

**Alternative:** Manual or GitOps-based recovery, which is simpler and more predictable.

### CR Replication with Standby Mode

**Rejected:** Replicating Kafka and KafkaNodePool CRs to remote clusters with `strimzi.io/standby=true` annotation.

**Reason:**
- Adds complexity in keeping replicated CRs in sync
- Risk of configuration drift between copies
- Requires additional logic to activate standby CRs during failover

**Alternative:** GitOps tools naturally provide CR storage and replication to new clusters.

### Service Mesh Integration

**Rejected:** Direct integration with service mesh solutions (Istio, Linkerd, Consul).

**Reason:**
- Service meshes solve different problems (traffic management, observability)
- Not all users want service mesh overhead
- Can be used alongside stretch clusters if desired

**Alternative:** Plugin architecture allows users to implement service mesh-aware providers if needed.

## Compatibility and Migration

### Operator Version Compatibility

**Requirement:** All operators in a stretch deployment MUST run the same Strimzi version.

**Reason:** Stretch clusters require coordinated reconciliation across clusters. Version mismatches can cause:
- Incompatible StrimziPodSet reconciliation logic
- Status reporting discrepancies
- Resource format incompatibilities

**Validation:** The central operator validates remote operator versions during startup by querying the Kubernetes API server version and checking for Strimzi-specific CRDs.
If version mismatches are detected, the operator:
1. Logs critical error with version details
2. Skips stretch cluster reconciliation for safety
3. Continues normal reconciliation for non-stretch Kafka clusters

**Upgrade Process:**

1. **Plan upgrade window:** Brief disruption during operator restarts
2. **Upgrade central operator:** Update Deployment in central cluster
3. **Upgrade remote operators:** Update Deployments in remote clusters (recommended: use GitOps for consistency)
4. **Validation:** Central operator automatically validates version compatibility on startup
5. **Resume operations:** Stretch cluster reconciliation resumes if validation passes

**Rollback:** If issues occur, downgrade all operators to previous version together.

### Kafka Version Upgrades

Kafka version upgrades follow standard Strimzi rolling update procedures with no special stretch specific handling:

**Process:**
1. Central operator orchestrates rolling update
2. Updates happen pod-by-pod to maintain quorum
3. Each cluster's brokers/controllers are updated independently
4. No client downtime if quorum is maintained

**Example:**
```bash
# Standard Kafka version upgrade (no stretch specific changes)
kubectl patch kafka my-cluster --type merge -p \
  '{"spec":{"kafka":{"version":"4.0.0"}}}'
```

The operator handles rolling updates across all clusters while ensuring Kafka quorum is maintained.

### Plugin Compatibility

**SPI Stability:** The `StretchNetworkingProvider` interface follows semantic versioning:
- **Minor version changes:** Backward-compatible additions (new optional methods)
- **Major version changes:** Breaking changes (method signature changes, removals)

**Plugin Validation:** Operator validates plugin compatibility at startup:
1. Loads plugin via ServiceLoader
2. Checks plugin-declared operator version requirements
3. Fails fast with clear error if incompatible

**Deprecation Policy:** Breaking SPI changes trigger major version bump. Old plugins continue working with deprecation warnings for 2 releases before removal.

### Migration Scenarios

#### Converting Regular Cluster to Stretch Cluster

**Status:** NOT supported for in place conversion in v1.

**Reason:** Converting existing single cluster deployments to stretch mode requires:
- Redistributing existing pods across clusters
- Changing pod network addressing
- Updating TLS certificates with new SANs
- Potential data movement

**Recommended Approach:**
1. Create new stretch cluster with desired topology
2. Use MirrorMaker 2 to replicate topics from old cluster to new
3. Switch clients to new cluster
4. Decommission old cluster after validation

**Future Consideration:** In-place migration may be explored if there is user demand.

#### Converting Stretch Cluster to Regular Cluster

**Scenario:** Consolidating stretch cluster to single Kubernetes cluster.

**Process:**
1. Remove `strimzi.io/enable-stretch-cluster: "true"` annotation from Kafka CR
2. Remove `strimzi.io/stretch-cluster-alias` annotations from all KafkaNodePools
3. Update all KafkaNodePool `targetCluster` to point to same cluster
4. Operator performs rolling update to reconfigure brokers with new networking
5. Manually delete stretch networking resources from remote clusters

**Limitation:** All pods must be schedulable to target cluster. If resources are insufficient, operation fails.

### Backward Compatibility

**Existing Clusters:** Unaffected by stretch cluster feature unless explicitly enabled.

**No CRD Changes:** All stretch configuration uses annotations (no CRD schema changes).

**No Behavior Changes:** Operators without stretch configuration behave identically to previous versions.

**Upgrade Path:** Users can upgrade to operators with stretch support without any changes to existing deployments.

## Stability and Maturity

**API Stability:** The stretch cluster API (annotations and configuration) is considered **beta** quality in the initial release.

**Expected Changes:**
- Annotation names are stable and will not change
- SPI interface may receive additions (backward-compatible)
- Breaking SPI changes will trigger major version bump with 2-release deprecation period

**Production Readiness:**
- Recommended for production use in **same datacenter multi-AZ deployments**
- Extensive testing validates performance and stability (50,000 msg/sec throughput with < 1ms latency)
- Early adopters encouraged to test in non-production environments first
- Performance characteristics well-documented based on real world testing

**Future Enhancements:**
- In-place migration from regular to stretch clusters
- Automated central cluster failover (if demand exists)
- Additional provider implementations

## Summary

This proposal introduces stretch Kafka cluster support to Strimzi through:

1. **Plugin Architecture**: Service Provider Interface (SPI) for networking implementations
2. **Multi-Cluster Management**: Central operator creates resources across clusters
3. **Garbage Collection**: ConfigMap-based owner references for proper cleanup
4. **Flexible Networking**: Support for MCS, NodePort, LoadBalancer, and custom providers
5. **High Availability**: Kafka continues operating even if central cluster fails
6. **Clean Separation**: Networking logic in plugins, core operator focuses on orchestration

The implementation maintains backward compatibility, requires no CRD changes, and provides a clear upgrade path for users.

