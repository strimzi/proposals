# Support DNS configuration

This proposal describes the motivation for adding optional nameserver configuration to Strimzi custom resources.

## Current situation

Strimzi relies on the default DNS settings provided by Kubernetes or OpenShift pods.
By default, these pods use a DNS policy called `ClusterFirst`,  which prioritizes resolving names within the Kubernetes cluster. 
No additional DNS configuration is specified at the pod level.
These defaults resolve domain names within the cluster first, before checking external sources.
For domains that don't match the cluster suffix, queries are forwarded to the upstream DNS servers defined by the node or CoreDNS configuration.
Strimzi resources thereby don't currently expose configuration that allows customization of name resolution on a resource level.

## Motivation

Certain configuration of Kubernetes/Openshift clusters require Strimzi resources that are deployed within different namespaces/projects to forward DNS queries for external resources to different nameservers than the one that is configured centralized in CoreDNS or on the node hosting the resource.
For example a dev and stage environment/namespace/project hosted in the same Kubernetes/Openshift cluster might need to target seperate target clusters for similar subdomains but are not able to override the nameserver used as this is controlled centrally for both namespaces/projects. 
By supporting the customization of nameserver configuration for Strimzi resources on the level of the resource this will allow the targeting of specific nameservers. 
This supports scenarios where resources in different namespaces/projects—while sharing the source cluster network—can still target matching target environments on separate, segregated networks.

## Proposal

By adding `dnsPolicy` and `dnsConfig` properties to Strimzi's common `PodTemplate` in the Strimzi API, users can control domain name resolution for Strimzi resources.

- `dnsPolicy` can specify resolution policies like `ClusterFirst`, `ClusterFirstWithHostNet`, `Default`, and `None`.
- `dnsConfig` can allow users to define nameservers, search domains, and resolution options.

This provides more control over the `/etc/resolv.conf` configuration for Strimzi resources, allowing domain name resolution to be customized at the resource level.
Strimzi's CRDs generated from from PodTemplate will thereby allow for user input for name resolution customization.
By appending these dnsConfig and dnsPolicy properties to WorkloadUtils in the Strimzi cluster operator for PodBuilder and PodTemplateSpecBuilder so that these will propagate to Pods controlled by the operator, the propagation from Strimzi CRD to the Pod resource will be accounted for.
Provisioned Pods will thereby utilize name resolution configuration based on the user input if specified and fall back to defaults, similar to the current situation, in case no dnsConfig or dnsPolicy is specified by the user. 

An example configuration could look as follows:

```yaml
kind: KafkaMirrorMaker2
spec:
  template:
    pod:
      dnsPolicy: "None"
      dnsConfig:
        nameservers:
          - 192.0.2.1 
        searches:
          - ns1.svc.cluster-domain.example
          - my.dns.search.suffix
        options:
          - name: ndots
            value: "2"
          - name: edns0
```
When the Pod above is created, the container gets the following contents in its /etc/resolv.conf file:

```
nameserver 192.0.2.1
search ns1.svc.cluster-domain.example my.dns.search.suffix
options ndots:2 edns0
```

## Affected/not affected projects

Affected:
- `api` module in the operator project
- `cluster-operator` module in the operator project

## Compatibility

Non-specified dnsConfig and dnsPolicy will not propagate these properties to the Pod resources and thereby comply to the current situation. 

## Rejected alternatives

-
