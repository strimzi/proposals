# Add support for user namespace Pods

This proposal adds Strimzi support for using user (Linux / container) namespaces with Strimzi Pods.
User namespaces allow users to isolate the user running inside the container from the one in the host and help to improve security.
User namespaces are enabled in Kubernetes by default from version 1.33 (requires support in the underlying container infrastructure).

_Note: Linux user namespaces are not related to Kubernetes namespaces in any way._

## Motivation

User namespaces have various advantages.
For Strimzi, the security benefits are the most relevant aspect.
User namespaces help to isolate the processes between different Pods and between the Pod and the host (Kubernetes worker node).
This helps mitigate the impact of CVEs in the infrastructure as well as in the operands.

More information about user namespaces can be found in the Kubernetes documentation:
* [Use a User Namespace with a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/)
* [Kubernetes v1.33: User Namespaces enabled by default!](https://kubernetes.io/blog/2025/04/25/userns-enabled-by-default/)

Adding support for user namespaces would allow Strimzi users to use this feature and improve their security.

## Proposal

In Kubernetes, the user namespaces are configured using the `hostUsers` flag in the [Pod specification](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.35/#podspec-v1-core).
By default, it is set to true.
That means that the Pod shares the user namespace with the underlying host.
When set to `false`, a new user namespace will be created for each Pod.

This proposal suggests adding support for the user namespaces in Strimzi Pods in two different ways:
* Individually on a per-pod basis through the Pod template 
* Automatically using the Pod Security Provider

### Configuring `hostUsers` through the Pod template

The `hostUsers` field will be added to the Pod template:
```yaml
kind: Kafka
spec:
  kafka:
    # ...
    template:
      pod:
        hostUsers: false
```

That will allow users to configure the flag and the user namespaces on a per-pod basis.

### Configuring `hostUsers` through the Pod Security Providers

However, configuring the flag individually for each Pod is not practical when you want it configured for all operands.
So we will also extend the Pod Security Provider pluggable interface to allow configuring the flag automatically for all Strimzi-managed Pods.

The `PodSecurityProvider` will be extended with new methods for configuring the `hostUsers` flag.
The new methods will return a `Boolean` and will be added for every Strimzi-managed Pod:
* `kafkaHostUsers` for Kafka nodes
* `kafkaConnectHostUsers` for Connect and MirrorMaker 2 nodes
* `bridgeHostUsers` for the HTTP Bridge
* `entityOperatorHostUsers` for the Entity Operator
* `kafkaExporterHostUsers` for the Kafka Exporter
* `cruiseControlHostUsers` for the Cruise Control

These methods will take a single parameter of the existing type `PodSecurityProviderContext`.
This type will be extended to include, in addition to the storage and Pod security context configuration, the `hostUsers` value from the Pod template.
By extending the existing class, we will allow the Pod Security Providers to configure the Pod security depending on the security context and storage configuration.

For backwards compatibility with any existing Pod Security Provider implementations, the `PodSecurityProvider` will have default implementations of the new methods.
They will all return the value configured in the Pod template or `null`.
When `null` is used to configure the `hostUsers` flag, the Kubernetes default will be used (currently `true`).

The following example shows the default implementation of the `kafkaHostUsers` method:

```java
    default Boolean kafkaHostUsers(PodSecurityProviderContext context) {
        return hostUsersOrNull(context);
    }

    private Boolean hostUsersOrNull(PodSecurityProviderContext context) {
        if (context != null)   {
            // Returns the value from the Pod template
            return context.userSuppliedHostUsers();
        } else {
            return null;
        }
    }
```

The new methods will be called from the Cluster Operator when creating the Pod definitions, similarly to how the Pod Security Context configuration is specified.

#### Built-in Pod Security Providers

The existing built-in `BaselinePodSecurityProvider` and `RestrictedSecurityProvider` implementations will remain unchanged and will use the default methods.
While it would make sense to use the user namespaces out of the box when `RestrictedSecurityProvider` is used, we cannot do that for backwards compatibility reasons (it would break the operands for users without supported user namespaces in their infrastructure or with some custom changes).
So for the time being, Strimzi users would need to implement their own Pod Security Providers to enable user namespaces for Strimzi Pods automatically without configuring it in the Pod template sections of the Strimzi custom resources.

_Enabling the user namespaces for Strimzi Pods in the `RestrictedSecurityProvider` by default might be revisited later once the feature is supported by most of our users and once we drop support for older Kubernetes versions._

## Affected projects

This proposal affects the Strimzi Cluster Operator and the Strimzi Pod Security Provider APIs.

## Backwards compatibility

This proposal is fully backwards compatible.
Any changes and new features it introduces are only optional and need to be activated by the user in the Strimzi custom resources or in their Pod Security Provider.

## Rejected alternatives

There are currently no rejected alternatives.
