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
* Do all partitions have enough in-sync replicas to be able to receive messages from producers (i.e. do the have more in-sync replicas than what is the minimal in-sync replicas setting)?
* Are the replicas leaders and ready to actually receive messages from producers

Strimzi currently maps the Kubernetes readiness probe to whether the Kafka broker is able to accept the client connections.
It monitors the broker state and when the listeners are started, it marks it as ready.
It does not reflect whether the replicas are in-sync, whether the partitions are under replicated etc.
This was intentional decisions and users can use other tools Strimzi provides to monitor the other aspects of Kafka readiness.

But there are some situations, where such readiness probe is not enough.
For example during voluntary disruptions and evictions.

### Disruptions

Kubernetes distinguishes between two types of disruptions: voluntary and involuntary.
Involuntary disruptions are situations when the pod need to be moved / restarted which are not controlled by Kubernetes.
This includes for example hardware failures or kernel failures and there is not much we can do about them.
Voluntary disruptions on the other hand are planned and controlled.
This includes for example draining worker nodes because of cluster scale-down or because the worker node needs some maintenance.

During the voluntary disruptions, Kubernetes will evict the pods from the node being drained.
That means that it will shut them down and start them on some other node (if available / possible).
During the evictions, Kubernetes will follow Pod Disruption Budget rules.
Pod Disruption Budget is a Kubernetes resource which defines how many pods matching some specific label selector can be evicted at once.
Pod evictions rely on the readiness probes.
Kubernetes will shutdown the allowed number of pods and wait until they are started somewhere else and become ready before shutting down the next pod(s).

This works well for some applications.
But not for the Strimzi Apache Kafka pods and their readiness probe.
Since the readiness probe doesn't guarantee that all replicas are in-sync again after the pod restart, it can happen that the next pod is restarted while the other pod is still syncing messages.
And this can cause partitions to become temporarily under replicated and unavailable to clients.
The only way how to make sure the Kafka cluster is not disrupted by voluntary disruptions which is available right now in Strimzi is to set the `maxUnavailable` option in the Pod Disruption Budget configuration for the Kafka brokers to `0`.
That will effectively block the automatic evictions of these pods and they would need to be moved manually (and during the manual process, users can control whether the replicas are in-sync or not).
In all other situations, voluntary disruptions can make the Kafka cluster temporarily unavailable for the clients.

## Motivation

The current design of the readiness probe Strimzi uses for Kafka broker pods is correct.
However, it is not sufficient in all situations as described in the previous section.
While the issue described there might happen only occasionally and it can be prevented by using manual evictions, it would be still nice if Strimzi could offer a better solution to those interested in it.

## Proposal

Strimzi already has a logic for rolling Kafka brokers while taking the state of the partitions and their replicas into account.
Strimzi monitors the state of the different replicas and rolls the next broker only when it knows that it will not cause any disruptions by making one or more partitions under replicated.
This logic is applied during the rolling updates done by Strimzi.
But users can also use it to roll individual brokers using the [`strimzi.io/manual-rolling-update` annotation](https://strimzi.io/docs/operators/latest/full/using.html#proc-manual-rolling-update-pods-str).

This logic can be also used to safely evict the broker pods during the voluntary disruptions.
When the voluntary disruptions such as node draining happens, Kubernetes will call the [Eviction API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#eviction-v1beta1-policy) to try to evict the Pod.
The eviction controller will evaluate the Pod Disruption Budgets matching given pod and when the `maxUnavailable` option is set to `0`, it will deny the eviction.
The calls to the Eviction API can be intercepted using an [admission web-hook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks).
The web-hook can evaluate whether the request is about any of the Kafka broker pods.
And in case it is, it use the `strimzi.io/manual-rolling-update` annotation to tell Strimzi Cluster Operator that this pod should be rolled.
When the operator sees the annotation in the next reconciliation, it will roll the pod using its own logic making sure the cluster availability is not affected.

This is already implemented and tested as part of the [Strimzi Drain Cleaner project](https://github.com/scholzj/strimzi-cistic-odpadu).
This proposal suggests to adopt this project under the strimzi umbrella.
It will live in the Strimzi GitHub organization in the repository called `drain-cleaner`.
Once it is adopted, we should:

* Create a CI for it in Azure Pipelines
* Cover it in Strimzi documentation
* Include it into Strimzi system tests (the Drain Cleaner project itself currently has only unit tests)
* Include the installation files in the Strimzi operators `install` folder

## Risks

The admission web-hook approach doesn't modify any of the existing operator code.
It works on the Kubernetes level only.
So there is no risk of breaking the operators.

The Drain Cleaner is also designed as an additional add-on to Strimzi operators.
Users can freely decide if they want to use it.
So there is minimal risk of breaking some existing environments.

## Affected / not affected projects

The Drain Cleaner project will have its own repository.
The only direct integration it has with other parts of Strimzi is the integration into the operator docs and system tests and having the installation files in the `install` folder of the operators repository.

## Compatibility

The Drain Cleaner is designed as an additional add-on to Strimzi operators.
It has no impact on backwards compatibility.
