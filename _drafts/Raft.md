---
title: Raft
date: 2019-07-29 00:00:00
tags: [Raft, Consensus Algorithm, Distributed System]
categories: [Distributed System]
---

## Raft FAQ

I got confused about these questions when I learnt Raft myself.

### Q1: Why we need both nextIndex and matchIndex?

From MIT's Q&A:
> ```nextIndex``` is a guess as to what prefix the leader shares with a given follower. It is generally quite **optimistic**, and is moved backwards only on negative responses, is used for performance.

> ```matchIndex``` is used for safety. It is a **conservative** measurement of what prefix of the log the leader shares with a given follower, and only updated when a follower positively acknowledges an AppendEntries RPC.

### Q2: Persistent Strategy

* ```votedFor```: A Raft instance should only vote for once in every single term. If not persisted, an instance can grant vote to another candidate even if it has voted before crash.
* ```commitIndex```:  A follower that crashes and comes back up will be told about the right commitIndex whenever the current leader sends it an AppendEntries RPC.
* ```lastApplied``` starts at zero after a reboot, thus its state needs to be completely recreated by replaying all log entries.

### Q3: Why should server check `votedFor == candidateId` for `RequestVote RPC`?

Posted on stack overflow: [Why Raft should grant vote when voteFor is candidateId?](https://stackoverflow.com/questions/57212795/why-raft-should-grant-vote-when-votefor-is-candidateid)

### Q4: How does Raft deals with delayed RPC replies?

Posted on stack overflow: [How does Raft deals with delayed replies in AppendEntries RPC?
](https://stackoverflow.com/questions/56677690/how-does-raft-deals-with-delayed-replies-in-appendentries-rpc)

### Q4: 

## Reference

> [1] [Raft (extended) (2014)](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)  
> [2] [MIT 6.824 Raft Lecture Notes, FAQs, Labs](https://pdos.csail.mit.edu/6.824/schedule.html)  
> [3] [Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)  
> [4] [Raft Q&A](https://thesquareplanet.com/blog/raft-qa/)
