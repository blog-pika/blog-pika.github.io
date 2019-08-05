---
title: A Note for Scala Sequence Pattern Matching
date: 2018-01-02 19:29:55
tags: [Scala, Pattern Matching]
categories: [Coding Notes]
---

This is a mistake I made when doing the coding assignment from course [Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1/programming/uctOq/huffman-coding). The assignment required us to implement Huffman coding in a functional way.

For descriptions please refer to the question I asked in Stack Overflow: [Scala Pattern Matching Error, “bad simple pattern: bad use of _* (sequence pattern not allowed)”](https://stackoverflow.com/questions/48039077/scala-pattern-matching-error-bad-simple-pattern-bad-use-of-sequence-patte).

The lesson is that:
> If you want to match against a sequence without specifying how long it can be, you can specify \_\* as the **last element** of the pattern.

Unfortunately, I used \_* at the head of the match.