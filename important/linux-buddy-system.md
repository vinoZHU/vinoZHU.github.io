---
title: Linux中的Buddy System
date: 2016-06-28 10:49:31
tags:
- Linux
- 内存管理
categories: Linux
---

#### 什么是Buddy System?

Buddy System是Linux Kernel 进行物理内存页管理的一个子系统。在Buddy System中，管理的一个基本单位是block，每一个block有若干个连续的物理页组成，物理页的个数为2^n，这个n在buddy system中被称为order。`相同order的block，挂载一条双向链表上。`当某个block空闲时，只要发现对应的伙伴也是空闲的，就和伙伴组成一个页数为2^（n+1）的block，挂载在order为(n+1)的双向链表上，换句话说一个页数为2^n的block，是由两个页数为2^（n-1）的伙伴block组成的。因此，一个block的伙伴肯定是和这个block在 **物理地址上是连续** 的。在Linux中，order的默认的取值范围是[0,10]。

linux中用`struct zone`来描述一个内存管理区(*ZONE_DMA:0~16MB  ZONE_NORMAL:16MB~896MB  ZONE_HIGHMEM:896MB~end*)
其中的struct free_area中的`每一个元素`表示的是由相同order的block构成的`双向链表`。free_area[i]表示该free_area处的链表的order为i。
``` c
struct zone {  
    ...  
         ...      
    struct free_area    free_area[MAX_ORDER];  
    ...  
    ...  
}
```
``` c
struct free_area {  
  //每个free_area内有MIGRATE_TYPES个双向链表，根据迁移类型划分()*迁移类型请查看文末*）。
  //free_list和struct page中的lru相连
    struct list_head    free_list[MIGRATE_TYPES];  
    unsigned long       nr_free;  //空闲块数量
};

```
在buddy system中，存入一个页会进行一次伙伴整合的迭代操作，直到不能再合并为止。那么，如何查找某个block的伙伴呢？在Linux中，一个block的伙伴信息记录在该block的`第一个物理页`的`struct page`上。也就是说，假若一个页在伙伴系统中，page->private的值为n,则后面连续的2^(n-1)个页面就在buddy system中。一个页面在buddy System中，就代表了该页是空闲的，可以被分配。

#### 查找伙伴
内核代码如下：
``` c
mm/page_alloc.c
		buddy_idx = __find_buddy_index(page_idx, order);
		buddy = page + (buddy_idx - page_idx);    
```

``` c
static inline unsigned long
__find_buddy_index(unsigned long page_idx, unsigned int order)
{
	return page_idx ^ (1 << order);
}
```
查找一个页数为2^n的block的伙伴计算方法是buddy_index=page_index^(1<<n)(此处的^表示异或)，buddy_index 表示伙伴块首页的页索引，page_index是该block首页的页索引，n代表order。

举例：
![](/images/linux/linux-buddy-system-2.png)

![](/images/linux/linux-buddy-system-3.png)
####  从buddy System中分配页

buddy System用来管理空闲的物理内存，接受进程的内存页申请。Linux提供给用户进程申请内存的接口函数中包含两个参数：一个是`GFP_***`，另一个是`order`。前者表示分配标志信息，从标志中可以知道进程要从哪个zone中申请物理内存，并在内存紧缺的情况下指导内存何时回收内存；后者表示需要分配2^order个连续的物理页。
buddy system根据参数order，在要求的zone中查找对应的链表。若该order的双向链表非空，则把该双向链表上的第一个block从buddy system中脱链，同时更新该双向链表上的block的数量，完成内存分配。若该order的双向链表为空，则查找order+1的双向链表，如果order+1的双向链表为空，则查找order+2的双向链表，直到查找到。假设查找到了第n个双向链表(n>order)，由于查找到的连续物理内存页的数量大于order，因此将该block一分为二，将后半部分block挂载到2^(n-1)的双向链表上，将前半部分再一分为二，将前半部分的后半部分挂载到2^(n-2)的双向链表上，将前半部分的前半部分再一分为二。。。直到与请求的order相等为止。

#### 释放页到buddy System

当非空闲的页被进程释放后，需要把页放回到伙伴系统中。由于block是作为一个整体被申请的，因此其释放也是以block为单位。若该block有2^n个连续的物理页，在释放的时候，根据buddy_index = page_index^(1<<n)查找该block的order为n的伙伴。若查找到伙伴，则将两个order为n的block合并为一个order为n+1的block。同时再查找order为n+1 block的伙伴，若找到则两个order为n+1的block继续合并…直到其不能合并为止，插入到最终order的双向链表的最前端。释放的迭代过程是分配页的逆过程.

#### linux中的迁移类型

在2.6.24之前的内核版本中，free_area结构中只有一个free_list数组，而从2.6.24开始，free_area结构中存有MIGRATE_TYPES个free_list,这些数组是根据页框的移动性来划分的，为什么要进行这样的划分呢?实际上也是为了减少碎片而提出的，我们考虑下面的情况：

![](/images/linux/linux-buddy-system-0.png)
图中一共有32个页，只分配出了4个页框，但是能够分配的最大连续内存也只有8个页框(因为伙伴系统分配出去的内存必须是2的整数次幂个页框)，内核解决这种问题的办法就是将不同类型的页进行分组。分配出去的页面可分为三种类型：

- 不可移动页(Non-movable pages)：这类页在内存当中有固定的位置，不能移动。内核的核心分配的内存大多属于这种类型
- 可回收页(Reclaimable pages)：这类页不能直接移动，但可以删除，其内容页可以从其他地方重新生成，例如，映射自文件的数据属于这种类型，针对这种页，内核有专门的页面回收处理
- 可移动页:这类页可以随意移动，用户空间应用程序所用到的页属于该类别。它们通过页表来映射，如果他们复制到新的位置，页表项也会相应的更新，应用程序不会注意到任何改变。

假如上图中大部分页都是可移动页，而分配出去的四个页都是不可移动页，由于不可移动页插在了其他类型页的中间，就导致了无法从原本空闲的连续内存区中分配较大的内存块。考虑下图的情况:
![](/images/linux/linux-buddy-system-1.png)

将可回收页和不可移动页分开，这样虽然在不可移动页的区域当中无法分配大块的连续内存，但是可回收页的区域却没有受其影响，可以分配大块的连续内存。


`内核对于迁移类型的定义如下:`

<mmzone.h>

``` c
#define MIGRATE_UNMOVABLE     0  
#define MIGRATE_RECLAIMABLE   1  
#define MIGRATE_MOVABLE       2  
#define MIGRATE_PCPTYPES      3 /* the number of types on the pcp lists */  
#define MIGRATE_RESERVE       3  
#define MIGRATE_ISOLATE       4 /* can't allocate from here */  
#define MIGRATE_TYPES         5  
 ```

前三种类型已经介绍过

`MIGRATE_PCPTYPES`是per_cpu_pageset，即用来表示每CPU页框高速缓存的数据结构中的链表的迁移类型数目

`MIGRATE_RESERVE`是在前三种的列表中都没用可满足分配的内存块时，就可以从MIGRATE_RESERVE分配

`MIGRATE_ISOLATE`用于跨越NUMA节点移动物理内存页，在大型系统上，它有益于将物理内存页移动到接近于是用该页最频繁地CPU

`MIGRATE_TYPES`表示迁移类型的数目
当一个指定的迁移类型所对应的链表中没有空闲块时，将会按以下定义的顺序到其他迁移类型的链表中寻找

参考：
- [认识Linux物理内存管理系统–Buddy System](http://blog.tek-life.com/%E8%AE%A4%E8%AF%86linux%E7%89%A9%E7%90%86%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E7%BB%9F-buddy-system/)
- [Linux伙伴系统(一)--伙伴系统的概述](http://blog.csdn.net/vanbreaker/article/details/7605367)
- [内存伙伴系统之查找伙伴算法](http://rick_stone.leanote.com/post/%E5%86%85%E5%AD%98%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E4%B9%8B%E6%9F%A5%E6%89%BE%E4%BC%99%E4%BC%B4)

本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/3.0/">知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议</a>进行许可,转载请在正文明显处注明原文地址。
