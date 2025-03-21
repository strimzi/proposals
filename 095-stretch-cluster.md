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

## Limitations and Considerations
While a stretch Kafka cluster offers several advantages, it also introduces some challenges and considerations:

- **Increased Network Complexity and Costs**: The communication between brokers and controllers across clusters relies on network connectivity, which can be less reliable and more costly than intra-cluster communication.
This necessitates careful consideration of network architecture and associated costs.

- **Latency Requirements**: Stretch Kafka clusters are best suited for environments with low-latency and high-bandwidth network connections between the Kubernetes clusters.
High latency can adversely affect the performance and synchronization of Kafka nodes, potentially leading to delays or errors in replication and client communication.
Defining the minimal acceptable latency between clusters is crucial to ensure optimal performance.

## Proposal

This proposal seeks to enhance the Strimzi Kafka operator to support stretch Kafka clusters, distributing broker, controller and combined Kafka nodes across multiple Kubernetes clusters.
The intent is to focus on high-availability of the Kafka control plane and data plane.
The proposal outlines high-level topology and design concepts for such deployments, with a plan to incrementally include finer design and implementation details for various aspects.

### Prerequisites

- **KRaft**: As Kafka and Strimzi transition towards KRaft-based clusters, this proposal focuses exclusively on enabling stretch deployments for KRaft-based Kafka clusters.
Strimzi will not support stretch clusters for Zookeeper-based deployments.

- **Multiple Kubernetes clusters**: Stretch Kafka clusters will require multiple Kubernetes clusters.
The recommended minimum number of Kubernetes clusters is 3 because this is the minimum number of controllers required to establish quorum and provide High Availability (HA).
However, the cluster operator will not block deployment of a stretch cluster with fewer than 3 Kubernetes clusters as this will be useful for test and development use cases.
The cluster operator will log a warning similar to the existing warning logged when deploying fewer than 3 ZooKeeper nodes.

- **Low Latency and High Bandwidth**:  Kafka clusters should be deployed in environments that provide low-latency and high-bandwidth communication between Kafka brokers and controllers.
Stretch Kafka clusters should be deployed in environments such as data centers or availability zones within a single region.
Deployments across distant regions, where high latency and limited bandwidth could impair performance, should be avoided.


- **A supported cloud native networking technology**: To enable networking between Kubernetes clusters currently requires an additional technology stack.
Manual configuration of a [Cloud Native Network](https://landscape.cncf.io/guide#runtime--cloud-native-network) project is required.
Any project that can support direct pod-to-pod communication across multiple Kubernetes clusters should be compatible with the proposed solution.
The current prototype detailed within this proposal has been tested with the following projects:
    1. [Submariner](https://submariner.io/) provides an implementation of [Kubernetes Multi-Cluster Services (MCS)](https://multicluster.sigs.k8s.io/guides/#implementation-status).
    The Kafka runtime pods use these new services to communicate between one another.
    Other MCS implementations are expected to be viable, but have not yet been prototyped.
    2. [Cilium](https://cilium.io/) provides support for a [Cluster Mesh](https://docs.cilium.io/en/stable/network/clustermesh/intro/) (an MCS API implementation is currently in beta).
    The MCS API is not used by the prototype, but manual re-configuration of Kubernetes CoreDNS is required.

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

#### Step 1: User configuration of a supported Cloud Native Network project

Multi-Cluster Services or a Cluster Mesh overlay network must be configured manually as a pre-requisite using a supported project.
This configuration of the network must be performed by the user prior to deployment of Strimzi cluster operators.
One aspect of the Cloud native network configuration is to define a unique identifier that is used to identify and distinguish different Kubernetes clusters when they are connected for networking.

The Strimzi cluster operator will use this identifier in two ways:
1. Generating `advertised.listeners` and `controller.quorum.voters` configuration.
2. Determining the target Kubernetes cluster for a Kafka node pool.

#### Step 2: Deploy a cluster operator to each Kubernetes cluster

##### Central cluster operator configuration

The operator running in the central Kubernetes cluster must be provided with the following information to allow creation of resources in remote clusters:

- The identifier for each Kubernetes cluster defined in 'Step 1'.
- A URL endpoint for the Kubernetes API server running in each remote cluster.
- Credential(s) to allow authentication with the remote Kubernetes API servers.

The information outlined above will be provided as an environment variable for the cluster operator.
The value of the environment variable uses a "map" format as shown below:

```yaml
- name: STRIMZI_K8S_CLUSTERS
  value: |
      cluster-id-a.url=<cluster-a URL>
      cluster-id-a.secret=<secret-name-cluster-a>
      cluster-id-b.url=<cluster-b URL>
      cluster-id-b.secret=<secret-name-cluster-b>
```

The secrets referenced here must contain the kubeconfig for the Kubenetes cluster available at the provided URL as the value of secret key 'kubeconfig'.
This allows the central Strimzi operator to authenticate with multiple Kubernetes clusters.

When this environment variable is set, the cluster operator will understand it needs to deploy a stretch Kafka cluster.
The custom actions taken as a result of this environment variable include:
1. The value of `STRIMZI_NETWORK_POLICY_GENERATION` is assumed to be `false` irrespective of the value set by the user.
This is to ensure network traffic between Kubernetes clusters is not blocked.
2. The `StrimziPodSet` resources created in a remote cluster will include an annotation `strimzi.io/orphaned-podset: true`.
This annotation will allow the remote cluster operator to reconcile the `StrimziPodSet` without a `Kafka` and `KafkaNodePool` CR being present in the same cluster.

##### Remote cluster operator configuration

The operator running in the remote cluster must be set to only reconcile `StrimziPodSet` resources by setting the existing environment variable:

```yaml
- name: STRIMZI_POD_SET_RECONCILIATION_ONLY
  value: true
```

This configuration will minimise the resources required by the cluster operator on the remote Kubernetes clusters and simplify logging output.

#### Step 3: Create Kafka and KafkaNodePool resources in the central cluster

##### Kafka CR

The `Kafka` CR must include a new annotation to specify the chosen Cloud native network project:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  annotations:
    strimzi.io/stretch-cluster-network: "submariner"
```

##### KafkaNodePool CR

The `KafkaNodePool` CR must include a new annotation to specify the target Kubernetes cluster where the node pool resources should be deployed:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  annotations:
    strimzi.io/stretch-cluster-id: "cluster3"
```

The ID used here is what the user specified in 'Step 1'.

#### Remote cluster resources created by the central operator in a Stretch Kafka deployment

In a Stretch Kafka setup, the Kafka and KafkaNodePool CRs are defined only in the central cluster.
However, these CRs are responsible for creating several Kubernetes resources that are required for Kafka broker and controller pods to be in a running state.
Since remote clusters do not have direct access to these CRs, the Strimzi Operator running in the central cluster must ensure that the necessary resources are created in the remote clusters as well.

The following table outlines the Kubernetes resources created by each Strimzi Custom Resource (Kafka, KafkaNodePool, and StrimziPodSet).
In a Stretch Kafka deployment, the operator on the central cluster must ensure that all resources created by the Kafka CR and KafkaNodePool CR are also replicated in the remote clusters, including Persistent Volume Claims (PVCs) for storage.

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

In addition, Strimzi also creates network policies to restrict traffic from outside the namespace.
To enable pod-to-pod communication across clusters, the cluster operator will not create any network policies.

Furthermore, depending on the listener configuration, the Kafka CR may also create LoadBalancer, Ingress, NodePort, and Route services to expose Kafka externally.
By ensuring that all necessary resources including ConfigMaps, Secrets, Services, and PVCs—are created in the appropriate clusters, the central operator enables  Kafka deployment across multiple Kubernetes clusters in a Stretch Kafka setup.

Operators deployed to remote clusters are only responsible for reconciling `StrimziPodSet` resources that are created remotely by the operator running in the central cluster.

This approach will allow users to manage the definition of their stretch Kafka cluster in a single location.

#### Prototype

A working prototype can be deployed using the steps outlined in a draft [README](https://aswinayyolath.github.io/stretch-kafka-docs/) that is being iteratively revised.

_Note: The prototype might not exactly align with this proposal so please refer to the `README` documentation when working with the prototype._

### Additional considerations and reference information

#### Authentication & Security Considerations for Multi-Cluster Access

The Strimzi operator itself remains agnostic to the authentication method used; It simply consumes the kubeconfig provided by the user. 
The responsibility for securing credentials, whether using tokens, certificates, or other mechanisms, lies with the user.
We have provided additional details on securing remote cluster credentials in a multi-cluster Kafka deployment [here](https://aswinayyolath.github.io/stretch-kafka-docs/Secure-remote-cluster-credentials/). 
This document outlines best practices for securing remote cluster credentials when using token-based kubeconfig authentication.

#### Example of multi-cluster advertised.listener and controller.quorum.voters

The Operator configures `advertised.listeners` and `controller.quorum.voters` to support cross-cluster communication in a stretch Kafka deployment.

**Default `advertised.listeners` format**

In a regular (single Kubernetes cluster) deployment, the `advertised.listeners` configuration uses this format:

```
advertised.listeners=REPLICATION-9091://<broker-pod-name>.<broker-service-name>.<namespace>.svc:9091,
PLAIN-9092://<broker-pod-name>.<broker-service-name>.<namespace>.svc:9092,
TLS-9093://<broker-pod-name>.<broker-service-name>.<namespace>.svc:9093
```

**Modified format for a stretch Kafka cluster**

When Kafka is deployed using "stretch mode", the operator modifies `advertised.listeners` to include a network identifier, such as the Submariner or Cilium `stretch-cluster-id`. The modified format is:

```
advertised.listeners=REPLICATION-9091://<broker-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9091,
PLAIN-9092://<broker-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9092,
TLS-9093://<broker-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9093
```

The `<stretch-cluster-id>` is dynamically retrieved from an annotation on the `KafkaNodePool` resource.

**Modified `controller.quorum.voters` format**

Similarly, `controller.quorum.voters` includes all Kafka controllers running in all of the Kubernetes clusters involved in the stretch Kafka cluster:

```
controller.quorum.voters=<controller-id>@<controller-pod-name>.<stretch-cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9090, ...
```

These updates ensure brokers and controllers can be discovered and communicate across clusters without relying on traditional external access methods.

#### Resource cleanup on remote Kubernetes clusters

In our POC, resources created in the remote clusters do not have `OwnerReferences`. The main reason is that the `Kafka` and `KafkaNodePool` CRs exist only in the central cluster. If resources in remote clusters were to reference them as owners, they would be immediately deleted by Kubernetes' garbage collection mechanism since their owners do not exist in the remote clusters.

Even if ownership across Kubernetes cluster boundaries were possible, it would introduce another issue: if the central cluster were to go down, any resources in remote clusters with OwnerReferences pointing to central cluster resources would also be deleted automatically. This is not the intended behaviour, as we want remote resources to remain operational even if the central cluster becomes temporarily unavailable.

The prototype does not yet have a mechanism for controlled cleanup of remote resources when a user deliberately deletes the Kafka and KafkaNodePool CRs from the central cluster. Currently, when a user deletes these CRs, all related resources in the central cluster are removed, but remote cluster resources remain. Finding a way to handle this cleanup while avoiding unintended deletions is an open challenge.

#### Rack Awareness in Stretch Clusters
We conducted initial experiments to enable rack awareness in a Stretch Kafka cluster, ensuring that replicas are distributed across multiple Kubernetes clusters based on topology labels. This approach improves fault tolerance by leveraging Kubernetes zone-aware scheduling or manually assigned zone labels for non-zone-aware clusters

For detailed implementation steps and test results, refer to our prototype [documentation](https://aswinayyolath.github.io/stretch-kafka-docs/Setting-up-Rack-Awareness-In-Stretch-Cluster/)

#### Disaster Recovery - Handling Central Cluster Failure

In a stretch Kafka deployment, the central Kubernetes cluster manages Kafka resources across multiple clusters, but Kafka brokers and controllers continue operating even if the central cluster fails.
We have performed some testing on the central cluster failure scenario, which is documented [here](https://aswinayyolath.github.io/stretch-kafka-docs/Testing-cluster-failover/).
The key challenge is restoring the administrative control plane while ensuring minimal downtime for Kafka clients.

##### Recovery Approaches
There are several possible recovery methods

###### 1. Manual Recovery (Baseline Approach)
Users bring up a new Kubernetes cluster and manually apply the Kafka, KafkaNodePool CRs.
Alternatively, an existing remote cluster can be promoted to central by applying these CRs and updating the Cluster Operator deployment.
This approach is simple and reliable but requires user intervention.

###### 2. GitOps-Based Recovery

Users store CRs in GitOps tools (e.g., ArgoCD, Flux).
When a new central cluster is created, GitOps tools automatically reconcile the resources.
This reduces manual effort but requires GitOps workflows to be set up in advance.

#### Entity operator
We would recommend that all KafkaTopic and KafkaUser resources are managed from the cluster that holds Kafka and KafkaNodePool resources, and that should be the cluster where the entity operator should be enabled.
This will maintain all resource management from a central point.
The expectation is that the entity operator will not be impacted by changes made in the prototype, but this currently requires verification.

Other aspects requiring consideration and research:
- rolling updates
- scaling brokers/controllers
- certificate rotation
- cruise control
- observability

## Affected/not affected projects

This proposal only impacts strimzi-kafka-operator project.

## Rejected Alternatives

The following approaches were considered but are out of scope for this proposal due to added complexity or implementation overhead

### CR Replication with Manual Failover

The central Cluster Operator replicates Kafka, KafkaNodePool across remote clusters as backup copies.
These CRs are annotated as inactive (e.g., strimzi.io/standby=true) so that remote clusters do not reconcile them.
In case of failure, users remove the standby annotation and update the CO deployment to promote the cluster to central.
While this accelerates recovery, it adds complexity in keeping replicated CRs up to date.

### Automated Failover

The system detects central cluster failure and automatically elects a new primary cluster.
Replicated CRs become active, and the CO resumes reconciliation from the new cluster.
This approach eliminates manual intervention but requires leader election, coordination, and consistent state replication, which are non-trivial challenges.
