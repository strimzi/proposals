# Add ability to create Cruise Control REST API users

This proposal is about giving developers the ability to create REST API users for the Cruise Control REST API.
This would allow roles and permissions to be defined to allow developers and third-party applications to access the Cruise Control REST API without having to disable HTTP basic authentication.

## Current situation

At this time, Strimzi creates two Cruise Control API users:

* `admin`      - Used by Strimzi for Cruise Control admin operations.
* `heathcheck` - Used by Strimzi for checking the readiness of the Cruise Control application.

The access to these API users are limited to Strimzi.
So when HTTP basic authentication is enabled for Cruise Control, no one can access the the Cruise Control API other than Strimzi.

## Motivation

There are a couple use cases where developers would want to access the Cruise Control API without disabling API security:

* Debugging Cruise Control in a production environment
* Running the Cruise Control UI alongside a Strimzi-managed Kafka cluster 

In order to allow developers and third-party applications to access the Cruise Control API without needing to disable API security, we would need to provide a method of creating API users.

## Proposal

### Goals

* A method of creating Cruise Control API users.

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

### How API user management is handled elsewhere

Cruise Control is not the only Kafka component which has API users.
For consitency across the code base, we should also consider the following components when designing our solution:

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
These resources are watched and managed by the the Strimzi User Operator, which creates Kafka users based on the descriptions provided by the resources.

For more details, checkout the Strimzi documentation [here](https://strimzi.io/docs/operators/latest/deploying#proc-configuring-kafka-user-str)

#### KafkaConnect users

At this time, Strimzi doesn't offer any method of creating or configuring KafkaConnect API users.
That being said, the direction we take with creating and managing Cruise Control API users could be used as a precedent for its design in the future.

## Alternatives

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

The problem with this approach is that it would add more complexity to the code base since it would require:
* A new Strimzi `CruiseControlUser` custom resource
* A watch on the new `CruiseControlUser` resource
* A reconciler to manage the `CruiseControlUser` resource.

### Developer managed API user configuration

Having the developer manage a single secret with their custom Cruise Control API user configuration would greatly reduce the complexity of the proposed design.
Strimzi could add the contents of this secret to the Cruise Control API user configuration file containing Strimzi's `admin` and `healthcheck` API users.

The schema could look something like this:

```yaml=
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  ...
spec:
  cruiseControl:
    apiUsersConfig:
      secret:
        secretName: cruise-control-api-users-secret
     ...
```

Although this would simplify the overall design it would also shift a lot of the maintanence responsibility to the developer.
The developer would need to create a valid Cruise Control API user configuration and encode it into a secret.

## Rejected alternatives

### Move API user configuration ConfigMap

Having the API users configuration exposed in a `ConfigMap` would significantly reduce the code complexity of configuration management.
That being said, the API users configuration contains sensitive information such as passwords and therefore should not be exposed via a ConfigMap.

## Future work

This proposal aims to set the direction and define the semantics of Cruise Control API users and maybe set the precedent for creating API users for other Kafka components like KafkaConnect.
It is not intended, by itself, to define the complete picture.
Future work _could_ include:

* Configurable API user roles
* Password renewals for generated passwords

