# Adopt the Drain Cleaner utility

## Current situation

Kubernetes readiness probes indicate whether a pod is ready to start serving clients.
The readiness probe is not used to just indicate the state of the pod to the user.
Kubernetes use it for several different things.
For example:

* Indicator whether a traffic should be routed to the pod
* During voluntary disruptions / evictions

The Kubernetes readiness probe concept does not naturally fit into how Apache Kafka works.
Apache Kafka brokers have several different levels of readiness which do not always correspond to Kubernetes view of readiness.
For example:

* Does the broker accept client connections? (this one corresponds roughly to Kubernetes readiness concept)
* Are all partition replicas in sync?
* Do all partitions have enough in-sync replicas to be able to receive messages from producers (i.e. do the have more in-sync replicas than what is the minimal in-sync replicas sestting)?
* Are the replicas leaders and ready to actually receive messages from prodcuers

Strimzi currently maps the Kubernetes readiness probe to whether the Kafka broker is able to accept the client connections.
It monitors te broker state and when the listebners are started, it marks it as ready.
It does not reflect whether the replicas are in-sync, whether the partitions are under replicated etc.
This was intentional decisions and users can use other tools Strimzi provides to monitor the other aspects of Kafka readiness.

But there are some situations, where such readiness probe is not enough.
For example during voluntary disruptions and evictions.

### Disruptions

Kubernetes distinguishes between two types of disruptions: voluntary and involuntary.
Involuntary disruptions are situstions when the pod need to be moved / restarted which are not controled by Kubernetes.
This includes for example hardware failures or kernel failures and there is not muich we can do about them.
Voluntary disruptions on the other hand are planned and controlled.
This includes for example draining worker nodes because of cluster scale-down or because the worker node needs some maintenance.

During the voluntary disruptions, Kubernetes will evict the pods from the node being drained.
That means that it will shut them down and start them on some other node (if available / possible).
During the evictions, Kubernetes will follow Pod Disruption Budget rules.
Pod Disruption Budget is a Kubernetes resource which defines how many pods matching some specific label selector can be evisted at once.
Pod evictions rely on the readiness probes.
Kubernetes will shutdown the allowed number of pods and wait until they are started somewhere else and become ready before shutting down the next pod(s).

This works well for some applications.
But not for the Strimzi Apache Kafka pods and their readiness probe.
Since the readiness probe doesn't guarantee that all replicas are in-sync again after the pod restart, it can happen that the next pod is restarted while the other pod is still syncing messages.
And this can cause partitions to become temporarily under replicated and unavailable to clients.
The only way how to make sure the Kafka cluster is not disrupted by voluntary disruptions which is available right now in Strimzi is to set the `maxUnavailable` option in the Pod Disruption Budget configuration for the Kafka brokers to `0`.
That will effectively block the automatic evictions of these pods and they would need to be moved manually (and during the manual process, users can control whether the replcias are in-sync or not).
In all other situations, voluntary disruptions can make the Kafka cluster temporarily unavailable for the clients.

## Motivation

Having some solution which would protect from the disks getting full would be a good improvement for Strimzi users.

## Proposal

Strimzi should adopt the [Kafka Static Quota plugin](https://github.com/lulf/kafka-static-quota-plugin).

### Quota types

It provides two types of quotas:

#### Per-broker produce and fetch quotas

Apache Kafka itself offers only quotas per client / user.
The Kafka Static Quota plugin allows configuring of an overall produce and/or fetch quota per-broker.
The quota will be distributed between the clients connected to the broker.
The produce / fetch quotas currently don't support per-listener quotas.
There is always only one quota for all listeners.
The replication traffic is not counted into the quota.

#### Storage quotas

The Kafka Static Quota plugin allows users to configure two storage limits: soft and hard.
After the soft limit is breached, it will start throttling the producers.
The allowed throughput is linearly decreased until the hard limit is reached.
Once the hard limit is reached, the allowed produce throughput will be 0.
The storage quotas currently don't support JBOD storage, only brokers with single log directory.

### Plugin adoption

In order to include the plugin into Strimzi images, we would need it to be available in Maven repositories.
This proposal suggests to fork the plugin under Strimzi and continue its development.
The name repository should be named `kafka-quotas-plugin`.
We can set up Azure Pipelines build for it and push it to Maven repositories.

We will add the plugin to our Kafka container images via the third-part libraries mechanism.
At this time, we should not add any integration into the Strimzi Kafka CRD.
The plugin and its quotas can be configured through `.spec.kafka.config`.
But we should document it and include it in our system tests.

We should also work on additional features.
For example:

* Support for JBOD storage
* Support for per-listener configuration

## Risks

The quota plugin will not be enabled by default, so there is minimal risk of it causing any problems.

## Affected / not affected projects

This proposal affects only the plugin and the operators repository where it will be added to the Apache Kafka images..

## Compatibility

The quota plugin will not be enabled by default, so there should be no impact on backwards compatibility.
