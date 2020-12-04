# Exploratory test module

Exploratory test module contains dialog application, which gives you possibility to create your own test scenarios. 

## Motivation

Every release, we do exploratory testing. We test Strimzi but with this tool it would be very easy to do such a thing. 
Furthermore, it will spare most of our time. 

For brevity, here is one of the scenarios:

1. Deploy ClusterOperator
2. Deploy Kafka
3. Deploy KafkaUser
4. Upgrade ClusterOperator
5. Wait until RU is finished
6. Deploy Kafka cluster with oauth support
7. Deploy oauth clients
8. Deploy Prometheus
9. Deploy Grafana
10. ...

And many more. This is just one scenario, which anyone can create with only choosing allowed choices. 
Note, that in every step you have to manually check if specific component is running. 
For instance, if you choose to 'Deploy Kafka cluster' then you need to list deployment and verify that everything works.

## Abstract Logic

Dialog application log looks like this and in code sample you can see that user already deploy one Cluster operator 
and he has two `only` choices:

1. deploy another `ClusterOperator`
2. deploy `Kafka`. 

```
 [INFO ] 2020-11-27 15:19:03.603 [main] io.strimzi.CreativeExploration - You have following choices with your thoughts:
 [0] - 'ClusterOperator' :1 deployed
 [1] - 'Kafka' :0 deployed
  1
```

He decided to 1, which is `Kafka`. Next step is confirm that `Kafka` is running on specific Kubernetes cluster. If yes, then
you need to type `y`, otherwise `n`. After `y` decision you can see that now user has 4 choices because `Kafka` is deployed.
 
 ```
 * [INFO ] 2020-11-27 15:19:04.105 [main] io.strimzi.CreativeExploration - Is [Kafka] running? [y/n] or you wanna stop testing write STOP
 * y
 * after:
 * [INFO ] 2020-11-27 15:19:04.646 [main] io.strimzi.CreativeExploration - You have following choices with your thoughts:
 * [0] - 'ClusterOperator' :1 deployed
 * [1] - 'Kafka' :1 deployed
 * [2] - 'KafkaUser' :0 deployed
 * [3] - 'KafkaTopic' :0 deployed
```

Moreover application is remember all choices, which means that if user see problem and for instance `Kafka` is not working 
properly then on question `Is [Kafka] running? [y/n]` user will type `n`, which will be stored to wrong test paths. The
path of the test you can image as some list. Example of paths:

```
Wrong paths: [
                [0] -> [1] -> [3],
                [0] -> [1] -> [2]
             ]

Success paths:
            [
                [0]
                [0] -> [1], 
             ]
```

In this case first wrong path, which invoked bug can be interpreted as:
 1. Deploy ClusterOperator
 2. Deploy Kafka
 3. Deploy KafkaTopic
 
 On the other, same applied to success paths. Note, that this is just a very simple example for demonstration reasons.

## Proposal

This proposal suggest to:
* Create module `exploratory-test` in `strimzi-kafka-operator` repository 
* Dependency on our `systemtest` module because of our components such as `Kafka`, `KafkaUser`. Furthermore oauth deployment,
tracing deployment and many more.

##### Advantages:
* reduce time when manually testing some components
* pre-cursor for marathon testing
* gives us report (which combination were buggy)
* everybody can play with it and find bugs
##### Dis-advantages:
* exploratory testing will be limited because of abstraction of the automatic deployment
