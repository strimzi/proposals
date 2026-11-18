# Support compressed HTTP requests in the Strimzi Kafka Bridge

This proposal adds HTTP request decompression to the Strimzi Kafka Bridge, so clients can send less data to the Bridge.

## Current situation

The HTTP Bridge does not support compressed request bodies.
Clients send every request uncompressed, including the JSON envelope and the record data.

The Kafka producer compression configured in the Bridge only reduces traffic between the Bridge and the Kafka brokers.
Users who want to compress HTTP requests have to run a reverse proxy in front of the Bridge to decompress them.

## Motivation

Cloud providers charge for traffic between regions, so clients that send requests to a Bridge in another region incur data transfer costs.
JSON record payloads are often repetitive and compress well, and the Bridge can decompress them without an extra proxy.

This proposal addresses [strimzi/strimzi-kafka-bridge#1198](https://github.com/strimzi/strimzi-kafka-bridge/issues/1198).

## Proposal

The Bridge will accept compressed bodies on all client API endpoints that accept a request body.
Clients enable compression per request with the `Content-Encoding` header, and users do not need to change the Bridge configuration.
The Bridge decompresses the body before JSON parsing, OpenAPI validation, and endpoint processing.

### Supported compression algorithms

The Bridge will support gzip and Zstandard.
Both are registered HTTP content codings.
The Bridge will use the decompression support available through Vert.x and validate the `Content-Encoding` header to enforce the supported encodings.

**Note:** For `zstd` decompression decompression needs no additional library, because `zstd-jni` is already a transitive dependency of `kafka-clients`.

A future proposal can add Snappy or LZ4 if users request them.

| Algorithm | `Content-Encoding` |
|-----------|--------------------|
| gzip | `gzip` |
| Zstandard | `zstd` |

### Request handling

Clients keep the existing `Content-Type` and compress the whole request body.
For example:

```http
POST /topics/my-topic HTTP/1.1
Content-Type: application/vnd.kafka.json.v2+json
Content-Encoding: gzip

<gzip-compressed JSON body>
```

Requests without `Content-Encoding`, or with `Content-Encoding: identity`, go through the existing processing path.
After decompression, the Bridge sends the same records to Kafka as it would for the equivalent uncompressed request.
The Kafka producer still applies its configured `compression.type`, independent of the HTTP request compression.

#### Multiple content encodings

HTTP allows multiple content encodings to be applied sequentially, for example `Content-Encoding: gzip, zstd`.
The Bridge will support only a single encoding per request, which is enough for the data transfer reduction use case.

Supporting multiple encodings would require custom logic in the Bridge to apply the decoders in reverse order, as Vert.x does not handle this automatically.
Requests specifying multiple encodings will therefore be rejected, even when each encoding is individually supported.

### Error handling

The Bridge returns errors in the existing error response format:

| Condition | HTTP status |
|-----------|-------------|
| Unsupported content encoding | `415 Unsupported Media Type` |
| Multiple content encodings not supported | `415 Unsupported Media Type` |
| Invalid compressed body | `400 Bad Request` |

An empty body with `Content-Encoding: gzip` gets the same response as an empty uncompressed request.

### Implementation

Request decompression will be enabled in [http server configuration](https://github.com/strimzi/strimzi-kafka-bridge/blob/main/src/main/java/io/strimzi/kafka/bridge/http/HttpBridge.java#L357):

```java
httpServerConfig.setCompressionConfig(
    new CompressionConfig()
        .setDecompressionEnabled(true)
        .addGzip()
        .addZstd());
```

The Bridge will validate the `Content-Encoding` header in a router root handler, before the request body is validated, and map any decompression failures to the errors above.

### Testing

Integration tests will cover the following scenarios:

- gzip and zstd requests produce the same Kafka records as uncompressed requests
- Requests with no `Content-Encoding` or with `identity` go through the existing processing path
- Unsupported encodings are rejected with `415 Unsupported Media Type`
- Stacked encodings are rejected with `415 Unsupported Media Type`
- Corrupt compressed bodies are rejected with `400 Bad Request`
- Empty compressed bodies get the same response as empty uncompressed requests

### Documentation

The documentation will be updated with the supported encodings, the new error responses, and an example of a compressed request.

### Metrics

The existing `strimzi_bridge_http_server_request_bytes` metric remains unchanged and measures the body size after decompression.
A new `strimzi_bridge_http_server_request_compressed_bytes` metric will measure the compressed body size of compressed requests, with the same labels plus an `encoding` label.
This allows users to compare compressed and decompressed body sizes and see how much data transfer compression saves.

## Affected/not affected projects

This proposal affects only the [Strimzi Kafka Bridge](https://github.com/strimzi/strimzi-kafka-bridge).

## Compatibility

Existing clients that send uncompressed requests keep working without changes.

## Rejected alternatives

### Use a reverse proxy

Users who already run a reverse proxy in front of the Bridge can decompress requests there, for example by creating an `EnvoyFilter` if using Istio.
Users without a reverse proxy would have to deploy and maintain one only to decompress requests for the Bridge.

### Accept compressed Kafka record batches

Clients could send compressed batches in [Kafka's record-batch format](https://kafka.apache.org/41/implementation/message-format/#record-batch), allowing the Bridge to forward them without decompression and recompression.

This approach is rejected because it requires a new client request format and a separate sending implementation, as the Java Kafka client has no public API for sending prebuilt batches.
The proposed solution works with the existing request format and Java Kafka client. The trade-off is that the Bridge decompresses requests and recompresses them when Kafka producer compression is enabled.
