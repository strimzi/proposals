# Reduce Strimzi test-client's images

This proposal suggests reducing Strimzi test-client's images.

## Current situation

Test-clients repository contains 2 HTTP - producer & consumer - and 4 Kafka - producer, consumer, admin & streams - clients.
For each client we are building separate image.
Kafka clients are then build for each supported Kafka version, which are specified in [`kafka.version`](https://github.com/strimzi/test-clients/blob/main/docker-images/kafka.version) file.
That means we can end up with 10 images for a release, when we are supporting 2 Kafka versions.
Image base is same for all the clients, only difference is used `jar`.

## Motivation

During releases of test-clients, we found few difficulties with current implementation.
The dependencies are mostly the same for all the clients.
After we added support for new architectures (`ppc64le`, `s390x`, `arm64`), the whole build process became chaotic.
At the same time, in our `systemtest` module, we need to specify image for each client.

## Proposal

Client's code bases will remain the same as they are.
The image build process will be different.
Both HTTP and Kafka clients will be inside one image.

### Image content

* `/opt/test-clients` - base folder for test-clients
* `/opt/test-clients/bin` - scripts for running each client
  * f.e: `http_producer_run.sh`
  * scripts will contain correct classpath, Java options and path to main class 
* `/opt/test-clients/lib` - all needed dependencies for running the clients

The approach will be similar to the `operator`'s image inside Strimzi.

### Image name

Current pattern:

* `quay.io/strimzi-test-clients/test-client-kafka-{client-type}:{version}-kafka-{kafka-version}`
* `quay.io/strimzi-test-clients/test-client-http-{client-type}:{version}`

will be changed to `quay.io/strimzi-test-clients/test-clients:{version}-kafka-{kafka-version}`.

### `Systemtest` implementation and changes 

* Environment variables will be reduced
  * `TEST_PRODUCER_IMAGE`, `TEST_CONSUMER_IMAGE` and other environment variables will be removed
  * `TEST_CLIENT_IMAGE` will be added
* Container configuration of each client will contain `args` section with appropriate `*_run.sh`
    ```yaml
     args:
      - /opt/test-clients/bin/http_producer_run.sh
    ```

## Advantages

* One image instead of six (for one Kafka version)
* Easier and clearer building system
* Simple usage in STs

## Affected/not affected projects

[Test-clients repository](https://github.com/strimzi/test-clients/) and `systemtest` module in operators repository.

## Rejected alternatives

There are no rejected alternatives at the moment.