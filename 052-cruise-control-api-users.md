# Add ability to create Cruise Control REST API users

This proposal is about giving developers the ability to create REST API users for the Cruise Control REST API.
This would allow roles and permissions to be defined to allow developers and third-party applications to access the Cruise Control REST API without having to disable HTTP basic authentication.

## Current situation

At this time, Strimzi creates two Cruise Control API users:

* `admin`      - Used by Strimzi for Cruise Control admin operations.
* `heathcheck` - Used by Strimzi for checking the readiness of the Cruise Control application.

Access to these API users is limited to Strimzi.
Since HTTP basic authentication is enabled for Cruise Control by default, no one can access the Cruise Control API other than Strimzi unless HTTP basic authentication is explicitly disabled.

### How API user management is handled elsewhere

Cruise Control is not the only Kafka component which has API users.
Although this proposal is strictly about Cruise Control API users, for consistency across the code base, we also considered the following components when designing our solution:

* Kafka users
* KafkaConnect users

#### Kafka users

In Strimzi, Kafka users are managed via custom resources.

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user-1
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      ...
```
These resources are watched and managed by the Strimzi User Operator, which creates Kafka users based on the descriptions provided by the resources.

For more details, checkout the Strimzi documentation [here](https://strimzi.io/docs/operators/latest/deploying#proc-configuring-kafka-user-str)

#### KafkaConnect users

At this time, Strimzi doesn't offer any method of creating or configuring KafkaConnect API users.
That being said, the direction we take with creating and managing Cruise Control API users could be used as a precedent for its design in the future.

The reason behind the apparent inconsistency between KafkaUser management and this alternative user management pattern is because we expect there to be few users for Cruise Control and Kafka Connect, because they are control APIs for their respective systems. 
In contrast, we expect there to be many Kafka users because Kafka is the data API and we expect each Kafka application will use its own principal.

## Motivation

There are a couple use cases where developers would want to access the Cruise Control API without disabling API security:

* Monitoring a Strimzi-managed Kafka cluster with the Cruise Control UI.
* Gathering Cruise Control specific statistical information that is not currently available via Strimzi Operator or Cruise Control sensor metrics e.g. detailed information surrounding cluster and partition load and user tasks.
* Debugging Cruise Control in a secured environment.

At this time, Strimzi Cruise Control integration offers limited access to Cruise Control data and features.
This limited access protects developers from performing cluster operations that could interfere with Strimzi Operator operations and cause damage to their clusters.
This limited access also prevents developers from accessing harmless monitoring data and enabling third party applications such as the Cruise Control UI which could make monitoring their Cruise Control data easier.

In order to allow developers and third-party applications to access the Cruise Control API without needing to disable API security, we would need to provide a method of creating API users.

## Proposal

### Goals

* A method of creating Cruise Control API users.

### Developer managed API user configuration

The proposal is to incorporate a developer-managed approach for configuring a single secret with customized Cruise Control API user configuration.

Cruise Control's authorization credential files adhere to [Jetty's HashLoginService's file format](https://eclipse.dev/jetty/javadoc/jetty-10/org/eclipse/jetty/security/HashLoginService.html): `username: password [,rolename ...]`.

By default Cruise Control defines three roles: `VIEWER`, `USER` and `ADMIN`.
```
VIEWER role: has access to the most lightweight kafka_cluster_state, user_tasks and review_board endpoints.
USER role: has access to all the GET endpoints except bootstrap and train.
ADMIN role: has access to all endpoints.
```
This proposal is for supporting the `USER` and `VIEWER` roles only.
For more details on these roles checkout the [Cruise Control Wiki](https://github.com/linkedin/cruise-control/wiki/Security#authorization)

So a developer could create a Cruise Control API authentication credentials file named `cruise-control-auth.txt`, with the following content:
```
userOne: passwordOne, USER
userTwo: passwordTwo, VIEWER
```

Then the developer can use this file to create a secret using the following command:
```
kubectl create secret generic cruise-control-api-users-secret  --from-file=key=cruise-control-auth.txt
```

The command would create a secret:
```yaml=
apiVersion: v1
kind: Secret
metadata:
  name: cruise-control-api-users-secret
type: Opaque
data:
  key: dXNlck9uZTogcGFzc3dvcmRPbmUsIFVTRVIKdXNlclR3bzogcGFzc3dvcmRUd28sIFZJRVdFUgo=
```

The developer could then reference the secret in a `spec.cruiseControl.apiUsers` section of the `Kafka` resource.

The schema would look like this:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  ...
spec:
  cruiseControl:
    apiUsers:
      type: hashloginservice (1)
      valueFrom: (2)
        secretKeyRef:
          name: cruise-control-api-users-secret
          key: key
     ...
```
(1) A `type` field is added here to describe the format of the data and for configuration flexibility in the future. 
This gives us the option to add different data format types in the future. 
In this example we use Jetty's HashLoginService format since that is the format which Cruise Control uses for its API user configuration.

(2) The `valueFrom` construct allows us to add more sources in the future if needed. 
This is also a pattern used in other Strimzi APIs already, for example Strimzi's logging configuration, password configuration in KafkaUser resources, metrics configuration, and more.

Strimzi would decode and use the contents of this secret to populate the Cruise Control API auth credentials file which would be mounted and used by the Cruise Control pod.

Its worth noting that if a developer were to provide a corrupt configuration file - one which specifies the `ADMIN` role or an incorrect format, the Strimzi Operator would ignore the custom configuration and log a warning.

Although this would simplify the overall design it would also shift a lot of the maintanence responsibility to the developer.
The developer would need to create a valid Cruise Control API user configuration and encode it into a secret.
That being said, this is still a reasonable amount of responsibility to leave to the developer.
This is because accessing the Cruise Control API directly is not the primary way of interacting with Cruise Control for a Strimzi-managed Kafka cluster and is expected to be used only by advanced developers in special use-cases.
However, with proper supporting documentation, this new feature will give developers a greater degree of control and assistance with special use-cases.

### Risks

Direct calls to the Cruise Control API like the write operations enabled by the `ADMIN` role could interfere with Strimzi Operator operations and cause damage to a Strimzi-managed Kafka cluster.

Supporting developers to create Cruise Control API users with non-`ADMIN` roles like `USER` and `VIEWER` roles would mitigate this risk completely.
This is because the `USER` and `VIEWER` roles are read-only roles and would not conflict with Strimzi cluster operations.
So this proposal is for enabling the `USER` and `VIEWER` roles only.

## Rejected alternatives

### Supporting `ADMIN` role

If we were to support developers to create Cruise Control API users with the `ADMIN` role, those users would have access to potentially destructive cluster operations.
For example, an API user with the ADMIN role would be able to execute a rebalance operation while the Strimzi Operator is in the middle of performing a rolling update of the Kafka brokers.

For the above reasons, this proposal advocates for disabling the creation of API users with `ADMIN` roles for now.

We can always enable the `ADMIN` role at a later date with a caveat in the documentation so that advanced developers could experiment with Cruise Control write operations at their own risk.

### Cruise Control user custom resource

Following the pattern used for Kafka users, we could have `CruiseControlUser` resources.

This would make the management of Cruise Control API users consitent with how we manage Kafka users

The spec could look somthing like this:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: CruiseControlUser
metadata:
  name: my-user-1
  labels:
    strimzi.io/cluster: my-cluster
spec:
  role: USER (3)
  password:
    valueFrom:
      secretKeyRef:
        name: my-secret (1)
        key: my-password (2)
```

(1) The name of the secret containing the predefined password.

(2) The key for the password stored inside the secret.

(3) By default Cruise Control defines three roles: `VIEWER`, `USER` and `ADMIN`. 
For more details checkout the [Cruise Control Wiki](https://github.com/linkedin/cruise-control/wiki/Security#authorization)
```
VIEWER role: has access to the most lightweight kafka_cluster_state, user_tasks and review_board endpoints.
USER role: has access to all the GET endpoints except bootstrap and train.
ADMIN role: has access to all endpoints.
```

The problem with this approach is that it would add significant complexity to the code base since it would require:
* A new Strimzi `CruiseControlUser` custom resource
* A watch on the new `CruiseControlUser` resource
* A reconciler to manage the `CruiseControlUser` resource.

Given that accessing the Cruise Control API directly is not the primary way of interacting with Cruise Control for a Strimzi-managed Kafka cluster and is expected to be used only by advanced users in special use-cases, this method would not be worth the complexity that it would add to the code base.

### Add API user list to Cruise Control spec 

For adding Cruise Control API users, we could update the Cruise Control spec with an `apiUsers` section.
In this `apiUsers` section, developers could provide a list of API users by name and security role.

The schema could look something like this:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  ...
spec:
  cruiseControl:
    apiUsers:
      - name: user
        role: USER (3)
     ...
```

When defined like above, Strimzi would generate a Secret containing a password for the specified API user.

In addition to the above, we could allow developers to supply a custom password per specified API user via a Secret like this:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  ...
spec:
  cruiseControl:
    apiUsers:
      - name: user
        role: USER (3)
        password:
          valueFrom:
            secretKeyRef:
              name: my-secret (1)
              key: my-password (2)
     ...
```

(1) The name of the secret containing the predefined password.

(2) The key for the password stored inside the secret.

(3) By default Cruise Control defines three roles: `VIEWER`, `USER` and `ADMIN`. 
For more details checkout the [Cruise Control Wiki](https://github.com/linkedin/cruise-control/wiki/Security#authorization)
```
VIEWER role: has access to the most lightweight kafka_cluster_state, user_tasks and review_board endpoints.
USER role: has access to all the GET endpoints except bootstrap and train.
ADMIN role: has access to all endpoints.
```

Strimzi would then aggregate the Cruise Control API auth credentials into a centralized configuration, then store it in a secret where it would be mounted and used by Cruise Control.

Given that accessing the Cruise Control API directly is not the primary way of interacting with Cruise Control for a Strimzi-managed Kafka cluster and is expected to be used only by advanced users in special use-cases, this method would not be worth the complexity that it would add to the code base.

## Future work

This proposal aims to set the direction and define the semantics of Cruise Control API users and maybe set the precedent for creating API users for other Kafka components like KafkaConnect.
It is not intended, by itself, to define the complete picture.
Future work _could_ include:

* Configurable API user roles
