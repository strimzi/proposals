# Enable SSL for Metrics Reporter

This proposal is to enable SSL for the Metrics Reporter in Strimzi. This would allow for exporting metrics over HTTPs.

## Current situation

The current metrics reporter in Strimzi does not support SSL, which means that metrics are exported over HTTP. This can be a security concern in some environments, as it allows for potential interception of sensitive data.

## Motivation

Enabling SSL for the Metrics Reporter would provide an additional layer of security for users who need to export metrics over HTTPs. This would allow for secure communication between the Metrics Reporter and any monitoring tools that are consuming the metrics, such as Prometheus or Grafana. Additionally, it would align with best practices for securing data in transit, especially in environments where sensitive information may be included in the metrics.

## Proposal

I propose to add support for SSL in the Metrics Reporter by allowing users to configure SSL settings, such as providing a keystore and truststore. This would involve updating the Metrics Reporter to use an HTTPs server instead of an HTTP server, and ensuring that the necessary SSL configurations are properly handled.

This proposal is strictly related to [Add support for TLS/SSL on the HTTP interface](/122-enable-ssl-for-kafka-bridge.md) and some design decisions will be shared between the two proposals, such as the configuration options for SSL.

### Configurations

We will start by adding new configuration options for the Metrics Reporter, such as:
- `prometheus.metrics.reporter.listener`: if set to an address starting with `https://`, the Metrics Reporter will use an HTTPs server instead of an HTTP server.
- `prometheus.metrics.reporter.listener.ssl.keystore.certificate.location`: The location of the keystore certificate file.
- `prometheus.metrics.reporter.listener.ssl.keystore.key.location`: The location of the keystore key file.
- `prometheus.metrics.reporter.listener.ssl.keystore.certificate.chain`: The certificate chain in PEM format.
- `prometheus.metrics.reporter.listener.ssl.keystore.key`: The key in PEM format.
- `prometheus.metrics.reporter.listener.ssl.enabled.protocols`: Comma separated list of enabled secure transport protocols.
- `prometheus.metrics.reporter.listener.ssl.enabled.cipher.suites`: Comma separated list of cipher suites that the server will support.

When SSL is enabled, the Metrics Reporter will use the provided keystore and truststore to establish secure connections with clients. The implementation will ensure that the necessary SSL configurations are properly handled, and that the Metrics Reporter can still function as expected when SSL is enabled.

Kafka already has support for SSL, so one would suggest reusing the existing SSL config options for the Metrics Reporter. However, I trust that the metrics reporter should have its own set of SSL config options. This is because it can have different requirements and configurations than the Kafka components, and it allows for more flexibility in how SSL is configured for the Metrics Reporter specifically. Additionally, it would avoid any potential conflicts or confusion that could arise from sharing SSL config options between the Metrics Reporter and Kafka components.

### Implementation

We will start the implementation by adding the support of the new configuration options for SSL in the metric reporter.

Prometheus HttpServer will be modified with a HttpsConfigurator that will be responsible for configuring the SSL settings based on the provided configuration options. This will involve setting up the SSL context, loading the keystore and truststore, and configuring the HTTPs server to use the SSL settings.

The implementation will also include error handling to ensure that any issues with SSL configuration are properly logged and do not cause the Metrics Reporter to fail unexpectedly. Additionally, we will ensure that the Metrics Reporter can still function as expected when SSL is enabled, and that it can handle both HTTP and HTTPs requests based on the configuration provided by the user.

## Affected/not affected projects

- metrics-reporter

## Compatibility

This proposal is designed to be backward compatible, as it will not change the existing behavior of the Metrics Reporter when SSL is not enabled. Users who do not wish to use SSL can continue to use the Metrics Reporter as they currently do, while those who want to enable SSL can do so by providing the necessary configuration options. Users already using the Metrics Reporter will not be affected by this change, as it will only impact those who choose to enable SSL.

## Rejected alternatives

Not rejected, but considered alternatives include making use of Firewalls and API Gateways to secure the communication between the Metrics Reporter and monitoring tools. However, these alternatives would require additional and complex configuration, and may not be feasible for all users. Enabling SSL directly in the Metrics Reporter provides a more straightforward and integrated solution for securing metrics export and is more industrial standard for securing data in transit.