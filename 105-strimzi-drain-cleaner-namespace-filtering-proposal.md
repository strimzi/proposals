# Add support for namespace filtering in Strimzi Drain Cleaner

## Summary

This proposal introduces support for namespace filtering in the Strimzi Drain Cleaner using a new `STRIMZI_DRAIN_NAMESPACES` environment variable. This feature enables the drain cleaner to only process eviction requests for pods in specified namespaces, addressing security and operational challenges in multi-tenant enterprise Kubernetes environments.

## Motivation

### Current Challenges

In large-scale multi-tenant Kubernetes environments, organizations face several challenges with the current drain cleaner implementation:

1. **Security Policy Violations**: Corporate security teams often mandate strict RBAC policies that prevent cluster-wide `pods/get` and `pods/patch` permissions for third-party service accounts, even when using Helm parameter-based restrictions.

2. **Operational Noise**: Kafka clusters typically exist in only 3-5 specific namespaces out of 100+ total namespaces in enterprise clusters. The drain cleaner attempts to access every pod during eviction events, generating thousands of permission denied errors in logs:
   ```
   KubernetesClientException: pods "app-pod" is forbidden: User "system:serviceaccount:kafka:strimzi-drain-cleaner" cannot get resource "pods" in API group "" in namespace "restricted-namespace"
   ```

3. **Multi-Tenant Isolation Requirements**: Large corporations need the ability to deploy one drain cleaner per cluster (as designed) while respecting namespace isolation boundaries enforced by security teams.

4. **Performance Impact**: Unnecessary API calls to restricted namespaces consume cluster resources and generate log volume that can mask real operational issues.

### Real-World Impact

This issue affects enterprise environments where:
- Kafka is deployed in specific namespaces (e.g., `kafka-prod`, `kafka-dev`, `messaging`)
- Security teams enforce least-privilege RBAC policies
- Compliance requirements mandate namespace-level access controls
- One drain cleaner instance must serve the entire cluster while respecting security boundaries

### Potential Negative Impacts

**Risk of Multiple Drain Cleaner Deployments**: With namespace filtering, users might be tempted to deploy multiple drain cleaner instances per cluster, which could lead to:
- Conflicting `ValidatingWebhookConfigurations`
- Duplicate webhook processing and potential race conditions
- Increased operational complexity and resource usage
- Certificate management challenges

**Mitigation**: Clear documentation must emphasize that:
- Only ONE drain cleaner instance should be deployed per cluster
- Namespace filtering is for security compliance, not to support multi-tenancy
- Deploying multiple instances will cause operational issues and is unsupported

## Goals

- **Primary Goal**: Enable namespace-level filtering to allow drain cleaner deployment in restrictive RBAC environments
- **Security Goal**: Reduce actual pod API calls to only configured namespaces while maintaining webhook functionality
- **Operational Goal**: Eliminate log noise from permission errors in unmanaged namespaces
- **Compatibility Goal**: Maintain full backward compatibility with existing deployments

## Non-Goals

- Modifying cluster-wide webhook registration (webhooks must still intercept all eviction requests)
- Changing RBAC configurations or permissions automatically
- Supporting multiple drain cleaner instances per cluster
- Automatic migration or configuration discovery

## Proposal

### Overview

Introduce a new environment variable `STRIMZI_DRAIN_NAMESPACES` that accepts a comma-separated list of namespace names. When configured, the drain cleaner will:

1. Continue receiving all eviction webhook requests (required for webhook functionality)
2. Filter requests at the application level before attempting pod access
3. For unwatched namespaces: immediately allow eviction without pod API calls (logged at DEBUG level)
4. For watched namespaces: process normally with full drain cleaner logic

### Configuration

**Environment Variable**: `STRIMZI_DRAIN_NAMESPACES`
- **Format**: Comma-separated namespace list (e.g., `"kafka-prod,kafka-dev,messaging"`) or `"*"` for all namespaces
- **Default**: `"*"` (process all namespaces - current behavior)
- **Behavior**:
  - `"*"` or unset: Process all namespaces (backward compatible)
  - Specific namespace list: Only process eviction requests from specified namespaces

### Application-Level Filtering

The filtering occurs in the `ValidatingWebhook.webhook()` method:

```java
private boolean isNamespaceWatched(String namespace) {
    if (drainNamespaces == null || drainNamespaces.trim().isEmpty() || drainNamespaces.trim().equals("*")) {
        return true; // Process all namespaces (current behavior)
    }
    
    List<String> drainNamespaceList = Arrays.asList(drainNamespaces.split(","));
    return drainNamespaceList.stream()
            .map(String::trim)
            .filter(ns -> !ns.isEmpty())
            .anyMatch(drainNs -> drainNs.equals(namespace));
}
```

**Request Flow**:
1. Webhook receives eviction request
2. Extract namespace from eviction metadata
3. Check if namespace is watched using `isNamespaceWatched()`
4. If unwatched: Log debug message and allow eviction immediately
5. If watched: Continue with existing drain cleaner logic

## Affected/not affected projects

**Affected Projects:**
- **strimzi-drain-cleaner**: Core implementation of namespace filtering functionality, including packaging and deployment templates

**Not Affected Projects:**
- **strimzi-operator**: No changes required to main operator logic
- **strimzi-bridge**: No impact on bridge functionality
- **strimzi-canary**: Independent component, no changes needed
- **test-container**: No changes to testing infrastructure

## Implementation Details

### Code Changes

**Core Logic** (`src/main/java/io/strimzi/ValidatingWebhook.java`):
- Add `@ConfigProperty` for `strimzi.drain.namespaces`
- Implement `isNamespaceWatched()` method
- Add namespace check before pod retrieval

**Configuration** (`src/main/resources/application.properties`):
```properties
# Comma-separated list of namespaces to process for eviction events. "*" or empty means all namespaces.
strimzi.drain.namespaces=*
```

**Deployment and Documentation**:
- This feature will be **documentation-only** and not included in the default installation files
- Users will need to manually configure the environment variable when needed
- RBAC permissions must remain cluster-wide for webhook functionality
- The Helm Chart will not be modified to avoid confusion about RBAC scope
- Clear documentation will guide users on proper configuration

### Behavior Examples

**Example 1: Watched Namespace**
```yaml
env:
  - name: STRIMZI_DRAIN_NAMESPACES
    value: "kafka-prod,kafka-dev"
```
- Eviction request for pod in `kafka-prod` namespace → Full Drain Cleaner processing
- Result: Normal Strimzi Drain Cleaner behavior

**Example 2: Unwatched Namespace**
- Eviction request for pod in `app-namespace` → Immediate allow
- Log: `DEBUG: Ignoring eviction request for Pod some-pod in namespace app-namespace - namespace not in watch list`
- Result: Eviction proceeds normally, no API calls made

**Example 3: Default Behavior**
```yaml
env:
  - name: STRIMZI_DRAIN_NAMESPACES
    value: "*"
```
- All namespaces processed (current behavior)
- Full backward compatibility maintained
- Can also be left unset for same behavior

### Security Considerations

1. **No RBAC Changes Required**: Existing cluster-wide permissions remain necessary for webhook operation
2. **Application-Level Security**: Filtering prevents unnecessary API calls to restricted namespaces
3. **Audit Trail**: All filtering decisions logged at appropriate levels
4. **Fail-Safe Design**: Invalid configuration defaults to watching all namespaces

## Alternatives Considered

### 1. RBAC-Level Filtering
**Approach**: Use Kubernetes RBAC to restrict access
**Rejected Because**: 
- Webhooks require cluster-wide registration
- Cannot selectively grant namespace access for webhook operations
- Would break fundamental webhook architecture

### 2. Multiple Drain Cleaner Instances
**Approach**: Deploy separate drain cleaners per namespace
**Why This Doesn't Work**:
- Requires multiple ValidatingWebhookConfigurations
- Potential conflicts and performance issues
- Goes against "one per cluster" design principle
- Complex certificate and configuration management

**Important Note**: While this proposal's namespace filtering might suggest multiple instances could work, this remains unsupported. Users must understand that attempting multiple deployments will cause operational issues. This must be clearly documented.

### 3. Webhook Namespace Selectors
**Approach**: Use ValidatingWebhookConfiguration `namespaceSelector`
**Rejected Because**:
- Requires cluster-level configuration changes
- Less flexible than application-level configuration
- Cannot be easily managed through Helm values
- Harder to update without webhook re-registration

### 4. Admission Controller Scope Changes
**Approach**: Attempted to make webhooks namespace-scoped instead of cluster-scoped
**Rejected Because**:
- ValidatingWebhookConfigurations cannot be namespace-scoped (Kubernetes limitation)
- Webhooks must be cluster-wide to intercept all eviction requests
- Would require fundamental redesign of Kubernetes admission control
- Not technically feasible within current Kubernetes architecture

## Test Plan

### Unit Tests
- `testNamespaceFilteringWithWatchedNamespace()`: Verify processing of watched namespaces
- `testNamespaceFilteringWithUnwatchedNamespace()`: Verify skipping of unwatched namespaces  
- `testNamespaceFilteringWithEmptyConfiguration()`: Verify backward compatibility
- `testNamespaceFilteringWithWhitespaceHandling()`: Verify configuration parsing

### Integration Tests
- Deploy drain cleaner with namespace filtering in multi-namespace test environment
- Verify eviction handling for both watched and unwatched namespaces
- Confirm no API calls made to unwatched namespaces
- Validate log output and behavior

### Backward Compatibility
- Existing tests will verify backward compatibility automatically
- Default behavior (`STRIMZI_DRAIN_NAMESPACES="*"` or unset) maintains current functionality
- No special upgrade testing needed as Drain Cleaner is a stateless application

## Implementation Plan

This feature will be included in the next Drain Cleaner release (likely version 1.5.0). Since the Drain Cleaner follows independent release cycles from the main Strimzi operator:

- Implement core functionality with comprehensive test coverage
- Update README.md with configuration instructions
- Include in the Strimzi Operator Deploying guide
- Release as a standard feature (no beta phase or feature gates needed)

## Documentation Updates

The following documentation will be updated:

1. **Drain Cleaner README.md**: 
   - Add `STRIMZI_DRAIN_NAMESPACES` environment variable description
   - Include configuration examples
   - Add warning about not deploying multiple instances

2. **Strimzi Operator Deploying Guide**:
   - Update the Drain Cleaner installation section
   - Document procedure for namespace filtering of multi-tenant environments
   - Include RBAC considerations and security benefits

## Future Enhancements

While not part of this proposal, future work could include:

1. **Dynamic Configuration**: Support for runtime configuration updates
2. **Namespace Pattern Matching**: Support for regex or wildcard patterns
3. **Enhanced Logging**: Structured logging with namespace filtering statistics
4. **Metrics Exposure**: Prometheus metrics for namespace filtering behavior

 