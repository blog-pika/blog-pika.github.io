---
title: Weenix04 - Virtual Memory Overview
date: 2018-04-15 00:00:00
tags: [Weenix, Virtual Memory, Operating System, C]
categories: [Weenix]
---

> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.


# Introduction
In the previous kernel assignment, we developed threads and processes, Virtual File System, and now it is time to implement Virtual Memory, with which the kernel is able to support the user-space processes. Personally, virtual memory is hard to understand, implement and debug. So it is a good idea to read the textbook, the weenix documentation, and the FAQ thoroughly.

Below is an important picture of virtual memory. 

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix%20-%20Virtual%20Memory%20Overview/Address-space%20representation%20in%20the%20textbook.png ">  

When the hello.c file is compiled into hello.exec file, the compiler will compile it different segments, which are loaded into the memory separately with their file object or anon object. Furthermore, memory is split into a fixed size of pages(in 32-bit machine 4KB per page), so a segment may have a lot of pages.

For example, after the text segment is compiled and loaded into the memory, it becomes the region "as_region 1000-7fff, rx, shared". The starting page number is 1000, the end page number is 7fff, so the segment has (7fff - 1000 + 1) pages in theory.

We can use the virtual address to access the segment. For example, if we want to access address 0x1000000, then the corresponding page is (0x1000000 << 12 = 0x1000). If this page is already in memory, we can get it immediately; otherwise, a page fault will occur and the file object will load it from disk into memory, then we can get it.

# Data Structure in Weenix
Below is a similar picture of the picture above, but described in the data structure in the Weenix kernel. In the picture below, the "PCB" is implemented by the struct "proc_t"; the "address space" is implemented by the struct "vmmap_t"; the "as region" is implemented by the struct "vmarea_t"; the "file object / anon object" is implemented by the struct "mmobj_t". Besides Weenix we have a data structure called "pframe", which has the one-to-one mapping relationship with a physical page.

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix%20-%20Virtual%20Memory%20Overview/Overview%20of%20Virtual%20memory%20data%20strcure.png">

## Struct vmmap_t and vmarea_t
Every process has its own address space and page table, so, in weenix, struc proc_t has a filed **vmmap_t**, which is the head of a linked list containing all the vmareas(as_regions) of this process. 

The vma_start is inclusive but the vma_end is exclusive, so for the as_region 1000-7fff, vma_start = 1000 and vma_end = 8000. Remember that in Weenix the basic unit for memory allocation is a page of 4KB size. So, **the mapping boundaries, vma_start, and vma_end are in terms of page numbers and not addresses**! Weenix provides several handy macros to do this kind of conversion in "page.h" such as converting the address to page number.
>#define ADDR_TO_PN(x) (((uint32_t)(x)) >> PAGE_SHIFT)

The field **vma_olink** is declared in the struct vmarea_t, but it is OK if you just ignore it. It seems this field is not used at all.

Every vmarea should have a file object/anon object, which is implemented as the struct "mmobj_t". **A vmarea uses a mmobj to "read pages"**. Since the physical page has a one-to-one mapping relationship with the frame, we can also say that **a vmarea use a mmobj to manage physical pages/ pframes.**

## Struct pframe
There is a filed "vma_off" in struct "vmarea_t" and a field "pagenum" in struct pframe. Why do we need them?

The reason is that ** a mmobj can be shared by multiple vmareas, so the pages a vmarea need may not be equal to the pframe number a mmobj have**, so we need the vma_off as an offset.

For example, in the picture below, the file object is shared by 2 vmareas. Each vmarea needs 2 pages but the mmobj owns 4 pframes. With the help of vma_off, we can know that the vmarea A needs the first 2 pages(starting from vma_off = 0);the vmarea B needs the last 2 pages(starting from vma_off = 2).

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix%20-%20Virtual%20Memory%20Overview/pframe_pagenum%20and%20vmarea_off.png">

By the way, I was confused about pframe at first: since the physical page has a one-to-one mapping relationship with pframe, why do we need a pframe?

One reason is that looking up using pagetable is one direction: you can only use a virtual address to find the physical address. With the help of pframe, you can do the look up bi-directions, such as find out which process is using a particular page frame.

Another reason is that Weenix uses 3 linked list of pframes to manages physical pages. A page is always in one of three categories: free, allocated, pinned. These linked lists can simplify the work of page out daemon.


# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)