# Deprecate and remove OpenAPI v2 (Swagger) support on the Strimzi HTTP bridge

This proposal is about deprecating the [OpenAPI v2](https://swagger.io/specification/v2/) (Swagger) specification support on the Strimzi HTTP bridge.
It also proposes a plan to remove such support after its deprecation across different bridge releases.
The deprecation and removal will leave the bridge supporting only the [OpenAPI v3](https://spec.openapis.org/oas/latest.html) specification.

## Current situation

Currently, the codebase provides two JSON files describing the HTTP endpoints exposed by the bridge.
The [`openapiv2.json`](https://github.com/strimzi/strimzi-kafka-bridge/blob/main/src/main/resources/openapiv2.json) uses OpenAPI v2.
The [`openapi.json`](https://github.com/strimzi/strimzi-kafka-bridge/blob/main/src/main/resources/openapi.json) uses OpenAPI v3.
In reality, the `openapi.json` file is used internally to "load" the HTTP endpoints definition via the Vert.x Web OpenAPI component in order to build the web routes.
The HTTP endpoints specification via OpenAPI is also used by the Vert.x Web OpenAPI component to validate parameters and body on the incoming HTTP requests.
The `openapiv2.json` is just used to be returned as resource when an HTTP client issues a request on the `/openapi` HTTP endpoint.
Exposing the OpenAPI specification via a dedicated HTTP endpoint is useful to external systems like API gateways or tools for API testing and for clients code auto-generation.

## Motivation

The bridge has been exposing the HTTP endpoints definition via the OpenAPI v2 specification for supporting external systems and tools still using Swagger.
Internally, it has always been using the OpenAPI v3 to "load" the HTTP endpoints definition, build the corresponding web routes and validate the parameters and body on the incoming HTTP requests.
The OpenAPI v2 specification can be considered obsolete as the latest [release](https://swagger.io/specification/v2/) happened 10 years ago.
Most of the API gateways, clients and tools are now supporting the OpenAPI v3 specification.
Furthermore, every time there are changes in the HTTP endpoints definition, we need to keep the two JSON files in sync, because they are used both for different purposes as explained before.

## Proposal

The proposal is about deprecating the OpenAPI v2 specification support in the next Strimzi HTTP bridge 0.29.0 release and removing it in the first major or minor release of 2025.
To make the transition smoothly, the idea is to have two new HTTP endpoints:

* `/openapi/v2`: still exposing the bridge HTTP endpoints definition with the OpenAPI v2 specification.
* `/openapi/v3`: exposing the bridge HTTP endpoints definition with the OpenAPI v3 specification.

During the deprecation period, starting with the 0.29.0 release, the HTTP endpoints definition will be available with both OpenAPI v2 and v3 specification on the two different endpoints.
Any HTTP request issued to the current `/openapi` endpoint will be forwarded to the `/openapi/v2` endpoint, still with the OpenAPI v2 specification.

At the end of the deprecation period, with the first major or minor release of 2025, the `/openapi/v2` will be handled to return the `410 Gone` HTTP status code instead.
Any HTTP request issued to the `/openapi` endpoint will be forwarded to the `/openapi/v3`, with the OpenAPI v3 specification.

## Affected/not affected projects

The Strimzi HTTP bridge is the only project to be affected by this proposal. 

## Compatibility

During the deprecation period, the compatibility with external systems and tools using the OpenAPI v2 specification is guaranteed with the newly added `/openapi/v2` endpoint and the current `/openapi` forwarding to it as described before.
Of course, after the removal, the bridge won't be compatible with OpenAPI v2 specification anymore.

## Rejected alternatives

One alternative was to deprecate with the 0.29.0 release and remove with the future 0.30.0, but it was rejected as not giving enough time to the users to adapt, also taking into account the effort for the implementation.