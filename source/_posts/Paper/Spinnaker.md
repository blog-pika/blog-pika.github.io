---
title: Spinnaker
date: 2019-08-19 20:12:09
tags: [Paxos, 6.824]
categories: [Paper]
---

This blog is my reading notes for Spinnaker. We can learn from this paper how to use consensus algorithm (Paxos) to do replication in database.

<!-- more -->

## Pros

Spinnaker features key-based range partitioning, 3-way replication, and a transactional get-put API with the option to choose either strong or timeline consistency on reads. Spinnaker will be available for reads and writes as long a majority of its replicas are alive.  The performance can be competitive with alternatives that provide weaker consistency guarantees. Compared to an eventually consistent datastore, Spinnaker can be as fast or even faster on reads and only 5% to 10% slower on writes.

## Cons

Only suitable in single datacenter and cannot tolerate network partition

## Data Model and API

Data Model is almost the same as BigTable. Besides, each API call is executed as a single-operation transaction.

```cpp
get(key, colname, consistent)
// Read a column value and its version number from a row. The setting of the ‘consistent’ ﬂag is used to choose the consistency level. Setting it to ‘true’ chooses strong consistency, and the latest value is always returned. Setting it to ‘false’ chooses timeline consistency, and a possibly stale value is returned in exchange for better performance.

put(key, colname, colvalue)
// Insert a column value into a row.

delete(key, colname)
// Delete a column from a row.

conditionalPut(key, colname, value, v)
// Insert a new column value into a row only if the column’s current version number is equal to ‘v’. Otherwise, an error is returned.

conditionalDelete(key, colname, v)
//Like conditional put but for delete.
```

## Architecture

The architecture of Spinnaker is also similar to BigTable. However, they use different ways to make logs and data fault-tolerant:

* BigTable: store data and logs on GFS
* Spinnaker: use `Paxos` to replicate logs to multiple nodes

### Replication

Like Bigtable, Spinnaker distributes the rows of a table across its cluster using range partitioning. Each node is assigned a base key range, which is replicated on the next N − 1 nodes. Each group of nodes involved in replicating a key range is denoted as a cohort. Figure below shows a Spinnaker cluster with 5 nodes and N = 3.

![Example of a Spinnaker cluster](/images/Spinnaker/A-Spinnaker-Cluster.png)

Each cohort has an elected leader, with the other 2 nodes acting as followers. The replication protocol has two phases: a `leader election phase`, followed by a `quorum phase` where the leader proposes a write and the followers accept it.

![The replication protocol](/images/Spinnaker/The-replication-protocol.png)

The steps of Spinnaker’s replication protocol in steady state are as follow:

1. When a client submits a write W, it always gets routed to the leader of the affected key range
2. The leader appends a log record for W to its log and then initiates a log force to disk
3. The leader appends W to its commit queue and sends a propose message for W to its followers
4. When the followers receive the propose message, they force a log record for W to disk, append W to their commit queue, and reply with an ack to the leader
5. After the leader gets an ack from the majority, it applies W to its memtable, effectively committing W
6. The leader returns a response to the client. 

Periodically, the leader sends an asynchronous commit message to the followers asking them to apply all pending writes up to a certain LSN to their memtable.

Strongly consistent reads are always routed to the cohort’s leader, so they are guaranteed to see the latest value for W. Timeline consistent reads can be routed to any node in the cohort, so they may see a stale value for W until its commit message is processed

### Recovery

Let f.cmt = follower’s last committed LSN and f.lst = the follower's last LSN in its log.

The recovery of a follower proceeds in two phases:

`local recovery`: simply replay log between snapshot and f.cmt to recover the memtable  
`catch up`: the logs between f.cmt and f.lst may or may not have been committed by the leader, so the follower has to consult the leader

The recovery of leader consists of re-election. The new leader may be not up-to-date, so it need to truncate the logs. However, since different key ranges write to the same log file (same as BigTable), Spinnaker uses skip-list to do logical truncation. 

## FAQ

The lecture of MIT has FAQ: [Spinnaker FAQ](https://pdos.csail.mit.edu/6.824/papers/spinnaker-faq.txt)

> Q: What is a shared write-ahead log?

A: Same as BigTable, one node only has one log file to improve performance. Different key ranges write to the same log file.

> Q: Periodically, the leader in Spinnaker sends an asynchronous commit message to the followers. Is this design better than sending the commit message as soon as the leader commits the log?

> Q: The paper says "A conditional put is guaranteed to have the same outcome on each node of the cohort because writes are executed in LSN order within a cohort." If using normal put, will the writes be applied out of order?

> Q: The paper says: "At the end of the catch up phase, the leader momentarily blocks new writes to ensure that that the follower is fully caught up." I don't think blocking the writes is necessary.

## Reference

1. [Using Paxos to Build a Scalable, Consistent, and Highly Available Datastore](https://pdos.csail.mit.edu/6.824/papers/spinnaker.pdf)  
2. [6.824 Schedule: Spring 2018](https://pdos.csail.mit.edu/6.824/schedule.html)
