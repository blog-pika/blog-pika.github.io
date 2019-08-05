---
title: Weenix07 - Shadow Object
date: 2018-04-20 00:00:00
tags: [Weenix, Operating System, C]
categories: [Weenix]
---


> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.

# Introduction
To support both fork and copy-on-write, we should make use of shadow objects, without which supporting these two features will be a great mess. You can refer to the textbook for details about shadow objects.

The shape of all shadow objects of a process is like a reversed tree and **the higher layer records the more up-to-date modifications**. Some similar ideas can be found in functional programming. 

For example, let's see the picture below. Initially process A has pages x0, y0, y0(layer0). When process A modifies page x0, the shadow object records the new page x1(layer1). When process A forks, process A and child process will have new shadow objects(layer2). When Process A modifies page z and Process B modifies page y, the modifications will be recorded in layer 2. When process B forks, process B, and child process C will have new shadow objects(layer3).

<img src="https://bitbucket.org/LarryTaoWang/pictureofblog/raw/master/Weenix/Weenix08%20-%20Shadow%20Object/Shadow-Object.png">

Therefore, we can trace the latest pages and the original pages conveniently. For process A, the latest pages are in the upper layer, x1, y0, z2; and the original pages are in the bottom, x0, y0, z0.

# Implementation 
We need to handle the logic for both copy-on-write and do-not-copy-on-not-write. There are 3 important and useful functions: pframe_get_resident() and pframe_get()/pframe_lookup(). 

pframe_get_resident() merely does lookup operation; pframe_get() also does lookup operation, however, if the pframe is not in the memory, pframe_alloc() and pframe_fill() will be called and a new pframe will be created; pframe_lookup() is identical to pframe_get(), which calls the lookuppage operation of mmobj and eventualy calls pframe_get(). 

Therefore, **we can use pframe_get_resident() to implement do-not-copy-on-not-write logic and use pframe_get() to implement copy-on-write logic.**

## shadow_lookuppage()
shadow_lookuppage() only handles the logic for do-not-copy-on-not-write magic. The logic is simple: start searching from mmobj o, the first pframe is the most up-to-date pframe that we need. 

**The desired page may not be in memory now.** In this case, we need to use pframe_lookup() to bring this pframe into memory(for files objects) or created a new pframe(for anonymous objects).

 The psuedo code is as follow:

```C
static int
shadow_lookuppage(mmobj_t *o, uint32_t pagenum, int forwrite, pframe_t **pf)
{
        if (need to do write){
                return pframe_get(o);
        }

        pframe_t *p = NULL; 
        iterate every layer of shadow object{
            if(pframe_get_resident(desired page)){
                p = desired page;
                break;
            }
        }

        if(cannot find the desired page){
            pframe_lookup(bottom_mmobj, desired page, &p);
        }

        *pf = p;

        return 0;
}
```



## shadow_fillpage()
Recall that when a process is created or does fork() operation, it will get a new layer of shadow object, which contains no pframes. At that time, if we want to look up a page for write, shadow_fillpage() will be called. 
The flow is like that:
>  shadow_lookuppage(forwrite == 1)   
-> pframe_get() and cannot get the pframe   
-> pframe_fill()  
-> shadow_fillpage()

The pseudo code is as follow:

```C
static int
shadow_fillpage(mmobj_t *o, pframe_t *pf)
{
        pframe_t *p = NULL;

        start iteration from o->mmo_shadowed {
             if(pframe_get_resident(desired page)){
                p = desired page;
                break;
            }
        }
        
        if(cannot find the desired page){
            mmobj_t *bottom = bottom mmobj;
            pframe_lookup(bottom_mmobj, desired page, &p);
        }

        memcpy(pf->pf_addr, p->pf_addr, PAGE_SIZE);

        return 0;
}
```

The basic logic is that: find the most up-to-date pframe and use memcpy to copy it into the pf pframe. The comment is more specific: "fill the page frame starting at address pf->pf_addr with the contents of the page identified by pf->pf_obj and pf->pf_pagenum."

**One thing should be noted is that the iteration starts from o->mmo_shadowed.** Because we need to copy the most up-to-date pframe into the latest shadow object and the most up-to-date pframe stays in the layer starting from o->mmo_shadowed.





# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)