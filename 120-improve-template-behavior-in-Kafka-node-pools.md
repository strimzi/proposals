# Improve template behavior in Kafka Node Pools

This proposal suggests changes to how we handle conflicts between template configurations in the `Kafka` CR (`.spec.kafka.template`) and `KafkaNodePool` CR (`.spec.template`).

## Current situation

Both `Kafka` and `KafkaNodePool` resources have their template sections:
* `.spec.kafka.template` in the `Kafka` CR
* `.spec.template` in the `KafkaNodePool` CR

All of the fields that can be configured in the `KafkaNodePool` template are also present in the `Kafka` CR template.
And the `Kafka` CR template also contains some additional fields unrelated to node pools.

The options that can be configured in both templates are:
* `StrimziPodSet` template
* `Pod` template
* per-pod `Service` template
* per-pod `Route` template
* per-pod `Ingress` template
* `PersistentClaimVolume` template
* Kafka container template
* Init container template

The template fields set in the `KafkaNodePool` resource apply only to Kubernetes resources corresponding to the given node pool.
While the template fields set in the `Kafka` resource apply to Kubernetes resources of all node pools belonging to the given Apache Kafka cluster.

Each of these templates has its own use-cases:
* The `Kafka` template can be used to configure all node pools from a single place
* The `KafkaNodePool` template can be used to configure only a specific node pool.

But what happens when both templates are set at the same time?
Currently, whenever the `template` field is used in the `KafkaNodePool` resource, the values set in the `Kafka` template will be ignored.
A complete override occurs even if the `KafkaNodePool` template only defines a subset of the available template fields, effectively resetting any configurations from the `Kafka` CR template.

For example, imagine `Kafka` CR with the following template:
```yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    template:
      pod:
        metadata:
          labels:
            mylabel: myvalue
      # ...
```

And the following template set in the `KafkaNodePool` CR:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: brokers
  labels:
    strimzi.io/cluster: my-cluster
spec:
  # ...
  template:
    kafkaContainer:
      env:
      - name: EXAMPLE_ENV_1
        value: example.env.one
```

How will the Pod belonging to this Kafka node pool be configured?
It will have the environment variable `EXAMPLE_ENV_1` set in the Kafka container.
But the label `mylabel=myvalue` will not be used because its configuration in the `Kafka` CR was overridden by the use of the template section in the `KafkaNodePool`.

## Motivation

The situation described above often causes confusion among our users.
And as part of the introduction of the `v1` CRD API, we discussed whether the conflicting fields should be deprecated and removed from the `Kafka` CR.
However, the outcome of the discussion was that the ability to set the template for all node pools in the `Kafka` CR is useful.
Instead, we want to change the conflict resolution behavior.
That should keep the useful aspect while trying to ease the current confusion.
While this proposal is not backwards compatible, it provides a worthwhile improvement to how templates are handled. 
We believe this improvement is worth the breaking change.

## Proposal

This proposal changes the way template conflicts will be handled.
Instead of ignoring the Kafka CR template, the operator will now perform a merge of the template sections from the `Kafka` and `KafkaNodePool` resources, based on their top-level fields (like `pod` and `kafkaContainer`). 
If a top-level field is defined in both templates, the `KafkaNodePool` version overrides the one from `Kafka`. 
If each resource defines different top-level fields, both are applied. 
For example, the Kafka resource might define the `pod` template, while `KafkaNodePool` defines the `kafkaContainer` template. 
The final configuration includes both.

For example, imagine `Kafka` CR with the following template:
```yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    template:
      pod:
        metadata:
          labels:
            mylabel: myvalue
      kafkaContainer:
        securityContext:
          runAsUser: 2000
      # ...
```

And the following template set in the `KafkaNodePool` CR:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: brokers
  labels:
    strimzi.io/cluster: my-cluster
spec:
  # ...
  template:
    kafkaContainer:
      env:
      - name: EXAMPLE_ENV_1
        value: example.env.one
```

With the proposal implemented, the `Pod` belonging to this node pool will have:
* The label `mylabel=myvalue` set
* The environment variable `EXAMPLE_ENV_1` set

However, the content of the `kafkaContainer` section will not be merged.
And the `Pod` will not have the security context set to run as user `2000`.

### Resetting the template

This feature provides an explicit way for a specific node pool to opt-out of inheriting the cluster-wide template settings in the `Kafka` resource.

Users will still be able to reset the template in the `KafkaNodePool` template.
But they will need to do it explicitly by setting the corresponding template section to empty object (`{}`).
The following example shows how to reset the template.
In this case, it will reset the `Pod` template from the `Kafka` CR:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: brokers
  labels:
    strimzi.io/cluster: my-cluster
spec:
  # ...
  template:
    pod: {}
    kafkaContainer:
      env:
      - name: EXAMPLE_ENV_1
        value: example.env.one
```

### Implementation

The proposal will be implemented in the `KafkaPool` class in the `cluster-operator` module and the related test classes.

### Documentation

The documentation will be updated with this change.
It will be also included in the release notes as it is a breaking change.

## Backwards compatibility

**This proposal is not backwards compatible!**
It will change how the template sections are handled for the Kafka cluster.
Depending on the exact configuration, this change might impact users and break or change their clusters.

Users can modify their custom resources to keep their existing configuration.
So it is important that we make sure they are aware of this change and can prepare for it.

## Rejected alternatives

### Using a feature gate

Using a feature gate to gate this change was considered.
However, this change is simple to work around by updating the custom resources.
The main issue we face is the user awareness.
Using a feature gate would only postpone when the change is rolled out to most users.
It would not improve the awareness.
For this reason, the use of feature gate was rejected.

### Merging the templates even at a deeper level

Merging the template fields even on a deeper level was considered.
For example, having different environment variables configured in the container template in `Kafka` and `KafkaNodePool` resources, merging them together and using both of them.
However, this would add significant complexity to the code base as well as to the user experience.
It would make it less predictable what the merge would actually result in.
And it would make it hard to unset the values set in the `Kafka` template from the `KafkaNodePool` template.
