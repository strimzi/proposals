<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Proxy-Based Kafka Per-Topic Encryption

The goal of this proposal is to provide topic-level encryption-at-rest for Kafka such that distinct encryption keys can be used for different topics. 
A core library is proposed which is deployed into a proxy. Proxies can flexibly be deployed in a variety of topologies and require no changes to Kafka clients and brokers. All encryption/decryption takes place at the proxy. An unencrypted topic's messages pass through the proxy with only the minimal overhead associated with proxying and message inspection.

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
   "kms" : {
     "type"        : "ibmkeyprotect",
     "url"         : "https://kms.abc.com",
     "credentials" : "dkT4WSCnrlNE7",
     "instance-id" : "11111111-2222-3333-4444-55555555555",
     "key-ref"     : "e2a73a6c4bf9"
  }   
 },
 {
   "topic" : "creditcard-data",
   "kms" : {
     "type"        : "ibmkeyprotect",
     "url"         : "https://kms.abc.com",
     "credentials" : "dkT4WSCnrlNE7",
     "instance-id" : "11111111-2222-3333-4444-55555555555",
     "key-ref"     : "9957cdf5f32a"
  }   
]
```

In this example, two topics will be encrypted with two different keys residing in the same KMS.

The following table describes the configuration elements.

| Element |  Description | Required | Comments |
|---------|--------------|----------|----------|
| `topic`   | The name of the Kafka topic to encrypt. | Yes | |
| `kms.type`| Identifies the type of KMS used. This is a string constant corresponding to an implementation of the internal KMS interface. |Yes| Currently `ibmkeyprotect` is supported in the reference implementation. An example of another type could be `vault`. |
|`kms.url` | The URL of the KMS system | Yes | |
|`kms.credentials` | An opaque credential used to access the KMS service. | No | The credential can be a token, apikey or user:password string but must be understood by the respective KMS system. |
| `kms.instance-id` | An instance-ID is any parameter in addition to the `kms.url` required to connect to the KMS. | No | Not all KMS systems require this parameter. IBM Key Protect does while Vault does not. If not required, this field can be left out of the policy definition.|
| `kms.key-ref` | A unique identifier of the key material used in encrypting the topic data.| Yes | The format of this field depends on the underlying KMS. |

_Because a policy contains credentials, the entire policy must be treated as a secret._ In a Kubernetes environment, policy could be passed to the encrypting proxy with the secrets mechanism. Policy could also be retrieved policy from a secrets store such as Vault.

Updates to policy are naturally possible but not under all circumstances. As topics can be created and deleted at any time, policy must be updated dynamically without restarting the proxy.  The proxy will have a management interface over which  notifications of policy updates are issued. 

Certain policy updates can produce a constellations not supported initially. Specifically, those changes resulting in a mix of encrypted and unencrypted data within a topic will not be supported (i.e., an unencrypted topic with data becomes encrypt or vice versa). Further, changing KMS parameters is possible only if the encryption semantics are fully preserved by the change. For example, if the encryption scheme stores AES keys in a KMS, moving the AES keys to a different KMS and reconfiguring policy for the new KMS is possible. However some policy changes result in a topic depending on more than one KMS. Such changes will not be supported.  As an example, if the encryption scheme depends on unique key material in the KMS, such as a default root key used for envelope encryption, a policy change will not be supported since the topic would consist of wrapped keys from two KMS systems. For such cases, a new topic must be created with the new KMS and the original topic data copied to the new topic, whereafter the first topic can be deleted.

### Message Handling
Message handling is concerned with facilities for intercepting messages, deserializing and inspecting them, invoking encryption as accorded by policy, modifying messages, and finally serializing and forwarding messages onward to Kafka brokers and clients. To accomplish encryption and decryption of data payloads, only Kafka *Produce requests* and *Fetch responses* need to be examined and potentially modified. High quality open source libraries exist already for implementing message introspection and serialization/deseriaization, e.g.,
https://github.com/Shopify/sarama.

### Message Encryption
As described in [Kafka message format documentation](https://kafka.apache.org/documentation/#messageformat), data is transmitted in [record batches](https://kafka.apache.org/documentation/#recordbatch). Within a batch, each [record](https://kafka.apache.org/documentation/#record) is comprised of several fields of which two may contain data and therefore are of direct relevance for topic encryption: the _value_ field and the _headers_ field, an array of optional [record header structures](https://kafka.apache.org/documentation/#recordheader). _Key_ fields are never encrypted in this proposal. Any functionality relying on keys such as partitioning and compaction are unaffected and thus preserved by topic encryption.

As high volumes of data will be encrypted, symmetric key encryption is the natural choice for efficiently ensuring the _confidentiality_ of stored topic messages.
The Advanced Encryption Standard (AES) is an efficient symmetric encryption algorithm. AES instructions are supported by many modern processors (e.g., Intel AES-NI) providing fast, constant time encryption and resilience to certain side-channel attacks. Galois/Counter Mode (GCM) is an authenticated encryption mode for block ciphers pro- viding _integrity_. AES-GCM lends itself to parallelization and is widely used for its combination of high-throughput, integrity and authentication properties. AES-GCM has been used in the reference implementation and is the recommended algorithm in this proposal. As encryption is modularized, the design can be extended to support other encryption schemes in the future.

An example is now described to illuminate the essential details of the message encryption process.
The encryption module is embedded in a proxy and the proxy, as an intermediary, terminates  connections between clients and the broker, placing the proxy in the position of intercepting all Kafka traffic. The message handling component examines all messages to identify the apikey (i.e., the Kafka message type). All message types besides Produce and Fetch are  immediately forwarded without any processing. With the help of the message handling library, Produce requests are deserialized and inspected more deeply to identify the topic for which records are destined.
If a topic matches a topic in the encryption policy, the contents of the _value_ field of each record is encrypted and replaced with the corresponding ciphertext. Metadata about the encryption (e.g., encrypted key or key reference, nonce) is added to the record's headers in order that this information is available later for decryption.

The encryption metadata is currently stored in Kafka record headers. Record headers are optional name value/pairs stored in an array associated with the record. Record header names are not required to be unique, thus name "collisions" are possible. Ordering and versioning are used to avoid problems arising from names used for metadata which coincidentally equal names chosen by a Kafka client. Ordering means that the record header is rewritten such that the encryption-related headers
occur first in the array. The first header always contains a version from which the exact
metadata field names can be derived. The decryption algorithm therefore can discern between headers added by the encryption module and those added by the client. During decryption, the headers previously added to hold encryption metadata are removed before forwarding the decrypted response to the client. The version field enables future migration and backwards compatibility should changes in semantics occur such as modified message formats or encryption options.

Although not part of our reference implementation, the encryption of header values is potentially an additional requirement to consider. In this case, policy would be extended to indicate that specific or all client header values be encrypted. Since record headers are already processed during encryption, and since the algorithm can discriminate between encryption and client headers, it should be an incremental extension to encrypt header data as well. Encrypting header values will not impinge on storing encryption metadata in headers as only client headers will be encrypted. 
Encrypting header names as well would likewise be a relatively simple extension however we recommend further investigation of the full implications of encrypting header names.

### Key Rotation
Periodic rotation of symmetric encryption keys is a security best practice and compliance requirement.
As Kafka data is immutable, there are no update semantics for replacing data stored at the broker with a new version encrypted with a new key. 

A simple, "brute force" approach to topic re-encryption is to copy the existing topic to a new topic associated with the new key. This would take place ideally as a maintenance action, conceivably with broker downtime for the duration of  copying. Thereafter clients must configure the new topic name.

Alternatives or enhancements to this base approach include modifications to the broker, optimized rewriting of
topic segment files, or the use of envelope encyption.  Such enhanced methods are certainly related and highly relevant but viewed as outside the scope of this core encryption proposal.  


### Integrating Topic Encryption in a Proxy
A proxy can be deployed as a free-standing process or as a sidecar in a Kubernetes pod. 
In both cases, the proxy intercepts and de-serializes Kafka messages, encrypts/decrypts  relevant records, 
and reserializes new messages. The overhead of a proxy potentially has an effect on performance.
It also means that the proxy version must always be in phase with that of the broker.
A proxy-based solution however has the advantage that both Kafka client and broker are unaware
of the proxy and do not require modification to support encryption at rest.

Whether the proxy runs under the control of the client or the broker is, in many ways, considered a configuration issue, however there are important differences, most notably regarding the relationship between broker and proxy. Broker-side proxies typically will be deployed together with brokers, assuring same Kafka protocol version support by proxy and broker. Additionally, configuration settings harmonizing proxy and broker are more easily made when both are deployed in close proximity. Client-side proxies, deployed independently of brokers, will need to
to have dynamic protocol version awareness of its own and the broker's version. In both deployment models,
the proxy will use [`ApiVersionsRequest`](https://kafka.apache.org/protocol#api_versions) to ascertain the
broker version in order to  handle potential version discrepancies.

Broker-side proxies must appear to Kafka clients as brokers. This requires that proxy addresses and ports appear both in Kafka client configurations and in Zookeeper responses. 
This can be achieved with the broker 
[`advertised.listeners`](https://kafka.apache.org/documentation/#advertised.listeners) property which allows hostnames and ports other than the brokers', namely those of the proxies, to be communicated to
clients through Zookeeper. Thus with configuration of the broker and knowledge of proxy hostnames and ports,
clients will communicate strictly with the proxies, not circumvent them.
This model poses no complications relating to the use of TLS. The proxy will possess
 certificates to serve incoming client connections as well as to connect to backend brokers, for example over mTLS.

SASL support is another area touched by the deployment of proxies.  In our reference work thus far, SASL was used to integrate with a native authentication environment. Further analysis is required to address the full
range of SASL options supported by Kafka and their respective implications for a proxy.

Client-side proxies presumably are deployed more numerously and configured somewhat differently than broker-side proxies.  Using the broker's advertised list, as described above, to route client requests to a list
of known proxies will not be practical on a client where only one proxy is viable, namely the local proxy.  In this case, the advertiser list may be configured diffently, possibly to `localhost` or a hostname assumed to be in a local hosts file. A central policy repository avoids maintaining multiple versions of encryption policy.

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
