# bare-k8s-redis-cluster [![Build Status](https://travis-ci.org/fktkrt/bare-k8s-redis-cluster.svg?branch=master)](https://travis-ci.org/fktkrt/bare-k8s-redis-cluster)
Deploying a StatefulSet Redis Cluster on K8s cluster, without dynamic volume provisioning, helm charts or redis-operator.

### Introduction

Inspired by:

https://redis.io/topics/cluster-tutorial 

https://rancher.com/blog/2019/deploying-redis-cluster/

It includes Travis-CI build and test workflow with KinD (Kubernetes in Docker), see this link for details: https://github.com/kubernetes-sigs/kind

### Basic concepts

Redis Cluster data sharding

Redis Cluster does not use consistent hashing, but a different form of sharding where every key is conceptually part of what we call an hash slot.

There are 16384 hash slots in Redis Cluster, and to compute what is the hash slot of a given key, we simply take the CRC16 of the key modulo 16384.

Every node in a Redis Cluster is responsible for a subset of the hash slots, so for example you may have a cluster with 3 nodes, where:

    Node A contains hash slots from 0 to 5460.
    Node B contains hash slots from 5461 to 10922.
    Node C contains hash slots from 10923 to 16383.

This allows to add and remove nodes in the cluster easily. For example if I want to add a new node D, I need to move some hash slot from nodes A, B, C to D. Similarly if I want to remove node A from the cluster I can just move the hash slots served by A to B and C. When the node A will be empty I can remove it from the cluster completely.

Because moving hash slots from a node to another does not require to stop operations, adding and removing nodes, or changing the percentage of hash slots hold by nodes, does not require any downtime.

Redis Cluster supports multiple key operations as long as all the keys involved into a single command execution (or whole transaction, or Lua script execution) all belong to the same hash slot. The user can force multiple keys to be part of the same hash slot by using a concept called hash tags.

Hash tags are documented in the Redis Cluster specification, but the gist is that if there is a substring between {} brackets in a key, only what is inside the string is hashed, so for example this{foo}key and another{foo}key are guaranteed to be in the same hash slot, and can be used together in a command with multiple keys as arguments.

Redis Cluster master-slave model

In order to remain available when a subset of master nodes are failing or are not able to communicate with the majority of nodes, Redis Cluster uses a master-slave model where every hash slot has from 1 (the master itself) to N replicas (N-1 additional slaves nodes).

In our example cluster with nodes A, B, C, if node B fails the cluster is not able to continue, since we no longer have a way to serve hash slots in the range 5461 to 10922.

However when the cluster is created (or at a later time) we add a slave node to every master, so that the final cluster is composed of A, B, C that are masters nodes, and A1, B1, C1 that are slaves nodes, the system is able to continue if node B fails.

Node B1 replicates B, and B fails, the cluster will promote node B1 as the new master and will continue to operate correctly.

However note that if nodes B and B1 fail at the same time Redis Cluster is not able to continue to operate.

![architecture](redis-cluster-architecture.png)

### Define a default StorageClass resource

See `default-sc.yml` for details.

### Define your PersistentVolumes

If you are not using a cloud provider with dynamic volume provisioning support, you should create a PV for each of your Redis nodes. 
You can configure an existing folder on your Kubernetes node, choose its size, define your StorageClass and you are good to go.

See `pv.yml` for details.

### Define your PersistentVolumeClaim

You need a `PersistentVolumeClaim` matching your `PersistentVolumes`.

See `pvc.yml` for details.

### Setting up a ConfigMap, StatefulSet and a Service

These steps are covered in `deploy-redis-cluster.yml`.

### Deploying the aforementioned objects
```
$ kubectl apply -f default-sc.yml

$ kubectl apply -f pv.yml
$ kubectl apply -f pvc.yml

$ kubectl apply -f deploy-redis-cluster.yml

```
### Testing your PersistentVolumes

```
user@docker:~/k8s-redis-cluster$ kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
pv-volume-0   150Mi      RWO            Retain           Bound    default/data-redis-cluster-0   default                 6m3s
pv-volume-1   150Mi      RWO            Retain           Bound    default/data-redis-cluster-1   default                 2m45s
pv-volume-2   150Mi      RWO            Retain           Bound    default/data-redis-cluster-2   default                 2m11s
pv-volume-3   150Mi      RWO            Retain           Bound    default/data-redis-cluster-3   default                 100s
pv-volume-4   150Mi      RWO            Retain           Bound    default/data-redis-cluster-4   default                 44s
pv-volume-5   150Mi      RWO            Retain           Bound    default/data-redis-cluster-5   default                 14s
```

### Creating the Redis Cluster with 3 master - 3 slave

Creating a Redis Cluster with `--cluster-replicas 1` creates 3 master and 3 slaves from 6 nodes.

```
$ kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
```

Sample output:

```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.244.0.32:6379 to 10.244.0.29:6379
Adding replica 10.244.0.33:6379 to 10.244.0.30:6379
Adding replica 10.244.0.34:6379 to 10.244.0.31:6379
M: 4e636e5c38ce0f7237bebd49a6aadddbc77276f2 10.244.0.29:6379
   slots:[0-5460] (5461 slots) master
M: 18f0eb64e1e1339de1bb1ceef4e69714078a9ffb 10.244.0.30:6379
   slots:[5461-10922] (5462 slots) master
M: 293b9137a515ef92d2f65607809bfa611d55c5f0 10.244.0.31:6379
   slots:[10923-16383] (5461 slots) master
S: 51a3346b81ffb5f2cd984f66be9dbfbe8138909f 10.244.0.32:6379
   replicates 4e636e5c38ce0f7237bebd49a6aadddbc77276f2
S: 9acc94516d466e61e30c436171914a94a6816b3a 10.244.0.33:6379
   replicates 18f0eb64e1e1339de1bb1ceef4e69714078a9ffb
S: 10d7a62ced7dada05e2944b35c8b64dcb5de8218 10.244.0.34:6379
   replicates 293b9137a515ef92d2f65607809bfa611d55c5f0
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 10.244.0.29:6379)
M: 4e636e5c38ce0f7237bebd49a6aadddbc77276f2 10.244.0.29:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 293b9137a515ef92d2f65607809bfa611d55c5f0 10.244.0.31:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 9acc94516d466e61e30c436171914a94a6816b3a 10.244.0.33:6379
   slots: (0 slots) slave
   replicates 18f0eb64e1e1339de1bb1ceef4e69714078a9ffb
S: 51a3346b81ffb5f2cd984f66be9dbfbe8138909f 10.244.0.32:6379
   slots: (0 slots) slave
   replicates 4e636e5c38ce0f7237bebd49a6aadddbc77276f2
S: 10d7a62ced7dada05e2944b35c8b64dcb5de8218 10.244.0.34:6379
   slots: (0 slots) slave
   replicates 293b9137a515ef92d2f65607809bfa611d55c5f0
M: 18f0eb64e1e1339de1bb1ceef4e69714078a9ffb 10.244.0.30:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

Checking cluster info

```
user@docker:~/k8s-redis-cluster$ kubectl exec -it redis-cluster-0 -- redis-cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:157
cluster_stats_messages_pong_sent:139
cluster_stats_messages_sent:296
cluster_stats_messages_ping_received:134
cluster_stats_messages_pong_received:157
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:296
```

You can test the roles of the nodes with the following for cycle:

```
$ for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -- redis-cli role; echo; done
```

```
redis-cluster-0
master
6776
10.244.0.32
6379
6776

redis-cluster-1
master
6776
10.244.0.33
6379
6776

redis-cluster-2
master
6776
10.244.0.34
6379
6776

redis-cluster-3
slave
10.244.0.29
6379
connected
6776

redis-cluster-4
slave
10.244.0.30
6379
connected
6776

redis-cluster-5
slave
10.244.0.31
6379
connected
6776
```


### Setting up external access

Create a service for external access.
See details in `redis-ext-service.yml`

Check your service with `kubectl describe svc redis-ext-service`

```
Name:                     redis-ext-service
Namespace:                default
Labels:                   app=redis-cluster
Annotations:              kubectl.kubernetes.io/last-applied-configuration:
                            {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"redis-cluster"},"name":"redis-ext-service","namespace":"...
Selector:                 app=redis-cluster
Type:                     NodePort
IP:                       10.110.100.64
Port:                     redisport  6379/TCP
TargetPort:               6379/TCP
NodePort:                 redisport  30036/TCP
Endpoints:                10.244.0.30:6379,10.244.0.31:6379,10.244.0.32:6379 + 3 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

You should see the IPs of your nodes as Endpoints.

### Test your Redis Cluster from outside your cluster

You have to use the `-c` flag when using redis-cli, see the following link for details: https://redis.io/topics/cluster-tutorial

```
$ redis-cli -c -h <your-kubernetes-node-ip> -p 30036
```
