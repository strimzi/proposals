# Using Red Hat Universal Base Image 8 as the new Strimzi base image

This proposal suggests to use Red Hat UBI8 as the new Strimzi base image.

## Current situation

Strimzi container images are currently based on the CentOS 7 container images.
The CentOS 7 container images are stored on Docker Hub and used to build all Strimzi images.

## Motivation

Using the CentOS 7 image has three main disadvantages:
* It is hosted on Docker Hub which has limited number of anonymous pulls (see [proposal 14](https://github.com/strimzi/proposals/blob/main/014-move-docker-images-to-quay.io.md) for more details).
* CentOS 7 images are not released too often (at the time of writing, the image is 2 months old), so we need to install lot of updates while building the images
* CentOS 7 received CVE fixes in batches often onůy after they are available in other images

_Note: There has been recently many discussion about changes in the CentOS project.
But to my knowledge, they affect only the CentOS 8 version.
CentOS Streams and CentOS 7 remain unchanged.
CentOS 7 should receive maintenance updates until the year 2024.
So there is no time pressure to move to other base image caused by these changes._

## Proposal

Strimzi should move to use Red Hat Universal Base Image 8 (UBI 8) as a base image.
UBI8 is based on Red Hat Enterprise Linux 8 and is available to everyone for free (it does not include any Red Hat support).
It can be pulled without any registration from `registry.access.redhat.com/ubi8/ubi-minimal:latest` and be redistributed (i.e. we can push images build on this base image into our repositories).
The details can be found in the [Red Hat UBI EULA](https://www.redhat.com/licenses/EULA_Red_Hat_Universal_Base_Image_English_20190422.pdf) and in the related [FAQ](https://developers.redhat.com/articles/ubi-faq#).

UBI8 receives normally updates earlier than CentOS 7.
New versions of the image are also released more often.
And since it is not hosted on Docker Hub, it is not subject to Docker Hub pull limits.

There are several versions of the UBI8 image.
The `minimal` is the smallest and should be used by Strimzi.

### Affected repositories

This proposal covers the container images from the following repositories:
* `strimzi-kafka-operator`
* `strimzi-kafka-bridge`
* `client-examples`
* `test-clients`
* `strimzi-canary`

### Implementation

If approved, the Dockerfile files in the affected Strimzi projects will be updated to use the `ubi-minimal` image.

## Compatibility

There are no compatibility issues.

## Rejected alternatives

For this proposal, I considered mainly the Red Hat and CentOS images.
Unlike images for other Linux distributions, they are very close to the current image.
So moving to the new base image should mean minimal disruption and not too many changes.
