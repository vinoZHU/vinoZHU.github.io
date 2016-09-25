---
title: linux内核笔记之进程（二）
date: 2016-08-16 20:31:04
tags: 
- Linux
- 内核
categories: Linux
---
#### 进程描述符
进程在运行时的所有信息，如进程优先级、地址空间、访问的文件等等，都是通过进程描述符来记录的。内核正是通过进程描述符来对进程进行管理。进程描述符的数据结构是`struct task_struct`,与进程相关的所有信息都包含在该数据结构中，可想而知该数据结构是十分复杂的。

在[**这里**](http://lxr.free-electrons.com/source/include/linux/sched.h#L1458)可以查看`task_struct`的完整结构，下图示意性地描述了该数据结构：
![](/images/linux-kernel-note/process-two-0.png)
在这里只讨论进程的某几个属性，其余属性在之后有关其他内容的文章中会逐渐涉及到，内核的学习需要联系的思维。

#### 进程状态

在[**这里**](http://lxr.free-electrons.com/source/include/linux/sched.h#L207)可以查看进程的所有状态，下面是一些常见的状态：

|状态|说明|
|:---:|:---|
| TASK_RUNNING |进程正在执行或者进程已经就绪，正在等待cpu时间片的调度。|
| TASK_INTERRUPTIBLE |进程因为等待一些条件而被挂起（阻塞）时的状态。此时的进程可以被硬中断、资源的释放、信号等事件（这些事件可以认为是等待的条件）唤醒，然后进程就会从该状态（阻塞）迅速转化成为就绪状态TASK_RUNNING。|
| TASK_UNINTERRUPTIBLE |意义与TASK_INTERRUPTIBLE类似，但是该状态的进程只有当等待的条件为true时才能被唤醒。一般情况下很少用到，但是在一些驱动程序中很有用。特别是对于驱动探测相关的硬件过程很重要，这个探测过程不能被一些其他的东西给中断，否则就会让进程进入不可预测的状态。|
| __TASK_STOPPED |进程停止执行，当进程接收到SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号之后就会进入该状态。|
| __TASK_TRACED |表示进程被debugger等进程所暂停（[ptrac()调用与调试器相关](/2016/08/12/how-debugger-work/)），此时任何一个信号都会让进程进入该状态|
|EXIT_DEAD |进程的执行被终止，但是其父进程还没有使用wait()等系统调用来获知它的终止信息，此时进程成为僵尸进程|
|EXIT_ZOMBIE |进程的最终状态，由于父进程进行了wait4()或者waitpid()系统调用，进程被系统删除|

#### 进程标识
类Unix操作系统通过进程标识符（PID）来标识一个进程(包括轻量级进程) ，每个进程的PID存放在进程描述符的pid字段中。PID是按顺序编号的，新创建的进程通常是前一个进程的PID+1。不过，32位系统的PID值在缺省情况中的上限为32767，64位是4194303，系统管理员可以通过修改`/proc/sys/kernel/pid_max`来改变这些值。

内核通过管理一个`pidmap_array`位图来记录当前已使用的PID，因为一个页框的大小为4KB,恰好为32767bit,所以32位体系结构中一个`pidmap_array `刚好占据一个单独的页。系统会一直保存这些页不被释放。

#### 标识当前进程
在内核代码中，经常通过current宏来获得当前在CPU上运行的进程描述符指针，例如，current->pid返回当前正在执行的进程的PID。我们来看下current宏究竟干了什么：

(Linux/arch/tile/include/asm/current.h)

```c
#define current get_current()
```

(Linux/include/asm-generic/current.h)

```c
#define get_current() (current_thread_info()->task)
```
```c
static __inline__ struct task_struct *get_current(void)
{
      return current_thread_info()->task;
}
```
`current_thread_info()`返回一个`struct thread_info`结构的指针
```c
struct thread_info {
          struct task_struct      *task;          /* main task structure */
          unsigned long           flags;          /* low level flags */
          unsigned long           status;         /* thread-synchronous flags */
          __u32                   cpu;            /* current CPU */
          int                     preempt_count;  /* 0 => preemptable, <0 => BUG */
  
          mm_segment_t            addr_limit;     /* thread address space:
                                                     0-0xBFFFFFFF for user-thread
                                                     0-0xFFFFFFFF for kernel-thread
                                                  */
  
          __u8                    supervisor_stack[0];
  };
```
返回的`thread_info`指针就是当前进程描述符中的thread_info字段的指针，而该结构中保存着所属进程的进程描述符指针。`为什么不直接把进程描述符指针记录在一个全局变量中？`早先的Linux版本的确是这么做的,通过强制引入一个全局静态变量current来标识正在运行进程的描述符。

既然现在是通过`thread_info`来标识当前进程的，那么如何获得该`thread_info`的地址呢？

Linux都把两个不同的数据结构紧凑地存放在一个单独为进程分配的存储区域内，一个是与进程描述符相关的小数据结构thread_info，叫做线程描述符。另一个是内核态的进程堆栈。

内核态的进程堆栈大小通常为8192个字节。考虑到效率的因素，内核让这8K空间占据连续的两个页框并让第一个页框的起始地址是2^13的倍数。当几乎没有可用的动态内存空间时，就会很难找到这两个连续页框，因为空闲空间可能存在大量碎片。因此，在80x86体系结构中，在编译时可以设置，以使内核堆栈和线程描述符跨越一个单独的页框。
![](/images/linux-kernel-note/process-two-1.png)

`esp`寄存器保存着内核态堆栈的栈顶地址，因为内核态堆栈和`thread_info`处于连续的2个页框，而`thread_info`处于页框的首地址，只需进行下列的汇编指令就能获得`thread_info`地址：

```
movl $0xffffe000, %ecx
andl $esp, %ecx
movl %ecx, p
```
其中变量p中的值便是`thread_info`的地址，而这3条汇编指令其实也就是`current_thread_info()`函数所做的事。

当进程切换到内核态之后，esp指针便保存内核栈的栈顶地址。内核直接通过寄存器的值，进行少许计算便能获得当前进程描述符，相比较于早先的全局静态变量，效率是明显提升了。

#### 如何组织进程
对于`TASK_RUNNING`的进程，内核早先把所有的这些进程放到一个进程双向链表中，但是随着进程数量的增加，维护成本会越来越高。调度程序在扫描链表时花费的时间也随之线性上升。Linix2.6中，优先级相同的进程才会放在一个对应优先级的链表中。如果优先级有n个级别，那么就有n个运行队列。在多处理器系统中，每个CPU都有着n个这样的运行队列。

关于运行队列，在之后整理进程调度的文章中会再次涉及到。

对于`__TASK_STOPPED`、`EXIT_DEAD `、`EXIT_ZOMBIE `状态的进程，内核没有建立相应的双向链表。对于这些进程的访问比较简单，或者通过PID，或者通过特定父进程的子进程链表。

对于`TASK_INTERRUPTIBLE `、`TASK_UNINTERRUPTIBLE `状态的进程，由于这些进程所等待的条件并不是同一个。而且进程所提供的信息无法满足快速检索的需要，因此有了等待队列。

在这里讨论的等待队列都表示进程在特定事件上的等待。一个等待队列表示一组睡眠的进程，这些进程所等待的条件都是同一个。当条件满足时，内核会唤醒这些进程。然而，当两个或多个进程互斥的访问某个资源时，此时内核必须只能唤醒一个进程，其余的进程需要继续睡眠。

等待队列由双向链表组成，每个等待队列都有一个队列头:

```c
struct __wait_queue_head {
         spinlock_t              lock;//自旋锁
         struct list_head        task_list;
 };
```

等待队列中的元素:

```c
struct __wait_queue {
         unsigned int            flags;
         void                    *private;//指向进程描述符
         wait_queue_func_t       func;//唤醒时需要调用的方法
         struct list_head        task_list;
 };
```
`struct list_head`在内核中是实现双向链表的关键，在这里不做讨论。

#### 根据PID找到进程描述符
Linux中有4个pid类型:

- PID：这是 Linux 中在其命名空间中唯一标识进程而分配给它的一个号码，称做进程ID号，简称PID。在使用 fork 或 clone 系统调用时产生的进程均会由内核分配一个新的唯一的PID值。
- TGID：在一个进程中，如果以CLONE_THREAD标志来调用clone建立的进程就是该进程的一个线程，它们处于一个线程组，该线程组的ID叫做TGID。处于相同的线程组中的所有进程都有相同的TGID；线程组组长的TGID与其PID相同；一个进程没有使用线程，则其TGID与PID也相同。
- PGID：另外，独立的进程可以组成进程组（使用setpgrp系统调用），进程组可以简化向所有组内进程发送信号的操作，例如用管道连接的进程处在同一进程组内。进程组ID叫做PGID，进程组内的所有进程都有相同的PGID，等于该组组长的PID。
- SID：几个进程组可以合并成一个会话组（使用setsid系统调用），可以用于终端程序设计。会话组中所有进程都有相同的SID。

既然每个进程都有一个与之关联的PID，在进程描述符中的pid字段保存着PID值，如果通过遍历进程链表来比较PID值，虽然可行，但是相当低效。那么内核肯定维护着PID与进程描述符之间的某种映射关系。内核可以通过这种关系，快速地根据PID找到对应的进程描述符。

内核是通过hash表来解决这个问题的。在内核初始化的时候会动态地为一个散列表数组分配空间,数组长度和机器内存有关。

```c
static struct hlist_head *pid_hash;
```
初始化函数为:

```c
void __init pidhash_init(void);
```


散列表数组的元素类型为 `struct hlist_head`，看下具体是什么:

```c
struct hlist_head {
        struct hlist_node *first;
};
```

其中包含一个`struct hlist_node`类型的指针:

```c
struct hlist_node {
        struct hlist_node *next, **pprev;
};
```
`struct hlist_node`中包含一个`struct hlist_node`类型的指针和`struct hlist_node*`类型的指针。

现在散列表和进程之间还没有建立联系，来看下进程中的`pids`(不是pid)字段的数据结构:
```c
struct pid_link
 {
         struct hlist_node node;
         struct pid *pid;
 };
```

再看下`struct pid`是什么东西:

```c
struct pid
{
        atomic_t count;
        unsigned int level;
        /* lists of tasks that use this pid */
        struct hlist_head tasks[PIDTYPE_MAX];
        struct rcu_head rcu;
        struct upid numbers[1];
};
```
其中的`tasks`字段，这是一个`struct hlist_head`类型的数组，长度为pid类型数量，也就是4。

再看下其中的`struct upid`类型：

```c
struct upid {
        /* Try to keep pid_chain in the same cacheline as nr for find_vpid */
        int nr;//用于计算hsah值
        struct pid_namespace *ns;
        struct hlist_node pid_chain;
};
```
下图用来帮助理解几者之间的联系:
![](/images/linux-kernel-note/process-two-2.png)

1. 内核初始化期间为pid_hash数组分配好合适的空间。
2. 当创建了一个新的进程之后，内核随之分配一个进程描述符，然后初始化其中的字段。
3. 根据 task_struct->pids（struct pid_link类型）->pid->numbers(struct upid类型)->nr的值，调用hash函数，取模，然后定位到pid_hash数组中的某一项。
4. 遍历该项的链表，如果其中没有一项的pid与新创建的相同（线程组的所有进程pid相同），则将该进程插入到链表尾。具体做法是将pids字段(struct pid_link类型)中的node插入到链表末尾。
5. 如果存在一项pid与新创建的进程相同，说明是同一线程组的进程。则通过task_struct->pids（struct pid_link类型）->pid->numbers(struct upid类型)->pid_chain（struct hlist_node类型）,将其链接到已存在项的tasks数组中对应pid类型的链表中去，这样的链表一共有4条。

#### 总结
本文主要讨论了进程描述符中的某几个属性，包括pid,进程状态。还简单介绍了current宏、进程链表以及如何通过pid快速找到对应进程描述符的地址。

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 




