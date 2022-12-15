# Stable identities for Kafka Connect worker nodes

This proposal covers improvements to how we deploy and manage Kafka Connect.
It also covers Kafka Mirror Maker 2 which is based on top of Kafka Connect.
Everything in this proposal applies implicitly also to Kafka Mirror Maker 2.

## Current situation

Kafka Connect cluster can consist of one or more worker nodes.
The Kafka Connect worker nodes run as Kubernetes Pods.
Strimzi is using the Kubernetes Deployment to run the Kafka Connect workers.
The number of `replicas` in the Kubernetes Deployment corresponds to the number of desired Kafka Connect worker nodes.
Strimzi Cluster Operator (CO) creates the Deployment resource and Kubernetes creates the Pods based on the Deployment.
Each Pod runs a single Java Virtual Machine (JVM) which acts as a Kafka Connect worker node.

The individual Kafka Connect worker nodes which are members of the same cluster communicate with each other through a REST API.
When configuring the worker nodes, each is assigned an advertised address and port which they use for the communication.
The advertised address forms the identity of the worker node.
When Kubernetes creates the Pods, they have a randomized name and are assigned an unique IP address.
This IP address is used as the advertised address and gives the worker node its identity.. 

When connectors and their tasks are deployed, they are scheduled to the individual worker nodes based on their identity.
Each connector task is running on a specific worker node with given identity (IP address).

When changes need to be rolled out to the Kafka Connect cluster, Strimzi CO will modify the Kubernetes Deployment.
Based on the changes, Kubernetes will roll the Connect Pods.
By default, Strimzi configures the Deployment to use the `RollingUpdate` strategy.
So out of the box, Kubernetes will:
1) Start a new Kafka Connect Pod
2) Wait until the new Pod gets ready
3) Stop the old Connect Pod
4) Move to next Pod (if any) and repeat the same

The new Pod will be assigned a new IP address which it will use for its advertised address.
Because of the new address, it will have a new _identity_.
So during the rolling update, the Kafka Connect cluster will not see this operation as a restart of existing worker node.
Instead, it will see it as a new worker node joining the cluster and later a previously existing node leaving the cluster.

_Note: Optionally, users can enable the `Recreate` deployment strategy in the `KafkaConnect` custom resource._
_In that case, Kubernetes will first delete the old Pod and only then create the new Pod._
_However, the new Pod still gets a new IP address so the identity of the Pod changes._

This means that the worker nodes where the connector tasks are scheduled all disappear during the rolling update and the tasks need to be rescheduled.
It will take some time (by default 5 minutes) for the cluster to detect that the old nodes left the cluster and reschedule the tasks on other nodes.
Until this happens, it is possible that the tasks will be scheduled to one of the pods which was deleted and are thus not running.

## Proposal

Giving the Kafka Connect workers stable identities should improve the scheduling of the connector tasks and minimize the number of task rebalances.
Since the worker identity is based on the Pod address, it means that we need to give the pods stable addresses.

To achieve that, we should migrate the Kafka Connect clusters from Kubernetes Deployment resource to the StrimziPodSet resources.
The Pods created by the StrimziPodSet will use stable pod names instead of the randomized names used for the Pods managed by Kubernetes Deployments.
The pods will be named `<cluster-name>-connect-<index>` where the `<cluster-name>` is the name of the custom resource and the `<index>` will be a sequential index number starting with 0.
For example, the pod names for Kafka Connect cluster created by `KafkaConnect` custom resource named `my-connect` with 3 replicas will be `my-connect-connect-0`, `my-connect-connect-1`, and `my-connect-connect-2`
This pattern follows the way the pods are currently named as well.

In addition to the StrimziPodSet resource used to manage the Pods, Strimzi will create a headless Kubernetes Service to give these pods a stable DNS names.
The headless service will be named `<cluster-name>-connect`.
For example `my-connect-connect`.
The existing `ClusterIP` type service names `<cluster-name>-connect-api` will remain unchanged and will round-robin across all the Connect worker pods.
The combination of the stable Pod names and the headless service will mean that each Kafka Connect worker pod will have a stable DNS name in the format `<cluster-name>-connect-<index>.<cluster-name>-connect.<namespace>.svc`.
For example `my-connect-connect-0.my-connect-connect.myproject.svc`.
This DNS name will be used as the advertised address for the worker and give it the stable identity.

### Rolling updates

Since the Pods will now have stable names, the rolling update will always first delete the old pod before creating the new pod.
There is no way to first create the new pod before deleting the old Pod.

The CO will decide if a rolling update is needed based on the differences between the existing Pods and the desired Pods.
Today, this is done by the Kubernetes Deployment Controller.
With StrimziPodSets, this would need to be done by the CO directly.
The CO will:
* Loop through all pods / replicas starting with the lowest index number and ending with the highest index number
* For each pod, it will:
    * Stop the old pod
    * Wait for the new Pod to be started (this will be done by the StrimziPodSetController)
    * Once the new Pod is started, it will wait until it becomes ready before moving to the next replica

### Scaling up or down

When scaling-up, Strimzi will start new Connect pods with the lowest available (unused) index numbers.
When scaling-down, the Pods will removed starting with the highest used index numbers.
Since all Pods will share the same configuration, there is no need to support things such as scaling own some pods from the middle or beginning of the sequence.

### Feature Gate

The stable identities for the Kafka Connect worker nodes will be introduced gradually using a Feature Gate.
There will be only one Feature Gate for both Kafka Connect and Kafka Mirror Maker 2.
The new Feature gate will be named `StableConnectIdentities`.
To enable this feature gate, the `UseStrimziPodSets` feature gate would need to be enabled as well.
The following table shows the expected graduation of the feature gate:

| Phase | Strimzi versions       | Default state                                          |
|:------|:-----------------------|:-------------------------------------------------------|
| Alpha | 0.34, 0.35             | Disabled by default                                    |
| Beta  | 0.36, 0.37             | Enabled by default                                     |
| GA    | 0.38 and newer         | Enabled by default (without possibility to disable it) |

### Migration from Kubernetes Deployments to StrimziPodSets

The migration from Kubernetes Deployments to StrimziPodSets will be used in the following situations:
* Upgrading from Strimzi versions which does not use StrimziPodSets for Kafka Connect to a Strimzi version using StrimziPodSets for Kafka Connect
    * This includes upgrades from Strimzi versions before introduction of the `StableConnectIdentities` feature gate
    * Or upgrades from version with disabled `StableConnectIdentities` feature gate to version where it is enabled
* Enabling of the `StableConnectIdentities` feature gate in Strimzi version which supports it

The migration will happen automatically and will use the following steps:
* If the user is using the `RollingUpdate` deployment strategy, Strimzi will
    1) Create an empty StrimziPodSet without any Pods
    2) Add new Kafka Connect Pod to the StrimziPodSet
    3) Wait for the new Pod to become ready
    4) Scale-down the Deployment by one replica to remove one Kafka Connect Pod from the Deployment
    5) ... repeat steps 2-4 until the StrimziPodSet has the desired number of Pods and the Deployment has 0 replicas
    6) Delete the Deployment
* If the user is using the `Recreate` deployment strategy, Strimzi will
    1) Create an empty StrimziPodSet without any Pods
    2) Scale-down the Deployment by one replica to remove one Kafka Connect Pod from the Deployment
    3) Add new Kafka Connect Pod to the StrimziPodSet
    4) Wait for the new Pod to become ready
    5) ... repeat steps 2-4 until the StrimziPodSet has the desired number of Pods and the Deployment has 0 replicas
    6) Delete the Deployment

Doing the things in the right oder is important because otherwise it might happen that there will be no space in the cluster to start the new Pod and the migration might get stuck.

### Downgrades

The migration from StrimziPodSets to Kubernetes Deployments will be used in the following situations:
* Downgrading from Strimzi versions with enabled `StableConnectIdentities` feature gate to Strimzi version with disabled `StableConnectIdentities` feature gate
* Disabling of the `StableConnectIdentities` feature gate in Strimzi versions which supports it

The migration will happen automatically and will use the following steps:
* If the user is using the `RollingUpdate` deployment strategy, Strimzi will
    1) Create a new Deployment with 0 replicas
    2) Scale-up the Deployment by increasing the replica count by 1 to create a new Pod
    3) Wait for the new Pod to become ready
    4) Remove one Pod from the StrimziPodSet to get it removed
    5) ... repeat steps 2-4 until the Deployment has the desired number of Pods and the StrimziPodSet has no Pods
    6) Delete the StrimziPodSet
* If the user is using the `Recreate` deployment strategy, Strimzi will
    1) Create a new Deployment with 0 replicas
    2) Remove one Pod from the StrimziPodSet to get it removed
    3) Scale-up the Deployment by increasing the replica count by 1 to create a new Pod
    4) Wait for the new Pod to become ready
    5) ... repeat steps 2-4 until the Deployment has the desired number of Pods and the StrimziPodSet has no Pods
    6) Delete the StrimziPodSet

Doing the things in the right oder is important because otherwise it might happen that there will be no space in the cluster to start the new Pod and the migration might get stuck.

#### Downgrades to Strimzi versions before the `StableConnectIdentities` feature gate

Automatic downgrades from Strimzi version with enabled `StableConnectIdentities` feature gate to a Strimzi version which does not support the `StableConnectIdentities` feature gate will not be supported.
Manual downgrade will be still possible.
It will require the user to manually delete the `StrimziPodSet` used to manage the Kafka Connect pods and (possibly) also delete the corresponding Kafka Connect Pods.
Deleting the Kafka Connect Pods manually will be needed only when downgrading to a Strimzi version which does not support or use StrimziPodSets.

## Impacted projects / components

This proposal impacts Kafka Connect and Kafka Mirror Maker 2 clusters.
Any other operands (including Kafka Mirror Maker 1) or other Strimzi projects are not impacted by this.

## Rejected alternatives

### Using StatefulSets instead of StrimziPodSets

I considered using StatefulSets instead of StrimziPodSets.
Some of the issues we had with StatefulSets when using the for the Kafka brokers do not apply to Kafka Connect today.
For example, we do not need asymmetric configuration of the worker nodes since Kafka Connect doesn't support advanced scheduling and you thus cannot ensure that some tasks get scheduled only on some _special_ nodes.

However, this idea was ultimately rejected as StrimziPodSets might give us more options in the future.
For example:
* If we in the future support stretched Kafka clusters across multiple Kubernetes clusters, it might make sense to support stretched clusters also for Connect.
  While you cannot easily schedule the tasks across the Pods running on the different Kubernetes clusters, having the workers running in all of will provide improved availability since the tasks can be more easily rescheduled if one of the Kubernetes clusters goes offline.
* It is also possible that in the future Kafka Connect will improve support for scheduling the connector tasks on particular nodes.
  Using StrimziPodSets would make it easier to leverage such feature.
