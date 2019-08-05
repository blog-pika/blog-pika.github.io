---
title: Weenix09 - Summary
date: 2018-04-20 00:00:02
tags: [Weenix, Operating System, C]
categories: [Weenix]
---

# Overview
Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua, Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them. 

The kernel assignment is divided into three parts. And in these blogs, I record some important ideas to understand the kernel, plus some mistakes we made.

# Catalogue
The first part of the assignment is to implement the completed life cycle of threads and processes with FIFO scheduling policy, scheduling algorithm, and mutex. You can refer to this blog:  
[Weenix01 - Process and Thread](http://blog.taowang.cloud/2018/03/01/Weenix/Weenix01%20-%20Process-and-Thread/)

The second part of the assignment is to build up a Virtual File System layer to bridge between the OS kernel and various Actual File Systems. You can refer to these blogs:  
[Weenix02 - Virtual File System Overview](http://blog.taowang.cloud/2018/04/02/Weenix/Weenix02%20-%20Virtual-File-System-Overview/)  
[Weenix03 - Debug Virtual File System](http://blog.taowang.cloud/2018/04/02/Weenix/Weenix03%20-%20Debug-Virtual-File-System/)

The third part of the assignment is to construct Virtual Memory module to support user-level program, which supports features such as system call, demanding pages, copy-on-write, and fork. You can refer to these blogs:  
[Weenix04 - Virtual Memory Overview](http://blog.taowang.cloud/2018/04/15/Weenix/Weenix04%20-%20Virtual%20Memeory%20Overview/)  
[Weenix05 - Implementation of Virtual Memory](http://blog.taowang.cloud/2018/04/15/Weenix/Weenix05%20-%20Implementation%20of%20Virtual%20Memory/)  
[Weenix06 - Debug anon_put()](http://blog.taowang.cloud/2018/04/15/Weenix/Weenix06%20-%20Debug%20anon_put()/)  
[Weenix07 - Shadow Object](http://blog.taowang.cloud/2018/04/20/Weenix/Weenix07%20-%20Shadow%20Object/)  
[Weenix08 - Fork](http://blog.taowang.cloud/2018/04/20/Weenix/Weenix08%20-%20Fork/)  

# Demo
After finishing the Weenix Kernel, you will get a user-space shell which is able to run the command and execute the binary file. The screenshot of the shell is like the picture below:  
<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix10-Summary/Demo.png">
