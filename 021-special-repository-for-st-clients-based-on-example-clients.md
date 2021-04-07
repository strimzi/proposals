# Special repository for ST clients based on example clients

This proposal suggests creating a new repository for `Strimzi` ST client based on 
[Strimzi client-examples](https://github.com/strimzi/client-examples).

## Current situation

Currently, we are using two clients in our STs:
 - `InternalKafkaClients` (based on `test-client` image) 
 - `example clients` (from [Strimzi client-examples](https://github.com/strimzi/client-examples)).

The plan is to remove the `InternalKafkaClients` and keep only `example clients`.

## Motivation

While testing `Strimzi` we need, in some cases, special configuration of clients, which is not implemented in the
`client-examples`.
How we discussed earlier, the `client-examples` repository should be really _exemplary_, and we should not add any extra
_configuration_ or _extensions_ to it. For this kind of enhancements we should have repository, which will have
`client-examples` as base, and we will be able to add special setting without disrupting the basic idea of example
clients.

## Proposal

 * Create a new repository for `systemtest client`
    * complex implementation of clients for testing
    * will be based on [Strimzi client-examples](https://github.com/strimzi/client-examples)
    * we'll be able to modify it with our special configuration
    * the main idea of example clients will be kept
    
 * The original `client-examples` repository will be kept
    
## Affected/not affected projects

Only `systemtest` part of the `Strimzi` will be affected.

## Rejected alternatives

There are no rejected alternatives at the moment.