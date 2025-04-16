# Stretch Kafka cluster

The Strimzi Kafka Operator currently manages Kafka clusters within a single Kubernetes cluster.
This proposal aims to extend its support to stretch Kafka clusters, where the brokers and controllers of a single Kafka cluster are distributed across multiple Kubernetes clusters.

## Current situation

At present, the availability of Strimzi-managed Kafka clusters is constrained by the availability of the underlying Kubernetes cluster.
If a Kubernetes cluster experiences an outage, the entire Kafka cluster becomes unavailable, disrupting all connected Kafka clients.

While it is possible to enhance availability by running Kubernetes across multiple availability zones and configuring Strimzi with affinity rules, tolerations, or topology spread constraints, this approach is limited to a single Kubernetes control plane. 
Additionally, if not self-hosted, deploying across multiple availability zones typically incurs higher cloud hosting costs. 
A stretch Kafka cluster offers an alternative approach by distributing Kafka brokers and controllers across multiple Kubernetes clusters, reducing dependency on a single cluster's control plane while improving resilience.

## Motivation

A stretch Kafka cluster allows Kafka nodes to be distributed across multiple Kubernetes clusters. This approach also facilitates many valuable use cases, such as:

- **High Availability**: Distributing Kafka brokers across multiple Kubernetes clusters significantly enhances resilience by enabling the system to tolerate the outage of an entire Kubernetes cluster without disrupting service to clients. 
Unlike MirrorMaker 2 (MM2), a stretch cluster provides strong data durability through synchronous replication and enables fast disaster recovery with automated client failover

- **Migration Flexibility**: A stretch Kafka cluster enables seamless migration, whether it's moving the entire cluster across Kubernetes environments or cloud providers without downtime, or relocating individual Kafka nodes as needed. 
This flexibility helps with maintenance, scaling, and workload transitions between environments.

- **Resource Optimization**: Efficiently utilizing resources across multiple clusters, which can be advantageous in environments with varying cluster capacities or during scaling operations.

## Proposal

This proposal seeks to enhance the Strimzi Kafka operator to support stretch Kafka clusters, distributing broker, controller and combined Kafka nodes across multiple Kubernetes clusters.
The intent is to focus on high-availability of the Kafka data plane.
The proposal outlines high-level topology and design concepts for such deployments, with a plan to incrementally include finer design and implementation details for relevant aspects.

### Limitations and Considerations
While a stretch Kafka cluster offers several advantages, it also introduces some challenges and considerations:

- **Increased Network Complexity and Costs**: The communication between brokers and controllers across clusters relies on network connectivity, which can be less reliable and more costly than intra-cluster communication.
This necessitates careful consideration of network architecture and associated costs.

- **Latency Requirements**: Stretch Kafka clusters are best suited for environments with low-latency and high-bandwidth network connections between the Kubernetes clusters.
High latency can adversely affect the performance and synchronization of Kafka nodes, potentially leading to delays or errors in replication and client communication.
Defining the minimal acceptable latency between clusters is crucial to ensure optimal performance.

### Prerequisites

- **KRaft**: This proposal focuses exclusively on enabling stretch deployments for KRaft-based Kafka clusters.
Strimzi will not support stretch clusters for Zookeeper-based deployments.

- **Multiple Kubernetes clusters**: Stretch Kafka clusters require multiple Kubernetes clusters.
The recommended minimum number of Kubernetes clusters is 3 because this is the minimum number of controllers required to establish quorum and provide High Availability (HA).
However, the cluster operator will not block deployment of a stretch cluster with fewer than 3 Kubernetes clusters as this will be useful for test and development use cases.
The cluster operator will log a warning similar to the existing warning logged when deploying fewer than 3 ZooKeeper nodes.

- **Low Latency and High Bandwidth**:  Kafka clusters should be deployed in environments that provide low-latency and high-bandwidth communication between Kafka brokers and controllers.
Stretch Kafka clusters should therefore be deployed in data centers or availability zones within a single region.
Deployments across geographically distant regions, where high latency and limited bandwidth could impair performance, should be avoided.

- **A supported cloud native networking technology**: To enable networking between Kubernetes clusters currently requires an additional technology stack.
Manual configuration of a [Cloud Native Network](https://landscape.cncf.io/guide#runtime--cloud-native-network) project is required.
Any project that can support direct pod-to-pod communication across multiple Kubernetes clusters should be compatible with the proposed solution.
The initial implementation will be based upon [Cilium](https://cilium.io/) that provides support for a [Cluster Mesh](https://docs.cilium.io/en/stable/network/clustermesh/intro/), but the design will allow for this to be extended to other mature CNCF networking projects.

A [prototype](#prototype) has been proven using Cilium and [Submariner](https://submariner.io/).
Submariner provides an implementation of [Kubernetes Multi-Cluster Services (MCS)](https://multicluster.sigs.k8s.io/guides/#implementation-status).

### High-level architecture

#### Topology of a stretch cluster

![Stretch cluster topology](./images/095-stretch-cluster-topology.png)

The diagram illustrates a topology comprising of three Kubernetes clusters.

One of these clusters is designated as the "Central cluster", while any additional clusters are considered "remote".
The central cluster acts as the control plane where a user will create all the custom resources for the Kafka cluster - Kafka, KafkaNodePool, KafkaUser, KafkaTopic etc.

A Kafka node pool definition can be configured to specify a Kubernetes cluster (central cluster or one of the remote clusters) as the deployment target.
The operator on the central cluster is responsible for creating all necessary resources for the node pool on the target Kubernetes cluster. 

### Low-level design and prototype

The following sections will describe the key aspects of the proposal.

#### Step 1: User configuration of Cilium cluster mesh

An overlay network must be [configured manually using Cilium](https://aswinayyolath.github.io/stretch-kafka-docs/Setting-up-cilium/).
When installing Cilium on a Kubernetes cluster, the user must specify a `cluster.name` to define a unique identifier for each cluster.

The Strimzi cluster operator will use this identifier in two ways:
1. Generating `advertised.listeners` and `controller.quorum.voters` configuration.
2. Determining the target Kubernetes cluster for node pool resources.

Kubernetes [CoreDNS must also be reconfigured](https://aswinayyolath.github.io/stretch-kafka-docs/Setting-up-cilium/#setting-up-coredns-for-multi-cluster-dns-resolution) to allow DNS resolution across multiple Kubernetes clusters.

#### Step 2: Deploy a cluster operator to each Kubernetes cluster

##### Central cluster operator configuration

The operator running in the central Kubernetes cluster must be provided with the following information to allow creation of resources in remote clusters:

- The identifier for each Kubernetes cluster defined in 'Step 1'.
- A URL endpoint for the Kubernetes API server running in each remote cluster.
- Credential(s) to allow authentication with the remote Kubernetes API server(s).
- The cross-cluster networking technology used for stretch cluster communication (e.g., Cilium, Submariner,...).

The information outlined above will be provided as environment variables for the cluster operator.
The values of the environment variables use the following format:

```yaml
- name: STRIMZI_REMOTE_KUBE_CONFIG
  value: |
      cluster-id-a.url=<cluster-a URL>
      cluster-id-a.secret=<secret-name-cluster-a>
      cluster-id-b.url=<cluster-b URL>
      cluster-id-b.secret=<secret-name-cluster-b>

- name: STRIMZI_STRETCH_CLUSTER_NETWORK
  value: "cilium" # or any other supported networking technology
```

The secrets referenced here must contain the kubeconfig for the Kubernetes cluster available at the provided URL as the value of secret key 'kubeconfig'.
This allows the central Strimzi operator to authenticate with multiple Kubernetes clusters.

**Example Secret**

Below is an example Kubernetes Secret containing a kubeconfig for a remote cluster

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-name-cluster-a
  namespace: strimzi
type: Opaque
data:
  kubeconfig: <base64-encoded-kubeconfig>
```

When the `STRIMZI_REMOTE_KUBE_CONFIG` environment variable is set, the Cluster Operator will recognize that it needs to deploy a stretch Kafka cluster. 
As part of this mode:
The custom actions taken as a result of this environment variable include:
1. The value of `STRIMZI_NETWORK_POLICY_GENERATION` is always treated as **`false`**, regardless of the value provided by the user or the default (true).
This is to ensure that network traffic between Kubernetes clusters is not blocked due to network policies.
The Cluster Operator will log a warning to inform the user that any value set for this variable is being ignored in stretch mode.
2. The `StrimziPodSet` resources created in a remote cluster will include an annotation `strimzi.io/remote-podset: true`.
This annotation will allow the remote cluster operator to reconcile the `StrimziPodSet` without a `Kafka` and `KafkaNodePool` CR being present in the same cluster.


The `STRIMZI_STRETCH_CLUSTER_NETWORK` environment variable defines the cloud-native networking technology used for cross-cluster communication in a stretch Kafka deployment.
Specifying this value in the Cluster Operator deployment is essential because different networking technologies may require different handling or integration steps by the operator.
For example, when using Submariner, the operator needs to create additional Kubernetes resources such as ServiceExport objects to enable seamless service discovery and communication across clusters.
In contrast, technologies like Cilium Cluster Mesh handle cross-cluster service resolution automatically and do not require any extra configuration from the operator.
By making this a required environment variable, we ensure that the operator is explicitly informed about the underlying network technology and can perform any necessary setup accordingly.
This approach also makes the implementation extensible and future-proof, as additional technologies can be supported by adding their respective handling logic to the operator based on this variable.


If the `STRIMZI_STRETCH_CLUSTER_NETWORK` environment variable is not explicitly defined, the Cluster Operator will default to "cilium", as it does not require any operator-driven configuration for cross-cluster service discovery.
A warning will be logged to inform users of this default behavior.
The warning will look something like:

```
STRIMZI_STRETCH_CLUSTER_NETWORK is not defined. Defaulting to 'cilium'.
```

This approach ensures that stretch mode remains functional even if the variable is omitted.
However, if the user has not set up any cross-cluster networking technology (such as Cilium, Submariner, or others) in the underlying Kubernetes clusters, the stretch cluster will not function correctly regardless of the operator configuration.
It is the user’s responsibility to ensure that appropriate cross-cluster networking is in place for pod-to-pod communication across clusters.

##### Remote cluster operator configuration

To optimize resource usage in the remote cluster, the operator can optionally be configured to reconcile only `StrimziPodSet` resources by setting the existing environment variable:

```yaml
- name: STRIMZI_POD_SET_RECONCILIATION_ONLY
  value: true
```

This is not required for stretch clusters to function correctly.
If this variable is not set, the operator will still work as expected.
It simply won't benefit from the reduced memory/CPU footprint and cleaner logs that come with narrowing the scope of reconciliation.
This configuration will minimise the resources required by the cluster operator on the remote Kubernetes clusters and simplify logging output.

#### Step 3: Create Kafka and KafkaNodePool resources in the central cluster

##### Kafka CR

There is no change required to the Kafka custom resource (Kafka CR).
The Kafka resource remains exactly as defined in the existing Strimzi API, and users can continue to define their cluster configurations in the standard way.
Most stretch-related configuration — such as cross-cluster networking technology and remote cluster access credentials — is handled through environment variables in the Cluster Operator deployment.
This simplifies the Kafka CR and keeps global operational concerns decoupled from workload specifications.
Cluster-specific configuration, such as the target remote cluster for each node pool, is specified in the KafkaNodePool CR via annotations.

##### KafkaNodePool CR

The `KafkaNodePool` CR remains in the central cluster but must include a new annotation to indicate the target Kubernetes cluster where the corresponding Kafka node resources should be deployed.
Based on this annotation, the Cluster Operator will create the associated resources — including StrimziPodSet, Services, Secrets, ConfigMaps, and others — in the specified remote cluster.
The `KafkaNodePool` itself is never deployed outside the central cluster.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  annotations:
    strimzi.io/stretch-cluster-id: "cluster3"
```

The identifier used here must match one of the values defined by the user in 'Step 1' and consequently added to the environment variable map value in 'Step 2'.
If this annotation is omitted or the identifier does not match any of the values defined in the `STRIMZI_REMOTE_KUBE_CONFIG` environment variable, the operator will deploy the node pool to the central Kubernetes cluster.

#### Remote cluster resources created by the central operator in a stretch cluster

In a stretch cluster, the Kafka and KafkaNodePool CRs are created only in the central cluster.
However, these CRs will trigger the cluster operator to create several Kubernetes resources that are required for Kafka broker and controller pods to start successfully.
The following table outlines the Kubernetes resources created by each Strimzi Custom Resource (`Kafka`, `KafkaNodePool`, and `StrimziPodSet`).

| Kubernetes Resource                          | Created by Kafka CR | Created by KafkaNodePool CR | Created by StrimziPodSet CR |
|----------------------------------------------|----------------------|----------------------------|----------------------------|
| **Service Account**                          | ✅                   |                            |                            |
| **Config Map for Broker Pods**               |                      | ✅                          |                            |
| **Config Map for Controller Pods**           |                      | ✅                          |                            |
| **Client CA Secret**                         | ✅                   |                            |                            |
| **Client CA Cert Secret**                    | ✅                   |                            |                            |
| **Cluster CA Secret**                        | ✅                   |                            |                            |
| **Cluster CA Cert Secret**                   | ✅                   |                            |                            |
| **Kafka Broker Certificates Secret**         | ✅                   |                            |                            |
| **Kafka Broker Service**                     | ✅                   |                            |                            |
| **Broker Pods**                              |                      |                            | ✅                          |
| **Controller Pods**                           |                      |                            | ✅                          |
| **Persistent Volume Claims (PVCs) for storage** |                      | ✅                          |                            |
| **Pod Disruption Budget**                    | ✅                   |                             |                            |

In a Stretch Kafka deployment, the operator on the central cluster must ensure that all resources derived from the Kafka CR and KafkaNodePool CR are also replicated in the remote clusters, including Persistent Volume Claims (PVCs) for storage.

In addition, the cluster operator normally creates network policies to restrict traffic from outside the namespace.
To enable pod-to-pod communication across Kubernetes clusters, the cluster operator will not create any network policies when configured to deploy a stretch cluster.

Furthermore, depending on the listener configuration, the Kafka CR may also create LoadBalancer, Ingress, NodePort, and Route services to expose Kafka externally.

Operators deployed to remote clusters are only responsible for reconciling `StrimziPodSet` resources that are created remotely by the operator running in the central cluster.

This approach will allow a Kafka administrator to manage the definition of their stretch cluster in a single location (control plane).

#### Prototype

A working prototype can be deployed using the steps outlined in a draft [README](https://aswinayyolath.github.io/stretch-kafka-docs/) that is being iteratively revised.

_Note: The prototype might not always exactly align with this proposal so please refer to the `README` documentation when working with the prototype._

### Additional considerations and reference information

#### Authentication and security considerations for multi-cluster access

The operator remains agnostic to the authentication method used; It simply consumes the kubeconfig provided by the user.
Users can choose from multiple authentication mechanisms based on their security requirements.
The responsibility for securing credentials, whether using tokens, certificates or other mechanisms, lies with the user.
Common approaches include token-based authentication using a long-lived `ServiceAccount` token, mTLS authentication with client TLS certificates, OIDC-based authentication leveraging an identity provider, and basic authentication with a username and password.
Each method provides different levels of security and flexibility, allowing users to integrate with their existing security infrastructure while ensuring secure access to remote Kubernetes clusters.

#### Example of multi-cluster advertised.listener and controller.quorum.voters

The operator configures `advertised.listeners` and `controller.quorum.voters` to support pod-to-pod communication in a stretch cluster.

**Default `advertised.listeners` format**

In a regular (single Kubernetes cluster) deployment, the `advertised.listeners` configuration uses this format:

```
advertised.listeners=REPLICATION-9091://<broker-pod-name>.<broker-service-name>.<namespace>.svc:9091,
PLAIN-9092://<broker-pod-name>.<broker-service-name>.<namespace>.svc:9092,
TLS-9093://<broker-pod-name>.<broker-service-name>.<namespace>.svc:9093
```

**Modified format for a stretch cluster**

When a Kafka cluster is deployed across multiple Kubernetes clusters, the operator modifies `advertised.listeners` to include a Kubernetes cluster identifier (`stretch-cluster-id`) that the user defines when [configuring Cilium](#step-1-user-configuration-of-cilium-cluster-mesh).
The modified format is:

```
advertised.listeners=REPLICATION-9091://<broker-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.cluster.local:9091,
PLAIN-9092://<broker-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.cluster.local:9092,
TLS-9093://<broker-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.cluster.local:9093
```

The `<stretch-cluster-id>` is dynamically retrieved from an [annotation](#kafkanodepool-cr) on the `KafkaNodePool` resource.

**Modified `controller.quorum.voters` format**

Similarly, `controller.quorum.voters` includes all Kafka controllers running in all of the Kubernetes clusters involved in the stretch cluster:

```
controller.quorum.voters=<controller-id>@<controller-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.cluster.local:9090, ...
```

These updates ensure brokers and controllers can be discovered and communicate across Kubernetes clusters without relying on traditional external access methods.

#### External access

When external listeners are configured, they will be set up across all participating clusters, along with their corresponding bootstrap addresses.
External access mechanisms, such as Routes, Ingress, or LoadBalancer services, will be deployed in each cluster, allowing clients to seamlessly connect to the stretch cluster.

#### Resource cleanup on remote Kubernetes clusters

Resources created in the remote clusters will not have `OwnerReferences` to avoid Kubernetes garbage collection from removing them as soon as they are created.
Finalizers will be added by default to the `Kafka` and `KafkaNodePool` resources in the central cluster to ensure the operator removes related remote cluster resources.
A user can disable the finalizers by setting the `STRIMZI_USE_FINALIZER` environment variable to `false`.

##### Rack awareness in stretch clusters
We conducted initial experiments to enable rack awareness in a Stretch Kafka cluster, ensuring that replicas are distributed across multiple Kubernetes clusters based on topology labels.
This approach improves fault tolerance by leveraging Kubernetes zone-aware scheduling or manually assigned zone labels for non-zone-aware clusters

For detailed implementation steps and test results, refer to our prototype [documentation](https://aswinayyolath.github.io/stretch-kafka-docs/Setting-up-Rack-Awareness-In-Stretch-Cluster/)

##### Disaster recovery; Handling central cluster failure

In a stretch Kafka deployment, the central Kubernetes cluster manages Kafka resources across multiple clusters, but Kafka brokers and controllers continue operating even if the central cluster fails.
We have performed some testing on the central cluster failure scenario, which is documented [here](https://aswinayyolath.github.io/stretch-kafka-docs/Testing-cluster-failover/).
The key challenge is restoring the administrative control plane while ensuring minimal downtime for Kafka clients.

###### Recovery approaches
There are several possible recovery methods:

###### 1. Manual recovery (baseline approach)
Users bring up a new Kubernetes cluster and manually apply the `Kafka` and `KafkaNodePool` CRs.
Alternatively, an existing remote cluster can be promoted to central by applying these CRs and updating the cluster operator deployment.
This approach is simple and reliable but requires user intervention.

###### 2. GitOps based recovery

Users store CRs in GitOps tools (e.g., ArgoCD, Flux).
When a new central cluster is created, GitOps tools automatically reconcile the resources.
This reduces manual effort but requires GitOps workflows to be set up in advance.

#### Entity operator
We recommend that all `KafkaTopic` and `KafkaUser` resources are managed from the central cluster: this is also where the entity operator should be deployed.
This will maintain the central cluster as the single control plane for resource management.
The expectation is that the entity operator will not be impacted any further by changes made to support a stretch cluster.

## Affected/not affected projects

This proposal only impacts strimzi-kafka-operator project.

## Rejected alternatives

The following disaster recovery approaches were considered but are out of scope for this proposal due to added complexity or implementation overhead:

### CR replication with manual failover

The central cluster operator replicates `Kafka` and `KafkaNodePool` resources across remote clusters as backup copies.
These CRs are annotated as inactive (e.g., `strimzi.io/standby=true`) so that remote clusters do not reconcile them.
In case of failure, users remove the standby annotation and update the operator deployment to promote the cluster to central.
While this accelerates recovery, it adds complexity in keeping replicated CRs up to date.

### Automated failover

The system detects central cluster failure and automatically elects a new primary cluster.
Replicated CRs become active, and the CO resumes reconciliation from the new cluster.
This approach eliminates manual intervention but requires leader election, coordination, and consistent state replication, which are non-trivial challenges.
