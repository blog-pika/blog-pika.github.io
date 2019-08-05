
---
title: Weenix05 - Implementation of Virtual Memory
date: 2018-04-15 00:00:01
tags: [Weenix, Operating System, C]
categories: [Weenix]
---

> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.
 
# Introduction
This blog illustrates some pitfalls and my understanding of some functions for implementing virtual memory for Weenix(exclude fork and copy-on-write). The overview of some basic components of virtual memory is presented by this picture.

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix%20-%20Virtual%20Memory%20Overview/Overview%20of%20Virtual%20memory%20data%20strcure.png">


# mmap.c
This file contains most of the function we need to manage vmmap, the linked list of vmareas. One pitfall is that the vma_off of vmarea is exclusive, so we need to be careful about whether we need to include the boundary vma_off in if statements or loop statements.

## vmmap_map()
The comment says "If file is NULL an anon mmobj will be used to create a mapping 0's.  If file is non-null that vnode's file will be mapped in for the given range." We are confused about what "create a mapping 0's" means. Later we found out that when you create an anon object, you need to fill the page with all 0, otherwise you cannot pass the memory check. The logic for filling the page can be implemented in anoc.c.

## vmmap_is_range_empty()
The comment says: "Returns 1 if the given address space has no mappings for the given range, 0 otherwise". We are confused about the definition at first. Later we found that it means that if the address space [startvfn, startvfn + npages) is not interleaving with any existed vmarea, return 1; otherwise return 0.

## vmmap_read() and vmmap_write()
Special thanks to my friend Weiji Lin for helping me understand this function.

These two functions are used by copy_to_user and copy_from_user, whose comments say "copy_to_user and copy_from_user are used to **copy to and from the user space** of the current process. They first check that the range of addresses has valid mappings, then call vmmap_read/write."

If we want to copy memory from one kernel-address to another kernel-address, we can simply call **memcpy (destination, source, num)**, because memory is continuous in kernel-space. However, if we want to copy data between user-address and kernel-address, things will become a little more complicated. The reason is that **a count with starting address of buf may span multiple vmareas**. One solution is **not to copy data across page boundaries**, which assures that the copy will not across different vmareas.

The implementation of memcpy() in user-space locates in "kernel1/user/lib/libc/string.c".

# syscall.c
The logic of most of the sys_* functions is quite similar and easy to understand. According to the comment, you need to "page_alloc() a temporary buffer" and then "call do_read(), and copy_to_user() the read bytes". Since page_alloc() only allocates one page, you have to do_read() and copy_to_user page by page.

"copy_to_user" we used to copy the primitives, the "user_strdup" function is used to copy the string.

# pagefault.c
## Overview
When a pagefault happens, _pt_fault_handler() calls handle_pagefault() after doing some error checkings.
The outline of pt_fault_handler is as follow:
```C
// Use vmmap_lookup() to find the corresponding vmarea;
// Check whether the vmarea has the corresponding permission;
// Use pframe_lookup() to get the missing page frame;
// Call pt_map to have the new mapping placed into the appropriate page table;
```

## Check permission
The comment says "Make sure to check the permissions on the **area** to see if the process has permission to do [cause]" Be careful that it is the permission of **vmarea** that we need to check, rather than the permission of pframe.

If the vmarea does not have the permission, we need to terminate the process, e.g., this vmarea is a text segment and we want to write to it. If we have the permission to write the vmarea, but the pframe is R/O, then it is time to do the copy-on-write. 

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix06%20-%20Implementation%20of%20Virtual%20Memory/Pagefault.png">


## Copy-on-write
The comment says: "Now it is time to find the correct page (don't forget about shadow objects, especially copy-on-write magic!). Make sure that if the user writes to the page it will be handled correctly."

Personally, this comment is a little misleading. We should not bother handling the copy-on-write logic at all, which should be implemented in shadow.c instead. The handle_pagefault() merely need to call pframe_lookup(). And thanks to the magic of polymorphism, the mmobj will do copy-on-write(or not) accordingly. 

## Kernel Memory and User Memory Mapping 
A pframe has 2 virtual address: one for the kernel space and one for the user-space, which are necessary for page fault. If we have a page fault at a user-space address, then the virtual address of this user-space is invalid, and the kernel cannot make use of it to write anything to the physical page. Instead, the kernel uses the kernel-space virtual address.

pf->pf_addr is a kernel-space virtual address; the page fault address (vaddr) is a user-space virtual address. In the implementation of pagefault(), finally we will call pt_map(), then these 2 addresses map to the same physical page.

# How hello world is executed
<img src = "https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix06%20-%20Implementation%20of%20Virtual%20Memory/How%20hello%20world%20execute.png">


# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)
