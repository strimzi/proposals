# Kafka Canary roadmap

This proposal describes a possible future set of improvement and features for the [Strimzi Canary](https://github.com/strimzi/strimzi-canary)

## Current situation

The project [README.md](https://github.com/strimzi/strimzi-canary/blob/main/README.md) explains the current feature set.

The Canary was added to Strimzi under [proposal 13](013-kafka-canary.md).
The implementation is in golang and uses the [Sarama](https://github.com/Shopify/sarama/) Kafka client.

## Motivation

We believe the Canary component could deliver significant value for Strimzi users, for example at the basis for an SLI on cluster availability and health.
However, these Canary improvements are held back by technical and non-technical issues:

* We've experienced [numerous bugs]((https://github.com/strimzi/strimzi-canary/issues?q=is%3Aissue+is%3Aclosed)) in the Kafka client library being used, and believe we're likely to see a long tail of further bugs.
* The client library doesn't support all the features we want. Those features would needed to realize the Canary's true potential.
* The Strimzi community's lack of golang expertise (or indeed enthusiasm) means it is somewhat unloved.

## Proposal

### Re-implement in Java

The Kafka project itself only provides support for JVM clients.
Naturally those clients are extremely well-tested, with both current and older broker versions. 
So generally speaking, we would expect the Canary to experience fewer issues due to client bugs.
It would also necessarily support the latest versions of the Kafka protocol, which would enable support for more features.

The Canary also stands out as being the only Strimzi project that's written in golang.
There's nothing wrong with golang, but there are non-technical drawbacks to this situation.
Most existing Strimzi developers have more experience with Java than golang.
Some don't really know golang at all.
It's difficult to motivate people to use another language if they're using it only for one relatively small part of their job.
Likewise, existing golang developers are not going to be drawn to contributing to Strimzi on account of a single golang component.
So over all we expect our community would be better able to engage with the Canary component if it was implemented in Java.

### Desirable features

There are several client features we would like to be able use.

* Support for SASL `OAUTHBEARER` mechanism. Sarama supports this.
* Support for forcing leader elections. Sarama doesn't support this.
* Support for transactional produce. Sarama doesn't seem to support this, at least not in a usable way. You can send the transaction-related requests, but there is no client library support akin to the `TransactionManager`, for ensuring the RPCs are sent correctly. 

### Differential observations

By running multiple producers and/or consumers, a canary could more accurately indicate where problems lay. 

For example, in a cluster with multiple listeners using different authentication mechanisms, if produce requests via a `SASL/OAUTHBEARER`-authenticated listener fail, but succeed via the `SASL/PLAIN`-authenticated listener you can infer that the problem is with the login mechanism. 

There are many dimensions in which you can do this: 

* DNS resolution
* TLS (e.g. different versions or detecting imminent broker certificate expiry
* Producing to replicas on different volumes
* With/without transactional producers, to flush out problems with transaction coordinators.
* Similarly with/without using consumer groups. Using different group ids to detect problems with group coordinators.

### Integration examples

Currently the Canary provides a binary and nothing more. 
This limits uptake of the canary because to use it for its intended purpose you need to configure prometheus scraping, alerting and dashboards to make sense of what the canary is telling you. 

We would like to provide example of all these things, as we [do for the operator](https://github.com/strimzi/strimzi-kafka-operator/tree/main/examples/metrics).

### Actionable documentation

Assuming the differential observations feature proves to provide worthwhile information, we would proceed to also provide documented procedures recommending corrective actions based on the specific faults it reported.

## Affected/not affected projects

This affects the Canary project only.

## Compatibility

The API of the existing canary comprises:

* The configuration env vars
* Metrics (via the Prometheus scrape endpoint)
* The health endpoints

### Configuration
While we can strive to minimize incompatibilities, it seems of questionable value.
The Java clients are normally configured via `.properties` files.
Providing a compatibility mapping from env var to properties would be a lot of work, given that the canary hasn't seem very wide usage.
Some options would become obselete (e.g. `KAFKA_VERSION`), others are incompatible with how the Java clients work (e.g. `KAFKA_BOOTSTRAP_BACKOFF_MAX_ATTEMPTS`).
So it seems better to provide a clean break.

### Metrics

It seems likely that metric names and labels can be maintained, so in that respect we could be compatible. 
Detailed assessment of whether the metric values were being measured in a compatible way is beyond the scope of this document.
If measurements were incompatible it would require existing users to adjust thresholds used in any alerting they might have that was based on the old metrics.

### Health endpoints

Providing compatibility for these will be trivial.

## Rejected alternatives

The main alternative would be to stick with golang and Sarama.
Doing this means:
* the feature improvements be blocked, waiting for support to be added for the missing features.
* the non-technical issues described would remain unaddressed. 