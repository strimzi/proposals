# Refactor KafkaConfig files in Strimzi Client Examples

This proposal is about simplifying the `Kafka*Config` files of the Java modules of the [Strimzi Client Examples repo](https://github.com/strimzi/client-examples).

## Current situation

The `Kafka*Config` files of the Java modules in the [Strimzi Client Examples repo](https://github.com/strimzi/client-examples) have become quite complex since they support the configuration of several specific Kafka client properties via environment variables.
They contain a lot of the logic for configuring the client. 
As an example, you can have a look at the logic around setting Kafka’s `security.protocol` option which is spread across several places in the code.

## Motivation

The main purpose of the Client Examples is to show users how to write a simple Kafka client based on the Kafka Consumer, Producer and Streams API. 
However, over time, we added a lot of complexity to the client code because it was being used in the System Tests at some point in time. 
So a user who wants to learn from the Client Examples has to dig through this code to understand what and how is configured. 
This is currently not easy because of how the configuration is generated through the Java code.
In addition to that, we found adding Kafka client configuration updates to the `Kafka*Config` files of the Java modules in the [Strimzi Client Examples repo](https://github.com/strimzi/client-examples) to be a little more verbose and messier than necessary.
The current method requires creating a field, getter, setter, and hard coded String per new Kafka client config field. 
This method does not scale well and makes for a longer and messier class. 
So for any users who want to write their own Kafka client in Java, this is neither a good inspiration nor good code to copy.

## Proposal

We could greatly reduce the complexity and size of the class if we standardize the naming scheme of environment variables used to configure the clients of the [Strimzi Client Examples repo](https://github.com/strimzi/client-examples)

We could standardize the env vars used to configure the Kafka client properties in the following manner:
```
KAFKA_<NAME_OF_KAFKA_PROPERTY_DELIMITED_BY_UNDERSCORES>
```
For example:
```
KAFKA_BOOTSTRAP_SERVERS for `bootstrap.servers` property
KAFKA_GROUP_ID for `group.id` property
…
```
Then we could create a properties file by looping through the environment variables and translating them into a properties configuration like so:
```
Properties prop = new Properties();
HashMap<String, String> envVars = System.getEnv();
for (Map.Entry<String, Object> entry : map.entrySet()) {
    String key = convertEnvVarToPropertyKey(entry.getKey());
    String value = entry.getValue();
    prop.put(key, value);
}
```
This removes the need for fields, getters, setters, and hard coded Strings for specific Kafka client properties.
Note that the configuration options which cannot be directly configured as Kafka configuration properties would be configured with environment variables starting with a `STRIMZI_` prefix.
These would be handled directly in the Java code. 
This change would include configuration options for things like tracing initialization (interceptors can be set through `KAFKA_` variables, but the tracer would still need to be initialized).
For example:
```
STRIMZI_TOPIC for topic configuration
STRIMZI_TRACING_SYSTEM for tracingSystem configuration
...
```
However, other functionality that does not cover the basic example use-case like blocking producer or transactions support used for the Strimzi system tests in the past would be removed.
Thanks to these changes, the majority of the configuration would take place in the YAML deployment files, be easier to read for the users, and be easier to _translate_ to other clients.

## Affected projects

The [Strimzi Client Examples repo](https://github.com/strimzi/client-examples)

## Rejected alternatives

Passing the configuration as a single properties file was considered. 
It would make the configuration more readable as there will be no transformation to the property names as we would do with the environment variables. 
The properties file can be passed from Config Map as a volume or as an environment variable (either from the ConfigMap or have it defined in the YAML directly). 
However, this seems to be less _Kubernetes native_ than using the environmental variables. So we rejected this alternative.


