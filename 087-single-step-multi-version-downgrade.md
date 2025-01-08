# Support Single step multi version downgrade for KRaft based clusters 

This proposal seeks to introduce support for multi-version downgrade of Strimzi in a single step where possible.
The proposal only relates to KRaft based clusters.

## Current situation

- Downgrading of Strimzi is currently supported where the Kafka version in use is supported by the 'to' Strimzi version. 
- Downgrading of Strimzi is not supported if the Kafka version in use is unsupported by the 'to' Strimzi version. Attempting this will result in an error during reconciliation of the Kafka CR. Downgrading the Kafka version to a version supported by the 'to' Strimi version will not resolve the reconciliation failure as the 'from' Kafka version is unknown to the 'to' Strimzi version (in contrast to the upgrade scenario where upgrading the Kafka version will enable reconciliation to subsequently succeed). The only way to successfully execute such a downgrade path is to iteratively downgrade Kafka and Strimzi ensuring at each step that the current Kafka version supported by the Strimzi version.  
- Downgrading of Kafka where the metadata version in use is greater than the maximum supported by the 'to' Kafka version is not supported

## Motivation

This proposal would make downgrade of Strimzi more user friendly, specifically, in the scenario where a single-step multi version upgrade has been performed and a need arises to rollback before the metadata version has been advanced. The proposal will allow the user perform the roll back in a single step rather than the multiple steps currently required.

## Proposal

During downgrade an attempt is made to read the Kafka information for the 'from' Kafka version by the 'to' version of the operator. When the 'from' Kafka version is not supported by the 'to' Strimzi version an error will be thrown because the version is unknown. Information such as the metadata version, interbroker protocol message format, log message format version are read from kafka-versions.yaml during the creation of a KafkaVersion object to represent the 'from' kafka version and the error message is generated as it cannot find the information for the unknown version.

However, while this information is important to know for the 'to' version in upgrade and downgrade, and for the 'from' version in upgrade, it is not important for the 'from' version in the downgrade as it is not subsequently used.

Kafka does not support downgrade where the metadata version in use is higher than the highest metadata supported by the 'to' Kafka version or where there has been changes in the metadata between the 'from' and 'to' versions. There is already logic in place in Strimzi to prevent such a downgrade from happening. This proposal does not propose any changes to this functionality, it shall continue to be not supported to downgrade in such a scenario. 

The proposed impact therefore is:
- Add handling to not throw an exception when the 'from' version is not known in a downgrade.

See https://github.com/strimzi/strimzi-kafka-operator/pull/10929

## Affected projects

strimzi-kafka-operator 

## Compatibility

N/A

## Rejected alternatives

Introducing similar functionality for Zookeeper based clusters rejected as the next version of Strimzi shall be the last to support Zookeeper and therefore there will never be a downgrade of a Zookeeper based cluster to that Strimzi version.
