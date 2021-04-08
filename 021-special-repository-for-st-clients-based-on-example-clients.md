# Special repository for ST clients based on example clients

This proposal suggests creating a new repository for `Strimzi` ST client based on [Strimzi client-examples](https://github.com/strimzi/client-examples).

## Current situation

Currently, we are using two clients in our STs:
 - `InternalKafkaClients` (based on `test-client` image) 
 - `example clients` (from [Strimzi client-examples](https://github.com/strimzi/client-examples)).

The plan is to remove the `InternalKafkaClients` and keep only `example clients`. 
The `test-client` is not sufficient anymore, it can create single producer/consumer, send/receive messages, and then we can assert result.
We are stuck here, and we need to wait until a producer/consumer is finished. 
With `example clients` we are able to do a lot more.
For example - create a continuous job for sending messages with delay, 
_stack_ the producers to create a _traffic_,  add extra configuration, use different types of producer/consumer (for Bridge, Kafka, ...) and many more.  

## Motivation

While testing `Strimzi` we need, in some cases, special configuration of clients, which is not implemented in the `client-examples`.
How we discussed earlier, the `client-examples` repository should be really _exemplary_, 
and we should not add any extra _configuration_ or _extensions_ to it. 
For this kind of enhancements we should have repository, which will have `client-examples` as base, 
and we will be able to add special setting without disrupting the basic idea of example clients.

## Proposal

 * Create a new repository for `systemtest client`
    * name will be `test-clients`
    * component owners will be same as for STs
    * PR checks:
       * DCO
       * build - `mvn` build, checkstyle (with some simple UTs or ITs)
    * complex implementation of clients for testing
    * we'll use both Kafka and Bridge clients from `client-examples`
    * will be based on [Strimzi client-examples](https://github.com/strimzi/client-examples) - we'll copy the
      `client-examples` code and then modify it - each repo will then _go their own way_
    * we'll be able to modify it with our special configuration
    * the main idea of example clients remain intact
    
 * The original `client-examples` repository will be kept

## Advantages

There are many things we can implement. 
Good example is returning exceptions and return codes into the `job` status (as we are using `k8s` jobs for deploying the example clients) and asserting it in tests. 
Currently, we have to grep exceptions from the job log - which can be a problem. 

## Images and releases

The images will be built as in `client-examples` after each merged PR and pushed to `strimzi` repository on `quay.io` with `latest` tag.
The `systemtest client` will have its own release cadence and versioning, which will depend on new features.  

## Kafka version

By default, the `systemtest client` will use latest released version of Kafka.
For using older version, the user will have to specify the `KAFKA_VERSION` environmental variable.
The image will contain `.jars` of all supported Kafka versions, the ones for specified version will be used.
If the user specifies unsupported Kafka version, error message will appear in _status_ and _log_ together with list of supported versions.
List of all supported versions will be present in `supported-kafka-version.yaml`.
We'll start with version `2.5.0`, as it's the oldest supported version by Strimzi,
and we'll support it until `test-clients` features will allow us to use that specific version or until we will decide to completely deprecate it.

## Implementation

Client will be implemented in Java, same as `client-examples`.

## Affected/not affected projects

Only `systemtest` part of the `Strimzi` will be affected.

## Rejected alternatives

There are no rejected alternatives at the moment.