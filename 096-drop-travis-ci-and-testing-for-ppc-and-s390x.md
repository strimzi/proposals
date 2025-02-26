# Drop Travis-CI and testing efforts for `ppc64le` and `s390x` architectures

This proposal suggests to drop current usage of [Travis CI](https://www.travis-ci.com/) in [strimzi-kafka-bridge](https://github.com/strimzi/strimzi-kafka-bridge) and [strimzi-kafka-oauth](https://github.com/strimzi/strimzi-kafka-oauth) repositories.

## Motivation

In the past, we adopted Azure Pipelines as the primary CI system for the Operators repository, using it to run tests, build documentation, and create artifacts and images.
The main reason for this transition was the improved resource quotas available for CNCF projects, which allowed us to run pipelines with far fewer restrictions than we had in Travis CI. 
However, Azure Pipelines does not provide agents for the `ppc64le` and `s390x` architectures.
This was not an issue for Operators, as we build multi-architecture images on `amd64` using _docker buildx_. 
However, for Strimzi Kafka Bridge and Strimzi Kafka OAuth, we decided to retain the testing pipelines for `ppc64le` and `s390x`.

In recent months, we have encountered several unexpected issues with Travis CI that required us to submit support tickets to restore our quota.
These issues are often not immediately noticed when they occur but are only discovered weeks later. 
It seems that no one even realizes that some pipelines fail to execute from time to time.
Another scenario is that jobs fail due to various Travis CI issues, yet we simply ignore the results and merge the PRs anyway.

Such cases create an unexpected maintenance burden with little added value:
- we must keep Travis CI configurations up to date, even though it is not our primary CI system and is only used for unit tests on the `ppc64le` and `s390x` architectures
- we occasionally need to contact Travis CI support to resolve issues
- no one actively reports bugs in `ppc64le` and `s390x` that could have been detected by pipelines running on Travis CI
- we generally do not pay attention to unstable pipelines on Travis CI and merge PRs regardless

_Note that Strimzi's Travis CI quota is sponsored by IBM._

## Proposal

This proposal suggests removing the existing Travis CI configurations from the Strimzi Kafka Bridge and Strimzi Kafka OAuth repositories, effectively eliminating Travis CI usage from the Strimzi organization.
Regarding the current support matrix, we will not make any changes, meaning that all currently supported architectures — `amd64`, `arm64`, `ppc64le`, and `s390x` — will continue to be supported by the Strimzi project.

We will newly use the following terminology:
- the `amd64` architecture will have the status `supported`, which means that all available tests will be run on this architecture on regular basis as we do now
- the `ppc64le`, `s390x`, and `arm64` architectures will have the status `supported, not tested`, meaning that tests will not be regularly executed on these architectures

All reported bugs related to the `ppc64le`, `s390x`, and `arm64` architectures will be appropriately triaged during Community calls and resolved based on the triage outcome, without any changes to the current process.

If accepted, we have to ensure that all testing is migrated from Travis CI to Azure Pipelines for unit, integration, and e2e tests running on `amd64` agents for every PR and commit into main branch for all affected repositories.

## Affected projects

This proposal affects the Strimzi Kafka Bridge and Strimzi Kafka OAuth repositories only. 

## Backwards compatibility

There is no impact on backwards compatibility.

## Rejected alternatives

Migrate Travis-CI workloads to [Testing Farm](https://docs.testing-farm.io/Testing%20Farm/0.1/index.html) that is Red Hat sponsored CI system.
This could allow us to continue running the tests on `ppc64le` and `s390x`, and add `arm64` architectures, however, it will not remove the maintenance burden of another CI system.
Overall, this alternative could end-up with the same situations as we are facing with Travis CI now.
