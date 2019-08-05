---
title: Weenix01 - Process and Thread
date: 2018-03-01 00:00:00
tags: [Weenix, Process, Thread, Operating System, C]
categories: [Weenix]
---

> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua, Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.

# Overview
The Weenix assignment is divided into 3 parts, and in the first part, we need to implement process, thread, and scheduling. In this assignment, we make some simplifications and assumes one process only has one thread.

A thread is the abstraction of CPU, it is also the basic unit for calculating. A process is the abstraction of the address space, that is, memory and a bunch of threads. 

The entrance of Weenix is bootstrap() function in kmain.c, which creates the first process, that is, the idle process and then use context_make_active() to run the idle process. The idle process then creates init process and pageout process. Init process is the ancestor of all other processes. We can do the required logic in init process. For example, in kernel 1, we can create and run the faber test; in kernel 2 we can run kernel shell in init process; in kernel 3 we can run the user-space shell in init process.

> boostrap() ->  
> idleproc_run() ->  
> initproc_run()

# Thread and Scheduling
The data structure of kthread is easy to understand. One thing to remember is that when a thread is first created, its kt_state should be KT_NO_STATE; after another thread makes it runnable, its kt_state becomes KT_RUN and enqueue into the run queue. 

But the code of scheduling is tricky. Let's look at them one by one.

## Sched_switch
The textbook offers the pseudo code for this function and "context_switch()" is extremely important. **When a thread gets the CPU again, it will execute right after the context_switch() function as if it never gives up CPU.**

The picture is as follow: 
<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix01%20/Context_Switch.png">


## Cancellable_sleep_on()
This function is also hard to understand. The idea of **Cancellable** in weenix kernel is similar to cancellation in pthreads, that is, **cancellable_sleep_on() is sched_sleep_on plus checking cancellation point.**

Since a kernel thread cannot die in a cancellation point, the return value of this function indicates whether this thread should go canceled or not. Afterward, the function that calls Cancellable_sleep_on() is responsible for how to deal with the canceled thread.

Therefore, the pseudo code is like that:
```C
int
sched_cancellable_sleep_on(ktqueue_t *q)
{       
        if the thread is canceled {
                return immediately;
        }
        
        do similar logic in sched_sleep_on;

        if the thread is canceled {
                return -EINTR;
        }else {
                return 0;
        }
}
```

# Mutex
You can refer to the textbook for details of the pseudocode of mutex, which is a little tricky. Let's see an example of how this code segment works, assuming that there are only 2 threads competing for the mutex.

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix01%20/Mutex.png">


# Proc
It is difficult to divide the responsibility of exit() and do_waitpid(). Generally, **when executing exit(), the process should clean as many resources as possible and wake up its parent if it is waiting; afterward, the parent will finish destroying the process within do_waitpid().**

## Do_waitpid()
When a process executes do_waitpid(), its thread should be blocked on its own kt_wchan. 

If the parent is waiting for a specific child thread, that is, do_waitpid(PID != -1), the implementation is simple; the parent process finds the child process with the given pid and then cleans up the child process.

If the parent is waiting for any child thread, that is, do_waitpid(PID == -1), things will be a little different. Now the parent process should wait for any pid to exit and then dispose of it. **If the parent is wakened up and there are multiple dead child processes, do_wait for the first dead child process.** 


# Pitfalls
## Fail to execute break in list iterate macros
Several macros have the similar format. 
```C
list_iterate_begin(...) {
    ...
    break;
} list_iterate_end();
```

In these macros, if you use break, the break operation will never succeed. Use **goto** instead.

## Pageout Process
The idle process has 2 children: pageout process and init process. Sometimes you need to handle the corner case of the pageout process.
 
# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)