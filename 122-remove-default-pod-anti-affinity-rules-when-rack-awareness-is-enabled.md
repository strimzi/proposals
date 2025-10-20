# Remove default Pod anti-affinity rules when rack awareness is enabled

This proposal suggests removing the default Pod anti-affinity rule the Strimzi Cluster Operator sets when rack awareness is enabled.

## Current situation

Strimzi supports rack awareness in Apache Kafka brokers as well as in client-based Kafka operands (Connect, MirrorMaker 2, HTTP Bridge).
Rack awareness in Kafka brokers helps to spread the partition replicas across the _racks_ (typically availability zones).
Placing replicas in as many racks as possible helps ensure availability of partitions and the data they store.
This configuration allows the Kafka cluster to remain operational even if one or more racks fail.
But to make it effective, brokers must be evenly distributed across racks.

Rack awareness in client-based operands serves a different purpose.
It allows clients to consume data from the nearest replicas, for example, from follower replicas in the same rack rather than the leader in a different rack.
This can reduce costs and optimize network utilization.
While rack awareness can improve consumer availability during rack failures, clients can still consume from the nearest replicas, even if they aren’t evenly distributed across racks.
So having the consumers equally distributed across the racks is not as critical as with the Kafka brokers.

To help distribute broker and consumer Pods across racks, Strimzi applies a default Pod anti-affinity rule.
But because Strimzi does not know the full cluster infrastructure and architecture, it cannot enforce pod distribution.
For example, when rack awareness is enabled, if the Kafka cluster is running in 3 racks and has 6 nodes, a _required_ anti-affinity rule would limit scheduling to just 3 of the brokers, one per rack.
That’s why Strimzi uses _preferred_ rules (best effort) instead of using _required_ rules.

## Motivation

The preferred rules are not really sufficient to guarantee anything.
Therefore, we normally suggest that users add their own rules that help ensure proper pod distribution.
But having a mix of our own default rules (which are preferred only) with the user-defined rules makes it less clear how the Pods should be scheduled.

In addition to that, over time, many new features were implemented in Strimzi and in Kubernetes.
And some of them make the use of Pod anti-affinity for distributing the pods across racks obsolete.
For example, users can now use node pools or [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) to distribute the pods across the racks.

These changes make Strimzi’s default anti-affinity rules unnecessary.

## Proposal

The default pod anti-affinity rules rules will be removed from all the operands.
Users will be responsible for making sure the Pods are properly distributed either by configuring their own affinity rules, topology spread constraints, or node pools.

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
Users who currently rely on the default rules will be required to add their own rules from Strimzi 0.49.0 on.
However, given the default rules provide no guarantees, these users are already running with a bad configuration.
So if this change helps to force them to fix the configuration, it can be seen as beneficial.

## Rejected alternatives

### Using default topology spread constraint

When we initially implemented this, the topology spread constraints API was not available.
So we used Pod anti-affinity instead.
Unlike anti-affinity rules, topology spread constraints can distribute pods evenly even when there are more pods than racks.
We could add default topology spread constraints instead of the default anti-affinity rules.

However, the default topology spread constraint can still cause confusion and conflicts.
Topology spread constraints may eventually be replaced by newer APIs, which could again require breaking changes for users. 
To avoid this, it’s better not to set any default scheduling rules and instead leave configuration decisions to the user.
Not setting any default rules is also better for the client-based operands which actually do not require any rules for the rack awareness to work properly.

So this alternative was rejected.

### Raise warnings when no affinity or topology spread constraints are configured

We could check the configuration.
And when the rack awareness is enabled and there are no affinity or topology spread constraints, we could raise a warning in logs and in the status conditions.
However, you can achieve the proper distribution even without these rules through node pools or storage.
So these warnings might raise permanent false alarms in some situations.
And when some rules are set, we have no way to actually check if these are the right rules.
So there might also be situations when the warnings won't be set, but the Pods are not distributed properly.
Given that warnings would be too unreliable, this alternative was rejected.