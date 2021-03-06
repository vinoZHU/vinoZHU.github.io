---
title: linux内核笔记之进程（三）
date: 2016-08-24 13:42:48
tags: 
- Linux
- 内核
categories: Linux
---
### 进程切换

为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换（process switch）、任务切换（task switch）或上下文切换（content switch）。

#### 硬件上下文

尽管每个进程都有自己的地址空间，但所有进程必须共享CPU寄存器。因此，在恢复一个进程的执行之前，内核必须确保每个寄存器装载了挂起进程时所需要的值。

进程恢复执行前必须装入寄存器的一组数据成为硬件上下文（hardware context）。硬件上下文是进程可执行上下文的一个自己，因为可执行上下文包含进程执行时所需要的所有信息。在Linux中，进程硬件上下午的一部分存放在`TSS段`，而剩余部分存放在`内核态堆栈`中。

在下面描述中，假定用prev局部变量表示切换出的进程描述符，next表示切换进的进程描述符,`这2个局部变量都存放在进程的内核堆栈中`。因此，我们把进程切换定义为这样的行为：保存prev硬件上下文，用next硬件上下文代替prev。因为进程切换经常发生，因此减少保存和装入硬件上下文所花费的时间是非常重要的。

早期Linux版本利用80x86体系结构所需提供的硬件支持，并通过far jmp1指令跳到next进程TSS描述符的选择符来执行进程切换。当执行这条指令时，CPU通过自动保存原来的硬件上下文，装入新的硬件上下文来执行硬件上下文切换。但Linux2.6使用软件执行进程切换，原因有：

通过一组mov指令逐步执行切换，这样能较好地控制所装入的数据的合法性，一面被恶意用户伪造。far jmp指令不会有这样的检查。
旧方法和新方法所需时间大致相同。
进程切换值发生在`内核态`，在执行进程切换之前，用户态进程使用的所有寄存器内容已保存在内核堆栈上，这也包括ss和esp这对寄存器的内容。

#### 任务状态段

80x86体系结构包含了一个特殊的段类型，叫任务状态段（Task State Segment，TSS）来存放硬件上下文，尽管Linux并不使用硬件上下文切换，但是强制它为系统中每个不同的CPU创建一个TSS，这样做主要有两个理由：

当80x86的一个CPU从用户态切换到内核态时，它就从TSS中获取内核态堆栈的地址。
当用户态进程试图通过in或out指令访问一个I/O端口时，CPU需要访问存放在TSS中的I/O许可位图以检查该进程是否有访问端口的权利。
更确切的说，当进程在用户态执行in或out指令时，控制单元执行下列操作：

检查eflags寄存器中的2位IOPL字段，如果字段的值为3，控制单元就执行I/O指令。否则，执行下一个检查。
访问tr寄存器以确定当前的TSS和相应的I/O许可权位图。
检查I/O指令中指定的I/O端口在I/O许可权位图中对应的位，如果该位清，这条指令就执行，否则控制单元产生一个异常。
tss_struct结构描述TSS的格式，`init_tss数组`为系统上每个不同的CPU存放一个TSS。在每次进程切换时，内核都更新TSS的某些字段以便相应的CPU控制单元可以安全地检索到它需要的信息。因此，`TSS反映了CPU上当前进程的特权级，但不必为没有在运行的进程保留TSS`。

每个TSS有它自己8字节的任务状态段描述符（Task State Segment Descriptor，TSSD）。和其他描述符一样，这个描述符包括指向TSS起始地址的32位Base字段，20位Limit字段。TSSD的S标志位被清0，以表示相应的TSS是`系统段`的事实。

Type字段被置位11或9以表示这个段实际上是一个TSS。在Intel的原始设计中，`系统中的每个进程都应当指向自己的TSS`；Type字段的第二个有效位叫Busy位；如果进程正由CPU执行，则该位置1，否则为0。在Linux的设计中，`每个CPU只有一个TSS`，因此Busy位总是为1.

由Linux创建的TSSD存放在全局描述符表（GDT）中，GDT的基地址存放在每个CPU的gdtr寄存器中。
![](/images/linux-kernel-note/process-three-0.png)
每个CPU的tr寄存器包含相应TSS的TSSD选择符，也包含了两个`隐藏的非编程字段`：TSSD的Base字段和Limit字段。这样，处理器就能够直接TSS寻址而不需要从GDT中检索TSS地址。

#### thread字段

在每次进程切换时，被替换的进程的硬件上下文必须保存在别处。不能像Intel原始设计那样保存在TSS中，因为`Linux为每个处理器而不是为每个进程使用TSS`。

因此，每个进程描述符包含一个类型为thread_struct的thread字段，只要进程被切换出去，内核就把其硬件上下文保存在这个结构中。随后可以看到，这个数据结构包含的字段涉及大部分CPU寄存器，但不包括eax、ebx等等这些通用寄存器。它们的值保留在内核堆栈中。

#### 执行进程切换
进程切换可能只发生在精心定义的点：schedule()函数，这个函数在专门整理进程调度的篇幅里讲解。这里，只关注内核如何执行一个进程切换。

进程切换由两步组成：

- 切换页全局目录以安装一个新的地址空间。
- 切换内核态堆栈和硬件上下文，因为硬件上下文提供了内核执行新进程所需要的所有信息，包含CPU寄存器。

第二步由`switch_to`宏来完成，源码如下:

http://lxr.free-electrons.com/source/include/asm-generic/switch_to.h#L25

```c
 /*
  * Context switching is now performed out-of-line in switch_to.S
  */
 extern struct task_struct *__switch_to(struct task_struct *,
                                        struct task_struct *);
 
#define switch_to(prev, next, last)                                      											                         \
        do {                                                            \
                ((last) = __switch_to((prev), (next)));                 			                                                   \
        } while (0)

#endif /* __ASM_GENERIC_SWITCH_TO_H */
```
首先，该宏有三个参数，prev、next和last，prev和next的作用仅是`内核堆栈中`的局部变量prev和next的占位符，即它们是输入参数，分别表示被替换进程和新进程描述符的地址在内存中的位置。

在任何进程切换中，涉及到的是三个进程而不是两个。假设内核决定暂停进程A而激活进程B，在schedule()函数中，prev指向A的描述符，而next指向B的进程描述符。switch_to宏一旦使A暂停，A的执行流就被冻结。

**此时，A的内核堆栈中的`prev`指向A的描述符，`next`指向B的描述符。B的内核堆栈中的`prev`指向B的描述符，`next`指向other(下一个要调度的进程描述符，目前未知)。**

随后，当内核想再次激活A，就必须暂停另一个进程C，因为这通常不是B，因为B有可能被其他进程比如C切换。于是就要用prev指向C而next指向A来执行另一个switch_to宏。当A恢复它执行的流时，就会找到它原来的内核栈，于是prev局部变量还是指向A的描述符而next指向B的描述符。

**此时，A的内核堆栈中的`prev`指向A的描述符，`next`指向B的描述符。C的内核堆栈中的`prev`指向C的描述符，`next`指向A的描述符。**

所以，`代表进程A执行的内核就失去了对C的任何引用`。但引用对于完成进程切换是有用的，所以需要保留。

switch_to宏的最后一个参数是输出参数，它表示宏把进程C的描述符地址写在内存的什么位置了，不过，这个是在恢复A执行之后完成的。在进程切换之前，宏把第一个输入参数prev表示的变量存入CPU的eax寄存器。在完成进程切换，A已经恢复执行时，宏把CPU的eax寄存器的内容写入由第三个参数last所指示的A在内存中的位置。因为CPU寄存器不会在切换点发生变化，所以C的描述符地址也存在内存的这个位置。在schedule()执行过程中，last参数指向A的局部变量prev，所以prev被C的地址覆盖。

#### __switch_to()函数
__switch_to()函数执行大多数开始于switch_to()宏的进程切换。这个函数作用于prev_p和next_p参数，这两个参数表示前一个进程和新进程。这个函数的调用不同于一般的函数调用。因为__switch_to()从eax和edx取参数prev_p和next_p，而不像大多数函数一样从栈中取参数。

http://lxr.free-electrons.com/source/arch/x86/kernel/process_32.c#L242

```c
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
        struct thread_struct *prev = &prev_p->thread,
                             *next = &next_p->thread;
        struct fpu *prev_fpu = &prev->fpu;
        struct fpu *next_fpu = &next->fpu;
        int cpu = smp_processor_id();
        struct tss_struct *tss = &per_cpu(cpu_tss, cpu);
        fpu_switch_t fpu_switch;

        /* never put a printk in __switch_to... printk() calls wake_up*() indirectly */

        fpu_switch = switch_fpu_prepare(prev_fpu, next_fpu, cpu);

        /*
         * Save away %gs. No need to save %fs, as it was saved on the
         * stack on entry.  No need to save %es and %ds, as those are
         * always kernel segments while inside the kernel.  Doing this
         * before setting the new TLS descriptors avoids the situation
         * where we temporarily have non-reloadable segments in %fs
         * and %gs.  This could be an issue if the NMI handler ever
         * used %fs or %gs (it does not today), or if the kernel is
         * running inside of a hypervisor layer.
         */
        lazy_save_gs(prev->gs);

        /*
         * Load the per-thread Thread-Local Storage descriptor.
         */
        load_TLS(next, cpu);

        /*
         * Restore IOPL if needed.  In normal use, the flags restore
         * in the switch assembly will handle this.  But if the kernel
         * is running virtualized at a non-zero CPL, the popf will
         * not restore flags, so it must be done in a separate step.
         */
        if (get_kernel_rpl() && unlikely(prev->iopl != next->iopl))
                set_iopl_mask(next->iopl);

        /*
         * Now maybe handle debug registers and/or IO bitmaps
         */
        if (unlikely(task_thread_info(prev_p)->flags & _TIF_WORK_CTXSW_PREV ||
                     task_thread_info(next_p)->flags & _TIF_WORK_CTXSW_NEXT))
                __switch_to_xtra(prev_p, next_p, tss);

        /*
         * Leave lazy mode, flushing any hypercalls made here.
         * This must be done before restoring TLS segments so
         * the GDT and LDT are properly updated, and must be
         * done before fpu__restore(), so the TS bit is up
         * to date.
         */
        arch_end_context_switch(next_p);

        /*
         * Reload esp0 and cpu_current_top_of_stack.  This changes
         * current_thread_info().
         */
        load_sp0(tss, next);
        this_cpu_write(cpu_current_top_of_stack,
                       (unsigned long)task_stack_page(next_p) +
                       THREAD_SIZE);

        /*
         * Restore %gs if needed (which is common)
         */
        if (prev->gs | next->gs)
                lazy_load_gs(next->gs);

        switch_fpu_finish(next_fpu, fpu_switch);

        this_cpu_write(current_task, next_p);

        return prev_p;
}
```

这个函数执行步骤如下：

- 执行由__unlay_fpu()宏代码产生的代码，以有选择地保存prev_p进程的FPU、MMX以及XMM寄存器的内容。

- 执行smp_processor_id()宏获得本地CPU的下标，即执行代码当前的CPU。该宏从当前进程的thread_info结构的cpu字段获得下标并保存到cpu局部变量。

- 把next_p->thread.esp0装入对应于本地CPU的TSS的esp0字段。其实，任何由sysenter汇编指令产生的从用户态到内核态的特权级转换将把这个地址拷贝到esp寄存器中。

- 把next_p进程使用的线程局部存储（TLS）段装载入本地CPU的全局描述符表。

- 把fs和gs段寄存器的内容分别存放在prev_p->thread.fs和prev_p->thread.gs中。esi寄存器指向prev_p->thread结构。

- 如果fs或gs段寄存器已经被prev_p或next_p进程中的任意一个使用，则将next_p进程的thread_struct描述符中保存的值装入这些寄存器。

- 用next_p->thread.debugreg数组内容装载dr0…dr7中的6个调试寄存器。只有在next_p被挂起时正在使用调试寄存器，这种操作才能进行。

- 如果必要，则更新TSS中的I/O位图。因为进程很少修改I/O权限位图，所以当且仅当一个进程在当前时间片内实际访问I/O端口时，真实的位图才被拷贝到本地CPU的TSS中。进程的定制I/O权限位图被保存在thread_info结构的io_bitmap_ptr字段指向的缓冲区中。当prev_p或者next_p拥有自己定制的I/O权限位图（io_bitmap_ptr不为null）时,TSS的io_bitmap字段被设为0x9000,否则为0x8000。

- 终止，prev_p参数被拷贝到eax，`因为缺省情况下任何C函数的返回值被传给eax寄存器`。所以eax的值在调用__switch_to()的过程中被保护起来；这很重要，因为调用该函数时会假定eax总是用来存放将被替换的进程描述符地址。

- 汇编语言指令ret把栈顶保存的返回地址装入eip程序计数器。不过，__swtich_to()函数时通过简单的跳转被调用的。因此，ret汇编指令在栈中找到标号为1的指令地址，其中标号为1的地址是由switch_to()宏推入堆栈的。


><font color= Darkorange>因本人水平有限，若文章内容存在问题，恳请指出。允许转载，转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 