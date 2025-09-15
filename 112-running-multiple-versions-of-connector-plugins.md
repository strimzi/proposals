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
  version: 4.1.0 # New field
  config:
    file: "/opt/kafka/LICENSE"
    topic: my-topic
```

This field will be optional and when set Strimzi will add `connector.plugin.version` to the Connector configuration before creating or updating it.
If the user also specifies `config.connector.plugin.version`, the version directly under `spec` will take precedence.
The type of the `version` field will be String, allowing the user to pass ranges like `[2.6.1,)`.

### CR status

When `version` is set Strimzi will also add the version to the `KafkaConnector` status:

```yaml
status:
  conditions:
    #...
  connectorStatus:
    #...
  observedGeneration: 1
  tasksMax: 2
  version: 4.1.0 # New field
  topics:
  - my-topic

```

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
      version: 4.1.0 # New field
      config:
        replication.factor: -1
        offset-syncs.topic.replication.factor: -1
        sync.topic.acls.enabled: "false"
        refresh.topics.interval.seconds: 600
```

## Affected/not affected projects

This only affects the cluster operator

## Compatibility

If the `connector.plugin.version` is specified for a connector running on an older version of Connect than 4.1.0 no error occurs but the version is chosen using the old Maven ordering process.
However, the ignored field is still listed in the status listing the connector config, so to reduce confusion for users if the user sets `version` on a `KafkaConnector` CR and the related `KafkaConnect` CR is not running version 4.1.0 or higher, Strimzi will add a warning to the status.

## Rejected alternatives

### Users specify version via config

We could leave the support as it is today and allow users to simply specify the version via the config.
However, providing a specific `version` field is a useful UX feature, similar to the `tasksMax` field, and the code change and maintenance overhead is small.


[kip_891]: https://cwiki.apache.org/confluence/display/KAFKA/KIP-891:+Running+multiple+versions+of+Connector+plugins