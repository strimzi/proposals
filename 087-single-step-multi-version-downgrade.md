# Support Single step multi version downgrade for Zookeeper based clusters 

This proposal seeks to introduce support for multi-version downgrade of Strimzi in a single step where possible.
The proposal only relates to Zookeeper based clusters.

## Current situation

- Downgrading of Strimzi is currently supported where the Kafka version in use is supported by the 'to' Strimzi version. 
- Downgrading of Strimzi where the Kafka version in use is unsupported by the 'to' Strimzi version is not supported. An attempt to do so will result in an error during reconcoliation of the Kafka CR. Downgrading the Kafka version to a version supported by the 'to' Strimi version will not resolve the reconciliation failure as the 'from' Kafka version is unknown to the 'to' Strimzi version (in contrast to the upgrade scenario where upgrading the Kafka version will enable reconcilation to subsequently succeed). The only way to successfully execute such a downgrade path is to iteratively downgrade Kafka and Strimzi ensuring the current Kafka vesion is at all stages supported by the Strimzi version.  
- Downgrading of Kafka where the inter broker protocol version in use is greater than the IBP version of the 'to' Kafka version is not supported. This is due to a limitation in Kafka downgrade support rather than any limitation in Strimzi.

## Motivation

This proposal would make downgrade of Strimzi more user friendly, in specific, in the scenario where a single-step multi version upgrade has been performed and a need arises to rollback before the IBP version has been stepped. The proposal will allow the user perform the roll back in a single step rather than the multiple steps currently required.

## Proposal

During downgrade an attempt is made to read the Kafka information for the 'from' Kafka version by the 'to' version of the operator. When the 'from' Kafka version is not supported by the 'to' Strimzi version an error will be thrown because the version is unknown. Information such as the zookeeper version, interbroker protocol message format, log message format version are read from kafka-versions.yaml during the creation of a KafkaVersion object to represent the 'from' kafka version and the error message is generated as it cannot find the information for the unknown version.

However, while this information is important to know for the 'to' version in upgrade and downgrade, and for the 'from' version in upgrade, it is not important for the 'from' version in the downgrade.
The only place any of this information for the 'from' version is used in the downgrade is in a log message to log if zookeeper needs to be downgraded. While this might be a useful log to output when the information is available, I dont think it could be considered important enough to be a reason for not supporting multi version single step downgrade.

Kafka does not support downgrade where the IBP version in use is higher than the IBP supported by the 'to' Kafka version. There is already logic in place in Strimzi to prevent such a downgrade from happening. This proposal does not propose any changes to this functionality, it shall continue to be not supported to downgrade in such a scenario. 

In the upgrade scenario where the Kakfa and IBP versions are not set in the CR, Strimzi will automatically step the Kafka version and then the IBP version to the defaults for the 'to' Strimzi version. It is NOT proposed to introduce similar behaviour to downgrade the IBP version automatically due to the above limitation in Kafka downgrade. Therefore the behaviour where the Kafka and IBP versions are not set in the CR will be to downgrade Kafka only if the IBP does not need to be downgraded. The already existing logic mentioned in the above paragraph will ensure this is the case.

Kafka feature version levels are only relevant for KRaft based clusters and are hence not relevent here as this proposal is focused on Zookeeper based clusters.

The proposed impacts therefore are:
- Add handling to not throw an exception when the 'from' version is not known in a downgrade.
- Update the logging message related to zookeeper needing to be updated to handle scenario where this not determined

See https://github.com/strimzi/strimzi-kafka-operator/pull/10802

## Affected/not affected projects

strimzi-kafka-operator 

## Compatibility

N/A

## Rejected alternatives

N/A
