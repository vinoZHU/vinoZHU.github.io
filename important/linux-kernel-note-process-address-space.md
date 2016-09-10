---
title: linux内核笔记之进程地址空间
date: 2016-09-10 13:34:16
tags: 
- Linux
- 内核
categories: Linux
---

#### 进程的地址空间

进程的地址空间由允许进程使用的全部线性地址组成，在32位系统中为0~3GB，每个进程看到的线性地址集合是不同的。

内核通过线性区的资源（数据结构）来表示线性地址区间，线性区是由起始线性地址，长度和一些访问权限来描述的。线性区的大小为页框的整数倍，起始地址为4096的整数倍。

下图展示了x86 Linux 进程的地址空间组织结构：
![](/images/linux-kernel-note/process-address-space-0.png)

- 正文段 .text ，这是CPU执行的机器指令部分。通常正文段是共享的，而且是只读的，以防止程序修改其自身的指令。
- 数据段 .data。数据段包含了程序中需要明确赋初值的变量。
- 非初始化数据段 bss。bss 起始于 IBM 704汇编语言中的 Block Storage Start 指令的首字母缩写,并且沿用至今。

#### 线性区描述符

进程地址空间中的堆、栈等，就是一个线性区，线性区的结构类型为`struct vm_area_struct`：
http://lxr.free-electrons.com/source/include/linux/mm_types.h#L299

```c
299 struct vm_area_struct {
300         /* The first cache line has the info for VMA tree walking. */
301 
302         unsigned long vm_start;         /* Our start address within vm_mm. */
303         unsigned long vm_end;           /* The first byte after our end address
304                                            within vm_mm. */
305 
306         /* linked list of VM areas per task, sorted by address */
307         struct vm_area_struct *vm_next, *vm_prev;
308 
309         struct rb_node vm_rb;
310 
311         /*
312          * Largest free memory gap in bytes to the left of this VMA.
313          * Either between this VMA and vma->vm_prev, or between one of the
314          * VMAs below us in the VMA rbtree and its ->vm_prev. This helps
315          * get_unmapped_area find a free area of the right size.
316          */
317         unsigned long rb_subtree_gap;
318 
319         /* Second cache line starts here. */
320 
321         struct mm_struct *vm_mm;        /* The address space we belong to. */
322         pgprot_t vm_page_prot;          /* Access permissions of this VMA. */
323         unsigned long vm_flags;         /* Flags, see mm.h. */
324 
325         /*
326          * For areas with an address space and backing store,
327          * linkage into the address_space->i_mmap interval tree.
328          */
329         struct {
330                 struct rb_node rb;
331                 unsigned long rb_subtree_last;
332         } shared;
333 
334         /*
335          * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
336          * list, after a COW of one of the file pages.  A MAP_SHARED vma
337          * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
338          * or brk vma (with NULL file) can only be in an anon_vma list.
339          */
340         struct list_head anon_vma_chain; /* Serialized by mmap_sem &
341                                           * page_table_lock */
342         struct anon_vma *anon_vma;      /* Serialized by page_table_lock */
343 
344         /* Function pointers to deal with this struct. */
345         const struct vm_operations_struct *vm_ops;
346 
347         /* Information about our backing store: */
348         unsigned long vm_pgoff;         /* Offset (within vm_file) in PAGE_SIZE
349                                            units */
350         struct file * vm_file;          /* File we map to (can be NULL). */
351         void * vm_private_data;         /* was vm_pte (shared mem) */
352 
353 #ifndef CONFIG_MMU
354         struct vm_region *vm_region;    /* NOMMU mapping region */
355 #endif
356 #ifdef CONFIG_NUMA
357         struct mempolicy *vm_policy;    /* NUMA policy for the VMA */
358 #endif
359         struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
360 };
```
- `vm_start`:线性区的起始地址
- `vm_end `:线性区的结束地址
- `vm_rb `:作为红黑树中的一个节点使用
- `vm_mm`:指向所在的内存描述符
- `vm_page_prot`:线性区中页框的访问权限
- `vm_flags `:线性区的标志
- `vm_next, vm_prev`:分别指向线性区链表中的下一个和上一个线性区描述符
- ... ...

#### 内存描述符

内存描述符中包含了与进程地址空间有关的所有信息，结构类型为`struct mm_struct`:
http://lxr.free-electrons.com/source/include/linux/mm_types.h#L395

```c
395 struct mm_struct {
396         struct vm_area_struct *mmap;            /* list of VMAs */
397         struct rb_root mm_rb;
398         u32 vmacache_seqnum;                   /* per-thread vmacache */
399 #ifdef CONFIG_MMU
400         unsigned long (*get_unmapped_area) (struct file *filp,
401                                 unsigned long addr, unsigned long len,
402                                 unsigned long pgoff, unsigned long flags);
403 #endif
404         unsigned long mmap_base;                /* base of mmap area */
405         unsigned long mmap_legacy_base;         /* base of mmap area in bottom-up allocations */
406         unsigned long task_size;                /* size of task vm space */
407         unsigned long highest_vm_end;           /* highest vma end address */
408         pgd_t * pgd;
409         atomic_t mm_users;                      /* How many users with user space? */
410         atomic_t mm_count;                      /* How many references to "struct mm_struct" (users count as 1) */
411         atomic_long_t nr_ptes;                  /* PTE page table pages */
412 #if CONFIG_PGTABLE_LEVELS > 2
413         atomic_long_t nr_pmds;                  /* PMD page table pages */
414 #endif
415         int map_count;                          /* number of VMAs */
416 
417         spinlock_t page_table_lock;             /* Protects page tables and some counters */
418         struct rw_semaphore mmap_sem;
419 
420         struct list_head mmlist;                /* List of maybe swapped mm's.  These are globally strung
421                                                  * together off init_mm.mmlist, and are protected
422                                                  * by mmlist_lock
423                                                  */
424 
425 
426         unsigned long hiwater_rss;      /* High-watermark of RSS usage */
427         unsigned long hiwater_vm;       /* High-water virtual memory usage */
428 
429         unsigned long total_vm;         /* Total pages mapped */
430         unsigned long locked_vm;        /* Pages that have PG_mlocked set */
431         unsigned long pinned_vm;        /* Refcount permanently increased */
432         unsigned long data_vm;          /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
433         unsigned long exec_vm;          /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
434         unsigned long stack_vm;         /* VM_STACK */
435         unsigned long def_flags;
436         unsigned long start_code, end_code, start_data, end_data;
437         unsigned long start_brk, brk, start_stack;
438         unsigned long arg_start, arg_end, env_start, env_end;
439 
440         unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */
441 
442         /*
443          * Special counters, in some configurations protected by the
444          * page_table_lock, in other configurations by being atomic.
445          */
446         struct mm_rss_stat rss_stat;
447 
448         struct linux_binfmt *binfmt;
449 
450         cpumask_var_t cpu_vm_mask_var;
451 
452         /* Architecture-specific MM context */
453         mm_context_t context;
454 
455         unsigned long flags; /* Must use atomic bitops to access the bits */
456 
457         struct core_state *core_state; /* coredumping support */
458 #ifdef CONFIG_AIO
459         spinlock_t                      ioctx_lock;
460         struct kioctx_table __rcu       *ioctx_table;
461 #endif
462 #ifdef CONFIG_MEMCG
463         /*
464          * "owner" points to a task that is regarded as the canonical
465          * user/owner of this mm. All of the following must be true in
466          * order for it to be changed:
467          *
468          * current == mm->owner
469          * current->mm != mm
470          * new_owner->mm == mm
471          * new_owner->alloc_lock is held
472          */
473         struct task_struct __rcu *owner;
474 #endif
475 
476         /* store ref to file /proc/<pid>/exe symlink points to */
477         struct file __rcu *exe_file;
478 #ifdef CONFIG_MMU_NOTIFIER
479         struct mmu_notifier_mm *mmu_notifier_mm;
480 #endif
481 #if defined(CONFIG_TRANSPARENT_HUGEPAGE) && !USE_SPLIT_PMD_PTLOCKS
482         pgtable_t pmd_huge_pte; /* protected by page_table_lock */
483 #endif
484 #ifdef CONFIG_CPUMASK_OFFSTACK
485         struct cpumask cpumask_allocation;
486 #endif
487 #ifdef CONFIG_NUMA_BALANCING
488         /*
489          * numa_next_scan is the next time that the PTEs will be marked
490          * pte_numa. NUMA hinting faults will gather statistics and migrate
491          * pages to new nodes if necessary.
492          */
493         unsigned long numa_next_scan;
494 
495         /* Restart point for scanning and setting pte_numa */
496         unsigned long numa_scan_offset;
497 
498         /* numa_scan_seq prevents two threads setting pte_numa */
499         int numa_scan_seq;
500 #endif
501 #if defined(CONFIG_NUMA_BALANCING) || defined(CONFIG_COMPACTION)
502         /*
503          * An operation with batched TLB flushing is going on. Anything that
504          * can move process memory needs to flush the TLB when moving a
505          * PROT_NONE or PROT_NUMA mapped page.
506          */
507         bool tlb_flush_pending;
508 #endif
509         struct uprobes_state uprobes_state;
510 #ifdef CONFIG_X86_INTEL_MPX
511         /* address of the bounds directory */
512         void __user *bd_addr;
513 #endif
514 #ifdef CONFIG_HUGETLB_PAGE
515         atomic_long_t hugetlb_usage;
516 #endif
517 #ifdef CONFIG_MMU
518         struct work_struct async_put_work;
519 #endif
520 };
```
- `mmap `:线性区描述符链表中的头元素
- `mm_rb `:线性区描述符所在红黑树的根
- `get_unmapped_area`:在进程地址空间中搜索有效线性地址区间的方法
- `mmap_base `:标识第一个分配的匿名线性区或文件内存映射的线性地址
- `task_size `:进程地址空间的大小
- `highest_vm_end `:能使用的最高线性地址
- `pgd `:指向页全局目录
- `mm_users `:次使用计数器
- `mm_count `:主使用计数器
- `nr_ptes `:页表项数量
- `map_count `：线性区数量
- `mmlist `:链接内存描述符链表中的相邻描述符
- ... ...


#### 线性区相关

进程所拥有的所有线性区通过一个简单地链表链接在一起，链表中的线性区按内存地址升序排列。内核通过进程的内存描述符的`mmap`字段找到线性区链表的第一个线性区。
![](/images/linux-kernel-note/process-address-space-2.png)

内核频繁执行的一个操作就是查找包含指定线性地址的的线性区，虽然可以通过遍历链表来查找，但是当线性区数量很庞大时，例如面向对象的数据库，此时的效率会变得非常低效。

Linux2.6把内存描述符存放在红黑树的数据结构中，当插入或删除一个线性区描述符时，内核通过红黑树搜索前后元素，并用搜索结果快速更新链表而不用扫描链表。一般来说，红黑树用来确定含有指定地址的线性区，而链表通常在扫描整个线性区集合的时候使用。

#### 内存描述符相关

进程、内存描述符、线性区描述符、线性地址之间的关系如下:
![](/images/linux-kernel-note/process-address-space-1.png)

所有内存描述符存放在一个双向链表中，每个描述符中的`mmlist`字段存放链表中相邻元素的地址。链表的第一个元素是`init_mm`的`mmlist`字段，`init_mm`是初始化阶段进程0所使用的内存描述符。

`mm_users`字段存放共享`mm_struct`数据结构的轻量级进程（线程）的个数，`mm_count`字段是内存描述符的主使用器，`mm_users`的所有使用者在`mm_count`中只占有一个单位，也就是说多个线程只使得`mm_count`的值增加了1。假如一个内核线程使用了该内存描述符，则`mm_count`的值增加1。

对于内核线程来说，因为仅运行在内核态，所以永远不会访问低于`TASK_SIZE`（3GB）的地址。每个进程的描述符中包含了两种内存描述符指针:`mm`和`active_mm`。对于普通进程，两者都指向进程的内存描述符，而内核线程的`active_mm`指向进程的内存描述符，`mm`为null。（PS:内核线程使用的全局页表存放在主内存描述符的pgd字段中）

#### 总结

本文简单地描述了进程地址空间中的一些主要数据结构以及之间的联系，关于源码以及更多细节内容在日后整理。



