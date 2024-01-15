# Pod Disruption Budget Generation Environment Variable Proposal

## Background and Problem Statement

The Strimzi Kafka operator currently lacks the option to disable the creation of Pod Disruption Budgets (PDBs). In certain infrastructures, PDB creations for users are denied, resulting in operational challenges. Providing an option to disable PDB creation in Strimzi would address these constraints.

## Proposed Solution

Introduce a new environment variable in Strimzi, [similar to the one for Network Policies](https://github.com/strimzi/proposals/blob/main/028-network-policy-generation-environment-variable.md), that allows global disabling of PDB creation. This proposal suggests the environment variable `STRIMZI_POD_DISRUPTION_BUDGET_GENERATION`, which by default is `true`. When set to `false`, the Strimzi operator will not generate Pod Disruption Budgets.

## Rationale

- **Flexibility:** Some users operate in environments with strict policies that prevent the creation of PDBs. Providing an option to disable PDB generation caters to these scenarios.
- **Consistency:** This approach is consistent with existing features, such as the Network Policy Generation Environment Variable.
- **Simplicity:** A global setting avoids the complexity of configuring this setting for each custom resource.

## Current Implementation

The Strimzi operator automatically creates PDBs to ensure high availability and minimize disruptions. However, this feature does not accommodate environments where PDB creation is restricted.

## Motivation

- **Operational Constraints:** In environments where PDB creation is not permitted, users are unable to deploy Strimzi effectively.
- **Configuration Flexibility:** Users need the ability to configure their Strimzi deployments to align with their specific operational policies and constraints.

## Proposal Details

1. **Environment Variable Introduction:** Implement `STRIMZI_POD_DISRUPTION_BUDGET_GENERATION` with a default value of `true`.
2. **Disabling PDB Generation:** When set to `false`, the operator will skip PDB operations for all Strimzi components. It will not create, modify or delete any PDB.
3. **Documentation and Guidance:** Update Strimzi documentation to include instructions and implications of disabling PDB generation.
4. **Scope:** This change affects only the Pod Disruption Budget aspect and does not alter any other functionalities of the Strimzi operator.

## Affected Projects

This proposal pertains solely to the [Strimzi Kafka Operator](https://github.com/strimzi/strimzi-kafka-operator).

## Compatibility

- **Default Behavior:** By retaining `true` as the default value, existing deployments remain unaffected.
- **Backward Compatibility:** This feature is an addition and does not alter existing functionalities.

## Rejected Alternatives

1. **CRD-based Option for PDB Disabling:** Initially considered, this was rejected for increasing complexity and requiring repetitive configuration.
2. **Global Disabling via Command-Line Flag:** Although an alternative, a command-line flag was deemed less flexible and consistent compared to an environment variable.

## Conclusion

Introducing an environment variable to globally control PDB generation in Strimzi provides the necessary flexibility for users operating in environments with strict PDB creation policies, while maintaining the ease of use and consistency with existing Strimzi features.
