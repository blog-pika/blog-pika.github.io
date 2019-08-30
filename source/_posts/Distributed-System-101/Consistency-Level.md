---
title: Consistency Level
date: 2019-08-19 08:41:09
tags:
---

Strongly-consistent systems are easier to program because they behave like a non-replicated systems. Building applications on top of eventually-consistent systems is generally more difficult because the programmer must consider scenarios that couldn't come up in a non-replicated system. On the other hand, one can often obtain higher performance from weaker consistency models.

## Timeline Consistency

A form of relaxed consistency that can provide faster reads at the expense of visibly anomalous behavior. All replicas apply writes in the same order (clients send writes to the leader, and the leader picks an order and forwards the writes to the replicas). Clients are allowed to send reads to any replica, and that replica replies with whatever data it current has. The replica may not yet have received recent writes from the leader, so the client may see stale data. If a client sends a read to one replica, and then another, the client may get older data from the second read. Unlike strong consistency, timeline consistency makes clients aware of the fact that there are replicas, and that the replicas may differ.
