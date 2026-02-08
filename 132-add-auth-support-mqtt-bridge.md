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

## Technical implementation

### Configuration

We will begin by adding new configuration options to the MQTT Bridge. These options will allow users to enable secure MQTT connections and specify the necessary SSL/TLS settings. It will look something like this:

```application.properties
mqtt.server.tls.port=8883
mqtt.server.ssl.enabled=true
mqtt.server.ssl.keystore=path/to/keystore.jks
mqtt.server.ssl.keystore.password=your-keystore-password
mqtt.server.ssl.truststore=path/to/truststore.jks
mqtt.server.ssl.truststore.password=your-truststore-password
```

For two-way SSL/TLS authentication, we will also need to configure the MQTT Bridge to require client certificates. This can be done by adding the following configuration option:

```application.properties
mqtt.server.ssl.client-auth=true
```

Optionally, we can also support the configuration for specifying the protocols and cipher suites.

Afer this, we are going to create a new configuration wrapper class to load and manage these new config options, say `MqttSslConfig`. This class will then be part of the existing `MqttConfig`. It would look something like this:

```java
public class MqttConfig extends AbstractConfig {
    // existing config options...

    private final MqttSslConfig sslConfig;

    // constructor and methods...

    /**
     * Gets the SSL/TLS configuration for the MQTT Bridge.
     * @return the SSL/TLS configuration for the MQTT Bridge
     */
    public MqttSslConfig getSslConfig() {
        return sslConfig;
    }

    // other methods...
}
```

### Handling authentication in the server

To handle the authentication logic, we will need to add a new handler to the pipeline of the server. This will require us to modify the existing `MqttServerInitializer`class to include the new handler. The final look of the `MqttServerInitializer` class will be something like this:

```java
public class MqttServerInitializer extends ChannelInitializer<SocketChannel> {
    private final MqttServerHandler mqttServerHandler;
    private final int decoderMaxBytesInMessage;
    private final MqttSslConfig sslConfig;
    
    // add SSL config to the constructor
    public MqttServerInitializer(..., MqttSslConfig sslConfig) {
        // existing initialization logic...
        this.sslConfig = sslConfig;
    }

    @Override
    protected void initChannel(SocketChannel ch) {

        if (sslConfig.isEnabled()) {
            SslContext sslContext = // logic to create SSL context using the provided keystore and truststore

            ch.pipeline().addLast("ssl", sslContext.newHandler(ch.alloc()));
        }

        ch.pipeline().addLast("decoder", new MqttDecoder(decoderMaxBytesInMessage));
        ch.pipeline().addLast("encoder", MqttEncoder.INSTANCE);
        ch.pipeline().addLast("handler", this.mqttServerHandler);
    }
}
``` 

Thankfully, Netty provides APIs to easily and SSL/TLS support, so we can leverage those APIs to implement the authentication mechanism without having to implement the SSL/TLS logic from scratch. 

We will introduce a new component to encapsulate the logic for the TLS/SSL handshake and the client certificate auth before adding the SSL handler to the pipeline. This component will look something like this:

```java
public class MqttSslAuthManager {
    private final MqttSslConfig sslConfig;

    public MqttSslAuthManager(MqttSslConfig sslConfig) {
        this.sslConfig = sslConfig;
    }

    /**
     * Creates an SSL context based on the provided SSL configuration.
     */
    public SslContext createSslContext() {
        return SslContextBuilder.forServer(createKeyManagerFactory())
        .trustManager(createTrustManagerFactory())
        .clientAuth(sslConfig.isClientAuth() ? ClientAuth.REQUIRE : ClientAuth.NONE)
        .build();
    }

    /**
     * Creates the KeyManagerFactory based on the provided keystore configuration.
     */
    private KeyManagerFactory createKeyManagerFactory() {
        // logic to create KeyManagerFactory using the provided keystore configuration
    }

    /**
     * Creates the TrustManagerFactory based on the provided truststore configuration.
     */
    private TrustManagerFactory createTrustManagerFactory() {
        // logic to create TrustManagerFactory using the provided truststore configuration
    }
}
```

Our `MqttServerInitializer` will then use this `MqttSslAuthManager` to create the SSL context and add the SSL handler to the pipeline. It would look something like this:

```java
 @Override
    protected void initChannel(SocketChannel ch) {
        if (sslConfig.isEnabled()) {
            MqttSslAuthManager sslAuthManager = new MqttSslAuthManager(sslConfig);
            SslContext sslContext = sslAuthManager.createSslContext();

            ch.pipeline().addLast("ssl", sslContext.newHandler(ch.alloc()));
        }

       // existing pipeline initialization logic...
    }
```

### Handling authorization

A very common way to implement authorization in MQTT is to make use of ACLs(Access Control Lists). Our MQTT Bridge is not a full MQTT broker and it's stateless, meaning that we do not persist any client session information, only the mapping rules. 

I will submit a separate proposal to implement the authorization. This will make this proposal more focused and easier to review. I need a clear understanding ot the authorization mechanism alternatives before I can propose a specific implementation for the authorization support in the MQTT Bridge.If it's okay to the Strimzi maintainers, I will submit the authorization proposal after we have implemented the authentication support in this proposal.

## Testing

We will need to implement both unit and integration tests to verify the functionality of the authentication support in the MQTT Bridge.

We can make use of the Netty `EmbeddedChannel` to test the authentication logic in isolation. This will allow us to simulate MQTT client connections and verify that the authentication mechanism is working as expected.

## Affected/not affected projects

This proposal will primarily affect the MQTT Bridge component of Strimzi.

## Compatibility

We are currently using MQTT version 3.1.1. In future, if we consider to upgrade to MQTT version 5.0, which has introduced new features to address authentication, we will need to ensure that this proposal implements authentication and authorization support in a way that is compatible with both MQTT 3.1.1 and MQTT 5.0.

## Rejected alternatives


