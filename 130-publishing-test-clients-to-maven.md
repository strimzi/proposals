# Publishing test-clients to Maven central repository

This proposal suggests publishing `test-clients` classes - mainly the `builders` - to Maven central repository.

## Current situation

Test-clients repository was created in order to have clients that are needed for the Strimzi operators system tests.
Originally, we used the `client-examples` repository and its clients, however, after multiple iterations of changes and discussion, we decided to create separate repository for clients used in tests.
The `test-clients` repository contains implementation of the Kafka clients (producer, consumer, streams), HTTP (producer and consumer), and CLI admin client tool.
These are bundled and released as an image, which then can be used in the Kubernetes' `Deployment` or `Job` in the tests - and that's how we are using them in our operators STs.
In the operators STs, we [implemented builders for these clients](https://github.com/strimzi/strimzi-kafka-operator/tree/main/systemtest/src/main/java/io/strimzi/systemtest/kafkaclients/internalClients), to have a simple manipulation.

## Motivation

The test-clients, even if they are used only in tests, seems to be used by other projects as well - by [Kroxylicious](https://github.com/kroxylicious) or [Streamshub](https://github.com/streamshub).
However, because we don't have any public builders they can use, they have to write their own.
The only available implementation is the one in the Strimzi operators repository - but it can be used only in the operators repository, it's not released anywhere.
Also, in the operators repository, the builders caused issues during the Maven build phase, as the goals were somehow mismatched, so the build failed on non-existing builders being used in the tests.
This was the main motivation in writing the builders inside the `test-clients` repository.

## Proposal

This proposal suggests publishing the test-clients builders into Maven central repository, allowing other projects to use them in their tests.
The artifacts will be released under `io.strimzi.test-clients` group ID.
For now, only released module will be `builders`.
Other modules will not be released at all.

In the future, we may start publishing the `clients` module, so users can use them as "external clients" (meaning that the clients will run locally on user's machine and will connect to Kubernetes or OpenShift cluster and particular bootstrap server, using NodePort, LoadBalancer, Route, etc.), as we use in operators system tests.
Thanks to that, we would be able to remove the external clients code from the Strimzi operators STs and have one implementation for everything.
However, this is not part of this proposal and can be discussed in future one.

Even though the `builders` module will be public, it will be still something which should be used only in tests - we will strongly advise to not use the test-clients in production environment.
Also, we will not be responsible for backporting fixes of CVEs to previous versions of the test-clients.
These things will be mentioned in the `README.md` of the repository.

## Affected/not affected projects

This proposal affects `test-clients` repository and `strimzi-kafka-operator` repository (because of the removal of the current builders from `systemtest` module).

## Compatibility

There is no backwards compatability issue.

## Rejected alternatives

Currently, there is no rejected alternative.