<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Decouple CA certificate management from end-entity certificate management

This proposal aims to decouple the management of the cluster and client CA certificates from the rolling out of trust of those certificates to components in a Strimzi cluster.

## Current situation

In order to provide a secure-by-default, TLS out-of-the-box experience Strimzi ended up implementing its own CA operations internally, within the cluster operator.
It does this by using `openssl` to generate a self-signed root CA certificate and key which is then used to directly sign EE certificates (i.e. intermediate certificates are not used).
In fact Strimzi has two such CAs.
The "cluster CA" is used for issuing certificates for Strimzi components, such as ZooKeeper and Kafka brokers.
The "clients CA" is used for issuing certificates for Kafka clients, for example via the User Operator.

The cluster CA root certificate is added to Strimzi component trust stores (so that, for example, ZooKeeper nodes trust each other's and brokers' certificates, and brokers trust certs issued to other brokers, etc).


![Cluster CA relationships](./images/073-cluster-ca.svg)

> In the diagram, the red lines show trust.
> For example, because the Kafka Broker trusts the Cluster CA certificate,
> and the Cluster CA certificate signed the ZK EE certificate
> the broker will trust the ZK EE certificate presented by the ZK node
> during TLS handshake.

The cluster CA root certificate also needs to be trusted by Kafka clients connecting to the cluster (so that clients trust the broker's EE certificates).
The client CA root certificate is added to the broker trust stores too, so that the brokers will trust certificates issued to Kafka client applications.

![Client CA relationships](./images/073-clients-ca.svg)

Currently as part of a reconciliation loop the CaReconciler handles three things:
1. Creating or renewing the cluster and client CA certificates and storing them in Kubernetes Secrets
2. Updating the Kubernetes Secret used by the cluster operator's Admin client instances to trust the cluster CA certificate
3. Triggering a rolling update of the ZooKeeper and Kafka nodes to trust the new CA certificates

Steps 2 and 3 above all make the assumption that all component end entity certificates have been signed by a single cluster CA certificate, which is stored in a specific Kubernetes Secret.

The cluster operator uses annotations to keep track of how the CA certificate and key are managed:
* strimzi.io/cluster-ca-key-generation tracks the CA key generation to determine whether new end entity certificates are required
* strimzi.io/cluster-ca-cert-generation tracks the CA certificate generation to determine whether components trust the CA

## Motivation

Removing the assumption that there is a _single_ root CA certificate stored in a special Kubernetes Secret that is directly used for trust would enable much more flexibility in CA handling.
This will be compatible with commonly deployed certificate management solutions such as [Cert Manager][cmio], or [Vault][vault].
These solutions generate certificates that (from the PoV of an end-entity certificate requester) are signed by arbitrary CAs.
Using those solutions therefore requires TLS peers to be able to trust multiple root CA certificates.
In addition these solutions cannot be responsible for rolling updates to components like Kafka.
This means that end-entity certificate generation and the rollout of trust in the CA certificates referenced in those end-entity certificates, needs to be decoupled.
This would be highly valuable for organizations with compliance requirements with regard to certificates.

## Proposal

Strimzi should be updated to decouple the management of the cluster and clients CA certificates, and the consumption of those CA certificates by components (such as ZooKeeper, Kafka and cluster operator).

To do this we will introduce new Kubernetes Secrets that are used to keep track of CA certificates and their current state.
These new Kubernetes Secrets are separate from the ones that are mounted into the components to use a truststore.
The cluster operator will update these new Kubernetes Secrets when it renews cluster or client CA certificates.
The cluster operator and user operator will copy the CA certificates from the new Kubernetes Secrets into the existing truststore Kubernetes Secrets that are mounted into components.
The cluster operator and user operator will update the new Kubernetes Secrets to reflect whether the CA certificates are currently trusted and/or in use or not.
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

The existing Kubernetes Secrets, named using `${cluster}-${ca}-certs` and passed via Secret volume mounts to containers would not change under this proposal.

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

New cluster creation process:
1. The CaReconciler creates the CA certificates and places them in the relevant Kubernetes Secret with the state UNTRUSTED
2. The cluster and user operator, during their respective reconcile loops:
   1. Add the CA certificate to the relevant component truststore Kubernetes Secret and change the state to TRUSTED_UNUSED
   2. Generate end entity certificates using the CA and add the EE certificate to the relevant component keystore Kubernetes Secret and change the state to TRUSTED_IN_USE

Update process (e.g. CA private key has been updated):
1. The CaReconciler generates a new CA certificate and places it in the relevant Kubernetes Secret with the state UNTRUSTED. It also updates the old CA certificate to have the state PHASE_OUT.
2. The cluster and user operator, during their respective reconcile loop:
   1. Add the CA certificate to the relevant component truststore Kubernetes Secret and change the state to TRUSTED_UNUSED
   2. Generate end entity certificates using the CA and add the EE certificate to the relevant component keystore Kubernetes Secret and change the state to TRUSTED_IN_USE
   3. Update the old CA certificate to ZOMBIE, indicating that no EE certicates are using the CA certificate
3. In a future loop the CaReconciler removes the CA certificate from the shared Kubernetes Secret (because it is observed to be in the `ZOMBIE` state).
4. The cluster and user operator remove the CA certificate from the truststore Kubernetes Secret

Notes:
1. The private key for the self-signed cluster CA certificate in a Kafka cluster called **foo**, can be stored in `foo-cluster-ca`.
2. The strimzi.io/cluster-ca-key-generation can be used by the cluster operator when managing the self-signed cluster CA certificate
3. A new annotation strimzi.io/cluster-ca-cert-fingerprint on components can be used to store the fingerprint of the issuing CA. This removes the need for the current annotation strimzi.io/cluster-ca-cert-generation which just holds an integer.

### Future changes

Once this proposal is implemented, it allows in future to update to change both the issuing of end entity certificates and the management of the CA certificates to be done by certificate management solutions, similarly to the [closed proposal][pr46].
This proposal is the first step because it decouples the management of the CA certificates, from the rolling out of trust of those CA certificates.
The next logical step will be to change the end-entity certificate issuance process, so that an end-entity certificate and it's root CA certificate can be received at the same time.
Currently it is assumed that the CA certificate is already issued and is used for signing the end-entity certificates, rather than an end-entity certificate has been received that has been signed by (from the PoV of the requesting operator) and arbitrary CA.

## Affected/not affected projects

This affects the strimzi-kafka-operator project only.

## Compatibility

Call out any future or backwards compatibility considerations this proposal has accounted for.

## Rejected alternatives

[Pull request 46][pr46] described a more complete picture of how we could update certificate management.
This proposal describes a smaller change that could be implemented first.
This first step brings the advantages of better separation of interests in terms of the Kubernetes Secrets used and making it easier to reason about the state of the system, making debugging easier.
By describing only this smaller change the aim is to make it easier to reach consensus and more likely that this change can be added to Strimzi in a timely manner.


[cmio]: https://cert-manager.io/
[pr46]: https://github.com/strimzi/proposals/pull/46
[vault]: https://www.vaultproject.io
