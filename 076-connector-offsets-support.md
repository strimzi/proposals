# Connector Offsets Support

Update the KafkaConnector resource to allow users to manage offsets

## Current situation

[KIP-875][kip] added first class offsets support to Kafka Connect.
This was done via four new endpoints:
* GET /connectors/{connector}/offsets
* PATCH /connectors/{connector}/offsets
* DELETE /connectors/{connector}/offsets
* PUT /connectors/{connector}/stop

Support for the "stop" endpoint was added via [PR 9095](https://github.com/strimzi/strimzi-kafka-operator/pull/9095).
Currently users that are utilising the KafkaConnector resource cannot make use of the API endpoints for managing offsets.

## Motivation

Previously if users wanted to list, update or delete offsets for a connector they had to interact with the relevant consumer groups for sink connectors or directly write to the Kafka Connect internal offsets topic for source connectors.
[KIP-875][kip] adds new endpoints to let the user do this more easily.
There are multiple reasons a user might want to manage offsets, for example (as listed in the KIP):
* Resetting the offsets for a connector while iterating rapidly in a development environment (so that the connector does not have to be renamed each time)
* Viewing the in-memory offsets for source connectors on the cluster in order to recover from accidental deletion of the Connect source offsets topic (this is currently possible with the config topic, and can be a life-saver)
* Monitoring the progress of running connectors on a per-source-partition basis 
* Skipping records that cause issues with a connector and cannot be addressed using existing error-handling features

## Proposal

Strimzi should include mechanisms to let users manage offsets when utilising the KafkaConnector resources.
This proposal suggests introducing a new annotation on the KafkaConnector resource to trigger each operation.
The operations are:
* List offsets.
* Alter offsets.
* Reset offsets.

The list offsets operation uses a ConfigMap for the output.
The alter offsets operation uses a ConfigMap for the input.

### Request/Response from the API endpoints

The response body received when using the GET endpoint, and the request body used when calling the PATCH endpoint have identical structures to allow users to easily fetch and then alter offsets.
The structure of the body is different for source and sink connectors.
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

### New annotation

All three actions will make use of a new annotation called `strimzi.io/connectorOffsets`.
The possible values will be `list`, `alter` and `reset`.

### Listing offsets

To list the current offsets the user will annotate the KafkaConnector resource with the new annotation `strimzi.io/connectorOffsets` set to `list`.
After the annotation is added, on the next reconciliation the operator will fetch the current offsets and create a Kubernetes ConfigMap containing the JSON response.
The operator will add an owner reference to the Kubernetes ConfigMap pointing to the KafkaConnector resource that was annotated.
If the ConfigMap already exists the operator will replace the data with the updated data and not add an owner reference.
The operator will then remove the annotation from the KafkaConnector CR.

If the user wants to see an updated set of offsets they will need to re-annotate the resource.

The name of the ConfigMap the operator creates/updates will be set by the user in the KafkaConnector CR.
The KafkaConnector CRD will be updated to contain a new property called `listOffsets` that must be set for the `strimzi.io/connectorOffsets=list` annotation to take effect.
The structure of the `listOffsets` property will be as below, to allow it to be extended in future if required:

```yaml
listOffsets:
  toConfigMap:
    name: my-connector-offsets
```

The ConfigMap will contain a single entry called `offsets.json` that contains the response received from the GET /connectors/{connector}/offsets endpoint.

For example for a source connector the ConfigMap might look like:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-source-connector-offsets
data:
  offsets.json: |
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

For a sink connector it might look like:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-sink-connector-offsets
data:
  offsets.json: |
    {
      "offsets": [
        {
          "partition": {
            "kafka_topic": "my-topic"
              "kafka_partition": 2
          },
          "offset": {
            "kafka_offset": 4
          }
        }
      ]
    }
```

Notes:
* If the `listOffsets` property is missing from the KafkaConnector CR when the `strimzi.io/connectorOffsets=list` annotation is added the operator will update the KafkaConnector CR status to include a warning message:
  ```yaml
  - lastTransitionTime: "2024-06-04T08:44:15.913138115Z"
    message: Failed to list the connector offsets due to missing property listOffsets in KafkaConnector CR.
    reason: ListOffsets
    status: "True"
    type: Warning
  ```
  The operator will leave the annotation on the KafkaConnector resource until either the list operation succeeds, or the user removes the annotation.
  This means the operator will retry the list on every reconciliation, allowing the condition to remain present for the user to see.

### Altering offsets

To alter offsets the user will annotate the KafkaConnector resource with the new annotation `strimzi.io/connectorOffsets` set to `alter`.
After the annotation is added, on the next reconciliation the operator will call the PATCH /connectors/{connector}/offsets endpoint to alter the offsets.
Once the patch is complete the operator will then remove the annotation from the KafkaConnector CR.

The operator will read in the entry called `offsets.json` from the ConfigMap.
The entry name matches the entry name used in the list operation, so that a user can list offsets, then edit the ConfigMap the operator created, then request the offsets be altered using that same ConfigMap.

The name of the ConfigMap the operator will use to construct the request body will be set by the user in the KafkaConnector CR.
The KafkaConnector CRD will be updated to contain a new property called `alterOffsets` that must be set for the `strimzi.io/connectorOffsets=alter` annotation to take effect.
The structure of the `alterOffsets` property will be as below, to allow it to be extended in future if required:

```yaml
alterOffsets:
  fromConfigMap:
    name: my-connector-offsets
```

Notes:
* The format of the request body will be validated to make sure it is valid JSON, but the operator won't perform any further validation, instead relying on the API endpoint to return a reasonable error message.
* If the request to the Connect API fails the operator will add a condition to the KafkaConnector CR status with a warning message that includes the response from the API, e.g.
  ```yaml
  - lastTransitionTime: "2024-06-04T08:44:15.913138115Z"
    message: Failed to alter the connector offsets due to "message from endpoint".
    reason: AlterOffsets
    status: "True"
    type: Warning
  ```
  The operator will leave the annotation on the KafkaConnector resource until either the patch operation succeeds, or the user removes the annotation.
  This means the operator will retry the patch on every reconciliation, allowing the condition to remain present for the user to see.
* Strimzi will shortcut and automatically fail to alter the offsets if the KafkaConnector resource does not have it's `state` set as `stopped`.
  Similar to if Connect returns on error, the operator will add a condition stating that the operation has failed because the connector is not stopped and therefore offsets cannot be altered, and leave the annotation on the resource.
  The user can update the KafkaConnector to stop the connector and alter the offsets at the same time.
  In that case the operator will first stop the connector, and once that API call returns, then it will make the call to alter the offsets.

### Resetting offsets

To reset the offsets for a particular connector the user will annotate the KafkaConnector resource with the new annotation `strimzi.io/connectorOffsets` set to `reset`.
When the operator observes this annotation it will use the DELETE /connectors/{connector}/offsets endpoint to reset all offsets for the connector.
When the reset has been completed the operator will remove this annotation from the resource.

If the request to the Connect API fails the operator will add a condition to the KafkaConnector CR status with a warning message that includes the response from the API, e.g.
  ```yaml
  - lastTransitionTime: "2024-06-04T08:44:15.913138115Z"
    message: Failed to alter the connector offsets due to "message from endpoint".
    reason: ResetOffsets
    status: "True"
    type: Warning
  ```
The operator will leave the annotation on the KafkaConnector resource until either the delete operation succeeds, or the user removes the annotation.
This means the operator will retry the delete on every operation, allowing the condition to remain present for the user to see.

Strimzi will shortcut and automatically fail to do the reset if the KafkaConnector resource does not have it's `state` set as `stopped`.
Similar to if Connect returns on error, the operator will add a condition stating that the operation has failed because the connector is not stopped and therefore offsets cannot be reset, and leave the annotation on the resource.
The user can update the KafkaConnector to stop the connector and reset the offsets at the same time.
In that case the operator will first stop the connector, and once that API call returns, then it will make the call to reset the offsets.

## Affected/not affected projects

This proposal only affects the Connect parts of the cluster-operator.
This proposal does not apply to the use of the KafkaMirrorMaker2 Custom Resource.
If users are using a KafkaConnector Custom Resource to manage the Mirror Maker connectors, they can manage offsets using the method described in this proposal.
We should have a different proposal to cover how we might support managing offsets for Mirror Maker connectors that are managed via the KafkaMirrorMaker2 Custom Resource.
This is because the structure of the offsets JSON object for these connectors is known, so Strimzi can provide a more guided experience for altering offsets.
For example, by allowing the user to specify the value of specific fields in the offset JSON, rather than them having to construct the full JSON object themselves.

## Compatibility

N/A

## Rejected alternatives

### Listing offsets continually
A previous version of this proposal suggested to have a property that prompted the operator to add the current offsets to the KafkaConnector status on every reconciliation.
This was rejected because it will be tricky to implement this in a way that doesn't result in cyclical reconciliations.
In addition, it seems likely that listing offsets will be used most often as a one-off action, rather than users needing to continually track offsets.

### Listing offsets in the KafkaConnector status

A previous version of this proposal suggested using the KafkaConnector status field to list the returned offsets.
This was rejected because updating the status would trigger an addition reconcile and the response from the endpoint may be a very large size if there are many partitions.
By using a ConfigMap, if the data is too large to fit into the ConfigMap the KafkaConnector CR can be updated to indicate this error, rather than risking updating the CR with a partial response.
In addition, since this proposal suggests using a ConfigMap to alter the offsets, the user can update an existing ConfigMap for altering offsets, rather than needing to copy and paste content.

### Altering offsets using the KafkaConnector resource

A previous version of this proposal suggested introducing a new Kubernetes Custom Resource to provide the request body for the PATCH endpoint.
This was rejected because introducing a new Custom Resource adds additional maintenance to the Strimzi project.

### Altering offsets using properties in the KafkaConnector resource
We could alter the offsets either using an annotation or spec change in the KafkaConnector resource.

Using an annotation on its own wouldn't really be feasible since the body of the request might be very complex.

We could use the spec described for `KafkaConnectorOffsets`, combined with an annotation to trigger the reset.
This has the benefit of not needing a new resource type, however, it means polluting the `KafkaConnector` resource with offset properties.
Having separate resources means users can also provide different permissions for the two kinds of resource.

### Allowing users to reset offsets when the connector is removed

A previous version of this proposal suggested introducing a new property on the KafkaConnector resource to allow users to request the offsets be reset when the connector is removed.
This was rejected because it would be hard to guarantee the operator can always successfully take this action.
For example if the operator was down when the KafkaConnector resource was deleted it would not know the value the property was set to when it came to delete the connector in Connect later.

[kip]: https://cwiki.apache.org/confluence/display/KAFKA/KIP-875%3A+First-class+offsets+support+in+Kafka+Connect
