# Default to _Restricted_ Pod Security Profile

Kubernetes supports configuration of a Security Context (SC) which can limit what the applications running on top of it are or are not allowed to do.
The SC can be configured on the Pod level (Pod Security Context) or on the container level.
When configured on the Pod level, it applies to all containers in given pod.
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

The detailed description of the profiles including of the list of allowed and not-allowed configurations for each profile is available in [Kubernetes documentation](https://kubernetes.io/docs/concepts/security/pod-security-standards/).

Together with the profiles, Kubernetes also introduced a Pod Security Admission plugin to enforce the security standards.
From Kubernetes 1.23, these features moved to Beta and are enabled by default.
By default it does not enforce any of the profiles.
Enforcing a specific profile can be configured either in the Kubernetes API server configuration or using namespace labels.
Once enabled, it will not allow creation of Pods which do not have the security context configured as described by the enforced profile.
You can learn more about the Pod Security Admission in the [Kubernetes documentation](https://kubernetes.io/docs/concepts/security/pod-security-admission/).

## Current situation

Currently, Strimzi provides a minimal configuration of the security context by default.
When running on Kubernetes and using persistent storage, we configure the group which should be used for the storage.
That is the only automatic configuration Strimzi makes.

Users can use the `.template` properties in the Strimzi custom resources to configure their own security context.
That allows them to customize it to match their own requirements and policies.
On OpenShift, the security context is automatically injected into the Pods when they are created based on the OpenShift SCC policies.
This typically involves dropping some capabilities, using some random unprivileged user ID etc.
In some cases, our users might also use similar systems to inject the security context using admission controllers and similar tools.

The _baseline_ profile basically means that you just use the default settings without requesting any additional privileges or capabilities.
So without any of the additional tooling mentioned above, Strimzi operators and the pods created by them today match the _baseline_ profile.
But if you would try to run Strimzi in a cluster where the _restricted_ profile is enforced, it would not work out of the box.

Apart from the Kaniko builder used for the Kafka Connect build, all our components are able to run under the _restricted_ profile.
But the operator doesn't configure the appropriate security context and the admission plugin will reject them.
Users have to _manually_ add the matching security context configuration into the `.template` sections of the custom resources and into the operator deployments to allow the pods to be created under the _restricted_ profile.
To match the _restricted_ profile, users would need to add the following security context:

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
So the security context needs to be basically configured for every container (including init containers, sidecars etc.).
So just for a single Kafka cluster, it might need to be configured on 9 different places of the Kafka CR (ZooKeeper, Kafka broker, Kafka init-container, Topic Operator, User Operator, TLS sidecar, Kafka Exporter, Cruise Control, another TLS sidecar).

## Proposal

This proposal suggests making Strimzi more secure by configuring the security context to match the _restricted_ profile by default.
This requires changes to the Strimzi Cluster Operator which will set the security context.
No changes should be required to the operands.

The security context will be set only when running on Kubernetes clusters.
When OpenShift is detected, the security context will not be set and OpenShift will be able to inject its own security context (unfortunately, the _restricted_ Kubernetes profile is currently not fully compatible with the OpenShift restricted SCC profile).
That should make sure that Strimzi still runs on OpenShift out of the box without any special changes.

The default security context will be set only if the user didn't specify custom security context in the custom resource.
In that case, the user's configuration will be used.
That means that users will be able to also disable the new default settings completely just by setting `securityContext: {}`.
(However, this would again need to be configured for each container separately).

As mentioned above, Kaniko does not support running under the _restricted_ profile since it currently needs to run under the `root` user.
This is because in order to build the new container image it requires to run under the _baseline_ profile.
Due to this, the operator will not set the restricted security context to the Kaniko pods and they will keep running under the _baseline_ profile.
Kafka Connect Build - which requires Kaniko - is an optional part of Strimzi.
Users who want to enforce the _restricted_ profile can build the Connect container image with additional plugins using a Dockerfile instead.

The default security context will be also set in the operator installation files for all our operators (and other components).

### Challenges

#### Custom security tooling

As mentioned above, the OpenShift _restricted_ SCC policy is not compatible with the Kubernetes _restricted_ profile.
To work around this, we will not configure the default security policy on OpenShift.
However, users might use similar tooling also in Kubernetes clusters which we cannot easily detect.
If such tooling conflicts with the _restricted_ profile, such users would need to configure the security context in the custom resources.
We have no way of knowing how many users use such tooling. 

#### Installation files not working on OpenShift

When we add the _restricted_ security context to the installation files of the operators, it would be possible to install them out of the box only on Kubernetes.
OpenShift users would need to edit them first to remove / update the security context.
This applies only to the YAML files.
OperatorHub has separate sources for OpenShift and Kubernetes.
And in the Helm Chart, there is an option to easily change the security context without editing the Helm Chart.

### Benefits

This proposal starts with focus on Pod Security Standards.
However, tightening the security is in general useful even when the security profiles are not restricted.
It would improve the security of our pods in all environments.

### Feature Gate

Due to the challenges described above, we will use Feature Gate to introduce this.
With this feature gate disabled, the _restricted_ security context will not be set.
With it enabled, the _restricted_ security context will be used by default.
The security context will be added to the installation files only when the feature gate moves to beta and is enabled by default.
The following table shows the expected graduation of the feature gate:

| Phase | Strimzi versions       | Default state                                                        |
|:------|:-----------------------|:---------------------------------------------------------------------|
| Alpha | 0.28, 0.29             | Disabled by default, security context not set in installation files  |
| Beta  | 0.30, 0.31             | Enabled by default and security context is set in installation files |
| GA    | 0.32 and newer         | Enabled by default (without possibility to disable it)               |

Just to note:
* Even after this feature gate graduates to GA, users will be still able to set the security context using the custom resources.
  So it will not be permanent, it will be just harder to override the defaults
* On OpenShift, the new default values will not be set even after the graduation of the feature gate.

## Rejected alternatives

### Permanent switch instead of Feature Gate

Instead of using a Feature Gate to introduce this feature, we can use a regular configuration option to enable or disable this feature.
This would work similarly to how the flag for disabling the Network Polices was implemented.
We would only need to decide what should be the default value (enabled or disabled by default).

The disadvantage of this would be that we would have to maintain this permanently in the code and not just while the feature gate is used.
(However we might need to do that for OpenShift anyway, this would be just second flag to check)

### Two sets of installation files

We could provide two sets of installation files.
One with the security context configured and one without it.
Users would be able to easily choose which one to use.
However, this could be also confusing for them if they do not know which set of files to use.
And it would make the docs and maintenance of the installation files more complicated.

## Compatibility

It will be still possible to use Strimzi as it is used today even after implementing this proposal.
But some users might need to change their configuration to maintain the same behavior.
The specific challenges are described in the _Proposal_ section.