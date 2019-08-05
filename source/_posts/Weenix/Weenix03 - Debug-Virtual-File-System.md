---
title: Weenix03 - Debug Virtual File System
date: 2018-04-02 00:00:01
tags: [Weenix, File System, Operating System, C]
categories: [Weenix]
---

> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua, Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.

# Overview
This article, which records some mistakes we made when implementing VFS for Weenix, is the follow-up for [Weenix - Virtual File System Overview](http://blog.taowang.cloud/2018/04/02/Weenix-Virtual-File-System-Overview/). With the help of the detailed documentation, understanding the architecture of the Weenix VFS system and implementing the core functions are not that hard. However, debugging the kernel is torturing, because it is difficult to analyze the nodes, even with the help of GDB. 

Below are 3 core functions for implementing VFS. You can read the comments of the source code of Weenix to get more information.
```C
/* This takes a base 'dir', a 'name', its 'len', and a result vnode. */
int lookup(vnode_t *dir, const char *name, size_t len, vnode_t **result);

/* When successful this function returns data in the following "out"-arguments:
 *  o res_vnode: the vnode of the parent directory of "name"
 *  o name: the `basename' (the element of the pathname)
 *  o namelen: the length of the basename
 *
 * For example: dir_namev("/s5fs/bin/ls", &namelen, &name, NULL,
 * &res_vnode) would put 2 in namelen, "ls" in name, and a pointer to the
 * vnode corresponding to "/s5fs/bin" in res_vnode. */
int dir_namev(const char *pathname, size_t *namelen, const char **name, vnode_t *base, vnode_t **res_vnode);

/* This function uses pathname to find the specified vnode if it exists */
int open_namev(const char *pathname, int flag, vnode_t **res_vnode, vnode_t *base);
```


# Records
Handling ref_count of file and vnode is the nightmare. Though it seems trivial to manage ref_count at the first glance, it is the root causes of the most bugs.
## 1. Use vput() too early
The **unlink** system call is supposed to be implemented in this way: _Use dir_namev() to find the vnode of the directory containing the dir to be removed. Then call the containing dir's unlink v_op_. When testing our implementation of **unlink** system call, the parent directory vnode returns _ENOTDIR(Not a directory)_ unexpectedly. At first, we think that there are some bugs in our **mknod** implementation or in dir_namev() implementation, so we get the wrong vnode. However, we cannot find any related bugs when reviewing these code segments. We also made some similar mistakes in the implementation of other system calls.

```C
/* Code segment of unlink system call - Wrong */
int dir_res = dir_namev(path, &name_len, &name, NULL, &dir_node);
...

/* Since we increase the vref count of parent_dir_vnode
 * We need to use vput to decrease it
 */
vput(dir_node);

/* A directory vnode has the lookup method while a file node does not */
if(dir_node->vn_ops->lookup == NULL){
    ...
    return -ENOTDIR;
}
```

Luckily we have GDB. After printing all the fields of the parent directory vnode, we found out that the vn_ops field points to NULL after executing vput(). This is because vput() function will delete most of a vnode's fields when the ref_count s the vnode reaches zero, including the lookup function. 

The solution is to delay executing vput(). 
```C
/* Code segment of unlink system call - Correct */
int dir_res = dir_namev(path, &name_len, &name, NULL, &dir_node);
...

/* A directory vnode has the lookup method while a file node does not */
if(dir_node->vn_ops->lookup == NULL){
    vput(dir_node);
    ...
    return -ENOTDIR;
}
``` 


## 2. Cannot pass vfstest after running thrtest
We can pass vfstest successfully if we run it before running thrtest. However, if we run vfstest after running thrtest, we will get this error: 
> panic in fs/ramfs/ramfs.c:255 ramfs_read_vnode(): assertion failed: inode && inode->rf_ino == vn->vn_vno
> Kernel Halting.

Using gdb we can found out that the assertion is failed when executing path_equals("/", "/"), or more specifically, the assertion failed when we lookup "" in the root node. 

We finally find the reason why we can't lookup the empty name after thrtest. Because the rmdir function in ramfs doesn't clean the entry->rd_ino, while it cleans the entry->rd_name. 
For example,  before we run the thrtest, as for the root vnode, the entry(entry->rd_name, entry->rd_ino) can be {".", 0}, {"..", 0}, {"dev", 1}, {"vfstest-xxx", 6}, {"", 0}, {"", 0}, then all the same. But when we run thrtest 2, the entry of the root vnode can be like {".", 0}, {"..", 0}, {"dev", 1}, {"dir000", 27}, {"", 0},.... After we called rmdir function, the entry just becomes {".", 0}, {"..", 0}, {"dev", 1}, {"", 27}(we remove the directory), {"", 0}, then we run the vfstest and lookup the empty name, we can just get the wrong entry->rd_ino and the corresponding file is already closed. So in 
> KASSERT(inode && inode->rf_ino == vn->vn_vno) 
we will get the panic in 
> fs/ramfs/ramfs.c:255 ramfs_read_vnode(): assertion failed: inode && inode->rf_ino == vn->vn_vno
> Kernel Halting. 
since inode is NULL.

## 3. File Permission
Made some mistakes when handling permission of files. For example, write to a directory should not be allowed.

## 4. Root Vnode is not handled Correctly
When we create the idle process and init process, we should set the root_vnode as their current_working_direcotry and increase the ref_count of root_vnode. When cleaning up the idle and init process, we should also decrease the ref_count of root_vnode accordingly. 

All processes except pageout daemon process have a current working directory, and you should handle this corner case elegantly. Otherwise, you will get page fault or your kernel cannot shutdown cleanly.

 
# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)