# Deprecate and remove JMX Trans

## Current situation

[JMX Trans](https://github.com/jmxtrans/jmxtrans) is a tool which allows data collection from JMX endpoints of Java applications to send them to other applications and services.
Strimzi integrates JMX Trans as part of the `Kafka` custom resource.
You can configure it in `.spec.jmxTrans` section:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  # ...
  jmxTrans:
    outputDefinitions:
      - outputType: "com.googlecode.jmxtrans.model.output.StdOutWriter"
        name: "standardOut"
      - outputType: "com.googlecode.jmxtrans.model.output.GraphiteOutputWriter"
        host: "mylogstash.com"
        port: 31028
        flushDelayInSeconds: 5
        name: "logstash"
    kafkaQueries:
      - targetMBean: "kafka.server:type=BrokerTopicMetrics,name=*"
        attributes:  ["Count"]
        outputs: ["standardOut"]
  # ...
```

## Issues

The JMX Trans tool seems to be stale.
The last release is from March 31st 2021 - so more than year and a half ago.
Because the last release is so old, there are also many CVEs in the different dependencies it uses.
In addition to that, it is falling behind in other aspects as well.
For example, while we are moving all our container images to Java 17, JMX Trans does not run on Java 17, so it needs to stick with Java 11.

## Proposal

Since the JMX Trans project does not seem to be developed anymore, we should first deprecate the JMX Trans support and if nothing changes, we should remove the support.

In the first phase - as part of Strimzi 0.33 (currently expected in the second part of December or early January) - we will:
* Deprecate the `.spec.jmxTrans` API in the Kafka custom resource.
* Update the docs to indicate that JMX Trans is deprecated.
* Announce the deprecation to the users as part of the Strimzi 0.33 communication (release notes, change log etc.).

In the second phase - as part of Strimzi 0.35 (currently expected to be release in March or April 2023) - we will:
* Retain the `.spec.jmxTrans` API in the Kafka custom resource.
  But it will stay deprecated and will be ignored by the operator (a warning will be issued if it is present in the custom resource).
* The Strimzi 0.35 and later will check for existence of the JMX Trans resources and delete them if they would exist.
  The rest of the operator code related to JMX Trans will be removed.
* The container image for JMX Trans will be removed.
* Remove JMX Trans from the docs.

In the final phase - as part of Strimzi 0.40 - we will:
* Completely remove the operator functionality which checks for the JMX Trans resources and delete them.
  Anyone who upgrades from Strimzi 0.34 or earlier to Strimzi 0.40 or later and had enabled JMX Trans will have to delete the resources manually.

The `.spec.jmxTrans` API in the Kafka custom resource will be removed in the next version of the API (either `v1` or `v1beta3`) as it cannot be removed earlier for backwards compatibility reasons.

If the JMX Trans project happens to revived between the initial phase and the second phase - either as the original project or as a fork - we can un-deprecate the API and keep supporting it.

## Affected/not affected projects

This proposal affects only the Strimzi Cluster Operator and its Kafka cluster with enabled JMX Trans.
Any other users or projects are not affected.

## Compatibility

This proposal suggests to deprecate and remove a currently supported feature.
So all its users will be affected.
Any other users - not using JMX Trans - will not be affected by this.

To maintain backwards compatibility of the Custom Resource Definitions and the API they provide, the `.spec.jmxTrans` object will be still part of the API and will not be removed, only deprecated.
But it will be ignored by the operator.

## Rejected alternatives

### Updating JMX Trans

We could try to help to contribute to the JMX Trans project or try to fork it and maintain it our self.
However, this alternative was rejected because we do not have the resources to do this ourselves.
