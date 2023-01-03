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

To address the issues with restarts and log recovery, KafkaRoller needs to be aware of the current broker state.  If a broker is performing log recovery, KafkaRoller should wait longer instead of force restarting it.

## Proposal
 
The objective of this proposal is to introduce a REST API endpoint on the KafkaAgent which exposes the BrokerState metric that it already uses. And modify KafkaRoller so that it queries the new endpoint to find out what state a broker is in and avoids unnecessary restarts.
 
1. Add a RESTful endpoint to KafkaAgent that exposes an endpoint which returns a value of metric
    - Configure Jetty server to only accept SSL connections on port 8443.
    - It will reuse the broker's own certificate, which will already be in the container.
    - Expose an endpoint `/broker-state`.
    - Add a servlet to handle a `GET` request and return an integer value of the `BrokerState` metric.
2. Update KafkaRoller to query the broker state from the KafkaAgent API
    - Use the Pod DNS name to query the KafkaAgent API.
    - Instead of checking the broker's readiness through `isReady()` function, utilise the new RESTful endpoint to determine if the broker is ready.
    - After restarting a broker if it does not become ready within the operational timeout period (5 minutes by default), check whether the broker is performing log recovery by querying the new endpoint:
    - If log recovery is in progress:
        - Wait for log recovery to complete until the operation timeout reaches.
        - First time the operation timeout is exceeded, log a WARN message and fail the reconciliation.
        - If the log recovery completes in the next reconciliation, issue a warning message that includes the total observed time from pod deletion/recreation to `RUNNING` state. This would be helpful to figure out a more appropriate timeout value to avoid this in the future.
    - If not doing log recovery, force restart the broker which is the current behavior.

### For future consideration:

- The operator might need access to more broker metrics. We can easily implement additional endpoints to expose them.
- The readiness check could also make use of the KafkaAgent endpoint to get the broker state instead of checking for a file on disk. An internal non-TLS endpoint may need to be implemented for it which would be elaborated on in a further proposal.
- The liveness and readiness checks potentially will need to access non-yammer metrics for KRaft, which could be exposed in the same manner, but would need a new mechanism to collect them.

## Affected projects

* strimzi/kafka-cluster-operator

## Compatibility

When the operator gets updated with the KafkaRoller change, it will expect the KafkaAgent endpoint to be there. However, KafkaAgent would only get updated after the operator.

To maintain backwards compatibility, the following process is suggested:
1. Roll out the KafkaAgent changes for the new API.
2. Roll out the KafkaRoller changes and gated behind a feature flag. By default, KafkaRoller will still have the current behavior and will not use the KafkaAgent API.
3. Users can enable KafkaRoller to use KafkaAgent API using the feature flag. The user has to make sure that they are running the version that has the KafkaAgent API before they enable the feature.
4. We can mark the feature flag deprecated immediately.
5. In the next major release, we can remove the feature flag and KafkaRoller can always use the KafkaAgent API.

## Rejected alternatives

- Why not use JMX to collect Kafka metrics? There is a user facing JMX settings that has its own authentication mechanism so we want to avoid any conflict with that.
- Why not use Kafka's metrics-reporter? Metric-reporter does not come up early enough in the Kafka process, which is needed for Kafka readiness/liveness probe. With java agent, we would have the endpoint up and available right away and start reporting on metrics immediately. Metrics-reporter also might not be set up until after the log recovery, so relevant metrics could be missing.