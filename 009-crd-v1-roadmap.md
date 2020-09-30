

# Roadmap to using the CRD v1 API

This describes (roughly) the steps Strimzi users will need to take to get to using v1beta2 Strimzi APIs which, in turn, are using the v1 CRD API. 
As such it talks about future versions of Strimzi. We don't yet know for sure what versions the features described would land in, so we'll use the terms
Strimzi X and Strimzi Z, where X and Z are unknown version numbers > 0.20.0 with X < Z. We've used Z rather than Y because it's not a given that Z is the version following X.
The requirement is that what we’re calling Strimzi Z is released around the same time that Kubernetes 1.21 is.

## Current situation

Currently Strimzi defines numerous `CustomResourceDefinitions` using the `apiextensions.k8s.io/v1beta1` API. Those CRDs define the "Strimzi CRs", as `v1alpha1` and `v1beta1` APIS in the `kafka.strimzi.io` API group.

Kubernetes 1.22 is expected to have removed CRD `apiextensions.k8s.io/v1beta1`. Its replacement, `apiextensions.k8s.io/v1`, has been available since Kubernetes 1.16.


## Motivation

To work on Kubernetes 1.22 Strimzi will need to use the `apiextensions.k8s.io/v1` CRD API. We can't have a single set of CRDs which support both Kubernetes 1.22 or later and also 1.15 or earlier. Furthermore, using `apiextensions.k8s.io/v1` requires that the valiation schemas in the CRD are all [structural schemas](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema).

## Proposal

### Strimzi 0.20.0

This will be the last version of Strimzi which supports Kubernetes 1.11-1.15. 

#### Users

Users should:

 * Rewrite their Kafka `spec.kafka.listeners` fields to use a list. 
 * Check they’re not using any “deprecated” fields in any of their resources.

### Strimzi X (approximately January 2021)

This release will be a gate release, meaning that all existing Strimzi users will have to pass through X in order to upgrade to Z.
This should work on Kubernetes 1.16-1.20. It can’t work on lower versions because it relies on having support for per-versions validation schemas in CRDs, a feature that wasn’t available before 1.16. It can’t work on higher versions because that don’t support `apiextensions.k8s.io/v1beta1` of the CRD API.

#### Development

We need to:

* Audit all existing Strimzi CRDs looking for things we need to change in order to have a structural schema. This is basically about checking that the `type` of each `property` in the schema is not going to need to change. Prime suspects include:
    
    - things which are declared as a `Map<String, Object>` in Java (hence an `object` in JSON Schema):

        * `Kafka.spec.kafka.jvmOptions`, `Kafka.spec.zookeeper.jvmOptions`, and `jvmOptions` in other CRDs. JVM options have to be specified in a certain order and the API presented by `jvmOptions` doesn't support this. 
        
        * Use of a `Map` in `metrics` is also worth reviewing. It ties the Strimzi API very strongly to the JMXExporter and often results in a large amount of JMXExporter configuration in the `Kafka` resource, which obscures the rest of the Kafka configuration.

        * Other uses of `Map<String, Object>` are more debatable. In particular `Kafka.spec.kafka.configs` (and similar for ZK) doesn't seem like a bad representation for what is, ultimately, a properties file. For `labels` and `annotations`, the Kubernetes API sets a precedent of using `object`, which Fabric8 represents as a `Map`, so annything else would be unintuitive.
    
    - Things which are a declared with type `boolean` should also be looked at with an eye to how future proof they are. A `string` can be constrained to having two values which can later be relaxed to allow more values, but this is not possible with a `boolean`. 

    For each of these we should make the analogous change that we did for `listeners` in 0.20.0.
* Develop the `v1beta2` schema. This must be structural. This must be backward compatible with `v1beta1`, by which I mean any schema-valid `v1beta2` CR must be a schema-valid `v1beta1` CR (though not vice versa).
* The Strimzi X CRD will declare `v1alpha1` (served=true, storage=false), `v1beta1` (served=true, storage=false) and `v1beta2` (served=true, storage=true)
* The operator will access the CRs via the `v1beta2` path (it will be able to observe `v1beta1` resources because the `None` conversion just changes the `apiVersion`). 
* Develop a tool which:
  * Can rewrite a `v1beta1` resource to an equivalent `v1beta2` resource, allowing users to easily modify their resources (e.g. in git).
  * Can apply/edit CRs in Kubernetes likewise.
  * Can update the CRD `spec.versions` to change the served field.
  * Can update the CRD `status.storedVersions`
  *  This isn’t completely necessary, but because kubectl doesn’t support the ability to change status it will be very difficult for users to follow the upgrade procedure without it.

#### Users

Users would have to be using Kubernetes 1.16 or greater.

Users will need to follow a specific procedure to upgrade:

1. Install the Strimzi X CRDs using `kubectl apply|replace -f` like normal.
2. Touch each of their CRs, changing the `apiVersion` to `v1beta2` (the tool can do this).
3. Edit the CRD so that `v1alpha1` and `v1beta1` are both served=false (the tool can do this).
4. Double check that all their CRs are using v1beta2 and all their apps are still functioning (the tool can do this).
5. Edit the CRD `status.storedVersions` to remove `v1alpha1` and `v1beta1` (the tool can do this).
At this point they’re all set for Strimzi Z

### Strimzi Z (approximately April 2021)

This should work on Kubernetes 1.16+ (including 1.22+). This will be the first release using `apiextensions.k8s.io/v1` CRD API. 

#### Development

The CRD will use `apiextensions.k8s.io/v1` CRD API.

#### Users

The upgrade will just be a matter of replacing the CRDs as usual.

### Alternative

Steps 2-5 in the procedure ascribed to Strimzi X, above are technically just preprequisites before installing CRDs which use the `apiextensions.k8s.io/v1` API. As such they could equally be done by users as part of the upgrade to Strimzi Z. The benefit of doing them sooner is that users would be able to perform those steps during their use of Strimzi X, which might be more convenient.


## Timing

Strimzi Z could be any time before the release of Kubernetes 1.22, but obviously the more time people have to upgrade the better and the more breathing space we have before 1.22 the better to iron out problems people might have in upgrading.


## Compatibility

Under this proposal Strimzi X would not support Kubernetes 1.15 or earlier.

## Rejected alternatives

### Conversion Webhooks

The CRD API allows two ways of converting APIs from one version to another. The `None` conversion is basically a no-op. The API server just reads the CR from etcd, changes its `apiVersion` and hands it out. This is what we used for v1alpha1→v1beta1. It means you can’t actually remove fields in v1beta1 which were in v1alpha1 without breaking API compatibility. 
The alternative is `Webhook` conversion, where you write a HTTP server which the API server calls to convert from one version to another. This can handle more complex schema changes and it’s likely we’ll want to use this in the future (for example to get rid of spec.zookeeper when KIP-500 is fully delivered).

The complication with Webhook conversion is that the CRD needs to supply a `caBundle` which is the x509 Certificate Authority that has signed the certificate that the webhook presents to the API server. In other words, the 
API server needs to be told to trust the webhook server. You either:

* Generate this CA and sign the webhook certificate at build time. This allows you to ship a CRD which the user doesn't need to change, but offers no security and subverts the point of the API server using HTTPS.
* Or you have to get a certificate signed by a CA at installation time, and update the CRD with the CA certificate used for the signing. This makes installation much harder since the user now needs to perform steps using `openssl`, `cfssl` or similar.
* Or you sign the webhook certificate with the internal Kubernetes CA, which means you can omit caBundle, but means the webhook becomes completely trusted by the whole kubernetes infrastructure, something which cluster admins are unlikely to want to contemplate.

While it's likely we'll need a conversion webhook at some point in the future, _requiring_ one in order to upgrade to Kubernetes 1.21 is not necessary and adds significant installation complexity to Strimzi.