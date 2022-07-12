# Pluggable Pod Security Profile

Kubernetes supports configuration of a Security Context (SC) which can limit what the applications running on top of it are or are not allowed to do.
The SC can be configured on the Pod level (Pod Security Context) or on the container level.
When configured on the Pod level, it applies to all the containers in a given Pod.
The security context allows you to configure different security related aspects of the containers.
For example:
* Users and group used for running the container
* Capabilities available to the applications
* _seccomp_ profiles
* SELinux options

The latest versions of Kubernetes introduced Pod Security Standards.
These standards can be used to enforce the security context configuration.
Pod Security Standards define three profiles to specify the security context allowed for applications:
* _Privileged_ for applications needing the widest possible permissions
* _Baseline_ which prevents some common security issues, but is minimally restrictive
* _Restricted_ which heavily restricts the application permissions and follows the best hardening practices

For a detailed description of the profiles, including what's allowed and disallowed with each profile, see [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) in the Kubernetes documentation.

Together with the profiles, Kubernetes also introduced a Pod Security Admission plugin to enforce the security standards.
From Kubernetes 1.23, these features moved to Beta and are enabled by default.
By default, it does not enforce any of the profiles.
Enforcing a specific profile can be configured either in the Kubernetes API server configuration or using namespace labels.
Once enabled, Kubernetes does not allow creation of Pods which do not have the security context configured as described by the enforced profile.
You can learn more about the Pod Security Admission in the [Kubernetes documentation](https://kubernetes.io/docs/concepts/security/pod-security-admission/).

## Current situation

Currently, Strimzi provides a minimal configuration of the security context by default.
When running on Kubernetes and using persistent storage, we configure the group which should be used for the storage.
That is the only automatic configuration Strimzi makes.

Users can use the `.template` properties in the Strimzi custom resources to configure their own security context.
This allows them to customize a security context that matches their own requirements and policies.
On OpenShift, the security context is automatically injected into the Pods when they are created based on OpenShift Security Context Constraints (SCC) policies.
This typically involves dropping some capabilities, using some random unprivileged user ID etc.
In some cases, Strimzi users might also use similar systems to inject the security context using admission controllers and similar tools.

The _baseline_ profile basically means that you just use the default settings without requesting any additional privileges or capabilities.
So without any of the additional tooling mentioned previously, Strimzi operators and the Pods created by them at present match the _baseline_ profile.
If you try to run Strimzi in a cluster where the _restricted_ profile is enforced, it does not work out of the box.

Apart from the Kaniko builder used for the Kafka Connect build (Kaniko currently requires running as root), all our components are able to run under the _restricted_ profile.
But the operator doesn't configure the appropriate security context and the admission plugin will reject them.
Users have to _manually_ add the matching security context configuration into the `.template` sections of the custom resources and into the operator deployments to allow the pods to be created under the _restricted_ profile.
To match the _restricted_ profile, users must configure the following security context:

```yaml
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              # The restricted allows the use of the NET_BIND_SERVICE capability.
              # But Strimzi does not require it for anything, so we do not include it here.
              drop:
                - ALL
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
```

While users can configure the security context, it is not user-friendly.
The Pod security context does not cover all the different options required by the _restricted_ profile.
So the security context has to be configured for every container (including init containers, sidecars etc.).
So a single Kafka cluster might need configuration in 8 different places in the `Kafka` custom resource: ZooKeeper, Kafka broker, Kafka init-container, Topic Operator, User Operator, TLS sidecar, Kafka Exporter, and Cruise Control.

## Proposal

This proposal suggests adding a pluggable mechanism for users to provide their own plugins to configure the Pod and container security contexts of all Strimzi components.
Out of the box, Strimzi will provide implementation supporting two profiles:
* The _baseline_ provider which will mirror the current behavior and can be used when running under the _baseline_ Kubernetes profile
* The _restricted_ provider which will allow to run under the _restricted_ Kubernetes profile

The _baseline_ profile will be used by default.
So users will not see any change by default.
But if they decide to, they will be abe to switch to the _restricted_ profile.

In addition, any user who will need some special configuration, will be able to provide a custom provider plugin.
This will allow them to use their own configurations without having to specify them in the `.template` sections in all custom resources.
This might be useful for users who use some custom security tooling and might have their own requirement for the security configuration.
But regular users will not need to write any custom plugins.

The pluggable mechanism should also make it easy to add additional implementations to Strimzi in the future.

### Plugin implementation

#### Interface

A new interface `PodSecurityProvider` will be added to the `api` module.
All provider plugins will implement this interface.
The reason for having this interface in the `api` module is that this module is distributed though Maven repositories.
Users will be able to write their implementations by using the `api` module as dependency.

The interface will have methods for providing security contexts for all Pods created by Strimzi, and all containers.
The methods providing the security context will get as a parameter an object of type `PodSecurityProviderContext`.
This will be an object created by Strimzi which will encapsulate the different infromation which strimzi will provide to the Provider plugin to make its decisions when generating the Kubernetes security context.
The `PodSecurityProviderContext` will be defines as another interface in the `api` module:

```java
interface PodSecurityProviderContext {
    Storage storage();
    PodSecurityContext userSuppliedContext();
}
```

This encapsulation should make it easier to add additional information in the future without changing the signature of the provider methods but only by adding new methods to the Strimzi provided context.

The provider methods will have the default implementation which will always just return the user-specified context.
For example (just few methods are listed in this example):

```java
public interface PodSecurityProvider {
    // ...

    default PodSecurityContext kafkaPodSecurityContext(PodSecurityProviderContext context) {
        return context.userSuppliedContext();
    }

    default SecurityContext kafkaContainerSecurityContext(PodSecurityProviderContext context) {
        return context.userSuppliedContext();
    }

    default SecurityContext kafkaInitContainerSecurityContext(PodSecurityProviderContext context) {
        return context.userSuppliedContext();
    }

    // ...
}
```

The interface will also have a `configure` method for configuring the provider.
The `configure` method will get as a parameter an object describing the platform (Kubernetes version, features / APIs) which can be taken into account when creating the security context.
This method will not have any default implementation.

```java
public interface PodSecurityProvider {
    // ...
    void configure(PlatformFeatures platformFeatures);
    // ...
}
```

`PlatformFeatures` in this method is another new interface which provides some of the methods provided by the `PlatformFeaturesAvailability` class which is part of the `operator-common` module`
It uses the features which might be relevant for generating the security context:

```java
public interface PlatformFeatures {
    KubernetesVersion getKubernetesVersion();
    boolean isOpenshift();
}
```

The `configure(...)` method can in addition to the `PlatformFeatures` also for example use environment variables to configure the provider.

#### Implementations

We will provide two implementations to support the baseline (default) and restricted profiles:
* `BaselinePodSecurityProvider`
* `RestrictedPodSecurityProvider`

_(Strimzi does not require any privileged access, so there is no need to provide a build in profile for that. Any user running Strimzi under the privileged can use the baseline provider which will work without any issues.)_

These implementations will be also part of the `api` module so that users can just extend them instead of implementing the provider from scratch.

The `BaselinePodSecurityProvider` will implement the current behaviour:
* Return the user-supplied security context when specified
* When the user-supplied context is `null` and we are on OpenShift:
    * A `null` will be returned (OpenShift injects its own Security Context in that case) for all Pods and containers
* When the user-supplied context is `null` and we are not on OpenShift:
    * Pods with storage will get a Security Context specifying the file system group as `0`
    * All other pods will get `null` Security Context
    * All containers will get `null` Security Context

In the YAML format, the generated security context will look like this:
* No container security contexts will be set
* No pod security contexts will be set for Pods without persistent storage
* For Pods with persistent storage, following security context will be set:
  ```yaml
  securityContext:
    fsGroup: 0
  ```

The `RestrictedPodSecurityProvider` will implement the following behaviour:
* Return the user-supplied security context when specified
* When the user-supplied context is `null`:
    * Pods with storage will get a Security Context specifying the file system group as `0`
    * All other pods will get `null` Security Context
    * For the Kaniko container, an exception will be thrown since Kaniko cannot run under the Restricted profile
    * All other containers will get a Security Context matching the _restricted_ Kubernetes profile:
        * Disable privilage escalation
        * Run as non-root
        * Specify the Seccomp profile as `RuntimeDefault`
        * Drop all capabilities

In the YAML format, the generated security context will look like this:
* All containers (apart from Kaniko) will have the following security context:
  ```yaml
  securityContext:
    allowPrivilegeEscalation: false
    runAsNonRoot: true
    secCompProfile:
      type: RuntimeDefault
    capabilities:
      drop:
        - ALL
  ```
* No pod security contexts will be set for Pods without persistent storage
* For Pods with persistent storage, following security context will be set:
  ```yaml
  securityContext:
    fsGroup: 0
  ```

#### Loading the plugins

In the Cluster Operator, the plugins will be loaded using the Java Service Provider Interface which will be used to find the class configured by the user.
A `PodSecurityProviderFactory` class will be part of the `cluster-oprator` module.
It will have a static initialize method which will load the class based on the configuration, call its `configure(...)` method and store it in a static field for further use.
It will also have static method `getProvider()` which will be used by the model classes to get the provider instance.

### Configuration

The provider which should be used will be configured using the `STRIMZI_POD_SECURITY_PROVIDER_CLASS` environment variable.
The environment variable will not be configured by default in the installation files.
And when not configured, it will use the `BaselinePodSecurityProvider` by default.

When a user decides to configure the class, they can specify the full class name (including the package).
They can also use the following keywords:
* `baseline` for `BaselinePodSecurityProvider`
* `restricted` for `RestrictedPodSecurityProvider`

If the configured class is not found, an error will be thrown during the intialization and the operator will exit.

### Installation files

There will be no changes to the installation files.
If a user wants to use Strimzi under a _restricted_ security profile, they can edit the installation files to specify the security context.
In this way, the installation files will continue to work on current environments.
This includes OpenShift, whose _restricted_ SCC policy is not compatible with the Kubernetes _restricted_ profile.

In the future, depending on how many users use the _restricted_ security profile, we can consider adding it to the installation files and using `RestrictedPodSecurityProvider`  by default.

## Benefits of this proposal

This proposal will make it easier for the users to run under the different Kubernetes security profiles.
It will also make it much easier to customize the security context configuration if needed by providing custom plugins.

## Rejected alternatives

### Using a separate module for the interface and its implementations

I considered creating a separate Java module for the plugin interface and its implementations instead of having them in the `api` module.
But it seemed unnecessary without giving some additional advantage and it would take the builds longer to run.

### Moving implementations to the Cluster Operator module

The implementations provided by Strimzi could also live in the `cluster-operator` module directly.
But in that case, users cannot just extend them, they would need to write their providers from scratch.

## Compatibility

The default behaviour will be exactly the same as before this change.
Only users who make use of the additional providers see some change.