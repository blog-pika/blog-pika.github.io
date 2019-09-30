---
title: Zookeeper
date: 2019-08-21 00:00:00
tags: [Zookeeper, 6.824]
categories: [Paper]
---

This blog is my reading notes for Zookeeper. We can learn from this paper how to use consensus algorithm to build replicated services with high read throughput.

<!-- more -->

## Data Model and API

ZooKeeper provides to its clients the abstraction of a set of data nodes (`znodes`), organized as a file system with only full data reads and writes API. Every znode maps to abstractions of the client application, typically corresponding to meta-data used for coordination purposes. All except for ephemeral znodes, can have children:

* Regular: Clients manipulate regular znodes by creating and deleting them explicitly
* Ephemeral: Clients create such znodes, and they either delete them explicitly, or let the system remove them automatically when the session that creates them terminates (deliberately or due to a failure)

![ZooKeeper hierarchical name space](/images/Paper/ZooKeeper-hierarchical-name-space.png)

A client connects to ZooKeeper and initiates a session. Sessions have an associated timeout. ZooKeeper considers a client faulty if it does not receive anything from its session for more than that timeout. A session ends when clients explicitly close a session handle or ZooKeeper detects that a clients is faulty.

When creating a new znode, a client can set a `sequential ﬂag`. Nodes created with the sequential flag set have the value of a monotonically increasing counter appended to its name.

ZooKeeper also implements `watches` to allow clients to receive timely notifications of changes without requiring polling. When a client issues a read operation with a watch flag set, the operation completes as normal except that the server promises to notify the client when the information returned has changed. Watches are one-time triggers associated with a session; they are unregistered once triggered or the session closes. Watches indicate that a change has happened, but do not provide the change.

The APIs are as follow:

* `create(path, data, flags)`: Creates a znode with path name path, stores data[] in it, and returns the name of the new znode. flags can be regular/ephemeral +  sequential flag
* `delete(path, version)`: Deletes the znode path if that znode is at the expected version
* `exists(path, watch)`: Returns true if the znode with path name path exists, and returns false otherwise. The watch flag enables a client to set a watch on the znode
* `getData(path, watch)`: Returns the data and meta-data, such as version information, associated with the znode. ZooKeeper does not set the watch if the znode does not exist
* `setData(path, data, version)`: Writes data[] to znode path if the version number is the current version of the znode
* `getChildren(path, watch)`: Returns the set of names of the children of a znode
* `sync(path)`: Waits for all updates pending at the start

Zookeeper provides a consistency model with ordering guarantees tailed for read heavy scenarios:

* `Linearizable writes order`: all requests that update the state of ZooKeeper are serializable and respect precedence
* `FIFO client order`: all requests from a given client are in the order that they were sent by the client.
* `Notification ordering`: if a client is watching for a change, the client will see the notification event before it sees the new state of the system after the change is made

It also provide additional two guarantees:

* `liveness` and  guarantees: if a majority of ZooKeeper servers are active and communicating the service will be available
* `durability`: if the ZooKeeper service responds successfully to a change request, that change persists across any number of failures as long as a quorum of servers is eventually able to recover.

## Architecture

The components of Zookeeper are as follow:

![Components of ZooKeeper Service](/images/Paper/Zookeeper-Components.png)

### Write

Write needs to be duplicated among servers, so it is expensive. When a write request comes:

1. write requests are forwarded to leader
2. leader calculates what the state of the system will be when the write is applied and transforms it into a transaction that captures this new state.
3. followers receive message proposals consisting of state changes from the leader and agree upon state changes. ZAB uses by default simple majority quorums to decide on a proposal. ZAB guarantees that changes broadcast by a leader are delivered in the order they were sent and all changes from previous leaders are delivered to an established leader before it broadcasts its own changes.

### Read

A read operation is fast since it is read local data. The drawback is that a read operation may return a stale value because the local server may lag behind.

For application requires latest data from the requesting server, Zookeeper implemented `sync`. The FIFO order guarantee of client operations together with the global guarantee of sync enables the result of the read operation to reflect any changes that happened before the sync was issued. We simply place the sync operation at the end of the queue of requests between the leader and the server executing the call to sync (not all servers). In order for this to work, the follower must be sure that the leader is still the leader. If there are pending transactions that commit, then the server does not suspect the leader. If the pending queue is empty, the leader needs to issue a null transaction to commit and orders the sync after that transaction.

When session transfer from one server to another server, it should not see data in the past. Zookeeper client records the last transaction it has seen (zxid). If the client connects to a new server, that new server ensures that its view of the ZooKeeper data is at least as recent as the view of the client by checking the last zxid of the client against its last zxid. If the client has a more recent view than the server, the server does not reestablish the session with the client until the server has caught up. The client is guaranteed to be able to find another server that has a recent view of the system since the client only sees changes that have been replicated to a majority of the ZooKeeper servers.

### Recovery

When a ZooKeeper server recovers from a crash, it needs to recover snapshot + replay all delivered messages to recover state. ZooKeeper uses periodic snapshots and only requires redelivery of messages since the start of the snapshot. We call ZooKeeper snapshots fuzzy snapshots since we do not lock the ZooKeeper state to take the snapshot. Since the resulting fuzzy snapshot may have applied some subset of the state changes delivered during the generation of the snapshot, the result may not correspond to the state of ZooKeeper at any point in time. However, since state changes are idempotent, we can apply them twice as long as we apply the state changes in order.

## Examples of primitives

ZooKeeper API can be used to implement more powerful primitives, e.g., configuration management, rendezvous, group membership, lock, R/W lock, barrier.

## FAQ

Here is the [FAQ from MIT 6.824](https://pdos.csail.mit.edu/6.824/papers/zookeeper-faq.txt)

> Q: What does `Linearizable` mean?

A: In a linearizable system, as soon as one client successfully completes a write, all clients reading from the database must be able to see the value just written. The value read is the most recent, up-to-date value, and doesn’t come from a stale cache or replica. In other words, linearizability is a recency guarantee. For detail please refer to _Designing Data Intensive Application, Chapter 9.2, Linearizability_.

> Q: What does `pipeline` mean?

A: MIT 6.824 FAQ: Zookeeper "pipelines" the operations in the client API (create, delete, exists, etc). What pipelining means here is that these operations are executed asynchronously by clients. The client calls create, delete, sends the operation to Zookeeper and then returns. At some point later, Zookeeper invokes a callback on the client that tells the client that the operation has been completed and what the results are. This asynchronous interface allow a client to pipeline many operations: after it issues the first, it can immediately issue a
second one, and so on. This allows the client to achieve high throughput; it doesn't have to wait for each operation to complete before starting a second one.

A worry with pipelining is that operations that are in flight might be re-ordered, which would cause the problem that the authors to talk about in 2.3. If a the leader has many write operations in flight followed by write to ready, you don't want those operations to be re-ordered, because then other clients may observe ready before the preceding writes have been applied. To ensure that this cannot happen, Zookeeper guarantees FIFO for client operations; that is the client operations are applied in the order they have been issued.

> Q: In the case of read requests, a server simply reads the state of the local database, will we get stale data?

A: Yes, we may read stale data. If we need the latest data, we can use `sync`.

> Q: Each read request is processed and tagged with a zxid that corresponds to the last transaction seen by the server. What does transaction stands for?

A: A write request.

## Reference

1. [ZooKeeper: Wait-free coordination for Internet-scale systems](https://pdos.csail.mit.edu/6.824/papers/zookeeper.pdf)  
2. [6.824 Schedule: Spring 2018](https://pdos.csail.mit.edu/6.824/schedule.html)  
3. _Designing Data Intensive Application, Chapter 9.2, Linearizability_  
