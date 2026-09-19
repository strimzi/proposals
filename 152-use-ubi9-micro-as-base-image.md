# Use ubi9-micro as a base image with RPM-database-preserving multi-stage build

This proposal suggests changing our base image from `ubi9-minimal` to `ubi9-micro`, using a three-stage build pattern that preserves the RPM database in the final image.

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

To follow security standards and harden our images, Strimzi should migrate to `ubi9-micro`.
The key goal is switching from the `minimal` image to the `micro` variant.
The `micro` image does not contain a package manager and the security surface is much smaller than UBI's minimal version.

### Minimal vs Micro

The key difference is that `ubi-micro` excludes the package manager (`microdnf`) and all of its dependencies.
This makes it Red Hat's [distroless](https://www.redhat.com/en/blog/introduction-ubi-micro) container image — built from the same RHEL packages but without the packaging tools.

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

More details can be found in the following sources: [RHEL 9 — Types of container images](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/types-of-container-images), [Introduction to UBI Micro](https://www.redhat.com/en/blog/introduction-ubi-micro), [UBI 9 Micro catalog](https://catalog.redhat.com/en/software/containers/ubi9-micro/61832888ef29c53a494d2771)

### General migration approach

General approach to migrating from current base images to `ubi9-micro` is the same for every project within the Strimzi org.

Because `micro` does not contain `microdnf` we need to handle the package installation in a builder stage and then copy the installed packages to the runtime image.
The critical detail is that the RPM database must be preserved in the final image so that vulnerability scanners (e.g. Quay, Clair, Trivy) can correctly attribute packages to their CVE records.
This is achieved by a three-stage build:

1. Start with `ubi9-micro` and copy its entire rootfs (including its RPM database) as the `--installroot` for the builder stage.
2. Use `ubi9-minimal` as the builder: run `microdnf --installroot /mnt/rootfs` to install packages into that copy of the `ubi9-micro` rootfs.
   Because the RPM database was already present in the rootfs before installation, every installed package is recorded in it correctly.
3. Use a fresh `ubi9-micro` as the final stage and copy `/mnt/rootfs` into it.
   The final image has no package manager but retains a fully accurate RPM database for scanner queries.

The mechanism will then consist of a multi-stage build, structured as follows:

- use `ubi9-micro` as a named stage (`base`) — this establishes the clean rootfs with RPM database
- use `ubi9-minimal` as a builder stage — copy the `base` rootfs to `/mnt/rootfs`, then install all required packages with `--installroot /mnt/rootfs --releasever 9`
- download tini (if used) directly into `/mnt/rootfs/usr/bin` in the builder stage
- use `ubi9-micro` again as the final image — copy `/mnt/rootfs` in

The approach may vary based on specific requirements of each project, but the core approach will remain the same.

### Detailed examples for `strimzi-kafka-operator`

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

**Proposed base/Dockerfile (ubi9-micro, 3-stage):**
```dockerfile
#####
# Stage 1: Establish clean ubi9-micro rootfs with its own RPM database
#####
FROM registry.access.redhat.com/ubi9/ubi-micro:latest AS base

#####
# Stage 2: Install runtime dependencies into the ubi9-micro rootfs using dnf --installroot
#####
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS builder

ARG JAVA_VERSION=21
ARG TARGETOS
ARG TARGETARCH

# Copy the ubi9-micro rootfs (including its RPM database) as the install root
COPY --from=base / /mnt/rootfs/

RUN microdnf install \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        --setopt=install_weak_deps=0 \
        --setopt=tsflags=nodocs \
        --releasever 9 \
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

# Download Tini directly into the rootfs
RUN curl -s -L https://github.com/krallin/tini/releases/download/... -o /mnt/rootfs/usr/bin/tini && \
    chmod +x /mnt/rootfs/usr/bin/tini

#####
# Stage 3: Build the final container image on ubi9-micro base
#####
FROM registry.access.redhat.com/ubi9/ubi-micro:latest

COPY --from=builder /mnt/rootfs /
```

#### Kafka images

Images that extend the base image (e.g., kafka) need an additional builder stage for their specific tools.
The same three-stage pattern is applied — a dedicated `ubi9-micro` stage seeds the RPM database, and a `ubi9-minimal` installer stage runs `--installroot`:

**Proposed kafka/Dockerfile — additional builder stage:**
```dockerfile
#####
# Stage 1: Establish clean ubi9-micro rootfs with its own RPM database
#####
FROM registry.access.redhat.com/ubi9/ubi-micro:latest AS kafka-base

#####
# Install Kafka-specific runtime tools into the ubi9-micro rootfs using dnf --installroot
#####
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS kafka-tools

# Copy the ubi9-micro rootfs (including its RPM database) as the install root
COPY --from=kafka-base / /mnt/rootfs/

RUN microdnf install \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        --setopt=install_weak_deps=0 \
        --setopt=tsflags=nodocs \
        --releasever 9 \
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
We will use `ubi9-micro` as a base and install `java-21-openjdk-headless` and other needed packages using the same three-stage pattern:

**Proposed maven-builder/Dockerfile**
```dockerfile
#####
# Stage 1: Establish clean ubi9-micro rootfs with its own RPM database
#####
FROM registry.access.redhat.com/ubi9/ubi-micro:latest AS base

#####
# Stage 2: Install runtime dependencies into the ubi9-micro rootfs using dnf --installroot
#####
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS builder

ARG JAVA_VERSION=21

COPY --from=base / /mnt/rootfs/

RUN microdnf install \
        --installroot /mnt/rootfs \
        --noplugins \
        --config /etc/dnf/dnf.conf \
        --setopt=cachedir=/var/cache/microdnf \
        --setopt=reposdir=/etc/yum.repos.d \
        --setopt=varsdir=/etc/dnf \
        --setopt=install_weak_deps=0 \
        --setopt=tsflags=nodocs \
        --releasever 9 \
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

#####
# Stage 3: Build the final container image on ubi9-micro base
#####
FROM registry.access.redhat.com/ubi9/ubi-micro:latest

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
Any changes planned as part of this proposal do not affect `buildah` or `kaniko` images that we use.

### Removing `shadow-utils`

The current images install `shadow-utils` to get the `useradd` command for creating non-root users during the build.
However, `shadow-utils` pulls in several dependencies and is never needed at runtime.
With `ubi9-micro` we can drop it entirely by writing user entries directly to `/etc/passwd`:

```dockerfile
# Before (with shadow-utils)
RUN useradd -r -m -u 1001 -g 0 kafka

# After (no shadow-utils needed)
RUN echo "kafka:x:1001:0::/home/kafka:/bin/false" >> /etc/passwd
```

This pattern is applied across all images that create users (`operator`, `kafka`, `maven-builder`).
For images that need a home directory (e.g., `maven-builder`), the directory is created and permissions set explicitly.

### Quay scan differences

We can compare scans from Quay.io for `1.1.0` images and the ones based on ubi9-micro (built on 4th July 2026).

- [1.2.0 images](https://quay.io/repository/strimzi/operator/manifest/sha256:6df3bf9f92d3d1907aca08ade8c6df6cdacd2e235756afad419ad582ce6a2c4e?tab=vulnerabilities) - 312 vulnerabilities (39 fixable)
- [ubi9-micro based](https://quay.io/repository/jstejska/operator/manifest/sha256:ad614488cc0643b66c081c806081d9a254382e40c02585742400f1da52197270?tab=vulnerabilities) - 166 vulnerabilities (1 fixable)

### FIPS compliance

`ubi9-micro` inherits FIPS configuration from the host.
Containers share the host kernel, and on RHEL 9 with FIPS mode enabled, the container runtime (`podman`, `cri-o`) [automatically enables FIPS mode](https://access.redhat.com/solutions/3149581) for containers.
This works the same for all UBI variants (micro, minimal, standard) — we do not need to do any special configuration on our side.

### Testing

This is quite a big change that could behave differently on different clusters.
As a minimal set of testing I would consider the following:
- check that images for all architectures are working fine
- running all our systemtests workflows on GitHub Actions against all Kubernetes versions we support
- running all our systemtests against multiple OpenShift versions (I will be able to handle this)
- running all upgrade tests from previous released version to latest main

## Affected projects

This proposal covers mostly the `strimzi-kafka-operator` repository with examples and testing strategy.
All other projects that produce container images can use the same strategy to migrate from current base images to `ubi9-micro`.

The projects within Strimzi that produce images are:
- `strimzi-kafka-operator`
- `strimzi-kafka-bridge`
- `drain-cleaner`
- `test-clients`
- `test-container`
- `client-examples`
- `kafka-access-operator`
- `mqtt-bridge`

However, it should be evaluated if it makes sense to use `ubi9-micro` in testing projects like `test-clients`, `test-container`, or in `client-examples`.

## Backwards compatibility

This proposal is fully backward compatible.

## Rejected alternatives

### UBI10-micro

UBI10 was evaluated as the initial target for this migration.
It was rejected because RHEL 10 raises the minimum hardware baseline on three of the four architectures we ship.
RHEL 9 requires [x86-64-v2, ARMv8.0-A, POWER9 and z14](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/considerations_in_adopting_rhel_9/ref_architectures_considerations-in-adopting-rhel-9), while RHEL 10 requires [x86-64-v3, ARMv8.0-A, POWER10 and z15](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/considerations_in_adopting_rhel_10/architectures).

The baseline applies to the container image, not only to the host operating system.
Red Hat's [container compatibility policy](https://access.redhat.com/support/policy/rhel-container-compatibility) states that the container host's hardware must meet the minimum hardware requirements of the image, using `RHEL 10 container images for x86_64 require x86-64-v3` as its own example, and classifies a RHEL 10 image on a RHEL 9 host as a "Workload Specific" configuration rather than a fully compatible one.

OpenShift has not followed RHEL 10 yet.
It still documents its minimum instruction set architectures as [x86-64-v2, ARMv8.0-A, Power 9 and z14](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_any_platform/installing-platform-agnostic), and in [OpenShift 4.22](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/release_notes/ocp-4-22-release-notes) RHCOS is based on RHEL 9.8 packages, with RHCOS 10.2 offered only as a Technology Preview.
Moving to UBI10 would make Strimzi unusable on clusters that OpenShift itself still fully supports.

The failure is not a graceful degradation.
A UBI10 image on a CPU below the baseline aborts during glibc initialisation with `Fatal glibc error: CPU does not support x86-64-v3` and ends up in `CrashLoopBackOff`.
We cannot avoid this by static linking, because all our images run a JVM dynamically linked against glibc.
The [Percona MongoDB operator](https://github.com/percona/percona-server-mongodb-operator/issues/2495) already hit this after the same migration, and the only workaround offered to affected users was to build a custom UBI9-based image.

### Project Hummingbird and RedHat Hardened Images

[Project Hummingbird](https://hummingbird-project.io/) is a Red Hat project that produces [hardened container images](https://www.redhat.com/en/blog/red-hat-hardened-images) aiming for [near-zero CVEs](https://www.redhat.com/en/blog/chasing-holy-grail-why-red-hats-hummingbird-project-aims-near-zero-cves).
We could use their OpenJDK base image and add additional tools we require like `bash`.
The distroless variant has no shell at all; the `-builder` variant includes `bash` and `dnf`.

Hummingbird offers separate [FIPS variants](https://hummingbird-project.io/docs/using/overview/) (`:latest-fips`) that ship FIPS 140-3 validated crypto modules baked into the image.
These variants [enforce FIPS-approved algorithms even on non-FIPS hosts](https://gitlab.com/redhat/hummingbird/examples/-/blob/main/README.md?ref_type=heads#tags--variants), providing a consistent experience for developers who don't control the host infrastructure.
Full FIPS validation still [requires the host kernel to be in FIPS mode](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/switching-rhel-to-fips-mode_security-hardening) — same as with UBI.

However, images from Project Hummingbird are [supported only on `amd64` and `arm64`](https://hummingbird-project.io/docs/using/overview/) architectures which is not suitable for us as we also support `ppc64le` and `s390x` architectures.
RedHat variant of Hummingbird images - Hardened Images (`hi`) does not support `ppc64le` and `s390x` architectures as well.

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