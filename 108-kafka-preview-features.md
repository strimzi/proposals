## Declarative Management of Kafka Preview Features in Strimzi

## Current Situation:

Kafka preview features (such as `kafka.shared.consumer`) need to be manually enabled using the ¨kafka-features.sh` or the Kafka Admin API.
This works in static clusters but presents issues in dynamic, testing or GitOps-managed environments.
There is currently no way to express these features declaratively within the Kafka custom resource.
Without declarative support, the following issues arise:
* Feature drift from manual changes.
* Loss of settings in ephemeral environments.
* Inability to track changes through Git.

### Kafka Preview Features Explained
Kafka preview features are opt-in capabilities introduced in newer Kafka versions to allow early access to experimental or upcoming functionality.
Early access or Preview features that are not production-ready yet, are disabled by default and must be explicitly enabled on each broker to become usable.

Kafka stores the state of these features in the metadata quorum (ZooKeeper or KRaft, depending on the deployment).
The feature state includes:
* Feature name (e.g., kafka.shared.consumer)
* Version 
*  Enabled/disabled flag

Currently, these features are enabled via:
* The `kafka-features.sh` tool
* The Kafka Admin API (Feature API)

However, these tools require manual execution after cluster creation, which is incompatible with automated GitOps workflows or testing environments.

This proposal introduces a way to declare desired preview features via the Kafka CR, allowing the Strimzi operator to apply and reconcile the correct state automatically.

## Motivation:

These features are disabled by default and must be explicitly enabled on each broker in the cluster.
In environments where clusters are frequently recreated, such as CI/CD pipelines or testing setups, manually enabling these features is both error-prone and hard to maintain.

- In Kafka, preview features are managed per-broker and stored in ZooKeeper or KRaft metadata.
- When a broker starts, it checks the feature state before making it available to Clients.
- If a feature is disabled or at a lower version, functionality depending on it may fail.

In GitOps workflows, lack of declarative configuration breaks the principle of having the full desired cluster state stored in version control.
As a result:

- Teams cannot easily maintain consistent environments across dev/staging/prod.

- Manual intervention is needed after each deployment or cluster recreation.

By supporting preview features through the Kafka CR, Strimzi can offer a seamless, GitOps-compatible, and maintainable experience for all cluster types.

Proposal:
I propose to add native declarative support for preview features in the Kafka CR.

CRD Changes
Extend the Kafka CRD with a new previewFeatures section under .spec.kafka:

spec:
  kafka:
    version: 3.7.0
    previewFeatures:
      - name: kafka.shared.consumer
        enabled: true
        version: 1

Operator Behaviour
1) On reconciliation, Strimzi queries the current preview feature state using the Kafka Admin API.

2) If a mismatch is detected, the Operator applies the desired state using the Admin API or kafka-features.sh.

3) Changes are reconciled continuously to prevent drift.

Benefits
- Declarative and GitOps-friendly.

- Drift protection through automatic reconciliation.

- Consistent environments across deployments.

Considerations
- Requires changes to the Kafka CRD schema.

- Operator logic must be extended to handle feature state reconciliation.

- Validation rules needed to ensure correct feature name, version, and enabled state.

Limitations

- Not declarative through the Kafka CR.

- No automatic reconciliation or drift protection.

- More error-prone due to lack of CR validation.

## Affected Projects:
- strimzi-kafka-operator — Kafka CRD schema and reconciliation logic.

- strimzi-images — Potential updates to include Admin API handling for preview features.

## Compatibility:

The previewFeatures field will be optional, and clusters without it will behave as they do today.

## Rejected alternatives:
External Script or Init Container
- A user-managed init container or script could run kafka-features.sh at broker startup.
- This would be configured via a ConfigMap, environment variables, or a mounted script.
