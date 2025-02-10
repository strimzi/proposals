# Proposal: Support for Custom Authentication Type in Kafka Clients

## Current situation

Strimzi currently supports several standard authentication mechanisms for Kafka clients, including:

- TLS-based authentication
- SASL mechanisms such as `scram-sha-256`, `scram-sha-512`, `plain`, and `oauth`

These mechanisms cover a wide range of use cases, but there is no built-in capability for users who need highly customized authentication logic.

## Motivation

Organizations may have unique security requirements that cannot be addressed by the existing authentication mechanisms. These requirements may involve integrating with proprietary security systems, leveraging custom token-based schemes, or handling complex credential management.

By providing support for custom authentication types, Strimzi will enable users to supply their own authentication logic while maintaining compatibility with Kafka security standards.

### Benefits:

- Greater flexibility for organizations with custom security requirements.
- Reduced need for workarounds when integrating with non-standard authentication systems.
- Better alignment with enterprise security architectures.

## Proposal

### Introduction

The proposal introduces a new Kafka client authentication type called `custom`. This type allows users to specify a fully qualified class name of a custom `AuthenticateCallbackHandler` implementation to handle their specific authentication logic.

### Configuration

The configuration for the custom authentication would look like this:

```yaml
authentication:
  type: custom
  customCallbackClass: com.example.kafka.auth.MyCustomAuthHandler
```

### Key Changes

1. **New Class:** `KafkaClientAuthenticationCustom`
    
    - Defines the `custom` authentication type.
    - Includes a property `customCallbackClass` for specifying the custom handler.
2. **Volume and Secret Handling:**
    
    - No volumes or secrets will be automatically mounted for the custom authentication type.
    - Users are responsible for managing any required resources.
3. **Validation:**
    
    - Strimzi will ensure the `customCallbackClass` property is provided and correctly formatted.

### Example Use Case

A user needs to authenticate against a proprietary identity service using a custom token validation mechanism. They implement the following custom handler:

```java
package com.example.kafka.auth;

import org.apache.kafka.common.security.auth.AuthenticateCallbackHandler;
import javax.security.auth.callback.Callback;
import java.util.Map;

public class MyCustomAuthHandler implements AuthenticateCallbackHandler {
    @Override
    public void configure(Map<String, ?> configs, String mechanism, List<Callback> callbacks) {
        // Custom configuration logic
    }

    @Override
    public void handle(Callback[] callbacks) {
        // Custom authentication logic
    }

    @Override
    public void close() {
        // Clean up resources if necessary
    }
}
```

## Affected/not affected projects

### Affected

- **Cluster Operator:** Will require changes to handle the `custom` authentication type.
- **API:** New class and CRD validation for `KafkaClientAuthenticationCustom`.
- **Documentation:** Updated guidance for configuring and using custom authentication.

### Not affected

- Kafka Brokers (custom logic is client-side).
- User Operator.

## Compatibility

This proposal maintains backward compatibility by introducing an additional authentication option. Existing configurations using standard authentication mechanisms remain unaffected.