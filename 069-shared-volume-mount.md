# Shared Volume Mount Changes

The proposed update to the Strimzi Kafka Operator introduces management of shared volume mounts for kafka connect cluster, 
enhancing task resilience across pods. It allows custom specification of the container directory for storage volume mounting 
and defines storage access modes.

## Current situation

In the existing setup, Strimzi Operator allows mounting of storage volumes for Apache Kafka and Zookeeper pods.  
However, it currently faces a limitation in terms of flexibility when managing shared volume mounts across kafka connect cluster.
It only allows mounting of configmaps and secrets. So if there are any files written to disk (by sink file connectors) then will  be lost
if the pod restarts. This limits task resiliency in distributed environments, which is a critical aspect of maintaining high availability and fault tolerance in such systems.

## Motivation

The primary motivation for this proposed enhancement to the Strimzi Kafka Operator is to ensure uninterrupted access to data across all kafka connect pods in a distributed cluster. In the current setup, if a pod goes down, the data stored in its mount path becomes inaccessible, potentially disrupting operations. By enabling shared volume mounts across all pods, we can ensure that even if one pod is down, the data remains accessible from other pods. This feature is crucial for maintaining high availability and fault tolerance, thereby significantly improving the resilience and efficiency of tasks of connectors. Hence, this proposed change is not just an improvement, but a necessity for robust and seamless data management in distributed kafka connect clusters.
 
Support for resilliency for sink file connnectors: Mounting volumes will allow a task that gets reallocted to a different pod to continue processing the file. Also allows for exactly once  sink file connectors to store their offsets to file without them being lost in case  of pod restart.
Support for Pod Running Logs: Ability to dump kafka connect logs to storage volume instead of just stdout.
GC Logs Management: Implement a JVM option to output GC logs to a file instead of stdout when GC logging is enabled. This will help in identifying JVM hangs and keep stdout uncluttered.
Additional diagnostics: Explore the possibility of mounting additional volumes into the Kafka Connect pods beyond ConfigMaps or Secrets. This could facilitate writing event data to parquet files for later analysis.

## Proposal

This change introduces enhancements to the Strimzi Kafka Operator for managing shared volume mounts, improving task resiliency across nodes. Key changes include:

1) **mountPath:** A string property to specify the container directory for mounting the storage volume. While the default is /var/lib/kafka/data, this allows for custom path specification. By default, it will try to mount read write many storage if accessMode is not specified.

2) **accessMode:** An optional string property defining the storage access mode, determining permissions like read-only or read-write.

These changes, tested successfully with Read-Write-Many on Ceph-block storage and NFS, provide greater control over volume configuration and usage, improving operator performance in distributed environments. 

**Sample Test**

 - This test case is to test the strimzi operator customization that allows mounting of shared volume across kafka-connect pods 
 - Ensure that files created in shared volume in one pod are visible to the other pods

**kafkaConnect.yaml**
```
    storage:
      type: persistent-claim
      size: 1Gi
      class: rook-cephfs
      deleteClaim: false
      mountPath: /mnt/shared
    template:
      pod:
        securityContext:
          runAsNonRoot: false
          fsGroup: 1001
```

**Outcome**

-  After deploying all the necessary charts and kafka-connect with the above yaml, ensure only one pvc data-odf-connect-cluster-connect is created, mounted and shared by all the connect pods

```
    $ kubectl get pvc
    NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    data-odf-cluster-kafka-0           Bound    pvc-7a09ed64-2216-48fa-b040-65ecec708472   2Gi        RWO            local-path     143m
    data-odf-cluster-kafka-1           Bound    pvc-4385d70e-da6a-4ed5-b52b-527ae2ea4feb   2Gi        RWO            local-path     143m
    data-odf-cluster-kafka-2           Bound    pvc-5b6cc835-529a-4265-9a42-b46403fbe795   2Gi        RWO            local-path     143m
    data-odf-cluster-zookeeper-0       Bound    pvc-5f505af1-077b-438b-a9fb-3974a7dae9d7   2Gi        RWO            local-path     143m
    data-odf-cluster-zookeeper-1       Bound    pvc-5177af7e-6cc9-4993-bcad-42ce5a422fdc   2Gi        RWO            local-path     143m
    data-odf-cluster-zookeeper-2       Bound    pvc-27e9587f-0ead-4e08-85a9-1177076be843   2Gi        RWO            local-path     143m
    data-odf-connect-cluster-connect   Bound    pvc-1f2eadb0-e2be-4277-9bb4-b2f5dfb8dcdf   3Gi        RWX            rook-cephfs    143m
    sftp-output-file-odf-sftp-0        Bound    pvc-c11a68fa-5865-4310-9194-624a3942b872   100Gi      RWO            rook-cephfs    20h
```

- Get a terminal to one of the kafka-connect pod and create a file at /mnt/shared as below
```
    $ kubectl exec -it odf-connect-cluster-connect-7fbd9589c-d4tx6 -- bash
        [kafka@odf-connect-cluster-connect-7fbd9589c-d4tx6 kafka]$ cd /mnt/shared/
        [kafka@odf-connect-cluster-connect-7fbd9589c-d4tx6 shared]$ ls -lah
        total 0
        drwxrwsr-x 2 root  1001  1 Aug 23 09:27 .
        drwxr-xr-x 1 root  root 20 Aug 23 09:22 ..
        -rw-r--r-- 1 kafka 1001  0 Aug 23 09:27 testFile
```

- Get a terminal to another one of the kafka-connect pod and ensure the created file is visible
```
    $ kubectl exec -it odf-connect-cluster-connect-7fbd9589c-d4tx6 -- bash
        [kafka@odf-connect-cluster-connect-7fbd9589c-d4tx6 kafka]$ cd /mnt/shared/
        [kafka@odf-connect-cluster-connect-7fbd9589c-d4tx6 shared]$ ls -lah
        total 0
        drwxrwsr-x 2 root  1001  1 Aug 23 09:27 .
        drwxr-xr-x 1 root  root 20 Aug 23 09:22 ..
        -rw-r--r-- 1 kafka 1001  0 Aug 23 09:27 testFile
```

## Affected/not affected projects

The proposed enhancement to the Strimzi Kafka Operator will primarily affect projects that rely on distributed environments and require high availability and fault tolerance. These projects will benefit from the improved management of shared volume mounts across nodes. However, projects that operate in a single-node environment or do not rely on shared volume mounts for their operations will not be directly affected by this proposal. It’s important to note that while not all projects may be directly affected, the overall robustness and efficiency brought about by this feature can indirectly benefit the entire Strimzi organisation.

## Compatibility

The proposed enhancement to the Strimzi Kafka Operator has been designed with both future and backward compatibility in mind. For future compatibility, the feature is built to be flexible and adaptable to accommodate potential changes or additions to the storage and access modes. This ensures that the feature remains relevant and useful as the system evolves. For backward compatibility, the feature maintains the existing functionality and behavior of the operator when the new options are not specified. This means that existing projects can continue to operate without any changes, ensuring a smooth transition and minimizing disruption to current operations..

## Rejected alternatives

While formulating this proposal, several alternatives were considered but ultimately rejected for various reasons:

- **Using Local Storage**: Initially, the idea of using local storage for each pod was considered. However, this was rejected because it would not allow data sharing across pods, which is a crucial requirement for ensuring high availability and fault tolerance in distributed systems.
- **Static Volume Mounts**: Another alternative was to use static volume mounts with predefined paths. This was rejected due to its lack of flexibility. It would not allow custom specification of the container directory for storage volume mounting, which is necessary for accommodating different system configurations and use cases.
- **Single Access Mode**: The proposal initially considered having a single, fixed access mode for all storage volumes. This was rejected because it would not provide the necessary flexibility for different use cases. Some scenarios may require read-only access, while others may need read-write access.

These alternatives were rejected because they did not meet the key requirements of flexibility, adaptability, and efficiency necessary for managing shared volume mounts in distributed environments.