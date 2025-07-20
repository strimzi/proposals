# Add namespace filtering support to Strimzi Drain Cleaner

**Authors:** Roman Melnyk (@aywengo)  
**Status:** Draft  
**Type:** Enhancement  
**Created:** 2025-07-19  
**Strimzi Version:** 0.48.0+  

## Summary

This proposal introduces namespace filtering capability to the Strimzi Drain Cleaner through a new `STRIMZI_DRAIN_WATCH_NAMESPACES` environment variable. This feature enables the drain cleaner to only process eviction requests for pods in specified namespaces, addressing security and operational challenges in multi-tenant enterprise Kubernetes environments.

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

Introduce a new environment variable `STRIMZI_DRAIN_WATCH_NAMESPACES` that accepts a comma-separated list of namespace names. When configured, the drain cleaner will:

1. Continue receiving all eviction webhook requests (required for webhook functionality)
2. Filter requests at the application level before attempting pod access
3. For unwatched namespaces: immediately allow eviction without pod API calls
4. For watched namespaces: process normally with full drain cleaner logic

### Configuration

**Environment Variable**: `STRIMZI_DRAIN_WATCH_NAMESPACES`
- **Format**: Comma-separated namespace list (e.g., `"kafka-prod,kafka-dev,messaging"`)
- **Default**: Empty string (watches all namespaces - current behavior)
- **Behavior**:
  - Empty/unset: Process all namespaces (backward compatible)
  - Non-empty: Only process eviction requests from specified namespaces

### Application-Level Filtering

The filtering occurs in the `ValidatingWebhook.webhook()` method:

```java
private boolean isNamespaceWatched(String namespace) {
    if (watchNamespaces == null || watchNamespaces.trim().isEmpty()) {
        return true; // Watch all namespaces (current behavior)
    }
    
    List<String> watchedNamespaceList = Arrays.asList(watchNamespaces.split(","));
    return watchedNamespaceList.stream()
            .map(String::trim)
            .filter(ns -> !ns.isEmpty())
            .anyMatch(watchedNs -> watchedNs.equals(namespace));
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
- Add `@ConfigProperty` for `strimzi.drain.watch.namespaces`
- Implement `isNamespaceWatched()` method
- Add namespace check before pod retrieval

**Configuration** (`src/main/resources/application.properties`):
```properties
# Comma-separated list of namespaces to watch for eviction events. Empty means all namespaces.
strimzi.drain.watch.namespaces=
```

**Deployment Templates** (in `packaging/` directory only):
- `packaging/install/*/060-Deployment.yaml`
- `packaging/helm-charts/helm3/strimzi-drain-cleaner/values.yaml`

### Behavior Examples

**Example 1: Watched Namespace**
```yaml
env:
  - name: STRIMZI_DRAIN_WATCH_NAMESPACES
    value: "kafka-prod,kafka-dev"
```
- Eviction request for pod in `kafka-prod` → Full drain cleaner processing
- Result: Normal Strimzi drain cleaner behavior

**Example 2: Unwatched Namespace**
- Eviction request for pod in `app-namespace` → Immediate allow
- Log: `DEBUG: Ignoring eviction request for Pod some-pod in namespace app-namespace - namespace not in watch list`
- Result: Eviction proceeds normally, no API calls made

**Example 3: Default Behavior**
```yaml
env:
  - name: STRIMZI_DRAIN_WATCH_NAMESPACES
    value: ""
```
- All namespaces processed (current behavior)
- Full backward compatibility maintained

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
**Rejected Because**:
- Requires multiple ValidatingWebhookConfigurations
- Potential conflicts and performance issues
- Goes against "one per cluster" design principle
- Complex certificate and configuration management

### 3. Webhook Namespace Selectors
**Approach**: Use ValidatingWebhookConfiguration `namespaceSelector`
**Rejected Because**:
- Requires cluster-level configuration changes
- Less flexible than application-level configuration
- Cannot be easily managed through Helm values
- Harder to update without webhook re-registration

### 4. Admission Controller Scope Changes
**Approach**: Modify webhook registration scope
**Rejected Because**:
- Changes fundamental architecture
- Reduces drain cleaner effectiveness
- Complex migration path for existing deployments

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

### Backward Compatibility Tests
- Verify existing deployments continue working without configuration changes
- Test upgrade scenarios from previous versions
- Confirm default behavior matches current implementation

## Graduation/Rollout Plan

### Phase 1: Implementation and Testing
- Implement core functionality in drain cleaner
- Add comprehensive test coverage
- Update packaging configurations
- Documentation updates

### Phase 2: Beta Release
- Include in next minor Strimzi release (0.48.0+)
- Mark as beta feature in release notes
- Gather community feedback
- Monitor for issues in test environments

### Phase 3: General Availability
- Promote to stable feature status
- Include in documentation as recommended pattern for multi-tenant environments
- Consider for inclusion in Helm chart best practices

## Documentation Updates

- **Configuration Reference**: Add `STRIMZI_DRAIN_WATCH_NAMESPACES` to environment variables documentation
- **Multi-Tenant Guide**: Create section on namespace filtering for enterprise deployments
- **Security Guide**: Document RBAC considerations and filtering benefits
- **Troubleshooting**: Add guidance for configuration and debugging
- **Migration Guide**: Document upgrade considerations for existing deployments

## Metrics and Monitoring

### Success Metrics
- Reduction in permission error log entries
- Decreased API server load from drain cleaner
- Community adoption in multi-tenant environments
- User feedback on deployment simplification

### Monitoring Considerations
- Log volume reduction in restricted environments
- API call patterns to Kubernetes API server
- Webhook response times and success rates

## Future Enhancements

While not part of this proposal, future work could include:

1. **Dynamic Configuration**: Support for runtime configuration updates
2. **Namespace Pattern Matching**: Support for regex or wildcard patterns
3. **Enhanced Logging**: Structured logging with namespace filtering statistics
4. **Metrics Exposure**: Prometheus metrics for namespace filtering behavior

## References

- [Strimzi Drain Cleaner Documentation](https://strimzi.io/docs/operators/latest/deploying.html#drain-cleaner)
- [Multi-Tenant Kubernetes Best Practices](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)
- [Kubernetes Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Related Pull Request: #164](https://github.com/strimzi/drain-cleaner/pull/164)

---

**Proposal Author**: Roman Melnyk (aywengo@gmail.com)  
**Implementation Reference**: [Drain Cleaner PR #164](https://github.com/strimzi/drain-cleaner/pull/164) 