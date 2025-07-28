# Add restart parameters to Kafka connectors

In order to support [Kafka KIP-745](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181308623) we need to accept two more parameters on restart kafka connectors: `includeTasks` and `onlyFailed`.

## Current situation

Currently when adding annotation restart these two new parameters are always false when calling the Kafka connect API.

## Motivation

We should be able to support customizing this parameters then user can choose behavior according with your requirements.

## Proposal

If you take a look on other very famous products like nginx, they allow most of configs using [annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/). Then my proposal is to do the same here, and we should have the following:

strimzi.io/restart # already exist
strimzi.io/restart-include-tasks # new, with boolean value
strimzi.io/restart-only-failed # new, with boolean value
strimzi.io/restart-task # already exist.

Today when calling the restart in Kafka Connect API we already set these parameters, right here:

```java
@Override
public CompletableFuture<Map<String, Object>> restart(String host, int port, String connectorName, boolean includeTasks, boolean onlyFailed) {
    return restartConnectorOrTask(host, port, "/connectors/" + connectorName + "/restart?includeTasks=" + includeTasks + "&onlyFailed=" + onlyFailed);
}
```

So, we won't change this behavior, instead we will set the values of `includeTasks` and `onlyFailed` based on the user expectations, since today we always set them to false:

```java
return VertxUtil.completableFutureToVertxFuture(apiClient.restart(host, port, connectorName, false, false))
```

The same way we check if there is a restart annotation, we will check if there is a `strimzi.io/restart-include-tasks` and `strimzi.io/restart-only-failed` annotations, and set the values accordingly, 
something like this:

```java
boolean restartIncludeTasks = hasRestartIncludeTasksAnnotation(resource, connectorName);
boolean restartOnlyFailedTasks = hasRestartOnlyFailedTasksAnnotation(resource, connectorName);
```

The final method should look like this:

```java
@SuppressWarnings({ "rawtypes" })
private Future<List<Condition>> maybeRestartConnector(Reconciliation reconciliation, String host, KafkaConnectApi apiClient, String connectorName, CustomResource resource, List<Condition> conditions) {
    if (hasRestartAnnotation(resource, connectorName)) {
        boolean restartIncludeTasks = hasRestartIncludeTasksAnnotation(resource, connectorName);
        boolean restartOnlyFailedTasks = hasRestartOnlyFailedTasksAnnotation(resource, connectorName);
        LOGGER.debugCr(reconciliation, "Restarting connector {}", connectorName);
        return VertxUtil.completableFutureToVertxFuture(apiClient.restart(host, port, connectorName, restartIncludeTasks, restartOnlyFailedTasks))
                .compose(ignored -> removeRestartAnnotation(reconciliation, resource)
                    .compose(v -> Future.succeededFuture(conditions)),
                    throwable -> {
                        // Ignore restart failures - add a warning and try again on the next reconcile
                        String message = "Failed to restart connector " + connectorName + ". " + throwable.getMessage();
                        LOGGER.warnCr(reconciliation, message);
                        conditions.add(StatusUtils.buildWarningCondition("RestartConnector", message));
                        return Future.succeededFuture(conditions);
                    });
    } else {
        return Future.succeededFuture(conditions);
    }
}
```

As you can see on code above, `strimzi.io/restart` is required in order to process the restart. We accept any sort of combinations between these two new annotations, 
but you should take a look on the [KIP-745](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181308623#KIP745:ConnectAPItorestartconnectorandtasks-RestartMethod) to understand how these parameters work together. 
As an example set `strimzi.io/restart-include-tasks` to false and `strimzi.io/restart-only-failed` to true will have no effect. 

## Affected/not affected projects

- http://github.com/strimzi/strimzi-kafka-operator/. 

## Compatibility

We will keep default value as false for both variables, keeping backward compatibility for users not using the new annotations.

## Rejected alternatives

Nothing to add here.
