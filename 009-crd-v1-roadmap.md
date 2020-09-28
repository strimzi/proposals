

# Roadmap to using the CRD v1 API

This describes (roughly) the steps Strimzi users will need to take to get to using v1beta2 Strimzi APIs which, in turn, are using the v1 CRD API. In it we talk about Strimzi 0.21.0 and 0.22.0, but the changes described under those headings wouldn’t necessarily need to be delivered in exactly those versions. Rather the requirement is that what we’re calling Strimzi 0.22 is released around the time that Kubernetes 1.21 is.

## Current situation

Currently Strimzi defines numerous CustomResourceDefinitions) using the `apiextensions.k8s.io/v1beta1`. Those CRDs define the "Strimzi CRs", as v1alpha1 and v1beta1 APIS in the `kafka.strimzi.io` API group.

Kubernetes 1.21 is expected to have removed CRD `apiextensions.k8s.io/v1beta1`. Its replacement, `apiextensions.k8s.io/v11`, has been available since Kubernetes 1.16.


## Motivation

To work on Kubernetes 1.21 Strimzi will need to use the v1 CRD API. We can't have a single set of CRDs which support both Kubernetes 1.21 or later and also 1.15 or earlier. Furthermore, using the v1 CRD API requires that the valiation schemas in the CRD are all "structural schemas".

## Proposal

### Strimzi 0.20

This will be the last version of Strimzi which supports Kubernetes 1.11-1.15. 

#### Users

Users should:

 * Rewrite their Kafka `spec.kafka.listeners` fields to use a list. 
 * Check they’re not using any “deprecated” fields in any of their resources.

### Strimzi "0.21.0"

This release will be a gate release, meaning that all existing Strimzi users will have to pass through 0.21 in order to upgrade to 0.22.
This should work on Kubernetes 1.16-1.20. It can’t work on lower versions because it relies on having support for per-versions validation schemas in CRDs, a feature that wasn’t available before 1.16. It can’t work on higher versions because that don’t support `v1beta1` of the CRD API.

#### Development

We need to:

* Audit all existing CRD looking for things we need to change in order to have a structural schema. This is basically about checking that the type of each property is not going to need to change.
* Prime suspects include things which are declared as a `Map<String, Object>` is Java, hence an object in JSON Schema. This includes `Kafka.spec.kafka.jvmOptions`, `Kafka.spec.zookeeper.jvmOptions`, and similar in other CRDs.
For each of these we should make the analogous change that we did for listeners in 0.20.0.
* Develop the `v1beta2` schema. This must be structural. This must be backward compatible with `v1beta1`, by which I mean any schema-valid `v1beta2` CR must be a schema-valid `v1beta1` CR (though not vice versa).
* The 0.21.0 CRD will declare `v1alpha1` (served=true, storage=false), `v1beta1` (served=true, storage=false) and `v1beta2` (served=true, storage=true)
* The operator will access the CRs via the `v1beta2` path (it will be able to observe `v1beta1` resources because the None conversion just changes the apiVersion). 
* Develop a tool which:
  * Can rewrite a `v1beta1` resource to an equivalent `v1beta2` resource, allowing users to easily modify their resources (e.g. in git).
  * Can apply/edit CRs in Kubernetes likewise.
  * Can update the CRD `spec.versions` to change the served field.
  * Can update the CRD `status.storedVersions`
  *  This isn’t completely necessary, but because kubectl doesn’t support the ability to change status it will be very difficult for users to follow the upgrade procedure without it.

#### Users

Users will need to follow a specific procedure to upgrade:

1. Install the 0.21.0 CRDs using `kubectl apply|replace -f` like normal.
2. Touch each of their CRs, changing the apiVersion to v1beta2 (the tool can do this).
3. Edit the CRD so that `v1alpha1` and `v1beta1` are both served=false (the tool can do this).
4. Double check that all their CRs are using v1beta2 and all their apps are still functioning (the tool can do this).
5. Edit the CRD `status.storedVersions` to remove `v1alpha1` and `v1beta1` (the tool can do this).
At this point they’re all set for Strimzi 0.22.0

### Strimzi "0.22.0"

This should work on Kubernetes 1.16+ (including 1.21+). This will be the first release using CRD v1 API. 

#### Development

The CRD will be CRD v1 API

#### Users

The upgrade will just be a matter of replacing the CRDs as usual.

### Alternative

Steps 2-5 in the procedure ascribed to Strimzi 0.21, above are technically just preprequisites before installing CRDs which use the v1 API. As such they could equally be done by users as part of the upgrade to Strimzi 0.22. The benefit of doing them sooner is that users would be able to perform those steps during their use of Strimzi 0.21, which might be more convenient.

## Affected/not affected projects

Call out the projects in the Strimzi organisation that are/are not affected by this proposal. 

## Timing

Strimzi 0.21 could be any time before the release of Kubernetes 1.21, but obviously the more time people have to upgrade the better and the more breathing space we have before 1.21 the better to iron out problems people might have in upgrading.



## Compatibility

Under this proposal Strimzi "0.22" would not support Kubernetes 1.15 or earlier.

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