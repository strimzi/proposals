
# Converting Strimzi UI Mono Repo

The following proposal is a recommendation that we move to a mono repo and separate the current UI into smaller packages.

## Current situation

Currently the [Strimzi UI](./011-strimzi-ui.md) is being developed as one monolithic application for the client and server.  In order to help isolate code into manageable packages it is recommended that the project moves to a mono repo with smaller packages. This architecture will help clearly define functional areas and help with separation of concerns within the UI client and server. We will continue to have one single source of truth for the UI by having a monorepo, however with smaller packages it will allow UIs to pull in only the packages that are needed to build their UI without additional dependencies.  

## Motivation

By moving to a mono repo the project will gain the following:
 * **Simplified dependency management for view layers:**  Each of the view layers can pull in the functionality that the view needs by including only the packages the are required for that view.
 
 * **Focus on immediate client applications needs:**  The view layer makes up what the user will interact with for each application being built.  Each applications views are swapped out at build time via webpack normalization. This process requires at minimum skelton code for each of the view layers be added in order for the client to build. Short term this can result in extra code being generated and additional rework in the future based on each of the clients immediate needs for their consumers.  In order to reduce this rework and allow each client to develop views as needed, it is recommended that view layers be stored in a separate package (e.g. ui-carbon, ui-patternfly).
  
 * **Separation of concerns:**  Each of the packages will focus on a specific area of concern.  This will help in code clarity as well as reusability.

 * **Simplify build process:**  By introducing the use of a mono repo it will reduce our dependency custom webpack and build scripts.  This will make it easier for contributors to understand how the project is built.

## Proposal

### Tools
In order to support a mono repo the following tools will be introduced:

- [`Lerna`](https://github.com/lerna/lerna) We are using Lerna to support the development of a mono repo and manage all the packages that make up this project.
  
- [`Yarn Package Manager`](https://yarnpkg.com/)  Yarn package manager allows us to manage which dependencies get install to make sure our software is rebuilt in a consistent stable manner.  We are also you using workspaces provided by yarn with lerna to build our mono repo.

### Package structure

The following is the proposed package structure for the mono repo:


| Package | Description | npm package |
| ------- | ----------- | ----------- |
| models | Contains the data models that will be used by the server and client | @strimzi-ui/models |
| services | Contains the business logic for the ui. This acts the controller in the MVC model. | @strimzi-ui/services
| ui-(Client-Framework) | Packages prefixed with ui in front act as the view layer.  There will be multiple view layers for each client / framework that is being developed for this project.  (e.g. ui-carbon, ui-patternfly) | @strimzi-ui/ui-patternfly, @strimzi-ui/ui-carbon
| ui-server | This package contains the server responsible for serving the client and proxying requests. | @strimzi/ui-server


## Affected/not affected projects

This will affect the strimzi-ui project.

## Compatibility

Not applicable

## Rejected alternatives

Not applicable
