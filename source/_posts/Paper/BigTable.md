---
title: BigTable
date: 2019-08-16 09:46:00
categories: [Paper]
---

This blog is my reading notes for Google Bigtable.

## Pros and Cons

## API

Client applications can write or delete values in Bigtable, look up values from individual rows, or iterate over a subset of the data in a table. Bigtable supports **single-row transaction**.

```cpp
// Writing to Bigtable

// Open the table
Table *T = OpenOrDie("/bigtable/web/webtable");

// Write a new anchor and delete an old anchor
RowMutation r1(T, "com.cnn.www");
r1.Set("anchor:www.c-span.org", "CNN");
r1.Delete("anchor:www.abc.com");
Operation op; Appaly(&op, &r1);
```

```cpp
// Reading from Bigtable
Scanner scanner(T);
ScanStream *stream;
stream = scanner.FetchColumnFamily("anchor");
stream->SetReturnAllVersions();
scanner.Lookup("com.cnn.www");
for (; !stream->Done(); stream->Next()) {
    printf("%s %s %lld %s\n", scanner.RowName(),
      stream->ColumnName(),
      stream->MicroTimestamp(),
      stream->Value());
}
```

## Data Model

Bigtable is a sparse, distributed, persistent multidimensional sorted map. The map is indexed by a `row key`, `column key`, and a `timestamp`, each value in the map is an uninterpreted array of bytes.

Every read or write of data under a single row key is **atomic**. The row range for a table is dynamically partitioned, which is called `tablet` and is the **unit of distribution and load balancing**.

Column keys are grouped into sets called column families, which form the basic **unit of access control**. All data stored in a column family is usually of the same type, which can be indexed with: `family:qualifier`.

Each cell in a Bigtable can contain multiple versions of the same data; these versions are indexed by timestamp. The client can specify either that only the last n versions of a cell be kept, or that only new-enough versions be kept. The out-dated versions are GC automatically.

![Bigtable-Data-Model](/images/Bigtable/Bigtable-Data-Model.png)

## Architecture

Under the hook, Bigtable uses several other Google infrastructure as below

* `GFS`: store logs and data files
* `Borg`: schedule jobs, manage resources, deal with machine failures and monitoring machine status
* `Chubby`: avoid multiple master, store the bootstrap location, discover tablet servers and finalize tablet server deaths

![Bigtable System Structure](/images/Bigtable/Bigtable-System-Structure.png)

For the components in BigTable:

* `Master`: responsible for assigning tablets to tablet servers, detecting the addition and expiration of tablet servers, balancing tablet-server load, and garbage collection of files in GFS
* `tablet server`: manages a set of tablets
* `clients`: every client has a client library, which caches tablet locations, so a client can communicate directly with tablet servers for reads and writes

![BigTable Tablet Representation](/images/Bigtable/BigTable-Tablet-Representation.png)

When a write operation arrives at a tablet server, the server checks that it is well-formed, and that the sender is authorized to perform the mutation. A valid mutation is written to the commit log. After the write has been committed, its contents are inserted into the memtable. When a read operation arrives at a tablet server, it is executed on a merged view of the sequence of SSTables and the memtable.

## Data Storage Engine

BigTable used the data structure of `SSTable`, `memtable` and `Bloomfilter`. Also, it uses column storage and compression for the column. For detail you can refer to _Designing Data-Intensive Applications_, Chapter 03: Storage and Retrieval.

### Service Discovery

Chubby is the key to discover all the servers and tablet information in BigTable. When a tablet server/master server starts, it creates, and acquires an exclusive lock on Chubby directory. Therefore, a server can find the information and status of other servers by asking Chubby.

Chubby if also the root for a three-level hierarchy to store tablet location information.

* first level: Chubby that contains the location of the root tablet
* second level: Root tablet contains the location of all tablets in a special METADATA table
* third level: each METADATA tablet contains the location of a set of user tablets

![Bigtable Tablet Location Hierarchy](/images/Bigtable/Bigtable-Tablet-location-hierarchy.png)

### Load Balance

Most Load balancing are done by the master except one:

* Master: a table is created or deleted, two existing tablets are merged to form one larger tablet
* Tablet Server: an existing tablet is split into two smaller tablets

The master is able to keep track of these changes because it initiates all. Tablet splits are treated specially since they are initiated by a tablet server. The tablet server commits the split by recording information for the new tablet in the METADATA table. When the split has committed, it notifies the master. In case the split notification is lost (either because the tablet server or the master died), the master detects the new tablet when it asks a tablet server to load the tablet that has now split. The tablet server will notify the master of the split, because the tablet entry it finds in the METADATA table will specify only a portion of the tablet that the master asked it to load.

### Fault Tolerance

GFS and Chubby are fault-tolerant, which can simplify the design of BitTable. **GFS takes care of the fault-tolerance of file and Bigtable uses Chubby to keep track of servers**. If Chubby fails, BigTable will soon be unavailable. For the master, it kills itself if its Chubby session expires. Since master only cares about load balancing and meta data, the crash of master is not a big deal, Borg can simply start a new master:

1. he master grabs a unique master lock in Chubby, which prevents concurrent master instantiations
2. The master scans the servers directory in Chubby to find the live servers.
3. The master communicates with every live tablet server to discover what tablets are already assigned to each server.
4. The master scans the METADATA table to learn the set of tablets.

What if the tablet server is unavailable? The master is responsible for detecting when a tablet server is no longer serving its tablets. If a tablet server is unavailable, it loses its exclusive lock. T**he master periodically asks each tablet server for the status of its lock**. If a tablet server reports that it has lost its lock, or if the master was unable to reach a server during its last several attempts, the master attempts to acquire an exclusive lock on the server’s file. If the master is able to acquire the lock, then Chubby is live and the tablet server is either dead or having trouble reaching Chubby, so the master ensures that the tablet server can never serve again by deleting its server file. Once a server’s file has been deleted, the master can move all the tablets that were previously assigned to that server into the set of unassigned tablets.

## FAQ

> Q: When a server crashes, will the data in memetable lost?

A: No, the write operation writes the logs in GFS first, so the memory data can be recovered from logs.

> Q: Will Chubby file or Root Tablet be the hotspot?

A: No. Client library does aggressive pre-fetching and caching so the root tablet file is not accessed that frequently.

> Q: How does this work: **The only mutable data structure that is accessed by both reads and writes is the memtable. To reduce contention during reads of the memtable, we make each memtable row copy-on-write and allow reads and writes to proceed in parallel.**

A: ???

> Q: How can two log writing threads protects mutations from GFS latency spikes?

A: ???

> Q: How can copy-on-write on each memtable row allows reads and writes to proceed in parallel?

A:???

> GFS可能出现重复记录或者padding，Bigtable如何处理这种情况使得对外提供强一致性模型？
 
> 为什么Bigtable设计成Root、Meta、User三级结构，而不是两级或者四级结构？

> 读取某一行用户数据，最多需要几次请求？分别是什么？

> 如何保证同一个tablet不会被多台机器同时服务？

> Tablet在内存中的数据结构如何设计？

> 如何设计SSTable的存储格式？
> 
> minor、merging、major这三种compaction有什么区别？
> 
> Tablet Server的缓存如何实现？
> 
> 如果tablet出现故障，需要将服务迁移到其它机器，这个过程需要排序操作日志。如何实现？
> 
> 如何使得tablet迁移过程停服务时间尽量短？
> 
> tablet分裂的流程是怎样的？
> 
> tablet合并的流程是怎样的？

## Reference

1. [Bigtable: A Distributed Storage System for Structured Data](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
2. [Youtube: BigTable: A Distributed Structured Storage System](https://www.youtube.com/watch?v=2cXBNQClehA)  
3. [有哪些分布式数据库书籍或论文比较好？](https://www.zhihu.com/question/37647788/answer/72960959)  
4. Designing Data-Intensive Applications, Chapter 03: Storage and Retrieval
