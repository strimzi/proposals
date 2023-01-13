# Analyse broker metrics before rolling Kafka

This proposal describes a solution for KafkaRoller's shortcoming with brokers in log recovery.

## Current situation

The current logic in KafkaRoller:
- Take a list of Kafka pods and reorder them based on the readiness.
- Roll unready pods first without considering why it could be unready.
- Before restarting a Kafka pod, consider if it is a controller or if it has an impact on the availability.
- If a Kafka pod does not become ready within the operational timeout (300000ms by default) after restarting, then it may force restart it without considering why it's taking a long time.

The current flow of KafkaRoller
![The current flow of KafkaRoller](images/046-kafka-roller-current-flow.png)

A Kafka broker can take a long time to become ready while performing log recovery. In this case, KafkaRoller could end up force restarting the broker and this can continue indefinitely, because of the log recovery on the broker startup again. KafkaRoller currently does not have a way to know what state a broker is in before making a decision to force restart it.

KafkaAgent currently collects the `kafka.server:type=KafkaServer,name=BrokerState` metric and creates a `kafka-ready` file on disk if the broker state value is greater than `3`(`RUNNING`) and not `127` (`UNKNOWN`). The Kafka readiness check passes if it finds the file on disk.

## Motivation

To address the issues with restarts and log recovery, KafkaRoller needs to be aware of the current broker state.  If a broker is performing log recovery, KafkaRoller should fail the reconciliation instead of force restarting it.

## Proposal
 
The objective of this proposal is to introduce a REST API endpoint within KafkaAgent, which exposes metrics from a Kafka broker. Furthermore, KafkaRoller would be modified so that it can query this endpoint to determine the current state of a broker, thus preventing unnecessary restarts. Additionally, KafkaRoller would allow a human operator to observe whether the log recovery process is making progress or is stuck.

The following implementation details will describe behaviour of the new API in KafkaAgent and how KafkaRoller would interact with it and handle a broker that is in log recovery.

### Implementation details

**KafkaAgent**:

- Add a web server to accept SSL connections on port 8443.
- Use Jetty for the web server since Kafka already uses it therefore avoids an additional dependency.
- Reuse the broker's own certificate that would be in the container for TLS authentication. 
- Configure the server to require TLS client authentication. 
- Expose an endpoint `v1/recovery-state` that includes the API version in the URI path.
- Extend KafkaAgent to collect `kafka.log:type=LogManager,name=remainingLogsToRecover` and `kafka.log:type=LogManager,name=remainingSegmentsToRecover` metrics.
- Use Jetty's native ServerHandler API to handle a `GET` request and return a JSON response that looks like this:
```
{
  "brokerState": 2,
  "remainingLogsToRecover": 123,
  "remainingSegmentsToRecover": 456
}
```
- Return HTTP 404 (Not Found) if a request is sent to an unknown version of the API e.g. `GET /v5/recovery-state` and `v5` does not exist.

**KafkaRoller**:

The proposed flow of KafkaRoller
![The proposed flow of KafkaRoller](images/046-kafka-roller-new-flow.png)

If a kafka pod does not become ready within the operational timeout (5 mins by default) take the following actions:
1. Check if the maximum number of retries has been reached. If it has, fail the reconciliation.
2. Ensure that the pod is not in a stuck state. The existing `isPodStuck()` can be used for this which returns true if the pod is in `CrashLoopBackOff`, `ImagePullBackOff`, `ContainerCreating` or `Pending` state. If the pod is stuck, continue with the current behaviour to retry rolling of the pod.
3. Use the Pod DNS name to query the KafkaAgent API endpoint.
4. If the API returns 503 (Service Unavailable), continue with the current behaviour to retry rolling of the pod.
5. Use the cluster operator's client certificate and keys to authenticate with the KafkaAgent API server.
6. Send `GET v1/recovery-state` request to the endpoint and check the value of `brokerState` from the response.
7. If the `brokerState` value is `2` (`RECOVERY`), follow the new logic for handling log recovery.
 
*Handling log recovery*

8. Set `inLogRecovery` to true. (`inLogRecovery` is a new boolean that would be added to the `RestartContext`).
9. Log the values of `remainingLogsToRecover` and `remainingSegmentsToRecover` from the request response in an `INFO` message.
10. Continue with the current behaviour to retry rolling of the pod.
11. On the next retry, check `inLogRecovery` inside the `restartIfNecessary()` function.
12. When `inLogRecovery` is set to true, call `awaitReadiness()` which periodically checks and waits for readiness until the operational timeout reaches.
13. If the operational timeout is reached, repeat from step 1.
14. If failing the reconciliation on step 1, include log recovery in the reason for failure.

### For future consideration:

- The operator might need access to more broker metrics. We can easily implement additional endpoints to expose them.
- The readiness check could also make use of the KafkaAgent endpoint to get the broker state instead of checking for a file on disk. An internal non-TLS endpoint would need to be implemented for it which can be elaborated on in a further proposal.
- The liveness and readiness checks potentially will need to access non-yammer metrics for KRaft, which could be exposed in the same manner, but would need a new mechanism to collect them.

## Affected projects

* strimzi/kafka-cluster-operator

## Compatibility

After the operator gets updated with the KafkaRoller change, it will expect the KafkaAgent endpoint to be present. However, the KafkaAgent may not have been updated yet.

To maintain backwards compatibility, use the new logic that handles log recovery only if KafkaRoller is able to successfully connect to the KafkaAgent endpoint. If the KafkaAgent endpoint is not responsive, continue with the current behaviour. This ensures compatibility with previous versions. Additionally, in the case that the KafkaAgent API is present, but the agent is stuck or not running due to some reason, then continuing with the current behaviour which may force restart the broker after a timeout could help recovering the agent.

## Rejected alternatives

- Why not use JMX to collect Kafka metrics? There is a user facing JMX settings that has its own authentication mechanism so we want to avoid any conflict with that.
- Why not use Kafka's metrics-reporter? Metric-reporter does not come up early enough in the Kafka process, which is needed for Kafka readiness/liveness probe. With java agent, we would have the endpoint up and available right away and start reporting on metrics immediately. Metrics-reporter also might not be set up until after the log recovery, so relevant metrics could be missing.
