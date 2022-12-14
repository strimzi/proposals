# StrimziPodSets graduation

This proposal proposes updated schedule for StrimziPodSet gradutation.

## Current situation

StrimziPodSets were proposed and approved as part of the [SP-031 StatefulSet Removal proposal](https://github.com/strimzi/proposals/blob/main/031-statefulset-removal.md) (visit the proposal for more details about the StrimziPodSets).
StrimziPodSets are currently behind a feature gate named `UseStrimziPodSets` which is enabled by default, but can be optionally disabled.
That means that the code paths still using StatefulSets are still maintained and tested.

The original schedule proposed the feature gate to move to beta (enabled by default) in Strimzi 0.29 and to GA (enabled by default without possibility to disable it) in Strimzi 0.31.
In reality, it moved to beta stage in Strimzi 0.30 and as of today (Strimzi 0.32) it is still in beta.

The StrimziPodSets are now enabled by default for 3 releases (0.30-0.32).
There are no known issues, bugs or missing features on Strimzi side

## Proposal

This proposal suggests an updated graduation schedule:
* Strimzi 0.33 and 0.34 will be released with the `USeStrimziPodSet` feature gate still in beta
* Unless some major issues are found during Strimzi 0.33 life-cycle (i.e. between the release of Strimzi 0.33 and 0.34), the feature gate will move to GA right after the 0.34.0 release and the code paths related to StatefulSets will be removed.
  The only StatefulSet related functionality which will remain will be related to upgrading from StatefulSets to StrimziPodSets.
  This will be the deletion of the old resources (StatefulSets, shared ConfigMaps etc).
  Strimzi 0.35 will be released with StrimziPodSets feature gate in GA being permanently enabled without the possibility to disable it.

Thanks to this timeline:
* Users will have additional time to test the StrimziPodSets with the 0.33 release
* Removing the StatefulSet support right after the 0.34 release will give us additional to ensure that the removal was done correctly (compared to removing it last minute before the Strimzi 0.34 release)

Assuming this proposal is approved, the 0.33 release can be used to announce this and encourage users to test the StrimziPodSets.
If any major bugs are found before the 0.34 release and before the StatefulSets code is removed, the timeline can be still reconsidered.

Moving forward with StrimziPodSets will make it easier for to continu the development of the additional features built on top of StrimziPodSets.
This includes KRaft support, Node pools or in the long term strecth clusters.
It will also simplify testing.

## Rejected alternatives

### Removing StatefulSets right before the 0.34 release

One considered alternative was to move the feature gate to GA already as part of 0.34 release.
However, this would either mean that we would remove the code _last minute_ before the release.
Or we would remove it right after the 0.33 release whihc would mean that if any major issue is found later and we decide to change the schedule, it will be complicated to revert the changes.