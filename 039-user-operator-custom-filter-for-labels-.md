# User Operator: Configurable exclusion of labels

This proposal is about adding the ability to filter the automatically assigned labels.
When User Operator creates `KafkaUser`, it also creates an associated Secret, where it automatically adds the label
`app.kubernetes.io/instance: <username>`.
Such a label could be used for many uses, and different users have various requirements and expectations.
Nevertheless, it could also lead to undesirable scenarios (e.g., [repeated deletion and re-creation of specific Secret](https://github.com/strimzi/strimzi-kafka-operator/issues/5690)).
Therefore, we propose to implement configurable exclusion of labels.

## Current situation

Presently, when a user attempts to create a `KafkaUser` defined as follows:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: tls
```
The User Operator then creates an associated Kubernetes `Secret`.
```yaml
kind: Secret
metadata:
  labels:
    app.kubernetes.io/instance: my-user
    app.kubernetes.io/managed-by: strimzi-user-operator
    app.kubernetes.io/name: strimzi-user-operator
    app.kubernetes.io/part-of: strimzi-my-user
    strimzi.io/cluster: my-cluster
    strimzi.io/kind: KafkaUser
  name: my-user
  namespace: myproject
...
```
We can see that User Operator automatically adds the `app.kubernetes.io/instance: my-user` label.
If we do not want the label that has been assigned, the only way to get rid of it is to filter it.

## Proposal

This proposal suggests adding configurable exclusion of labels.
We can implement this feature similarly to [issue](https://github.com/strimzi/strimzi-kafka-operator/pull/4791).
We can create an environment variable and inject such value through the `Kafka` custom resource.
Specifically, in the `spec.entityOperator.template.userOperatorContainer.env`.
For instance, we can have the following `Kafka` resource:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  ...
  entityOperator:
    template:
      userOperatorContainer:
        env:
          name:   STRIMZI_LABELS_EXCLUSION_PATTERN
          value:  "^app.kubernetes.io/.*$"
    topicOperator: {}
    userOperator: {}
```
Then, we would obtain a value from the environment variable and parse it in the `UserOperatorConfig` class.
Based on the environment variable value, we will determine what to do (a) env is not specified, and therefore we do not exclude any label from Secret
(b) env is specified by a regex, and then we will filter and remove such labels which match the input regex.

### More implementation details

#### Parsing part

As we mentioned details about the `UserOperatorConfig` class, we will elaborate more on the specific details in this section.
Moreover, we will obtain the environment variable inside the `fromMap` method used to construct the class.
```java
public static UserOperatorConfig fromMap(Map<String, String> map) {
    ...
    // 
    String strimziLabelsExclusionPatternEnvVar = map.get(UserOperatorConfig.STRIMZI_LABELS_EXCLUSION_PATTERN);
    
    if (strimziLabelsExclusionPattern != null) {
        // note that we will compile such regex into FSM only once here and thus eliminate workload inside KafkaUserModel
        strimziLabelsExclusionPattern = Pattern.compile(strimziLabelsExclusionPatternEnvVar);
    }
    ...
}
```

#### Exclusion part

The exclusion part will be placed in the `KafkaUserModel` class.
Moreover, we would need to get the value of the environment variable to such a class.
In the `UserOperatorConfig` class, we will obtain value, and then in the `KafkaUserOperaor` class, specifically in the following method:
```java
protected Future<KafkaUserStatus> createOrUpdate(Reconciliation reconciliation, KafkaUser resource) {
    KafkaUserModel user;
    KafkaUserStatus userStatus = new KafkaUserStatus();

    try {
        user = KafkaUserModel.fromCrd(resource, config.getSecretPrefix(), config.isAclsAdminApiSupported(),config.isKraftEnabled(), 
            config.getStrimziLabelsExclusionPattern); // <-- this one we will inject into KafkaUserModel class
    } catch (Exception e) {
        StatusUtils.setStatusConditionAndObservedGeneration(resource, userStatus, Future.failedFuture(e));
        return Future.failedFuture(new ReconciliationException(userStatus, e));
    }
    ...
}
```
Then, we can use such value to store it as an instance attribute of `KafkaUserModel` and implement pre-processing of labels in the
following method:
```java
protected Secret createSecret(Map<String, String> data) {
    final Map<String, String> labels = Util.mergeLabelsOrAnnotations(labels.toMap(), templateSecretLabels);
    // here, we have to do pre-processing (i.e., filtering) of labels by exclusion pattern
    // filter by value of instance variable `this.getStrimziLabelsExclusionPattern`
    
    return return new SecretBuilder()
        .withNewMetadata()
            .withName(getSecretName())
            .withNamespace(namespace)
            .withLabels(labels)
            .withAnnotations(Util.mergeLabelsOrAnnotations(null, templateSecretAnnotations))
            .withOwnerReferences(createOwnerReference())
        .endMetadata()
        .withType("Opaque")
        .withData(data)
        .build();
}
```

## Compatibility

This proposal does not change any of the existing CRDs or the Kubernetes secrets that are being created.
The user only needs to modify Kafka's custom resource to use such a feature.

## Rejected alternatives

1. We considered the alternative to remove such labels entirely simply. However, this could lead to un-excepted behaviour (i.e., breaking any users relying on them).
