
Add support for custom kafka users names with "`.spec.userName`" 

## Current situation

Currently the KafkaUser name is taken from the Kubernetes resource name (which forces specific naming for Kafka users .. no Uppercase letters or underscores)

That's acceptible for new clusters, however there are cases that having the option to define custom KafkaUsers names wil be more efficeint


## Motivaion (Use Case)

_*Please consider the following use case:*_

While prepairing for Migrating Kafka clusters to be managed by Strimzi, migrating the ACLs from the source cluster to the Strimzi clsuter was required.

We've developed a small tool that reads the ACLs from the old cluster & generates Strimzi user-operator CRDs. 

That solved migrating the ACLs, however there is a limitation .. The old cluster may have a Users with names that are not Valid for Kubernetes objects naming .. forcing us to change the migrated users names (Uppercase to lowercase && underscores to dashes)

Here is an Example:

> Waiting for approval to opensource the tool 


```bash
❯ kafka-acls --bootstrap-server <BOOTSTRAP:PORT> --command-config /tmp/config-old-qa.properties --list --principal User:user__m***
```

```ruby
ACLs for principal `User:user__m***`
Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=mob***, patternType=LITERAL)`: 
        (principal=User:user__m***, host=*, operation=WRITE, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=m__o***, patternType=LITERAL)`: 
        (principal=User:user__m***, host=*, operation=READ, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=mo_bi***, patternType=PREFIXED)`: 
        (principal=User:user__m***, host=*, operation=READ, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=mo_in***, patternType=LITERAL)`: 
        (principal=User:user__m***, host=*, operation=WRITE, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=mo_bi***, patternType=LITERAL)`: 
        (principal=User:user__m***, host=*, operation=READ, permissionType=ALLOW) 
```

See the generated KafkaUser resource

```bash
python3 kafka_acl_converter.py --convert-acls-to-strimzi-crd --cluster-name kafka-cluster --user user__m***
```

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: user--m***
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-cluster
    migrated: "true"  # original_user_name --> user__m***
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: "mo_in***"
          patternType: literal
        operations: 
          - Write
        host: "*"
      - resource:
          type: topic
          name: "mob***"
          patternType: literal
        operations: 
          - Write
        host: "*"
      - resource:
          type: topic
          name: "m__o***"
          patternType: literal
        operations: 
          - Read
        host: "*"
      - resource:
          type: group
          name: "mo_bi***"
          patternType: prefix
        operations: 
          - Read
        host: "*"
```



## Proposal

Add support for custom kafka users names with "`.spec.userName`" (similar to the topic-operator `.spec.topicName`) which will allow to migrate the Users with thier exact same name.


