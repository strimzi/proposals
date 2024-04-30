# Improve handling of CA renewals and replacements in operands based on the Kafka clients

This proposal addresses the Strimzi issue [strimzi/strimzi-kafka-operator#7726](https://github.com/strimzi/strimzi-kafka-operator/issues/7726).

## Current situation

Strimzi currently supports following operands, which are based on Kafka clients:
* Kafka Connect
* Kafka MirrorMaker 1
* Kafka MirrorMaker 2
* Kafka Bridge

In these components, users can enable TLS encryption and configure the trusted certificates in the custom resources:

```yaml
# ...
tls:
  trustedCertificates:
    - secretName: my-secret
      certificate: my-ca-public-key.crt
    - secretName: my-secret
      certificate: my-second-public-key.crt
    - secretName: my-other-secret
      certificate: my-other-ca-public-key.crt
```

Users can configure multiple trusted certificates by specifying the Kubernetes Secret where the trusted certificate is stored and the corresponding key within that Secret.
These certificates are added to the trust store of the operand as trusted certificates.

Similar logic is also used for other configurations, such as configuring trusted certificates for communication with OAuth authorization servers.

## Motivation

The configuration of trusted certificates works fine under normal circumstances.
But it has limitations when the CA that should be trusted is renewed or replaced.

For simple CA renewals, typically both new and old CA public keys are valid at the same time (for a limited period of time - e.g. 30 days) and only one of them needs to be trusted.
In an ideal situation, when renewing the CA, the Kubernetes Secret is updated and the new public key value automatically replaces the old value under the same key, such as `ca.crt`.
Strimzi will then automatically roll the operand to load the new public key.
This is how it works for example when the operand is connected to a Strimzi-based Apache Kafka cluster running in the same namespace.
You can use the public key Secret that is part of the Kafka cluster:
```yaml
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
```

And everything will happen completely automatically as described above.

But if the Apache Kafka cluster is running in a different namespace, different Kubernetes cluster or is not powered by Strimzi, it might be more complicated:
* Different Kubernetes Secrets might be used for the old and new public keys
* The same Kubernetes Secret might be used, but the public key value might be stored under a different key inside the Secret

That means the user has to edit the custom resource at the right time and add the new certificate to the list of the trusted certificates.

Things get even more complicated when the CA is completely replaced.
In such cases, the operand needs to typically trust both the old and new CA at the same time.
This is because during the migration of the Apache Kafka cluster from the old CA to the new one, different brokers might use server certificates from both CAs at the same time.
If you migrate to the new CA manually, you might still be able to orchestrate everything by editing the custom resource and modifying the trusted certificates as needed.

But when the whole process is automated, it might not be easy to orchestrate it manually.
This applies even in the ideal situation where the Apache Kafka cluster is based on Strimzi and runs in the same namespace.
For example, the following happens when a user triggers CA replacement using the `strimzi.io/force-replace` annotation: 
1. Strimzi generates the new CA public and private keys and stores the new CA public key as `ca.crt` in the Kubernetes Secret.
   The public key of the old CA is renamed to `ca-<timestamp>.crt`.
2. Strimzi starts rolling the Kafka brokers to make sure they trust the new CA.
   At this point in time, the Kafka brokers still use the old server certificates signed by the old CA.
3. The Kafka client based operand (e.g. Kafka Connect cluster) using the typical `.spec.tls` configuration:
   ```yaml
   tls:
      trustedCertificates:
         - secretName: my-cluster-cluster-ca-cert
           certificate: ca.crt
   ```
   detects that the `ca.crt` in the Secret was updated and triggers a rolling update of the operand to use the new CA.
   After the rolling update, it ceases to trust the old CA still in use by the Kafka brokers, resulting in the operand failing to connect to the brokers.
4. After all brokers are rolled to trust the new CA, another rolling update happens and new server certificates signed by the new CA are rolled out to the Kafka brokers in another rolling update.
5. Once all brokers are rolled once again and use the new CA, the operand such as Kafka Connect will be able to connect again and start working.
   The impact and length of the outage will depend on many different aspects:
   * How many nodes does the operand such as Kafka Connect have.
     If it is only one replica, the whole operand will be down.
     With multiple replicas, only one replica should be down.
   * How big and busy the Kafka cluster is.
     A small cluster might take few minutes to move to the new CA completely.
     A big cluster with many nodes that are busy and need more time to re-sync between restarts might take much longer.
6. After all brokers (and other components such as ZooKeeper, Topic and User Operator, Cruise Control etc.) use the new CA, another rolling update will be done to completely remove the trust to the old CA by removing the earlier `ca-<timestamp>.crt` from the Secret and rolling all components once again.
   This last rolling update does not impact operands such as Kafka Connect.

## Proposal

This proposal suggests improving the API of our custom resources that we use for configuring the trusted certificates.
It should allow loading more certificates without an exact specification of the keys in the Secret.
To do so, we should add a new field called `extension`.
This field will allow users to specify the extension of the files that should be loaded from the Secret without specifying the exact name.
The existing `certificate` field will remain supported.
But it will not be marked as required anymore and instead the CRDs will require one of the fields `certifciate` or `extension` to be specified.

This would allow users to instruct Strimzi to load all files with a specific extension as trusted certificates.
For example with a Secret like this:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
data:
  ca1.crt: <CA-1>
  ca2.crt: <CA-2>
  ca3.crt: <CA-3>
```

The user can load all three CAs with the following configuration:
```yaml
  tls:
    trustedCertificates:
      - secretName: my-secret
        extension: crt
```

As today, when one of the existing certificates changes, Strimzi will automatically toll the operand to update the trusted certificates.
The same will happen also when a new record is added to the Secret that matches the extension - e.g. `ca-4.crt`.

In the scenario described in the _Motivation_ section, the new API will work like this:
1. Strimzi generates the new CA public and private keys and stores the new CA public key as `ca.crt` in the Kubernetes Secret.
   The public key of the old CA is renamed to `ca-<timestamp>.crt`.
2. Strimzi starts rolling the Kafka brokers to make sure they trust the new CA.
   At this point in time, the Kafka brokers still use the old server certificates signed by the old CA.
3. The Kafka client based operand (e.g. Kafka Connect cluster) using the new `.spec.tls` configuration:
   ```yaml
   tls:
      trustedCertificates:
         - secretName: my-cluster-cluster-ca-cert
           extension: crt
   ```
   detects that the `ca.crt` in the Secret was updated and a new `ca-<timestamp>.crt` added, triggering a rolling update of the operand to use the new CA.
   After the rolling update, it trusts both the old and new CAs and continues working without any problems while the CA is being replaced.
4. After all brokers are rolled to trust the new CA, another rolling update happens and new server certificates signed by the new CA are rolled out to the Kafka brokers in another rolling update.
5. Once all brokers are rolled once again and use the new CA, the operand such as Kafka Connect will be able to connect again and start working.
   The impact and length of the outage will depend on many different aspects:
   * How many nodes does the operand such as Kafka Connect have.
     If it is only one replica, the whole operand will be down.
     With multiple replicas, only one replica should be down.
   * How big and busy the Kafka cluster is.
     A small cluster might take few minutes to move to the new CA completely.
     A big cluster with many nodes that are busy and need more time to re-sync between restarts might take much longer.
6. After all brokers (and other components such as ZooKeeper, Topic and User Operator, Cruise Control etc.) use the new CA, another rolling update will be done to completely remove the trust to the old CA by removing the earlier `ca-<timestamp>.crt` from the Secret and rolling all components once again.
   This last rolling update does not impact operands such as Kafka Connect.
7. The removal of `ca-<timestamp>.crt` from the Secret will cause rolling update of the client-based operand (e.g. Connect).
   This rolling update will remove the trust to the old CA from the operand as well.

### Examples

The examples for the operands based on Kafka clients will be updated to use the new field:
```yaml
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        extension: crt
```

This will make sure that when used with a Strimzi based cluster, everything - including CA replacement - will work without any outages and issues.

## Affected projects

This proposal affects only the Strimzi Cluster Operator project.

## Backwards compatibility

There is no impact on backwards compatibility.
This proposal does not remove any Strimzi APIs.
It only extends them.
All existing custom resources will work without any change.

## Rejected alternatives

### Using a regular expression instead of extension

One option considered for this proposal was to use a pattern to specify the matching certificates instead of extension.
However, when using a pattern / regular expression:
* The correct value would depend on where is the pattern applied (e.g. in Java, in shell when starting the container image, in some different language in the future).
* For non-expert users it might be hard to understand the exact format and the different regular expression variants.
* It would be hard to validate the the pattern / regular expression.

That is why this option was rejected.

### Hardcoding the extension

Another considered option was to not add a new field to specify the `extension` at all in the API and just make the `certificate` field optional.
When the `certificate` field would not be specified, all items from the Secret with the `.crt` extension would be loaded.
This option was rejected because in some cases, users might use different extension such as `.pem` for public keys and having it hardcoded might make it not work for them.

It was also considered to not add the `extension` field, but instead of hardcoding the `.crt` extension to simply try to load everything from the Secret.
But in this situation, we might struggle to identify the fields that contain a public key.
We would likely need to parse them and that might not produce reliable results.
So this alternative was rejected as well.

### Making `certificate` and `extension` fields completely optional

Another option - similar to the previous one - was to keep the `extension` field.
But make specifying `extension` (as well as `certicicate`) fully optional.
For example:
```yaml
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
```

And when none of them would be specified, it will be treated as if `extension` was set to `crt`.
This is something what can be considered and added even later if desired.
