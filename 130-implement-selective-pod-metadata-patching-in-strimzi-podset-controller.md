# Selective Pod Metadata Patching in StrimziPodSetController

Add support for in-place patching of Pod metadata (labels and annotations) without requiring a rolling restart.
Previously, any change to a Pod would trigger a rolling update; now metadata-only changes are applied via Strategic Merge Patch.

## Current situation

Currently, when the StrimziPodSetController detects a difference between the desired Pod state (from the StrimziPodSet) and the actual Pod state in Kubernetes, it will initiate a rolling restart of the Pod.

This happens regardless of whether the change is a metadata-only change (labels/annotations) or a change that actually requires the Pod to be recreated (e.g., container image, volumes, etc.).

The existing logic in `maybeCreateOrPatchPod()` uses only Pod readiness and a hash-based comparison (`PodRevision`) to determine if a Pod needs to be replaced:

```java
boolean podUpToDate = PodRevision.hasChanged(existingPod, desiredPod);
if (!podUpToDate && isPodReady(existingPod) && isPodCurrent(existingPod)) {
    // Pod is fine, nothing to do
} else {
    // Pod needs to be deleted and recreated -> triggers rolling restart
}
```

This means that adding a new annotation to a Pod (e.g., for monitoring or tracing purposes) will cause an unnecessary Pod restart and potential service disruption.

## Motivation

1. **Reduce unnecessary restarts**: Adding labels or annotations to Pods should not require deleting and recreating the Pod. This causes unnecessary downtime and can disrupt consumer applications.

2. **Improved user experience**: Users often want to add custom labels or annotations for various operational purposes (e.g., cost tracking, monitoring integration). These changes should be seamless.

3. **Better alignment with Kubernetes practices**: Kubernetes itself allows in-place patching of Pod metadata. Strimzi should leverage this capability.

4. **target a technical debt**: A TODO comment at line 466 of `StrimziPodSetController.java` explicitly requested this enhancement.

## Proposal

Introduce a `PodDiff` utility class that categorizes differences between Pods into three categories:

1. **NONE**: No changes detected
2. **METADATA_ONLY**: Only labels and/or annotations differ
3. **REQUIRES_RESTART**: Changes to spec or other fields that require Pod recreation

### PodDiff Utility Class

A new utility class `PodDiff` will be added to categorize Pod changes:

```java
public class PodDiff {
    public enum DiffType {
        NONE,
        METADATA_ONLY,
        REQUIRES_RESTART
    }

    public static DiffType diff(Pod current, Pod desired) {
        // Check if metadata has changed
        boolean metadataChanged = hasMetadataChanged(current, desired);
        
        // Check if spec has changed (using PodRevision hash)
        boolean specChanged = PodRevision.hasChanged(current, desired);
        
        if (!metadataChanged && !specChanged) {
            return DiffType.NONE;
        } else if (metadataChanged && !specChanged) {
            return DiffType.METADATA_ONLY;
        } else {
            return DiffType.REQUIRES_RESTART;
        }
    }
    
    private static boolean hasMetadataChanged(Pod current, Pod desired) {
        // Compare labels and annotations
        ObjectMeta currentMeta = current.getMetadata();
        ObjectMeta desiredMeta = desired.getMetadata();
        
        return !Objects.equals(currentMeta.getLabels(), desiredMeta.getLabels())
            || !Objects.equals(currentMeta.getAnnotations(), desiredMeta.getAnnotations());
    }
}
```

### Changes to StrimziPodSetController

1. **Add `patchPodMetadata()` helper method**:

```java
private Future<Void> patchPodMetadata(Pod existingPod, Pod desiredPod) {
    String name = existingPod.getMetadata().getName();
    
    // Create a patch with only the metadata changes
    Pod patch = new PodBuilder()
        .withNewMetadata()
            .withName(name)
            .withNamespace(existingPod.getMetadata().getNamespace())
            .withLabels(desiredPod.getMetadata().getLabels())
            .withAnnotations(desiredPod.getMetadata().getAnnotations())
        .endMetadata()
        .build();
    
    return podOperator.patchAsync(reconciliation, patch);
}
```

2. **Modify `maybeCreateOrPatchPod()`** to use `PodDiff`:

```java
private Future<Void> maybeCreateOrPatchPod(Pod desiredPod, Pod existingPod) {
    if (existingPod == null) {
        return createPod(desiredPod);
    }
    
    PodDiff.DiffType diffType = PodDiff.diff(existingPod, desiredPod);
    
    switch (diffType) {
        case NONE:
            // Pod is already in desired state
            return Future.succeededFuture();
            
        case METADATA_ONLY:
            // Only metadata changed, patch instead of recreate
            LOGGER.debugCr(reconciliation, "Pod {} has metadata changes only, patching", 
                existingPod.getMetadata().getName());
            return patchPodMetadata(existingPod, desiredPod);
            
        case REQUIRES_RESTART:
            // Full recreation required
            return deletePodAndRecreate(existingPod, desiredPod);
    }
}
```

### Unit Tests

Comprehensive unit tests will be added for `PodDiff`:

- Test with identical Pods → `NONE`
- Test with label changes only → `METADATA_ONLY`
- Test with annotation changes only → `METADATA_ONLY`
- Test with both label and annotation changes → `METADATA_ONLY`
- Test with container image change → `REQUIRES_RESTART`
- Test with volume changes → `REQUIRES_RESTART`
- Test with metadata + spec changes → `REQUIRES_RESTART`

## Affected/not affected projects

This only affects the Strimzi cluster operator, specifically the `StrimziPodSetController` class.

## Compatibility

This change is backward compatible. The behavior for spec changes remains the same (Pod recreation). 
The only difference is that metadata-only changes will now be handled more efficiently without Pod restarts.

Users who previously observed Pod restarts when adding labels/annotations will now see the changes applied in-place.
This is an improvement and should not cause any issues for existing deployments.

## Rejected alternatives

### Always patching Pods instead of recreating

We considered always using patches for all Pod changes. However, Pods are largely immutable in Kubernetes - you cannot patch fields like `containers`, `volumes`, or `nodeSelector` after creation. Therefore, the selective approach (patch metadata, recreate for spec changes) is the only viable solution.

### Using Server-Side Apply for all changes

While Server-Side Apply (SSA) could be used, it introduces complexity around field ownership and conflict resolution. For the specific use case of metadata patching, a simple Strategic Merge Patch is sufficient and more straightforward to implement.

### Ignoring metadata differences entirely

We considered simply ignoring metadata differences in the comparison logic. However, this would mean user-specified labels and annotations would never be applied to existing Pods, which is not the desired behavior.
