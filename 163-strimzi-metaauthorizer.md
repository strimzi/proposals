# Strimzi Meta-Authorizer

This proposal introduces the Strimzi Meta-Authorizer - an Apache Kafka authorizer tailored for Strimzi that authorizes the internal Strimzi components using hardcoded rules and delegates all other authorization decisions to the authorizer configured by the user.
It allows Strimzi to stop configuring its internal components as Kafka super-users.

## Current Situation

Every Strimzi-based Apache Kafka cluster has several internal components that connect to it using the Kafka protocol:

* Kafka nodes (brokers and controllers) for replication, KRaft communication, and request forwarding.
  This also includes the plugins running inside the Kafka nodes, such as the Cruise Control Metrics Reporter, the Strimzi Quotas plugin, or the Tiered Storage Remote Log Metadata Manager.
* Cluster Operator
* Topic Operator
* User Operator
* Cruise Control
* Kafka Exporter
* (In the future) Strimzi Sova / Extra Metrics

These components connect to the internal control plane (`CONTROLPLANE-9090`) and replication (`REPLICATION-9091`) listeners.
They authenticate either using mTLS (for example `User:CN=my-cluster-entity-topic-operator,O=io.strimzi`) or, with the [SEP-150 proposal](https://github.com/strimzi/proposals/blob/main/150-configurable-security-of-internal-communication.md), using Kubernetes Service Account tokens (for example `User:system:serviceaccount:myproject:my-cluster-entity-operator`).

When authorization is enabled in the `Kafka` custom resource, Strimzi configures all these components as super-users:

```properties
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:CN=my-cluster-kafka,O=io.strimzi;User:CN=cluster-operator,O=io.strimzi;User:CN=my-cluster-entity-topic-operator,O=io.strimzi;User:CN=my-cluster-entity-user-operator,O=io.strimzi;User:CN=my-cluster-kafka-exporter,O=io.strimzi;User:CN=my-cluster-cruise-control,O=io.strimzi
```

The super-users bypass the authorizer completely.
They are allowed to do anything, on any resource, and on any listener.
The same super-users are used regardless of whether the user uses the `simple` authorization or a `custom` authorizer.

## Motivation

Configuring the internal components as super-users is simple and robust.
But it gives every component much more rights than it actually needs:

* Kafka Exporter only needs to describe topics and consumer groups to export the metrics.
  But as a super-user, it can read and write all data, delete topics, or change ACLs.
* The User Operator only manages ACLs, quotas, and SCRAM-SHA credentials.
  But as a super-user, it can read all data in the Kafka cluster.
* The Topic Operator only manages the topics.
  But as a super-user, it can also read all the data stored in them, manage ACLs, or change quotas.
* Cruise Control needs access only to its own topics and to the cluster-level rebalancing APIs.
  But as a super-user, it can read all data from any topic.

This goes against the principle of least privilege.
When the credentials of any of these components leak (for example, the Kafka Exporter certificate or a Service Account token of one of the components), the attacker gets full administrative rights to the whole Kafka cluster.
With least-privilege rules, the leaked Kafka Exporter credentials would allow the attacker only to describe the topics and consumer groups.

The super-users also apply to all listeners, not just to the internal listeners.
Kafka matches the super-users only by the principal name.
It does not check which listener the client connected to or how the principal was authenticated.
So any client that authenticates on a user listener with the same principal as one of the internal components becomes a super-user as well.

With the mTLS authentication, the internal components use certificates signed by the cluster CA, and the user listeners trust the clients CA instead.
But using different CAs does not protect the super-user principals.
A user can get a certificate signed by the clients CA (or by any other CA trusted by a user listener) with the same subject as one of the internal components, for example `CN=my-cluster-kafka-exporter,O=io.strimzi`.
This is easy, for example, when users manage their own clients CA and issue their own user certificates.
A client using such a certificate would have the principal `User:CN=my-cluster-kafka-exporter,O=io.strimzi` and would become a super-user.
The same applies to `custom` authentication or to principal mapping rules that can produce the same principal names as the internal components.

The Service Account-based authentication from [SEP-150](https://github.com/strimzi/proposals/blob/main/150-configurable-security-of-internal-communication.md) makes this even easier.
Users might want to use the same Kubernetes Service Account tokens to authenticate their own applications on the user listeners (for example using the `custom` authentication with an OAuth library).
In such a case, any application able to get a token for one of the Strimzi Service Accounts would become a super-user on the user listener as well.

Apache Kafka does not support super-users or ACLs scoped to a specific listener.
And Strimzi cannot use the ACLs to grant the internal components only the rights they need, because:

* ACLs work only with the `StandardAuthorizer`.
  Custom authorizers (such as OPA or Keycloak-based authorizers) use their own rules and ignore the Kafka ACLs.
* The Kafka nodes need to be authorized before the ACLs are loaded from the KRaft metadata log (for example, to register with the controllers and replicate the metadata).
* The ACLs are managed through the Kafka Admin API and can be changed or deleted by anyone with the right permissions.

That is why this proposal suggests a new authorizer that understands the Strimzi architecture and the needs of the Strimzi components.

## Proposal

This proposal introduces a new Kafka authorizer called _Strimzi Meta-Authorizer_ (`io.strimzi.kafka.metaauthorizer.MetaAuthorizer`).
The Meta-Authorizer is configured as the Kafka authorizer.
When it starts, it creates the user-configured authorizer (for example, the `StandardAuthorizer` when using the `simple` authorization) and uses it as a delegate.
For each authorization request, it decides whether to handle it itself or whether to delegate it.

### Authorization Decisions

The decision depends on the listener that received the request and on the principal:

| Listener          | Principal                     | Decision                                                                      |
|:------------------|:------------------------------|:------------------------------------------------------------------------------|
| User listener     | Any principal                 | Delegated to the user-configured authorizer                                   |
| Internal listener | Internal Strimzi component    | Hardcoded rules of the component; anything outside the rules is denied        |
| Internal listener | Any other principal           | Delegated to the user-configured authorizer                                   |

The internal components therefore get only the rights they need and only on the internal listeners.
On the user listeners, the internal principals are just regular users and are authorized by the user-configured authorizer.

The internal components are _strictly denied_ anything outside their rules.
The request is not delegated to the user-configured authorizer, and the internal components cannot get any additional rights through ACLs or rules of the user-configured authorizer.
That ensures that the rules are always enforced regardless of the user-configured authorizer and its configuration.

The third row of the table is needed because of how KRaft forwards requests.
The brokers forward many administrative requests (for example `CreateTopics`, `DeleteTopics`, `CreatePartitions`, `CreateAcls`, `DeleteAcls`, `AlterClientQuotas`, or `IncrementalAlterConfigs` for topics) to the controllers wrapped in an `Envelope` request.
The controller authorizes the `Envelope` request with the principal of the broker.
Then it authorizes the inner request with the principal of the original client, but with the name of the controller listener (`CONTROLPLANE-9090`).
To not break the forwarded requests of the regular users, requests from all principals that do not belong to the internal Strimzi components are delegated even on the internal listeners.
The forwarded requests from the internal components are authorized using the hardcoded rules.

When no user-configured authorizer is set, the Meta-Authorizer allows all requests that would be delegated (the same as running Kafka without an authorizer).
The internal components are still restricted to their rules.
Strimzi will always configure the delegate, so this applies only when the Meta-Authorizer is used outside of Strimzi.

### Hardcoded Rules

The rules are derived from the Kafka APIs that each component actually uses.
The rules only allow operations.
The implied Kafka operations apply as well (`Read`, `Write`, `Delete`, and `Alter` imply `Describe`, `AlterConfigs` implies `DescribeConfigs`).

| Component                    | Cluster                                              | Topics                                                                                                       | Groups          |
|:-----------------------------|:-----------------------------------------------------|:-------------------------------------------------------------------------------------------------------------|:----------------|
| Kafka nodes                  | Everything                                           | Everything                                                                                                   | Everything      |
| Cluster Operator             | `Describe`, `DescribeConfigs`, `Alter`, `AlterConfigs` | All topics: `Describe`, `DescribeConfigs`                                                                  | -               |
| Topic Operator               | `Describe`, `DescribeConfigs`                        | All topics: `Create`, `Delete`, `Alter`, `Describe`, `DescribeConfigs`, `AlterConfigs`                       | -               |
| User Operator                | `Describe`, `Alter`, `DescribeConfigs`, `AlterConfigs` | -                                                                                                          | -               |
| Cruise Control               | `Describe`, `DescribeConfigs`, `Alter`, `AlterConfigs` | All topics: `Describe`, `DescribeConfigs`, `AlterConfigs`; Cruise Control topics: `Create`, `Read`, `Write`, `Alter` | -    |
| Kafka Exporter               | `Describe`                                           | All topics: `Describe`                                                                                       | All groups: `Describe` |
| Strimzi Sova / Extra Metrics | `Describe`                                           | All topics: `Describe`                                                                                       | All groups: `Describe` |

The components use the following Kafka APIs:

* **Cluster Operator**: `DescribeCluster`, `DescribeMetadataQuorum`, `ListTopics`, `DescribeTopics`, `DescribeConfigs` (brokers and topics), `IncrementalAlterConfigs` (brokers and broker loggers), `UpdateFeatures`, `UnregisterBroker`, `UnregisterController`, `DescribeClientQuotas`, and `AlterClientQuotas`.
* **Topic Operator**: `CreateTopics`, `DeleteTopics`, `CreatePartitions`, `DescribeTopics`, `DescribeConfigs` (topics and brokers), `IncrementalAlterConfigs` (topics), `ListPartitionReassignments`, and `DescribeCluster`.
* **User Operator**: `DescribeAcls`, `CreateAcls`, `DeleteAcls`, `DescribeUserScramCredentials`, `AlterUserScramCredentials`, `DescribeClientQuotas`, and `AlterClientQuotas`.
* **Cruise Control**: `DescribeCluster`, `ListTopics`, `DescribeTopics`, `DescribeConfigs`, `DescribeLogDirs`, `DescribeReplicaLogDirs`, `AlterPartitionReassignments`, `ListPartitionReassignments`, `ElectLeaders`, `AlterReplicaLogDirs`, `IncrementalAlterConfigs` (replication throttles on brokers and topics), `CreateTopics` and `CreatePartitions` (for its own topics), and producing to and consuming from its own topics.
  The Cruise Control consumers use manual partition assignment without a consumer group, so no group rules are needed.
* **Kafka Exporter**: `Metadata`, `ListOffsets`, `ListGroups`, `DescribeGroups`, and `OffsetFetch`.
* **Strimzi Sova**: `Metadata`, `ListOffsets`, `ListGroups`, `DescribeGroups`, and `OffsetFetch`.

The names of the Cruise Control topics can be customized by the user in the `Kafka` custom resource.
So they are configurable in the Meta-Authorizer as well.

The Kafka nodes are allowed everything.
They use the internal listeners for replication, KRaft communication, request forwarding, and for the various plugins running inside the Kafka nodes.
Enumerating all of this would be fragile, especially for plugins that might be added by users.
And the Kafka nodes have access to all the data stored in the Kafka cluster anyway.

When a new version of any of the components starts using a new Kafka API, the rules will need to be updated and a new Meta-Authorizer version will need to be released (see the _Implementation_ section).

### Configuration

The Meta-Authorizer is configured using options with the `strimzi.metaauthorizer.` prefix in the Kafka node configuration:

```properties
authorizer.class.name=io.strimzi.kafka.metaauthorizer.MetaAuthorizer

strimzi.metaauthorizer.authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
strimzi.metaauthorizer.listeners=CONTROLPLANE-9090,REPLICATION-9091

strimzi.metaauthorizer.kafka.user=User:CN=my-cluster-kafka,O=io.strimzi
strimzi.metaauthorizer.clusteroperator.user=User:CN=cluster-operator,O=io.strimzi
strimzi.metaauthorizer.topicoperator.user=User:CN=my-cluster-entity-topic-operator,O=io.strimzi
strimzi.metaauthorizer.useroperator.user=User:CN=my-cluster-entity-user-operator,O=io.strimzi
strimzi.metaauthorizer.kafkaexporter.user=User:CN=my-cluster-kafka-exporter,O=io.strimzi
strimzi.metaauthorizer.cruisecontrol.user=User:CN=my-cluster-cruise-control,O=io.strimzi
strimzi.metaauthorizer.cruisecontrol.topics=strimzi.cruisecontrol.metrics,strimzi.cruisecontrol.modeltrainingsamples,strimzi.cruisecontrol.partitionmetricsamples

# User-configured super-users only
super.users=User:CN=my-admin
```

| Option                                         | Description                                                                                                  |
|:-----------------------------------------------|:-------------------------------------------------------------------------------------------------------------|
| `strimzi.metaauthorizer.listeners`             | Comma-separated list of the internal listeners (required)                                                    |
| `strimzi.metaauthorizer.authorizer.class.name` | Class name of the user-configured authorizer                                                                 |
| `strimzi.metaauthorizer.kafka.user`            | Principal of the Kafka nodes                                                                                 |
| `strimzi.metaauthorizer.clusteroperator.user`  | Principal of the Cluster Operator                                                                            |
| `strimzi.metaauthorizer.topicoperator.user`    | Principal of the Topic Operator                                                                              |
| `strimzi.metaauthorizer.useroperator.user`     | Principal of the User Operator                                                                               |
| `strimzi.metaauthorizer.kafkaexporter.user`    | Principal of Kafka Exporter                                                                                  |
| `strimzi.metaauthorizer.cruisecontrol.user`    | Principal of Cruise Control                                                                                  |
| `strimzi.metaauthorizer.cruisecontrol.topics`  | Comma-separated list of the Cruise Control topics (defaults to the default Strimzi Cruise Control topics)    |

The principals use the Kafka `<type>:<name>` format.
That way, they support both the mTLS-based principals and the Service Account-based principals.
For example, with the Service Account-based authentication, the Topic and User Operators share the Entity Operator Service Account.
When the same principal is configured for multiple components, it gets the union of their rules.

Only the components that are deployed will be configured.
Components without a configured principal do not get any rules.

The complete Kafka configuration is passed to the user-configured authorizer.
So its own configuration options (such as `super.users` or the options of a custom authorizer) keep working without any change.
Strimzi will configure only the user-configured super-users from the `Kafka` custom resource in the `super.users` option.

The Cluster Operator will generate the whole configuration.
The `strimzi.metaauthorizer.` prefix will be added to the forbidden options in the `.spec.kafka.config` section of the `Kafka` custom resource, so that users cannot override it.
Users will continue to configure the authorization in the `.spec.kafka.authorization` section of the `Kafka` custom resource as today.
No changes to the Strimzi API are needed.

### Integration with Apache Kafka

The Meta-Authorizer implements the Kafka `Authorizer` interface.
It also implements the `ClusterMetadataAuthorizer` interface and forwards its calls (such as `loadSnapshot`, `addAcl`, `removeAcl`, or `completeInitialLoad`) to the user-configured authorizer when it implements this interface as well.
This is required for the `StandardAuthorizer`, which receives the ACLs stored in the KRaft metadata log only through this interface.
It also implements the `Monitorable` interface and forwards the plugin metrics to the user-configured authorizer.
Initially, the Meta-Authorizer will have no metrics on its own.
But its own metrics can be added in the future.

The hardcoded rules do not depend on the user-configured authorizer.
So the internal components are authorized even before the `StandardAuthorizer` loads the ACLs from the KRaft metadata log.

The `ClusterMetadataAuthorizer` interface is an internal Kafka interface that is not covered by the Kafka compatibility guarantees.
The Meta-Authorizer therefore needs to be tested against all supported Kafka versions.
The prototype is built against Kafka 4.4 and also compiles against Kafka 4.3.
If the internal interface changes in a future Kafka version, we can release separate Meta-Authorizer artifacts for different Kafka versions and use them in the corresponding third-party libraries of the Kafka container images.

### Behavior Differences Compared to the Super-Users

Apart from the restricted rights of the internal components, there are some other differences in behavior compared to the super-users:

* The authorization decisions for the internal components are logged by the `io.strimzi.kafka.metaauthorizer.MetaAuthorizer` logger and not by the Kafka `kafka.authorizer.logger` logger.
  Denied requests are logged on the `INFO` level, allowed requests on the `DEBUG` level.
* The authorization metrics of the `StandardAuthorizer` (such as `authorization-allowed-rate-per-minute`) count only the decisions of the user-configured authorizer and not the decisions made by the Meta-Authorizer for the internal components.
* The internal components cannot get any additional rights through ACLs or through the rules of the user-configured authorizer on the internal listeners.
* The internal component principals are not super-users on the user listeners anymore.
  If they are used on a user listener, they are authorized by the user-configured authorizer as any other user.

### Known Limitation

The brokers forward many requests to the controllers without authorizing them first (for example `CreateTopics`, `DeleteTopics`, `CreatePartitions`, `CreateAcls`, `DeleteAcls`, `AlterClientQuotas`, `AlterUserScramCredentials`, or `IncrementalAlterConfigs` for topics).
The controller authorizes them with the original principal, but with the controller listener name.
It cannot tell on which listener the client originally connected.

So when a client authenticates on a user listener with a principal that is identical to the principal of one of the internal components, it will get the hardcoded rules of the component for the forwarded requests.
For example, a client authenticated as the Topic Operator on a user listener will be able to create and delete topics.
But it will not be able to read or write data, because the produce and fetch requests are authorized directly by the broker.

This is still an improvement compared to the current situation, where such a client would be a super-user on all listeners for all requests.
And it would get only the rules of the one component it impersonates, and only for the forwarded requests.
We also do not expect regular users to have the same usernames as the internal components.
Fixing this would require changes in Apache Kafka to pass the original listener name to the controller as part of the forwarded request.

### Feature Gate

The Meta-Authorizer will be introduced gradually using a new feature gate named `UseMetaAuthorizer`.
The following table shows the expected graduation of the `UseMetaAuthorizer` feature gate:

| Phase | Strimzi versions | Default state                                          |
|:------|:-----------------|:-------------------------------------------------------|
| Alpha | 1.5.0 - 1.6.0    | Disabled by default                                    |
| Beta  | 1.7.0 - 1.10.0   | Enabled by default                                     |
| GA    | 1.11.0 and newer | Enabled by default (without possibility to disable it) |

When the feature gate is disabled, the Cluster Operator will configure the authorization in the same way as today (with the user-configured authorizer and the internal components as super-users).
When the feature gate is enabled, the Cluster Operator will configure the Meta-Authorizer as described in this proposal.
The feature gate has effect only on Kafka clusters with enabled authorization.
Kafka clusters without authorization are not affected.

The Meta-Authorizer will be part of the Kafka container images regardless of the feature gate state.
So users can enable and disable the feature gate at any time.
Changing the feature gate changes the Kafka node configuration and causes a rolling update of the Kafka nodes.
During the rolling update, some nodes will use the super-users and other nodes will already use the Meta-Authorizer.
Both configurations authorize the internal components, so the rolling update does not cause any disruption.

The long Beta phase is intentional.
The Meta-Authorizer changes the security model of every Kafka cluster with authorization.
Many users run Strimzi with custom authorizers or with their own plugins and tools.
And when the hardcoded rules miss something, the internal components might stop working.
The long Beta phase gives users enough time to report any issues while they can still disable the feature gate.
It also allows us to validate more advanced aspects such as performance impact.

### Implementation

The Meta-Authorizer will be developed as a separate project in a new GitHub repository in the Strimzi organization (for example `strimzi/meta-authorizer`).
It will be released and distributed as a Maven artifact through Maven Central, in the same way as other Strimzi Kafka plugins such as the [Strimzi Quotas plugin](https://github.com/strimzi/kafka-quotas-plugin).

Having a separate project has several advantages:

* It makes the Meta-Authorizer easier to maintain.
  It has its own build, tests, benchmarks, and release cycle, independent of the operators.
* It keeps the `strimzi-kafka-operator` repository lean.
  The operators repository does not need to build, test, and benchmark another Kafka plugin as part of its build.
* It gives us flexibility in supporting different Kafka versions.
  If needed (for example, when the internal `ClusterMetadataAuthorizer` interface changes), we can release different Meta-Authorizer artifacts or versions for different Kafka versions.

The released Meta-Authorizer artifact will be added to the Kafka container images through the third-party libraries (`docker-images/artifacts/kafka-thirdparty-libs`) of all supported Kafka versions.
Each Kafka version can use the Meta-Authorizer version (or artifact) that is compatible with it.

The hardcoded rules are coupled to the Kafka APIs used by the Strimzi components.
When a component starts using a new Kafka API, the rules need to be updated in the Meta-Authorizer repository first.
A new Meta-Authorizer version then needs to be released and used in the Kafka container images before the change to the component can be released.
However, such changes should be relatively rare and known well ahead.
The Meta-Authorizer integration tests and the Strimzi system tests running with the feature gate enabled should catch any missing rules.
Many of the components (such as Cruise Control, Kafka Exporter, or Strimzi Sova) are already developed outside of the operators repository, so the rules cannot be kept together with all the components anyway.

### Performance Impact

The authorizer is called for every Kafka request.
So it is important that the Meta-Authorizer does not slow down the regular users whose requests are delegated to the user-configured authorizer.

To understand the performance impact, the Meta-Authorizer project contains JMH micro-benchmarks that compare two setups:

* **Baseline**: The `StandardAuthorizer` with the internal components configured as super-users (the current Strimzi setup)
* **Meta-Authorizer**: The Meta-Authorizer delegating to the `StandardAuthorizer`, with the internal components configured in the Meta-Authorizer and no super-users

Both setups load the same ACLs through the `ClusterMetadataAuthorizer` interface in the same way as the Kafka nodes do.
Next to the ACLs used by the test user, the benchmarks add between 0 and 50000 additional ACLs for 500 other users (literal and prefixed, for topics and groups, allow and deny).
Each `authorize` call contains 1, 10, or 50 actions (for example, a produce request writing to 10 different topics).
The benchmarks can be run with `mvn -Pperf -DskipTests verify` in the Meta-Authorizer project.

The results below were measured on an Apple M1 Max laptop with Java 21 and Kafka 4.4.0.
They should be used to compare the two setups and not as absolute numbers.
A negative difference means that the Meta-Authorizer is faster.

#### Regular Users

The authorization of the regular users is the main concern.
These requests are delegated to the user-configured authorizer.
On top of the delegate, the Meta-Authorizer does only a single lookup of the listener name in a `HashSet` (and a single `HashMap` lookup of the principal on the internal listeners).

| Scenario (1 thread, 3 JVM forks)                                    | Baseline | Meta-Authorizer | Difference |
|:--------------------------------------------------------------------|---------:|----------------:|-----------:|
| Allowed request, 1000 ACLs, 1 topic per request                     |  0.51 µs |         0.41 µs |       -19% |
| Allowed request, 1000 ACLs, 10 topics per request                   |  4.35 µs |         4.38 µs |        +1% |
| Allowed request, 1000 ACLs, 50 topics per request                   | 22.29 µs |        21.51 µs |        -4% |
| Allowed request, 10000 ACLs, 1 topic per request                    |  0.63 µs |         0.52 µs |       -17% |
| Allowed request, 10000 ACLs, 10 topics per request                  |  5.55 µs |         5.04 µs |        -9% |
| Allowed request, 10000 ACLs, 50 topics per request                  | 26.51 µs |        24.14 µs |        -9% |
| Idempotent producer initialization check, 1000 ACLs                 |    66 µs |           71 µs |        +7% |
| Idempotent producer initialization check, 10000 ACLs                |   820 µs |          811 µs |        -1% |

The Meta-Authorizer does not cause any measurable slowdown for the regular users.
The differences in both directions are caused by the noise and by the variance of the JIT compilation between the different JVM runs and not by the Meta-Authorizer itself.
The Meta-Authorizer also does not allocate any additional memory on this path.

The only scenario where the Meta-Authorizer was slower in repeated runs is the idempotent producer initialization check with 1000 ACLs (+7%).
This check uses the default Kafka implementation that iterates over all ACLs.
The difference disappears with 10000 and 50000 ACLs (-1% and +3%).
And this check is done only when an idempotent producer initializes and not for every request.

#### Internal Components

| Scenario (1 thread, 1000 ACLs, 10 actions per request)              | Baseline | Meta-Authorizer | Speed-up      |
|:--------------------------------------------------------------------|---------:|----------------:|--------------:|
| Kafka node on the replication listener                              |  1054 ns |           62 ns | 17x faster    |
| Topic Operator, allowed request                                     |  1097 ns |          173 ns | 6x faster     |
| Topic Operator, request outside its rules (denied)                  |  1099 ns |          181 ns | 6x faster     |
| Cruise Control, idempotent producer initialization check            |    98 ns |           31 ns | 3x faster     |

The internal components are authorized significantly faster.
They also allocate around 10 times less memory per request (for example, 80 bytes instead of 880 bytes for the Kafka nodes).
The `StandardAuthorizer` checks the super-users only after converting the principal to a string and it records its authorization metrics for every action.
The Meta-Authorizer does only two map lookups and a short scan of the rules of the component.
The results do not depend on the number of ACLs, because neither the super-users nor the hardcoded rules use the ACLs.

#### Concurrency

Kafka nodes authorize requests from many threads in parallel.
The following results were measured with 8 parallel threads, 10000 ACLs, and 10 actions per request:

| Scenario (8 threads, 2 JVM forks)                                   | Baseline | Meta-Authorizer | Difference         |
|:--------------------------------------------------------------------|---------:|----------------:|-------------------:|
| Regular user, allowed request                                       |  36.2 µs |         34.4 µs | -5%                |
| Regular user, denied request                                        |  46.2 µs |         45.0 µs | -3%                |
| Regular user, request forwarded to the controller                   |  32.8 µs |         37.4 µs | +14% (within noise)|
| Regular user, idempotent producer initialization check              |   868 µs |          920 µs | +6%                |
| Kafka node on the replication listener                              |  16.2 µs |        0.069 µs | 230x faster        |
| Topic Operator, allowed request                                     |  15.9 µs |        0.169 µs | 94x faster         |
| Cruise Control, idempotent producer initialization check            |  1.72 µs |        0.033 µs | 53x faster         |

With 8 threads, every `StandardAuthorizer` call is around 6-7 times slower than with a single thread in both setups.
The main reason is the contention on the synchronized authorization metrics of the `StandardAuthorizer`.
The regular users show no meaningful difference between both setups.
The forwarded request runs the same code as the regular allowed request with one additional `HashMap` lookup and its difference is within the measurement error of the baseline (±17%).
The internal components, which do not use the `StandardAuthorizer` anymore, avoid the contention completely.

#### Summary

* The Meta-Authorizer does not cause any measurable performance degradation for the regular users.
* The internal components - and especially the Kafka nodes replicating the data - are authorized significantly faster and with less memory allocation.
* The micro-benchmarks measure only the authorization itself, which is only a small part of the processing of a Kafka request.
  So the overall impact on the Kafka cluster performance is expected to be negligible in both directions.

### Testing Strategy

* The Meta-Authorizer project will contain unit tests covering the configuration, the hardcoded rules of each component (every combination of operation and resource type), and the delegation.
* It will also contain integration tests running an in-JVM KRaft cluster with separate brokers and controllers, the Meta-Authorizer delegating to the `StandardAuthorizer`, and no super-users.
  They verify the hardcoded rules on the internal listeners (including the requests forwarded to the controllers), the strict deny, the delegation of the forwarded requests from the regular users, and the loading of the ACLs from the KRaft metadata log.
  The integration tests will be run against all supported Kafka versions.
* The Cluster Operator unit tests will cover the generated configuration with the feature gate enabled and disabled and with both internal authentication types (mTLS and Service Account).
* The system tests will be run with the feature gate enabled in the same way as for the other feature gates.
  A new system test will verify that the internal components are restricted to their rules (for example, that the Kafka Exporter credentials cannot be used to produce messages or create topics).

### Documentation

The feature gate will be documented in the feature gates section of the documentation.
The documentation will also describe the rules for each of the internal components, the behavior differences compared to the super-users, and the known limitation.

## Affected Projects

This proposal affects the following projects:

* The new Meta-Authorizer project and its GitHub repository
* The `strimzi-kafka-operator` repository (the Cluster Operator and the Kafka container images)

No other Strimzi projects are affected.
But when other Strimzi components connecting to the internal listeners (such as Strimzi Sova) start using new Kafka APIs, the Meta-Authorizer rules will need to be updated.

## Backwards Compatibility

When the `UseMetaAuthorizer` feature gate is disabled, there are no changes to the current behavior.

When the feature gate is enabled, the behavior changes for the internal component principals:

* The internal components are restricted to their hardcoded rules.
  Users who use the credentials of the internal components for their own tools (for example, the Cluster Operator certificate for administrative scripts) might be denied some operations.
* The internal component principals are no longer super-users on the user listeners.
* ACLs or other rules configured for the internal components in the user-configured authorizer have no effect on the internal listeners.
* The authorization decisions for the internal components are logged by a different logger and are not included in the `StandardAuthorizer` metrics.
* The performance might differ.

However, the Strimzi `Kafka` custom resource API does not change.
And we do not expect these changes to have a major impact on Strimzi users.

## Rejected Alternatives

There are currently no rejected alternatives.