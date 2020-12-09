# Exploratory test module

Exploratory test module contains dialog application, which gives you possibility to create your own test scenarios. 

## Motivation

Every release, we do exploratory testing. We test Strimzi but with this tool it would be very easy to do such a thing. 
This tool also provides `transcript` for scenarios, which invoked bug. In other words, user can easily
share with developers or testers reproducible procedure with all YAMLs executed by user. 
Furthermore, it will spare most of our time. 

For brevity, here is one of the scenarios:

1. TODO: pre-defined installation of CO???
2. Provide Kafka CR YAML
3. Provide KafkaUser CR YAML
6. Provide Kafka CR with oauth support
7. Deploy oauth clients
8. Provide YAMLs for installation of Prometheus
9. Provide YAMLs for installation of Grafana

And many more. This is just one scenario, which anyone can create. 
Note, that in every step you have to manually check if specific component is running. 
For instance, if you choose to 'Provide Kafka CR YAML' then you need to list Kafka CR, Pods and verify that everything works.

### Idea

Module provides two dialog applications, which has a lot in common:
1. BaselineExploratory - dialog application, which executes predefined scenario 
2. CreativeExploratory - dialog application, which executes input YAMLs provided by the user

Moreover, applications remembers all your choices, which means that if user see problem and for instance `Kafka` is not working 
properly then on question `Is [Kafka] running? [y/n]` user will type `n`, which will store this path to `bug` paths. 
 On the other, it has also list of `success` paths.

#### BaselineExploratory 

Baseline exploration consists of several steps:
1. input pre-defined YAMLs = user has to prepare these YAMLs
2. start baseline application
3. application applies first pre-defined YAML
4. (interaction with app) yes/no (is component running?)
5. application applies next pre-defined YAML
6. (interaction with app) yes/no (is component running?)
7. if there is another component to apply go to `5th step` else go to next step 
8. transcript with wrong and success paths (reporting) = user end the end of testing will have report about his work

#### CreativeExploratory

Creative exploration consists of several steps:
1. start exploration application
2. (interaction with app) user has to provide YAML, which he wants to apply
3. (interaction with app) yes/no (is component running?)
4. if there is another component that user wants to apply go to `3th step` else go to next step 
5. transcript with wrong and success paths (reporting) = user end the end of testing will have report about his work

### Reporting

In the last phase of testing user will have transcript of all his decisions with YAMLs. For clarity, here is one of transcript:

```
[
    [
        echo "apiVersion: apps/v1
              kind: Deployment
              metadata:
                name: strimzi-cluster-operator
                labels:
                  app: strimzi
              spec:
                replicas: 1
               ..." 
        | 
        oc apply -f -        
    ],
    [
        echo "apiVersion: kafka.strimzi.io/v1beta1
             kind: Kafka
             metadata:
               name: my-cluster
             spec:
               kafka:
                 version: 2.6.0
                 replicas: 3
             ..."
        | 
        oc apply -f -
    ],
    [
        echo "apiVersion: kafka.strimzi.io/v1beta1
                kind: KafkaUser
                metadata:
                  name: my-user
                  labels:
                    strimzi.io/cluster: my-cluster
                spec:
                ..."
        | 
        oc apply -f -
    ]
    ...
```

As you can see it is list of commands, which is needed to do to reproduce same behavior that user had. In the plain text
it will look like this:

```
======
1.step
======
echo "apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: strimzi-cluster-operator
        labels:
          app: strimzi
      spec:
        replicas: 1
       ..." 
| 
oc apply -f -  
======
2.step
======
echo "apiVersion: kafka.strimzi.io/v1beta1
     kind: Kafka
     metadata:
       name: my-cluster
     spec:
       kafka:
         version: 2.6.0
         replicas: 3
     ..."
| 
oc apply -f -
======
3.step
======
echo "apiVersion: kafka.strimzi.io/v1beta1
        kind: KafkaUser
        metadata:
          name: my-user
          labels:
            strimzi.io/cluster: my-cluster
        spec:
        ..."
| 
oc apply -f -
```

## Proposal

This proposal suggest to:
* Create module `exploratory-test` in `strimzi-kafka-operator` repository 

##### Advantages:
* reduce time when manually testing some components
* pre-cursor for marathon testing
* gives us report (which combination were buggy)
* everybody can play with it and find bugs
##### Dis-advantages:
* TODO: :D
* user has to input syntactically valid YAMLs (sometimes hard? :D )
