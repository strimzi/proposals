# Add SSL/TLS support to MQTT Bridge

This proposal is to add support for SSL/TLS to the MQTT Bridge component of Strimzi.
This would allow MQTT clients to connect to the MQTT Bridge over an encrypted MQTT over TLS connection.

## Current situation

No SSL/TLS support is currently available in the MQTT Bridge.
This means that MQTT clients connect to the MQTT Bridge over an unencrypted connection.
This can expose MQTT traffic to interception or tampering when clients connect over untrusted networks.

## Motivation

The motivation for adding SSL/TLS support to the MQTT Bridge is to enhance the transport security of MQTT Bridge deployments.
Since a common use case for the MQTT Bridge is to connect IoT devices to Kafka, MQTT traffic might pass through networks where encryption is required.
Supporting MQTT over TLS allows users to protect data in transit without changing the MQTT protocol version or the bridge mapping behavior.

## Proposal

This proposal adds support for MQTT over TLS by allowing users to configure a server certificate and private key for the MQTT listener.
When SSL/TLS is configured, the MQTT Bridge will add a Netty SSL handler before the MQTT decoder and encoder in the server pipeline.

This proposal is limited to SSL/TLS encryption for MQTT client connections.
It does not change MQTT topic handling, Kafka mapping behavior, or MQTT client session semantics.

## Technical implementation

### Configuration

We will begin by adding new configuration options to the MQTT Bridge.
These options will allow users to enable secure MQTT connections and specify the necessary SSL/TLS configurations.
We will not add a separate TLS port configuration.
The MQTT Bridge will continue to use a single MQTT listener, and MQTT over TLS will use the existing MQTT server port configuration with the standard MQTT over TLS port, 8883.
It will look something like this:

```application.properties
mqtt.server.port=8883
mqtt.server.ssl.certificate.location=path/to/server-cert.pem
mqtt.server.ssl.key.location=path/to/server-key.pem
mqtt.server.ssl.certificate=--BEGIN CERTIFICATE--\n...\n--END CERTIFICATE--
mqtt.server.ssl.key=--BEGIN PRIVATE KEY--\n...\n--END PRIVATE KEY--
```

> Note: All configuration options that support both inline and file-based values are mutually exclusive. 
Inline values take precedence over file-based values.

Optionally, we can also support the configuration for specifying the protocols and cipher suites:

```application.properties
mqtt.server.ssl.enabled.protocols=TLSv1.2,TLSv1.3
mqtt.server.ssl.enabled.ciphers=TLS_AES_128_GCM_SHA256,TLS_AES_256_GCM_SHA384
```

> If these options are not specified, the MQTT Bridge will use the default SSL/TLS configurations provided by the Java runtime.

After this, we are going to create a new configuration wrapper class to load and manage these new config options, say `MqttSslConfig`.
This class will then be part of the existing `MqttConfig`.
The SSL/TLS state will be derived from the configured MQTT server port rather than from a separate enable flag; for example, `MqttSslConfig.isEnabled()` can return true when the configured port is 8883.
It would look something like this:

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

### Handling SSL/TLS in the server

To handle SSL/TLS connections, we will need to add the Netty SSL handler to the pipeline of the server.
This will require us to modify the existing `MqttServerInitializer` class to include the new handler.
The final look of the `MqttServerInitializer` class will be something like this:

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
            SslContext sslContext = // logic to create SSL context using the provided SSL/TLS configurations

            ch.pipeline().addLast("ssl", sslContext.newHandler(ch.alloc()));
        }

        ch.pipeline().addLast("decoder", new MqttDecoder(decoderMaxBytesInMessage));
        ch.pipeline().addLast("encoder", MqttEncoder.INSTANCE);
        ch.pipeline().addLast("handler", this.mqttServerHandler);
    }
}
``` 

Thankfully, Netty provides APIs to easily add SSL/TLS support, so we can leverage those APIs without having to implement the SSL/TLS logic from scratch. 

We will introduce a new component to encapsulate the logic for creating the Netty `SslContext` before adding the SSL handler to the pipeline.
This component will look something like this:

```java
public class MqttSslContextProvider {
    private final MqttSslConfig sslConfig;

    public MqttSslContextProvider(MqttSslConfig sslConfig) {
        this.sslConfig = sslConfig;
    }

    /**
     * Creates an SSL context based on the provided SSL configuration.
     */
    public SslContext createSslContext() {
        return SslContextBuilder.forServer(createKeyManagerFactory())
        .build();
    }

    /**
     * Creates the KeyManagerFactory based on the provided keystore configuration.
     */
    private KeyManagerFactory createKeyManagerFactory() {
        // logic to create KeyManagerFactory using the provided keystore configuration
    }
}
```

Our `MqttServerInitializer` will then use this `MqttSslContextProvider` to create the SSL context and add the SSL handler to the pipeline.
It would look something like this:

```java
 @Override
    protected void initChannel(SocketChannel ch) {
        if (sslConfig.isEnabled()) {
            MqttSslContextProvider sslContextProvider = new MqttSslContextProvider(sslConfig);
            SslContext sslContext = sslContextProvider.createSslContext();

            ch.pipeline().addLast("ssl", sslContext.newHandler(ch.alloc()));
        }

       // existing pipeline initialization logic...
    }
```

## Testing

We will need to implement both unit and integration tests to verify the functionality of SSL/TLS support in the MQTT Bridge.

We can make use of the Netty `EmbeddedChannel` to test the server pipeline initialization in isolation.
This will allow us to verify that the SSL handler is added before the MQTT decoder when SSL/TLS is configured.
Integration tests should verify that MQTT clients can connect successfully over TLS using the configured server certificate and key.

## Affected/not affected projects

This proposal will primarily affect the MQTT Bridge component of Strimzi.

## Compatibility

We are currently using MQTT version 3.1.1.
SSL/TLS is transport-level security and does not depend on MQTT protocol features, so this proposal should be compatible with both MQTT 3.1.1 and a future MQTT 5.0 upgrade.

## Rejected alternatives

Handling authentication and authorization is out of the scope for this proposal. They should be addressed separately. 