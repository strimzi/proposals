# Strimzi Admin Server

The Strimzi Admin repository was setup in December 2019 to hold an implementation of an Admin API. The repository has not had any contributions yet so this proposal sets out what the structure of an Admin Server might look like in order to get to the initial implementation of the server.

## Motivation

An API server will:
* provide a consolidated backend API for additional interfaces like a browser based UI or a CLI.
* be capable of supporting both REST and GraphQL interfaces.
* allow for more sophisticated APIs to be built on top of the existing APIs providing value-add to existing content.

## Proposal

### The server
The proposal is to make a modular server using the Vert.x-Web toolkit which has strong support for creating both REST and GraphQL interfaces. The server contains a single Vert.x verticle which listens for inbound traffic on 1 or more configurable ports. Allowing multiple configurable ports will allow the server to expose different functionality on each port, the simplest example of this being exposing an HTTP API on one port and a separate HTTPS API on a different port. The server module itself does not contain any API implementations but defines a service interface using the Java SPI and will load modules that implement that interface from the classpath. The implementation of the APIs that the server then exposes can be created independently and will be automatically included in the running server if they are available on the classpath when the server bootstraps.

The reason for the modular approach is that it allows the strimzi project to create a base set of functionality that can easily be extended by a downstream project. Not only is it easy to extend the API but it can be done in isolation. This removes the requirement of the extensions needing to track the strimzi source repositories. For example, lets assume that strimzi only implements a Kafka administration API which provides a set of APIs based around the Kafka Admin Client. A downstream project then would like to add a Schema Registry to the cluster and needs to add an API in support of the new functionality. With the modular server, they can define the API and its implementation, bundle the new functionality into a JAR file and add the new JAR file to the classpath. A secondary benefit of a modular approach is that it allows the strimzi user to configure the server to only contain the functionality that they would like to use providing for a leaner server.

The classpath of the server can be configured at build time or run time. To configure at build time, the server and dependant modules can be bundled in a fat-JAR and obviously, at run time, the JVM can be configured in multiple ways to include JAR files on the classpath.

### REST API Implementations
REST modules would use the Vert.x Web API Contract support and specifically the OpenAPI3 support to define the shape of the API and bind the operations to a set of handlers. The advantage of using the OpenAPI3 definition is the support it contains for validation which allows you to build a more secure API without having to pollute the implementation with validation code. It leaves the implementation of the operations to focus entirely on the business logic. A secondary advantage of using the OpenAPI3 specification is the numerous tools that are freely available to create generated documentation of the API.
 
 The Vert.x OpenAPI3 support will read an OpenAPI3 specification YAML file and allow the developer to map the original operations to handler methods. It creates a Vert.x Router which can be mounted on a Vert.x HTTP/S server. The module would implement the service interface defined by the server which will contain a method that the server can call to load the OpenAPI3 router and mount the router as a subrouter at a specified mount point to the root path. So, for example, if the OpenAPI3 spec defined endpoints `GET /topics` and `POST /topics` and these were mounted on the root path at the mount point `/admin` then the server would listen for incoming requests of `GET /admin/topics` and `POST /admin/topics` and route them to the handlers defined in the module. The mount points create a namespace so should not be empty and should be unique and the server should police that in order to prevent name clashes.  

### GraphQL API Implementations
GraphQL modules would use the Vert.x GraphQL Handler support. GraphQL is still maturing as a technology although the Java implementation is stable and is being actively developed. GraphQL requests consist of queries, mutations and subscriptions. Queries and mutations, like REST, follows a request/response model and are implemented against a single HTTP endpoint with the actual request being specified in the payload as a json object. Subscriptions however are conversational and are normally implemented using web sockets. Vert.x GraphQL supports  the Apollo Websocket server model. The GraphQL API is defined by a schema and a runtime library that connects the nodes and properties to resolvers, also called datafetchers in the Java implementation. A datafetcher is responsible for returning the value for the node or property that it is assigned to. If the library can determine the value of a node or property then there is a default property datafetcher that removes the requirement of defining a custom datafetcher for every node and property in the graph.

The Java GraphQL library allows you to modularise the schema. It has two methods for doing this, the first being a simple merge of multiple schema definition files. Below is an example:
```
File adminSchema = loadSchema("admin.graphqls");
File registrySchema = loadSchema("registry.graphqls");

TypeDefinitionRegistry typeRegistry = new TypeDefinitionRegistry();

typeRegistry.merge(schemaParser.parse(adminSchema));
typeRegistry.merge(schemaParser.parse(registrySchema));
``` 
The second mechanism allows you to extend a type in the schema. For example, if you have the following schema definitions:
```
type Topic {
   name: String
}

extend type Topic {
   partitionCount: Integer
   replicationFactor: Integer
}
```
the result would be the same as defining the following:
```
type Topic {
   name: String
   partitionCount: Integer
   replicationFactor: Integer
}
```
The GraphQL parser does not permit type re-definitions so, a name clash in the schema will create an error and the schema will be rejected. The proposal is to use the Java service interface to define a GraphQL module service which loads a schema for that module and a runtime wiring which is the definition that connects the nodes and properties to the implementation. The schema and wiring are then merged to give a single consolidated GraphQL API. This would allow a module to easily create both a REST interface and a GraphQL interface for the module and share a lot of the implementation between the two. 

### Security
Vert.x networking is based on Netty and works in a very similar manner. When a request is received by the server, it is funneled through a pipe and that pipe has a number of interceptors or handlers that can inspect the contents at that point and ignore or modify it before allowing it to proceed through the pipe. Vert.x also creates a routing context that travels through the pipe with the request and is available for inspection and/or modification by all the handlers. A handler can decided whether the request should be passed on to the next downstream handler or the request is complete and the downstream handlers should be bypassed and they have access to the full request through the routing context. The handlers that are called for a particular request is defined by the Vert.x `Router` which maps patterns to particular handlers. To implement a security handler we add a handler that matches all paths, i.e. the root path (`/`) and ensure that it appears in the router at the top of the list of patterns. When the handler is called, it creates a generic security context and attaches the security context to the routing context making it accessible to downstream handlers.   

### Client Service
A similar model to the security handlers can be employed to create a client service. Examples of client services are the Vert.x web-client for HTTP requests to backend services or the Vert.x Kafka Admin Client. The client service would be added to the routing context and a downstream handler could access the client service and use it to obtain a properly configured client. The advantages of this approach is that the combination of the security handlers and the client service hides a lot of complexity leaving the implementation of the endpoint handlers or the datafetchers to focus on the business logic. A second advantage is that the client service can manage the closing of clients when requests have completed.

### Extending via a downstream project
Using the infrastructure defined above, if a downstream project wishes to add a new feature to the API then it can proceed as follows. It creates a maven project that outputs a fat JAR. The entry point for the JAR implements the REST service interface and/or the GraphQL service interface. If the extension is adding a REST api, they then define an openapi yaml spec of the REST api and the implementations of the handlers for all the operations in the yaml spec. They then implement the method in the REST service interface to return a fully configured `OpenAPI3RouterFactory`. Similarly for the GraphQL, they define a GraphQL schema file which defines the api and a `RuntimeWiring` to link the schema nodes and properties to their implementing datafetchers. They then implement the method in the GraphQL service interface to return a fully configured `TypeDefinitionRegistry` containing the schema and the `RuntimeWiring` containing the implementation. The JAR file is then added to the classpath of the REST server and it will automatically load and expose the new API at startup time.