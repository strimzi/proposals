# Move Container Images to Quay.io

Docker is introducing from November 2020 service limits for container images stored in Docker Hub.
Free accounts will have following rate limits:
* 100 pulls for anonymous users per 6 hours
* 200 pulls for authenticated users (on the free plan) per 6 hours

In addition, it is also planned that images without any activity (image push or pull) will be kept only for 6 months (this might not start to apply already from November but only later).
For more details, see [Docker Hub Pricing & Subscriptions](https://www.docker.com/pricing).

## Motivation

The limits might impact Strimzi on several levels:
* Developers developing Strimzi who might need to pull the images often for development and test purposes
* CIs running under different accounts
* Users who might want to pull the images

The image expiration policy would also remove the images from our Docker Hub account and make them not available to users using older versions.

## Proposal

This proposal suggests to:
* Start using [Quay.io](https://quay.io/) as our new container registry for all master builds (`:latest` images).
* Start using [Quay.io](https://quay.io/) for releases starting with Strimzi Kafka Operators 0.21.0 and Strimzi Kafka Bridge 0.20.0 releases.
* Move client-examples and any UI related images to as well.

Additionally, we should make a copy of previous releases to [Quay.io](https://quay.io/) as well in order to:
* Make sure the releases are not lost.
* If needed, users can manually change their installation files for operator releases 0.20.0 and earlier and bridge releases 0.19.0 and earlier to use [Quay.io](https://quay.io/) as well.

Our container image structure (names, number of images etc.) changed significantly in Strimzi 0.12.0 release.
So the copying to [Quay.io](https://quay.io/) should be done for all images for release 0.12.0 and later.
This will also help to avoid confusion among users who find the old images not used anymore and believe that they are the latest versions.
The copying should be done only for releases - release candidates or the `latest` images will not be copied (the `latest` images should be pushed from new builds instead).

[Quay.io](https://quay.io/) also offers some more advanced features such as robot (service) accounts for easier CI integration or security scanning.

## Rejected alternatives

There are several available container repositories I'm aware of and which I considered:
* Google Cloud and Amazon AWS registries are bound to an account and are not for free (AFAIK you pay for used storage and data transfers). So we would need to organize a shared Strimzi account and make sure the costs are covered.
* GitHub container registry is currently available only as a beta. Any future pricing and availability is not clear.
* Docker Hub offers a program for Open Source projects. But it is not even clear what does this program offer for accepted open source projects - it exists just as an application form. So this does not seem to be transparent.
