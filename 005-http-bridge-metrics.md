# HTTP Kafka bridge metrics

This proposal is about defining useful metrics we can expose from the HTTP Kafka bridge and a related Grafana dashboard.
Right now, they are just Vert.x metrics from the HTTP server provided out of the box.

## Vert.x provided HTTP server metrics

Following a list of HTTP server related metrics and corresponding Grafana panels.

The number of opened HTTP connections to the bridge.

![HTTP connections](/images/005-http-bridge-metrics/http_connections.png)

Requests being processed.

![Requests being processed](/images/005-http-bridge-metrics/requests_being_processed.png)

Requests processed/secs grouped by HTTP method.

![Requests by HTTP method](/images/005-http-bridge-metrics/requests_by_method.png)

The requests rate in total and even grouped by response codes (2XX, 4XX, 5XX).

![Requests and Response codes](/images/005-http-bridge-metrics/requests_by_code.png)

Bytes received /sec and Bytes sent /sec.

![Bytes received](/images/005-http-bridge-metrics/bytes_received.png)

![Bytes sent](/images/005-http-bridge-metrics/bytes_sent.png)

Furthermore, I used an HTTP requests metrics with different regex for getting information on different endpoints.
These HTTP requests/sec metrics bring labels like `code` (HTTP response code), `local` (bridge server listening), `remote` (client issuing the request), `method`, and `path`.
I created each one based on `method` and `path` as the following (which reflect some of the operations we have on the bridge).

* SEND to `/topics/<topic>`
* SEND TO PARTITIONS to `/topics/<topic>/partitions/<partition>`
* CREATE CONSUMER to `/consumers/<group>`
* SUBSCRIBE to `/consumers/<group>/instances/<consumer>/subscription`
* DELETE CONSUMER to `/consumers/<group>/instances/<consumer>`
* POLL to `/consumers/<group>/instances/<consumer>/records`
* COMMIT to `/consumers/<group>/instances/<consumer>/offsets`

![HTTP requests](/images/005-http-bridge-metrics/http_requests.png)

Maybe for the above, it could be useful just doing the sum and not having the rate for each `remote` client on each `path` and `method`.
I was thinking about filtering them by topics but it’s possible only for the producer part because the topic is in the `path` while when a consumer subscribes to a topic, it’s specified in the body which is not available from this metrics; maybe it could be part of a custom metric to add.

Maybe using another available metric about response time for processing each HTTP request (always provided with same label).

They are not reported here but we are also getting JVM related metrics as for the operators about memory, GC, and so on.

## Kafka clients provided metrics

The Kafka clients (admin, consumer and producer) used by the bridge exports metrics via JMX.
This metrics can be exposed in the Prometheus format (as the previous Vert.x related ones) for monitoring the Kafka part of the bridge.

Useful metrics about Kafka part could be the following.

TBD

## Rejected alternatives

Creating custom metrics for the Kafka part was rejected thanks to the JMX ones already available out-of-box.

# Next steps

Agreeing on which kind of HTTP server metrics we should keep and if some aggregation/filtering could make more sense.
Exporting the Kafka clients JMX metrics in the Prometheus format.
Agreeing on which Kafka client metrics to use for the Kafka part of the bridge itself.
Building a final Grafana dashboard for the bridge.