# Exploratory test module

The exploratory test contains dialogue application, which gives you the possibility to create your test scenarios. 

## Motivation

Every release, we do exploratory testing and test Strimzi. 
This tool also provides `transcript` for all scenarios that the user tried. In other words, the application stores success and wrong scenarios. These scenarios end the end of exploration are provided to a user. User then can 
share with developers or testers reproducible procedure with all YAMLs executed by the user. 

For brevity, here is one of the scenarios:

1. Provide default configuration of CO
2. Provide Kafka CR YAML
3. Provide KafkaUser CR YAML
6. Provide Kafka CR with OAuth support
7. Deploy OAuth clients
8. Provide YAMLs for installation of Prometheus
9. Provide YAMLs for installation of Grafana

And many more. This is just one scenario, which anyone can create. 
Note, that in every step you have to manually check if a specific component is running. 
For instance, if you choose to 'Provide Kafka CR YAML' then you need to list Kafka CR, Pods and verify that everything works.

### Idea

The module provides two dialogue applications, which has a lot in common:
1. BaselineExploratory - dialogue application, which executes the predefined scenario 
2. CreativeExploratory - dialogue application, which executes input YAMLs provided by the user

Moreover, applications remember all your choices, which means that if the user sees the problem and for instance `Kafka` is not working properly then on the question `Is [Kafka] running? [y/n]` user will type `n`, which will store this path to `bug` paths. 
 On the other, it has also the list of `success` paths.

#### BaselineExploratory 

Baseline exploration consists of several steps:
1. input pre-defined YAMLs = pre-defined examples in repository or user can prepare his own YAMLs
2. start baseline application
3. application applies first pre-defined YAML
4. (interaction with app) yes/no (is component running?)
5. application applies next pre-defined YAML
6. (interaction with app) yes/no (is component running?)
7. if there is another component to apply to go to `5th step` else go to next step 
8. transcript with wrong and success paths (reporting) = user end the end of testing will have a report about his work

#### CreativeExploratory

Creative exploration consists of several steps:
1. start exploration application
2. (interaction with app) the user has to provide YAML, which he wants to apply
3. (interaction with app) yes/no (is component running?)
4. if there is another component that the user wants to apply to go to `3rd step` else go to next step 
5. transcript with wrong and success paths (reporting) = user end the end of testing will have a report about his work

### Reporting

In the last phase of testing user will have a transcript of all his decisions with YAMLs. For clarity, here is one of the transcript:

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

As you can see it is the list of commands, which is needed to do to reproduce the same behaviour that the user had. In the plain text
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

This proposal suggests to:
* Create new repository `exploratory-test` in `strimzi`

##### Advantages:
* reduce time when manually testing some components
* pre-cursor for marathon testing
* gives us report (which combination were buggy)
* everybody can play with it and find bugs
##### Dis-advantages:
* It may lead to the fact that the scenarios will not be original and sometimes it will be tested redundantly
* the user has to input syntactically valid YAMLs
