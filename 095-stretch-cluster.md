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

## Prerequisites

- **KRaft**: As Kafka and Strimzi transition towards KRaft-based clusters, this proposal focuses exclusively on enabling stretch deployments for KRaft-based Kafka clusters.
While Zookeeper-based deployments are still supported, they are outside the scope of this proposal.

- **Multiple Kubernetes clusters**: Stretch Kafka clusters will require multiple Kubernetes clusters.
To ensure Kafka controller quorum in the event of a Kubernetes cluster outage, it is recommended to distribute controllers across at least three Kubernetes clusters. In a two cluster setup, there is a risk of losing quorum if the wrong cluster fails, which can lead to Kafka becoming unavailable.

- **Low Latency and High Bandwidth**:  Kafka clusters should be deployed in environments that provide low-latency and high-bandwidth communication between Kafka brokers and controllers.
Stretch Kafka clusters should be deployed in environments such as data centers or availability zones within a single region.
Deployments across distant regions, where high latency and limited bandwidth could impair performance, should be avoided.


- **A supported cloud native networking technology**: To enable networking between Kubernetes clusters currently requires an additional technology stack.
The prototype detailed within this proposal requires advance manual setup of an overlay network offered by a [Cloud Native Network](https://landscape.cncf.io/guide#runtime--cloud-native-network) project to provide connectivity between Kafka runtimes in different Kubernetes clusters.
There are currently two projects that can successfully support the prototype operator:
    1. [Submariner](https://submariner.io/) provides an implementation of [Kubernetes Multi-Cluster Services (MCS)](https://multicluster.sigs.k8s.io/guides/#implementation-status).
    The Kafka runtime pods use these new services to communicate between one another.
    Other MCS implementations are expected to be viable, but have not yet been prototyped.
    2. [Cilium](https://cilium.io/) provides support for a [Cluster Mesh](https://docs.cilium.io/en/stable/network/clustermesh/intro/) (an MCS API implementation is currently in beta).
    The MCS API is not used by the prototype, but manual re-configuration of Kubernetes CoreDNS is required.

    The draft [README](https://aswinayyolath.github.io/stretch-kafka-docs) for the prototype includes detailed steps describing how to pre-configure both [Submariner](https://aswinayyolath.github.io/stretch-kafka-docs/setting-up-submariner/) and [Cilium](https://aswinayyolath.github.io/stretch-kafka-docs/Setting-up-cilium/).

## Design

### Topology of a stretch cluster

![Stretch cluster topology](./images/095-stretch-cluster-topology.png)

The diagram illustrates a topology comprising of three Kubernetes clusters.

One of these clusters is designated as the "Central cluster", while any additional clusters are considered "remote".
The central cluster acts as the control plane where a user will create all the custom resources for the Kafka cluster - Kafka, KafkaNodePool, KafkaUser, KafkaTopic etc.

A Kafka node pool definition can be configured to specify a Kubernetes cluster (central cluster or one of the member clusters) as the deployment target.
The operator on the central cluster is responsible for creating all necessary resources (including `StrimziPodSet` resources) for the node pool on the target Kubernetes cluster. 

Operators deployed to remote clusters are only responsible for reconciling `StrimziPodSet` resources that are created remotely by the operator running in the central cluster.

This approach will allow users to manage the definition of their stretch Kafka cluster in a single location.

### Prototype

A working prototype can be deployed using the steps outlined in a draft [README](https://aswinayyolath.github.io/stretch-kafka-docs/) that is being iteratively revised.
The following sections will describe the key aspects of the prototype highlighting areas that would benefit from community feedback and input.

#### User configuration of a supported Cloud Native Network project

Multi-Cluster Services or a Cluster Mesh overlay network must be configured manually as a pre-requisite using a supported project.
This configuration must be performed by the user prior to deployment of Strimzi cluster operators.

Each Kubernetes cluster joined to the network is assigned a unique identifier.
This identifier is used by the prototype central operator to build:
1. Valid broker and controller service endpoints values for `advertised.listeners` and `controller.quorum.voters` within the appropriate ConfigMap resources.
2. Broker certificates with appropriate Subject Alternative Names.
The prototype currently expects this identifier to be supplied by the user as a label value on the Kafka node pool resource:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  ...
  labels:
    ...
    strimzi.io/submariner-cluster-id: "cluster1"
```

_Note: The prototype operator [code](https://github.com/aswinayyolath/strimzi-kafka-operator/compare/main...aswinayyolath:strimzi-kafka-operator:strecth-cluster-prototype?expand=1) currently expects `submariner-cluster-id`, but this is subject to change for something more generic once additional technologies have been prototyped. One suggestion is `strimzi.io/cluster-network-id`._

_Note: This network identifier could have the same value as the Kubernetes cluster identifier described in the following section and both aspects would benefit from community feedback and input._

#### User configuration of the deployment topology

The operator running in the central Kubernetes cluster must be provided with the following information to allow creation of resources in remote clusters:

- An identifier for each remote Kubernetes cluster.
- A URL endpoint for the Kubernetes API server running in each remote cluster.
- Credential(s) to allow authentication with the remote Kubernetes API servers.

For simplicity and to quickly achieve a working prototype, the information outlined above is currently provided as an environment variable to the central operator.
The value of the environment variable uses a "map" format as shown below:

```yaml
- name: STRIMZI_K8S_CLUSTERS
  value: |
      cluster-id-a.url=<cluster-a URL>
      cluster-id-a.secret=<secret-name-cluster-a>
      cluster-id-b.url=<cluster-b URL>
      cluster-id-b.secret=<secret-name-cluster-b>
```

The secrets referenced here must currently contain the kubeconfig for the Kubenetes cluster available at the provided URL as the value of secret key 'kubeconfig'.
This allows the central Strimzi operator to authenticate with multiple Kubernetes clusters.

Additionally, a Kafka node pool resource definition must indicate which cluster will host the Kafka brokers/controllers for that node pool.
The prototype uses a `spec.cluster` field within a KafkaNodePool definition that will have a value that matches one of the identifiers from the above map.
If the `spec.cluster` field is missing from the KafkaNodePool definition, it is currently assumed that the broker/controller pods will run on the central Kubernetes cluster.

_Note: This cluster identifier could have the same value as the network identifier described in the previous section. Community feedback/input very welcome!_

#### Authentication & Security Considerations for Multi-Cluster Access

The central Strimzi operator needs access to multiple Kubernetes clusters to manage Kafka nodes across them. 
This is typically achieved by storing the necessary credentials in Kubernetes Secrets, with each secret containing a kubeconfig file that provides authentication details for a remote cluster.

To securely manage access, users should ensure that their kubeconfig files follow best practices for minimal privilege access. 
The operator itself does not impose specific authentication mechanisms but relies on the credentials provided by the user

##### Recommended Approach for Token-Based Authentication
For users opting for token-based authentication, we recommend

###### 1. Using a Dedicated ServiceAccount with Scoped Permissions

- Instead of embedding a full administrator kubeconfig, create a `ServiceAccount` in each remote cluster.
- Assign only the necessary permissions using a `Role` and `RoleBinding`.

Example YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: stretch-operator
  namespace: <namespace>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: stretch-operator-role
  namespace: <namespace>
rules:
- apiGroups:
  - core.strimzi.io
  resources:
  - strimzipodsets
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
- apiGroups:
  - ""
  resources:
  - services
  - configmaps
  - secrets
  - serviceaccounts
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
- apiGroups:
  - multicluster.x-k8s.io
  resources:
  - serviceexports
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: stretch-operator-rolebinding
  namespace: <namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: stretch-operator-role
subjects:
- kind: ServiceAccount
  name: stretch-operator
  namespace: <namespace>

```

###### 2. Generating a Long-Lived Token for the ServiceAccount

- Since ServiceAccount tokens in Kubernetes 1.24+ are short-lived, a manually created ServiceAccount token secret ensures persistent authentication.


Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-long-lived-secret
  annotations:
    kubernetes.io/service-account.name: stretch-operator
type: kubernetes.io/service-account-token
```

###### 3. Storing Minimal Credentials in Kubeconfig
- Instead of using a full admin kubeconfig, generate a scoped kubeconfig containing only the necessary token.

Example Kubeconfig

```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://api.example.com:6443
  name: example-cluster
contexts:
- context:
    cluster: example-cluster
    namespace: <namespace>
    user: stretch-operator-user
  name: stretch-cluster-context
current-context: stretch-cluster-context
kind: Config
preferences: {}
users:
- name: stretch-operator-user
  user:
    token: <TOKEN>
```

##### Flexibility for Other Authentication Mechanisms
While token-based authentication is one option, users may choose alternative methods based on their security policies. 
Some users might

- Use mTLS authentication, which requires client TLS certificates inside the kubeconfig instead of tokens.
- Use username/password authentication with external authentication providers.
- Integrate OIDC-based authentication for federated identity management.

The Strimzi operator itself remains agnostic to the authentication method used. 
It simply consumes the kubeconfig provided by the user. 
The responsibility of securing credentials—whether using tokens, certificates, or other mechanisms—lies with the user.

##### Expected Benefits

-  Users can apply least-privilege principles by configuring kubeconfig files with scoped permissions.
- Long-lived credentials prevent frequent authentication failures.
- The operator supports any authentication mechanism that Kubernetes itself supports, allowing users to integrate it into their existing security model.

#### Other prototype configuration items and limitations

1. The central cluster operator needs to know which of the Cloud Native Network projects has been pre-configured. This is currently set on the Kafka CR using an annotation:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  ...
  annotations:
    ...
    strimzi.io/cross-cluster-type: "submariner"
```

2. All Strimzi cluster operators involved must be aware they are involved in reconciling a stretch Kafka cluster by means of a new environment variable:

```yaml
- name: STRIMZI_STRETCH_MODE
  value: 'true'
```

Within the prototype operator, this is used to conditionally break the expectation that Kafka, KafkaNodePool are located in the same Kubernetes cluster as the StrimziPodSet and Pod resources.

3. The existing Strimzi network policies block cross-cluster network traffic so the prototype requires the following environment variable to be set on the central operator deployment:

```yaml
- name: STRIMZI_NETWORK_POLICY_GENERATION
  value: 'false'
```

4. Submariner [requires an adjustment](https://aswinayyolath.github.io/stretch-kafka-docs/Deploying-Strimzi-in-Stretch-Mode/#updating-clusterrole-for-submariner) to the `ClusterRole` for the Strimzi cluster operator in the central Kubernetes cluster.

5. The prototype operator currently creates new Kubernetes client instances at the required stage of reconciliation for the range of Kubernetes resources involved.
This approach was quick to implement, but results in duplicate [code](https://github.com/aswinayyolath/strimzi-kafka-operator/compare/main...aswinayyolath:strimzi-kafka-operator:strecth-cluster-prototype?expand=1#diff-ac9b17492d6e343d1a402e70ab4460b2f56e6d3a66600f707cb354523384c651R263).
A design for better management of Kubernetes clients for remote clusters is required and is dependent upon community input.

**All aspects of user configuration described above would benefit from further discussion within the community.**

## Additional considerations and reference information

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

When Kafka is deployed using "stretch mode", the operator modifies `advertised.listeners` to include a network identifier, such as the Submariner or Cilium `cluster-id`. The modified format is:

```
advertised.listeners=REPLICATION-9091://<broker-pod-name>.<cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9091,
PLAIN-9092://<broker-pod-name>.<cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9092,
TLS-9093://<broker-pod-name>.<cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9093
```

The `<cluster-id>` is dynamically retrieved from a label on the `KafkaNodePool` resource.

**Modified `controller.quorum.voters` format**

Similarly, `controller.quorum.voters` includes all Kafka controllers running in all of the Kubernetes clusters involved in the stretch Kafka cluster:

```
controller.quorum.voters=<controller-id>@<controller-pod-name>.<cluster-id>.<broker-service-name>.<namespace>.svc.clusterset.local:9090, ...
```

These updates ensure brokers and controllers can be discovered and communicate across clusters without relying on traditional external access methods.

#### Resource cleanup on remote Kubernetes clusters

In our POC, resources created in the remote clusters do not have `OwnerReferences`. The main reason is that the `Kafka` and `KafkaNodePool` CRs exist only in the central cluster. If resources in remote clusters were to reference them as owners, they would be immediately deleted by Kubernetes' garbage collection mechanism since their owners do not exist in the remote clusters.

Even if ownership across Kubernetes cluster boundaries were possible, it would introduce another issue: if the central cluster were to go down, any resources in remote clusters with OwnerReferences pointing to central cluster resources would also be deleted automatically. This is not the intended behaviour, as we want remote resources to remain operational even if the central cluster becomes temporarily unavailable.

The prototype does not yet have a mechanism for controlled cleanup of remote resources when a user deliberately deletes the Kafka and KafkaNodePool CRs from the central cluster. Currently, when a user deletes these CRs, all related resources in the central cluster are removed, but remote cluster resources remain. Finding a way to handle this cleanup while avoiding unintended deletions is an open challenge.

### Rack Awareness in Stretch Clusters
We conducted initial experiments to enable rack awareness in a Stretch Kafka cluster, ensuring that replicas are distributed across multiple Kubernetes clusters based on topology labels. This approach improves fault tolerance by leveraging Kubernetes zone-aware scheduling or manually assigned zone labels for non-zone-aware clusters

For detailed implementation steps and test results, refer to our prototype [documentation](https://aswinayyolath.github.io/stretch-kafka-docs/Setting-up-Rack-Awareness-In-Stretch-Cluster/)

### Disaster Recovery - Handling Central Cluster Failure

In a stretch Kafka deployment, the central Kubernetes cluster manages Kafka resources across multiple clusters, but Kafka brokers and controllers continue operating even if the central cluster fails.
We have performed some testing on the central cluster failure scenario, which is documented [here](https://aswinayyolath.github.io/stretch-kafka-docs/Testing-cluster-failover/).
The key challenge is restoring the administrative control plane while ensuring minimal downtime for Kafka clients.

#### Recovery Approaches
There are several possible recovery methods

##### 1. Manual Recovery (Baseline Approach)
Users bring up a new Kubernetes cluster and manually apply the Kafka, KafkaNodePool CRs.
Alternatively, an existing remote cluster can be promoted to central by applying these CRs and updating the Cluster Operator deployment.
This approach is simple and reliable but requires user intervention.

##### 2. GitOps-Based Recovery

Users store CRs in GitOps tools (e.g., ArgoCD, Flux).
When a new central cluster is created, GitOps tools automatically reconcile the resources.
This reduces manual effort but requires GitOps workflows to be set up in advance.

##### 3. CR Replication with Manual Failover

The central Cluster Operator replicates Kafka, KafkaNodePool across remote clusters as backup copies.
These CRs are annotated as inactive (e.g., strimzi.io/standby=true) so that remote clusters do not reconcile them.
In case of failure, users remove the standby annotation and update the CO deployment to promote the cluster to central.
This accelerates recovery but adds complexity in keeping replicated CRs up to date.

##### 4. Automated Failover (Future Consideration)
The system detects central cluster failure and automatically elects a new primary cluster.
Replicated CRs become active, and the CO resumes reconciliation from the new cluster.
This offers zero manual intervention but requires leader election, coordination, and consistent state replication, which are non-trivial.

#### Recommended Approach
As a starting point, we recommend manual recovery (Option 1) and GitOps-based recovery (Option 2) due to their simplicity and lower implementation overhead.
We do not plan to implement automated failover (Option 4) at this stage. However, CR replication (Option 3) remains an option if users demand a faster failover mechanism.

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
