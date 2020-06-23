# Design Documentation Proposal

This document is intended to progress discussion about how to write and maintain [design documentation](https://en.wikipedia.org/wiki/Software_design_description) for Strimzi.

The main goal of design documents for Strimzi is to make contributors more effective by forcing all involved to think through the design and gather feedback from others. People often think the point of a design doc is to to teach others about some system or serve as documentation later on. While those can be beneficial side effects, they are not the goal in and of themselves.
By having a set of design documentation, behaviours of Strimzi as a whole and each of the components are well defined and are agreed upon in plain English.


## Where is Strimzi today

Currently there are no design documents for Strimzi, the closest thing Strimzi has would probably be the generated docs for classes, but these are more user oriented than design oriented.


## Proposed Design Documentation format

The design documents would live alongside the code for simplicity, so that code chanhes could be audited with their corresponding design doc changes.

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

Walk through the flows in great detail to concretize the implementation. Feel free to include diagrams where relevant and sub-sections for edge-cases and implementation details.


## Summary of changes

Alongside code changes, behavioural changes of the operator will also be documented in the above described design documentation format.

Newly contrinbuted proposals or issues would describe how their changes might alter the design documentation and are expected to be delivery alongside their code changes.