---
title: Weenix06 - Debug anon_put()
date: 2018-04-15 00:00:02
tags: [Weenix, Operating System, C]
categories: [Weenix]
---

> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua, Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.


# Introduction
One of the functions we need to write in the Weenix assignment is anon_put(). You can read the following comments to understand this function.


The pseudocode of the wrong version of **anon_put()** is as follow:
```C
/*
 * Decrement the reference count on the object. If, however, the
 * reference count on the object reaches the number of resident
 * pages of the object, we can conclude that the object is no
 * longer in use and, since it is an anonymous object, it will
 * never be used again. You should unpin and uncache all of the
 * object's pages and then free the object itself.
 */
static void
anon_put(mmobj_t *o)
{   
        o->mmo_refcount--;
        if(o->mmo_refcount == o->mmo_nrespages + 1){
                Iterate_pframe_list{
                    pframe_unpin(pf);
                    pframe_free(pf);
                }

                slab_obj_free(anon_allocator, o);
        }
}
```


# Problem And Analysis
In the implemetion below, we got kernel panic:
> panic in mm/slab.c:342 slab_obj_free(): assertio failed: !obj_bufctl(allocator, obj)->sb_free && "INVALID FREE!"

By tracking the code execution with GDB, we can see that **slab_obj_free(anon_allocator, o);** is executed twice. And executing free more than once for the same mmobj will cause the kernel panic.

The reason is that the design of pframe_free() is not as we expect. Inside pframe_free(), it calls anon_put() recursively. The source code is as follow:
```C
/*
 * Deallocates a pframe (reclaims the page frame for use by something else).
 * The page should not be pinned, free, or busy. Note that if the page is dirty
 * it will not be cleaned. This removes the page's reference to its mmobj.
 *
 * This routine may block in the mmobj put operation.
 * @param pf the page to free
 */
void
pframe_free(pframe_t *pf)
{
       // Some codes which are not important....
        o->mmo_nrespages--;

        /* Now that pf has effectively been freed, dereference the corresponding
         * object. We don't do this earlier as we are modifying the object's counts
         * and also because this op can block */
        o->mmo_ops->put(o);
}
```

The execution flow of the wrong version of anon_put() function is as follow. We can see that slab_obj_free(anon_allocator, o); is executed twice.  
<img src="https://bytebucket.org/LarryTaoWang/pictureofblog/raw/7bb447cdfe96078af3a860edf3c3d434e247c08b/Weenix/Weenix%20-%20Debug%20Hello%20Work%20-%20anon_put%28%29/Wrong%20anon_put%28%29%20work%20flow.png">  

# Solution
If we change our code a little bit, then everything works well.
```C
static void
anon_put(mmobj_t *o)
{   
        if(o->mmo_refcount == o->mmo_nrespages + 1){
                Iterate_pframe_list{
                    pframe_unpin(pf);
                    pframe_free(pf);
                }

                slab_obj_free(anon_allocator, o);
        }
        o->mmo_refcount--;
}
```

The execution flow of the correct version of anon_put() function is as follow.
<img src="https://bytebucket.org/LarryTaoWang/pictureofblog/raw/7bb447cdfe96078af3a860edf3c3d434e247c08b/Weenix/Weenix%20-%20Debug%20Hello%20Work%20-%20anon_put%28%29/Correct%20anon_put%28%29%20work%20flow.png">  




# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)