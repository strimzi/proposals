# Use the admin-server REST API in strimzi-ui 

The [admin-server](012-admin-server.md) is designed to support a REST API and a GraphQL API. The 
[strimzi-ui](011-strimzi-ui.md) is currently designed to use the GraphQL API.

Having started using the GraphQL API it is clear that the entities and operations that the UI requires are better 
suited to a REST API. This proposal updates the [strimzi-ui](011-strimzi-ui.md) design to use the REST API exposed by 
the [admin-server](012-admin-server.md).
 
## Current situation

Currently [strimzi-ui](011-strimzi-ui.md) is using the GraphQL API from the [admin-server](012-admin-server.md).

## Motivation

GraphQL provides a number of benefits for APIs including:

* a way for clients to follow references between entities in a single request
* a way for clients to describe the data they want, and get returned only that data
* a typed schema (with the ability to define new types)
* built in support for schemas

It also brings a number of challenges:

* support for API versions is not so well defined as REST
* federating APIs is much harder and introduces a performance hit
* complexity both in the server code and in the client code
* whilst querying data is simple, mutating data is not as simple when using GraphQL

Applying this to Strimzi we feel that the disadvantages of using GraphQL outweigh the benefits:

* the data exposed does not have complex bi-directional relationships - the number of entities is small and entities do
  not have a lot of references to another entity - this removes much of the benefit of GraphQL
* typically the UI requires the entire entity (there is not a lot of "optional" data)
* strimzi-ui must federate APIs (across Kafka instances and across Kafka and configuration) - federating this using REST
  is simple but complex with GraphQL
* the Strimzi Bridge already exposes a REST API, so using REST fits better with the strimzi ecosystem
* as the surface area of the API is small, the code complexity of using GraphQL outweighs the benefits seen for a large 
  API (e.g. via built in schema support)
* supporting versioned APIs will be beneficial for the API as it can allow multiple versions of strimzi to be used with 
  a single UI

## Proposal

### Prioritize development of the strimzi-admin REST support

The strimzi-admin project currently intends to support both REST and GraphQL. If this proposal is adopted then the 
GraphQL support will be dropped and the REST support continued.

### Adjust strimzi-ui to use REST API

The strimzi-ui project will be adjusted to use the REST API:

* the entity model will be generated from the OpenAPI REST API schema
* the introspection features currently defined for the strimzi-ui project will be replaced by metadata APIs.

## Affected/not affected projects

The strimzi-ui project is affected (as outline in the Proposal). Other projects are not. 

## Compatibility

There are no compatability issues, as the API is still under development.

## Rejected alternatives

There were no rejected alternatives.
