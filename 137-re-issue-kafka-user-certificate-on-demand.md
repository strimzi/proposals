# Re-issue KafkaUser's certificate on demand

## Current situation

Currently, when users would like to renew or re-issue the `KafkaUser`'s certificate without waiting for automatic renewal, they can do so by deleting the `Secret` of the particular `KafkaUser`.
Once User Operator then goes through the regular reconciliation, it observes that the `Secret` for the `KafkaUser` is missing, and it creates a new one.

## Motivation

Even though the workaround with deleting of the `Secret` works, it would be a nice enhancement to provide a way how to re-issue the certificate using annotation, rather than advising users to delete the `Secret` to do so.
This would be useful in cases that users would like to get certificate with new validity or when testing new changes.
Additionally and most importantly, deleting the `Secret` can lead in service interruption (and race conditions), when there is restart of the application Pod without `Secret` being in place, because User Operator didn't go through the reconciliation yet.
Providing an annotation instead would ensure that there will be no service interruption and that the certificate will be renewed in next reconciliation.
It would also follow the way we have for CA certificates, which can be renewed or replaced by annotations `strimzi.io/force-renew` or `strimzi.io/force-replace`.

## Proposal

This proposal suggests adding new annotation - `strimzi.io/re-issue-certificate` - that will be configurable on `Secret` resource of the `KafkaUser`.
Users will be able to annotate the `Secret` as follows:

```shell
kubectl annotate secret my-user strimzi.io/re-issue-certificate="true" -n my-namespace
```

The annotation accepts a boolean value and if the user annotates the `Secret` with `strimzi.io/re-issue-certificate: true`, on the next reconciliation User Operator will renew the certificate of the `KafkaUser`.

That will be handled in the `KafkaUserModel#maybeGenerateCertificates()`, where we will check if the existing `Secret` contains this annotation and if it is set to `true`.
If yes, it will trigger the renewal by using the `generateNewCertificate(reconciliation, clientsCa);`.
Otherwise, the behavior will not change, it will go through the checks as before - if the keys are from same CA, if it's not expired, etc.

After the certificate is renewed, the annotation will be automatically removed from the `Secret` - as we will create a new `Secret` replacing the current one.
There will be no special handling needed.

## Affected/not affected projects

The only affected project is the `strimzi-kafka-operator` repository - especially User Operator part of the code.

## Compatibility

This proposal is fully backwards compatible, it adds new option to renew/re-issue the `KafkaUser` certificate on demand.

## Rejected alternatives

### Handling the annotation on `KafkaUser` level

Handling the annotation on `KafkaUser` level was considered, however the handling inside the `KafkaUserModel` would be more complicated, and it would differ to the way how we handle the CAs (by annotating the `Secret`).
Other than that, handling it on the `Secret` level helps with removal of the annotation (as we will create completely new Secret).
Lastly, in case that we would handle the annotation on `KafkaUser` level, GitOps operators like Flux or ArgoCD _may_ replace the CR - meaning that race condition can happen before User Operator actually picks up the annotation from the `KafkaUser` CR.

### Configure date inside the annotation instead of boolean value

Configuring date in value field of the annotation was considered as well.
It would again differ from the way we picked for CA renewal - where we are using boolean value.
Other than that, it would make handling of the re-issue a bit more complicated - we would have to implement a way how to handle date, time, and also provide template in which users can specify the date, together with checks if the date is correct, and many more.
Finally, it would complicate users lives in case that they would like to re-issue the certificate immediately - which would be the most common case.