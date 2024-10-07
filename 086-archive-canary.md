# Archive Canary project written in Go

This proposal is about archiving the [Canary project written in Go](https://github.com/strimzi/strimzi-canary).

## Current situation

The Canary project, currently written in Go programming language, is component for monitoring the Kafka cluster.
It provides functionality to periodically check the availability of the Kafka cluster during normal run, upgrades, downgrades, and rolling updates.
That is done by connection check to the cluster, but also producer and consumer, that periodically do the message transmission between Kafka cluster and Canary.
The Canary then provides Prometheus metrics and alerts for users or cluster admins to react on the Kafka cluster issues.

## Motivation

The Canary, as mentioned, is written in Go, however the Strimzi organization (and the engineers working on the projects inside this organization) is 
focusing on Java programming language.
That means the team is missing needed expertise in Go in order to solve various issues that Canary can have and has.
Based on the previous proposal from PR [#58](https://github.com/strimzi/proposals/pull/58), the Sarama Kafka client library lacks of features that are available in official 
Java Kafka client library - and the Sarama library is not the official Kafka library in comparison to the Java one.
Additionally, there are multiple [issues created in the Canary project](https://github.com/strimzi/strimzi-canary/issues) without any comment or resolution for quite a long time.
The project's dependencies weren't updated for two years now, meaning that there can be a lot of CVEs, unresolved issues with newer Kafka versions, and more.
Finally, the inactivity on the project proves that there is not enough resources in order to maintain such project on level of fixing the issues and CVEs, neither to add more features to the tool.

Because of the issues mentioned on the above lines, I'm proposing to archive the Canary project.

## Canary in Java

After few releases of Canary we realized that there is not enough expertise for the Go language and Sarama Kafka client doesn't provide functionality we need, 
the [PR with proposal to move Canary to Java](https://github.com/strimzi/proposals/pull/58) was created.
The proposal contained all issues with the current implementation together with proposed implementation and changes for the Canary in Java.
In parallel with the proposal, the POC was written in Java and is available in the [im-konge/canary-java](https://github.com/im-konge/canary-java) repository.
However, during the implementation process, we found out that few things are not possible using Java Kafka clients (for example the connection check that was one of the main features of Canary) and that
metrics related to Sarama client are not relevant anymore.
Changes like this would break the backwards compatibility, meaning that it would not be 1:1 copy of the Canary in Go.

Other than that, we thought about changing the metrics provided by Canary to be more insightful, but we didn't agree on what the metrics should be.
We had discussions with few community users about how the Canary can be more useful to them, but the users ended up writing their own Canary-like tool 
with functionality useful to them.

Because we didn't move forward with the proposal and agreement on the future of the Canary, after more than one year from the moment the proposal was created, we decided to close the 
proposal, and we agreed to archive the project on [community meeting on May 30th 2024](https://youtu.be/UpStul__uCw?si=GTA5edXJEnGgxP1a).
Additionally, we mentioned that in case that community finds Canary useful, provides information about what can be improved (in terms of metrics, functionality), and we will have enough
capacity and resources to maintain it, we can open a new proposal for writing the Canary vol.2 in Java.

## Proposal

After this proposal is approved, we should:
- archive the Canary project
- remove the Canary install files from Strimzi operators repository 
  - now from the `/packaging/install` folder
  - as part of the next Strimzi release, the installation files will be removed from the `/install` folder
- delete the mentions about the Canary project from the documentation and automation files (Makefiles)
- inform users about archiving the project

In case that community: 

- finds Canary useful 
- will be able to provide additional information about what metrics or functionality can be improved or used
- we will be able to maintain the project

we can write a new proposal for Canary vol.2 in Java.

## Affected/not affected projects

The only affected project is the [Canary](https://github.com/strimzi/strimzi-canary) that should be archived.
In the Strimzi operators repository, the only affected parts are the [installation files](https://github.com/strimzi/strimzi-kafka-operator/tree/main/install/canary) and 
[in development installation files](https://github.com/strimzi/strimzi-kafka-operator/tree/main/packaging/install/canary) that should be deleted by this proposal.

## Compatibility

The backwards compatibility is not relevant in this case, as the project will be archived and there is no other solution currently that would replace it.

## Rejected alternatives

### Maintaining Canary in Go and providing additional functionality

One of the rejected alternative was to keep updating the Canary in Go and add more functionality to it.
This was discussed and rejected because of:
- lack of Go experts in the Strimzi organization
- resources - it would take a lot of time to learn Go and properly testing every new change without knowing how it will work
- missing functionality in the Sarama Kafka client

### Maintaining Canary for dependency updates

Another one alternative was to keep Canary project and updating the dependencies.
Some of the dependency updates can be without breaking changes, but from time to time there are changes in the dependencies that requires additional 
changes to the code, which brings us to the same situation as the previous alternative - someone would need to do the changes to the code, 
test it properly and then release it.
Because of these issues, we decided to reject this alternative.

### Rewrite Canary in Java

Final alternative was to rewrite Canary in Java, but as was mentioned in the [Canary in Java](#canary-in-java), it would not be the same Canary as in Go, 
and we didn't agree on how the new metrics of the Canary in Java (and the overall implementation) should look like.
Because of this, we decided to reject this alternative.