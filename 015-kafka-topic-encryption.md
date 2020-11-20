<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Proxy-Based Kafka Per-Topic Encryption

The goal of this proposal is to provide topic-level encryption-at-rest for Kafka such that distinct encryption keys can be used for different topics. 
A core library is proposed which is deployed into a proxy. Proxies can flexibly be deployed in a variety of topologies and require no changes to Kafka clients and brokers. All encryption/decryption takes place at the proxy. An unencrypted topic's messages pass through the proxy without being changed and without
paying any performance penalty.

This proposal is based on a complete, working implementation.

## Current situation

Strimzi does not have a current capability in this area. 

## Motivation

Apache Kafka does not directly support any form of encryption-at-rest for data stored at a broker.
Nevertheless, Kafka is increasingly used as a store of data, not just as a
means of transferring it from one location to another. In an
enterprise, this means that Kafka must conform to the same security and
compliance requirements as conventional data storage systems such as relational databases. 
To compensate, Kafka service providers typically use disk or file system encryption. 
This approach has well-known shortcomings, most notably the ability of anyone with appropriate file system permissions, such as system administrators, to read data. Security best practices and compliance standards
explicitly call for application- or database-level encryption to protect sensitive information against such exposures.
This document proposes a technical solution for providing upper-layer encryption-at-rest for Kafka systems.

## Proposal

### Overview
An implementation of topic encryption is proposed whereby the message stream between client and broker is intercepted. Incoming data messages (Produce requests) are inspected to determine whether their payload
should be encrypted according to a policy. If so, the data portions are encrypted and the modified
message is forwarded to the broker. As a result, the topic data is stored by the broker in encrypted form.
On the reverse direction, encrypted responses (responses to Fetch requests) are decrypted prior to being sent to clients.

As defined in the encryption policy, each topic can be encrypted by a different key,
allowing brokers to store a mix of encrypted and unencrypted data, where
data owners can manage the keys to their topics.
Keys ideally are stored in a key management system with access policies and logging. 

A core topic-encryption component, termed the _Encryption Module_, is proposed which is then deployed in a proxy. 

The diagram below depicts the main components of the proposal, illustrating clients sending and receiving plaintext messages while the proxy exchanges ciphertext messages with the broker:

![overview](images/015-kafkaenc-overview.png)

### Components
#### Encryption Module
The Encryption Module is the top-level component which encapsulates encryption functionality and is embedded in a proxy.
A central design goal is adaptability through modularity. Encrypter/decrypter(s), the policy metadata service and the key management service (KMS) are expressed as interfaces and pluggable. The Encryption Module instantiates these components during initialization, allowing a configurable combination of KMS, metadata and encryption implementations.

#### Encrypter-Decrypter
The Encrypter-Decrypter sub-component performs the actual encryption and decryption of messages, implementing a particular encryption algorithm and making use of the KMS service to obtain topic keys.

#### Key Service
The Key Service is used in the Encryption Module to handle interactions with a key management system and to perform optional primitives such as those related to envelope encryption. The interface is generic in order to accommodate various key management implementations.

#### Policy Service
Encrypter-decrypters require a means to lookup encryption requirements and parameters for a topic. As a corresponding standard does not exist, the core component will interact with a generic internal interface for retrieving policy on how a topic is, or not is, to be encrypted. Implementers can develop and configure their own implementation of the policy service, be it a remote microservice, a local pre-configured cache, a cluster registry, etc.

For each topic to be encrypted, a policy exists detailing:

- topic name
- encryption algorithm(s), cipher suite, and encryption/decryption steps
- key management server address
- key identifier
- optional initialization information

In our reference implementation, policies are encoded in JSON. An example topic encryption policy,
used in testing, is shown below:

```
[
{
"topic" : "juicy-topic",
"kms_url" : "https://kms.abc.com",
"kms_instance": "11111111-2222-3333-4444-55555555555",
"key_ref" : "e2a73a6c4bf9",
"credential" : "dkT4WSCnrlNE7"
},
{
"topic" : "creditcard-data",
"kms_url" : "https://kms.abc.com",
"kms_instance": "11111111-2222-3333-4444-55555555555",
"key_ref" : "9957cdf5f32a",
"credential" : "dkT4WSCnrlNE7"
}
]
```

In this example, two topics will be encrypted with two different keys residing in the same KMS.

### Message Handling
Message handling is concerned with facilities for intercepting messages, deserializing and inspecting them, invoking encryption as accorded by policy, modifying messages, and finally serializing and forwarding messages onward to Kafka brokers and clients. To accomplish encryption and decryption of data payloads, only Kafka *Produce requests* and *Fetch responses* need to be examined and potentially modified. High quality open source libraries exist already for implementing message introspection and serialization/deseriaization, e.g.,
https://github.com/Shopify/sarama.

### Message Encryption
As described in [Kafka message format documentation](https://kafka.apache.org/documentation/#messageformat), data is transmitted in [record batches](https://kafka.apache.org/documentation/#recordbatch). Within a batch, each [record](https://kafka.apache.org/documentation/#record) is comprised of several fields of which two may contain data and therefore are of direct relevance for topic encryption: the _value_ field and the _headers_ field, an array of optional [record header structures](https://kafka.apache.org/documentation/#recordheader). _Key_ fields are never encrypted in this proposal. Any functionality relying on keys such as partitioning and compaction are unaffected and thus preserved by topic encryption.

An example is now described to illuminate the essential details of the message encryption process.
The encryption module is embedded in a proxy and the proxy, as an intermediary, terminates  connections between clients and the broker, placing the proxy in the position of intercepting all Kafka traffic. The message handling component examines all messages to identify the apikey (i.e., the Kafka message type). All message types besides Produce and Fetch are  immediately forwarded without any processing. With the help of the message handling library, Produce requests are deserialized and inspected more deeply to identify the topic for which records are destined.
If a topic matches a topic in the encryption policy, the contents of the _value_ field of each record is encrypted and replaced with the corresponding ciphertext. Metadata about the encryption (e.g., encrypted key or key reference, nonce) is added to the record's headers in order that this information is available later for decryption.

The encryption metadata is currently stored in Kafka record headers. Record headers are optional name value/pairs stored in an array associated with the record. Record header names are not required to be unique, thus name "collisions" are possible. Ordering and versioning are used to avoid problems arising from names used for metadata which coincidentally equal names chosen by a Kafka client. Ordering means that the record header is rewritten such that the encryption-related headers
occur first in the array. The first header always contains a version from which the exact
metadata field names can be derived. The decryption algorithm therefore can discern between
headers added by the encryption module and those added by the client. The version field
is enables future migration and backwards compatibility should changes in semantics occur such as modified message formats or encryption options.

Although not part of our reference implementation, the encryption of header values is potentially an additional requirement to consider. In this case, policy would be extended to indicate that specific or all client header values be encrypted. Since record headers are already processed during encryption, and since the algorithm can discriminate between encryption and client headers, it should be just an incremental extension encrypt header data as well. Encrypting header values will not impinge on storing encryption metadata in headers as only client headers will be encrypted. 
Encrypting header names as well would likewise be a relatively simple extension however we recommend further investigating the full implications of encrypting header names.

### Embedding Topic Encryption in a Proxy
A proxy can be deployed as a free-standing process or as a sidecar in a Kubernetes pod. 
In both cases, the proxy needs to intercept and de-serialize Kafka messages, encrypt/decrypt the relevant records, 
and reserialize them into a new message. The overhead of a proxy potentially has an effect on performance.
It also means that the proxy version must always be in phase with that of the broker.
A proxy-based solution however has the advantage that both Kafka client and broker are unaware
of the proxy and do not require any modification or configuration to support encryption at rest.

Whether the proxy runs under the control of the client or the broker is considered a configuration
issue, i.e. exactly the same proxy would be used in both cases, but when under the control of the
client it would be their responsibility to ensure that producers/consumers are passing through a proxy with the same configuration.

Envoy is one possible framework for creating proxies. Envoy’s connection 
pipeline is based on network filters which are linked into filter chains, enabling rich capabilities. 
Recent support of WebAssembly (WASM) provides more flexible and dynamic way
to extend Envoy and embed Kafka topic encryption.

Envoy is but one viable approach, certainly not the only means, to embed topic encryption
in a proxy. 

## Affected/not affected projects

This proposal does not conflict or overlap with specific Strimzi projects but rather
provides a complementary, new functionality which ideally will become part of the
Strimzi Kafka distribution.

## Compatibility

As this is a new capability, there are initially no backwards compatibility issues relating to this proposal.
Because the proxy intercepts Kafka connections and modifies messages, this proposal is tied tightly to Kafka versions.
The proxy will incorporate any ongoing protocol and message format changes in future Kafka versions while supporting 
earlier Kafka clients.

## Rejected alternatives

Different approaches to implementing Kafka topic encryption have been considered, most notably
client-side encryption and embedding encryption in the Kafka broker.

### Client-based encryption
In the client-based model, Kafka producers encrypt messages and consumers decrypt them.
They must share policy in order to determine which keys should be used
for which topics as well as access a common KMS for accessing shared keys.
The broker is oblivious to encryption and no broker changes are required. 
Encryption-at-rest is thus achieved with any broker, even those not under a client's control.

We have implemented encrypting clients in both Java and python.
While Kafka provides an interceptor API in its Java client, no such construction exists for the python client.
As a result, we programmed a wrapper around a python Kafka client (there are more than one python clients) in order to intercept calls, subsequently transforming messages and delegating to the contained client instance. Such custom solutions must be repeated for each language client. 

In summary, client encryption requires additional configuration at the edge systems
and coordination between producers and consumers. 
As there is no standardization in the structure of Kafka client libraries,
multiple versions of the topic encryption libraries must be developed
and maintained in various languages (e.g., Java, python, C, golang, etc.).

### Broker modification

A broker-based model involves modification to the Apache Kafka source code in order to embed encryption directly in the broker. An advantage of this approach is that encryption at rest becomes native to the broker, corresponding to traditional database level encryption. One drawback however is the need to maintain a fork of the broker source for each supported version. Furthermore, modifications to broker internals have the potential of disrupting carefully engineered optimizations or critical sections. Many organizations will prefer the general advantage of running standard Kafka distributions as opposed to custom brokers.
