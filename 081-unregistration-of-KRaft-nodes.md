# Unregistration of KRaft nodes

This proposal discusses the support for KRaft node unregistration when removing the nodes.
This is one of the missing features in Strimzi's KRaft support.

## Current situation

When removing Apache Kafka nodes from a KRaft-based cluster, the nodes should be unregistered using the Apache Kafka Admin API.
When the nodes are not properly unregistered, the cluster will keep tracking them and they would cause problems later.
For example, when you upgrade your cluster to a new Kafka minor version and try to also bump the metadata version, the metadata version update will be refused because the removed nodes that are still registered with the old Kafka version and do not support the new metadata version.

Strimzi currently doesn't support the node unregistration.
This has two main reasons:
* Apache Kafka did not properly document this process and we found out about it only by coincidence.
* Apache Kafka has only limited support for unregistering nodes.
  While the Kafka Admin API supports unregistration of the nodes, it does not let you list the registered nodes.
  So you do not know what nodes should be unregistered.
  This is causing a problems to stateless management tools such as Strimzi operators who might not reliably know what nodes existed before.
  The proper solution for this is part of the [KIP-1073](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1073%3A+Return+inactive+observer+nodes+in+DescribeQuorum+response) that is currently under discussion.

## Motivation

Despite the missing APIs being an important limitation of the KRaft implementation, Apache Kafka does not plan to address this in Apache Kafka 3.x releases.
But this limitation raised a lot of discussions and concerns in the Strimzi community after we added it to our list of known KRaft limitations.
So this proposal tries to address it in an alternative way that is not dependent on Apache Kafka and its Admin API.

## Proposal

This proposal explains the initial workaround that can be implemented purely on the Strimzi level to handle node unregistration.
It also outlines the future / long-term solution that will be used once the implementation of this feature is completed in Apache Kafka.

### Initial workaround implementation

In order to be able to unregister the removed Kafka nodes, we need to be aware of the nodes that were removed.
Strimzi Cluster Operator is aware of the scale-downs that it directly executed.
These are the scale-downs resulting from changing the `.spec.replicas` field in the `KafkaNodePool` resources.
But this information is not persisted anywhere and will not survive a crash or restart of the operator or a reconciliation failure that might happen between the nodes being scaled down and the node unregistration.
Strimzi Cluster Operator is also not directly involved in some scale-down operations such as when a KafkaNodePool resource is deleted.
In this case, the scale-down is done by Kubernetes garbage collection.

To work around this limitation, the `Kafka` custom resource will be extended and will store a full list of the node IDs in its status section.
During the reconciliation, this field will be used to get the information about the previously existing nodes.
The _previously existing_ nodes from the status will be compared with the nodes that are still in use to determine which nodes were removed and should be unregistered.
Only after unregistering the nodes, the node IDs will be removed from the field in the `Kafka` CR status section.

This way, the information about the removed nodes:
* Will be persisted even in case of operator crash or restart
* Will be persisted even in case of reconciliation failure between the scale-down and the node unregistration
* Will allow us to detect nodes removed by other means such as by deleting a `KafkaNodePool` resource

The node unregistration will be done in the reconciliation before the KRaft metadata version update.
That way, the nodes should be unregistered before the metadata update happens in case the scale-down and KRaft metadata upgrade are done in the same reconciliation.
The unregistration will use the corresponding Kafka Admin API method ([`unregisterBroker`](https://kafka.apache.org/38/javadoc/org/apache/kafka/clients/admin/Admin.html#unregisterBroker(int))).

#### API changes

The new field in the `Kafka` CR will be called `nodeIds` (the same name as already used in `KafkaNodePool` CR).
As the new field will be used only temporarily by the workaround implementation, the description of the field will make it clear that:
* This field is supposed to be used for internal purposes only
* It will be removed in the future once it is not needed anymore

The `.status` section with the new field will look something like this:

```yaml
  status:
    nodeIds:
      - 0
      - 1
      - 2
      - 1000
      - 2000
      - 3000
```

### Future implementation with Kafka Admin API

Once Apache Kafka implements the [KIP-1073](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1073%3A+Return+inactive+observer+nodes+in+DescribeQuorum+response), we will do the following:

* Deprecate and immediately stop using the `.status.nodeIds` field in the Kafka custom resource status section.
  (This field has to remain part of the Strimzi API until the new API version `v1` where it can be removed.)
* Use the new Kafka Admin API to get the list of registered nodes and use it as the basis for unregistering the removed nodes.

### Limitations

This proposal provides a reliable solution for any Kafka nodes that are removed from the Kafka clusters after it was implemented.
But it will not be able to identify any nodes that were added in the past.
So for any pre-existing KRaft clusters that removed some nodes in the past, users will still need to unregister these nodes manually using the Kafka Admin API.

This limitation only exists with the initial implementation based on tracking the node IDs in the status section of the Kafka CR.
It will not apply to the future implementation based on the Kafka Admin API described in the previous section as the Kafka Admin API will provide all registered nodes.

## Affected projects

This proposal affects the Strimzi Cluster Operator only.

## Backwards compatibility

This proposal is fully backwards compatible.

## Rejected alternatives

### Wait until the implementation is complete in Apache Kafka

The main alternative that was considered was to do nothing for the time being and wait until Apache Kafka completes the KRaft implementation.
Once that would be available, we use implement the node unregistration fully relying on the Apache Kafka Admin API.
Until then, users will need to _unregister_ the nodes manually.

However, there is a lot of uncertainty over when will this be implemented in Apache Kafka.
As of today, the KIP-1073 is still under discussion, so it is not clear whether it will be done at least in Apache Kafka 4.0 or only later.
And given the discussions and concerns this raised on Strimzi Slack, this alternative was rejected.

### Using the `.status.nodeIds` field from the `KafkaNodePool` resource

The `KafkaNodePool` resources already have their own `.status.nodeIds` field that lists all the nodes belonging to the node pool.
However, this field cannot be used for this purpose, because:
* It would not allow us to capture the nodes removed through a node pool deletion
* It needs to be updated much earlier in reconciliation flow in order to capture the node IDs being added or removed from the node pool
