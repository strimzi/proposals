# MirrorMaker Connector Offsets Support

Update the KafkaMirrorMaker2 resource to allow users to manage offsets

Note: This proposal focuses on the KafkaMirrorMaker2 resource.
Any reference to MirrorMaker is in regard to this resource, not the deprecated KafkaMirrorMaker resource.

## Current situation

[KIP-875][kip] added first class offsets support to Kafka Connect.
This was done via four new endpoints:
* GET /connectors/{connector}/offsets
* PATCH /connectors/{connector}/offsets
* DELETE /connectors/{connector}/offsets
* PUT /connectors/{connector}/stop

Support for the "stop" endpoint was added via [PR 9095](https://github.com/strimzi/strimzi-kafka-operator/pull/9095).
Currently users that are utilising the KafkaMirrorMaker2 resource cannot make use of the API endpoints for managing offsets.

Proposal [#076][proposal-76] discusses how to add support for these API endpoints to the KafkaConnector resource.
However it did not cover the KafkaMirrorMaker2 resource.

## Motivation

Previously if users wanted to list, update or delete offsets for a MirrorMaker connector they had to directly read/write from/to the Kafka Connect internal offsets topic.
[KIP-875][kip] adds new endpoints to let the user do this more easily.
There are multiple reasons a user might want to manage offsets, for example (as listed in the KIP):
* Resetting the offsets for a connector while iterating rapidly in a development environment (so that the connector does not have to be renamed each time)
* Viewing the in-memory offsets for source connectors on the cluster in order to recover from accidental deletion of the Connect source offsets topic (this is currently possible with the config topic, and can be a life-saver)
* Monitoring the progress of running connectors on a per-source-partition basis 
* Skipping records that cause issues with a connector and cannot be addressed using existing error-handling features

## Proposal

Strimzi should include mechanisms to let users manage offsets when utilising the KafkaMirrorMaker2 resources.
Many users of MirrorMaker make use of this resource, rather than the KafkaConnector resource, so it's important to support it here as well.
Similar, to [proposal #76][proposal-76], this proposal suggests introducing a new annotation on the KafkaMirrorMaker2 resource to trigger each operation.
The operations are:
* List offsets.
* Alter offsets.
* Reset offsets.

The list offsets operation uses a ConfigMap for the output.
The alter offsets operation uses a ConfigMap for the input.

In addition, a separate new annotation is proposed to allow the user to select which MirrorMaker connector to apply the action to.

### Request/Response from the API endpoints

The response body received when using the GET endpoint, and the request body used when calling the PATCH endpoint have identical structures to allow users to easily fetch and then alter offsets.
The structure of the request/response body is not fully defined for all Kafka Connect source connectors.
This is because source connectors determine their own structure for offsets depending on the source system they are integrating with.
However, for each of the MirrorMaker connectors (MirrorSourceConnector, MirrorCheckpointConnector, MirrorHeartbeatConnector) we know the expected structure.

MirrorSourceConnector format:
```
{
  "offsets": [
    {
      "partition": {
        "cluster": "east-kafka",
        "partition": 0,
        "topic": "mirrormaker2-cluster-configs"
      },
      "offset": {
        "offset": 0
      }
    }
  ]
}
```

MirrorCheckpointConnector format:
```
{
  "offsets": [
    {
      "partition": {
        "partition": 4,
        "topic": "inventory",
        "group": "my-group"
      },
      "offset": {
        "offset": 0
      }
    }
  ]
}
```

MirrorHeartbeatConnector format:
```
{
  "offsets": [
    {
      "partition": {
        "sourceClusterAlias": "east-kafka",
        "targetClusterAlias": "west-kafka"
      },
      "offset": {
        "offset": 0
      }
    }
  ]
}
```

### New annotations

All three actions will make use of a new annotation called `strimzi.io/connector-offsets`.
The possible values will be `list`, `alter`, and `reset`.

A second new annotation will be added called `strimzi.io/mirrormaker-connector`.
It will be required when `strimzi.io/connector-offsets` is set.
The value will be the name of the connector to apply the action to, for example `east-kafka->west-kafka.MirrorSourceConnector`.

### Listing offsets

To list the current offsets the user will annotate the KafkaMirrorMaker2 resource with the new annotation `strimzi.io/connector-offsets` set to `list`, and the `strimzi.io/mirrormaker-connector` annotation.
After the annotations are added, on the next reconciliation the operator will fetch the current offsets for the connector and create a Kubernetes ConfigMap containing the JSON response.
The operator will add an owner reference to the Kubernetes ConfigMap pointing to the KafkaMirrorMaker2 resource that was annotated.
If the ConfigMap already exists the operator will patch the ConfigMap with updated data and not add an owner reference.
This means any existing keys in the ConfigMap that the operator is not updating will remain as before.
Once the list operation is complete the operator will then remove both the `strimzi.io/mirrormaker-connector` and `strimzi.io/connector-offsets` annotations from the KafkaMirrorMaker2 CR.

If the user wants to see an updated set of offsets they will need to re-annotate the resource.

The name of the ConfigMap the operator creates/updates will be set by the user in the KafkaMirrorMaker2 CR.
The KafkaMirrorMaker2 CRD will be updated to contain a new connector property called `listOffsets` that must be set for the `strimzi.io/connector-offsets=list` annotation to take effect.
The structure of the `listOffsets` property will be as below, to allow it to be extended in future if required:

```yaml
spec:
  #...
  mirrors:
    - sourceConnector:
        #...
        listOffsets:
          toConfigMap:
            name: my-connector-offsets
      checkpointConnector:
        #...
        listOffsets:
          toConfigMap:
            name: my-connector-offsets
      heartbeatConnector:
        #...
        listOffsets:
          toConfigMap:
            name: my-connector-offsets
```

The ConfigMap will contain a single entry, containing the response received from the GET /connectors/{connector}/offsets endpoint.
The name of the entry will be based on the connector name, however since 
[`>` characters are not allowed in ConfigMap keys](https://kubernetes.io/docs/concepts/configuration/configmap/#configmap-object) the name will be altered to replace `->` with `--`.
For example a connector called `east-kafka->west-kafka.MirrorSourceConnector` will have offsets placed under `east-kafka--west-kafka.MirrorSourceConnector.json`.
This will make it possible to use the same ConfigMap for different connectors.

For example for a MirrorSourceConnector the ConfigMap might look like:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-mm2-offsets
  ownerReferences:
    - apiVersion: kafka.strimzi.io/v1beta2
      blockOwnerDeletion: false
      controller: false
      kind: KafkaMirrorMaker2
      name: my-mm2
      uid: 1234
data:
  east-kafka--west-kafka.MirrorSourceConnector.json: |
    {
      "offsets": [
        {
          "partition": {
            "cluster": "east-kafka",
            "partition": 0,
            "topic": "mirrormaker2-cluster-configs"
          },
          "offset": {
            "offset": 0
          }
        }
      ]
    }
```

For a MirrorCheckpointConnector it might look like:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-mm2-offsets
  ownerReferences:
    - apiVersion: kafka.strimzi.io/v1beta2
      blockOwnerDeletion: false
      controller: false
      kind: KafkaMirrorMaker2
      name: my-mm2
      uid: 5678
data:
  east-kafka--west-kafka.MirrorCheckpointConnector.json: |
    {
      "offsets": [
        {
          "partition": {
            "partition": 4,
            "topic": "inventory",
            "group": "my-group"
          },
          "offset": {
            "offset": 0
          }
        }
      ]
    }
```

For a MirrorHeartbeatConnector it might look like:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-mm2-offsets
  ownerReferences:
    - apiVersion: kafka.strimzi.io/v1beta2
      blockOwnerDeletion: false
      controller: false
      kind: KafkaMirrorMaker2
      name: my-mm2
      uid: 5678
data:
  east-kafka--west-kafka.MirrorHeartbeatConnector.json: |
    {
      "offsets": [
        {
          "partition": {
            "sourceClusterAlias": "east-kafka",
            "targetClusterAlias": "west-kafka"
          },
          "offset": {
            "offset": 0
          }
        }
      ]
    }
```

Notes:
* If the `listOffsets` property is missing from the KafkaMirrorMaker2 CR when the `strimzi.io/connector-offsets=list` annotation is added the operator will update the KafkaMirrorMaker2 CR status to include a warning message:
  ```yaml
  - lastTransitionTime: "2024-06-04T08:44:15.913138115Z"
    message: Failed to list the connector offsets for east-kafka->west-kafka.MirrorCheckpointConnector due to missing property listOffsets in KafkaMirrorMaker2 CR.
    reason: ListOffsets
    status: "True"
    type: Warning
  ```
  The operator will leave the `strimzi.io/connector-offsets=list` annotation and the `strimzi.io/mirrormaker-connector` annotation on the KafkaMirrorMaker2 resource until either the list operation succeeds, or the user removes the annotations.
  This means the operator will retry the list on every reconciliation, allowing the condition to remain present for the user to see.
* If only one of the `strimzi.io/connector-offsets` and `strimzi.io/mirrormaker-connector` annotations is present on the KafkaMirrorMaker2 the operator will leave the annotation on the resource and update the KafkaMirrorMaker2 CR status to include a warning message similar to if `listOffsets` is missing.

### Altering offsets

To alter offsets the user will annotate the KafkaMirrorMaker2 resource with the new annotation `strimzi.io/connector-offsets` set to `alter`, and the `strimzi.io/mirrormaker-connector` annotation.
After the annotations are added, on the next reconciliation the operator will call the PATCH /connectors/{connector}/offsets endpoint to alter the offsets for the connector.
Once the patch operation is complete the operator will then remove both the `strimzi.io/mirrormaker-connector` and `strimzi.io/connector-offsets` annotations from the KafkaMirrorMaker2 CR.

The operator will read in the entry called `<CONNECTOR_NAME>.json` from the ConfigMap.
The entry name matches the entry name used in the list operation, so that a user can list offsets, then edit the ConfigMap the operator created, then request the offsets to be altered using that same ConfigMap.
This means the connector name expected will have `->` replaced with `--`, similar to in the list offsets operation.

The name of the ConfigMap the operator will use to construct the request body will be set by the user in the KafkaMirrorMaker2 CR.
The KafkaMirrorMaker2 CRD will be updated to contain a new connector property called `alterOffsets` that must be set for the `strimzi.io/connector-offsets=alter` annotation to take effect.
The structure of the `alterOffsets` property will be as below, to allow it to be extended in future if required:

```yaml
spec:
  #...
  mirrors:
    - sourceConnector:
        #...
        alterOffsets:
          fromConfigMap:
            name: my-connector-offsets
      checkpointConnector:
        #...
        alterOffsets:
          fromConfigMap:
            name: my-connector-offsets
      heartbeatConnector:
        #...
        alterOffsets:
          fromConfigMap:
            name: my-connector-offsets
```

The expected format will match the output from the list offsets action.

For example for a MirrorSourceConnector the ConfigMap `<CONNECTOR_NAME>.json` data field must contain an entry that matches:

```json
{
  "offsets": [
    {
      "partition": {
        "cluster": "east-kafka",
        "partition": 0,
        "topic": "mirrormaker2-cluster-configs"
      },
      "offset": {
        "offset": 0
      }
    }
  ]
}
```

For a MirrorCheckpointConnector the ConfigMap `<CONNECTOR_NAME>.json` data field must contain an entry that matches:

```json
{
  "offsets": [
    {
      "partition": {
        "partition": 4,
        "topic": "inventory",
        "group": "my-group"
      },
      "offset": {
        "offset": 0
      }
    }
  ]
}
```

For a MirrorHeartbeatConnector the ConfigMap `<CONNECTOR_NAME>.json` data field must contain an entry that matches:

```json
{
  "offsets": [
    {
      "partition": {
        "sourceClusterAlias": "east-kafka",
        "targetClusterAlias": "west-kafka"
      },
      "offset": {
        "offset": 0
      }
    }
  ]
}
```

Notes:
* The data supplied by the user will be validated to make sure it is syntactically valid JSON, but the operator won't perform any further validation, instead relying on the API endpoint to return a reasonable error message.
* If the request to the Connect API fails the operator will add a condition to the KafkaMirrorMaker2 CR status with a warning message that includes the response from the API, e.g.
  ```yaml
  - lastTransitionTime: "2024-06-04T08:44:15.913138115Z"
    message: Failed to alter the connector offsets for east-kafka->west-kafka.MirrorCheckpointConnector due to "message from endpoint".
    reason: AlterOffsets
    status: "True"
    type: Warning
  ```
  The operator will leave the annotations on the KafkaMirrorMaker2 resource until either the patch operation succeeds, or the user removes the annotations.
  This means the operator will retry the patch on every reconciliation, allowing the condition to remain present for the user to see.
* Strimzi will shortcut and automatically fail to alter the offsets if the connector does not have it's `state` set as `stopped` in the KafkaMirrorMaker2 resource.
  Similar to if Connect returns on error, the operator will add a condition stating that the operation has failed because the connector is not stopped and therefore offsets cannot be altered, and leave the annotations on the resource.
  The user can update the KafkaMirrorMaker2 to stop the connector and alter the offsets at the same time.
  In that case the operator will first stop the connector, and once that API call returns, then it will make the call to alter the offsets.
* If the specified ConfigMap `data` does not contain an entry called `<CONNECTOR_NAME>.json`, or the entry fails validation this will be treated as an error, and the operator will add a condition as above, leaving the annotations on the resource.
* The operator will only examine the contents of the fields it is expecting in the ConfigMap, ignoring any other fields.
* If only one of the `strimzi.io/connector-offsets` and `strimzi.io/mirrormaker-connector` annotations is present on the KafkaMirrorMaker2 the operator will leave the annotation on the resource and update the KafkaMirrorMaker2 CR status to include a warning message.

### Resetting offsets

To reset connector offsets the user will annotate the KafkaMirrorMaker2 resource with the new annotation `strimzi.io/connector-offsets` set to `reset`, and the `strimzi.io/mirrormaker-connector` annotation.
When the operator observes these annotations it will use the DELETE /connectors/{connector}/offsets endpoint to reset all offsets for the connector.

Once the delete operation is complete the operator will then remove both the `strimzi.io/mirrormaker-connector` and `strimzi.io/connector-offsets` annotations from the KafkaMirrorMaker2 CR.

If the request to the Connect API fails the operator will add a condition to the KafkaMirrorMaker2 CR status with a warning message that includes the response from the API, e.g.
  ```yaml
  - lastTransitionTime: "2024-06-04T08:44:15.913138115Z"
    message: Failed to reset the connector offsets for east-kafka->west-kafka.MirrorCheckpointConnector due to "message from endpoint".
    reason: ResetOffsets
    status: "True"
    type: Warning
  ```
The operator will leave the annotations on the KafkaMirrorMaker2 resource until either the delete operation succeeds, or the user removes the annotations.
This means the operator will retry the delete on every operation, allowing the condition to remain present for the user to see.

Strimzi will shortcut and automatically fail to do the reset if the connector does not have its `state` set as `stopped` in the KafkaMirrorMaker2 resource.
Similar to if Connect returns on error, the operator will add a condition stating that the operation has failed because the connector is not stopped and therefore offsets cannot be reset, and leave the annotation on the resource.
The user can update the KafkaMirrorMaker2 to stop the connector and reset the offsets at the same time.
In that case the operator will first stop the connector, and once that API call returns, then it will make the call to reset the offsets.

If only one of the `strimzi.io/connector-offsets` and `strimzi.io/mirrormaker-connector` annotations is present on the KafkaMirrorMaker2 the operator will leave the annotation on the resource and update the KafkaMirrorMaker2 CR status to include a warning message.

## Affected/not affected projects

This proposal only affects the KafkaMirrorMaker2 parts of the cluster-operator.
It has been designed to work in a similar way to [proposal #76][proposal-76] so that code can be shared with KafkaConnector.

## Compatibility

N/A

## Rejected alternatives

### Applying actions to multiple connectors

Instead of making the `strimzi.io/mirrormaker-connector` required, we could make it optional or a comma-separated list.

In the case where it was optional, if only the `strimzi.io/connector-offsets` annotation is present, the action would be applied to all mirroring routes and all connectors within mirroring routes that are configured in the KafkaMirrorMaker2 resource.
If the `strimzi.io/mirrormaker-connector` annotation is also present, the action would only be applied to the connector identified in the annotation.

In the case where it was a comma-separated list it would be required and explicitly list the connectors to apply the action to.

This would be a nice feature to allow users to quickly list, alter and reset offsets for many connectors at once.
Particularly for listing and resetting offsets this seems very useful.

However it will add a lot of complexity because the operator reconciles one connector at a time.
This means we would need to introduce a mechanism to keep track of when we can remove the annotations and deal with error cases if for example one connector is stopped, but another isn't.

### Validating the structure of alterOffsets ConfigMap

A previous version of this proposal described more complex validation of the JSON provided for altering offsets.
This included validating the specific fields that are needed, and even not requiring fields that the operator could infer.

However, this would make the operator vulnerable to changes in the way the MirrorMaker connectors store their offsets so was rejected in favour of less strict validation.

### Preventing the user from altering or reseting offsets for the MirrorCheckpointConnector and MirrorHeartbeatConnector

The MirrorCheckpointConnector and MirrorHeartbeatConnector both store offsets in Kafka, however they do not actually use those offsets.
MirrorMaker users may still track offsets to track the progress of those connectors, but since they are not used, it isn't immediately obvious why a user would want to reset or alter them.
This [Apache Kafka PR](https://github.com/apache/kafka/pull/14005) talks about some of the reasons why these operations may still be desired by users.
In addition, it is possible that in future these connectors will make use of these offsets.
For these reasons this proposal explicitly allows users to perform list/alter/reset on all MirrorMaker connectors.

[kip]: https://cwiki.apache.org/confluence/display/KAFKA/KIP-875%3A+First-class+offsets+support+in+Kafka+Connect
[proposal-76]: 076-connector-offsets-support.md