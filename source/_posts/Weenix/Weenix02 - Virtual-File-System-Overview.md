---
title: Weenix02 - Virtual File System Overview
date: 2018-04-02 00:00:00
tags: [Weenix, File System, Operating System, C]
categories: [Weenix]
---

> Weenix is the kernel assignment of my OS course. I and my teammates Xiaowei Cheng, Yanghua, Huang, Donghua Yin finishes all the assignment together, including designing, coding, reviewing, debugging. It is a great pleasure to work with and learn from them.

# Overview
Before this post we have successfully implemented the thread related code for Weenix kernel. Now it is time to add support for the file system. Weenix has already provided us with the **Actual File System**(AFS) implementation, therefore, we only need to implement the code related to the **Virtual File System(VFS)**.

VFS is the abstraction layer beyond the AFS. VFS is like the interface while the AFS is the actual implementation. For example, VFS specifies that the AFS should implement read(), write() operations and how they should be implemented; different AFS can implement read() and write() in their own way. For Weenix, the AFS can be RAMFS, S5FS, etc.; for Linux, the AFS can be ext2, ext3, ext4, etc.; for Windows, the AFS can be FAT12, FAT32, FAT64, etc.. 

Below are the 4 most important structs for VFS: struct **proc**, struct **file**, struct **vnode**, struct **fs**. For detailed information, please refer to the source code and the documentation of Weenix from Brown University.

<img src="https://bytebucket.org/LarryTaoWang/pictureofblog/raw/f09ad8a1f1a76f16ab34c2dd0ac4702b4a324b00/Weenix/VFS%20structs.png">



# Important Structs 
To be short, proc struct stores a table of file descriptors, which can be used to find file struct. File struct stores the information of a file descriptor, which can be used to find a vnode struct. A vnode can be used to find an actual file system, struct fs. 

## Struct proc
Every process struct* has its own file descriptor table, which is shared by its threads. When a child process is created, it will inherit this table(not the file!) from the parent process. 

We can use p_files[NFILES] attribute to find the corresponding file struct of a file descriptor.


## Struct file
Every entry in this table maps to a file struct, which stores the meta-data of the file descriptor and the associated vnode. One thing to notice is that the mode of a file descriptor is independent of the mode of a file. For example, if we open file A with only read permission, i.e.,
```C
int fd = open("A", O_RDONLY, 0);
```
Even though file "A" is created with both write and read permissions, you can only do read operation to this file, because the file descriptor is read-only.

Another important attribute is ref_count, which is the number of references to this struct. Different file descriptor may point to the same file struct, i.e., dup or creating a child process. When the ref_count is down to 0, the file struct will be cleaned up. 

<img src="https://bytebucket.org/LarryTaoWang/pictureofblog/raw/f09ad8a1f1a76f16ab34c2dd0ac4702b4a324b00/Weenix/File%20Descriptor%20After%20Fork.png">

We can use node attribute to find the corresponding vnode struct of a file.

## Struct vnode
One important attribute is the **vnode_ops**. VFS defines the operations an AFS should implement but different AFS can have different implementations. When we call the operation in VFS, the implementation of the corresponding AFS instance will be called. This feature is called **polymofism**. If you have worked with OOD languages such as C++ or Java, you should be quite familiar with this concept. C implements polymorphism with the function pointer. For further information, you can google "Objected-Oriented C".

**vn_ref_count** is the number of references to this vnode. But this can be much more complexed than ref_count in fs struct, because Weenix caches file into memories, and the memory pages will influence the ref_count of vnode. 

Two important things to remember is that
 1. ref_count_of_vnode = ref_count_of_file + ref_count_from_memory_page
 2. If ref_count_of_vnode = ref_count_from_memory_page, then no file is using this vnode, that is, only the cache is using the vnode. At this time we can safely free up this vnode.

If you want more information, the Weenix documentation has an excellent introduction to this attribute as well as the corner case and optimization. 

We can use **fs** attribute to find the corresponding vnode struct of a file.

One thing I want to complain is the design of the vnode struct and fs struct. These structs do not have a name attribute, which is supposed to correspond to the path name. The lacking path information makes debugging much harder. When using gdb to print the information of a node, since we don't know the path name, we need to guess which pathname it belongs by analyzing the node is manually.

 
# Reference
> [1] _Operating Systems in Depth_, Thomas W. Doeppner, Brown University  
> [2] [Weenix Documentation
](https://cs.brown.edu/courses/cs167/content/weenix-doc.pdf)