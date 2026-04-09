# Enable SSL for Metrics Reporter

This proposal is to enable SSL for the Metrics Reporter in Strimzi.
This would allow for exporting metrics over HTTPS.

## Current situation

The current metrics reporter in Strimzi does not support SSL.
This means that metrics are exported over HTTP.
This can be a security concern in some environments, as it allows for potential interception of sensitive data.

## Motivation

Enabling SSL for the Metrics Reporter would provide an additional layer of security for users who need to export metrics over HTTPS.
This would allow for secure communication between the Metrics Reporter and any monitoring tools that are consuming the metrics, such as Prometheus or Grafana.
Additionally, it would align with best practices for securing data in transit, especially in environments where sensitive information may be included in the metrics.

## Proposal

This proposal adds support for SSL in the Metrics Reporter by allowing users to configure SSL settings.
When SSL is enabled, the Metrics Reporter will serve metrics over HTTPS instead of HTTP.
The user chooses either HTTP or HTTPS by configuring the `prometheus.metrics.reporter.listener` property.
Both cannot be enabled at the same time.

### Configurations

The Metrics Reporter will support two methods for providing SSL certificates and keys in PEM format:
1. Path to PEM files
2. Inlined PEM content

PEM is the standard format in cloud-native environments where Strimzi and Prometheus operate.
> Caveats: JKS/PKCS12 keystores are not supported as the Strimzi operator does not use them for anything new.

The Metrics Reporter will use its own configuration namespace prefixed with `prometheus.metrics.reporter.listener.ssl.` to avoid conflicts with Kafka's own SSL settings.
The Metrics Reporter serves a different purpose than the Kafka broker or client SSL configuration and having a separate namespace avoids any confusion.

#### Listener configuration

- `prometheus.metrics.reporter.listener`: The listener address for the Metrics Reporter.
If set to an address starting with `https://`, the Metrics Reporter will use an HTTPS server.
If set to an address starting with `http://`, the Metrics Reporter will use a plain HTTP server.
Only one protocol can be active at a time.

#### PEM file path configuration

The certificate and key can be provided as paths to PEM files on disk:

- `prometheus.metrics.reporter.listener.ssl.certificate.location`: The path to the PEM file containing the server certificate or certificate chain.
- `prometheus.metrics.reporter.listener.ssl.key.location`: The path to the PEM file containing the private key.

#### Inlined PEM configuration

Alternatively, the certificate and key can be provided inline as PEM-encoded strings:

- `prometheus.metrics.reporter.listener.ssl.certificate.chain`: The server certificate or certificate chain in PEM format, provided as an inline string.
- `prometheus.metrics.reporter.listener.ssl.key`: The private key in PEM format, provided as an inline string.

If both file path and inline configurations are provided, the inline configuration takes precedence.

#### Additional SSL configuration

- `prometheus.metrics.reporter.listener.ssl.enabled.protocols`: Comma-separated list of enabled secure transport protocols (e.g. `TLSv1.2,TLSv1.3`).
- `prometheus.metrics.reporter.listener.ssl.enabled.cipher.suites`: Comma-separated list of cipher suites that the server will support.

### Implementation

The implementation will add support for the new SSL configuration options in the Metrics Reporter.

The metrics reporter currently uses the Prometheus `HTTPServer` from the `io.prometheus.metrics.exporter.httpserver` library.
When the listener is configured with `https://`, the `HTTPServer.Builder` will be configured with an `HttpsConfigurator` to serve metrics over HTTPS.
This will involve:
1. Loading the certificate and private key from PEM files or inline PEM content.
2. Initializing an `SSLContext` with the loaded key material.
3. Configuring the HTTPS server to use the SSL context for secure communication.

If SSL is misconfigured (e.g. missing certificate, unreadable PEM file, mismatched key), the Metrics Reporter will fail to start with a clear and descriptive error message.
This ensures that misconfigurations are caught early rather than silently falling back to an insecure state.

When SSL is not enabled, the Metrics Reporter will continue to function as an HTTP server with no changes to existing behavior.

## Affected/not affected projects

This change will primarily affect the metrics reporter project.
This proposal does not cover configuring SSL from the Strimzi Operators when the Metrics Reporter is used in a Strimzi deployment.
Such integration would require a separate proposal to handle how the operator provisions and mounts certificates for the Metrics Reporter.

## Compatibility

This proposal is designed to be backward compatible.
It will not change the existing behavior of the Metrics Reporter when SSL is not enabled.
Users who do not wish to use SSL can continue to use the Metrics Reporter as they currently do.
Users who want to enable SSL can do so by providing the necessary configuration options.
Users already using the Metrics Reporter will not be affected by this change, as it will only impact those who choose to enable SSL.

## Rejected alternatives

Using firewalls or API gateways to terminate TLS in front of the Metrics Reporter was considered.
However, in typical monitoring setups, metrics are collected locally (e.g. Prometheus scraping a co-located exporter).
Adding a TLS-terminating proxy in front of a local endpoint would add unnecessary complexity and operational overhead.
Enabling SSL directly in the Metrics Reporter provides a more straightforward and integrated solution for environments that require encryption of metrics data in transit.
