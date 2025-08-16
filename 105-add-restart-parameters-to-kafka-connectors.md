# Add restart parameters to Kafka connectors

In order to support [Kafka KIP-745](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181308623) we need to accept two more parameters on restart Kafka connectors: `includeTasks` and `onlyFailed`. 
This includes MirrorMaker 2 connectors as well.

## Current situation

Currently when adding annotation restart these two new parameters are always false when calling the Kafka Connect API.

## Motivation

We should be able to support customizing this parameters then user can choose behavior according with your requirements.

## Proposal

The idea is to allow users to use combination of arguments, like these: 

```yaml
strimzi.io/restart=includeTasks,onlyFailed          # restart with args: includeTasks=true and onlyFailed=true
strimzi.io/restart=includeTasks                     # restart with args: includeTasks=true and onlyFailed=false
strimzi.io/restart=onlyFailed                       # restart with args: includeTasks=false and onlyFailed=true
strimzi.io/restart=true                             # restart with args: includeTasks=false and onlyFailed=false
strimzi.io/restart=false,includeTasks,onlyFailed    # do not restart, fail and log error because you can't set args and boolean value together
strimzi.io/restart=true,includeTasks,onlyFailed     # do not restart, fail and log error because you can't set args and boolean value together
strimzi.io/restart=includeTasks,wrongArg            # do not restart, fail and log error because wrongArg is not supported
```

If you are using MirrorMaker 2, you can use the same arguments, but the annotation is a bit different:

```yaml
strimzi.io/restart-connector=<mirrormaker_connector_name>:includeTasks,onlyFailed # restart with args: includeTasks=true and onlyFailed=true
strimzi.io/restart-connector=<mirrormaker_connector_name>:includeTasks # restart with args: includeTasks=true and onlyFailed=false
strimzi.io/restart-connector=<mirrormaker_connector_name>:onlyFailed # restart with args: includeTasks=false and onlyFailed=true
strimzi.io/restart-connector=<mirrormaker_connector_name> # restart with args: includeTasks=false and onlyFailed=false
strimzi.io/restart-connector=includeTasks,onlyFailed # do not restart, fail and log error because connector name is required
```

Today when calling the restart in Kafka Connect API we already set these parameters, we won't change this behavior, instead we will set the values of `includeTasks` and `onlyFailed` based on the user expectations, since today we always set them to false:

```java
return VertxUtil.completableFutureToVertxFuture(apiClient.restart(host, port, connectorName, false, false))
```

The first change will be on the method `hasRestartAnnotation`, so it can return true or false according with mixed cases showed previously. Next step will be create methods to check if `includeTasks` and `failedTasks` are set, suggestion:

```java
boolean restartIncludeTasks = restartAnnotationHasIncludeTasksArg(resource, connectorName);
boolean restartOnlyFailedTasks = restartAnnotationHasOnlyFailedTasksArg(resource, connectorName);
```

The final method should look like this:

```java
@SuppressWarnings({ "rawtypes" })
private Future<List<Condition>> maybeRestartConnector(Reconciliation reconciliation, String host, KafkaConnectApi apiClient, String connectorName, CustomResource resource, List<Condition> conditions) {
    if (hasRestartAnnotation(resource, connectorName)) {
        boolean restartIncludeTasks = restartAnnotationHasIncludeTasksArg(resource, connectorName);
        boolean restartOnlyFailedTasks = restartAnnotationHasOnlyFailedTasksArg(resource, connectorName);
        LOGGER.debugCr(reconciliation, "Restarting connector {}, IncludeTasks {}, OnlyFailedTasks {}", connectorName, restartIncludeTasks, restartOnlyFailedTasks);
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

## Affected/not affected projects

- http://github.com/strimzi/strimzi-kafka-operator/. 

## Compatibility

We will keep default value as false for both variables, keeping backward compatibility for users not using the new annotations.

## Rejected alternatives
There are other alternatives considered and the reason why not chosen is as follows:

1. Create one different annotation for each new argument. Can cause confusion because user should care about order, it means always set restart current annotation after the argument annotation, which could cause unexpected behaviors.
2. JSON format inside restart annotation. Hard to user legibility and more chance to typo errors. 

