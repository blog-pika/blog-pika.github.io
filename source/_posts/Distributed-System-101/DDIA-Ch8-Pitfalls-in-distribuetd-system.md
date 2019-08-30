---
title: DDIA - Ch8 - Pitfalls in Distributed System
date: 2019-08-11 00:00:00
tags: [Distributed System, DDIA]
categories: [Reading Notes, Designing Data-Intensive Applications]
---

## Abstract  

This blog is my reading notes for _Designing Data-Intensive Applications_, Chapter 08: The Trouble With Distributed System.

Working with distributed systems is fundamentally different from writing software on a single computer. The main difference is that we can know nothing for sure. This chapter first introduces how components can be unreliable in a distributed system, and why designing distributed system can be challenging. Then the author discusses how to get truth from uncertainty. He also introduces some theoretical models and how can a distributed algorithm defined as correct.

<!-- more -->

## Typical UnReliable Component

### Unreliable Network

Most distributed systems are asynchronous networks and have unbounded delays, that is, they try to deliver packets as quickly as possible, but there is no upper limit on the time it may take for a packet to arrive. So Whenever you try to send a packet over the network, it may be lost or arbitrarily delayed. Likewise, the reply may be lost or delayed, so if you don’t get a reply, you have no idea whether the message got through.

Though TCP retries transparently, in the application level, you may also retry a few times, wait for a timeout to elapse, and eventually declare the node dead if you don’t hear back within the timeout. Even better, rather than using configured constant timeouts, systems can continually measure response times and their variability, and automatically adjust timeouts according to the observed response time distribution. `Akka` and `Cassandra` use a Phi Accrual failure detector to dynamically adjust the timeout.

**You do need to know how your software reacts to network problems and ensure that the system can recover from them.** If your network is normally fairly reliable, a valid approach may be to simply show an error message to users while your network is experiencing problems.

### Unreliable Clock

Each machine on the network has its own clock, which is an actual hardware device: usually a quartz crystal oscillator. The quartz clock in a computer is not very accurate: it drifts (runs faster or slower than it should), so **each machine has its own notion of time, which may be slightly faster or slower than on other machines**. A commonly used mechanism to synchronize clocks is the `Network Time Protocol (NTP)`, which allows the computer clock to be adjusted according to the time reported by a group of servers.

Modern computers have at least two different kinds of clocks: a `time-of-day clock` to get `timestamp`, and a `monotonic clock` to get `time duration`. Time-of-day clock returns the current date and time according to some calendar. However, it may be forcibly reset by the NTP server, suddenly jump forward or back in time. Even worse, NTP synchronization can only be as good as the network delay, so there is a limit to its accuracy when you’re on a congested network with variable packet delays. Therefore, **replying on Time-of-day clock is dangerous**.

On the contrary, NTP cannot cause the monotonic clock to jump forward or backward. Since a monotonic clock is guaranteed to always move forward, it is usually fine to use a monotonic clock to measure time duration in distributed system.

There are two ways to get reliable timestamps

* `Logical Clocks`: which are based on incrementing counters rather than an oscillating quartz crystal, are a safer alternative for ordering events. Logical clocks do not measure the time of day or the number of seconds elapsed, only the **relative ordering of events**.
* `Google Spanner` provides a _TrueTime_ API, which returns [earliest, latest]. The interval are the earliest possible and the latest possible timestamp. Therefore, the clock knows that the actual current time is somewhere within that interval. Since Spanner uses atomic clocks, the interval are usually within 7ms. In order to ensure that transaction timestamps reflect causality, Spanner deliberately waits for the length of the confidence interval before committing a read-write transaction. By doing so, it ensures that any transaction that may read the data is at a sufficiently later time, so their confidence intervals do not overlap.

A problem of clock is that even if it is defective, most things will seem to work fine. If some piece of software is relying on an accurately synchronized clock, the result is more likely to be silent and subtle data loss than a dramatic crash. Thus, **if you use software that requires synchronized clocks, it is essential that you also carefully monitor the clock offsets between all the machines.** Any node whose clock drifts too far from the others should be declared dead and removed from the cluster.

### Process Pause

A running thread can be preempted at any point and resume it at some later time, without the thread even noticing: e.g.:

* Many programming language runtime have a `GC` that occasionally needs to stop all running threads, which can pauses for several minutes in the worst case! Even so-called "concurrent" garbage collectors cannot fully run in parallel with the application code
* Execution may also be suspended and resumed arbitrarily, e.g., when VM migrates from one host to another host, when the operating system context-switches to another thread, when the user closes the lid of their laptop
* If the application performs synchronous `disk access`, a thread may be paused waiting for a slow disk I/O operation to complete. If the operating system is configured to allow paging, a simple memory access may result in a page fault that requires a page from disk to be loaded into memory. In many languages, disk access can happen surprisingly, even if the code doesn’t explicitly mention file access. For example, the Java class loader lazily loads class files when they are first used, which could happen at any time in the program execution.
* A Unix process can be paused by sending it the SIGSTOP signal

Therefore, we can’t assume anything about timing. **A node in a distributed system must assume that its execution can be paused for a significant length of time at any point, even in the middle of a function. During the pause, the rest of the world keeps moving and may even declare the paused node dead because it’s not responding.** Eventually, the paused node may continue running, without even noticing that it was asleep until it checks its clock sometime later.

In the code below, if code enters `if (lease.isValid()` but suddenly preempted. During its pause other nodes can become the new leader. If it continues to process request as it is supposed to do as a leader, the system may go wrong.

```C
while (true) {
    request = getIncomingRequest();

    // Ensure that the lease always has at least 10 seconds remaining 
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) { 
        lease = lease.renew();
    }

    if (lease.isValid()) {
        process(request);
    }
}
```

We can limit the impact of garbage collection in the following two ways:

* Treat GC pauses like brief planned outages of a node, and to let other nodes handle requests from clients while one node is collecting its garbage. If the runtime can warn the application that a node soon requires a GC pause, the application can stop sending new requests to that node, wait for it to finish processing outstanding requests, and then perform the GC while no requests are in progress
* Use the garbage collector only for short-lived objects and to restart processes periodically, before they accumulate enough long-lived objects to require a full GC of long-lived objects

## Challenges in Distributed System

Uncertainty is the defining characteristic of distributed systems. Whenever software tries to do anything involving other nodes, there is the possibility that it may occasionally fail, or randomly go slow, or not respond at all (and eventually time out). In distributed systems, we try to build tolerance of partial failures into software, so that the system as a whole may continue functioning even when some of its constituent parts are broken.

To tolerate faults, the first step is to detect them, but even that is hard. Most systems don’t have an accurate mechanism of detecting whether a node has failed, so most distributed algorithms rely on `timeouts` to determine whether a remote node is still available. However, timeouts can’t distinguish between network and node failures, and variable network delay sometimes causes a node to be falsely suspected of crashing.

Once a fault is detected, making a system tolerate it is not easy either: there is no global variable, no shared memory, no common knowledge or any other kind of shared state between the machines. Nodes can’t even agree on what time it is, let alone on anything more profound. The only way information can flow from one node to another is by sending it over the unreliable network. Major decisions cannot be safely made by a single node, so we require protocols that enlist help from other nodes and try to get a quorum to agree.

If you’re used to writing software in the idealized mathematical perfection of a single computer, where the same operation always deterministically returns the same result, then moving to the messy physical reality of distributed systems can be a bit of a shock. Conversely, distributed systems engineers will often regard a problem as trivial if it can be solved on a single computer, and indeed a single computer can do a lot nowadays. If you can avoid opening Pandora’s box and simply keep things on a single machine, it is generally worth doing so.

However, scalability is not the only reason for wanting to use a distributed system. Fault tolerance and low latency (by placing data geographically close to users) are equally important goals, and those things cannot be achieved with a single node.

## How to know the Truth

A node cannot necessarily trust its own judgment of a situation. A distributed system cannot exclusively rely on a single node, because a node may fail at any time, potentially leaving the system stuck and unable to recover. Instead, many distributed algorithms rely on a quorum, that is, voting among the nodes: decisions require some minimum number of votes from several nodes in order to reduce the dependence on any one particular node. Most commonly, the quorum is an absolute majority of more than half the nodes.

## System Model and Algorithms Scope

Algorithms for distributed system need to be written in a way that does not depend too heavily on the details of the hardware and software configuration on which they are run. This in turn requires that we somehow formalize the kinds of faults that we expect to happen in a system. We do this by defining a system model, which is an abstraction that describes what things an algorithm may assume.

With regard to timing assumptions, three system models are in common use:

* `Synchronous model`: The synchronous model assumes bounded network delay, bounded process pauses, and bounded clock error. Bounded does not imply exactly zero; it just means that they will never exceed some fixed upper bound. The synchronous model is not a realistic model of most practical systems
* `Partially synchronous model`: Partial synchrony means that a system behaves like a synchronous system most of the time, but it sometimes exceeds the bounds for network delay, process pauses, and clock drift. **This is a realistic model of many systems.**
* `Asynchronous model`: In this model, an algorithm is not allowed to make any timing assumptions. In fact, it does not even have a clock, so it cannot use timeouts. Some algorithms can be designed for the asynchronous model, but it is very restrictive.

With regard to node failures. The three most common system models are:

`Crash-stop faults`:  a node can fail in only by crashing: suddenly stop responding at any moment, and thereafter that node is gone forever
`Crash-recovery faults`: nodes may crash at any moment, and perhaps start responding again after some unknown time. Stable storage is assumed to be preserved across crashes, while the in-memory state is assumed to be lost
`Byzantine faults`: Nodes may do absolutely anything, **including trying to trick and deceive other nodes**.

For modeling real systems, the partially synchronous model with crash-recovery faults is generally the most useful model. But how do distributed algorithms cope with that model? To define what it means for an algorithm to be correct, we need to define safety and liveness properties. Safety is often informally defined as nothing bad happens, and liveness as something good eventually happens.:

* `Safety`: If a safety property is violated, we can point at a particular point in time at which it was broken (for example, if the uniqueness property was violated, we can identify the particular operation in which a duplicate fencing token was returned). After a safety property has been violated, the violation cannot be undone—the damage is already done.
* `Liveness`: may not hold at some point in time (for example, a node may have sent a request but not yet received a response), but there is always hope that it may be satisfied in the future (namely by receiving a response).

**For distributed algorithms, it is common to require that safety properties always hold**, in all possible situations of a system model. That is, even if all nodes crash, or the entire network fails, the algorithm must nevertheless ensure that it does not return a wrong result. However, **with liveness properties we are allowed to make caveats**: for example, we could say that a request needs to receive a response only if a majority of nodes have not crashed, and only if the network eventually recovers from an outage. The definition of the partially synchronous model requires that eventually the system returns to a synchronous state—that is, any period of network interruption lasts only for a finite duration and is then repaired.

## Reference

> [1] _Designing Data-Intensive Applications_, Chapter 08: The Trouble With Distributed System
