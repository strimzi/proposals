# KRaft controller configuration via Kafka Agent

This proposal is about enabling the current Kafka Agent to get the configuration from KRaft `controller` nodes and make it available to the Kafka Roller in the Strimzi Cluster Operator.
This way the Kafka Roller can determine the need to roll the `controller` nodes on configuration changes.

## Current situation

The Kafka Agent is a Java agent, running alongside the Kafka process, which provides the following features:

* check that the broker is connected to ZooKeeper and it is also in a valid running state and make this information available to the Kubernetes platform via the liveness and readiness probes (it creates some specific files on the broker disk).
* expose an HTTP endpoint, securely reachable by the Kafka Roller, to provide information about the current broker state and ongoing log recovery operations (more details in the proposal [#48](https://github.com/strimzi/proposals/blob/main/048-avoid-broker-restarts-when-in-recovery.md)).

With the above features, the Kubernetes platform can deal with restarting the Kafka container if it's not ready or alive and the Kafka Roller can roll the Kafka brokers depending on their state.

Furthermore, the Kafka Roller is already able to connect to Kafka brokers in order to get their configuration and check if they need an update, dynamically or by rolling them, in order to get a new configuration.

As today, the Kafka Agent is not started when the cluster is deployed in KRaft mode.

## Motivation

The current Kafka Agent, together with the Kafka Roller, are good for dealing with updating and/or rolling Kafka brokers when working with ZooKeeper-based deployments.
With Kafka moving to KRaft and the community deprecating ZooKeeper, the Strimzi project has to provide the best experience to the users for a seamlessly migration.
Currently, the Strimzi Cluster Operator already supports the KRaft mode (behind a feature gate).
In KRaft mode, each node's `process.roles` must be one of `controller` or `broker` or `controller,broker`, but currently the Kafka Roller is only able to get the configuration of nodes having the `broker` role.
The current configuration is needed to determine whether any action is necessary and if so whether a dynamic configuration update is sufficient, or whether restarting the node is required.
The current Kafka implementation of the `controller` role has some limitations instead:

* it doesn't support the `METADATA` API, so it's not possible to specify a `controller` address for bootstrapping a Kafka Admin client (as it is used in the Kafka Roller for the `broker`(s))
* it doesn't support the `DESCRIBE_CONFIGS` API, so even if an Admin client was able to connect, it cannot retrieve the current `controller` configuration (as it is available in the Kafka Roller for the `broker`(s))
* furthermore beyond simply reading the Kafka source code there's no way of knowing (either via the Kafka Admin client, or otherwise) which configuration properties are pure `broker`, pure `controller` or applicable to both roles.

Due to the above limitations, the Kafka Roller is not able to connect to a `controller` node by using a Kafka Admin client instance, retrieving the current configuration and dealing with a dynamic update and/or rolling.

> Despite the missing support for the above APIs, the `controller` role supports `INCREMENTAL_ALTER_CONFIGS` API, so assuming the Kafka Roller was able to connect, getting configuration and dealing with changes, it would be able to update it.
> For this proposal we will not be supporting the dynamic reconfiguration of pure `controller` nodes.

The best solution would have above limitations being addressed in the Kafka upstream project.
The [KIP-919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum) aims to address the first one, enabling a Kafka Admin client to connect directly to a `controller` node.
Once connected, getting the configuration from the `controller` node is not currently part of the KIP so something to discuss in the upstream community (maybe raising it in the KIP-919 or opening a new one).
The problem related to the `controller` only configuration properties is not taken into account yet.

While the KRaft mode is already defined as "production ready" and the migration process from ZooKeeper-mode is getting improvements and stability, the Strimzi project has to keep the pace in order to provide the best experience to the users.
Having the best support and fix the issues in the Kafka upstream project could take long time, maybe not landing on time in the next Kafka 3.6.0 release.
For this reason, a temporary solution, built into the Strimzi project, will unlock the next chunk of work for supporting an automatic ZooKeeper-mode to KRaft migration within the Strimzi Cluster Operator.

## Proposal

### Running Kafka Agent when KRaft mode enabled

As today, the Kafka Agent doesn't run when KRaft mode is enabled (based on a condition in the `kafka_run.sh` bash script).
On one side, it makes sense because for Kubernetes liveness and readiness probes we are using a different approach for KRaft (more details on proposal [#46](https://github.com/strimzi/proposals/blob/main/046-kraft-liveness-readiness.md)) and not creating some files on the brokers.
On the other side, we are not providing the broker and log recovery status to the Kafka Roller when KRaft mode is enabled.

The Kafka Agent gets some parameters (separated by `:`).

```shell
KAFKA_OPTS="${KAFKA_OPTS} -javaagent:$(ls "$KAFKA_HOME"/libs/kafka-agent*.jar)=/var/opt/kafka/kafka-ready:/var/opt/kafka/zk-connected:$KEY_STORE:$CERTS_STORE_PASSWORD:$TRUST_STORE:$CERTS_STORE_PASSWORD"
```

The first two are the paths to the files to be created on the file system to set the Kafka broker as ready and connected to ZooKeeper (used by the Kubernetes liveness and readiness probes).
The rest are key and certificate stores (with related password), to allow the Kafka Roller to connect to the Kafka Agent via HTTPS.
The first two are not needed when in KRaft mode so they could be left just as empty.

```shell
KAFKA_OPTS="${KAFKA_OPTS} -javaagent:$(ls "$KAFKA_HOME"/libs/kafka-agent*.jar)=::$KEY_STORE:$CERTS_STORE_PASSWORD:$TRUST_STORE:$CERTS_STORE_PASSWORD"
```

This would be recognized by the Kafka Agent as a "running cluster in KRaft mode".
This way the Kafka Agent would not take care of handling these files for the Kubernetes liveness and readiness probes in ZooKeeper-mode.
We could think about having a dedicated additional parameter for that, but what we aim with this proposal is anyway a temporary solution and also we are moving away from ZooKeeper, so at some point the first two parameters would be totally removed.

### Exposing KRaft `controller` configuration

During the Kafka start up script, the node configuration is generated and saved on the disk in the `/tmp/strimzi.properties` file.
As soon as the Kafka Agent runs, it can read this file and load its content in memory in a JSON format.
The top level JSON value is an object with all string-typed values (i.e. even if they are numbers).
Following, an example of the JSON object bringing the `controller` configuration.

```json
{
    "broker.id": "0",
    "node.id": "0",
    "process.roles": "controller",
    "controller.listener.name": "CONTROL_PLANE-9090",
    ...
    ...
}
```

Such a configuration is returned through the `/v1/node-configuration` HTTP endpoint.
Because this file never changing once the broker/controller process has started, even if the node has its configuration dynamically updated, there is no need for the Kafka Agent to watch the file.
It is enough to load its content on the Kafka Agent startup and keep it as cached in memory.
The same content will be just returned every time the Kafka Roller is asking for it.

If the Kafka Roller is talking to a previous version of the Kafka Agent where the `/v1/node-configuration` HTTP endpoint doesn't exist, it will get a `404 (NOT FOUND)` error.
In this case, it will log an error message and fail the reconciliation.

### Kafka Roller usage

The usage of the `controller` node configuration in the Kafka Roller is not a goal of this proposal, but for better understanding it is useful to mention it.

When going through the available nodes, as a set of `NodeRef` records, the Kafka Roller is able to get the role of each of them by using the corresponding boolean `controller` and `broker` fields.
Detecting a `controller`, the Kafka Roller queries the Kafka Agent to get the `controller` node configuration in JSON format via HTTPS on the `/v1/node-configuration` endpoint.
The Kafka Roller also has an hardcoded list of the `controller` only configuration properties (which has to be defined taking a look at the Kafka codebase).
When there is a change in the `Kafka` custom resource, the Kafka Roller checks if some of the `controller` specific parameters are changed, between the desired configuration in the custom resource and the one got from the Kafka Agent.
If there is any difference, the Kafka Roller just rolls the `controller` node.

## Affected/not affected projects

The main affected project is the Kafka Agent one.
Also the bash script for running Kafka is affected in order to start the Kafka Agent when KRaft mode is enabled.
The Kafka Roller is not impacted because the part related to it using the information exposed by the Kafka Agent is not a goal of this proposal.

## Compatibility

This proposal is not going to break any backward compatibility.
The Kafka Agent will run in KRaft mode as well but not handling the files used by the Kubernetes platform for checking liveness and readiness of the Kafka broker.
It will still provide broker and log recovery status on the corresponding HTTP endpoint.
It is going to add one more HTTP endpoint for providing `controller` node configuration.
In case of Kafka Roller talking to an old Kafka Agent which doesn't provide the `controller` node configuration, it will log an error message and fail the reconciliation.

## Rejected alternatives

Waiting for the missing APIs related issues to be addressed in the Kafka upstream project first and then moving forward with the work on the automated migration within Strimzi.
This would push the migration support from ZooKeeper-mode to KRaft too far in the future.
