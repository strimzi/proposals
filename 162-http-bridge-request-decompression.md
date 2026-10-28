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
The Bridge will use the decompression support available through Vert.x and Netty and validate the `Content-Encoding` header to enforce the supported encodings.
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

### Error handling

The Bridge returns errors in the existing error response format:

| Condition | HTTP status |
|-----------|-------------|
| Unsupported encoding or multiple content encodings | `415 Unsupported Media Type` |
| Invalid compressed body | `400 Bad Request` |

## Affected/not affected projects

This proposal affects only the [Strimzi Kafka Bridge](https://github.com/strimzi/strimzi-kafka-bridge).

## Compatibility

Existing clients that send uncompressed requests keep working without changes.

## Rejected alternatives

### Use a reverse proxy

Users who already run a reverse proxy in front of the Bridge can decompress requests there, for example by creating an `EnvoyFilter` if using Istio.
Users without a reverse proxy would have to deploy and maintain one only to decompress requests for the Bridge.
