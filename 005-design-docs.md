# Design Documentation Proposal

This document is intended to progress discussion about how to write and maintain [design documentation](https://en.wikipedia.org/wiki/Software_design_description) for Strimzi.

The main goal of design documents for Strimzi is to make contributors more effective by forcing all involved to think through the design and gather feedback from others. People often think the point of a design doc is to to teach others about some system or serve as documentation later on. While those can be beneficial side effects, they are not the goal in and of themselves.
By having a set of design documentation, behaviours of Strimzi as a whole and each of the components are well defined and are agreed upon.

The benefits of adding design documentation to Strimzi are:
- Design docs/additions could help when there is conflict about the direction of a proposal/code change.
- It's hard for new contributors to get into the project and understand behaviours of certain components, this may be fixed by extra technical documentation.
- If just a couple of named contributors moved on to other things, the velocity of the project would drop considerably. Design doc would help in this case too, capturing conventions and behaviour enforcements that normally only get picked up and tweaked at review time.



## Where is Strimzi today

Currently there are no design documents for Strimzi, the closest thing Strimzi has would probably be the generated docs for classes, but these are more user oriented than design oriented.


## Proposal

### When to write a design document

It would be appropriate to add a design document when a significant new feature is being added, examples of this being:
- A new custom resource type being added, a design document explaining how its configuration would be implemented.
- A new feature to a pre-existing custom resource type, e.g. new component
- A reworking of an old feature, deprecating the old feature, including if applicable of how a user would migrate.

### Where to write a design document

A design document should be a markdown file located in the `design` directory of the `strimzi-kafka-operator` project.
Design documents for other Strimzi projects such as Oauth should also be located here as the intent is that the design documents are all colocated and that all components are a part of 'the operator' project.

### When is a design document complete

A design document is considered complete when the reviewers and maintainers believe the document to accurately reflect the current implementation of the components behaviour. Once approved the document is merged and is understood to be correct, any mistakes or changes to behaviour should be corrected in the form of a pull request.

### How to write a design document

The design documents would live in the strimzi-kafka-operator repo and would match the current code state, so that code changes could be audited with their corresponding design doc changes.

A design document would be split into `X` parts

**Title**

The title of the design doc, normally this would pertain to a single component or one larger aspect of the whole of the Strimzi project. New pages would be discussed and decided on as part of a proposal where needed.

**Overview**

A high level summary that any user of Strimzi should be able understand and use to decide if it’s useful for them to read the rest of the doc. It should be 3 paragraphs max.

**Context**

A description of the problem the program/code tackles, why the program/code is necessary, what people need to know to assess this project, and how it fits into the technical strategy of Strimzi.

**Goals and Non-Goals**

The Goals section should describe the user-driven impact of the code and specify how to measure success using metrics, normally that would be tested.
Non-Goals are equally important to describe which problems you won’t be fixing so everyone is on the same page.

**Milestones**

A list of measurable checkpoints, if there is future progress planned, including a milestone for the future in the design doc maye be useful for a reader to understand where the project is heading.

**High Level Description**

High level description describing the current implementation as well as example flow(s) to illustrate how users interact with this system and/or how data flow through it. This should build on the context.

**Technical Architecture section**

Walk through the flows in great detail to concertise the implementation. Feel free to include diagrams where relevant and sub-sections for edge-cases and implementation details.

## Summary of changes

Alongside code changes, behavioural changes of the operator will also be documented in the above described design documentation format.

Newly contributed proposals or issues would describe how their changes might alter the design documentation and are expected to be delivered alongside their code changes.

Design documents accumulate and form a body of design information for the project, and should be kept up to date with design changes. 



## Alternative implementation - Strimzi Improvement Proposal (SIP)

Inspired and closely resembling [Kafka Improvement Proposals (KIP)](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals), we merge the concept of design documentation and proposals together, meaning that each proposal is a point in time descriptor of current Strimzi behaviour and a proposal as to how it should change.

Small changes should just need a couple brief paragraphs, whereas large changes need detailed design discussions.

### SIP format

A SIP should contain the following sections (matching those of a `KIP`):

**Motivation** - describe the problem to be solved
**Proposed Change** - describe the new thing you want to do. This may be fairly extensive and have large subsections of its own. Or it may be a few sentences, depending on the scope of the change.
**New or Changed Public Interfaces**: impact to any of the "compatibility commitments" described above. We want to call these out in particular so everyone thinks about them.
**Migration Plan and Compatibility** - if this feature requires additional support for a no-downtime upgrade describe how that will work
**Rejected Alternatives** - What are the other alternatives you considered and why are they worse? The goal of this section is to help people understand why this is the best solution now, and also to prevent churn in the future when old alternatives are reconsidered.
