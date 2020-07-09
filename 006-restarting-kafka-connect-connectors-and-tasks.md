# Restarting Kafka Connect connectors and tasks

This proposal adds support for restarting Kafka Connect connectors and connector tasks via annotations added to the `KafkaConnector` and `KafkaMirrorMaker2` custom resources and via overridable default behaviour of the `KafkaConnector` and `KafkaMirrorMaker2` operators.


## Current situation

When Kafka Connect connectors or tasks throw uncaught exceptions, the Kafka Connect framework stops the connector/task and puts it into a `FAILED` state. The Kafka Connect REST API includes two endpoints to restart a connector and a task:
 - `POST /connectors/{connector-name}/restart`
 - `POST /connectors/{connector-name}/tasks/{task-id}/restart`
Sending a request to one of these endpoints will cause the Connect framework to restart the connector/task.

Currently, the Strimzi operators do not have any special handling for connectors or tasks that are in the `FAILED` state. Users must make HTTP POST requests to the Connect restart endpoints if they want to restart a connector/task.

A connector/task does not need to be in the `FAILED` state to be restarted via the Connect REST API; a connector or task can be restarted at any time. Restarting a connector or task causes the Connect framework to re-initialise the connector/task and reactivate message processing. Note that restarting a connector does not restart its tasks.

The default network policies created by the Strimzi operators restrict incoming Connect REST API calls to only allow access from the operator pod, so users must add supplementary network policies to be able to make calls to the restart REST APIs.


## Motivation

Enabling users to restart connectors and tasks means there are almost (1) no further Connect REST APIs that users must call to interact with connectors/tasks when using CRs that create connectors. This means the CRs and their operators can exclusively be used to manage connectors and tasks - Strimzi users do not have to call the Connect REST API directly. As a corollary, this means the Connect REST listener does not need to be available to users; it could be secured so that only the operator can interact with the REST API.

(1) - Kafka 2.5 introduced the `GET /connectors/{connector-name}/topics` Connect endpoint to list the topics that a connector has used since the connector was created or the list was reset. Kafka 2.5 also introduced the `PUT /connectors/{connector-name}/topics/reset` Connect endpoint to clear the list of topics that a connector has used.


## Proposal

This proposal adds two new separate but related features:

1. Two new annotations that cause the operator to restart connectors or tasks. They can be applied to the `KafkaConnector`, and the `KafkaMirrorMaker2` custom resources. The annotation acts as a trigger for a single restart call by the operator, and is removed from the CR when the restart REST API call is successfully called.

2. New default behaviour in the `KafkaConnector` and `KafkaMirrorMaker2` operators that automatically attempts to restart connectors and tasks if they are in the `FAILED` state. The `KafkaConnector` and `KafkaMirrorMaker2` CRs can use a new annotation to disable the automatic restarting behaviour if required.

These two features could be delivered separately as they do not rely on each other.


### `strimzi.io/restart` annotation

An annotation named `strimzi.io/restart` can be applied to the `KafkaConnector` and `KafkaMirrorMaker2` custom resources. The value of the annotation and the CR it is applied to determines the connector that is restarted. Once the operator successfully calls the restart REST API endpoint, the CR is patched to remove the annotation. The operator attempts to call the restart REST API endpoint once per reconcile cycle.

When applied to a `KafkaConnector` CR, the connector defined by the CR is restarted. Any value can be assigned to the annotation. For example:
```
apiVersion: eventstreams.ibm.com/v1alpha1
kind: KafkaConnector
metadata:
  name: my-source-connector
  annotations:
    strimzi.io/restart: "true"
  ...
```

When applied to a `KafkaMirrorMaker2` CR, the annotation value contains the name of the connector that is restarted. For example:
```
apiVersion: eventstreams.ibm.com/v1alpha1
kind: KafkaMirrorMaker2
metadata:
  name: my-mm2-cluster
  annotations:
    strimzi.io/restart: "source-cluster->target-cluster.MirrorSourceConnector"
  ...
```

### `strimzi.io/restart-task` annotation

An annotation named `strimzi.io/restart-task` can be applied to the `KafkaConnector`, and `KafkaMirrorMaker2` custom resources. The value of the annotation and the CR it is applied to determines the connector task that is restarted. Once the operator successfully calls the restart REST API endpoint, the CR is patched to remove the annotation. The operator attempts to call the restart REST API endpoint once per reconcile cycle.

When applied to a `KafkaConnector` CR, the annotation value contains the ID of the task that is restarted. For example:
```
apiVersion: eventstreams.ibm.com/v1alpha1
kind: KafkaConnector
metadata:
  name: my-source-connector
  annotations:
    strimzi.io/restart-task: "0"
  ...
```

When applied to a `KafkaMirrorMaker2` CR, the annotation value contains the name of the connector and the ID of the task that is restarted, delimited by a `:` character. For example:

```
apiVersion: eventstreams.ibm.com/v1alpha1
kind: KafkaMirrorMaker2
metadata:
  name: my-mm2-cluster
  annotations:
    strimzi.io/restart-task: "source-cluster->target-cluster.MirrorSourceConnector:2"
  ...
```


### Restarting `FAILED` connectors and tasks

The `KafkaConnector` and `KafkaMirrorMaker2` operators will have new default behaviour that automatically restarts connectors or tasks that are in the `FAILED` state. For each connector or task that is in the `FAILED` state, the operator will call the Connect REST API `restart` endpoint once per reconcile cycle to attempt to restart the connector/task.

To disable the new default behaviour a new annotation named `strimzi.io/auto-restart` with a value of `false` can be applied to the `KafkaConnector` and `KafkaMirrorMaker2` CRs. This will be useful when users know that external factors will cause connectors/tasks to be in the `FAILED` state and they do not want the operator attempting automatic restarts. It also means users can keep the current behaviour of the `KafkaConnector` and `KafkaMirrorMaker2` operators if they wish.

For example, the following CR disables the automatic restarting of the connector and it's tasks when they are in the `FAILED` state:
```
apiVersion: eventstreams.ibm.com/v1alpha1
kind: KafkaConnector
metadata:
  name: my-source-connector
  annotations:
    strimzi.io/auto-restart: "false"
  ...
```

## Compatibility

Automatically restarting connectors and tasks in the `FAILED` state means that the default behaviour of the `KafkaConnector` and `KafkaMirrorMaker2` operators will be slightly different. This can be overridden using the `strimzi.io/auto-restart` annotation if desired.


## Rejected Alternatives

The default behaviour of the `KafkaConnector` and `KafkaMirrorMaker2` operators could be left as-is, and the new `strimzi.io/auto-restart` annotation could be applied to explicitly enable the automatic restarting behaviour. It was felt that the most useful 'Kubernetes-native' behaviour would be to attempt automatic restarts, in a similar way to how failing pods are recreated by their parent deployments.
