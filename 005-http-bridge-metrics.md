# HTTP Kafka bridge metrics

This proposal is about defining useful metrics we can expose from the HTTP Kafka bridge and a related Grafana dashboard.
Right now, they are just Vert.x metrics from the HTTP server provided out of the box.

# Vert.x provided HTTP server metrics

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

# Custom bridge metrics

While the above metrics provides information about the HTTP part of the bridge, we are lacking the Kafka part.
We could add some custom metrics which could have more info than the ones provided via HTTP requests:

* Created consumers
    * Could bring labels like “consumergroup”, “instanceid”, “base_uri” (not sure if this one is really useful). It could increase a consumers counter on the dashboard.
* Subscription (requests)
    * Could bring labels as “topics” but it should be a list, how to leverage that? What when the client uses “topic_pattern” and not specific topics in the request?
* Deleted consumer
    * Could bring labels like “consumergroup”, “instanceid”. It could decrease the previous consumers counter on the dashboard.
* Poll (requests)
    * Could bring labels like “consumergroup”, “instanceid”, number of “records” polled
* Send (requests)
    * Could bring labels like “topic” and number of “records” sent

Maybe implementing these specific ones, we could get rid the HTTP requests related ones for the various bridge operations but leaving the more general ones about rates by methods and by response codes (which could raise some malfunctions in the bridge or clients if too many 4xx or 5xx are there). The same could apply to the response time metrics.

# Next steps

Agreeing on which kind of HTTP server metrics we should keep and if some aggregation/filtering could make more sense.
Agreeing on the proposed custom bridge metrics for the Kafka part of the bridge itself.
Building a final Grafana dashboard for the bridge.