<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Decouple CA certificate management from end-entity certificate management

This proposal aims to decouple the management of the cluster and client CA certificates from the rolling out of trust of those certificates to components in a Strimzi cluster.

## Current situation

When deploying Kafka clusters, in order to provide a secure-by-default, TLS out-of-the-box experience Strimzi ended up implementing its own CA operations internally, within the cluster operator.
It does this by using `openssl` to generate a self-signed root CA certificate and private key which is then used to directly sign end-entity (EE) certificates (i.e. intermediate certificates are not used).
In fact Strimzi has two such CAs.
The "cluster CA" is used for issuing certificates for Strimzi components:
* ZooKeeper nodes
* Kafka nodes
* User and Topic operators
* Cruise Control
* Kafka Exporter

The "clients CA" is used for issuing certificates for user applications using the User Operator, or through another external mechanism chosen by the user.

The cluster CA root certificate is added to Strimzi component trust stores (so that, for example, ZooKeeper nodes trust each other's and brokers' certificates, and brokers trust certs issued to other brokers, etc).

![Cluster CA relationships](./images/073-cluster-ca.svg)

> In the diagram, the red lines show trust.
> For example, because the Kafka Broker trusts the Cluster CA certificate,
> and the Cluster CA certificate signed the ZK EE certificate
> the broker will trust the ZK EE certificate presented by the ZK node
> during TLS handshake.

The cluster CA root certificate also needs to be trusted by Kafka clients connecting to the cluster so that clients trust the certificates presented by the brokers.
The client CA root certificate is added to the broker trust stores too, so that the brokers will trust certificates issued to Kafka client applications.

![Client CA relationships](./images/073-clients-ca.svg)

### Existing Kubernetes Secrets

Strimzi currently uses the following Kubernetes Secrets to store the CA certificates and private keys:
* <CLUSTER_NAME>-cluster-ca-cert for the Cluster CA root certificate
* <CLUSTER_NAME>-cluster-ca for the Cluster CA private key
* <CLUSTER_NAME>-clients-ca-cert for the Clients CA root certificate
* <CLUSTER_NAME>-clients-ca for the Clients CA private key

Strimzi currently uses the following Kubernetes Secrets to store end-entity certificates (signed by the cluster CA) that components will present to clients:
* <CLUSTER_NAME>-kafka-brokers for Kafka brokers
* <CLUSTER_NAME>-zookeeper-nodes for ZooKeeper nodes

Strimzi currently uses the following Kubernetes Secrets to store client certificates (signed by the cluster CA) that components will present to Kafka or ZooKeeper when making requests:
* <CLUSTER_NAME>-cluster-operator-certs for the Cluster Operator
* <CLUSTER_NAME>-entity-topic-operator-certs for the Entity Topic Operator
* <CLUSTER_NAME>-entity-user-operator-certs for the Entity User Operator
* <CLUSTER_NAME>-kafka-exporter-certs for the Kafka Exporter
* <CLUSTER_NAME>-cruise-control-certs for Cruise Control

Strimzi also uses the following annotations on Kubernetes Secrets:
* strimzi.io/ca-cert-generation to indicate the generation of the Cluster CA or Client CA certificate in the Kubernetes Secret
* strimzi.io/ca-key-generation to indicate the generation of the Cluster CA or Client CA private key in the Kubernetes Secret

Strimzi uses the following annotations on Kubernetes Pods and Kubernetes Secrets to indicate the generation of the Cluster CA certificate used to sign the certificates that are either stored in the Kubernetes Secret, or presented by the Kubernetes Pod:
* strimzi.io/cluster-ca-cert-generation for certificates signed by the Cluster CA
* strimzi.io/clients-ca-cert-generation for certificates signed by the Clients CA

Strimzi uses the following annotation on Kubernetes Pods to indicate the generation of the Cluster CA private key used to sign the cluster CA certificate that the pod currently trusts
* strimzi.io/cluster-ca-key-generation

### Existing reconcile flow

The reconcile loop for certificates is split over a couple of classes that run one after another, those are:
1. CaReconciler
2. ZooKeeperReconciler
3. KafkaReconciler
4. EntityOperatorReconciler
5. CruiseControlReconciler
6. KafkaExporterReconciler

Currently, as part of a reconciliation loop, the CaReconciler handles three things:
1. Verifying the Cluster CA and Client CA Kubernetes Secrets. For each CA:
   1. If Strimzi is managing the CA it either creates or renews the CA certificate and key and stores them in the Kubernetes Secrets listed above
   2. If Strimzi is **not** managing the CA it checks that the Kubernetes Secrets for the CA certificate and private key are present in the cluster
2. Reconciles the <CLUSTER_NAME>-cluster-operator-certs Kubernetes Secret used by the cluster operator's Admin client when connecting to Kafka and ZooKeeper
   1. This only occurs if all pods in the cluster have the correct strimzi.io/cluster-ca-key-generation annotation, meaning the Cluster CA certificate that is being put into the Kubernetes Secret is already trusted by all components
3. Maybe triggering a rolling update of the ZooKeeper and Kafka nodes to trust a new Cluster CA certificate. This only happens if either:
   1. The Cluster CA key got replaced during the current reconciliation, or
   2. Some pods do not trust the latest Cluster CA certificate, which implies the operator previously updated the Cluster CA key and a rolling update was started during a previous reconciliation, but it was not completed

ZooKeeperReconciler and KafkaReconciler classes:
1. Generate the certificates they will present to clients and store them in the Kubernetes Secrets listed above, with the cluster-ca-cert-generation (and clients-ca-cert-generation for Kafka) annotation set to the current value passed in with the Cluster CA (and Clients CA).
2. Create the pods for the cluster and add the cluster-ca-cert-generation and cluster-ca-key-generation annotations, set to the current values passed in with the Cluster CA.

EntityOperatorReconciler, CruiseControlReconciler and KafkaExporterReconciler generate the certificates they will use to authenticate with Kafka and ZooKeeper and store them in the Kubernetes Secrets listed above, with the cluster-ca-cert-generation set to the current value passed in with the Cluster CA.

### Example flows

When a new cluster is first created the flow is:

1. Either user or CaReconciler creates the Cluster and Client CA certificates and keys and stores them in Kubernetes Secrets
2. The CaReconciler generates the Cluster Operator certificate and stores it in the <CLUSTER_NAME>-cluster-operator-certs Kubernetes Secret
3. The component reconcilers (KafkaReconciler, ZooKeeperReconciler, EntityOperatorReconciler etc) generate their certificates, store them in Kubernetes Secrets and annotate the Kubernetes Secrets and Pods with the cert and key generations

When the CA private key is updated (either because it has expired or the user has incremented the generation to indicate they've manually updated the Kubernetes Secret):
1. If Strimzi is managing the CA, the CaReconciler renews the Cluster and Client CAs certificates and keys. The old certificate and key are kept in the Kubernetes Secret and renamed to have the key ca-YYYY-MM-DDTHH-MM-SSZ.crt.
2. The CaReconciler skips updating the <CLUSTER_NAME>-cluster-operator-certs Kubernetes Secret (since the components still only trust the old CA)
3. The CaReconciler rolls the ZooKeeper and Kafka nodes to trust the new CA certificates
4. The ZooKeeper and Kafka reconcilers generate new certificates signed by the new CA key and roll the pods to use the new certificates
5. The EntityOperator, CruiseControl and KafkaExporter reconcilers generate new certificates signed by the new CA key amd roll the pods to use the new certificates
6. A new reconciliation starts
7. The CaReconciler updates the <CLUSTER_NAME>-cluster-operator-certs Kubernetes Secret with the new CA certificate (since the generations are now correct)
8. The CaReconciler removes the old CA certificates and keys that have been named ca-YYYY-MM-DDTHH-MM-SSZ.crt.

The existing logic assumes two things:
 - all certificates used by components have been signed by the same root CA certificate
 - the CaReconciler runs and (if required) has either created or updated the CA certificates and keys before the other reconcilers run, i.e. the CA certificate is already present when e.g. the KafkaReconciler starts

## Motivation

In future it would be beneficial if Strimzi can integrate with commonly deployed certificate management solutions, such as [Cert Manager][cmio], or [Vault][vault].
These solutions work by having the user request certificates from a particular issuer.

Integrating with these solutions requires a change in how things are managed. Specifically:
 - Strimzi cannot guarantee that the issuer will issue every certificate signed by the same root CA certificate. 
   - For example Cert Manager will include the CA certificate to trust when it issues the certificate (if the CA certificate is not a public one)
   - Using those solutions therefore requires TLS peers to be able to trust multiple root CA certificates.
 - These solutions cannot be responsible for rolling updates to components like Kafka.
   - This means that end-entity certificate generation and the rollout of trust in the CA certificates referenced in those end-entity certificates, needs to be decoupled.
 - Certificates will likely be requested in an asynchronous manner, which means it can't be assumed that e.g. the CA certificate is already present when the component reconcilers run.
This would be highly valuable for organizations with compliance requirements with regard to certificates.

In addition, currently it is hard to see the current state of trust within the cluster when certificates are being renewed.
The proposed changes make it easier to reason about the current state and track the roll out of trust.

## Proposal

Strimzi should be updated to decouple the management of the cluster and clients CA certificates, and the consumption of those CA certificates by components (such as ZooKeeper, Kafka and cluster operator).

To do this we will introduce new shared Kubernetes Secrets that are used to keep track of CA certificates that are currently in use and their current state with respect to issued end entity certificates.
These new Kubernetes Secrets are separate from the ones that are used by Strimzi to store the latest CA certificate and key (or by the user when they are managing their own certificate).
The cluster operator will update these new Kubernetes Secrets when it renews cluster or client CA certificates (or observes that the user has updated them).
The cluster operator and user operator will copy the CA certificates from the existing Kubernetes Secrets into the new shared Kubernetes Secrets.
The new shared Kubernetes Secrets will be mounted into components to use as a truststore.
The cluster operator and user operator will update the new shared Kubernetes Secrets to reflect whether the CA certificates are currently trusted and/or in use or not.
This will make it easier to reason about which CA certificates are in use/trusted, and allow the management of CA certificates to be evolved further in future (see [Future changes](#future-changes)).
These new Kubernetes Secrets can be updated by multiple processes (i.e the cluster operator when issuing the CA certificates, and the cluster and user operators when updating the state of CA certificates).

### New Kubernetes Secret format for CA certificate state

A new Kubernetes Secret is introduced which is primarily updated by the cluster operator in its role as a CA, but also updated by the cluster operator and user operator to reflect whether the individual CA certificates are trusted and/or in use in the cluster.

```yaml
kind: Secret
metdata:
  name: ${cluster}-${ca-type}-trust # e.g. foo-cluster-ca-trust
type: strimzi.io/trust
data:
  ${fingerprint}.${state}: <PEM encoded CA certificate>
```

The `fingerprint` is the fingerprint/thumbprint of the certificate. I.e. the SHA-1 hash of the ASN.1 DER-encoded form of the X509 certificate, which is a commonly used way of identifying certificates using common tooling.
The `state` can be any of the states of the CA certificate trust state machine described below.
Including the state in the item name facilitates understanding the current role of the Kubernetes Secret within the cluster.

Problems from multiple processes concurrently updating the Kubernetes Secret are avoided by using `metadata.resourceVersion` to make conditional requests when updating that Secret. Note also that only one operator makes each state transition.

### CA certificate state machine

The "CA certificate trust state machine" has the following states:

```
  UNTRUSTED // not yet trusted everywhere it needs to be 
   |
   v
  TRUSTED_UNUSED // trusted everywhere it needs to be, but no EE certificates yet issued with this CA certificate
   |
   v
  TRUSTED_IN_USE // trusted everywhere, EE certificates issued using this CA certificate
   |
   v
  PHASE_OUT // trusted everywhere, but new end-entity certificates will not be issued using this CA certificate
   |
   v
  ZOMBIE // still trusted by some component, but with no issued end-entity certificates in current use.
```

When a new cluster is first created the flow is:

1. Either user or CaReconciler creates the Cluster and Client CA certificates and keys and stores them in Kubernetes Secrets.
2. The CaReconciler takes the Cluster CA certificate and places it in the new shared Kubernetes Secret with the state UNTRUSTED
3. At the beginning of the reconcile loop of components, the cluster operator generates the Cluster Operator certificate and stores it in the <CLUSTER_NAME>-cluster-operator-certs Kubernetes Secret
   It does this even though the state is UNTRUSTED, because the Kubernetes Secret was missing
4. The component reconcilers (KafkaReconciler, ZooKeeperReconciler, EntityOperatorReconciler etc):
   1. generate their certificates, store them in Kubernetes Secrets
   2. create their pods annotated with the certificates Kubernetes Secret and new shared Kubernetes Secret mounted onto the pod
5. At the end of the reconcile loop of components, the cluster operator updates the shared Kubernetes Secret to change the state of the CA certificate to TRUSTED_IN_USE

When the CA private key is updated (either because it has expired or the user has incremented the generation to indicate they've manually updated the Kubernetes Secret):

1. If Strimzi is managing the CA, the CaReconciler renews the Cluster and Client CAs certificates and keys. The old certificate and key can be removed from the Kubernetes Secret.
2. The CaReconciler takes the Cluster CA certificate and places it in the new shared Kubernetes Secret with the state UNTRUSTED. It also updates the old CA certificate to have the state PHASE_OUT.
3. The component reconcilers (KafkaReconciler, ZooKeeperReconciler, EntityOperatorReconciler etc) see the new UNTRUSTED CA certificate and:
   1. Roll the pods to trust the new CA certificate
4. At the end of the reconcile loop the cluster operator updates the shared Kubernetes Secret to change the state of the CA certificate to TRUSTED_UNUSED
5. This change triggers a new reconcile loop
6. At the beginning of the reconcile loop the cluster operator sees there is a CA certificate in the state TRUSTED_UNUSED and regenerates the Cluster Operator certificate, signed by the new CA key and stores it in the <CLUSTER_NAME>-cluster-operator-certs Kubernetes Secret
7. The component reconcilers (KafkaReconciler, ZooKeeperReconciler, EntityOperatorReconciler etc) see the new TRUSTED_UNUSED CA certificate and:
   1. Generate new certificates signed by the new CA key
   2. Roll the pods to use the new certificates
8. At the end of the reconcile loop the cluster operator updates the shared Kubernetes Secret to change the state of the CA certificate to TRUSTED_IN_USE and the old CA certificate to ZOMBIE, indicating that no end entity certificates are using the CA certificate
9. In a future loop the CaReconciler removes the old CA certificate from the shared Kubernetes Secret (because it is observed to be in the `ZOMBIE` state).

Notes:
The existing annotations strimzi.io/ca-cert-generation and strimzi.io/ca-key-generation can still be used by the CaReconciler and the cluster operator to track which certificates the Kubernetes Pods are trusting/using.

### Future changes

Once this proposal is implemented, it allows in future to update to change both the issuing of end entity certificates and the management of the CA certificates to be done by certificate management solutions, similarly to the [closed proposal][pr46].
Other expected changes will be:
 - Refactor the end-entity certificate issuance process, so that components request a certificate and asynchronously are issued a certificate.
 - Refactor the end-entity certificate issuance process, so that when an issued certificate is received, the root CA certificate and certificate chain is included.
 - Refactor the CA certificate management process to allow alternative implementation internally, for example using the Cert Manager API.

## Affected/not affected projects

This affects the strimzi-kafka-operator project only.

## Compatibility

Call out any future or backwards compatibility considerations this proposal has accounted for.

## Rejected alternatives

### Previous proposal

[Pull request 46][pr46] described a more complete picture of how we could update certificate management.
This proposal describes a smaller change that could be implemented first.
This first step brings the advantages of better separation of interests in terms of the Kubernetes Secrets used and making it easier to reason about the state of the system, making debugging easier.
By describing only this smaller change the aim is to make it easier to reach consensus and more likely that this change can be added to Strimzi in a timely manner.

### Using separate Kubernetes Secrets for each component

We could have a separate trust secret for each component (Kafka, ZooKeeper, EntityOperator, CruiseControl, KafkaExporter).
However this would result in a lot of new Kubernetes Secrets and even though the Reconciler classes are separate it is one overall process, so it is acceptable to handle the state transitions at the end of each reconcile loop.

### Using new annotation for tracking certificates

A previous version of this proposal described a new annotation that used the fingerprint of the certificate, instead of the generation currently used.
This can only be used for tracking the rolling update to establish trust, not for rolling out end-entity certificates.
This has been removed from this proposal in favour of keeping the existing method of using a generation, then we can always revisit this in a future proposal to support tools like Cert Manager.

[cmio]: https://cert-manager.io/
[pr46]: https://github.com/strimzi/proposals/pull/46
[vault]: https://www.vaultproject.io
