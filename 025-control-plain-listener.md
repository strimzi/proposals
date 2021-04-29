# Control Plane Listener

This proposal suggests to use separate Control Plane listener and the plan how it should be introduced.

## Current situation

Strimzi currently has one internal listener.
It is used by the Kafka brokers for data replication but also for coordination between the controller and the other brokers.
And in addition to that, it is also used by the different Strimzi components:
* Operators
* Kafka Exporter
* Cruise Control

## Motivation

Kafka already for some time supports using separate listeners for data replication and coordination.
It was implemented as part of [KIP-291](https://cwiki.apache.org/confluence/display/KAFKA/KIP-291%3A+Separating+controller+connections+and+requests+from+the+data+plane).
The main reason for using separate listeners is that the replication traffic which is very data-intensive does not increase the latency the coordination traffic when separate listeners are used.
Strimzi should make use of it and have separate listeners for the different tasks.

## Proposal

Introducing the separate control plain listener has impact on backwards compatibility and on upgrade / downgrade procedures.
During the upgrade or downgrade, different brokers which are part of the same cluster would have a different configuration.
Some of them will try to coordinate using the dedicated control plane listener while others will try to coordinate using the replication listener.
Because of this, the different members of the cluster will not be able to communicate and the cluster will not be available to clients during the upgrade or downgrade.
This can be easily handled while upgrading.
But it is not possible to handle it when downgrading as the version the user would be downgrading from will not have a chance to revert the control plane listener changes.

To mitigate this, this proposal suggests to use [Feature Gates](https://github.com/strimzi/proposals/blob/main/022-feature-gates.md) to introduce the control plane listener over multiple releases and cause minimal disruption for the users.
Strimzi 0.23 will add new internal listener using port 9090.
This listener will be enabled by default and configured in the same way as the replication listener.
But it will not be configured as the control plane listener.

A new feature gate `ControlPlaneListener` will be added and disabled by default.
When enabled, this feature gate will configure the Kafka brokers to use the new listener for the control plane communication.
Replication, Strimzi operators, and other components such as Cruise Control or Kafka Exporter will keep using the existing replication listener.

Since it will be disabled by default at first, it will have no impact on backwards compatibility:
* Upgrades from earlier versions will add the new listener.
  But it will not be used for anything yet and will not break the Kafka cluster.
* Downgrades will remove the listener, but that will not cause any issues because it will not be used for anything.
* Users who would want to try it or use it already during the early releases will be able to upgrade to the new Strimzi version supporting it and enable the feature gate.
  That will configure the Kafka broker to use the new listener.
  They will be also able to disable it again later.
  In case they would need to do a downgrade to a Strimzi version not supporting the `ControlPlaneListener` feature gate, they would need to disable it first.

Later - after several Strimzi releases - the `ControlPlaneListener` feature gate will be changed to enabled by default.
By that time, most users upgrading from the Strimzi versions which already support this feature gate will already have the new listener on port 9090 enabled.
So the upgrade will proceed without any complications.
Similarly, users downgrading to Strimzi versions already supporting the `ControlPlaneListener` feature gate will be able to downgrade without any problems.
Only users upgrading from / downgrading to Strimzi versions not supporting the `ControlPlaneListener` feature gate will need to manually disable it first in order to allow it to happen without any unavailability of the Kafka cluster.

At the end, after more Strimzi release, the `ControlPlaneListener` feature gate will move to the GA phase.
The control plane listener will be enabled by default and the feature gate will be removed.
It will not be possible to disable it anymore.
Upgrade from / downgrades to Strimzi versions not supporting the `ControlPlaneListener` will not be possible anymore.

The following table shows in which Strimzi versions is the state of the feature gate expected to change.
This plan is subject to change in case of any problems appear doing the different phases.

| Phase | Strimzi versions       | Default state                                          |
|:------|:-----------------------|:-------------------------------------------------------|
| Alpha | 0.23, 0.24, 0.25, 0.26 | Disabled by default                                    |
| Beta  | 0.27, 0.28, 0.29, 0.30 | Enabled by default                                     |
| GA    | 0.31 and newer         | Enabled by default (without possibility to disable it) |

_(The actual version numbers are subject to change)_

Strimzi operators and other components (Cruise Control, Kafka Exporter) will keep using the replication listener on port 9091 for communication with the Kafka brokers.

## Compatibility

The introduction of this feature is designed to minimize the compatibility impacts.

## Affected components

Only the Cluster Operator and Kafka brokers are impacted by this proposal.
Other operators or other operands are not affected.

## Rejected alternatives

Introducing this feature without the possibility to downgrade back to previous Strimzi versions was considered but rejected.