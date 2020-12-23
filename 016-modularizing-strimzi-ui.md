
# Modularizing Strimzi UI

The following proposal is a recommendation that we move from one monolithic web client to a web client that is made up of smaller packages.

## Current situation

Currently the [Strimzi UI](./011-strimzi-ui.md) is being developed as one monolithic application for the web client and web client server.  In order to help isolate code into manageable packages it is recommended that the project moves to a modularized architecture.  

In the proposed architecture change the web client will be split up into smaller packages. This architecture will help clearly define functional areas and help with separation of concerns within the web client and the web client server. We will continue to have one single source of truth by using the current repo, however with smaller packages it will allow the multiple web clients (e.g. Carbon web client, PatternFly web client, etc.) to pull in only the packages that are needed to build that client.

## Motivation

By implementing this proposal the project will gain the following benefits:

 * **Simplified dependency management for view layers:**  Each of the view layers that make up the web clients, can pull in the functionality that the view needs by including only the packages the are required for that view.
 
 * **Focus on immediate needs for each web client:**  To build each of the different web clients, currently the views that are displayed to the user are swapped out at build time.  This is done via webpack normalization. In order to support this at minimum skelton code for each of the views need to be added in order for each client to build. Short term this can result in extra code being generated and additional rework in the future based on each of the web clients needs for their consumers.  In order to reduce this rework and allow each web client to develop views as needed, it is recommended that view layer be stored in a separate package (e.g. ui-carbon, ui-patternfly).
  
 * **Separation of concerns:**  Each of the packages will focus on a specific area of concern.  This will help in code clarity as well as reusability.

 * **Simplify build process:**  By introducing smaller packages it will reduce our dependency on custom webpack and build scripts.  This will make it easier for contributors to understand how the project is built.

## Proposal

### Tools
In order to support a modular architecture the following tools will be introduced:

- [`Lerna`](https://github.com/lerna/lerna) We are using Lerna to support the development of each package and manage all the packages that make up this project.
  
- [`Yarn Package Manager`](https://yarnpkg.com/)  Yarn package manager allows us to manage which dependencies get install to make sure our software is rebuilt in a consistent stable manner.  We are also using workspaces provided by yarn with lerna to build each of the packages.

### Package structure

The following is the proposed package structure for the mono repo:


| Package | Description | npm package |
| ------- | ----------- | ----------- |
| models | Contains the data models that will be used by the server and client | @strimzi-ui/models |
| services | Contains the business logic for the ui. This acts the controller in the MVC model. | @strimzi-ui/services
| ui-(Web-Client-Framework) | Packages prefixed with ui in front act as the view layer.  There will be multiple view layers for each web client that is being developed for this project.  (e.g. ui-carbon, ui-patternfly, etc.) | @strimzi-ui/ui-patternfly, @strimzi-ui/ui-carbon
| ui-server | This package contains the web client server responsible for serving the ui and proxying requests. | @strimzi/ui-server


## Affected/not affected projects

This will affect the strimzi-ui project.

## Compatibility

Not applicable

## Rejected alternatives

Not applicable
