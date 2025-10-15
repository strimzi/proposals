# Remove default Pod anti-affinity rules when rack awareness is enabled

This proposal suggests removing the default affinity the Strimzi Cluster Operator sets when rack awareness is enabled.

## Current situation

Strimzi supports rack awareness in Apache Kafka brokers as well as in client-based Kafka operands (Connect, MirrorMaker 2, HTTP Bridge).
Rack awareness in Kafka brokers helps to spread the partition replicas across the _racks_ (typically availability zones).
This is important to guarantee availability of the partitions and of the data they store by having them present in as many racks as possible.
Thanks to that, your Kafka cluster can survive a failure of one (or even more) racks.
But to make it effective, it is also critical to have the Kafka brokers equally distributed across the racks as well.

Rack awareness in client-based operands has a different purpose.
It allows the clients to consume data from the closest replicas (for example from follower replicas in the same rack instead of consuming from the leader replica in a different rack).
That might help to reduce costs, optimize network utilization, etc.
While it helps to keep the consumers available during an unavailability of one of the racks, the consumers can use the closest replicas regardless of being distributed across the racks or not.
So having the consumers equally distributed across the racks is not as critical as with the Kafka brokers.

To help to distribute the broker and consumer Pods, Strimzi adds a default Pod anti-affinity rule to the Pods.
That helps to have them distributed across the racks.
But because we do not have the full understanding of the infrastructure and architecture of the cluster, we cannot really enforce the distribution of the Pods.
For example, if the Kafka cluster is running in 3 racks and has 6 nodes, a required anti-affinity rule would allow us to schedule only 3 of the brokers.
And that is why we can set only _preferred_ rules with best effort guarantees instead of using the _required_ rules.

## Motivation

The preferred rules are not really sufficient to guarantee anything.
Therefore we normally suggest that users add their own rules that help them guarantee the Pods being distributed properly.
But having a mix of our own default rules (which are preferred only) with the user-defined rules makes it less clear how the Pods should be scheduled.

In addition to that, over time, many new features were implemented in Strimzi and in Kubernetes.
And some of them make the use of Pod anti-affinity for distributing the pods across racks obsolete.
For example, users can now use node pools or [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) to distribute the pods across the racks.

I believe that this makes the default affinity rules that Strimzi sets obsolete.

## Proposal

The default affinity rules will be removed from all the operands.
Users will be responsible for making sure the Pods are properly distributed either by setting their own affinity rules, topology spread constraints, or by using node pools.

### Implementation

The implementation of this is expected to be simple and straightforward.
We just remove the default rules and the related methods for merging the affinities.
This will also help to simplify our code base.

### Documentation

The documentation will be updated with this change.
We will make it clear that users should also add their own rules for spreading the Pods across the racks.
We will also add examples for doing this using topology spread constraints and per-rack node pools.

## Backwards compatibility

**This proposal is not backwards compatible!**
Users who currently rely on the default rules will be required to add their own rules.
However, given the default rules provide no guarantees, these users are already running with a bad configuration.
So if this change helps to force them to fix the configuration, it can be seen as beneficial.

## Rejected alternatives

### Using default topology spread constraint

When we initially implemented this, the topology spread constraints API was not available.
So we used Pod anti-affinity instead.
Topology spread constraints are better than anti-affinity rules, because they can distribute clusters with more Pods than available racks.
We could add default topology spread constraints instead of the default anti-affinity rules.

However, the default topology spread constraint can still cause confusion and conflicts.
Also, maybe in a few years, there will be an even better API to replace the topology spread constraints with and we will again need to break things for the users.
So maybe it is best to not have any default rules and leave it to the users.
Not setting any default rules is also better for the client-based operands which actually do not require any rules for the rack awareness to work properly.

So this alternative was rejected.

### Raise warnings when no affinity or topology spread constraints are configured

We could check the configuration.
And when the rack awareness is enabled and there are no affinity or topology spread constraints, we could raise a warning in logs and in the status conditions.
However, you can achieve the proper distribution even without these rules through node pools or storage.
So these warnings might raise permanent false alarms in some situations.
And when some rules are set, we have no way to actually check if these are the right rules.
So there might also be situations when the warnings won't be set, but the Pods are not distributed properly.
Given the warnings would be unreliable, this alternative was rejected.