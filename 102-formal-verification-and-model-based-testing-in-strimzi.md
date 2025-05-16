# Formal Verification and Model-Based Testing in Strimzi

**This proposal introduces formal verification and [model-based testing](#glossary) ([MBT](https://www.qt.io/quality-assurance/blog/understanding-model-based-testing)) into the Strimzi.** 
It aims to apply these techniques not only to the operators (e.g., User Operator, Topic Operator) but also to critical components of the Cluster Operator, KafkaRoller, CruiseControl integration parts, and other core subsystems.
The goal is to incrementally model the behavior of these components using formal specifications and use them to automatically generate high-coverage test traces that validate [invariants](#glossary), [safety properties](#glossary), and [temporal properties](#glossary) under real-world conditions.

To illustrate the approach, the **User Operator was selected as a proof of concept (PoC)**. 
Its behavior was formally specified, and [test traces](#glossary) were automatically generated from the model and replayed against the real implementation. 
This PoC demonstrates how MBT can validate [invariants](#glossary), and systematically explore a wide range of system behaviors.

## Current situation

Strimzi currently relies on a combination of unit, integration, and end-to-end tests, all of which are manually written and maintained. 
While this test suite effectively covers many expected use cases, it has inherent limitations when it comes to validating complex, stateful, and distributed behaviors.
Specifically, the current approach does not adequately support:
1. Exploration of nondeterministic interleaving
2. Simulation of transient Kubernetes faults
3. Exhaustive verification of safety (e.g., users never transition to invalid states.) and liveness (e.g., secrets are eventually created) properties
4. Consistency across reconciliation cycles

Moreover, there is currently no formal specification of the expected behavior for Strimzi components. 
This means that correctness properties such as invariants or [temporal properties](#glossary) are not explicitly defined, verified, or enforced — they are only implied through the existing implementation and test code.

## Motivation

Strimzi currently uses a combination of unit, integration, and system tests to validate operator behavior. 
These tests are manually written and typically cover happy paths, selected failure scenarios, and some regression cases. 
While this approach is well-established, it has limitations when validating complex, stateful components that operate under concurrency, failover, and eventual consistency conditions.

This proposal introduces a new testing methodology based on formal specifications and model-based testing (MBT), which allows us to:
1. Formally define expected system behavior using executable models
2. Automatically generate test traces (e.g., sequences of user creation, updates, deletions, etc.)
3. Replay those traces against real Strimzi operators (e.g., User Operator) to validate behavior
4. Dynamically check invariants (e.g., “all Ready users must have valid secrets”) and [temporal properties](#glossary) (e.g., “secrets are eventually created after user creation”)

The model serves both as test input and as a precise, readable form of documentation. 
> [!NOTE]
> In practice, it is often easier to understand than the code itself, especially when onboarding contributors or maintaining older subsystems. 

Because the same model is used to generate test traces, it remains automatically synchronized with the implementation, ensuring documentation is never stale.

MBT is not intended to replace existing test types — rather, it complements unit and integration testing by systematically exploring combinations of operations and system states that are difficult to cover manually.
An initial proof of concept has been completed for the `User Operator` to showcase its benefits. 

## Proposal

This proposal introduces formal specification and model-based testing (MBT) as a complementary testing strategy within the Strimzi project. 
The central idea is to model the behavior of Strimzi components as executable specifications and use these models to automatically generate test traces that are then replayed against real operator implementations.

The approach has already been validated through a working proof of concept for the `User Operator`. 
This PoC includes a complete formal model in [Quint](https://github.com/informalsystems/quint) (for those who know TLA+, Quint is transpiler to TLA+ but with much more programming syntax).
It includes a test harness for replaying model-generated traces and a suite of runtime invariant checks — a technique known as runtime verification in the formal methods domain.
The proposal now aims to build on that foundation, integrating the methodology into the Strimzi test suite and expanding its use to additional components.

To ensure clarity, reproducibility, and maintainability, the PoC follows a consistent layout and integration strategy within the existing test infrastructure:
- **Model placement**: Quint specifications are stored under `src/test/resources/specification/`, organized by component (e.g., `UserOperatorModel.qnt`, `TopicOperatorModel.qnt`).
- **Test traces**: MBT traces are committed alongside the model under `specification/traces/`. 
Moreover, we would pre-generated our test traces to have each test run of MBT same and reproducible. 
It wouldn't make sense to generate new test traces each build. 
In my observations this approach achieved 81% coverage by running 500 steps in the whole state space.
This number of steps could vary for different models for sure but in case of our current model (i.e. UserOperator) 500 steps are enough.
Furthermore, we generate multiple traces — the best results were achieved using 10 distinct traces, each consisting of 500 steps. 
These traces vary in configuration, such as toggling the ACL operator (enabled or disabled), and exercising other nondeterministic options.
- **Test classes**: MBT tests are placed alongside other integration tests in `src/test/java/...`, using the naming convention `*MbtIT.java` to clearly distinguish them from unit and system tests.

One can find the model and PoC implementation here:
- PoC of Formal Specification for User Operator in Quint: [UserOperatorModel.qnt](https://github.com/see-quick/verification/blob/main/formal_verification/quint/strimzi/user-operator/UserOperatorModel.qnt).
- Model-Based Testing Implementation PoC: [UserOperatorModelMbtIT.java](https://github.com/strimzi/strimzi-kafka-operator/compare/main...see-quick:strimzi-kafka-operator:model-based-testing-user-operator-with-specification)
--- 

### Additional Benefits

#### Interactive Exploration via Simulation

A key advantage of formal specifications is that they are executable, meaning contributors can interactively explore how the system behaves under different scenarios, without running any real cluster.

For example, using the Quint CLI:
```bash
quint run UserOperatorModel.qnt --invariant=AllInvariantsHold --mbt --max-steps 30 --verbosity 2
```
...runs a model-based simulation for 30 steps, starting from the empty system state. 
It prints a trace of actions, non-deterministic decisions (e.g., authentication type, user name, quotas enabled), and evolving system state (e.g., users, secrets, annotations).
This gives a transparent view of how the system evolves step-by-step (of course there are a lot of abstracted from the real app):
For instance initial state may look like:
```
[State 0]
{
  globalState: {
    eventQueue: [],
    processedEvents: 0,
    secrets: Set(),
    users: Map()
  },
  mbt::actionTaken: "init",
  parameters: {
    aclsEnabled: true,
    potentialUsers: Set("Alice", "Bob", "Carol"),
    ...
  }
}
```
and final state:
```
[State 30]
{
  globalState: {
    secrets: Set("Alice", "Bob", "Carol"),
    users: Map(
      "Bob" -> {
        metadata: { name: "Bob" },
        spec: { authentication: TlsAuthentication({ enabled: true }), ... },
        status: { conditions: { status: "Ready" } }
      },
      // Alice, Carol...
      ...
    ),
    ...
  },
  mbt::actionTaken: "updateUser",
  mbt::nondetPicks: {
    u: Some("Bob"),
    authType: Some("tls"),
    ...
  }
}
```

This is extremely valuable for developers, as it provides a clear picture of how the system state evolves over time. 
It naturally leads to insightful questions during simulation, such as:
1. What happens when we create a KafkaUser with empty authentication?
2. What if we create a KafkaUser with empty authentication and quotas?
3. What happens if we:
    1.	Create a KafkaUser with empty authentication (no secret generated?)
    2.	Update it to use TLS or SCRAM-SHA authentication (is the secret created?)
    3.	Then delete that KafkaUser (is the secret cleaned up?)
4. What if we define a KafkaUser with TLS authentication, and later update the same CR to use SCRAM-SHA?

These kinds of what-if explorations are seamlessly supported by Quint’s executable model, giving developers immediate feedback on complex state transitions and edge cases — long before they ever write a line of real code.

---

#### Measurable Coverage Improvements

While code coverage is not the main goal of MBT, it provides a useful baseline for comparison.
Here are some numbers observed in the User Operator PoC:

| **Metric**      | **MBT PoC** | **Without MBT** |
|-----------------|-------------|-----------------|
| Line Coverage   | **89%**     | 83%             |
| Branch Coverage | **78%**     | 71%             |

Even when run **independently** (without combining with the rest of the test suite), the MBT suite alone achieves:
- ~80% line coverage
- ~69% branch coverage
and all achieved from a single test class replaying traces generated from the model.

More importantly, the MBT approach unlocks the ability to detect edge cases and subtle race conditions that are difficult to catch through conventional test methods — particularly in components like the Topic Operator or Cluster Operator subsystems (e.g., KafkaRoller), where the reconciliation logic and state transitions are far more complex.

#### Profiling

One of the less obvious but powerful advantages of this approach is **profiling**.
Because MBT systematically exercises nearly the entire lifecycle of the component — including creation, update, deletion, and failure scenarios — it essentially mirrors the runtime behavior of the operator and its subcomponents (e.g., AclOperator, QuotasOperator, etc.).
This makes it ideal for profiling under realistic workloads and observing performance bottlenecks that may not surface under traditional happy-path testing.

Additionally, MBT makes it easy to simulate high-throughput, real-world-like scenarios using deterministic traces — for example, submitting 1000+ operations within seconds and observing system behavior under load.
Thanks to the flexibility of the Quint modeling language, we can customize the number of traces, as well as bound the number of steps per trace, allowing precise control over test complexity and duration.
These traces can then be replayed deterministically in Java, enabling repeatable simulations and debugging workflows under well-defined scenarios.


## Affected / Not Affected Projects

Currently, the proof of concept (PoC) targets only the **User Operator**, so it is the only component directly affected at this stage.
However, the broader goal is to extend model-based testing and formal verification to other critical components within the `strimzi-kafka-operator`.

## Compatibility

This proposal introduces formal specification and model-based testing (MBT) purely as an internal quality assurance and validation mechanism.
There are **no changes to public APIs**, CRDs or any user-facing behaviour.
Therefore, this enhancement is **fully backward-compatible**

## Rejected alternatives

1. Relying only on unit and integration tests 
2. Manual specification in documentation

## Disadvantages and Maintainability

While this proposal brings significant benefits, there are trade-offs to consider:
- **Learning curve**: 
Contributors will need to learn `Quint` and basic formal modeling concepts. 
While the models are readable and well-documented, this still requires onboarding effort.
- **Not a full replacement**: 
MBT does not eliminate the need for unit/integration tests. 
Instead, it complements them. 
Contributors will still need to write focused unit tests for logic not expressible in models.
- **Model upkeep**:
As the system evolves, models must be updated. 
This proposal assumes the model will be maintained by contributors familiar with Quint (primarily by me, initially).

## Future Work

If this proposal proves successful  for the `User Operator`, follow-up efforts may include:
- Modeling the **Topic Operator** to ensure correct topic reconciliation and eventual consistency.
- Formalizing the **KafkaRoller** logic to prevent race conditions during rolling upgrades.
- and many parts within **Cluster Operator**

# Conclusion

Formal verification and MBT are increasingly being adopted by industry leaders. 
For example, Amazon Web Services uses TLA+ specifications to verify the correctness of foundational components like S3 and DynamoDB. 
Microsoft and Intel have applied formal methods to critical distributed systems and hardware validation. By adopting formal specification and MBT, Strimzi can move toward a new level of confidence, rigor, and coverage — especially in areas where traditional tests fall short. 
This proposal lays a strong foundation for a scalable, maintainable, and verifiable testing strategy across the entire operator stack.

## Glossary

- **Formal Specification** - A mathematically precise description of system behavior. In this context, it defines the allowed operations, states, and transitions of Strimzi components using a modeling language (e.g., [Quint](https://github.com/informalsystems/quint)).
- **Model-Based Testing (MBT)** - A testing technique where test cases are automatically derived from a formal model of the system. MBT explores different event sequences and system states to check that the implementation conforms to the model.
- **Test Trace** - A sequence of actions (e.g., create user, update quotas, delete user) generated from the model, used as input for running tests against the real system.
- **Invariant** - A property that must always hold true during system execution, regardless of the sequence of events. Example: “All users marked as Ready must have an associated Secret." (f.e., this invariant would be violated, if we create a plain user without any authentification)
- **Temporal Property** - A rule about the order or timing of events in the system. Example: “If a user is created, a secret must eventually be created.” These are often expressed using temporal logic.
- **Safety Property** - A type of invariant asserting that “nothing bad ever happens.” Example: “A user should never be marked Ready without valid credentials.”
- **Liveness Property** - A property asserting that “something good eventually happens.” Example: “Every reconciliation eventually results in a consistent state.”
- **Executable Model** - A formal specification that can be used to simulate behavior, check properties, and generate test traces (in this case it's our test driver).