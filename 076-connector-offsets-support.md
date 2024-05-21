# Connector Offsets Support

Update the KafkaConnector resource to allow users to manage offsets

## Current situation

[KIP-875][kip] added first class offsets support to Kafka Connect.
This was done via three new endpoints:
* GET /connectors/{connector}/offsets
* PATCH /connectors/{connector}/offsets
* DELETE /connectors/{connector}/offsets
* PUT /connectors/{connector}/stop

Support for the "stop" endpoint was added via [PR 9095](https://github.com/strimzi/strimzi-kafka-operator/pull/9095).
Currently users that are utilising the KafkaConnector resource cannot make use of the API endpoints for managing offsets.

## Motivation

Previously if users wanted to list, update or delete offsets for a connector they had to interact with the relevant Kafka topics directly.
[KIP-875][kip] adds new endpoints to let the user do this more easily.
There are multiple reasons a user might want to manage offsets, for example (as listed in the KIP):
* Resetting the offsets for a connector while iterating rapidly in a development environment (so that the connector does not have to be renamed each time)
* Viewing the in-memory offsets for source connectors on the cluster in order to recover from accidental deletion of the Connect source offsets topic (this is currently possible with the config topic, and can be a life-saver)
* Monitoring the progress of running connectors on a per-source-partition basis 
* Skipping records that cause issues with a connector and cannot be addressed using existing error-handling features

## Proposal

Strimzi should include mechanisms to let users manage offsets when utilising the KafkaConnector resources.
This proposal suggests different approaches for each action, since they are all slightly different.
In summary:
 - listing offsets via the KafkaConnector status and enabled/disabled through a property
 - altering offsets via a new resource called KafkaConnectorOffsets
 - deleting offsets via an annotation on the KafkaConnector resource

### Request/Response from the API endpoints

The request/response used when interacting with the Kafka Connect API is different for source and sink connectors.
This is because sink connectors use regular Kafka offsets, whereas source connectors determine their own structure for offsets.
This difference is reflected in the approaches outlined in this proposal.

Source connector format is:
```
{
  "offsets": [
    {
      "partition": {
        // Connector-defined source partition
      },
      "offset": {
        // Connector-defined source offset
      }
    }
  ]
}
```

Sink connector format is:
```
{
  "offsets": [
    {
      "partition": {
        "kafka_topic": // Kafka topic
        "kafka_partition": // Kafka partition
      },
      "offset": {
        "kafka_offset": // Kafka offset
      }
    }
  ]
}
```

### Listing offsets
The KafkaConnector resource will be updated with a new property to enable the current offsets to be listed in the resource status.

The new property will be a boolean property called `showOffsets` and default to `false`. It should default to `false` because enabling it requires the operator to do an additional API call every time it reconciles the connector.
If there are a lot of connectors running in the cluster this is a lot of additional API calls.

When the property is set to `true` the status will include the offsets.

The details in the status will be similar to the format returned by the Connect API:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
#...
status:
  #...
  connectorOffsets:
    lastUpdateTime: "2024-05-21T09:08:58.467712198Z"
    source:
      - partition: "{\"connector-determined-partition\": \"1\"}" # value is a String containing JSON as received from the API
        offset: "{\"connector-determined-offset\": \"1\"}" # value is a String containing JSON as received from the API
    sink:
      - partition:
          kafkaTopic: "my-topic"
          kafkaPartition: 1
        offset:
          kafkaOffset: 0
```

Notes:
* Only one of the `source` and `sink` properties will be present.
* The operator will update the `lastUpdateTime` whenever it updates the offsets section of the status.
* If the `showOffsets` property is set to `false` the operator will leave the last observed offsets in the status, keeping the `lastUpdateTime` so the user sees the offsets list has not been updated.

### Altering offsets
To alter offsets a new `KafkaConnectorOffsets` custom resource will be added.
This is similar to the approach used for Cruise Control with `KafkaRebalance`.
This approach is used because the request requires a complex object to specify the desired offsets.

The new resource should look like:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnectorOffsets
metadata:
  name: my-source-connector-offsets
  labels:
    # The strimzi.io/cluster label identifies the KafkaConnect instance
    # in which this connector is running
    strimzi.io/cluster: my-connect-cluster
spec:
  connectorName: my-source-connector
  sourceOffsets:
    - partition: "{\"connector-determined-partition\": \"1\"}" # value is a String containing JSON can be directly passed to the API
      offset: "{\"connector-determined-offset\": \"1\"}" # value is a String containing JSON can be directly passed to the API
  sinkOffsets:
    - partition:
        kafkaTopic: "my-topic"
        kafkaPartition: 1
      offset:
        kafkaOffset: 0
```

Notes:
* Only one of `sourceOffsets` and `sinkOffsets` should be specified.
* The format matches the format shown in the status when listing offsets so it can be copy pasted here.
* Both the `offset` properties are optional since the API allows these to be passed as `null` (this resets the offsets for that partition).
* When the offset patch has been completed the status will have a `Ready` condition of `true`.
* If the request fails the status will include the response from the API.
* Strimzi will not attempt to verify that the connector is already stopped, it will rely on the API failing if that is the case.
  This is to simplify the logic and means we don't have to account for what happens if the KafkaConnector spec has been updated but the connector isn't stopped yet.

### Resetting offsets

To reset the offsets for a particular connector the user will annotate the KafkaConnector resource with a new annotation: `strimzi.io/offsets-reset=true`.
When the reset has been completed the operator will remove this annotation from the resource.

If the reset fails the operator will add a condition to the KafkaConnector resource called `strimzi.io/offsets-reset` to communicate the failure and remove the annotation.

Strimzi will shortcut and automatically fail to do the reset if the KafkaConnector resource is not stopped.
The user can update the KafkaConnector to stop the connector and reset the offsets at the same time.
In that case the operator will first stop the connector, and once that API call returns, then it will make the call to reset the offsets.

## Affected/not affected projects

This only affects the Connect parts of the cluster-operator.

## Compatibility

If a `KafkaConnectorOffsets` resource is created against a Kafka Connect version that does not support the APIs, the API calls will fail and the operator will put a suitable message in the status to indicate what has happened.

## Rejected alternatives

### Listing offsets by default
This would result in a lot of additional API calls

### Altering offsets using the KafkaConnector resource
We could alter the offsets either using an annotation or spec change in the KafkaConnector resource.

Using an annotation on its own wouldn't really be feasible since the body of the request might be very complex.

We could use the spec described for `KafkaConnectorOffsets`, combined with an annotation to trigger the reset.
This has the benefit of not needing a new resource type, however, it means polluting the `KafkaConnector` resource with offset properties.
Having separate resources means users can also provide different permissions for the two kinds of resource.

[kip]: https://cwiki.apache.org/confluence/display/KAFKA/KIP-875%3A+First-class+offsets+support+in+Kafka+Connect
