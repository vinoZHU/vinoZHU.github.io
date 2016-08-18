---
title: Linux中的冷热页机制概述
date: 2016-06-28 19:01:36
tags:
- Linux
- 理解计算机
categories: Linux
---

#### 什么是冷热页？

在Linux Kernel的物理内存管理的[Buddy System](/2016/06/28/linux-buddy-system/)中，引入了冷热页的概念。冷页表示该空闲页已经不再高速缓存中了(一般是指L2 Cache)，热页表示该空闲页仍然在高速缓存中。冷热页是针对于每CPU的，每个zone中，都会针对于所有的CPU初始化一个冷热页的per-cpu-pageset.

#### 为什么要有冷热页？

作用有3点：
- Buddy Allocator在分配order为0的空闲页的时候，如果分配一个热页，那么由于该页已经存在于L2 Cache中了。CPU写访问的时候，不需要先把内存中的内容读到Cache中，然后再写。如果分配一个冷页，说明该页不在L2 Cache中。一般情况下，尽可能用热页，是容易理解的。什么时候用冷页呢？While allocating a physical page frame, there is a bit specifying whether we would like a hot or a cold page (that is, a page likely to be in the CPU cache, or a page not likely to be there). If the page will be used by the CPU, a hot page will be faster. If the page will be used for device `DMA` the CPU cache would be invalidated anyway, and a cold page does not waste precious cache contents.
简单翻译一下：当内核分配`一个`物理页框时，有一些规范来约束我们是分配热页还是冷页。当页框是CPU使用的，则分配热页。当页框是DMA设备使用的，则分配冷页。因为DMA设备不会用到CPU高速缓存，所以没必要使用热页。

- Buddy System在给某个进程分配某个zone中空闲页的时候，首先需要用自旋锁锁住该zone,然后分配页。这样，如果多个CPU上的进程同时进行分配页，便会竞争。引入了per-cpu-set后，当多个CPU上的进程同时分配页的时候，竞争便不会发生，提高了效率。另外当释放单个页面时，空闲页面首先放回到per-cpu-pageset中，以减少zone中自旋锁的使用。当页面缓存中的页面数量超过阀值时，再将页面放回到伙伴系统中。

- 使用每CPU冷热页还有一个好处是，能保证某个页一直黏在1个CPU上，这有助于提高Cache的命中率。

#### 冷热页的数据结构
``` c
struct per_cpu_pages {
        int count;              // number of pages in the list

        int high;               // high watermark, emptying needed

        int batch;              // chunk size for buddy add/remove

         // Lists of pages, one per migrate type stored on the pcp-lists
         每个CPU在每个zone上都有MIGRATE_PCPTYPES个冷热页链表（根据迁移类型划分）
         struct list_head lists[MIGRATE_PCPTYPES];

 };
```
在Linux中，对于UMA的架构，冷热页是在一条链表上进行管理。热页在前，冷页在后。CPU每释放一个order为0的页，如果per-cpu-pageset中的页数少于其指定的阈值，便会将释放的页插入到冷热页链表的开始处。这样，之前插入的热页便会随着其后热页源源不断的插入向后移动，其页由热变冷的几率便大大增加。

#### 怎样分配冷热页

在分配order为0页的时候(冷热页机制只处理单页分配的情况)，先找到合适的zone,然后根据需要的`migratetype`类型定位冷热页链表（每个zone，对于每个cpu,有3条冷热页链表，对应于：MIGRATE_UNMOVABLE、MIGRATE_RECLAIMABLE、MIGRATE_MOVABLE）。若需要热页，则从链表头取下一页（此页最“热”）；若需要冷页，则从链表尾取下一页（此页最“冷”）。

分配函数（关键部分已添加注释）：
``` c
 /*
  * Really, prep_compound_page() should be called from __rmqueue_bulk().  But
  * we cheat by calling it from here, in the order > 0 path.  Saves a branch
  * or two.
  */
 static inline
 struct page *buffered_rmqueue(struct zone *preferred_zone,
             struct zone *zone, int order, gfp_t gfp_flags,
             int migratetype)
 {
     unsigned long flags;
     struct page *page;
     //分配标志是__GFP_COLD才分配冷页
     int cold = !!(gfp_flags & __GFP_COLD);

 again:
     if (likely(order == 0)) {
         struct per_cpu_pages *pcp;
         struct list_head *list;

         local_irq_save(flags);
         pcp = &this_cpu_ptr(zone->pageset)->pcp;
         list = &pcp->lists[migratetype];
         if (list_empty(list)) {
           //如果缺少页，则从Buddy System中分配。
             pcp->count += rmqueue_bulk(zone, 0,
                     pcp->batch, list,
                     migratetype, cold);
             if (unlikely(list_empty(list)))
                 goto failed;
         }

         if (cold)
         //分配冷页时，从链表尾部分配，list为链表头，list->prev表示链表尾
             page = list_entry(list->prev, struct page, lru);
         else
         //分配热页时，从链表头分配
             page = list_entry(list->next, struct page, lru);
        //分配完一个页框后从冷热页链表中删去该页
         list_del(&page->lru);
         pcp->count--;
     } else {//如果order!=0(页框数>1)，则不从冷热页链表中分配
         if (unlikely(gfp_flags & __GFP_NOFAIL)) {
             /*
              * __GFP_NOFAIL is not to be used in new code.
              *
              * All __GFP_NOFAIL callers should be fixed so that they
              * properly detect and handle allocation failures.
              *
              * We most definitely don't want callers attempting to
              * allocate greater than order-1 page units with
              * __GFP_NOFAIL.
              */
             WARN_ON_ONCE(order > 1);
         }
         spin_lock_irqsave(&zone->lock, flags);
         page = __rmqueue(zone, order, migratetype);
         spin_unlock(&zone->lock);
         if (!page)
             goto failed;
         __mod_zone_page_state(zone, NR_FREE_PAGES, -(1 << order));
     }

     __count_zone_vm_events(PGALLOC, zone, 1 << order);
     zone_statistics(preferred_zone, zone, gfp_flags);
     local_irq_restore(flags);

     VM_BUG_ON(bad_range(zone, page));
     if (prep_new_page(page, order, gfp_flags))
         goto again;
     return page;

 failed:
     local_irq_restore(flags);
     return NULL;
 }
```

参考：
- [认识Linux/ARM 中的冷热页](http://blog.tek-life.com/%E8%AE%A4%E8%AF%86linuxarm-%E4%B8%AD%E7%9A%84%E5%86%B7%E7%83%AD%E9%A1%B5/)
- Linux源码

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流,转载请在正文明显处注明原文地址，谢谢！</font>

