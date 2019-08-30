---
title: DDIA - Ch6 - Partition
date: 2019-08-05 00:00:00
tags: [Partition, Distributed System, DDIA]
categories: [Reading Notes, Designing Data-Intensive Applications]
---

## Abstract  

This blog is my reading notes for _Designing Data-Intensive Applications_, Chapter 06: Partition.

`Partitioning` means splitting a large dataset into smaller subsets, which are then placed on different nodes in a shared-nothing cluster. It is necessary when you have so much data that storing and processing it on a single machine is no longer feasible. Also it has advantages over **scalability**: a large dataset can be distributed across many disks, and the query load can be distributed across many processors.

<!-- more -->

In this blog we will look at the following topics one by one:

* How to partition datasets and avoid hot spot?
* How to rebalance Data?
* How to use secondary indexes in partition?
* How to route request?

## Approaches to Partitioning

We have following ways of partition:

* `Key Range Partition`: assign a continuous range of keys to each partition. Within each partition, keys are kept in sorted order. The advantage is that efficient range queries are possible, but there is a risk of hot spots if the application often accesses keys that are close together in the sorted order. Partitions are typically rebalanced dynamically by splitting the range into two sub-ranges when a partition gets too big.
* `Hash Partition`: a hash function is applied to each key, and a partition owns a range of hashes. Range queries are inefficient, but data are distributed more evenly. When partitioning by hash, it is common to create a fixed number of partitions in advance, to assign several partitions to each node, and to move partitions from one node to another when nodes are added or removed. Dynamic partitioning can also be used.
* `Hybrid Partition`: a table can be declared with a compound primary key consisting of several columns. Only the first part of that key is hashed to determine the partition, but the other columns are used as a concatenated index for sorting the data. This approach enables an elegant data model for one-to-many relationships. For example, on a social media site, one user may post many updates. If the primary key for updates is chosen to be (user_id, update_timestamp), then you can efficiently retrieve all updates made by a particular user within some time interval, sorted by timestamp.

If the partitioning is unfair, so that some partitions have more data or queries than others, we call it skewed. A partition with disproportionately high load is called a `hot spot`. Today, most data systems are not able to automatically compensate for such a highly skewed workload, so it’s the responsibility of the application to reduce the skew.

One simple technique is that, if one key is known to be very hot, add a random number to the beginning or end of the key can split the writes to the key evenly across different keys, i.e., distributed to different partitions. The disadvantage is that this technique now requires additional bookkeeping and reading the data now need to query from all keys and combine them.

## Rebalancing

`Rebalancing` is the process of moving load from one node in the cluster to another, which is called for the following changes: query throughput increases, dataset size increases, node fails. Usually it is expected to meet some minimum requirements:

* After rebalancing, the load should be shared fairly between the nodes in the cluster.
* While rebalancing is happening, the database should continue accepting reads and writes.
* No more data than necessary should be moved between nodes, to make rebalancing fast and to minimize the network and disk I/O load.

We also have 3 approaches to do that:

* `Fixed number of partitions`: can be used if the dataset is not highly variable: create many more partitions than there are nodes, and assign several partitions to each node. If a node is added to the cluster, the new node can steal a few partitions from every existing node until partitions are fairly distributed once again. In this partition, the number of partitions is usually fixed when the database is first set up and not changed afterward. Also, partition splitting is usually not implemented. **The number of partitions configured at the outset = the maximum number of nodes * number of partitions per node**, so you need to choose it high enough to accommodate future growth. Used in **Riak**, **Elasticsearch**, **Couchbase**, and **Voldemort**
* `Dynamic partitioning proportional to the size of the dataset`: When a partition grows to exceed a configured size, it is split into two partitions so that approximately half of the data ends up on each side of the split. Conversely, if lots of data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent partition. In the initial state when the dataset is small or empty, an initial set of partitions to be configured. Used by **HBase** and **MongoDB**
* `Dynamic partitioning proportional to the number of nodes`: When a new node joins the cluster, it randomly chooses a fixed number of existing partitions to split, and then takes ownership of one half of each of those split partitions while leaving the other half of each partition in place. The randomization can produce unfair splits, but when averaged over a larger number of partitions, the new node ends up taking a fair share of the load from the existing nodes. Used in Cassandra and Ketama.

Fully automated rebalancing can be convenient but unpredictable. Rebalancing is an expensive operation, if it is not done carefully, this process can overload the network or the nodes and harm the performance of other requests while the rebalancing is in progress. In combination with automatic failure detection, other nodes can conclude an overloaded node to be dead and automatically rebalance the cluster to move load away from it, which may cause a cascading failure.

## Secondary index

since dataset is partitioned by primary key, secondary indexes don’t map neatly to partitions. Thus, many key-value stores have avoided secondary indexes because of their added implementation complexity, but some have started adding them because they are so useful for data modeling. There are two methods:

* `Document-partitioned indexes`: each partition is completely separate and maintains its own **local secondary indexes**, covering only the documents in that partition. This means that only a single partition needs to be updated on write, but a read of the secondary index requires a scatter/gather across all partitions, which is expensive.

* `Term-partitioned indexes`: construct a **global secondary index** that covers data in all partitions. The global index is also partitioned to avoid bottleneck, but it can be partitioned differently from the primary key index. When a document is written, a distributed transaction across all partitions that need to be updated is required, which is expensive. However, a read can be served from a single partition and is efficient.

## Request Routing

When a client wants to make a request, how does it know **which node to connect to**? On a high level, there are a few different approaches to this problem.

* Allow clients to contact any node. If that node does not own the partition, it forwards the request to the appropriate node
* Send all requests from clients to a routing tier first, which determines the node that should handle each request and forwards it accordingly. This routing tier does not itself handle any requests; it only acts as a partition-aware load balancer.
* Require that clients be aware of the partitioning and the assignment of partitions to nodes, thus they can connect directly to the appropriate node without any intermediary.

In all cases, the key problem is: **how does the component making the routing decision and learn about changes in the assignment of partitions to nodes**? This is an instance of a more general problem called `service discovery` Any piece of software that is accessible over a network has this problem, especially if it is aiming for high availability.

Many distributed data systems rely on a separate `coordination service` such as ZooKeeper to keep track of this cluster metadata. Each node registers itself in ZooKeeper, and ZooKeeper maintains the authoritative mapping of partitions to nodes. Other actors, such as the routing tier or the partitioning-aware client, can subscribe to this information in ZooKeeper. Whenever a partition changes ownership, or a node is added or removed, ZooKeeper notifies the routing tier so that it can keep its routing information up to date.

Or the system can use a `gossip protocol` among the nodes to disseminate any changes in cluster state. Requests can be sent to any node, and that node forwards them to the appropriate node for the requested partition. Cassandra and Riak adopt this way.

## Reference

> [1] _Designing Data-Intensive Applications_, Chapter 06: Partition
