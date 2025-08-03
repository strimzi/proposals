# Make _ignored_ users in the User Operator configurable

The User Operator currently has two hardcoded usernames for which any ACLs are ignored.
These users are `ANONYMOUS` (unauthenticated users) and `*` (all users).
When reconciling the ACL rules and seeing any rules for one of these usernames, it will ignore them instead of deleting them.

This feature and the two specific users were originally added based on requests from our users and are useful in some niche scenarios.
For example, when enabling authentication and authorization on an existing insecure Kafka cluster, being able to grant the `ANONYMOUS` user some basic ACL rights might help with a smooth transition.
Similarly, being able to grant some basic rights to all users might also be useful in some scenarios.

The solution with a hardcoded ignore list was chosen because the `KafkaUser` resources cannot be used to manage ACLs for these usernames as they are not valid Kubernetes resource names.
These ACLs have to be managed directly using the Kafka Admin API.
And we needed to make sure these rules would not be immediately deleted by the User Operator.

This proposal suggests changing how the ignored users work.
It proposes to:
* Make the ignored usernames configurable instead of hardcoded
* Extend the ignored users to the whole User Operator (i.e., to Quotas and SCRAM-SHA512 users)

## Motivation

The main motivation for this change is the security aspect of this feature.
Ignoring the ACL rules for the `ANONYMOUS` and/or `*` entities can pose significant risks.
If an attacker manages to create ACL rights for one of them, it might give them extensive access rights to the Kafka cluster which would otherwise not be available to them.
And unlike ACL rules created for regular users, the User Operator would just ignore these rules.

To further increase the risk, this mechanism is documented only briefly in a single paragraph in our documentation.
That makes it easy to miss, and most Strimzi users probably don't know about this _feature_.
And while the User Operator logs an INFO level message when it is ignoring any ACL rules, the log message can also be easily missed.

This proposal suggests making the list of ignored users configurable with an empty default value (i.e., no users will be ignored by default).
For most users, this should have zero impact.
And users who need the `ANONYMOUS` or `*` users to be ignored can configure them explicitly in the configuration.
The explicit configuration will ensure that they are aware that the ACL rules for these users would be ignored even without carefully reading every single documentation page, because they had to configure it.

In addition to the main motivation - security - this proposal should deliver some additional benefits.
It proposes extending the ignore list to all User Operator-managed resources, including ACLs, quotas, and SCRAM-SHA-512 credentials.
It also proposes using a regular expression pattern instead of a list of users to allow more flexibility in how the ignored users are defined.
This would give our users more flexibility when using the User Operator in standalone mode.
They would be able to use the User Operator with existing Kafka clusters that already have some other users defined by configuring the ignore pattern to ignore the internal users and not interfere with them while managing other users through the User Operator and `KafkaUser` resources.

## Proposal

A new configuration option will be introduced in the User Operator.
It will be named `STRIMZI_IGNORED_USERS_PATTERN`.
Its default value will be `null`, meaning that no users will be ignored and all will be taken into account.
Users will be allowed to set it to a regular expression pattern to define which users should be ignored.

There will be no new field to configure this option in the `Kafka` CR.
Users will be able to configure it as a custom environment variable in the `.spec.entityOperator.template` section.
Users of the standalone User Operator would be able to configure it directly in the User Operator deployment.

From the User Operator configuration, this option will be passed to the _Simple ACL Operator_, where it will be used instead of the current hardcoded list of ignored users.
The new configuration option will also be added to the _Quotas Operator_ and _Scram Credentials Operator_ classes and used in their `getAllUsers` methods.
The `getAllUsers` method is the part where we collect all _existing_ users (users who have some entry in the ACL, quotas, or SCRAM credential configurations).
And this is the only place where we need to filter out the ignored users.
The pattern will be matched directly against the usernames as provided by Kafka.
So it should match against usernames including the `CN=` for mTLS users and similar.

We will continue to log the ignored ACL rules, quotas, or SCRAM credentials.
But the logging will be done only on the DEBUG level as the ignored users will now be explicitly configured by the users.
Thus, the log messages serve debugging purposes rather than security purposes as with the hardcoded list.

## Conflicts between ignored user and `KafkaUser` resource

It is possible that users would create a `KafkaUser` resource with a name that matches the ignored users pattern.
The reconciliation of such a user would proceed normally, and the User Operator will manage the ACLs, quotas, and credentials for such a user.
This is because it will not be affected by the user not being returned by the `getAllUsers` method (we will know about this user from the custom resources).

However, it might be unpredictable what happens when such a `KafkaUser` resource is deleted.
When the User Operator catches the deletion event and reconciles it, it will delete the user completely.
But if the user is deleted while the User Operator is not running or is otherwise busy (full event queue, ongoing reconciliation for given user, etc.), the deletion of the user will be just ignored, and all ACLs, quotas, and credentials will remain in place.

If a `KafkaUser` resource name matches the ignored users pattern, it is considered a _user configuration error_, and the unpredictable results are considered acceptable behavior.

## Performance impact

For most users, no performance impact is expected as they would not be using this feature at all.
For users who enable this feature to ignore `ANONYMOUS` and `*` usernames (as with the legacy approach), no major impact is expected either.
For users who enable this feature with the standalone User Operator and an existing Kafka cluster not managed by Strimzi, the performance impact will depend on the complexity of the ignore pattern and the number of ACLs, quotas, and credentials that will be ignored.
But in this case, this can be considered as a corresponding cost for this feature.
And given we in general don't expect the User Operator to manage thousands of users, it should not make this feature unusable.

## Migration path

Users who want to keep the existing functionality will be provided with the YAML example how to set the `STRIMZI_IGNORED_USERS_PATTERN` option in the `Kafka` CR or in the standalone User Operator deployment including the exact value to continue ignoring the `ANONYMOUS` and `*` users.

## Documentation changes

When implementing the proposal, the release notes will contain a warning about this change and describe the migration path.
It is also recommended to mention this change in the _What's new_ video or blog post we usually do for new releases.
This should help ensure that users are aware of this change.
The regular documentation will be updated to describe the new feature as well.

## Affected projects

This proposal affects the Strimzi Operators repository.
Code changes are required in the User Operator code and possibly to the configuration classes in the `operator-common` module.
It would also require the documentation to be updated.

## Backwards compatibility

This proposal is not backwards compatible.
Any users relying on the `ANONYMOUS` and `*` usernames being ignored would need to update the User Operator configurations.
While I believe there are some users relying on this feature and thus affected by this change, I expect it to be a very small part of our user base.
And that is why I think this breaking change is worth it to make the majority of our user base more secure.

## Rejected alternatives

### Feature gate

A Feature Gate might allow introducing this breaking change across multiple releases.
However, in this case, a feature gate is not likely to smooth out the transition.
For users not following the release notes properly, it will still deliver the breaking change as a _surprise_.
And the fix is pretty straightforward - updating the User Operator configuration and adding the ignore list.

### Using a list instead of regular expression pattern

Letting users configure a list of ignored users instead of a regular expression pattern would allow us to improve Strimzi security by removing the risks related to the hardcoded list.
But it would make it hard for users to use the standalone User Operator to manage only some users in an existing Apache Kafka cluster.
The list of users would be potentially too big.
And unlike the regular expression pattern, it would not allow ignoring users based on prefixes.

### Applying the ignored users only to ACLs

Using the ignore users list or pattern only for the ACL rules would address the security risks caused by the current solution.
But it would make it harder for users to use the standalone User Operator to manage only some users in an existing Apache Kafka cluster as it would be deleting their custom quotas or SCRAM-SHA-512 users.
