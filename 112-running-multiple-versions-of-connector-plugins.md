# Running multiple versions of Connector plugins

In Apache Kafka 4.1.0 onwards it's possible to run multiple versions of the same Connector plugin on the same Connect cluster.
Support for this was added via [KIP-891][kip_891].

Strimzi should support this feature as part of the `KafkaConnector` CR.

## Current situation

Today Strimzi provides the `KafkaConnector` CR which looks something like:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: my-source-connector
  labels:
    strimzi.io/cluster: my-connect-cluster
spec:
  class: org.apache.kafka.connect.file.FileStreamSourceConnector
  tasksMax: 2
  config:
    file: "/opt/kafka/LICENSE"
    topic: my-topic
```

If there are multiple versions of the same connector loaded into the Connect cluster then Connect chooses the version to use.
This is based on the [Maven Version Order](https://maven.apache.org/pom.html#Version_Order_Specification) and the latest version based on that is used.
There are a few other rules which are listed in [KIP-891][kip_891].

## Motivation

Once Strimzi supports Apache Kafka version 4.1.0, without any changes users will be able to select the version of the connector they want by specifying `connector.plugin.version` in the `config` section of the `KafkaConnector` CR.
However, since the versioning is normally important to people managing workloads, it makes sense that version should be part of the main `KafkaConnector` API, rather than only supported via the `config` section.

## Proposal

The `KafkaConnector` CR will be updated to have a `version` field:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: my-source-connector
  labels:
    strimzi.io/cluster: my-connect-cluster
spec:
  class: org.apache.kafka.connect.file.FileStreamSourceConnector
  tasksMax: 2
  version: "[2.6.1,)" # New field
  config:
    file: "/opt/kafka/LICENSE"
    topic: my-topic
```

This field will be optional and when set Strimzi will add `connector.plugin.version` to the Connector configuration before creating or updating it.
The type of the `version` field will be String, allowing the user to pass ranges like `[2.6.1,)`.

### MirrorMaker2

The `version` field will also be added to the `KafkaMirrorMaker2` CR and status:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: my-mirror-maker-2
spec:
  version: 4.1.0
  replicas: 1
  connectCluster: "cluster-b"
  clusters:
    #...
  mirrors:
  - sourceCluster: "cluster-a"
    targetCluster: "cluster-b"
    sourceConnector:
      tasksMax: 1
      version: "[2.6.1,)" # New field
      config:
        replication.factor: -1
        offset-syncs.topic.replication.factor: -1
        sync.topic.acls.enabled: "false"
        refresh.topics.interval.seconds: 600
```

## Affected/not affected projects

This only affects the cluster operator

## Compatibility

It is expected that this change will land in a version of Strimzi that supports both Apache Kafka 4.0.0 and 4.1.0.
This means there are two compatibility problems to address:
* Users who are already specifying `connector.plugin.version` via the `config` field.
* Users who are using an older version of Kafka Connect than 4.1.0

### Existing use of `connector.plugin.version`

In the first release of Strimzi that supports this feature `connector.plugin.version` will not be added to the forbidden list for configurations specified under `spec.config`.
Instead, if `connector.plugin.version` is set under `config` Strimzi will leave it there and add a warning to the KafkaConnector status.
We will also add a note to the release notes that in future setting `connector.plugin.version` will not be allowed.
If both `version` and `config.connector.plugin.version` are set, then `version` will take precedence.

In the following release of Strimzi, `connector.plugin.version` will be added to the forbidden list.
This allows time for users to migrate.
Allowing one release seems reasonable since it is a small change to make in the CR and the multiple version feature in Apache Kafka is still quite new so many people won't have started using it yet.

### Using older Kafka Connect versions
If the `connector.plugin.version` is sent as part of the connector configuration to an older version of Connect than 4.1.0 no error occurs but the version is chosen using the old Maven ordering process.
Since the version of the connector is only included in the status in Connect 4.1.0 onwards the `KafkaConnector` CR status will not contain the version in this case.
Strimzi will not perform any validation to ensure the `version` field is used alongside a Connect version that supports it.
Instead, we will add a note to the field description in the CRD that notes it only works for 4.1.0 onwards.
However, the ignored field is still listed in the status listing the connector config, so to reduce confusion for users if the user sets `version` on a `KafkaConnector` CR and the related `KafkaConnect` CR is not running version 4.1.0 or higher, Strimzi will add a warning to the status.

## Rejected alternatives

### Users specify version via config

We could leave the support as it is today and allow users to simply specify the version via the config.
However, providing a specific `version` field is a useful UX feature, similar to the `tasksMax` field, and the code change and maintenance overhead is small.

### Validate the Connect version

A previous version of this proposal suggested adding a warning to the `KafkaConnector` status if a user set `version` on a `KafkaConnector` CR that was for a Connect cluster not running version 4.1.0 or higher.
However, due to the way that the code in Strimzi is structured (we don't currently have access to the `KafkaConnect` object when reviewing the `KafkaConnector` config) this would require passing a new 
variable through multiple methods, or require validating the configuration at a strange point in the codebase.
Given that this code would be removed anyway once 4.0.0 is no longer supported, and the lack of the `version` in the `KafkaConnector` status is a good indicator of whether the version is being used 
I believe this is unnecessary.

[kip_891]: https://cwiki.apache.org/confluence/display/KAFKA/KIP-891:+Running+multiple+versions+of+Connector+plugins