---
title: linux内核笔记之进程调度
date: 2016-08-26 16:23:54
tags: 
- Linux
- 内核
categories: Linux
---
#### 概览
传统Unix操作系统的调度算法必须实现几个互相冲突的目标：进程响应时间尽可能快，后台作业的吞吐量尽可能高，进程的饥饿现象尽可能避免，低优先级和高优先级进程的需要尽可能调和等等。决定什么时候以怎样的方式选择一个新进程运行的这组规则就是所谓的调度策略（scheduling policy）。

Linux的进程调度是基于分时技术（time-sharing）。允许多个进程“并发”运行就意味着CPU 的时间被粗略地分成“片”，给每个可运行进程分配一片。
    
当然，单处理器在任何给定的时刻只能运行一个进程。当一个并发执行的进程其时间片或时限（quantum）到期时还没有终止，进程切换就可以发生。分时依赖于定时中断，因此，对进程是透明的。为保证CPU 分时，不需要在程序中插入额外的代码。
    
在Linux 中，进程的优先级是动态的。调度程序跟踪进程做了些什么，并周期性地调整它们的优先级。在这种方式下，在较长的时间间隔内没有使用CPU的进程，通过动态地增加它们的优先级来提升它们。相应地，对于已经在CPU上运行了较长时间的进程，通过减少它们的优先级来处罚它们。每个进程在创建之初有一个基本的优先级，执行期间调度系统会动态调整它的优先级，交互性高的任务会获得一个高的动态优先级，而交互性低的任务获得一个低的动态优先级。

#### 进程分类
传统上把进程分类为“I/O 受限（I/O-bound）”或“CPU受限（CPU-bound）”。前者频繁地使用I/O 设备，并花费很多时间等待I/O操作的完成；而后者是需要大量CPU 时间的数值计算应用程序。

另一种分类法把进程区分为三类:**交互式进程** 、**批处理进程** 、**实时进程**。

交互式进程经常与用户交互，需要花很多时间等待键盘和鼠标操作。典型的交互式程序是命令shel、文本编辑程序、图形应用程序等。批处理程序不必与用户交互，经常在后台运行。因为这些进程不必被很快地相应，因此常受到调度程序的慢待。典型的批处理程序是编译程序、数据库搜索引擎、科学计算等。实时进程有很强的调度需要，他们不会被低优先级的进程阻塞，响应的时间很短。典型的实时程序有视频和音频应用程序、机器人控制程序、从物理传感器上收集数据的程序等。

这2种分类法在一定程度上互相独立。例如一个批处理程序可能是I/O受限型的（如数据库服务器），也可能是CPU受限型的（图像绘制程序）。Linux的调度算法可以明确的区分实时程序，但是没有办法区分交互式程序和批处理程序。Linux根据进程的过去行为，通过特定的算法区分交互式程序和批处理程序。因为交互式程序需要给用户一个良好的体验，所以Linux调度程序对交互式程序比较偏爱。

#### 进程优先级
交互式程序和批处理程序都叫做**非实时进程（普通进程）**，每个非实时进程都有自己的静态优先级（nice值），`值越大优先级越低`。。nice值是所有Unix系统的标准化概念，在OS X系统中nice值代表分配给进程的时间片的绝对值，`而Linux中代表时间片的比例`。通过`ps -el`命令查看系统中的进程列表，结果中标记`NI`的一列就是进程对应的nice值。

对于**实时进程**，实时优先级的范围是从1（最低优先级）~ 99（最高优先级），含义与nice值相反。任何实时进程的优先级总高于非实时进程（普通进程）。你可以通过`ps -eo stata,uid,pid,ppid,rtprio,time,com`查看系统中的进程列表，在`RTPRIO`列的就是实时优先级，如果显示`-`，则该进程不是实时进程。

#### 时间片
时间片是一个数值，它表明进程在被抢占前所能持续运行的时间。一般来说，调度策略必须规定一个默认的时间片，但是Linux的CFS调度器并没有直接分配时间片到进程，而是将处理器的使用比划分给了进程。这样一来，进程所获得的处理器时间其实是和系统负载密切相关的。nice值作为权重将调整使用比，值越大，使用比越小。Linux中CFS调度器的抢占时机取决于新进程所消耗的使用比。通过这种方式，CFS确保了进程调度中能有恒定的公平性，而将切换频率置于不断变动中。

不过，当可运行进程的数量区域无限时，每个进程获得使用比则趋于0，岂不是时间都花在切换进程上了？CFS为此引入了每个进程获得的时间片最小粒度，默认是1ms,也就是每个进程最少能获得1ms的运行时间。

#### 调度类型
每个Linux进程都按照以下调度类型被调度：

- SCHED_FIFO
  先进先出的实时进程。如果没有优先级更高的可运行的实时进程，则当前运行的实时进程想运行多久便运行多久，即使还有其他优先级相同的可运行实时进程

- SCHED_RR
  时间片轮转的实时进程。保证对所有相同优先级的实时进程公平地分配CPU时间。

- SCHED_NORMAL
  普通的分时进程。
 
#### 调度算法

早期Linux的调度算法非常简单，在每次进程切换时，内核扫描可运行进程的链表，然后选择“最佳”的进程来运行。这个算法的主要缺点是选择进程所消耗的时间与可运行进程的数量相关。当可运行进程的数量很多时，扫描花费的时间很不可观。

如今Linux采用的算法为“完全公平调度算法”，简称`CFS`。CFS是一个针对普通进程的调度器，期调度类型为`SCHED_NORMAL`。在#时间片#中已经讲到，nice值在CFS中被作为进程获得处理器运行比的权重。

CFS使用一个`struct sched_entity`数据结构来记录进程的调度情况，包括运行时间。进程描述符中的`se`字段便是该数据类型,`se`是当前进程的调度实体，该结构部分内容如下:

http://lxr.free-electrons.com/source/include/linux/sched.h?v=2.6.24#L866

```c
struct sched_entity {
        struct load_weight      load;           /* for load-balancing */
        struct rb_node          run_node;
        unsigned int            on_rq;

        u64                     exec_start;
        u64                     sum_exec_runtime;
        u64                     vruntime;
        u64                     prev_sum_exec_runtime;


};
```
`vruntime`变量存放进程的虚拟运行时间，CFS使用`vruntime`来记录一个程序到底运行了多长时间以及它应该再运行多久。可以简单地认为`vruntime`通过以下公式计算:

```c
vruntime ＝ delta_time / load
```
其中`delta_time`表示该进程以运行的时间，`load`表示进程对CPU的使用比，和nice值相关。当运行相同时间时，使用比大的进程vruntime更小，表明能获得更多的CPU时间。更新函数通过时钟中断调用。

**进程选择**

CFS调度算法的核心便是选择具有最小vruntime的进程，CFS使用红黑树来组织可运行进程队列，并且通过红黑树迅速找到最小vruntime的进程。红黑树中的每一个节点的值便是`vruntime`,越小的`vruntime`，位置越靠左下，所以只需找到最左下的节点的进程。

#### 调度相关数据结构

##### struct rq

```c
struct rq {
279         /* runqueue lock: */
280         spinlock_t lock;
281 
282         /*
283          * nr_running and cpu_load should be in the same cacheline because
284          * remote CPUs use both these fields when doing load calculation.
285          */
286         unsigned long nr_running;
287         #define CPU_LOAD_IDX_MAX 5
288         unsigned long cpu_load[CPU_LOAD_IDX_MAX];
289         unsigned char idle_at_tick;
290 #ifdef CONFIG_NO_HZ
291         unsigned char in_nohz_recently;
292 #endif
293         /* capture load from *all* tasks on this cpu: */
294         struct load_weight load;
295         unsigned long nr_load_updates;
296         u64 nr_switches;
297 
298         struct cfs_rq cfs;
299 #ifdef CONFIG_FAIR_GROUP_SCHED
300         /* list of leaf cfs_rq on this cpu: */
301         struct list_head leaf_cfs_rq_list;
302 #endif
303         struct rt_rq rt;
304 
305         /*
306          * This is part of a global counter where only the total sum
307          * over all CPUs matters. A task can increase this counter on
308          * one CPU and if it got migrated afterwards it may decrease
309          * it on another CPU. Always updated under the runqueue lock:
310          */
311         unsigned long nr_uninterruptible;
312 
313         struct task_struct *curr, *idle;
314         unsigned long next_balance;
315         struct mm_struct *prev_mm;
316 
317         u64 clock, prev_clock_raw;
318         s64 clock_max_delta;
319 
320         unsigned int clock_warps, clock_overflows;
321         u64 idle_clock;
322         unsigned int clock_deep_idle_events;
323         u64 tick_timestamp;
324 
325         atomic_t nr_iowait;
326 
327 #ifdef CONFIG_SMP
328         struct sched_domain *sd;
329 
330         /* For active balancing */
331         int active_balance;
332         int push_cpu;
333         /* cpu of this runqueue: */
334         int cpu;
335 
336         struct task_struct *migration_thread;
337         struct list_head migration_queue;
338 #endif
339 
340 #ifdef CONFIG_SCHEDSTATS
341         /* latency stats */
342         struct sched_info rq_sched_info;
343 
344         /* sys_sched_yield() stats */
345         unsigned int yld_exp_empty;
346         unsigned int yld_act_empty;
347         unsigned int yld_both_empty;
348         unsigned int yld_count;
349 
350         /* schedule() stats */
351         unsigned int sched_switch;
352         unsigned int sched_count;
353         unsigned int sched_goidle;
354 
355         /* try_to_wake_up() stats */
356         unsigned int ttwu_count;
357         unsigned int ttwu_local;
358 
359         /* BKL stats */
360         unsigned int bkl_count;
361 #endif
362         struct lock_class_key rq_lock_key;
363 };
```

##### struct cfs_rq

```c
struct cfs_rq {
232         struct load_weight load;
233         unsigned long nr_running;
234 
235         u64 exec_clock;
236         u64 min_vruntime;
237 
238         struct rb_root tasks_timeline;
239         struct rb_node *rb_leftmost;
240         struct rb_node *rb_load_balance_curr;
241         /* 'curr' points to currently running entity on this cfs_rq.
242          * It is set to NULL otherwise (i.e when none are currently running).
243          */
244         struct sched_entity *curr;
245 
246         unsigned long nr_spread_over;
247 
248 #ifdef CONFIG_FAIR_GROUP_SCHED
249         struct rq *rq;  /* cpu runqueue to which this cfs_rq is attached */
250 
251         /*
252          * leaf cfs_rqs are those that hold tasks (lowest schedulable entity in
253          * a hierarchy). Non-leaf lrqs hold other higher schedulable entities
254          * (like users, containers etc.)
255          *
256          * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a cpu. This
257          * list is used during load balance.
258          */
259         struct list_head leaf_cfs_rq_list;
260         struct task_group *tg;  /* group that "owns" this runqueue */
261 #endif
262 };
```

##### struct rt_rq

```c
struct rt_rq {
266         struct rt_prio_array active;
267         int rt_load_balance_idx;
268         struct list_head *rt_load_balance_head, *rt_load_balance_curr;
269 };
```

`struct rq`为一个运行队列，每个CPU都有一个这样的数据结构，在`rq`中有`struct cfs_rq`类型的cfs字段，该字段表示一个CFS调度器的运行队列，`rq`中的`struct rt_rq`类型的rt字段表示一个实时进程的运行队列。具体细节日后有时间在整理。