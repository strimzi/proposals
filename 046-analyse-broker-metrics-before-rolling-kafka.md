# Analyse broker metrics before rolling Kafka

This proposal describes a solution for KafkaRoller's shortcoming with brokers in log recovery.

## Current situation

The current logic in KafkaRoller:
- Take a list of Kafka pods and reorder them based on the readiness.
- Roll unready pods first without considering why it could be unready.
- Before restarting a Kafka pod, consider if it is a controller or if it has an impact on the availability.
- If a Kafka pod does not become ready within the operational timeout (300000ms by default) after restarting, then force restart it without considering why it's taking a long time.

A Kafka broker can take a long time to become ready while performing log recovery. In this case, KafkaRoller ends up force restarting the broker and this can continue indefinitely, because of the log recovery on the broker startup again. KafkaRoller currently does not have a way to know what state a broker is in before making a decision to force restart it.
 
KafkaAgent currently collects the `kafka.server:type=KafkaServer,name=BrokerState` metric and creates a `kafka-ready` file on disk if the broker state value is `3`(`RUNNING`). The Kafka readiness check passes if it finds the file on disk.

## Motivation

To address the issues with restarts and log recovery, KafkaRoller needs to be aware of the current broker state.  If a broker is performing log recovery, KafkaRoller should fail the reconciliation instead of force restarting it.

## Proposal
 
The objective of this proposal is to introduce a REST API endpoint within KafkaAgent, which exposes metrics from a Kafka broker. Furthermore, KafkaRoller would be modified so that it can query this endpoint to determine the current state of a broker, thus preventing unnecessary restarts. Additionally, KafkaRoller would also display the remaining logs and segments to recover, allowing a human operator to observe the recovery process and determine if it is making progress or is stuck.

### Implementation details

*KafkaAgent*:
- Add a web server to accept SSL connections on port 8443.
- Use Jetty for the web server since Kafka already uses it therefore avoids an additional dependency.
- Reuse the broker's own certificate that would be in the container for TLS authentication. 
- Configure the server to require TLS client authentication. 
- Expose an endpoint `v1/brokermetrics` that includes the API version in the URI path.
- Extend KafkaAgent to collect `kafka.log:type=LogManager,name=remainingLogsToRecover` and `kafka.log:type=LogManager,name=remainingSegmentsToRecover` metrics.
- Take the metric name as a parameter in the request query e.g `v1/brokermetrics?name=BrokerState`. Use the value of the MBean's `name` field for the request  e.g. `BrokerState` and  `remainingLogsToRecover`.
- Use Jetty's native ServerHandler API to handle a `GET` request and return 
    - an integer value for `BrokerState`.
    - an integer value for  `remainingLogsToRecover`.
    - an integer value for `remainingSegmentsToRecover`.
- If no metric name was provided in the request or the requested metric name is not supported, return HTTP 404 (Not Found) to the client.

*KafkaRoller*:
- Use the Pod DNS name to query the KafkaAgent API.
- Use the cluster operator's client certificate and keys to authenticate with the KafkaAgent API server.
- Instead of checking the broker's readiness through `isReady()` function, utilise the new RESTful endpoint to determine if the broker is ready.
- Include an additional field in the status section of the Kafka CR. This will hold data that will be utilised by KafkaRoller in case the reconciliation process is repeated while the log recovery process is still ongoing:
```
status:
    <kafka_pod_name>:
      restartTimeStamp: 
      logRecoveryExceededTimeout: <true|false>
```
- When KafkaRoller restarts a broker, write the creation timestamp to the new field, `restartTimeStamp`.
- If a kafka pod does not become ready within the operational timeout, check the broker state by sending a request to the new endpoint:
    - If log recovery is in progress, apply the new logic for handling log recovery.
    - Otherwise, continue with the current behaviour.

*Logic for handling log recovery*
- Wait for log recovery to complete until the operation timeout (5 mins by default) reaches.
- While waiting for log recovery to complete, check its progress by periodically sending a request to the new endpoint for `remainingLogsToRecover` and `remainingSegmentsToRecover` metrics and log them in an `INFO` message.
- If the operation timeout is exceeded, check the value of `logRecoveryExceededTimeout` field in the CR:
    - If false, log a `WARN` message, fail the reconciliation and set `logRecoveryExceededTimeout` to true.
    - If true, fail the reconciliation.
- If kafka pod becomes ready, check the value of `logRecoveryExceededTimeout`. If set to true:
    - Calculate the time passed since the `restartTimeStamp` and log it in a WARN message. This would be helpful to figure out a more appropriate timeout value to avoid this in the future.
    - Empty `restartTimeStamp` and `logRecoveryExceededTimeout` values.

### For future consideration:

- The operator might need access to more broker metrics. We can easily implement additional endpoints to expose them.
- The readiness check could also make use of the KafkaAgent endpoint to get the broker state instead of checking for a file on disk. An internal non-TLS endpoint would need to be implemented for it which can be elaborated on in a further proposal.
- The liveness and readiness checks potentially will need to access non-yammer metrics for KRaft, which could be exposed in the same manner, but would need a new mechanism to collect them.

## Affected projects

* strimzi/kafka-cluster-operator

## Compatibility

After the operator gets updated with the KafkaRoller change, it will expect the KafkaAgent endpoint to be present. However, the KafkaAgent may not have been updated yet.

To maintain backwards compatibility, use the new logic that handles log recovery only if KafkaRoller is able to successfully connect to the KafkaAgent endpoint. If the KafkaAgent endpoint is not available yet or the agent is not running, the current logic of restarting the broker after a timeout period should be used. This ensures compatibility with previous versions. Additionally, in the case that the KafkaAgent API is available but the agent is stuck or not running for some reason, the current logic's force restart of the broker may help recover the agent.

## Rejected alternatives

- Why not use JMX to collect Kafka metrics? There is a user facing JMX settings that has its own authentication mechanism so we want to avoid any conflict with that.
- Why not use Kafka's metrics-reporter? Metric-reporter does not come up early enough in the Kafka process, which is needed for Kafka readiness/liveness probe. With java agent, we would have the endpoint up and available right away and start reporting on metrics immediately. Metrics-reporter also might not be set up until after the log recovery, so relevant metrics could be missing.
