# Use ubi10-micro as a base image

This proposal suggests changing our base image from `ubi9-minimal` to `ubi10-micro`.

## Current situation

Current Strimzi images are based on [`ubi9-minimal`](https://catalog.redhat.com/en/software/containers/ubi9/ubi-minimal/615bd9b4075b022acc111bf5).
UBI 9 is based on Red Hat Enterprise Linux 9.
Its minimal variant reduces the security footprint by using `microdnf` instead of full `dnf` and by removing many common packages that are present in the standard UBI image.
The image is published by Red Hat and is available to users without any needed subscriptions.

## Motivation

With the current situation around AI and security where basically every day a new CVE is opened we should aim to have a minimal security surface in our images.
As security is one of our goals we should make sure that images we provide to users are always up-to-date and have minimal CVEs that could affect the users.
Our existing security scans also show a lot of base CVEs that are not directly influencing Strimzi, but we can get rid of them to make the scans cleaner and users less worried about potential security issues.

## Proposal

To follow security standards and harden our images, Strimzi should migrate to `ubi10-micro`.
This change contains two steps:
- Bump UBI version from 9 to 10. [RHEL 10 was released on May 20, 2025](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux-10) (announced at Red Hat Summit) and it contains the latest improvements on the OS side.
- Switch from `minimal` image to `micro`. The micro image does not contain a package manager and the security surface is much smaller than UBI's minimal version.

### Minimal vs Micro

The key difference is that `ubi-micro` excludes the package manager (`microdnf`) and all of its dependencies.
This makes it Red Hat's [distroless](https://www.redhat.com/en/blog/introduction-ubi-micro) container image - built from the same RHEL packages but without the packaging tools.

| Feature                      |        `ubi-minimal`         |              `ubi-micro`               |
|------------------------------|:----------------------------:|:--------------------------------------:|
| Package manager              |          `microdnf`          |                  None                  |
| Shell (`bash`)               |             Yes              |                   No                   |
| Base image size (compressed) |            ~33 MB            |                ~7.5 MB                 |
| Pre-installed packages       |             ~100             |                  ~15                   |
| Package installation         |  `RUN microdnf install ...`  | Multi-stage build with `--installroot` |
| Freely redistributable       |             Yes              |                  Yes                   |
| Architecture support         | amd64, arm64, ppc64le, s390x |      amd64, arm64, ppc64le, s390x      |
| FIPS support                 |     Inherited from host      |          Inherited from host           |

More details can be found in the following sources: [RHEL 10 — Types of container images](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/building_running_and_managing_containers/types-of-container-images), [Introduction to UBI Micro](https://www.redhat.com/en/blog/introduction-ubi-micro), [UBI 10 Micro catalog](https://catalog.redhat.com/en/software/containers/ubi10-micro/66f2abd91123095c735db44f)

### Changes in Dockerfiles

Because `micro` does not contain `microdnf` we need to handle the package installation in a builder stage and then copy the installed packages to the runtime image.
This is done using `microdnf --installroot` which installs packages into a chroot directory, and then the chroot contents are copied into the `ubi-micro` runtime stage.

#### Base image + operators

**Current base/Dockerfile (ubi9-minimal):**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS downloader
# ... download Tini ...

FROM registry.access.redhat.com/ubi9/ubi-minimal:latest

RUN microdnf install -y java-21-openjdk-headless openssl shadow-utils && \
    microdnf reinstall -y tzdata && \
    microdnf clean all -y

COPY --from=downloader /usr/bin/tini /usr/bin/tini
```

**Proposed base/Dockerfile (ubi10-micro):**
```dockerfile
# Install runtime dependencies into a chroot and download prerequisites
FROM registry.access.redhat.com/ubi10/ubi-minimal:latest AS builder

ARG JAVA_VERSION=21

RUN mkdir -p /mnt/rootfs && \
    microdnf install \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        --setopt=install_weak_deps=0 \
        --setopt=tsflags=nodocs \
        --releasever 10 \
        -y \
        java-${JAVA_VERSION}-openjdk-headless \
        openssl \
        bash \
        tzdata \
    && microdnf \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        clean all

# ... download Tini (unchanged) ...

# Distroless runtime
FROM registry.access.redhat.com/ubi10/ubi-micro:latest
COPY --from=builder /mnt/rootfs /
COPY --from=builder /usr/bin/tini /usr/bin/tini
```

#### Kafka images
Images that extend the base image (e.g., kafka) need an additional builder stage for their specific tools:

**Proposed kafka/Dockerfile — additional builder stage:**
```dockerfile
FROM registry.access.redhat.com/ubi10/ubi-minimal:latest AS kafka-tools

RUN mkdir -p /mnt/rootfs && \
    microdnf install \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        --setopt=install_weak_deps=0 \
        --setopt=tsflags=nodocs \
        --releasever 10 \
        -y \
        net-tools \
        hostname \
        findutils \
        tar \
        gzip \
        unzip \
        curl-minimal \
    && microdnf \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        clean all

FROM strimzi/base:latest
COPY --from=kafka-tools /mnt/rootfs /
# ... rest of Dockerfile unchanged ...
```

#### Maven builder

For `maven-builder` we use `registry.access.redhat.com/ubi9/openjdk-21:latest` as its base image.
`openjdk-21:latest` has similar CVE surface as `ubi9-minimal` so we will adopt there similar approach as for other images.
We will use `ubi10-micro` as a base and install `java-21-openjdk-headless` and other needed packages:

**Proposed maven-builder/Dockerfile**
```dockerfile
FROM registry.access.redhat.com/ubi10/ubi-minimal:latest AS builder

ARG JAVA_VERSION=21

RUN mkdir -p /mnt/rootfs && \
    microdnf install \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        --setopt=install_weak_deps=0 \
        --setopt=tsflags=nodocs \
        --releasever 10 \
        -y \
        java-${JAVA_VERSION}-openjdk-headless \
        maven \
        curl-minimal \
        bash \
        tzdata \
    && microdnf \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        clean all

FROM registry.access.redhat.com/ubi10/ubi-micro:latest

LABEL org.opencontainers.image.source='https://github.com/strimzi/strimzi-kafka-operator'

ARG strimzi_version

LABEL name='maven-builder' \
    vendor='Strimzi' \
    version="${strimzi_version}" \
    release="${strimzi_version}" \
    summary='Maven builder image of the Strimzi Kafka Operator.' \
    description='Builder image used to dynamically create Kafka Connect images with user-provided plugins.'

COPY --from=builder /mnt/rootfs /

RUN echo "strimzi:x:1001:0::/home/strimzi:/bin/false" >> /etc/passwd && \
    mkdir -p /home/strimzi && \
    chown 1001:0 /home/strimzi && \
    chmod 770 /home/strimzi

USER 1001
```

#### Buildah + Kaniko

For `buildah` and `kaniko` we do not build new images based on our base image, but we just retag existing upstream images.
Any changes planned as part of this proposal does not affect `buildah` or `kaniko` images that we use.

### Removing `shadow-utils`

The current images install `shadow-utils` to get the `useradd` command for creating non-root users during the build.
However, `shadow-utils` pulls in several dependencies and is never needed at runtime.
With `ubi10-micro` we can drop it entirely by writing user entries directly to `/etc/passwd`:

```dockerfile
# Before (with shadow-utils)
RUN useradd -r -m -u 1001 -g 0 kafka

# After (no shadow-utils needed)
RUN echo "kafka:x:1001:0::/home/kafka:/bin/false" >> /etc/passwd
```

This pattern is applied across all images that create users (`operator`, `kafka`, `maven-builder`).
For images that need a home directory (e.g., `maven-builder`), the directory is created and permissions set explicitly.

### Quay scan differences

We can compare scans from Quay.io for `1.1.0` images and the ones based on ubi10-micro (built on 4th July 2026).

- [1.1.0 images](https://quay.io/repository/strimzi/operator/manifest/sha256:92931ea0fad3380ea45a9b13ec61f717292e34967345566107e540432127f966?tab=vulnerabilities&fixable=true) - 247 vulnerabilities (15 fixable)
- [ubi10-micro based](https://quay.io/repository/jstejska/operator/manifest/sha256:e2069af869c8821cdc2eacd8df87aea3fd141d189539f0b3f0b8d1acf84759c7?tab=vulnerabilities&fixable=true) - 16 vulnerabilities (1 fixable)

### FIPS compliance

`ubi10-micro` inherits FIPS configuration from the host.
Containers share the host kernel, and on RHEL 9/10 with FIPS mode enabled, the container runtime (`podman`, `cri-o`) [automatically enables FIPS mode](https://access.redhat.com/solutions/3149581) for containers.
This works the same for all UBI variants (micro, minimal, standard) — we do not need to do any special configuration on our side.

### Testing

This is quite a big change that could behave differently on different clusters.
As a minimal set of testing I would consider the following:
- check that images for all architectures are working fine
- running all our systemtests workflows on GitHub Actions against all Kubernetes versions we support
- running all our systemtests against multiple OpenShift versions (I will be able to handle this)
- running all upgrade tests from previous released version to latest main

## Affected projects

All projects that produce images are affected:
- `strimzi-kafka-operator`
- `strimzi-kafka-bridge`
- `drain-cleaner`
- `test-clients`
- `test-container`
- `client-examples`
- `kafka-access-operator`
- `mqtt-bridge`

## Backwards compatibility

This proposal is fully backward compatible.

## Rejected alternatives

### Use ubi9-micro

We could use `ubi9-micro`, however, at some point we will anyway update to UBI 10 and there is no reason to not do it as part of this proposal.

### Project Hummingbird

[Project Hummingbird](https://hummingbird-project.io/) is a Red Hat project that produces [hardened container images](https://www.redhat.com/en/blog/red-hat-hardened-images) aiming for [near-zero CVEs](https://www.redhat.com/en/blog/chasing-holy-grail-why-red-hats-hummingbird-project-aims-near-zero-cves).
We could use their OpenJDK base image and add additional tools we require like `bash`.
The distroless variant has no shell at all; the `-builder` variant includes `bash` and `dnf`.

Hummingbird offers separate [FIPS variants](https://hummingbird-project.io/docs/using/overview/) (`:latest-fips`) that ship FIPS 140-3 validated crypto modules baked into the image.
These variants [enforce FIPS-approved algorithms even on non-FIPS hosts](https://gitlab.com/redhat/hummingbird/examples/-/blob/main/README.md?ref_type=heads#tags--variants), providing a consistent experience for developers who don't control the host infrastructure.
Full FIPS validation still [requires the host kernel to be in FIPS mode](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/switching-rhel-to-fips-mode_security-hardening) — same as with UBI.

However, images from Project Hummingbird are [supported only on `amd64` and `arm64`](https://hummingbird-project.io/docs/using/overview/) architectures which is not suitable for us as we also support `ppc64le` and `s390x` architectures.

This option can be revisited in the future once there will be more architectures in the support matrix.

### Wolfi base image

[Wolfi OS](https://edu.chainguard.dev/open-source/wolfi/overview/) is used as the base in images produced by [Chainguard](https://edu.chainguard.dev/chainguard/chainguard-images/overview/).
They also offer Strimzi hardened images.
Wolfi is an open source project [licensed under Apache 2.0](https://edu.chainguard.dev/open-source/wolfi/faq/) which is fine for us.
Chainguard describes their images as distroless, specially curated to run in cloud-native environments.

However, there are several differences that make it not suitable for us:
- the free tier only provides the `:latest` tag with no version pinning — if a new version introduces a breaking change, there is no way to stay on the previous version without a paid subscription
- it uses `apk` instead of `dnf`/`microdnf` so we would need to rewrite most of our Dockerfiles
- it is [supported only on `amd64` and `arm64`](https://edu.chainguard.dev/chainguard/chainguard-images/overview/#architecture/)

With these differences, I consider Wolfi as not suitable for Strimzi at this time.