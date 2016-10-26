---
title: linux内核笔记之内存管理
date: 2016-08-30 20:10:18
tags: 
- Linux
- 内核
categories: Linux
---
#### 非一致内存访问（NUMA）

我们习惯上认为计算机内存是一种均匀、共享的资源。在忽略硬件高速缓存的情况下，我们期望不管内存单元处于何处，也不管CPU处于何处，CPU对内存单元的访问都需要相同的时间。但是，在某些体系结构中并不总是成立。

在NUMA模型中，给定CPU对不同内存单元的访问时间可能不一样。系统的物理内存被划分为几个节点(node)。在一个单独的节点内，任一给定CPU访问页面所需的时间都是相同的。然而，对不同的CPU，这个时间可能就不同。每个节点的物理内存又可以分为几个管理区（zone）。

Linux支持NUMA模型，但是Linux将所有物理内存看作一个节点。在存放节点的链表中，只有一个元素。

#### 内存管理区

80x86体系结构存在两种硬件约束:

- ISA总线的DMA处理器只能对RAM的前16M寻址。
- 在具有大量RAM的现代32位计算机中，因为线性地址空间太小，CPU不能直接访问所有的物理内存。

为了应对这两种限制，Linux2.6把每个内存节点的物理内存划分为3个管理区。在80x86 UMA体系结构中的管理区为:

- ZONE_DMA(包含低于16M的内存页框)
- ZONE_NORMAL(包含高于16M低于896M的内存页框)
- ZONE_HIGHMEM(包含高于896M的所有内存页框)

ZONE_DMA中的页框可以由老式基于ISA的设备通过DMA使用。

ZONE_NORMAL中的页框全部映射到内核使用的第4个GB地址空间，这部分页框的映射方式很简单，例如物理地址为`0x10000000`,只需要直接加上3GB的地址，得到的线性地址为`0xc0000000`。

ZONE_HIGHMEM和ZONE_NORMAL都属于“常规”页框，但内核使用的第4个GB地址空间的最后128M才能映射到ZONE_HIGHMEM的部分页框。

#### 分区页框分配器

分区页框分配器是一个内核子系统，处理对连续页框组的内存分配请求。这部分内容对于理解Linux的内存管理十分关键，它的主要组成如下:
![](/images/linux-kernel-note/memory-manage-0.png)

#### 伙伴系统

linux内核的伙伴算法最大限度的减少了内存的碎片，其实应该说成是尽自己最大的努力减少了内存的碎片。其思想就是将物理内存分成11个块链表，每个链表包含的是大小为1,2,4,8...512,1024的连续页框块。举例来说要分配256个连续页框，会先到块大小为256的链表中查找空闲块，若有直接返回，若没有，去大小为512的链表中进行查找，将512大小块分为两部分，一部分返回，一部分插入256大小的链表中，若512大小的链表中还没有，到1024大小的链表中查找，取出256大小的块，将剩下的512,256的块分别插入到各个链表中，内存释放的过程则是相反的。

满足以下条件的两个快称为伙伴:

- 两个块具有相同的大小，记作b。
- 它们的物理地址是连续的。
- 第一个块的的第一个页框的物理地址是`2*b*2^12`的倍数。

#### 每CPU页框高速缓存（冷热页）

内核经常请求和释放`单个页框`，为了提升性能，每个内存管理区定义了一个“每CPU”高速缓存。冷页表示该空闲页已经不再高速缓存中了(一般是指L2 Cache)，热页表示该空闲页仍然在高速缓存中。冷热页是针对于每CPU的，每个zone中，都会针对于所有的CPU初始化一个冷热页的per-cpu-pageset.

#### 分配页框小结

当内核接收一个分配若干个页框的请求时，如果页框数量为1，则从每CPU页框高速缓存中分配一个页框。如果页框数大于1，则从伙伴系统中分配符合要求的若干个连续页框。其中的大致流程如下：

1. 管理区分配器选择一个合适的内存管理区。
2. 如果需要的页框数量为1，则从每CPU页框高速缓存中获取一个空闲的页框。一般情况下获取一个热页，但如果用于DMA等不需要CPU参与的请求，获取一个冷页。
3. 如果数量大于1，则从伙伴系统中选择一个大于0中的最小order链表，然后返回一个块。

如果冷热页中无空闲页框，则从伙伴系统中分配一些页框给冷热页机制。


#### slab机制

当我们需要分配一些小对象时，如果还使用页框分配器，那未免太浪费了，无论是从时间上还是空间上。slab分配器用于这些小对象的分配，一个slab由若干个页框构成，可由页框分配器分配。

每个slab根据其内部对象的大小，都被链接到一个相应的高速缓存描述符中:
![](/images/linux-kernel-note/memory-manage-1.png)
其中的高速缓存的大小为2的幂，从32B到131072B共13个级别，所链接的slab链表中的每个slab中的对象的大小都与其对应。其中每个级别的高速缓存描述符都存放于`malloc_sizes`数组中，数组的元素类型为`struct cache_sizes`：

```c
struct cache_sizes {
        size_t                  cs_size;
        struct kmem_cache       *cs_cachep;
#ifdef CONFIG_ZONE_DMA
        struct kmem_cache       *cs_dmacachep;
#endif
};
extern struct cache_sizes malloc_sizes[];
```
高速缓存描述符(kmem_cache)如下:

```c
struct kmem_cache {  
/* 1) per-cpu data, touched during every alloc/free */  
    struct array_cache *array[NR_CPUS];/*local cache*/  
/* 2) Cache tunables. Protected by cache_chain_mutex */  
    unsigned int batchcount;  
    unsigned int limit;  
    unsigned int shared;  
  
    unsigned int buffer_size;/*slab中对象大小*/  
    u32 reciprocal_buffer_size;/*slab中对象大小的倒数*/  
/* 3) touched by every alloc & free from the backend */  
  
    unsigned int flags;     /* constant flags */  
    unsigned int num;       /* # of objs per slab */  
  
/* 4) cache_grow/shrink */  
    /* order of pgs per slab (2^n) */  
    unsigned int gfporder;  
  
    /* force GFP flags, e.g. GFP_DMA */  
    gfp_t gfpflags;  
  
    size_t colour;/*着色块个数*/ /* cache colouring range */  
    unsigned int colour_off;/* cache的着色块的单位大小 */    /* colour offset */  
    struct kmem_cache *slabp_cache;  
    unsigned int slab_size;/*slab管理区大小,包含slab对象和kmem_bufctl_t数组*/  
    unsigned int dflags;        /* dynamic flags */  
  
    /* constructor func */  
    void (*ctor)(void *obj);  
  
/* 5) cache creation/removal */  
    const char *name;  
    struct list_head next; //链接下一个高速缓存描述符 
  
/* 6) statistics */  
#ifdef CONFIG_DEBUG_SLAB  
    unsigned long num_active;  
    unsigned long num_allocations;  
    unsigned long high_mark;  
    unsigned long grown;  
    unsigned long reaped;  
    unsigned long errors;  
    unsigned long max_freeable;  
    unsigned long node_allocs;  
    unsigned long node_frees;  
    unsigned long node_overflow;  
    atomic_t allochit;/*cache命中计数，在分配中更新*/  
    atomic_t allocmiss;/*cache未命中计数，在分配中更新*/  
    atomic_t freehit;  
    atomic_t freemiss;  
  
    /* 
     * If debugging is enabled, then the allocator can add additional 
     * fields and/or padding to every object. buffer_size contains the total 
     * object size including these internal fields, the following two 
     * variables contain the offset to the user object and its size. 
     */  
    int obj_offset;  
    int obj_size;  
#endif /* CONFIG_DEBUG_SLAB */  
  
    /* 
     * We put nodelists[] at the end of kmem_cache, because we want to size 
     * this array to nr_node_ids slots instead of MAX_NUMNODES 
     * (see kmem_cache_init()) 
     * We still use [MAX_NUMNODES] and not [1] or [0] because cache_cache 
     * is statically defined, so we reserve the max number of nodes. 
     */  
    struct kmem_list3 *nodelists[MAX_NUMNODES];  
    /* 
     * Do not add fields after nodelists[] 
     */  
};  
```
最后面的`struct kmem_list3`结构如下:

```c
struct kmem_list3 {  
    struct list_head slabs_partial;/*slab链表，包含空闲对象和已分配对象的slab描述符*/  
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

一个高速缓存描述符包含3条slab链表，通过`struct kmem_list3`字段链接，每个同大小（对象）的高速缓存描述符链接在一起：
![](/images/linux-kernel-note/memory-manage-2.png)

slab描述符如下：

```c
struct slab {  
    struct list_head list;  /*用于将slab链入kmem_list3的链表*/  
    unsigned long colouroff;/*该slab的着色偏移*/  
    void *s_mem;            /*指向slab中的第一个对象*/  
    unsigned int inuse;     /*已分配出去的对象*/  
    kmem_bufctl_t free;     /*下一个空闲对象的下标*/  
    unsigned short nodeid;  /*节点标识号*/  
};
```

一个slab中每个对象的大小都是相同的2的幂，其中的对象描述符为`kmem_bufctl_t`,它是一个无符号整数，用于表示对象在slab中的下标：

`typedef unsigned int kmem_bufctl_t;`

对象描述符存放在slab描述符后面，按照描述符是否存放在slab中，slab描述符和对象描述符的关系如下：
![](/images/linux-kernel-note/memory-manage-3.png)


高速缓存描述符中的`struct array_cache *array[NR_CPUS];/*local cache*/  `表示本地高速缓存数组，每个CPU对应一个本地高速缓存。

```c
struct array_cache {  
    unsigned int avail;/*本地高速缓存中可用的空闲对象数*/  
    unsigned int limit;/*空闲对象的上限*/  
    unsigned int batchcount;/*一次转入和转出的对象数量*/  
    unsigned int touched;   /*标识本地CPU最近是否被使用*/  
    spinlock_t lock;  
    void *entry[];  /*这是一个伪数组，便于对后面用于跟踪空闲对象的指针数组的访问 
             * Must have this definition in here for the proper 
             * alignment of array_cache. Also simplifies accessing 
             * the entries. 
             */  
};
```
当我们要求内核给我们分配一个小对象时，是从本地高速缓存中获取的。**本地高速缓存中的对象依次放置在该本地高速缓存描述符的地址后面。**从本地高速缓存获取对象时，首先根据当前CPU，从本地高速缓存数组中获取对应的到本地高速缓存描述符指针`ac`，`((void **)(ac+1))[--ac->avail]`返回的就是所分配的对象。

#### slab分配对象小结

当我们需要一个小对象时，首先根据该对象大小，选择合适的**高速缓存**(kmem_cache)。然后获取当前CPU id,到对应的**本地高速缓存**(array_cache)中获取对象。如果无空闲对象，则从空闲slab链表或者半空闲slab链表中分配一些对象，数量为本地高速缓存描述符中`batchcount `值。如果slab链表中的对象都已被使用，则从伙伴系统中分配若干个页框作为slab添加到高速缓存中，并且初始化，然后分配对象给本地高速缓存。

#### 总结

本文从整体上描述了Linux中的内存管理，并没有涉及到具体源码以及细节问题，更多细节待日后整理。



