# Support Single step multi version downgrade for KRaft based clusters 

This proposal seeks to introduce support for multi-version downgrade of Strimzi in a single step where possible.
The proposal only relates to KRaft based clusters.

## Current situation

- Downgrading of Strimzi is currently supported where the Kafka version in use is supported by the 'to' Strimzi version. 
- Downgrading of Strimzi is not supported if the Kafka version in use is unsupported by the 'to' Strimzi version. Attempting this will result in an error during reconciliation of the Kafka CR. Downgrading the Kafka version to a version supported by the 'to' Strimi version will not resolve the reconciliation failure as the 'from' Kafka version is unknown to the 'to' Strimzi version (in contrast to the upgrade scenario where upgrading the Kafka version will enable reconciliation to subsequently succeed). The only way to successfully execute such a downgrade path is to iteratively downgrade Kafka and Strimzi ensuring at each step that the current Kafka version supported by the Strimzi version.  
- Downgrading of Kafka where the metadata version in use is greater than the maximum supported by the 'to' Kafka version is not supported

## Motivation

This proposal would make downgrade of Strimzi more user friendly, specifically, in the scenario where a single-step multi version upgrade has been performed and a need arises to downgrade before the metadata version has been advanced. The proposal will allow the user perform the roll back in a single step rather than the multiple steps currently required.

## Proposal

During downgrade an attempt is made to read the Kafka information for the 'from' Kafka version by the 'to' version of the operator. When the 'from' Kafka version is not supported by the 'to' Strimzi version an error will be thrown because the version is unknown. Information such as the metadata version, interbroker protocol message format, log message format version are read from kafka-versions.yaml during the creation of a KafkaVersion object to represent the 'from' kafka version and the error message is generated as it cannot find the information for the unknown version.

However, while this information is important to know for the 'to' version in upgrade and downgrade, and for the 'from' version in upgrade, it is not currently used for the 'from' version in the downgrade. It might be potentially useful in some some future downgrade scenarios, but none that are currently supported.

Kafka does not support downgrade where the metadata version in use is higher than the highest metadata supported by the 'to' Kafka version or where there has been changes in the metadata between the 'from' and 'to' versions. There is already logic in place in Strimzi to prevent such a downgrade from happening. This proposal does not propose any changes to this functionality, it shall continue to be not supported to downgrade in such a scenario. 

### Impacts:

There are two impacts as detailed in the below sub-sections.

#### Handle unknown versions

KraftVersionChangeCreator.detectToAndFromVersions() invokes the version() method in KafkaVersion.Lookup which throws an exception if the requested version is not known.
This exception can be caught in detectToAndFromVersions() in the downgrade scenario and an instance of KafkaVersion created that has null values for the unknown data (such as metadata version). The version(s) from the strimzi.io/kafka-version annotations on the existing pods will be compared with the version from the custom resource to identify if a downgrade is taking place.
As this data is not currently read anywhere in the code the null values will not have any negative effect.

#### Handling for specific versions

It is possible that future Kafka versions might need extra steps to be executed for a particular upgrade/downgrade path.
Similarly future versions of the operator might be capable of making configuration changes that previous operator versions would not be capable of undoing. 

Documentation of the downgrade procedure for any upgrade/downgrade paths where such issues arise should document the specific risks for those paths and how or if it is possible to downgrade back in those scenarios.
It should also be documented that it is the responsibility of the user to ensure they validate the downgrade path they are taking to ensure no such issues exist. 


### System Tests

The following procedure would test the new functionality:
- Deploy the latest operator
- Deploy a Kafka cluster with the highest supported Kafka version but metadata version set to the appropriate version for the Kafka version the cluster will be downgrade to in the following steps
- Downgrade the operator to the most recent version which does not support the Kafka version used in the cluster.
- Downgrade the Kafka version used in the cluster to a version supported by the downgraded operator

HOWEVER, this verifies the functionality in the previous version of the operator rather than the latest version. The system tests are intended to test the latest version so there is no point creating a system test following this procedure. 

I therefore propose we do not introduce a new system test for this and instead cover it in unit test.

## Affected projects

strimzi-kafka-operator 

## Compatibility

N/A

## Rejected alternatives

Introducing similar functionality for Zookeeper based clusters rejected as the next version of Strimzi shall be the last to support Zookeeper and therefore there will never be a downgrade of a Zookeeper based cluster to that Strimzi version.

#### Handle unknown versions

Change the KafkaVersion class to an interface with two different implementations. 
One implementation for the case where the Kafka version is known (which would be functionally equivalent to the current KafkaVersion class), and another implementation for unknown Kafka versions.
The version() method in the Lookup class will return an instance of the appropriate class with tha calling method able to distinguish, when necessary, if the version is known or not based on the class of the returned instance.
The methods in the implementation for the unknown versions will throw an exception for all methods for which the information is not available (i.e. all except the version method) to ensure an error is generated if any code mistakenly tries to use information that is not available for an unknown version. 

#### Handling for specific versions

Add a new field to KafkaStatus, named minOperatorVersionForRollback, which is set by the operator during Kafka upgrade when it is known that specific extra steps are needed for a downgrade to succeed.
When a downgrade is identified (in KraftVerionChangeCreator.detectToAndFromVersions()) the version of the operator is compared with the value of minOperatorVersionForRollback, if set.
An exception will be thrown if the requirement is not satisfied. 

For example, if specific steps need to be implemented in the operator to upgrade/downgrade from Kafka version 'x' to Kafka version 'y' and such steps are implemented in operator version 'n' then as part of the implementation in operator version 'n' to support the 'x' -> 'y' upgrade, the minOperatorVersionForRollback will bet set to 'n'.
Any attempt to downgrade a Kafka cluster from 'y' -> 'x' with operator version earlier than 'n' shall be rejected by the operator.
