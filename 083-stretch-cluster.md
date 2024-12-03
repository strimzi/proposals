# Stretch Kafka cluster

The Strimzi Kafka operator currently manages Kafka clusters within a single Kubernetes cluster.
This proposal aims to extend this support to stretch Kafka clusters, where brokers and controllers of a single Kafka cluster are distributed across multiple Kubernetes clusters.

## Current situation

At present, the availability of Strimzi-managed Kafka clusters is limited by the availability of the underlying Kubernetes cluster.
If a Kubernetes cluster experiences an outage, the entire Kafka cluster becomes unavailable, disrupting all connected Kafka clients.

## Motivation

A stretch Kafka cluster allows Kafka nodes to be distributed across multiple Kubernetes clusters. This approach also facilitates maby valuable use cases, such as:

- **High Availability**: Distributing Kafka brokers across multiple Kubernetes cluster significantly enhances resilience by enabling the system to tolerate the outage of a Kubernetes cluster without disrupting service to clients.

- **Migration Flexibility**: The ability to move Kafka nodes between Kubernetes environments without downtime, supporting maintenance or migrations.

- **Resource Optimization**: Efficiently utilizing resources across multiple clusters, which can be advantageous in environments with varying cluster capacities or during scaling operations.

### Limitations and Considerations
While a stretch Kafka cluster offers several advantages, it also introduces some challenges and considerations:

- **Increased Network Complexity and Costs**: The communication between brokers and controllers across clusters relies on network connectivity, which can be less reliable and more costly than intra-cluster communication.
This necessitates careful consideration of network architecture and associated costs.

- **Latency Requirements**: Stretch Kafka clusters are best suited for environments with low-latency network connections between the Kubernetes clusters.
High latency can adversely affect the performance and synchronization of Kafka nodes, potentially leading to delays or errors in replication and client communication.
Defining the minimal acceptable latency between clusters is crucial to ensure optimal performance.

## Proposal

This proposal seeks to enhance the Strimzi Kafka operator to support stretch Kafka clusters, distributing brokers and controllers across multiple Kubernetes clusters.
The intent is to focus on high-availability of the data plane.
The proposal outlines high-level topology and design concepts for such deployments, with a plan to incrementally include finer design and implementation details for various aspects.

### Prerequisites

- **Multiple Kubernetes Clusters**: Stretch Kafka clusters will require multiple Kubernetes clusters.
Ideally, an odd number of clusters (at least three) will be needed to maintain Kafka controller quorum and tolerate a Kubernetes cluster outage.

- **Low Latency**: Kafka clusters should be deployed in environments that allow low-latency communication between Kafka brokers and controllers.
Stretch Kafka clusters should be deployed in environments such as data centers or availability zones within a single region, and not across distant regions where high latency could impair performance.

- **KRaft**: As Kafka and Strimzi transition towards KRaft-based clusters, this proposal focuses exclusively on enabling stretch deployments for KRaft-based Kafka clusters.
While Zookeeper-based deployments are still supported, they are outside the scope of this proposal.

### Design

The cluster operator will be deployed in all Kubernetes clusters and will manage Kafka brokers and controllers running on that cluster.
One Kubernetes cluster will act as the control point for defining custom resources (Kafka, KafkaNodePool) required for stretch Kafka cluster.
The KafkaNodePool custom resource will be extended to include information about a Kubernetes cluster where the pool should be deployed.
The cluster operator will create necessary resources (StrimziPodSets, services etc.) on the target clusters specified within the KafkaNodePool resource.

This approach will allow users to specify/manage the definition of stretch Kafka cluster in a single location.
The operators will then create necessary resources in target Kubernetes clusters, which can then be reconciled/managed by operators on those clusters.

### Reconciling Kafka and KafkaNodePool resources
![Reconciling Kafka and KafkaNodePool resources](./images/083-reconciling-kafka-knp.png)

### Reconciling StrimziPodSet resources
![Reconciling SPS](./images/083-reconciling-sps.png)

#### KafkaNodePool changes
A new optional field (`target`) will be introduced in the KafkaNodePool resource specification, to allow users to specify the details of the Kubernetes cluster where the node pool should be deployed.
This section will include the target cluster's URL (Kubernetes cluster where resources for this node pool will be created) and the secret containing the kubeconfig data for that cluster.

An example of the KafkaNodePool resource with the new fields might look like:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controller
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  target:
    clusterUrl: <K8S Cluster URL>
    secret: <SecretName>
  listenerConfig:
    - configuration:
          bootstrap:
            alternativeNames:
              - bootstrap-ext.<kubernetes infra host>
              - broker-100.<kubernetes infra host>
              - broker-101.<kubernetes infra host>
            host: bootstrap-ext.<kubernetes infra host>
          brokers:
          - broker: 100
            host: broker-100.<kubernetes infra host>
          - broker: 101
            host: broker-101.<kubernetes infra host>
          class: nginx
        name: connectext
  roles:
    - controller
  storage:
   .........
```

#### Kafka changes

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
    strimzi.io/stretch-mode: enabled
spec:
  kafka:
    version: 3.7.0
    metadataVersion: 3.7-IV4
    listeners:
      - port: 9093
        tls: true
        name: connectext
        type: ingress
```

A new annotation (`stretch-mode: enabled`) will be introduced in Kafka custom resource to indicate when it is representing a stretch Kafka cluster
This approach is similar to how Strimzi currently enables features like KafkaNodePool (KNP) and KRaft mode.

In a stretch Kafka cluster, we'll need bootstrap and broker services to be present on each Kubernetes cluster and be accessible from other clusters.
The Kafka reconciler will identify all target clusters from KafkaNodePool resources and create these services in target Kubernetes clusters.
This will ensure that even if the central cluster experiences an outage, external clients can still connect to the stretch cluster and continue their operations without interruption.

#### Cross-cluster communication

#### Cross-Cluster Communication Using Submariner

For Kafka brokers and controllers distributed across multiple Kubernetes clusters, cross-cluster communication is essential.
Submariner is a tool that facilitates this type of communication by connecting different Kubernetes clusters through secure networking, allowing data transfer without relying solely on traditional methods such as LoadBalancers or Ingresses.

#### Current Limitations

The Strimzi Kafka operator currently sets up Kafka listeners for internal communication between brokers and controllers within a single Kubernetes cluster.
These services are typically defined as headless services, accessible only within the cluster’s network (e.g., `my-cluster-broker-0.my-cluster-kafka-brokers.svc.cluster.local`).
However, these internal addresses are not accessible outside the cluster, 
while Kubernetes provides native solutions like Ingress to expose services externally, these solutions can be slower
and may introduce latency due to the additional routing overhead. Submariner offers a more efficient alternative by 
enabling direct communication between clusters through secure IP routing.


#### How Submariner Facilitates Cross-cluster Communication

When multiple Kubernetes clusters are connected using Submariner,
services of type ClusterIP can be exported, making them accessible across participating clusters in the network.

To export a service using Submariner, the following command can be used

```
subctl export service --kubeconfig <CONFIG> --namespace <NAMESPACE> my-cluster-kafka-brokers
```

This command creates a ServiceExport resource in the specified namespace,
signaling Submariner to register the service with the Submariner Broker. 
The Broker acts as the coordinator for cross-cluster service discovery, 
utilizing the Lighthouse component to allow services in different clusters 
to discover and communicate with each other. 
Submariner sets up tunnels and routing tables, ensuring direct and secure traffic flow between clusters.

Once a service is exported, it becomes accessible through a global DNS name format: `<service-name>.<namespace>.svc.clusterset.local`. 
This DNS name enables reachability for any cluster in the Submariner deployment. 
For instance, the `advertised.listener` configuration in a Kafka setup would be updated from `my-cluster-broker-0.my-cluster-kafka-brokers.svc.cluster.local` 
to `my-cluster-broker-0.cluster1.my-cluster-kafka-brokers.svc.clusterset.local`, where `cluster1` represents the Submariner cluster ID.
Similarly the `controller.quorum.voters` property also need to be updated to use Submariner exported servicename. 
This ensures that `advertised.listener` and `controller.quorum.voters` addresses are reachable from any connected cluster.


#### Considerations for SSL/TLS Verification

For SSL hostname verification between pods, the Subject Alternative Name (SAN) entries in the certificates must include the FQDNs of the Submariner-exported services. 
This can be achieved in the following ways:

1. **Defining SANs in the Kafka CR**: Users can specify the Submariner-exported FQDNs in the alternativeNames field within the Kafka CR's listener configuration. 
For example:

```yaml
listeners:
  - name: tls
    port: 9093
    type: internal
    tls: true
    configuration:
      bootstrap:
        alternativeNames:
          - my-cluster-broker-0.cluster2.my-cluster-kafka-brokers.strimzi.svc.clusterset.local
          - my-cluster-broker-1.cluster2.my-cluster-kafka-brokers.strimzi.svc.clusterset.local
          - my-cluster-broker-100.cluster2.my-cluster-kafka-brokers.strimzi.svc.clusterset.local
          - my-cluster-broker-101.cluster2.my-cluster-kafka-brokers.strimzi.svc.clusterset.local
```

This ensures that the SANs are included in the certificates provided to each broker. 
However, this approach may inject all broker FQDNs into every broker's certificate, which is not ideal.

2. **Controller Pods**: Unlike brokers, controller pods do not inherit listener configurations from the Kafka CR and use a single control plane listener (TCP 9090). 
This requires adding SANs for the controller’s communication manually, which is not currently supported by Strimzi and is not considered an optimal solution.

#### Extending KafkaNodePool Custom Resource for Cross-Cluster Communication

To integrate Submariner as a cross-cluster communication solution in Strimzi, 
the KafkaNodePool Custom Resource (CR) should be extended to include configuration fields that support various cross-cluster technologies. 
This section outlines the proposed changes to the KafkaNodePool CR, 
along with an explanation of their roles in enabling communication across Kubernetes clusters.

#### Proposed Changes to the KafkaNodePool Custom Resource
To make the KafkaNodePool CR more flexible and future-proof, we propose the following additions

1. **New Cross-Cluster Technology Field**: Introduce a new field called crossClusterTechnology in the KafkaNodePool CR. 
This field will allow users to specify the technology they wish to use for cross-cluster communication. 
Initially, Submariner will be supported, but this design accommodates future integration with other technologies.

2. **Submariner-Specific Configuration**: If Submariner is chosen as the cross-cluster technology, users must provide a submarinerClusterId. 
This identifier is crucial for Submariner, as it uniquely represents each cluster for tunnel creation and communication.

#### Updated KafkaNodePool CR Example

Below is an updated example of the KafkaNodePool CR with the proposed changes

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controller
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - controller
  storage:
    .........
  crossClusterTechnology:
    technology: submariner
    configuration:
      submarinerClusterId: cluster1
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - broker
  storage:
    ......
    crossClusterTechnology:
      technology: submariner
      configuration:
        submarinerClusterId: cluster1
```

#### Explanation of Changes

1. crossClusterTechnology.technology Field

Purpose: Specifies the cross-cluster communication technology to be used (e.g., submariner, skupper, istio...)

Example: `technology: submariner` indicates that Submariner is the chosen solution for enabling communication across clusters.

2. configuration

Purpose: Contains nested section for each possible technology
         Only the relevant section is populated based on the technology specified

Example: `crossClusterTechnology.configuration.submarinerClusterId` Field

Defines the unique cluster ID used by Submariner to establish tunnels and identify clusters.

Example: `submarinerClusterId: cluster1` assigns a unique identifier for the cluster involved in cross-cluster communication.

#### Operator's Role in Utilizing the CR Fields

The Strimzi Operator will parse the crossCluster section to identify the chosen technology and apply the relevant configurations

- **Generating advertised.listener and controller.quorum.voter Addresses**: The Operator will use the submarinerClusterId 
to construct addresses in the format `<broker-name>.<submarinerClusterId>.<namespace>.svc.clusterset.local`. 
This ensures that `advertised.listener` and `controller.quorum.voters` are accessible across connected clusters, facilitating data replication and leader election.

- **Certificate SANs Configuration**: During the creation of SSL/TLS certificates for brokers and controllers, 
the Operator must include the Submariner-exposed service addresses in the Subject Alternative Name (SAN) entries. 
This guarantees that hostname verification succeeds when traffic flows between clusters.

#### Example of Updated advertised.listener and controller.quorum.voters

The Operator will update advertised.listener and controller.quorum.voters configurations as follows

```yaml

advertised.listeners=REPLICATION-9091://my-cluster-broker-0.cluster1.my-cluster-kafka-brokers.strimzi.svc.clusterset.local:9091,PLAIN-9092://my-cluster-broker-0.cluster1.my-cluster-kafka-brokers.strimzi.svc:9092,TLS-9093://my-cluster-broker-0.cluster1.my-cluster-kafka-brokers.strimzi.svc:9093

controller.quorum.voters=101@my-cluster-controller-101.cluster2.my-cluster-kafka-brokers.strimzi.svc.clusterset.local:9090,1@my-cluster-controller-1.cluster1.my-cluster-kafka-brokers.strimzi.svc.clusterset.local:9090
```

These updates ensure brokers and controllers can be discovered and communicate across clusters without relying on traditional external access methods.

#### Resource cleanup on remote Kubernetes clusters
As some of the Kubernetes resources will be created on a remote cluster, we will not be able to use standard Kubernetes approaches for deleting resources based on owner references.
The operator will need to delete remote resources explicitly when the owning resource is deleted.

- The exact mechanism that will be used for such cleanup in various scenarios is not detailed out yet and will be added here before the proposal is complete.

#### Network policies
In a stretch Kafka cluster, some Network policies will be relaxed to allow communication from other Kubernetes clusters that are specified as targets in various KafkaNodePool resources.
This will allow brokers/controllers on separate Kubernetes clusters to communicate effectively.

#### Secrets
We need to create Kubernetes Secrets in the central cluster that will store the credentials required for creating resources on the target clusters.
These secrets will be referenced in the KafkaNodePool custom resource.

#### Entity operator
We would recommend that all KafkaTopic and KafkaUser resources are managed from the cluster that holds Kafka and KafkaNodePool resources, and that should be the cluster where the entity operator should be enabled.
This will allow all resource management/configuration form a central place.
The entity operator should not be impacted by changes in this proposal.

## Additional considerations

Once the general approach is agreed, this proposal will be updated to include any impact on other important aspects like:
- rolling updates
- scaling brokers/controllers
- certificate rotation
- cruise control
- observability

## Affected/not affected projects

This proposal only impacts strimzi-kafka-operator project.


## Rejected alternatives

- Use network technologies like skupper, submariner etc to allow internal services to be visible on other clusters
  -  introduces additional dependencies and complexity to the Strimzi project

![Synchronized ClusterOperator](./images/083-synchronized-clusteroperator.png)


An alternative approach considered was setting up a stretch Kafka cluster with synchronized `KafkaStretchCluster` and `Kafka` custom resources (CRs).
The idea was to introduce a new CR called `KafkaStretchCluster`, which would contain details of all the clusters involved in the stretch Kafka deployment.
The spec would include information such as cluster names, secrets for connecting to each Kubernetes cluster, and a list of node pools across the entire stretch cluster.

The Kafka CR could be created in any of the Kubernetes clusters, and it would be propagated to the remaining clusters through coordinated actions by the Cluster Operator.
Similarly, changes to the Kafka CR could be made in any Kubernetes cluster, and once detected by the Cluster Operator, the changes would be propagated to the CRs in the other clusters.
The `KafkaNodePool` resources would be deployed to individual Kubernetes clusters, requiring users to apply the KafkaNodePool CR in each cluster separately.

### Pros

- Users can modify CRs even if one of the clusters in the setup is unreachable, as there is no central cluster where all CRs must be created.
- This model offers greater resilience and higher fault tolerance.


### Cons

- This approach is challenging to implement, as the Cluster Operator must be coordinated to propagate CR changes across clusters.
- The secrets required to connect to all the Kubernetes clusters would need to be available to all clusters, raising potential security concerns.
- Since Kafka CRs are spread across clusters, it becomes difficult to identify the overall cluster state from the Kafka status, as each CR is reconciled by its own Cluster Operator.
