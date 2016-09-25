---
title: Linux slab机制中的colouroff
date: 2016-06-29 18:52:25
tags:
- Linux
- 理解计算机
categories: Linux
---

#### slab简介

slab分配器是Linux内存管理中非常重要和复杂的一部分，其工作是针对一些经常分配并释放的对象，如进程描述符等，这些对象的大小一般比较小，如果直接采用伙伴系统来进行分配和释放，不仅会造成大量的内碎片，而且处理速度也太慢。而slab分配器是基于对象进行管理的，相同类型的对象归为一类(如进程描述符就是一类)，每当要申请这样一个对象，slab分配器就从一个slab列表中分配一个这样大小的单元出去，而当要释放时，将其重新保存在该列表中，而不是直接返回给伙伴系统。slab分配对象时，会使用最近释放的对象内存块，因此其驻留在CPU高速缓存的概率较高。

#### slab数据结构
用于描述和管理cache的数据结构是struct kmem_cache，在struct kmem_cache中有一个struct kmem_list3*nodelists[`MAX_NUMNODES`]，
每个nodelists中都有着3条slab双向链表。
``` c
struct kmem_list3 {  
    struct list_head slabs_partial; /*slab链表，包含空闲对象和已分配对象的slab描述符*/  
    struct list_head slabs_full;   /*slab链表，只包含非空闲的slab描述符*/  
    struct list_head slabs_free;   /*slab链表，只包含空闲的slab描述符*/  
    unsigned long free_objects;    /*高速缓存中空闲对象的个数*/  
    unsigned int free_limit;       /*空闲对象的上限*/  
    unsigned int colour_next;       /*下一个slab使用的颜色*/  
    spinlock_t list_lock;  
    struct array_cache *shared; /* shared per node */  
    struct array_cache **alien; /* on other nodes */  
    unsigned long next_reap;    /* updated without locking */  
    int free_touched;       /* updated without locking */  
};  
```

（其中的`MAX_NUMNODES`在UMA机制中为1，Linux把物理内存划分为三个层次来管理：存储节点（Node）、管理区（Zone）和页面（Page）。UMA对应一致存储结构，它只需要一个Node就可以描述当前系统中的物理内存）。

每个slab由一个或多个页框组成
``` c
struct slab {  
    struct list_head list;  /*用于将slab链入kmem_list3的链表*/  
    unsigned long colouroff;/*该slab的着色偏移*/  
    void *s_mem;            /*指向slab中的第一个对象*/  
    unsigned int inuse;     /*已分配出去的对象*/  
    kmem_bufctl_t free;     /*下一个空闲对象的下标*/  
    unsigned short nodeid;  /*节点标识号*/  
};  
```

**其中的colouroff代表的是什么呢？**

#### 为什么要colouroff?

比如cache line 32 字节,  字节0-31一次从内存写入/读取, 字节32-63一次从内存写入/读取…..


另外cache对应到内存位置不是任意的

Cache 地址0 对应到 内存地址0 , 32 ,64 ….

Cache 地址1 对应到 内存地址1 , 33 ,65 ….

…


一个slab大小肯定是整数页,所以起始地址末12位为零, 即都于cache0 对应.

然后2个slab的每一个obj大小一样, 所以2个slab每个obj都对应相同的cache line.

这样2个位置相同的obj都要频繁访问,比较容易使cache来回刷新,效率降低.


着色就是在第二个slab的起始位置空一个cache line出来, 这样2个slab每个obj对应的cache错开一个, 这样2个位置相同的obj即使频繁访问,也不会用一个相同cache line.

#### cache line

一个L1 DATA CACHE相当于一块小的内存，`我们假设它为16K大`，它会与一般物理内存交互。  
它和内存交互一般一次传输16个字节(32个字节),也就是:  


CACHE 字节0-15一次写到/读取物理内存 ，字节16-31一次写到/读取物理内存.32-47 ... ...  

这些一次被传输的字节被称为cache line。  

**注意!!!这里的cache line中的cache指的是cpu中的L1 DATA CACHE ,而不是slab算法中的结构cache,这两个容易混淆。下面提到的cache都是指L1 DATA CACHE,而不是slab算法中的结构cache**  

--------------------------------------------------------------  
#### cache line对齐和colouroff

另外，cache写到物理内存的位置不是任意的，  
我们假定内存为64K,那么cache地址0的数值只能和物理内存的地址0, 16K, 32K交互；cache地址1的数值只能和物理内存的地址1, 16K+1, 32K+1交互  
。。。 。。。cache地址16K-1的数值只能和物理内存的地址6K-1, 16K+16K-1, 32K+16K -1交互  

这说明了两点:  

(1)假设对象A的一个字段长为16个字节，如果它放在物理地址 0-15,那么它将和cache的第一个cache line 交互，如果放在物理地址 8-23,那么  
如果CPU要访问这个字段，必须将第一个和第二个cache line 都读入，才能获得这个字段的信息，显然这样速度慢，所以一般字段需要`cache line对齐`，  
在这里就是16个字节对齐。  


(2)关于colouroff  

一般一个对象某些字段访问频繁些。  
假定一个cache(这个cache指slab的cache,不是上面提到CPU的L1 DATA CACHE)占用5个页面也就是20K.  
假定其中对象大小为32个字节，前16个字节访问频繁许多。  

假定对象A起始于物理地址0，对象C起始于31,对象B起始于物理地址16K,那么对象A，对象B的前16个字节都和第一个cache line 交互，后16个字节都和第二个cache line 交互  
对象C前16个字节与第3个cache交互。  

我们假定内核访问A后就访问B,再访问A,交错进行，并且前16个字节次数都是50次，后16个为10次。C也是。  

这样第一个cache line 要交互100次，第二个20次，一共120次。  

如果让对象B向后移动16个字节，也就是对象B的前16个字节与第二个cache line 交互，后16个与第3个交互。  
那么第一个为2次，因为只有开头结尾2次要与内存交互，其它每次都在L1 DATACACHE 中写就可以了。第2个cache line为20次左右(后面的只须在CACHE中读写),第3个cache line为20次，  
3个line一共才41次，你不妨仔细模拟一下。  

所以进行错位能降低CACHE的交互次数，从而提高CPU处理速度能力。  

这个错位(也就是上面的16个字节)就是colouroff.

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 
