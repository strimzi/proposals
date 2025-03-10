# cert-manager Maintenance Windows

Proposal [100](https://github.com/strimzi/proposals/blob/main/100-external-certificate-manager.md) proposed integrating Strimzi with [cert-manager](https://cert-manager.io/) as an option for managing certificates.
Proposal 100 did not mention anywhere the option for users to specify maintenance time windows.
This proposal describes how maintenance time windows should be incorporated into the cert-manager integration implementation.

## Current situation

Strimzi allows users to specify maintenace time windows to manage when the Cluster Operator will renew certificates.
This setting only applies when Strimzi is managing the Cluster/Clients CA, not when it is provided by the user.

## Motivation

When using cert-manager for certificate management, Strimzi will not have any control over when certificates are renewed.
The Kafka Pods must be restarted to present new certificates, and users should be able to control when that is done.

## Proposal

When Strimzi is configured to use cert-manager for managing certificates (as described in proposal 100), users can specify a maintenance time window to influence when Strimzi will start using new Cluster CA end-entity certificates.
This will only apply to Cluster CA end-entity certificates, because in the proposal the user is responsible for managing the CA and providing the Cluster and Clients CA public keys to Strimzi.

Users will configure maintenance time windows as follows:

* Configure an array of strings using the `spec.maintenanceTimeWindows` property of the Kafka resource.
* Each string is a cron expression interpreted as being in UTC (Coordinated Universal Time)

This proposal proposes an update to the section [Tracking changes to Cluster CA end-entity certificates]https://github.com/strimzi/proposals/blob/main/100-external-certificate-manager.md#tracking-changes-to-cluster-ca-end-entity-certificates) as follows:

```markdown
197 #### Tracking changes to Cluster CA end-entity certificates
198 
199 Cert-manager will be responsible for renewing all Cluster CA end-entity certificates.
200 When a certificate is renewed cert-manager will update the related Secret.
201 During a reconciliation Strimzi will check the hash of the certificate stored in cert-manager Secrets to see if an update is needed.
202 
203 - When cert-manager renews an end-entity certificate Strimzi will perform cert path validation using the `java.security` libraries to determine whether the new certificate is trusted by the latest Cluster CA public cert.
204 - If the certificate is trusted:
203 + When cert-manager renews an end-entity certificate Strimzi will verify if the maintenance time windows is satisfied and if it is, perform cert path validation using the `java.security` libraries to determine whether the new certificate is trusted by the latest Cluster CA public cert.
204 + If the maintenance time windows is satisfied and the certificate is trusted:
205 * Strimzi will copy over the new certificate into its own Secret.
206 * Strimzi will update the `strimzi.io/server-cert-hash` annotation to match the new certificate.
207 * Strimzi will update the `strimzi.io/cluster-ca-cert-generation` annotation to match the `strimzi.io/ca-cert-generation` annotation on the `<CLUSTER_NAME>-cluster-ca-cert` Secret.
208
209 - If the certificate is not trusted:
210 - * Strimzi will do nothing and complete the reconciliation loop as normal.
211 - * During future reconciliations Strimzi will repeat the cert path validation steps until the user has updated the Cluster CA public cert given to Strimzi to one that trusts the new certificate.
209 + If the maintenance time windows is not satisfied, or the certificate is not trusted:
210 + * Strimzi will do nothing and complete the reconciliation loop as normal.
211 + * During future reconciliations Strimzi will repeat the maintenance time windows and cert path validation checks until the user has updated the Cluster CA public cert given to Strimzi to one that trusts the new certificate and the maintenance time windows is satisfied.
```

## Affected/not affected projects

This affects the Cluster Operator.

## Compatibility

N/A - this proposal does not have any additional compatibility concerns beyond those listed in the original cert-manager proposal.

## Rejected alternatives

### Respecting maintenance time windows for Cluster and Clients CA public keys

Strimzi could verify that maintenance time windows are satisfied before rolling the Kafka Pods to trust a new Cluster or Clients CA public key.
Proposal 100 requires the user to provide the Cluster and/or Clients CA public key to Strimzi.
This means although the public key may be issued by cert-manager, the user can choose when to provide it to Strimzi.
The user can choose to only update the public key during their maintenance time windows, removing the need for Strimzi to take this into account when rolling Pods to trust a new Cluster or Clients CA public key.
