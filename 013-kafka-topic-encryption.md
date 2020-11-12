<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Proxy-Based Kafka Per-Topic Encryption

*Provide a brief summary of the feature you are proposing to add to Strimzi.*

The goal of this proposal is to provide topic-level encryption-at-rest for Kafka. A core library is proposed which is deployed into a proxy. Proxies can flexibly be deployed in a variety of topologies and require no changes to Kafka clients and brokers.

This proposal is based on a complete, working implementation.


## Current situation

*Describe the current capability Strimzi has in this area.*

Strimzi does not have a current capability in this area

## Motivation

*Explain the motivation why this should be added, and what value it brings.*

Apache Kakfa does not directly support any form of encryption-at-rest for data stored at a broker.
Nevertheless, Kafka is increasingly used as a store of data, not just as a
means of transferring it from one location to another. In an
enterprise, this means that Kafka must conform to the same security and
compliance requirements as conventional data storage systems such as relational databases. 
To compensate, Kafka service providers typically use disk or file system encryption. 
This approach has well-known shortcomings, most notably the ability of anyone with appropriate file system permissions, such as system administrators, to read data.  Security best practices and compliance standards
explicitly call for application- or database-level encryption to protect sensitive information against such exposures.
This document proposes a technical solution for providing upper-layer encryption-at-rest for Kafka systems.


## Proposal

*Provide an introduction to the proposal. Use sub sections to call out considerations, possible delivery mechanisms etc.*

A core topic-encryption component, termed the _Encryption Module_, is proposed which is then deployed in a proxy. The 

### Enryption Module
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
- encryption algorithm(s), cipher suite,  and encryption/decryption steps
- key management server address
- key identifier
- optional initialization information


### Message Interception
Message interception is concerned with facilities for inspecting messages, consulting a policy, and accordingly applying encryption so that messages are passed to the broker in encrypted form and returned to Kafka clients as decrypted plaintext. Specifically, Kafka Produce requests and Fetch responses are examined and potentially modified to respectively encrypt and decrypt messages.

### Proxy
A proxy can be implemented as a free-standing process or as a sidecar. In both cases, the proxy needs to
intercept and de-serialize the Kafka messages, encrypt/decrypt the relevant parts, 
and reserialize them. The overhead of a proxy potentially has an effect on performance.
It also means that the proxy version must always be in phase with that of the broker.
A proxy-based solution however has the advantage that both Kafka client and broker are unaware
of the proxy and do not require any modification or configuration to support encryption at rest.

Initially a free-standing proxy will be developed. 

A second proxy option is Envoy. Envoy is a general framework for creating proxies within Kubernetes used for example by Istio. Envoy’s connection is based on network filters that can be linked into filter chains, allowing rich capabilities to be composed from small microservices. This is contrast to monolithic proxy solutions like NGINX. Envoy's fairly recent support of WebAssembly (WASM) is the preferred model for introducing the topic encryption into an Envoy proxy.



## Affected/not affected projects

Call out the projects in the Strimzi organisation that are/are not affected by this proposal. 


## Compatibility

Call out any future or backwards compatibility considerations this proposal has accounted for.

## Rejected alternatives

Call out options that were considered while creating this proposal, but then later rejected, along with reasons why.
