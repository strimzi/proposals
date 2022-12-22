<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Analyse broker metrics before rolling kafka

This proposal describes a solution for KafkaRoller's shortcoming with brokers in log recovery. 

## Current situation

The current logic in Kafka Roller:
- Take a list of kafka pods and reorder them based on the readiness. 
- Roll unready pods first without considering why it could be unready.
- Before restarting a kafka pod, consider if it is a controller or it has an impact on the availability.
- If a kafka pod does not become ready within the operational timeout (300000ms by default) after restarting, then force restart it without considering why it's taking a long time.

A Kafka broker can take a long time to become ready while performing log recovery. In this case, KafkaRoller ends up force restarting the broker and this can continue indefinetly, because of the log recovery on the broker startup again. KafkaRoller currently does not have a way to know what state a broker is in before making a decision to force restart it.

KafkaAgent currently collects `kafka.server:type=KafkaServer,name=BrokerState` metric and creates a `kafka-ready` file on disk if the broker state value is `3`(`RUNNING`). The Kafka readiness check passes if it finds the file on disk.

## Motivation 

To address the issue described above, the KafkaRoller needs to be aware of the current broker state, so that if a broker is performing log recovery, then it waits longer instead of force restarting it.

## Proposal

The proposal is a REST API endpoint on the KafkaAgent which exposes the BrokerState metric that it already uses. And modify KafkaRoller so that it queries the new endpoint to find out what state a broker is in. 

1. Add a web server to KafkaAgent that exposes an endpoint which returns a value of metric
    - Configure Jetty server to only accept SSL connections on a port 8443.
    - Expose an endpoint `/broker-state`.
    - Add a servlet to handle a `GET` request and return an integer value of `BrokerState` metric.
    - It will reuse broker's own certificate, which will already be in the container.
2. Update KafkaRoller to query the broker state from the KafkaAgent API
    - Currently KafkaRoller obtains headless service for a given pod and uses it to boostrap Kafka AdminClient. It can use the same mechanism to get the address for the KafkaAgent endpoint.
    - Instead of checking broker's readiness through `isReady()` function, utilise the new restful endpoint to determine if the broker is ready.
    - After restarting a broker if it does not become ready within the operational timeout period, check whether the broker is performing log recovery by querying the new endpoint:
        - If log recovery is in progress, do not restart the broker.
        - Otherwise, force restart the broker which is the current behabviour.
    - Add a new configurable timeout for log recovery that is 60 minutes by default.
    - If the log recovery does not complete within the timeout period, then fail the reconciliation and report an error for the human operator to investigate further. 

### For future consideration:
- The operator might need access to more broker metrics. We can easily implement additional endpoints to expose them.
- The readiness check could also make use of the KafkaAgent endpoint to get the broker state instead of checking for a file on disk.
- The liveness and readiness checks potentially will need to access non yammer metrics for KRAFT, which could be exposed in the same manner, but would need a new mechanism to collect them.

## Affected/not affected projects

* strimzi/kafka-cluster-operator

## Compatibility

When the operator gets updated with the KafkaRoller change, it will expect the KafkaAgent endpoint to be there. However, KafkaAgent would only get updated after the operator.

1. Roll out the KafkaAgent changes for the new API.
2. Roll out the KafkaRoller changes and gated behind a feature flag. By default, KafkaRoller will still have the current behaviour and will not use the KafkaAgent API. 
3. User can enable KafkaRoller to use KafkaAgent API using the feature flag. The user has to make sure that they are running the version that has the KafkaAgent API before they enable the feature. 
4. We can mark the feature flag deprecetad immediately. 
5. In the next major release, we can remove the feature flag and KafkaRoller can always use the KafkaAgent API.

## Rejected alternatives
- Why not use VertX? Strimzi is moving away from using VertX. Kafka already uses Jetty which is convenient for KafkaAgent, as there is no need to introduce a new dependency.
- Why not use JMX to collect Kafka metrics? There is user facing JMX settings that has its own authentication mechanism so we want to avoid any conflict with that. 
- Why not use Kafka's metrics-reporter? Metric-reporter does not come up early enough in the kafka process, which is needed for kafka readiness/liveness probe. With java agent, we would have the endpoint up and available right away and start reporting on metrics immediately. Metrics-reporter also might not be set up until after the log recovery, so relevant metrics could be missing.