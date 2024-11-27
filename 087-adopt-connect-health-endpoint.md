# Adopt the Kafka Connect health check endpoint

In a Kubernetes node, the kubelet component uses the configured liveness probe to know when to restart a container (i.e. deadlock).
Additionally, it uses the configured readiness probe to know when a container is ready to start accepting traffic.

A Kafka Connect health check HTTP endpoint is available since Kafka 3.9.0 release ([KIP-1017](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1017%3A+Health+check+endpoint+for+Kafka+Connect)).
This proposal describes a possible approach for adopting the health check endpoint for Kafka Connect and Mirror Maker 2 components.

## Current situation

Kafka Connect provides a REST API for managing connectors.
Strimzi users can deploy Kafka Connect in distributed mode by simply creating a `KafkaConnect` resource.
A `KafkaMirrorMaker2` resource can be used to mirror data between Kafka clusters, and it is reconciled reusing the Kafka Connect logic.
Both of these components use the `/` HTTP (root) endpoint for liveness and readiness probes on port 8083 (rest-api).

This is the default HTTP probe configuration shared by both components:

| Property name       | Default value | Description                                                                                  |
|---------------------|---------------|----------------------------------------------------------------------------------------------|
| initialDelaySeconds | 60            | The initial delay before first the health is first checked.                                  |
| timeoutSeconds      | 5             | The timeout for each attempted health check.                                                 |
| periodSeconds       | 10            | How often to perform the probe.                                                              |
| successThreshold    | 1             | Minimum consecutive successes for the probe to be considered successful after having failed. |
| failureThreshold    | 3             | Minimum consecutive failures for the probe to be considered failed after having succeeded.   |

Strimzi does not support the HTTPs protocol for Kafka Connect REST API ([KIP-208](https://cwiki.apache.org/confluence/display/KAFKA/KIP-208%3A+Add+SSL+support+to+Kafka+Connect+REST+interface)).

Example output with `/` endpoint (probes only care about the HTTP response status code):

```sh
$ kubectl exec my-cluster-kafka-0 -- curl -s http://my-connect-cluster-connect-api:8083 | jq
{
  "version": "3.9.0",
  "commit": "a60e31147e6b01ee",
  "kafka_cluster_id": "_xIIeVIOQMKXimv_l96WtQ"
}
```

## Motivation

Using the `/` endpoint for Kafka Connect and Mirror Maker 2 health checks is a common approach, but it does not actually test for readiness, because these requests can be completed before the worker is started.
Instead, the `/health` endpoint waits for the worker startup phase to complete, which is made of internal topics creation if they do not exists, internal topics full read, and cluster join.

If the worker has not yet completed the startup phase, or it is unable to respond in time, the response will have a 5xx status code.
All Kafka Connect endpoints have a timeout of 90 seconds, with the exception of the `/health` endpoint where it is hardcoded to 10 seconds for faster unhealty worker detection.
Unlike the `/` endpoint, the `/health` endpoint response message includes error details that can help with troubleshooting.

Example output with `/health` endpoint (probes only care about the HTTP response status code):

```sh
$ kubectl exec my-cluster-kafka-0 -- curl -s http://my-connect-cluster-connect-api:8083/health | jq
{
  "status": "healthy",
  "message": "Worker has completed startup and is ready to handle requests."
}
```

## Proposal

Strimzi supports both Kafka 3.8 and 3.9 branches.

The `/health` endpoint will be used for both readiness and liveness probes of Kafka Connect and Mirror Maker 2 components only when `.spec.version` is either null or doesn't start with "3.8".

When Kafka 3.8 branch will be out of support, we will be able to get rid of the 3.8 fallback logic, and always use the `/health` endpoint.
Unit tests related to this feature will fail when Kafka 3.8 support will be removed from the code base.

Default liveness and readiness probes configuration won't change.

## Affected/not affected projects

The only affected project is the Cluster Operator, in particular Kafka Connect and Mirror Maker 2 components.

## Compatibility

This change is backwards compatible, and there should be no need to update Kafka Connect and Mirror Maker 2 probe configurations.

The following test results show that there isn't a significant difference in performance between the `/health` and `/` endpoints.
Note: pod ready time does not include the image pull time, and response time is computed as 95p over 200 requests with 10 seconds period.

| Endpoint | Pod ready time in seconds | Response time in ms |
|----------|---------------------------|---------------------|
| /        | 65                        | 3.0286              |
| /health  | 62                        | 3.7525              |

## Rejected alternatives

Wait until Kafka 3.8 branch is out of support in Strimzi and then switch to `/health`.
This approach is fine, but we want to make new Kafka features available in Strimzi as early as possible.
