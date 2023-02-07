# Enhance Image Extension for Custom Authentication

## Introduction

Strimzi provides an automated and managed way of running a Kafka cluster, but some teams still prefer hosting their Kafka cluster using cloud providers. The current `kafka-base` image provided by Strimzi can be easily extended for custom JARs, but the generated configuration scripts make it difficult to add custom configuration properties for those JARs. This proposal aims to enhance the image extension by allowing teams to easily add custom JARs and provide custom configuration properties for those JARs.

## Problem Statement

The current configuration generation scripts in the `kafka-base` image prevent overriding the `security.protocol` and adding custom properties without also overriding the entire generate script. This makes it difficult to configure a custom jar and authenticate with a Kafka cluster hosted by a cloud provider.

## Proposed Solution

The proposed solution is to update the generate scripts to interpolate a `<COMPONENT>_ADDITIONAL_CONFIGURATION` environment variable and append its value (a multi-line string of ini-formatted properties) to the bottom of the properties file. This way, the properties supplied by this environment variable would not be overridden by Strimzi. 

## Affected Components

The following components would be affected by this proposal:
- Kafka Connect
- MirrorMaker2
- MirrorMakerProducer
- MirrorMakerConsumer
- Kafka Bridge

## Rejected Alternatives

A `.spec.authentication.mechanism.custom` option has been proposed as a way to allow authentication overrides, but this proposal focuses on solving the problem through image extension rather than the CRD. This approach would allow the Strimzi project to focus on Strimzi Kafka and simply provide a valid extension path.
