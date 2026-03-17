# Dedicated executor service for HTTP Bridge async Kafka-related operations

This proposal aims to introduce a dedicated executor service for asynchronous Kafka-related operations in the HTTP Bridge, replacing the current usage of the default JVM's shared `ForkJoinPool.commonPool()`.
This change prevents thread explosion when the bridge is deployed with CPU limits less than or equals to 2 cores.

## Current situation

The HTTP Bridge currently uses `CompletableFuture` async methods (i.e. `runAsync()`, `supplyAsync()`, `whenCompleteAsync()`) without an explicit `Executor` parameter.
According to the [Java SE 17 CompletableFuture documentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/CompletableFuture.html):

> "All async methods without an explicit Executor argument are performed using the ForkJoinPool.commonPool() (unless it does not support a parallelism level of at least two, in which case, a new Thread is created to run each task)."

The `ForkJoinPool.commonPool()` parallelism is calculated as:

```
parallelism = Runtime.getRuntime().availableProcessors() - 1
```

This creates a critical problem in containerized environments, for example:

| CPU Limit | availableProcessors | ForkJoinPool Parallelism | Behavior |
|-----------|---------------------|-------------------------|----------|
| 1 core | 1 | 0 | New thread per task |
| 2 cores | 2 | 1 | New thread per task |
| 3+ cores | 3+ | 2+ | Uses thread pool |

The impact of current behavior is the following:

- Thousands of threads created under load with low CPU limits.
- Memory overhead from thread stacks (typically around 1MB per thread).
- Performance degradation from excessive context switching.
- Resource exhaustion risk in high-throughput scenarios.

This particularly affects Kubernetes deployments with CPU limits less than or equals to 2 cores or even bare metal/virtual machines deployments with same limits.

### Current workaround

The only available workaround is setting the JVM system property `java.util.concurrent.ForkJoinPool.common.parallelism` globally.
When using the `KafkaBridge` custom resource to deploy the HTTP bridge in Kubernetes via the Strimzi Cluster Operator, it means adding such system property in the `jvmOptions` section.


```yaml
...
kind: KafkaBridge
...
spec:
  jvmOptions:
    javaSystemProperties:
      - name: java.util.concurrent.ForkJoinPool.common.parallelism
        value: "4"
...
```

## Motivation

Leveraging the `ForkJoinPool.commonPool()` works fine with more than 2 CPUs (parallelism greater than 2) but it shows a bad behavior when deployed with CPU limits less than or equal to 2 cores.
In these constrained environments, the pool creates a new thread per task instead of reusing a bounded pool, leading to thread explosion, excessive memory consumption, and performance degradation.

This CPU-dependent behavior creates several problems:

- No control over parallelism: Users cannot tune the thread pool size for bridge async Kafka-related operations based on their specific workload characteristics. The parallelism is determined solely by the CPU limit, which may not align with the actual I/O concurrency needs of Kafka operations.
- Different behavior across environments: The bridge behaves differently between development environments (typically unlimited CPUs, using thread pool) and production deployments with CPU limits (creating unbounded threads), making it difficult to predict and test resource usage.
- Resource exhaustion risk: In high-throughput scenarios with low CPU limits, unbounded thread creation leads to memory exhaustion and potential pod eviction.

A dedicated executor service solves these issues by:

- Providing consistent, bounded behavior regardless of CPU limits.
- Enabling users to tune the parallelism based on their specific workload (message throughput, I/O patterns, latency requirements).
- Improving observability by using named threads and exposing metrics.
- Preventing resource exhaustion through bounded queues and pool.

## Proposal

The proposal is about implementing a dedicated `ThreadPoolExecutor` instance with the following characteristics:

- Bounded thread pool: Fixed pool with configurable size.
- Bounded queue: Used when the thread pool is full to avoid rejecting the tasks and preventing OOM under high load. It's configurable as well.
- Named threads: Threads named `kafka-bridge-async-N` for easy identification during debug.
- Rejection policy: Use `AbortPolicy` to raise and exception when the pool and queue are full, keeping the Vert.x event loop responsive and replying to the HTTP client with a proper error code for retry logic.

### Additional configuration parameters

Two new configuration parameters will be introduced.

The `bridge.executor.pool.size` to define the size of the thread pool used by the executor service for async Kafka-related operations.
If not set, it will default to `max(4, Runtime.getRuntime().availableProcessors() * 2)` to take into account:

- Kafka operations are I/O-bound (blocking on network). While threads wait for I/O, CPUs can serve other threads.
- Ensures sufficient parallelism even with 1-2 CPUs, preventing the original thread explosion problem by using a minimum of 4 threads.

The `bridge.executor.queue.size` to define the size of the queue for tasks waiting for an executor thread.
When both the pool and queue are full, new requests are rejected with a proper error to the HTTP client for retry logic.
If not set, it will default to `1000`.

### HTTP Bridge changes

A new class `HttpBridgeExecutor` is introduced to create a custom executor service.
It creates the `ThreadPoolExecutor` as described above and implements custom `ThreadFactory` for named threads.

The new parameters will be part of the `BridgeConfig` class in order to be set within the `application.properties` file.

The `HttpBridge` will instantiate the custom executor service to be passed to all sink and source endpoint instances.

Finally, all `CompletableFuture`-related "async" calls will be changed in order to accept the custom executor service as an additional parameter.

### Strimzi Cluster Operator support

The `KafkaBridge` custom resource needs to be extended to expose the new configuration parameters:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  # ... existing fields ...

  # New optional executor configuration fields
  executorPoolSize: 20
  executorQueueSize: 1000
```

Both `executorPoolSize` and `executorQueueSize` are optional fields.
If set, the Strimzi Cluster Operator will reflect their values within the corresponding properties into the `application.properties` file used for the HTTP bridge.
If not set, the bridge will use the default values as already described.

### Metrics and Observability

The executor service will expose metrics through the existing HTTP bridge `/metrics` endpoint using Micrometer, which is already integrated for Vert.x framework metrics.

The implementation uses Micrometer's `ExecutorServiceMetrics.monitor()` utility to automatically track executor metrics with minimal boilerplate.
Other custom metrics can be added.

Some of the exposed metrics would provide:

- current number of threads in the pool
- tasks waiting in queue
- total completed tasks
- current queue depth
- available queue slots

These metrics enable users to:

- understand the load in terms of pool and queue size.
- identify if there is the need for tuning.
- track rejected operations.

## Affected/not affected projects

Both the HTTP bridge and the Strimzi Cluster Operator are affected.
The HTTP bridge needs the implementation and usage of the custom executor service together with the possibility to configure pool and queue sizes for it.
The Strimzi Cluster Operator changes are related to leveraging the new configuration parameters from within the `KafkaBridge` custom resource.

## Compatibility

This proposal doesn't introduce any breaking changes in the HTTP bridge and the Strimzi Cluster Operator.
Both configuration parameters are optional with well defined defaults if not explicitly set.

## Rejected alternatives

### Document the JVM property workaround

As already mentioned, it's possible to workaround the issue by setting the JVM system property `java.util.concurrent.ForkJoinPool.common.parallelism` when the CPU limits are not enough.
It was rejected because would require more Java knowledge to understand without really fixing the root problem which is caused by the normal behavior of the `ForkJoinPool`.
Also, in general, using the default `ForkJoinPool` seems not to be the better approach so we made a mistake to use it since the beginning instead of a custom executor service.

### Use Vert.x worker threads

The async calls could be done by using the Vert.x worker threads for blocking tasks but, of course, it would be a step back on our direction towards using Vert.x as less as possible.
We already moved away from Vert.x worker threads to use `CompletableFuture`(s) for this reason.

### Use Java virtual threads

Using virtual threads could be an improvement for the future which can't be done right now because the HTTP bridge stays on Java 17, while virtual threads require Java 21+.
Moving to use Java 21 within the HTTP bridge is not possible right now because we decided to provide still support for Java 17 in the next future.