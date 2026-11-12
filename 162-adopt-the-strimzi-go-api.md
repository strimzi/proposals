# Adopt the Strimzi Go API

This proposal suggests adopting the [Strimzi Go API](https://github.com/scholzj/strimzi-go) project under the Strimzi GitHub organization.
It also describes how the project will be moved and how its existing users can migrate to the new location.

## Current Situation

Strimzi provides its custom resource APIs as a Java library (the `api` module).
Java applications can use it together with the Fabric8 Kubernetes client to manage Strimzi resources.
But Strimzi does not provide any APIs for other programming languages.

The Strimzi Go API project is currently hosted in my personal GitHub account in the [`scholzj/strimzi-go`](https://github.com/scholzj/strimzi-go) repository.
It provides the Strimzi custom resource APIs for the Go programming language.
It lets users manage Strimzi resources using the Kubernetes Go client (`client-go`).
It contains:

* Go types for all Strimzi custom resources (currently the `v1` API of the `kafka.strimzi.io` and `core.strimzi.io` API groups)
* Generated `ClientSets`, informers, and listers for the Strimzi custom resources
* Examples showing how to use the APIs
* Tests for the APIs, including serialization tests and tests against real Strimzi CRDs running in a Kubernetes cluster

The Go types are generated from the Strimzi `api` module by a small Java-based generator (`strimzi-go-generator`) that is part of the repository.
The `ClientSets`, informers, and listers are generated from the Go types using the standard Kubernetes code generators.
Thanks to that, adding support for new Strimzi versions requires only minimal manual work.

The project has been available since January 2025.
It has regular releases following the Strimzi releases (the latest version `0.12.0` supports Strimzi 1.3.x).
It is used, for example, by the [Keksposé](https://github.com/scholzj/kekspose), [Strimzi Shutdown](https://github.com/scholzj/strimzi-shutdown), and [Strimzi Backup](https://github.com/scholzj/strimzi-backup) projects.
It is also used by other users who build Go applications or Kubernetes operators that integrate with Strimzi.

## Motivation

Go is one of the most widely used programming languages in the Kubernetes ecosystem.
Many Kubernetes tools and operators are written in Go.
Having official Strimzi Go APIs makes it easier for them to integrate with Strimzi.

Users who want to integrate with Strimzi from Go currently have to choose between:

* Using the _unstructured_ Kubernetes client and working with Strimzi resources without any type safety
* Writing and maintaining their own Go types for the Strimzi custom resources
* Using the Strimzi Go API project from my personal GitHub account

The Strimzi Go API project currently says clearly that it is not part of the Strimzi CNCF project.
Some users might be reluctant to depend on a project that lives in a personal GitHub account and is maintained by a single person.
Moving it to the Strimzi GitHub organization would give users more confidence in using it.
It would also make it easier for other Strimzi maintainers and contributors to help maintain the project and keep it in sync with new Strimzi releases.

## Proposal

This proposal suggests adopting the Strimzi Go API project under the Strimzi umbrella.
It will live in the Strimzi GitHub organization in a repository called `go` (`strimzi/go`).
It will be maintained by the Strimzi maintainers in the same way as our other subprojects.

### Moving the Repository

The repository will be moved by using the GitHub [repository transfer](https://docs.github.com/en/repositories/creating-and-managing-repositories/transferring-a-repository) feature.
The repository will be transferred from `scholzj/strimzi-go` to the Strimzi GitHub organization and renamed to `go`.
Repeating `strimzi` in the repository name is not needed once it lives in the Strimzi GitHub organization.

Transferring the repository keeps the Git history, issues, pull requests, releases, and stars.
GitHub also automatically redirects all requests for the old repository (web, Git, and API) to the new repository.
The redirect works for both the transfer and the rename.
This includes the requests made by the Go tooling.
So existing users are not immediately affected and can migrate at their own pace.

To keep the redirects working, no new repository named `strimzi-go` will ever be created in the `scholzj` GitHub account or in the Strimzi GitHub organization.

### Go Module Path

The Go module path is currently `github.com/scholzj/strimzi-go`.
Transferring the repository does not change the module path.
The module path is defined in the `go.mod` file and is also used in the import statements in the generated code.

The module path will be changed to `github.com/strimzi/go` to match the new location of the repository.
This is a breaking change for the users of the library, as they will need to update the imports in their code.
However, the project is still at version `0.x`, and the change will be done only once when moving to the Strimzi GitHub organization.

All versions released before the move will remain available under the old module path.
They are cached by the Go module proxy (`proxy.golang.org`) and can also be fetched from the new repository thanks to the GitHub redirect.
So existing builds will continue to work without any changes.
New versions will be released only under the new module path.

### Migration Path

The move will be done in the following steps:

1. The repository will be transferred from `scholzj/strimzi-go` to the Strimzi GitHub organization and renamed to `strimzi/go`.
2. A final release under the old module path will be created (for example `0.12.1`).
   This release will mark the old module as deprecated by adding a `Deprecated` comment to the `go.mod` file.
   The deprecation message will point users to the new module path:
   ```go
   // Deprecated: The Strimzi Go API moved to github.com/strimzi/go. Please use the new module path instead.
   module github.com/scholzj/strimzi-go
   ```
3. The module path will be changed to `github.com/strimzi/go` in the `go.mod` file, in the `strimzi-go-generator`, and in the `hack/codegen.sh` script.
   All the generated code will be re-generated to use the new module path.
4. The next regular release (for example `0.13.0`) will be released under the new module path.
5. The change will be announced in the release notes, in the `README.md` file, and on the Strimzi communication channels.

The deprecation release ensures that the users who do not follow the announcements will learn about the move as well.
The Go tooling shows the deprecation message to the users (for example in `go get` or `go list -m -u all` commands).
It is also shown by IDEs using `gopls` and on [pkg.go.dev](https://pkg.go.dev/).

The users will migrate to the new module path by updating the imports in their Go code and updating their dependencies.
For example:

```sh
grep -rl 'github.com/scholzj/strimzi-go' --include='*.go' . | xargs perl -pi -e 's#github.com/scholzj/strimzi-go#github.com/strimzi/go#g'
go get github.com/strimzi/go@latest
go mod tidy
```

The migration instructions will be included in the `README.md` file and in the release notes.

Users who do not migrate will continue using the last release under the old module path.
They will not receive any new versions supporting new Strimzi releases until they migrate.

### Other Changes After the Adoption

Once the project is adopted, we should also do the following changes:

* Remove the note saying that the project is not part of the Strimzi CNCF project from the `README.md` file
* Update the license headers in the generated code (`hack/boilerplate.go.txt`) to use the same copyright notice as the other Strimzi projects
* Move the Java code of the `strimzi-go-generator` from the `cz.scholz` package to the `io.strimzi` package
* Add the files required for Strimzi subprojects such as `CODE_OF_CONDUCT.md`, `GOVERNANCE.md`, and `MAINTAINERS.md`
* Configure the repository in the same way as other Strimzi repositories (DCO checks, branch protection, Dependabot etc.)
* Mention the Go APIs on the Strimzi website

The project will keep using GitHub Actions for its CI.
It already has a GitHub Actions workflow that builds the project, checks that the generated code is up-to-date, and runs the tests against the Strimzi CRDs deployed in a Kind cluster.
The initial migration would use the existing CI as is.
But after the migration is done, we will _standardize_ the CI with other repositories (using shared actions where possible), onboard to license and CVE scanning, etc.

### Versioning and Releases

The project will keep its current versioning scheme.
Each minor release of the Strimzi Go API corresponds to a minor release of Strimzi.
The `README.md` file contains a table with the supported Strimzi versions for each Strimzi Go API release.
The releases will be done by the Strimzi maintainers by tagging the release in the Git repository and creating the GitHub release.
No other release artifacts are needed, as Go modules are distributed directly from the Git repository.

The project will continue to be released separately from the Strimzi operators.
Ideally, a new release supporting the new Strimzi version will be done shortly after each Strimzi release.

## Affected/Not Affected Projects

The Strimzi Go API project will have its own repository.
It does not require any changes to other Strimzi projects.

It depends only on the released Strimzi `api` module (used by the generator) and on the released Strimzi CRDs (used in the tests).
So it does not need to be released together with the Strimzi operators.
Any changes to the Strimzi `api` module and the CRDs are picked up when the project is updated to the new Strimzi version.

## Compatibility

All existing releases of the Strimzi Go API will remain available and usable under the old module path `github.com/scholzj/strimzi-go`.
Users using them will not be affected by the move.

The change of the module path to `github.com/strimzi/go` is a breaking change.
Users will need to update their imports when upgrading to the first release under the new module path.
The migration path described in this proposal minimizes the impact on the users and makes sure they are informed about the change.

## Rejected Alternatives

### Creating a New Repository and Asking Users to Move

Instead of transferring the repository, we could create a new repository in the Strimzi GitHub organization with a copy of the code and ask the users to move to it.
The original repository would be archived.
This would make it possible to keep full control over the old repository.
But it would lose the issues, pull requests, releases, and stars of the original repository.
It would also not provide any redirects to the new repository.
And users who do not follow the announcements could easily miss that the project moved.
The archived original repository would still be available to users, so this approach does not provide any benefits over the repository transfer.

### Keeping the Original Module Path

We could transfer the repository but keep the old module path `github.com/scholzj/strimzi-go`.
This would avoid the breaking change for the users.
But it would rely on the GitHub redirect forever.
It would also be confusing for the users, as the module path would not match the repository location.
And it would not be possible to fix it later without the same breaking change.

### Using a Vanity Import Path

We could use a custom import path not tied to GitHub, such as `strimzi.io/go`.
This requires serving a `go-import` meta tag from the Strimzi website.
It would make any future moves of the repository transparent to the users.
But it creates a dependency on the Strimzi website for all users of the library.
It also makes the project setup more complicated, and it is not commonly used by other CNCF projects.
So this proposal suggests using the plain GitHub-based module path instead.
