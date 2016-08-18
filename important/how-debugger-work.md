---
title: 调试器基本工作原理
date: 2016-08-12 16:34:29
tags: 
- 理解计算机
- Linux
categories: 理解计算机
---
在开发的过程中调试是必不可少的步骤，相应的IDE就能帮我们轻松搞定。但是调试背后的原理我觉得有必要了解，本文描述的是一些调试器方面的基本原理，不当之处欢迎指出。

#### ptrace()系统调用

Linux下的调试器实现的主要组成部分就是`ptrace()`,这是一个功能众多且相当复杂的工具，能允许一个进程控制另一个进程的运行，而且可以监视和渗入到进程内部。其中这2个进程是父子进程的关系，调试的进程是父进程，被调试的则是子进程。

OS X下`ptrace()`的原型如下：

```c
int ptrace(int request, pid_t pid, caddr_t addr, int data);
```
其中man手册如此描述参数：

`
The request argument specifies what operation is being performed; the
meaning of the rest of the arguments depends on the operation, but except for one special case noted below,  all ptrace() calls are made by the tracing process, and the pid argument specifies the process ID of the traced process.
`

也就是说`request`参数表示操作的类型，其他3个参数在不同的操作下意义可能不同，`pid`永远指向**被调试**的进程。
`request`可以有以下值：

- PT_TRACE_ME
- PT_DENY_ATTACH
- PT_CONTINUE
- PT_STEP
- PT_KILL
- PT_ATTACH
- PT_ATTACHEXC
- PT_DETACH 

某些值由父进程使用，某些则由子进程使用，比如`PT_TRACE_ME `由子进程调用，从命名上也可以看出来(追踪我)。其中的具体含义可以查看man手册，此处就不多解释了。

#### 单步调试

从高层设计来说，我们要写一个程序，它产生一个子进程用来执行一个用户指定的命令，而父进程跟踪这个子进程。首先，main函数是这样的：

```c
int main(int argc, char** argv)
{
    pid_t child_pid;

    if (argc < 2) {
        fprintf(stderr, "Expected a program name as argument\n");
        return -1;
    }

    child_pid = fork();
    if (child_pid == 0)
        run_target(argv[1]);//子进程执行需要被追踪的指令
    else if (child_pid > 0)
        run_debugger(child_pid);//父进程追踪子进程
    else {
        perror("fork");
        return -1;
    }

    return 0;
}
```

其中`run_target(char arg)`函数的核心内容如下：

```c
ptrace(PT_TRACE_ME, 0, 0, 0);
execl(arg, arg, 0);
```

首先执行ptrace系统调用，request值为PT_TRACE_ME，表明进程期待被父进程追踪，这时候一旦调用了`execl()`,进程被停止运行，然后接收到一个SIGTRAP信号，并等待父进程的操作。

父进程`run_debugger()`函数的核心内容如下：

```c
wait(&wait_status);

    while (WIFSTOPPED(wait_status)) {
        /* Make the child execute another instruction */
        if (ptrace(PT_STEP, child_pid, 0, 0) < 0) {
            perror("ptrace");
            return;
        }
 
        /* Wait for child to stop on its next instruction */
        wait(&wait_status);
    }
```

循环外面的`wait()`调用阻塞父进程，直到有子进程停止运行。此时执行while循环的内容，如果子进程由于发送的信号而停止运行，WIFSTOPPED就返回true。循环内的代码就做了2件事，首先调用`ptrace()`,request值为PT_STEP，使得pid为child_pid的进程单步执行下一条指令。然后再次调用`wait()`,阻塞等待子进程停止。依次循环，直到子进程结束。

#### 断点

在调试的时候我们可以在期望的地方打上断点，程序运行到断点处自动停止运行。这在Linux中是通过中断机制来实现的，将断点处的指令用`int 3`来替代，int指令可以产生一个中断，后面的数字是中断号，每个中断号都对应一个处理例程，3号例程专门用来处理调试时的断点。

要在被调试进程中的某个目标地址上设定一个断点，调试器需要做下面两件事情：

1.  保存目标地址上的数据

2.  将目标地址上的第一个字节替换为int 3指令

然后，当调试器向操作系统请求开始运行进程时，进程最终一定会碰到int 3指令。此时进程停止，操作系统将发送一个信号。这时就是调试器(父进程)再次出马的时候了，接收到一个其子进程（或被跟踪进程）停止的信号，然后调试器要做下面几件事：

1.  在目标地址上用原来的指令替换掉int 3

2.  将被跟踪进程中的指令指针向后递减1。这么做是必须的，因为现在指令指针指向的是已经执行过的int 3之后的下一条指令。

3.  由于进程此时仍然是停止的，用户可以同被调试进程进行某种形式的交互。这里调试器可以让你查看变量的值，检查调用栈等等。

4.  当用户希望进程继续运行时，调试器负责将断点再次加到目标地址上（由于在第一步中断点已经被移除了），除非用户希望取消断点。

#### 调试信息

当你在某些函数的入口处设置断点时，调试器如何知道该在哪里停止目标进程的运行呢？当你希望查看一个变量的值时，调试器又是如何找到它并展示给你呢？答案就是——调试信息。

**调试信息是在编译器生成机器码的时候一起产生的。它代表着可执行程序和源代码之间的关系。这个信息以预定义的格式进行编码，并同机器码`一起存储`。**许多年以来，针对不同的平台和可执行文件，人们发明了许多这样的编码格式。其中一种最重要的格式就是DWARF，作为Linux以及其他类Unix平台上的ELF可执行文件的调试信息格式，如今的DWARF可以说是无处不在。

通过下面的C代码构建一个名为traceprog2的可执行文件来做下实验:

```c
#include <stdio.h>

void do_stuff(int my_arg)
{
    int my_local = my_arg + 2;
    int i;

    for (i = 0; i < my_local; ++i)
        printf("i = %d\n", i);
}

int main()
{
    do_stuff(2);
    return 0;
}
```

通过objdump –h导出ELF可执行文件中的段头信息，我们注意到其中有几个段的名字是以.debug_打头的，这些就是DWARF格式的调试段：

```c
26 .debug_aranges 00000020  00000000  00000000  00001037
                 CONTENTS, READONLY, DEBUGGING
27 .debug_pubnames 00000028  00000000  00000000  00001057
                 CONTENTS, READONLY, DEBUGGING
28 .debug_info   000000cc  00000000  00000000  0000107f
                 CONTENTS, READONLY, DEBUGGING
29 .debug_abbrev 0000008a  00000000  00000000  0000114b
                 CONTENTS, READONLY, DEBUGGING
30 .debug_line   0000006b  00000000  00000000  000011d5
                 CONTENTS, READONLY, DEBUGGING
31 .debug_frame  00000044  00000000  00000000  00001240
                 CONTENTS, READONLY, DEBUGGING
32 .debug_str    000000ae  00000000  00000000  00001284
                 CONTENTS, READONLY, DEBUGGING
33 .debug_loc    00000058  00000000  00000000  00001332
                 CONTENTS, READONLY, DEBUGGING
```
每行的第一个数字表示每个段的大小，而最后一个数字表示距离ELF文件开始处的偏移量。调试器就是利用这个信息来从可执行文件中读取相关的段信息，从而找到可执行程序和源代码之间的关系。


><font color= Darkorange>如若觉得本文尚可，欢迎转载交流,转载请在正文明显处注明原文地址，谢谢！</font>

