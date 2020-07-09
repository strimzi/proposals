# Design Documentation Proposal

This document is intended to progress discussion about how to write and maintain [design documentation](https://en.wikipedia.org/wiki/Software_design_description) for Strimzi.

The main goal of design documents for Strimzi is to make contributors more effective by forcing all involved to think through the design and gather feedback from others. People often think the point of a design doc is to to teach others about some system or serve as documentation later on. While those can be beneficial side effects, they are not the goal in and of themselves.
By having a set of design documentation, behaviours of Strimzi as a whole and each of the components are well defined and are agreed upon.

The benefits of adding design documentation to Strimzi are:
- Design docs/additions could help when there is conflict about the direction of a proposal/code change.
- It's hard for new contributors to get into the project and understand behaviours of certain components, this may be fixed by extra technical documentation.
- If just a couple of named contributors moved on to other things, the velocity of the project would drop considerably. Design doc would help in this case too, capturing conventions and behaviour enforcements that normally only get picked up and tweaked at review time.



## Where is Strimzi today

Currently there are no design documents for Strimzi, the closest thing Strimzi has would probably be the generated docs for classes - these are more user oriented than design oriented. Some classes also have extensive documentation comments found at the top of their files explaining their usage and design.


## Proposal

### When to write a design document

It would be appropriate to add a design document when anything which touches a 'Public API'.
Any of the following should be considered a "public API":
- Additional Custom Resource Definitions (CRD) or structure changes of any of the project's CRDs (excluding documentation of field)
- Changes in how CRDs are used or function (e.g. annotation changes)
- Anything which would add, remove or change any other public Strimzi APIs (e.g. New endpoints in the bridge, or new REST APIs).
- Anything which changes the accessibility of any of Kafka's public APIs (e.g. changes to how metrics get exposed, changing which broker configs are reserved to the operators, or locking/exposing functionality either via the network, or Kafka protocol, such as authorization)


### Where to write a design document

A design document should be a markdown file located in the `design` directory of the repository they are located in, for instance all design documents relating to the strimzi operator(s) should be located in the `strimzi-kafka-operator` repository under the `design` folder.

#### Strimzi Design Document Organization
Design documents should be splt into logical areas, for repositories that aren't `strimiz-kafka-operator` I suspect this would start as a single or just a few succinct documents.
For `strimzi-kafka-operator` I would suggest splitting design documentation from 'public API' perspective, that is to say all documents relating to the `Kafka` custom resource would be separated into its own folder called `Kafka`, similarly for `KafkaConnect` etc.
For design documentation that would cover the whole operator and not just any single custom resource I would suggest a folder named `operator` this would describe behaviours not unique to any one specific reconcile loop or verticle.


### When is a design document complete

A design document is considered complete when the reviewers and maintainers believe the document to accurately reflect the current implementation of the components behaviour. Once approved the document is merged and is understood to be correct, any mistakes or changes to behaviour should be corrected in the form of a pull request.

A lot of implementation issues, problems and compromises are only clear during or after development of a feature or change, so a design doc is recommended to be written alongside the code development, this helps in avoiding the writing of invalid design documents. 

Having the design document within the pull request allows contributors and reviewers to evolve the design while developing and during the review. This allows tweaks to the design, for current discussion and future reading, as well as to the code accordingly. This ensures the code and design docs evolve in parallel.

A design document should be trusted by readers to be right, as part of this it would be highly recommended that test scenarios be included with the design document in the form of behaviour driven tests in the [Gherkin Syntax](https://cucumber.io/docs/bdd/better-gherkin/). This would allow reviewers to better identify potential missing tests and more easily link up test scenarios with their associated unit, integration and system tests.
An example of this might be something like:
```
Given a user deletes the CA certificate
When the operator reconciles
Then the operator will replace it
```
or
```
Given a KafkaRebalance proposal is approved
When the KafkaRebalance enters the Ready state
Then the cluster will be rebalanced to 'X' state
When the KafkaRebalance is refreshed
Then the KafkaRebalance has no proposal
And the KafkaRebalance will be in the Ready state
```


### How to write a design document

The design documents would live in the related repository in the Strimzi Organization and would match the current code state, so that code changes could be audited with their corresponding design doc changes.

A design document would be split into the following parts:

**Title**

The title of the design doc, normally this would pertain to a single component or one larger aspect of the whole of Strimzi. New pages would be discussed and decided on as part of a proposal where needed.

**Overview and Context**

A high level summary that any user of Strimzi should be able understand and use to decide if it’s useful for them to read the rest of the doc.
A description of the problem the program/code tackles, why the program/code is necessary, what people need to know to assess the success of this design.
Optionally include a high level description describing the current implementation as well as example flow(s) to illustrate how users interact with this system and/or how data flow through it.

**Goals and Non-Goals**

The Goals section should describe the user-driven impact of the code and specify how to measure success using metrics, normally that would be tested.
Non-Goals are equally important to describe which problems you won’t be fixing so everyone is on the same page.

**Technical Architecture section**

Walk through the flows in great detail of the implementation. Feel free to include diagrams where relevant and sub-sections for edge-cases and implementation details.
This section would contain test scenarios in the form of behaviour driven tests in the [Gherkin Syntax](https://cucumber.io/docs/bdd/better-gherkin/) if applicable.


## Summary of changes

In an effort to improve where we are, the above guidelines are suggestions for future contributors but no enforced or mandated. This means that we will not place the entire burden on the first person to come along who wants to make a change in a given area. 
Instead with larger coontributions and proposed changes we would request contributors to add to whatever framework of design documentation already exists.

Alongside code changes, behavioural changes of the operator will also be documented in the above described design documentation format.

Newly contributed proposals or issues would preferably describe how their changes might alter the design documentation and would be requested to be delivered alongside their code changes.

Design documents accumulate and form a body of design information for the project, and should be kept up to date with design changes. 



## Rejected Alternative implementation - Strimzi Improvement Proposal (SIP)

**NOTE:** After discussion the concepts of proposals and design documents shoudl be separated for the moment.

Inspired and closely resembling [Kafka Improvement Proposals (KIP)](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals), we merge the concept of design documentation and proposals together, meaning that each proposal is a point in time descriptor of current Strimzi behaviour and a proposal as to how it should change.

Small changes should just need a couple brief paragraphs, whereas large changes need detailed design discussions.

### SIP format

A SIP should contain the following sections (matching those of a `KIP`):

**Motivation** - describe the problem to be solved
**Proposed Change** - describe the new thing you want to do. This may be fairly extensive and have large subsections of its own. Or it may be a few sentences, depending on the scope of the change.
**New or Changed Public Interfaces**: impact to any of the "compatibility commitments" described above. We want to call these out in particular so everyone thinks about them.
**Migration Plan and Compatibility** - if this feature requires additional support for a no-downtime upgrade describe how that will work
**Rejected Alternatives** - What are the other alternatives you considered and why are they worse? The goal of this section is to help people understand why this is the best solution now, and also to prevent churn in the future when old alternatives are reconsidered.
