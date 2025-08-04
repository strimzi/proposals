Declarative Management of Kafka Preview Features in Strimzi
This proposal suggests enabling declarative configuration and reconciliation of Kafka preview features through the Kafka custom resource (CR) in Strimzi.

Current situation
Kafka preview features (such as kafka.shared.consumer) need to be manually enabled using the kafka-features.sh CLI or the Kafka Admin API. This works in static clusters but presents issues in dynamic, ephemeral, or GitOps-managed environments. There is currently no way to express these features declaratively within the Kafka CR.

Without declarative support, the following issues arise:

Feature drift from manual changes

Loss of settings in ephemeral environments

Inability to track changes through Git

Motivation
Preview features in Kafka can enable teams to test cutting-edge functionality. However, without a way to configure them declaratively:

GitOps workflows are broken

Teams cannot easily maintain consistent environments across dev/staging/prod

Manual intervention is needed after each deployment or recreation of the cluster

By supporting these features through the Kafka CR, Strimzi can offer a seamless and maintainable experience for all cluster types.

Proposal
I came up with two proposals to address the problem:

Method 1: Native Support in Kafka CR
Extend the Kafka CRD to support a new previewFeatures section:

spec:
  kafka:
    version: 3.7.0
    previewFeatures:
      - name: kafka.shared.consumer
        enabled: true
        version: 1
Operator Behavior:
On reconciliation, Strimzi checks current feature states using the Kafka Admin API.

If a mismatch is found, it applies the desired feature version using the Admin API or kafka-features.sh.

Benefits:
Declarative and GitOps-friendly

Drift protection via automatic reconciliation

Consistent environments

Considerations:
Requires changes to the Kafka CRD and Operator logic

New validation rules needed for preview feature schema

Method 2: External Script or Init Container
Allow users to manage preview features manually through a user-supplied container or script.

Design:
A custom init container or sidecar runs kafka-features.sh at broker startup.

Configurable via ConfigMap, environment variables, or mounted script.

containers:
  - name: kafka-feature-initializer
    image: custom/kafka-feature-init:latest
    volumeMounts:
      - name: config
        mountPath: /config
    env:
      - name: FEATURES
        value: "kafka.shared.consumer:1:true"
Benefits:
Simple implementation with no changes to Strimzi codebase

Good for quick prototyping or custom environments

Limitations:
Not declarative via CR

No automatic reconciliation or drift protection

Error-prone due to lack of CR validation

Affected projects
strimzi-kafka-operator (Kafka CRD schema and reconciliation logic)

strimzi-images for an official feature initializer container (for Method 2)

Compatibility
Method 1 introduces an optional field in the Kafka CR, so it is backward-compatible.
Method 2 is fully external and requires no changes to Strimzi.

Rejected alternatives
Manual Configuration Only
Continuing with only manual setup is fragile and error-prone, particularly in dynamic or production environments.

Feature Management via Annotations
Annotations are not structured enough to support versioned feature configurations.


Conclusion
 I recommend moving forward with Method 1 to enable via the Kafka CR, while optionally supporting Method 2 for short-term flexibility.
