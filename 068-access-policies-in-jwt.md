# Pass Access Policies (ACLs) in JWT

This proposal addresses the possibility of passing access policies in the JWT tokens.

## Current situation

Currently, authorization is supported via [Keycloak](https://www.keycloak.org/).
Keycloak must be deployed, and the identities must be managed there.
This makes using Strimzi in larger systems more complicated.
Most users use their own identity management systems.
In order to integrate it with Strimzi, they would have to duplicate the identities in Keycloak.
This solution doesn't follow the single source of truth architecture.
It also requires to run a redundant service - Keycloak.

Strimzi supports Simple authorization, which is based on the authorizer plugin from Apache Kafka,
and Keycloak authorizer.
It also supports custom authorizers, which can be used to point to the implementation of a custom authorizer class.
There is currently no inbuilt support for JWT-based authorization.
If a user wants to use JWT-based authorization, they have to implement their own authorizer and use the custom authorizer feature.
As the documentation states:

> In addition to the Kafka custom resource configuration, the JAR file containing the custom authorizer class along with its dependencies must be available on the classpath of the Kafka broker.

This proposal aims to add the JWT-based authorizer to Strimzi, so it can be selected in the configuration as
another option apart from the simple and Keycloak ones, without having to implement a custom authorizer.

## Motivation

Having a generic solution is always preferable as it allows for less effort to be maintained.
While we cannot achieve a perfect generic solution due to the nature of OAuth2, and how different
identity providers are implemented, we can strive to create a solution as generic as possible.

Passing additional data in the JWT seems to be supported by the majority of identity providers.
Therefore, we can use this to pass the access policies in the JWT.
This solution will work in most cases and with most identity providers, however it will not
scale to infinity.
It will, however, be decoupled from Keycloak and therefore be more generic.

## Why JWT as an authorization mechanism?

The official [JWT](https://jwt.io/introduction) lists some use cases:

> Authorization: This is the most common scenario for using JWT. Once the user is logged in, each subsequent request will include the JWT, allowing the user to access routes, services, and resources that are permitted with that token. Single Sign On is a feature that widely uses JWT nowadays, because of its small overhead and its ability to be easily used across different domains.

JWT supports custom claims, which can be used to pass additional claims/roles.
For example [Spring Security](https://docs.spring.io/spring-authorization-server/reference/guides/how-to-custom-claims-authorities.html) can pass both claims and roles in the JWT,
which can be used to authorize the user.

All OAuth2 providers support custom claims in the JWT in one way or another.
We have listed the most popular ones in the [Support for roles or claims in popular identity providers](#support-for-roles-or-claims-in-popular-identity-providers) section with links to the documentation
on how custom claims or roles are supported.

The concept of roles can be directly mapped to the concept of access policies in Kafka.

## Proposal

Identity providers support passing custom fields in the JWT token.
Some providers allow the data type to be string, while others allow a list of strings.
There are not many (if any) identity providers that allow a list of objects.

Therefore, in this proposal we allow the ACLs to be passed as either.
e.g. for a list of strings:
```json
{
  "acls": [
    "cluster_x:topic_1:read",
    ":topic2:write"
  ]
}
```

and for a string:
```json
{
  "acls": "cluster_x:topic_1:read,topic2:write"
}
```

The field name is configurable with default value `acls`.

### Using roles to pass ACLs

Sometimes the identity provider supports very fine customization of the JWT,
but this is not always the case.
However, pretty much all identity providers support the concept of roles or claims.
Roles are often used to define the access level of a user or service.
For example, the database service is given the `admin` role, while the frontend service is given the `read` role. 
In the application layer, the role is checked to see if the user has the necessary permissions when the user tries to perform an action.

We can use fine-grained roles to represent ACLs.
Instead of naming the role `admin`, which implicitly implies all permissions, we can name the role `*:topic:*:*` or if we want to limit the permissions - `*:topic:my_topic:read`.
This way we are using the name of the role to encode the permissions the user has,
instead of using a generic name such as `admin` and having the application layer understand what `admin` means.

As an example we can define the roles for a service A in the IDP to be:
```json
"roles": [
  "*:topic:input_topic:read",
  "*:topic:output_topic:write",
],
```
Now when these "roles" are passed in the JWT, we can use them as Kafka ACLs in order to authorize service A.

### ACL Syntax

A single ACL has the syntax `CLUSTER_NAME:RESOURCE_TYPE:RESOURCE_SPEC:PERMITTED_ACTIONS`.
The separator `:` is chosen because it is a common separator in the Unix world.

If the ACLs are passed as a string, they are separated by `,`.
i.e., `CLUSTER_NAME:RESOURCE_TYPE:RESOURCE_SPEC:PERMITTED_ACTIONS,CLUSTER_NAME:RESOURCE_TYPE:RESOURCE_SPEC:PERMITTED_ACTIONS`

- `RESOURCE_TYPE` can be `topic` or `group`, or the shortened versions `t` and `g`.
- `PERMITTED_ACTIONS` is a list of a subset of `read`, `write`, `create`, `delete`, `alter`, `describe`, `cluster_action`, `describe_configs`, `alter_configs`, `idempotent_write`, `create_tokens`, `describe_tokens`, `all`, or the shortened versions `r`, `w`, `c`, `d`, `a`, `de`, `ca`, `dc`, `ac`, `iw`, `ct`, `dt`. The short version for `all` is `*`.
The list items are separated by a `+`.
- `CLUSTER_NAME` is the name of the cluster, and `RESOURCE_SPEC` is the name of the resource (topic/group).
These fields can start or end with `*` to match any prefix/suffix.

### Defaults
If `CLUSTER_NAME` or `RESOURCE_SPEC` is empty, it is considered a wildcard `*`, which matches any cluster or resource.

If `RESOURCE_TYPE` is empty, it takes the default value `topic`.

If `PERMITTED_ACTIONS` is empty, no actions are allowed.

The defaults for `CLUSTER_NAME`, `RESOURCE_SPEC`, and `RESOURCE_TYPE` are chosen for convenience.
The default for `PERMITTED_ACTIONS` is none to prevent accidental access to all resources.

### Examples of ACLs

- `my_cluster:t:topic1:r+w` - allows reading and writing to `topic1` in the cluster `my_cluster`.
- `:::` - denies all actions on all topics in all clusters.
- `:::*` - allows all actions on all topics in all clusters.
- `my_cluster:group:*_app2:read` - allows reading from all groups ending with `_app2` in the cluster `my_cluster`.
- `::edge_*:write+r` - allows writing and reading topics starting with `edge_` in all clusters.

### Implementation details

The new authorizer will be implemented as a new class, similarly to
[KeycloakAuthorizer](https://github.com/strimzi/strimzi-kafka-oauth/blob/229daee85b096804d16e2904c8c0f1add599cc99/oauth-keycloak-authorizer/src/main/java/io/strimzi/kafka/oauth/server/authorizer/KeycloakAuthorizer.java),
making use of the [KeycloakRBACAuthorizer](https://github.com/strimzi/strimzi-kafka-oauth/blob/229daee85b096804d16e2904c8c0f1add599cc99/oauth-keycloak-authorizer/src/main/java/io/strimzi/kafka/oauth/server/authorizer/KeycloakRBACAuthorizer.java).
This is in order to support both KRaft and Zookeeper.

```java
public class TopicAccess
{
  // topic name or pattern
  public final String pattern;
  // allowed operation
  public final AclOperation operation;
}

public class JWTAuthorizer implements ClusterMetadataAuthorizer {
    private StandardAuthorizer delegate;
    private KeycloakRBACAuthorizer singleton;

    @Override
    public void configure(Map<String, ?> configs) {
        // ...
    }

    @Override public List<AuthorizationResult> authorize(AuthorizableRequestContext requestContext, List<Action> actions) {
        KafkaPrincipal principal = requestContext.principal();
        if (!(principal instanceof OAuthKafkaPrincipal)) {
          // simple ACL delegation
        }
        BearerTokenWithPayload token = ((OAuthKafkaPrincipal) principal).getJwt();
        // check token validity
        BearerTokenWithPayload jwt = principal.getJwt();
        ObjectNode payload = jwt.getClaimsJSON();

        // get the acls from the payload
        List<TopicAccess> topicAccesses = extractTopicAccesses(topicAccessesRaw, prefix);
        List<ClusterAccess> clusterAccesses = extractClusterAccesses(clusterAccessesRaw, prefix);

        List<AuthorizationResult> results = new ArrayList<>(actions.size());
        for (Action action : actions) {
          if (checkTopicJwtClaims(topicAccesses, action) || checkClusterJwtClaims(clusterAccesses, action)) {
              results.add(AuthorizationResult.ALLOWED);
          } else {
            results.add(AuthorizationResult.DENIED);
          }
        }
        return results;
    }

    // sketch of how the topic access check could look like
    public static boolean checkTopicJwtClaims(List<TopicAccess> topicAccesses, Action requestedAction) {
    for (TopicAccess t : topicAccesses) {
      switch (requestedAction.resourcePattern().resourceType()) {
          case TOPIC:
              if (matchTopicPattern(requestedAction, t) && checkTopicAccess(t.operation, requestedAction))
                  return true;
              break;
          case CLUSTER: // ...
          case GROUP: // ...
      }
      return false;
    }

    private static boolean checkTopicAccess(AclOperation claimedOperation, Action requestedAction) {
      switch (requestedAction.operation()) {
        case READ:
          return List.of(ANY, ALL, READ).contains(claimedOperation);
        case WRITE:
        // ...
      }
    }

    // ...
}
```

Operations are described in the [Kafka docs](https://kafka.apache.org/documentation/#operations_in_kafka).
Allowing an operation will follow the same logic as in [AclEntry.supportedOperations](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/security/authorizer/AclEntry.scala#L99).

The above code will be supplied with unit tests and documentation explaining how to set up Strimzi with
the new authorizer.
We would also include two example setups.
One using Azure Active Directory B2C and the other with Keycloak.
The setup will be similar for other identity providers, with the main difference being the configuration of the identity provider itself and the `acls` field.

## Scalability

This solution is intended for small to medium-sized clients,
that have a couple 10s of permissions.
Since permissions would be passed as strings, the overhead would be proportional to the number and kind of
permissions.

10s of permissions in the JWT would be very little overhead in terms of computation.
The main overhead is the size of the JWT.
If the average topic name length is 20 characters, with 10 ACLs the overhead would be 200-300 bytes.

Passing 100s of permissions would be too much of an overhead, so it is advised against.
Passing 1000s of permissions is not feasible.

## Affected/not affected projects

This proposal will not affect any other projects.
It will be added as a new feature, an alternative implementation of the Keycloak Authorizer.

## Compatibility

This proposal is backward compatible and will not affect any existing functionality.

## Rejected alternatives

There do not currently exist alternatives apart from the Keycloak Authorizer.
Using another OAuth2 provider is not possible without implementing a new authorizer,
which is tightly coupled with said provider.
While that is an alternative, we either have to let every user implement their own authorizer,
or we have to implement a new authorizer for every OAuth2 provider.
This is not feasible.
Using more generic solutions, such as OpenFGA, overcomplicates the approach to the problem.
Such solutions would need to be implemented on top of OAuth2, while OAuth2 works by default with JWT.

Using RegEx for cluster/topic name matching would be very extensible.
However, it would overcomplicate the solution.
Also, it would make it harder to debug and profile in case the user inputs a wrong or very complex RegEx.
While glob patterns are not as powerful, they are simpler to understand and have consistent performance.

## Support for roles or claims in popular identity providers

Adding custom attributes/roles/groups to the token works as follows in the different auth solutions:

In Azure Active Directory B2C:

The attribute can be a string/list of strings.

- https://learn.microsoft.com/en-us/azure/active-directory-b2c/user-flow-custom-attributes?pivots=b2c-user-flow
- https://learn.microsoft.com/en-us/azure/active-directory-b2c/client-credentials-grant-flow?pivots=b2c-user-flow

In AWS Cognito:

The attribute type can be string/number/bool so probably not an object.
Also, the roles might be fixed to have the aws prefixes.
Supports a list of objects and strings.
Might need more configurable field name/parsing nested structures.

- https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AddCustomAttributes.html
- https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_SchemaAttributeType.html
- https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html

In OneLogin:

Does not currently support the field "values", which is an array. Supports adding roles as a semicolon separated list.

- https://developers.onelogin.com/api-docs/2/api-authorization/list-claims
- https://developers.onelogin.com/api-docs/1/users/set-custom-attribute
- https://developers.onelogin.com/api-docs/2/api-authorization/add-claim

In Google Firebase:

Custom claims are of the form "claim": true, so not just a list of strings, but a list of attributes. I guess this could be used to model the allow/deny kafka rules, or use the same custom string format as for every other auth solution and ignore the true/false value.

- https://firebase.google.com/docs/auth/admin/create-custom-tokens
- https://firebase.google.com/docs/auth/admin/custom-claims

In Auth0:

Can use custom claims as roles/groups.
event.authorization has a field roles which is a list of permissions for the user.
It supports an array of strings.

- https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow/call-your-api-using-the-client-credentials-flow#sample-use-cases
- https://auth0.com/docs/customize/actions/flows-and-triggers/machine-to-machine-flow#m2m-client-credentials
- https://auth0.com/docs/get-started/apis/scopes/sample-use-cases-scopes-and-claims
- https://auth0.com/docs/customize/actions/flows-and-triggers/login-flow/event-object

In Okta:

Group claims could represent roles.
Supports a list of strings

- https://developer.okta.com/docs/guides/customize-tokens-returned-from-okta/main/
- https://developer.okta.com/docs/guides/customize-tokens-groups-claim/main/
