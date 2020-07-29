# Design Documentation Proposal

This document is intended to progress discussion about how to write and maintain [design documentation](https://en.wikipedia.org/wiki/Software_design_description) for Strimzi.

The main goal of having design documents for Strimzi is so that behaviours of Strimzi as a whole and each of the components are well defined and are agreed upon.

The benefits of adding design documentation to Strimzi are:
- Design docs/additions could help when there is conflict about the direction of a proposal/code change.
- It's hard for new contributors to get into the project and understand behaviours of certain components, this may be fixed by extra technical documentation.
- If just a couple of named contributors moved on to other things, the velocity of the project would drop considerably. Design doc would help in this case too, capturing conventions and behaviour enforcements that normally only get picked up and tweaked at review time.


## Where is Strimzi today

Currently there are no design documents for Strimzi, the closest thing Strimzi has would probably be the generated docs for classes - these are more user oriented than design oriented. Some classes also have extensive documentation comments found at the top of their files explaining their usage and design, these are useful, but a more higher level summary of how all the classes fit into the 'big picture' of the component(s) would be useful for contributors.


## Proposal

### When to write a design document

A design document should be written so that the high-level purpose and function of a component or mechanism is clear.

For instance a design document should exist for individual components of Strimzi.
i.e. Kafka, Zookeeper, Cruise Control - rationalizing their inclusion within Strimzi, what they achieve as part of a larger whole.
Components such as the strimzi `cluster-operator` would have a high level overview something along the lines of:

```
The Strimzi cluster-operator is a component that manages user specified components in the Kubernetes cluster. It achieves this by reading specifications from custom resource objects.... (Continued)

The cluster-operator is split into multiple operators that are managed by [Vertx](link), these operators are:
- [Kafka Operator](../kafka-operator.md)
- [KafkaUser Operator](../kafka-user-operator.md)
(Continued)
```

The above is just an example but I think it demonstrates the value of a deeper context for how the Strimzi project is written but without describing implementation details. Moreover this should be fairly simple to maintain as the overall architecture and flow doesn't change often and should need little more than a single sentence change when they do.


### Where to write a design document

A design document should be a markdown file located in the `design` directory of the repository the component code is located in, for instance all design documents relating to the strimzi operator(s) should be located in the `strimzi-kafka-operator` repository under the `design` directory.

#### Strimzi Design Document Organization
Design documents should be split into logical areas, for repositories that aren't `strimzi-kafka-operator` I suspect this would start as a single or just a few succinct documents.
For `strimzi-kafka-operator` I would suggest splitting design documentation from a functionality perspective, that is to say all documents relating to the `Kafka` reconcile and custom resource would be separated into its own folder called `kafka.md` or `kafka-operator.md`, similarly for `KafkaConnect` etc.
For design documentation that would cover the whole operator and not just any single custom resource I would suggest a file named `operator.md` this would describe behaviours not unique to any one specific reconcile loop or verticle and appropriate links to other files containing specific (but still high-level) differences between reconciles could be described there. 


### How to write a design document

The design documents would live in the related repository in the Strimzi Organization and would match the current code state. Naturally it is hard to align the design documents and the code behaviour, but it would be the responsibility of a reviewer to identify if any of the changes are impactful enough to potentially affect the design docs. Due to the below described simplicity of the design docs, if they are found to be out of date, it should be fairly simply to raise a pull request with the behavioural changes documented.

A design document would be split into the following parts:

**Title**

The title of the design document, normally this would pertain to a single component or one larger aspect of the whole of Strimzi. New pages inclusion (but not contents) would be discussed and decided on as part of a proposal if applicable, or during development if the complexity of a component merits a design document as judged by the maintainers.

**Overview, Context and Description**

A high level summary that any user of Strimzi should be able understand and use to decide if it’s useful for them to read the rest of the doc.
A description of the problem the component/code tackles, why the program/code is necessary, what people need to know to assess the success of the implementation.

**Configuration**
Describe how users interact with this component or system. I.e. configured via envars, custom resources, REST calls...
This information will be invaluable to a reader to understand where the configuration points are and how to interact with the system/component.

**Dependencies and Scope**
Is this intended to be a standalone component, are other components dependent on it, or is it dependent on other components.
Is this intended for a single Kafka cluster or to interact with multiple.


### When is a design document complete

Design document change is considered complete when the reviewers and maintainers believe the updated document to accurately reflect the implementation of the components behaviour. 
Once approved, the document is merged and is understood to be correct, any mistakes or changes to behaviour should be corrected in the form of a pull request alongside the code that changes the behavior.
A design document is never truly 'completed', as the code evolves the document(s) evolve with it.

A lot of implementation issues, problems and compromises are only clear during or after development of a feature or change, so a design doc is to be written alongside the code development, this helps in avoiding the writing of invalid design documents and help highlight potentially problematic elements of implementation or important holes in the testing strategy. 
Having the design document within the pull request allows contributors and reviewers to evolve the design while developing a pull request and during the review.
This strategy of design document development allows tweaks to the design document as a result of discussion, as well as to the code that implements the behaviour that the document described. This ensures the code and design docs evolve in parallel.


## Summary of changes

In an effort to improve where we are, the above guidelines are suggested for future contributions that are either new components or significant changes to the design or behaviour of existing components. This could result in some components only being partially documented but over time design documents for existing components can be contributed.
