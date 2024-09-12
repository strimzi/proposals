# OpenFeature Integration to Strimzi

Proposal to integrate OpenFeature into Strimzi to enhance feature management capabilities.
The integration of OpenFeature across all operators will be streamlined, thanks to the enhancements proposed in [#74](https://github.com/strimzi/proposals/pull/118).

## Current Situation
Strimzi, currently leverages environment variables for feature management. While this approach is effective, it requires rolling updates for any changes, which can lead to service downtime. 
Moreover, it lacks the flexibility offered by modern feature flag systems, which can provide dynamic control with zero downtime when toggling features.

## Motivation

To provide Strimzi users with dynamic control over features without needing service restarts, enhancing the efficiency and adaptability of their deployments.

## Proposal

This proposal suggests the integration of OpenFeature, providing two primary methods for feature management:

For the current users (i.e., having classic ENV variable STRIMZI_FEATURE_GATES), we will use [Enviroment Variable Provider](https://github.com/open-feature/java-sdk-contrib/tree/main/providers/env-var)
already implemented by OpenFeature folks. Some of the common characteristics of such provider are:
1. Default method currently used. 
2. Feature gates are set via environment variables.
3. Requires rolling updates for changes.

The third one is the most important for us because if we remove it, we will benefit it from in testing and also reducing 
overall time during changes features, which user might want to include in their infrastructure.

On the other side for the more complex users with need of flagging system, we would support of [FlagD](https://github.com/open-feature/flagd),
which is pretty known because [OpenFeature is CNCF Incubating project](https://www.cncf.io/projects/openfeature/).
Common characteristics of this provider are:
1. New method proposed for more complex users, which needs better management over multiple features (of Strimzi)
2. Integrates with FlagD for dynamic feature flagging
3. Allows changes without the need for rolling updates.

### Implementation Steps

Currently, there are a few classes where we use `FeatureGates` related stuff (i.e., FeatureGates abstraction).
Those classes are:
1. ClusterOperatorConfig 
2. EntityTopicOperator 
3. EntityUserOperator 
4. AbstractConnectOperator
5. KafkaReconciler
6. ZooKeeperReconciler

In each of these classes, we essentially do the following:

```java
result.featureGatesEnvVarValue = config.featureGates().toEnvironmentVariable()
```

Here, we parse the currently configured `FeatureGates` from the `STRIMZI_FEATURE_GATES` environment variable and assign it to specific classes (e.g., the `EntityTopicOperator` class).
Furthermore, if we dive deeper into the implementation of the `UserOperator` class, we retrieve the feature gates as follows:
```java
/**
 * @return Feature gates configuration
 */
public FeatureGates featureGates() {
    return get(FEATURE_GATES);
}
```
These are fetched directly from `ConfigParameter`, which encapsulates a map implementation with additional utility methods (e.g., a parser to convert the string representation into a specified type).
In this case, the `FeatureGates` type is defined as:
```java
 /**
 * Configuration string with feature gates settings
 */
public static final ConfigParameter<FeatureGates> FEATURE_GATES = new ConfigParameter<>("STRIMZI_FEATURE_GATES", parseFeatureGates(), "", CONFIG_VALUES);
```
Here, the `parseFeatureGates()` method is defined, which calls the constructor of the `FeatureGates` class.

This is the basic flow of how it currently works, and my proposal is to modify the `FeatureGates` class to use `EnvVarProvider`, which would cover all aspects of the current implementation. 
For this, we would need to add a few dependencies (e.g., `dev.openfeature.sdk` and `dev.openfeature.contrib.providers.env-var`).
Additionally, we can easily extend support by adding other providers like `FlagDProvider` for users who want to use a feature flagging system (i.e., FlagD with its `OpenFeature` Operator). 

The overall setup would look like this:

```java
// ...
// imports omitted for brevity
// ...

/**
 * Class for handling the configuration of feature gates
 */
public class FeatureGates {

    private static final String CONTINUE_ON_MANUAL_RU_FAILURE = "ContinueReconciliationOnManualRollingUpdateFailure";
    private static final String FLAGD_ENABLED_ENV_VAR = "FLAGD_ENABLED"; // Environment variable to toggle FlagD

    private final Client featureClient;
    private final FeatureProvider provider;

    // When adding new feature gates, do not forget to add them to allFeatureGates(), toString(), equals(), and `hashCode() methods
    private FeatureGate continueOnManualRUFailure;

    /**
     * Constructs the feature gates configuration.
     *
     * @param featureGateConfig String with a comma-separated list of enabled or disabled feature gates
     */
    public FeatureGates(String featureGateConfig) {
        // Set the appropriate provider based on the environment variable
        this.provider = isFlagDEnabled() ? new FlagdProvider() : new EnvVarProvider();
        OpenFeatureAPI.getInstance().setProvider(this.provider);
        this.featureClient = OpenFeatureAPI.getInstance().getClient();

        // Validate and parse the featureGateConfig if it's provided
        if (featureGateConfig != null && !featureGateConfig.trim().isEmpty()) {
            List<String> featureGates;

            // Validate the format of the feature gate configuration string
            if (featureGateConfig.matches("(\\s*[+-][a-zA-Z0-9]+\\s*,)*\\s*[+-][a-zA-Z0-9]+\\s*")) {
                featureGates = asList(featureGateConfig.trim().split("\\s*,+\\s*"));
            } else {
                throw new InvalidConfigurationException(featureGateConfig + " is not a valid feature gate configuration");
            }

            // Validate each feature gate in the config to ensure it is recognized
            for (String featureGate : featureGates) {
                featureGate = featureGate.substring(1); // Remove the + or - sign

                // Only validate feature gates but do not apply them manually
                switch (featureGate) {
                    case CONTINUE_ON_MANUAL_RU_FAILURE:
                        // This is a valid feature gate; continue with processing
                        break;
                    default:
                        throw new InvalidConfigurationException("Unknown feature gate " + featureGate + " found in the configuration");
                }
            }
        }

        // Fetch feature gates using OpenFeature
        boolean continueOnManualRUFailureValue = fetchFeatureFlag(CONTINUE_ON_MANUAL_RU_FAILURE, true, Boolean.class);
        setValueOnlyOnce(continueOnManualRUFailure, continueOnManualRUFailureValue);

        // Validate interdependencies (if any)
        validateInterDependencies();
    }

    /**
     * Checks whether FlagD is enabled via environment variables.
     *
     * @return True if FLAGD_ENABLED is set to "true", otherwise false.
     */
    private boolean isFlagDEnabled() {
        String flagDEnabled = System.getenv(FLAGD_ENABLED_ENV_VAR);
        return flagDEnabled != null && flagDEnabled.equalsIgnoreCase("true");
    }

    /**
     * Fetches the feature flag using OpenFeature and applies a default value if not present.
     *
     * @param flagName     The name of the feature flag
     * @param defaultValue The default value if the flag isn't set
     * @param <T>          The type of the feature flag (Boolean, String, Integer, etc.)
     * @param returnType   The class of the return type for determining which get method to call
     * @return The value of the feature flag
     */
    private <T> T fetchFeatureFlag(String flagName, T defaultValue, Class<T> returnType) {
        try {
            // Handle different types based on returnType
            if (returnType == Boolean.class) {
                return returnType.cast(featureClient.getBooleanValue(flagName, (Boolean) defaultValue));
            } else if (returnType == String.class) {
                return returnType.cast(featureClient.getStringValue(flagName, (String) defaultValue));
            } else if (returnType == Integer.class) {
                return returnType.cast(featureClient.getIntegerValue(flagName, (Integer) defaultValue));
            } else if (returnType == Double.class) {
                return returnType.cast(featureClient.getDoubleValue(flagName, (Double) defaultValue));
            } else {
                throw new IllegalArgumentException("Unsupported feature flag type: " + returnType.getSimpleName());
            }
        } catch (Exception e) {
            // Fallback in case of any issues fetching the flag
            return defaultValue;
        }
    }

    /**
     * Fetches and updates the feature gates state dynamically from the OpenFeature API.
     */
    public void updateFeatureGateStates() {
        if (isFlagDEnabled()) {
            // Fetch dynamically from FlagD and update internal states
            this.continueOnManualRUFailure.setValue(fetchFeatureFlag(CONTINUE_ON_MANUAL_RU_FAILURE, true, Boolean.class));
        } else {
            this.continueOnManualRUFailure.setValue(continueOnManualRUFailureEnabled());
            // Fallback to static configuration if FlagD is not enabled
        }
    }

    // other methods not mentioned for brevity

    /**
     * Feature gate class represents individual feature fate
     */
    static class FeatureGate {
        private final String name;
        private final boolean defaultValue;
        private Boolean value = null;

        /**
         * Feature fate constructor
         *
         * @param name          Name of the feature gate
         * @param defaultValue  Default value of the feature gate
         */
        FeatureGate(String name, boolean defaultValue) {
            this.name = name;
            this.defaultValue = defaultValue;
        }

        // other methods not mentioned for brevity
    }
}
```
In the context of `EnvVarProvider`, it should be fairly simple to implement. 
By using the new provider and adapting the current implementation, it should function the same as our existing approach.
Alternatively, when using `FlagDProvider`, there are a couple of options: (i) an user can deploy just standalone application FlagD as a deployment, 
or (ii) deploy the [OpenFeature Operator](https://github.com/open-feature/open-feature-operator), which supports FlagD as one of its flagging systems. 
While researching, I also discovered several other feature flagging systems, such as:
1. [Go Feature Flag](https://gofeatureflag.org/)
2. [CloudBees Feature Management](https://www.cloudbees.com/capabilities/feature-management)
3. [Split](https://www.split.io/)
4. [Harness](https://harness.io/products/feature-flags)
5. [LaunchDarkly](https://launchdarkly.com/)
6. [Flagsmith](https://flagsmith.com/)
7. [Flipt](https://www.flipt.io/)

In my experience, `OpenFeature` is the best fit for our needs, given its community, support, and its inclusion in the CNCF ecosystem.
Conceptual design of the communication could be illustrated like this:

    +-------------------------------+      +--------------------------------+
    |        OpenFeature Operator    |     |             Strimzi            |
    |   (Manages Feature Flags/FlagD)|     | (Manages Kafka Cluster in K8s) |
    +-------------------------------+      +--------------------------------+
                  |                                        |
                  |                                        |
         +----------------+                      +---------------------------+
         | FlagD Server   |                      | Cluster Operator, UO, TO  |
         | Centralized    |   <-- API Calls -->  |  (No sidecar needed)      |
         | Feature Flags  |                      +---------------------------+
         +----------------+

Where in each component (i.e., ClusterOperator, UserOperator and TopicOperator), we will
fetch feature flags dynamically from the OpenFeature API, which is managed by OpenFeature Operator.
Each operator’s logic that is controlled and **centralized** by FeatureGates class (e.g., enabling new behaviors, managing rolling updates) 
will dynamically receive flag updates from `FlagD` via the OpenFeature Operator.

Example of the feature flags 
This is configuration example of the current `STRIMZI_FEATURE_GATES`
```yaml
# Flags for our backend application
apiVersion: core.openfeature.dev/v1beta1
kind: FeatureFlag
metadata:
  name: strimzi-feature-gates
  labels:
    app: strimzi-feature-gates
spec:
  flagSpec:
    flags:
      feature-gate-a:
        variants:
          'on': true
          'off': false
        defaultVariant: 'off'
      feature-gate-b:
        variants:
          'on': true
          'off': false
        defaultVariant: 'on'
        state: ENABLED
#        ... and more
```

Moreover, if we want to have different `FEATURE_GATES` in components (such as `UserOperator`) one would need to configure it 
```yaml
# Flags for TopicOperator
apiVersion: openfeature.dev/v1alpha1
kind: FeatureFlag
metadata:
  name: topic-operator-feature-flags
  labels:
    app: topic-operator
spec:
  flagsSpec:
      flags:
        feature-gate-c:
          variants:
            'on': true
            'off': false
          defaultVariant: 'off'
          state: ENABLED 
 #        ... other flags for TopicOperator
```
and for `UserOperator`
```yaml
# Flags for UserOperator
apiVersion: openfeature.dev/v1alpha1
kind: FeatureFlag
metadata:
  name: user-operator-feature-flags
  labels:
    app: user-operator
spec:
  flagsSpec:
      flags:
        feature-gate-d:
          variants:
            'on': true
            'off': false
          defaultVariant: 'on'
          state: ENABLED
 #        ... other flags for UserOperator
```
and then we would need to implement in reconcile loop of each component call for OpenFeature API using its client.
For `UserOperator` that's `UserControllerLoop` class.
```java
// UserControllerLoop.java content 
class UserControllerLoop {

    private final FeatureGates featureGates;  // Add this to handle feature gates
    // ... other attributes not mentioned for brevity


    /**
     * The main reconciliation logic which handles the reconciliations.
     *
     * @param reconciliation    Reconciliation identifier used for logging
     */
    @Override
    protected void reconcile(Reconciliation reconciliation) {
        LOGGER.infoCr(reconciliation, "{} will be reconciled", reconciliation.kind());

        //  update the state of feature gates dynamically from FlagD
        featureGates.updateFeatureGateStates();
        LOGGER.infoCr(reconciliation, "Fetching from FlagD: continueOnManualRUFailureEnabled is enabled: {}", featureGates.continueOnManualRUFailureEnabled());

        KafkaUser user = userLister.namespace(reconciliation.namespace()).get(reconciliation.name());

// ...
```
And `maybeUpdateFeatureGateA()` would change the state of inner FeatureGate instance for each component.
Meaning that for TopicOperator we will have different `FEATURE_GATES` as for `UserOperator` if necessary.
```java
/**
 * Fetches and updates the feature gates state dynamically from the OpenFeature API.
 */
public void maybeUpdateFeatureGateA() {
    if (isFlagDEnabled()) {
        // Fetch dynamically from FlagD and update internal states
        this.continueOnManualRUFailure.setValue(fetchFeatureFlag(CONTINUE_ON_MANUAL_RU_FAILURE, true, Boolean.class));
    } else {
        this.continueOnManualRUFailure.setValue(continueOnManualRUFailureEnabled());
        // Fallback to static configuration if FlagD is not enabled
    }
}
```
After such update we can easily access those updated values by simply calling:
```java
if (featureGates.continueOnManualRUFailureEnabled()) {
    // ... and do some logic...
}
```
and it would be accessible from `UserControllerLoop` class with form of getter. 
For now we do not have any `FEATURE_GATES` for `UserOperator` so there will be no such logic needed
but maybe in the future we can simply add methods for each `FEATURE_GATE`; meaning for `UserOperator` we will have
`featureGateA`, `featureGateB`, TopicOperator will have `featureGateB`, `featureGateC` and `ClusterOperator` has `featureGateD`
and in each of their reconciles loop we would simply call just:
```java
// UserOperator reconcile() loop
reconcile() {
    // ...
    maybeUpdateFeatureGateA();
    maybeUpdateFeatureGateB();
    // ...
}

// TopicOperator reconcile() loop
reconcile() {
    // ...
    maybeUpdateFeatureGateB();
    maybeUpdateFeatureGateD();
    // ...
}

// ClusterOperator reconcile() loop
reconcile() {
    // ...
    maybeUpdateFeatureGateD();
    // ...
}
```
Table showing feature gates support for each component for clarity.

| Operator            | Feature Gates  |
|---------------------|----------------|
| **UserOperator**    | `featureGateA` |
|                     | `featureGateB` |
| **TopicOperator**   | `featureGateB` |
|                     | `featureGateC` |
| **ClusterOperator** | `featureGateD` |

And that way we can follow the pattern for other components such as `Topic Operator` and `ClusterOperator`.

### Potential configuration of Feature Gates per Kafka cluster

With `OpenFeature` there is a possibility to define feature gates specific to each Kafka cluster by using the cluster name as part of the metadata or by associating flags with specific clusters. 
This allows you to customize feature gates per cluster within the `ClusterOperator`.
To design feature gates based on the Kafka cluster name for the `ClusterOperator`, we can extend the configuration to include cluster-specific feature gates. Here’s a potential design:

#### Example YAML Configuration for Cluster-specific Feature Gates

```yaml
# Feature gates for Kafka clusters
apiVersion: core.openfeature.dev/v1beta1
kind: FeatureFlag
metadata:
  name: kafka-cluster-feature-flags
  labels:
    app: kafka-cluster
spec:
  flagSpec:
    flags:
      kafka-cluster-a-feature-gate-a:
        variants:
          'on': true
          'off': false
        defaultVariant: 'on'
        state: ENABLED
      kafka-cluster-b-feature-gate-b:
        variants:
          'on': true
          'off': false
        defaultVariant: 'off'
        state: ENABLED
    # Additional cluster-specific feature gates can be added here
```

### Benefits

- **Flexibility:** Users can toggle features without redeploying or restarting services.
- **Faster Iterations/Testing:** Features can be tested and rolled out quickly, speeding up development cycles.
- **Centralized Management:** FlagD integration allows centralized control of feature flags, simplifying management across multiple components.
- **Scalability:** The approach scales efficiently for larger deployments without adding operational complexity.
- **Backwards Compatibility:** The proposal maintains support for the existing `STRIMZI_FEATURE_GATES` method, ensuring a smooth transition.

### Potential Challenges

- **Complexity:** Increased complexity in configuration management.
- **Dependency:** Additional dependency on the FlagD service.

## Affected/Not Affected Projects

`Cluster Operator`, `Topic Operator` and `User Operator`; meaning the modification will be done in scope of `strimzi-cluster-operator` project.

## Questions

1. What if an user configure `STRIMZI_FEATURE_GATES` as environment variable and also configure flagging system?

Flagging system has priority and if flagging system is used then `STRIMZI_FEATURE_GATES` should be ignored.

2. What if an user configured flagging system and then move on classic `STRIMZI_FEATURE_GATES` env-var?

If we follow proposal design, then it would simply fetch value, which is set from `STRIMZI_FEATURE_GATES` environment variable (expected).

## Compatibility

The introduction of OpenFeature is backwards compatible, designed to enhance, not replace, current configurations.

## Rejected Alternatives

- **Single Provider Approach:** Initially considered using only FlagD, but rejected to maintain flexibility for users accustomed to the current environment variable method.

This proposal aims to modernize Strimzi's feature management, providing a bridge to more dynamic configuration methods while respecting traditional deployment practices.
