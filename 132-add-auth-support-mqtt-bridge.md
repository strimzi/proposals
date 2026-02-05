# Add authentication/Authorization support to MQTT Bridge

This proposal is to add support for authentication and authorization to the MQTT Bridge component of Strimzi. This would allow devices to secure their MQTT Bridge deployments and **control access to the MQTT topics and resources**.

## Current situation

No authentication or authorization support is currently available in the MQTT Bridge. This means that anyone can connect to the MQTT Bridge and access any topics or resources without any restrictions. This can lead to security vulnerabilities and unauthorized access to sensitive data.

## Motivation

The motivation for adding authentication and authorization support to the MQTT Bridge is to enhance the security of the MQTT Bridge deployments. By implementing authentication and authorization mechanisms, devices can ensure that only authorized devices can access the MQTT Bridge and its resources. This will help to protect sensitive data and prevent unauthorized access.

## Proposal

Since the most ideal use case for the MQTT bridge is to connect IoT devices to Kafka, and some of these devices may be under constraints environments, I propose to implement a simple authentication mechanism by enabling secure MQTT connections(also known as MQTT over TLS) and using client certificates for authentication. This approach is widely supported by MQTT clients and provides a secure way to authenticate devices without the need for complex authentication protocols.

Since our bridge is not a full MQTT broker, we will not implement a full authorization mechanism. Instead, we can implement a simple topic-based access control mechanism that allows devices to specify which topics they are allowed to publish or subscribe to. This can be achieved by using client certificates to identify devices and associating them with specific topic permissions.

And also, using usernames and passwords for authentication is another alternative, however, our bridge is not a full MQTT broker, and implementing a secure password storage and management system might be out of scope for this project. Therefore, we will focus on using client certificates for authentication to secure the connections to the MQTT Bridge.

## Affected/not affected projects

This proposal will primarily affect the MQTT Bridge component of Strimzi.

## Compatibility

We are currently using MQTT version 3.1.1. In future, if we consider to upgrade to MQTT version 5.0, which has introduced new features to address authentication, we will need to ensure that this proposal implements authentication and authorization support in a way that is compatible with both MQTT 3.1.1 and MQTT 5.0.

## Rejected alternatives


