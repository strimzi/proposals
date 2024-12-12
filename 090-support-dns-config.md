<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Support DNS configuration

This proposal describes the motivation for adding optional nameserver configuration to Strimzi custom resources.

## Current situation

Nameserver configuration for Strimzi custom resources falls back on defaults that apply to the underlying Kubernetes/Openshift Pod resources. 
Defaults for these underlying resources are defined at the level of a Pod. 
The defaults are a dnsPolicy of ClusterFirst without any further specified dnsConfig.
These defaults assure that domain names first undergo cluster domain resolution before checking external sources.
For domains that don't match the cluster suffix, queries are forwarded to the upstream DNS servers defined by the node or CoreDNS configuration.
Strimzi resources thereby don't currently expose configuration that allows customization of name resolution on a resource level.

## Motivation

Certain configuration of Kubernetes/Openshift clusters require Strimzi resources that are deployed within different namespaces/projects to forward DNS queries for external resources to different nameservers than the one that is configured centralized in CoreDNS or on the node hosting the resource.
For example a dev and stage environment/namespace/project hosted in the same Kubernetes/Openshift cluster might need to target seperate target clusters for similar subdomains but are not able to override the nameserver used as this is controlled centrally for both namespaces/projects. 
By supporting the customization of nameserver configuration for Strimzi resources on the level of the resource this will allow to target specific nameservers. Thereby, it allows scenarios where resources in different namespace/projects, allthough sharing the source cluster network, to target matching target environments which are on segregated networks.

## Proposal

By appending Strimzi's common PodTemplate in the Strimzi api with additional properties dnsPolicy (to convey ClusterFirst, ClusterfirstWithHostNet, Default and None as name resolution policies) and dnsConfig (to specify nameservers, conditional subdomains to search for and options to decide resolution behaviors), the system configuration file /etc/recolv.conf for Strimzi resources can be controlled, thereby deciding how domain name resolution should be performed on a resource level. 
Strimzi's CRDs generated from from PodTemplate will thereby allow for user input for name resolution customization.
By appending these dnsConfig and dnsPolicy properties to WorkloadUtils in the Strimzi cluster operator for PodBuilder and PodTemplateSpecBuilder so that these will propagate to Pods controlled by the operator, the propagation from Strimzi CRD to the Pod resource will be accounted for.
Provisioned Pods will thereby utilize name resolution configuration based on the user input if specified and fall back to defaults, similar to the current situation, in case no dnsConfig or dnsPolicy is specified by the user. 


## Affected/not affected projects

io.strimzi.api for the PodTemplate.
io.strimzi.cluster-operator for resources that utilize PodTemplate: kafka, kafkabridge, kafkaconnect, kafkamirromaker, kafkamirrormaker2, kafkanodepool.

## Compatibility

Non-specified dnsConfig and dnsPolicy will not propagate these properties to the Pod resources and thereby comply to the current situation. 

## Rejected alternatives

-
