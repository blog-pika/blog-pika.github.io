---
title: Weenix08 - Fork
date: 2018-04-20 00:00:01
tags: [Weenix, Operating System, C, Fork]
categories: [Weenix]
---


> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.

# Overview
The faq of the fork system call is very detailed and the pseudo code is like that:
```C
int
do_fork(struct regs *regs)
{
        /* Create a copy of the curproc, handles vmmap_t, shadow object, etc accordingly. */
        proc_t *child_proc = proc_clone_except_thread();
       
        uncache_pt_and_tlb();
        
        /* Create a clone of the curthr, associate it with the child_proc */
        kthread_t *child_thr = create_child_thread(child_proc, regs);
        sched_make_runnable(child_thr);

        regs->r_eax = child_proc->p_pid;

        return child_proc->p_pid;       
}
```

# How to Distinguish Return Value?
One challenging question is how to distinguish parent process and child process, then return different value accordingly? **The solution is in the r_eax register, which stores the return value of a function.** 

If we set the r_eax registers of the child process and parent process accordingly, their return value will be different. Therefore, when we create the child thread, we should set regs->r_eax to 0; as for the parent process, we should set the regs->r_eax to child_proc->p_pid. 

# Why do we need flush TLB?
In the picture below, if we do not unmap the user-space page table, then the page x1 is R/W. Since x1 is R/W and in the memory, process A, and B can lookup this page directly, that is, process A and B share this page. This is not what we expect for forking, instead, process A and B should each has a private copy of page B.  

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix09%20-%20Fork/Fork-Unmap-User-space.png">

That's why we need to unmap the entire user-space page table when doing fork(). **Pages will be set to R/O on the next page fault if reading.**


We met another problem when implementing uncache_pt_and_tlb(). At first, we do flushing in a for loop, which is incredibly slow. 
```C
for(from USER_MEM_LOW; to USER_MEM_HIGH){
    tlb_flush_range
}
```

However, if we use tlb_flush_all(), which is implemented by the hardware, it will be much faster.

## How Vmarea changes when forking?
If the mmobj is shared, then the change is as follow:  
<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix09%20-%20Fork/Fork(Shared).png">

If the mmobj is private, then the change is as follow:  
<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix09%20-%20Fork/Fork(Private).png">


# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)