# Restructure the installation files

This proposal suggests restructuring to the folders of the [`strimzi-kafka-operator` GitHub repository](https://github.com/strimzi/strimzi-kafka-operator).

## Current situation

Currently, the Strimzi installation files in our master branch correspond to the images built from the master branch.
That means they are under constant development and often work only with the freshly pulled images.
That suits well for development work, but not necessarily for the users.
They often checkout the GitHub repository, see the install or example folders and try them, not realising that they're using `master` rather than a stable release.
If they just use them for a short time it should not matter.
But if they keep using them, they will run into issues later.

## Motivation

Improve the experience of the users who checkout Strimzi and want to install / try it.

## Proposal

We should change the way we manage these files.
This proposal suggests following:
* Create a new subdirectory `packaging`
    * This subdirectory will contain the _in-development_ versions of `install`, `examples` and `helm-chart` directories.
    * Any changes in regular PRs, generated CRD files etc. will be done in this directory
    * New releases will use these files for packaging
* The original `install`, `examples` and `helm-chart` directories will be kept, but will contain the files for the latest stable release
    * Updates to them will be done only during / after release
    * They will not be changed with regular PRs

### Risks

The proposal keeps existing directories with more or less the same files but changes their purpose.
That can be confusing for people using them today for the right purpose.
It might be also confusing for new contributors who might try to change them in their PRs

## Affected/not affected projects

This proposal affects only the [`strimzi-kafka-operator` GitHub repository](https://github.com/strimzi/strimzi-kafka-operator).

## Compatibility

Call out any future or backwards compatibility considerations this proposal has accounted for.

## Rejected alternatives

Following options were considered but I decided against them at the end:

### Moving the released files into separate directory

* A new directory `deploy` will be created
    * This directory will have its own versions of the `install`, `examples` and `helm-chart` directories
    * Their contain will correspond to the latest stable release
    * They will be updated only when a new release happens, but not with regular PRs
    * `deploy` directory is common also on some other projects, so people should understand its purpose

The disadvantage would be that old links would stop working and people might be confused.
On the other hand, it would make life easier for people used to use the current directories for development since they would not find them instead of installing the last release by mistake.

### Removing the released Helm Chart

The released Helm Charts are on the website and can be pulled from there.
I'm not sure how often the latest Helm Chart is really installed from file directly.
but we anyway generate it during the release.
So having it there should not add too much effort.
