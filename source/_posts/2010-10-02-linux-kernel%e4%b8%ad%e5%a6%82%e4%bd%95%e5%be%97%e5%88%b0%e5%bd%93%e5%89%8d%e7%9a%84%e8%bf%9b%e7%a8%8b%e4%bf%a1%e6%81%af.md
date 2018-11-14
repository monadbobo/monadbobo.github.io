---
id: 36
title: linux kernel中如何得到当前的进程信息
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=36'
permalink: >-
  /2010/10/02/linux-kernel%e4%b8%ad%e5%a6%82%e4%bd%95%e5%be%97%e5%88%b0%e5%bd%93%e5%89%8d%e7%9a%84%e8%bf%9b%e7%a8%8b%e4%bf%a1%e6%81%af/
views:
  - '44'
bot_views:
  - '11'
categories:
  - kernel
tags:
  - kernel
  - linux
  - process
  - qemu
translate_title: how-to-get-the-current-process-information-in-linux-kernel
date: 2010-10-02 11:30:04
---
我这里内核版本是2.6.35,cpu架构是x86_32.先来看linux下进程的结构。

首先我们要知道在linux中第一个进程是内核进程，pid为0，它是所有的进程的父进程。这个进程也叫swapper，或者说是idle.
  
<!--more-->


  
这个进程是静态初始化的，定义是在init_task.c中，如下：
  
[code lang=&#8221;c&#8221;]
  
union thread_union init_thread_union __init_task_data =
  
{ INIT_THREAD_INFO(init_task) };

#define __init_task_data __attribute__((__section__(".data..init_task")))
  
[/code]
  
这里要注意的就是__init_task_data这个宏，这个宏就是将init_thread_union放到.data..init_task这个段里面。还有就是这个是一个thread_union的联合体，我们后面会介绍这个。

这个进程主要用途就是保证至少会有一个进程还在运行，也就是说当没有进程运行的情况下，它是最后的那个进程,就是swaaper 进程.init进程就是由它派生出来的。

然后来看对应的初始化，在header32.s中，这里是将init_thread_union的地址加上THREAD_SIZE(栈的大小)装载进esp，然后__BOOT_DS装载进ss。32为x86默认栈的大小为8k，因此一开始esp的值就是init_thread_union的地址＋栈的大小。
  
```
  
/\* Set up the stack pointer \*/
  
lss stack_start,%esp
  
ENTRY(stack_start)
  
.long init_thread_union+THREAD_SIZE
  
.long __BOOT_DS
  
```

我们知道在linux内核中，进程被分为两部分，一部分是thread_info,保存在内核栈中，为了保持很小，因此只保存了必须的几个域，它有一个变量task，指向task_struct,这个结构保存了进程相关的所有信息。而对应的task_struct也保存了一个变量stack指向的就是一个thread_union的联合体,先来看这个联合体的结构：
  
```
  
union thread_union {
  
struct thread_info thread_info;
  
unsigned long stack[THREAD_SIZE/sizeof(long)];
  
};
  
```

可以看到它其中一部分是thread_info，一部分是stack，其实这个很好理解，那就是表示它会分配THREAD_SIZE大小的内存然后前面一部分，也就是低地址部分保存thread_info,而高地址部分(剩余部分)存放内核栈.这里我们要知道，所有的进程是共享一个内核栈的(x86_64位的smpj架构下会有多个内核栈,此时一个cpu上的进程共享本cpu上的内核栈，不过具体为什么x86_64下面需要多个kernel stack还不清楚.)，而内核栈的thread_info部分也就是保存了当前正在运行的进程(smp下面也就是当前cpu正在运行的进程)，内核使用current能够直接存取当前正在运行的任务还有current_thread_info来直接取得当前的thread_info,接下来我们就来看这2个函数的实现。

首先是smp下面kernel stack和current_task的定义
  
```
  
DEFINE_PER_CPU(struct task_struct *, current_task) = &init_task;
  
EXPORT_PER_CPU_SYMBOL(current_task);

DEFINE_PER_CPU(unsigned long, kernel_stack) =
  
(unsigned long)&init_thread_union &#8211; KERNEL_STACK_OFFSET + THREAD_SIZE;
  
EXPORT_PER_CPU_SYMBOL(kernel_stack);

```
  
可以看到kernel stack和current_task都是为每个cpu定义了一个，这样的话，读取这两个我们之需要简单的使用percpu_read_stable就可以了，来看current的实现。

```
  
struct task_struct;
  
static __always_inline struct task_struct *get_current(void)
  
{
  
//读取current_task这个变量
  
return percpu_read_stable(current_task);
  
}

#define current get_current()
  
```

然后是current_thread_inf用来直接取得当前的cpu上的thread_info的地址，这里它的计算是这样的，把esp的最后n(THREAD_SIZE栈大小的位数)位清空，那么就是基地址了，也就是thread_info的值.
  
```
  
register unsigned long current_stack_pointer asm("esp") __used;

/\* how to get the thread information struct from C \*/
  
static inline struct thread_info *current_thread_info(void)
  
{
  
//得到thread_info
  
return (struct thread_info *)
  
(current_stack_pointer & ~(THREAD_SIZE &#8211; 1));
  
}
  
```

通过上面的分析，现在我们得到当前进程信息就很容易了，通过$esp来清除后13位的值，得到thread_info,然后通过thread_info自然很容易得到current_task了。
  
下面就是通过gdb来验证我们的方法。

> (gdb) b start_kernel
  
> (gdb) c
  
> //打印当前的task的stack，也就是thread_info地址
  
> (gdb) p current_task->stack
  
> $35 = (void *) 0xc1698000
  
> //打印esp的值
  
> (gdb) p $esp
  
> $36 = (void *) 0xc1699fe4
  
> (gdb) p current_task
  
> $38 = (struct task_struct *) 0xc169f460
  
> //将$esp的值后13位清空，然后就是当前的进程的thread_info的值，然后打印出它的comm，进程名，可以看到就是0号进程。
  
> (gdb) p ((struct thread_info *)0xc1698000)->task->comm
  
> $41 = &#8220;swapper\000\000\000\000\000\000\000\000&#8221;
  
> //通过下面可以看到地址和current_task相同。
  
> (gdb) p ((struct thread_info *)0xc1698000)->task
  
> $40 = (struct task_struct *) 0xc169f460