# Rename the default branch of Strimzi GitHub repositories

CNCF is committed to use inclusive naming.
As a CNCF project, we should try to comply with the suggestions and recommendations.
One of these is naming of the default branch of our GitHub repositories.

## Current situation

Currently, all our GitHub repositories use `master` branch as the default branch.
This was the default name used by GitHub in the past as well as the name used by most projects.

## Motivation

The main motivation is following the recommendations of the [Inclusive Naming Initiative (INI)](https://inclusivenaming.org/).
Members of INI include CNCF as well as many other organizations and companies (including employers of many Strimzi maintainers, committers and contributors).
Please read the [INI Word replacement list](https://inclusivenaming.org/language/word-list/) for the reasoning why the name `master` might not be recommended.

## Proposal

We should change the name of the default branch from `master` to `main`.
`main` is the new default name used by GitHub for newly created repositories.
It is also one of the recommendations from the [INI Word replacement list](https://inclusivenaming.org/language/word-list/).

GitHub tries to make renaming the default branch as easy as possible.
Renaming the branch should automatically update all PRs, branch protection rules, redirect links and more.
For more details about renaming he default branch, see [Renaming the default branch from `master`](https://github.com/github/renaming).

### Risks

Despite GitHub trying to make it as easy as possible, renaming the default branch will still cause some disruptions:

* Users / Developers will need to get used to the new default branch name (for example when using commonly used commands such as `git rebase master` etc.)
* Users / Developers will need to update their local repositories to use the new branch

## Affected / not affected projects

This proposal affects all Strimzi GitHub repositories.

## Compatibility

Users will need to switch to the new default branch.
But no compatibility issues should be causes to the actual Strimzi applications.

## Rejected alternatives

This proposal follows the INI recommendations.
There are currently no rejected alternatives.
We could consider also using some other name for the default branch than `main`, but there do not seem to be any reasons to not follow GitHub's new default name.
