---
layout: post
title:  "内核API"
date:   2016-10-14 17:09:35
categories: [编程, Linux, Kernel]
tags: [Linux, Kernel, ]
description: ""
---

第 2 部分：可延迟函数、内核微线程以及工作队列
---


&emsp;&emsp;本文研究多个用于在内核环境当中延迟处理的方法（特别是在 Linux 内核版本 2.6.27.14 当中）。
尽管这些方法针对 Linux 内核，但方法背后的理念， 对于系统架构研究具有更广泛的意义。例如，
可以将这些理念应用到传统的嵌入式系统当中，取代原有的调度程序来进行任务调度 。

&emsp;&emsp;在开始研究用于内核中的可延迟函数之前， 让我们先了解一下相关问题的背景情况。 
操作系统会因为一个硬件事件而产生中断（例如出现了一个来自网卡的数据包）， 对该事件的处理过程从一个中断开始。 
通常，中断会导致大量任务停止。 其中一些任务是在中断上下文完成的，然后任务被传递给软件栈来继续处理（参见 图 1）。

图 1. Top-half 以及 bottom-half 处理过程

![图 1. Top-half 以及 bottom-half 处理过程]({{ IMAGE_PATH }}/images/figure1.gif)

&emsp;&emsp;问题在于，有多少任务需要在中断上下文完成？ 关于中断上下文的问题是，在此期间部分或者全部中断可以被禁止，
这就增加了处理其他硬件问题的时延（并导致处理习惯的改变）。 因此，有必要简化中断过程中要完成的任务， 
把一些任务转移到内核上下文中去完成（在该上下文， 处理器资源更有可能被高效共享）。

&emsp;&emsp;正如 图 1 所示, 在中断上下文所完成的处理过程称为 top half，
基于中断并被推出中断上下文之外的处理过程称为 bottom half （top half 要依据 bottom half 来安排后续的处理过程）。
bottom-half 处理过程在内核上下文完成，这意味着允许中断操作。 因此通过延迟时间不敏感的任务，
来更迅速处理高频率中断事件， 能够带来性能的优化。

### 有关 bottom halves 的历史

&emsp;&emsp;最近的 2.6 内核中有几个不同的计时器模式，其中最简单、最不精确（但适用于大多数实例）
的模式就是计时器 API。这个 API 允许构造在 jiffies 域（最低 4ms 超时）中运行的计时器。
还有一个高精确度计时器 API，它允许构造在以纳秒定义的时间中运行的计时器。
根据您的处理器和处理器运行的速度，您的里程（mileage）可能会不同，但这个 API 的确提供了一种方法来在
 jiffies 滴答间隔下调度超时。

### 标准计时器

&emsp;&emsp;Linux 能够快速响应各种功能需求，延迟功能也不例外。 自 Linux2.3版本内核开始，
就提供了软中断功能， 具有一组 32 个静态定义的 bottom halves。 作为静态元素，其定义在编译过程中完成
（不同于新的动态机制）。 软中断用于在内核线程上下文中处理时间要求严格的处理过程（软件中断）。
可以在 ./kernel/softirq.c 中找到软中断的来源。 在 2.3 版本的 Linux 内核中还引入了微线程
（参见 ./include/linux/interrupt.h）。 微线程的构建基于软中断，用于允许动态生成可延迟函数。
最终，在 2.5 版本 Linux 内核中引入了工作队列（参见 ./include/linux/workqueue.h）。 
工作队列允许将任务延迟到中断上下文之外，进入内核处理上下文。

&emsp;&emsp; 现在我们探讨一下任务延迟、微线程以及工作队列的动态机制。

### 微线程介绍

&emsp;&emsp;软中断最初为具有 32 个软中断条目的矢量， 用来支持一系列的软件中断特性。 
当前，只有 9 个矢量被用于软中断， 其中之一是 TASKLET_SOFTIRQ（参见 ./include/linux/interrupt.h）。 
虽然软中断还存在于内核中，推荐采用微线程和工作队列，而不是分配新的软中断矢量。

&emsp;&emsp;微线程是一个延迟方法，可以实现将已登记的函数进行推后运行。
top half（中断处理程序）完成少量的任务，然后安排微线程在晚些的 bottom half 中执行。

#### 清单 1. 声明并调度微线程

```
/* Declare a Tasklet (the Bottom-Half) */
void tasklet_function( unsigned long data );

DECLARE_TASKLET( tasklet_example, tasklet_function, tasklet_data );

...

/* Schedule the Bottom-Half */
tasklet_schedule( &tasklet_example );
```


&emsp;&emsp;一个给定的微线程只运行在一个 CPU 中（就是用于调用该微线程的那个 CPU），
同一微线程永远不会同时运行在多个 CPU 中。 但是不同的微线程可以同时运行在不同的 CPU 中。

&emsp;&emsp;微线程可由 tasklet\_struct 结构体表示（参见 图 2）， 
其中包含了用于管理和维护微线程的必要数据 （状态，通过 atomic\_t 来实现允许/禁止控制，函数指针，数据，以及链表引用）。

图 2. tasklet_struct 结构体的内部情况

![图 2. tasklet_struct 结构体的内部情况]({{ IMAGE_PATH }}/images/figure2.gif)

通过软中断机制来调度微线程，当机器处于严重软件中断负荷之下时， 可通过 ksoftirqd（一种每 CPU 内核线程）软中断来调度。 下面将探讨微线程应用编程接口（API）中支持的各类函数

### 微线程 API

&emsp;&emsp;微线程通过宏调用来定义 DECLARE\_TASKLET（参见 清单 2）。 在底层，
该宏调用只是利用所提供的信息对结构体 tasklet\_struct 进行初始化（微线程名，函数， 以及微线程专有数据）。 
默认情况下，微线程处于允许状态，这意味着它可以被调度。 还可以利用宏 DECLARE\_TASKLET\_DISABLED 
将微线程默认声明为禁止状态。 这时需要调用函数 tasklet\_enable 来实现微线程可被调度。 
可以分别利用函数 tasklet\_enable 和函数 tasklet\_disable 实现允许和禁止一个微线程（从调度的角度）。 
函数 tasklet\_init 也存在并利用用户提供的微线程数据来对 tasklet_struct 进行初始化。

#### 清单 2. 微线程的创建以及 enable/disable 函数

```
DECLARE_TASKLET( name, func, data );
DECLARE_TASKLET_DISABLED( name, func, data);
void tasklet_init( struct tasklet_struct *, void (*func)(unsigned long),
      unsigned long data );
void tasklet_disable_nosync( struct tasklet_struct * );
void tasklet_disable( struct tasklet_struct * );
void tasklet_enable( struct tasklet_struct * );
void tasklet_hi_enable( struct tasklet_struct * );
```

&emsp;&emsp;有两个 disable 函数，每一个都对微线程发出 disable 请求， 但是，微线程被终止后，
只有 tasklet\_disable 返回（其中 tasklet\_disable\_nosync 可能在终止发生之前返回）。
disable 函数允许微线程被 “掩码”（也就是说，并不执行），直到 enable 函数被调用为止。 
存在两个 enable 函数： 一个用于正常优先级调度（tasklet\_enable），另一个用于允许高优先级调度
（tasklet\_hi_enable）。 正常优先级调度通过 TASKLET\_SOFTIRQ-level 软中断来执行， 
高优先级调度则通过 HI_SOFTIRQ-level 软中断执行。

&emsp;&emsp;由于存在正常优先级和高优先级的 enable 函数， 因此要有正常优先级和高优先级的调度函数（参见 清单 3）。 
每个函数利用特殊的软中断矢量来为微线程排队（tasklet\_vec 用于正常优先级， 而 tasklet\_hi\_vec 用于高优先级）。 
来自高优先级矢量的微线程先得到服务，随后是来自正常优先级矢量的微线程。 
注意，每个 CPU 维持其自己的正常优先级和高优先级软中断矢量。

### 清单 3. 微线程调度函数

```
void tasklet_schedule( struct tasklet_struct * );
void tasklet_hi_schedule( struct tasklet_struct * );
```


&emsp;&emsp;最后，微线程生成之后，就可以通过函数 tasklet\_kill 来停止微线程（参见 清单 4）。 
函数 tasklet\_kill 保证微线程不会再运行， 并且，如果按进度该微线程应该运行，将会等到它运行完，
然后再 kill 该线程。 tasklet_kill_immediate 只在指定的 CPU 处于 dead 状态时被采用。

#### 清单 4. 微线程 kill 函数

    void tasklet_kill( struct tasklet_struct * );
    void tasklet_kill_immediate( struct tasklet_struct *, unsigned int cpu );

&emsp;&emsp;通过该 API，可见微线程 API 比较简单，实现也很简单。 可以通过 ./kernel/softirq.c 
和 ./include/linux/interrupt.h 来了解微线程的实现机制。

### 有关微线程的简单例子

&emsp;&emsp;我们来看一个使用微线程 API 的简单例子（参见 清单 5）。 如这里所示，
微线程函数（my\_tasklet\_function 和 my\_tasklet\_data）通过相关数据生成， 然后由 DECLARE\_TASKLET 
来声明一个新的微线程。 当该模块被插入后，微线程将被调度，这保证它在今后可执行。 当该模块被卸载，
函数 tasklet_kill 将被调用来保证微线程不处于可调度状态。


&emsp;&emsp;我们来检查一下这些 API 函数的实际运行情况。清单 1 提供了一个简单的内核模块，
用于展示简单计时器 API 的核心特点。在 init\_module 中，您使用 setup\_timer 初始化了一个计时器，
然后调用 mod\_timer 来启动它。当计时器过期时，将调用回调函数 my\_timer\_callback。最后，
当您删除模块时，计时器删除（通过 del\_timer）发生。（注意来自 del_timer 的返回检查，它确定计时器是否还在使用。）

#### 清单 5. 微线程处于内核模块上下文的简单例子

```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/interrupt.h>

MODULE_LICENSE("GPL");

char my_tasklet_data[]="my_tasklet_function was called";

/* Bottom Half Function */
void my_tasklet_function( unsigned long data )
{
  printk( "%s\n", (char *)data );
  return;
}

DECLARE_TASKLET( my_tasklet, my_tasklet_function, 
     (unsigned long) &my_tasklet_data );

int init_module( void )
{
  /* Schedule the Bottom Half */
  tasklet_schedule( &my_tasklet );

  return 0;
}

void cleanup_module( void )
{
  /* Stop the tasklet before we exit */
  tasklet_kill( &my_tasklet );

  return;
}
```
---

### 工作队列介绍

&emsp;&emsp;工作队列是实现延迟的新机制，从 2.5 版本 Linux 内核开始提供该功能。 不同于微线程一步到位的延迟方法，
工作队列采用通用的延迟机制， 工作队列的处理程序函数能够休眠（这在微线程模式下无法实现）。 
工作队列可以有比微线程更高的时延，并为任务延迟提供功能更丰富的 API。 从前，延迟功能通过 keventd 对任务排队来实现，
但是现在由内核工作线程 events/X 来管理。

&emsp;&emsp;工作队列提供一个通用的办法将任务延迟到 bottom halves。 处于核心的是工作队列
（结构体 workqueue\_struct）， 任务被安排到该结构体当中。 任务由结构体 work\_struct 来说明， 
用来鉴别哪些任务被延迟以及使用哪个延迟函数（参见 图 3）。 events/X 内核线程（每 CPU 一个）
从工作队列中抽取任务并激活一个 bottom-half 处理程序（由处理程序函数在结构体 work_struct 中指定）。

图 3. 工作队列背后的处理过程

![图 3. 工作队列背后的处理过程]({{ IMAGE_PATH }}/images/figure3.gif)

&emsp;&emsp;由于 work_struct 中指出了要采用的处理程序函数， 因此可以利用工作队列来为不同的处理程序进行任务排队。 
现在，让我们看一下能够用于工作队列的 API 函数。

### 工作队列 API

&emsp;&emsp;工作队列 API 比微线程稍复杂，主要是因为它支持很多选项。 
我们首先探讨一下工作队列，然后再看一下任务和变体。

&emsp;&emsp;通过 图 3 可以回想工作队列的核心结构体是队列本身。 该结构体用于将任务安排出 top half ，
进入 bottom half ，从而延迟它的执行。 工作队列通过宏调用生成 create\_workqueue，
返回一个 workqueue\_struct 参考值。 可以通过调用函数 destroy_workqueue 来远程遥控工作队列（如果需要）：

    struct workqueue_struct *create_workqueue( name );
    void destroy_workqueue( struct workqueue_struct * );

&emsp;&emsp;通过工作队列与之通信的任务可以由结构体 work\_struct 来定义。 通常，
该结构体是用来进行任务定义的结构体的第一个元素（后面有相关例子）。 
工作队列 API 提供三个函数来初始化任务（通过一个事先分配的缓存）； 参见 清单 6。 
宏 INIT\_WORK 提供必需的初始化数据以及处理程序函数的配置（由用户传递进来）。 
如果开发人员需要在任务被排入工作队列之前发生延迟，可以使用宏 INIT\_DELAYED\_WORK 和 INIT\_DELAYED\_WORK_DEFERRABLE。

#### 清单 6. 任务初始化宏

```
INIT_WORK( work, func );
INIT_DELAYED_WORK( work, func );
INIT_DELAYED_WORK_DEFERRABLE( work, func );
```

&emsp;&emsp;任务结构体的初始化完成后，接下来要将任务安排进工作队列。 可采用多种方法来完成这一操作
（参见 清单 7）。 首先，利用 queue\_work 简单地将任务安排进工作队列（这将任务绑定到当前的 CPU）。 
或者，可以通过 queue\_work\_on 来指定处理程序在哪个 CPU 上运行。 两个附加的函数为延迟任务提供相同的功能
（其结构体装入结构体 work\_struct 之中，并有一个 计时器用于任务延迟 ）。

#### 清单 7. 工作队列函数

```
int queue_work( struct workqueue_struct *wq, struct work_struct *work );
int queue_work_on( int cpu, struct workqueue_struct *wq, struct work_struct *work );

int queue_delayed_work( struct workqueue_struct *wq,
      struct delayed_work *dwork, unsigned long delay );

int queue_delayed_work_on( int cpu, struct workqueue_struct *wq,
      struct delayed_work *dwork, unsigned long delay );
```

可以使用全局的内核全局工作队列，利用 4 个函数来为工作队列定位。 这些函数（见 清单 8）模拟 清单 7，只是不需要定义工作队列结构体。



#### 清单 8. 内核全局工作队列函数

```
int schedule_work( struct work_struct *work );
int schedule_work_on( int cpu, struct work_struct *work );

int scheduled_delayed_work( struct delayed_work *dwork, unsigned long delay );
int scheduled_delayed_work_on( 
    int cpu, struct delayed_work *dwork, unsigned long delay );
```

&emsp;&emsp;还有一些帮助函数用于清理或取消工作队列中的任务。想清理特定的任务项目并阻塞任务， 
直到任务完成为止， 可以调用 flush\_work 来实现。 指定工作队列中的所有任务能够通过调用 flush\_workqueue 来完成。 
这两种情形下，调用者阻塞直到操作完成为止。 为了清理内核全局工作队列，可调用 flush\_scheduled_work。

```
int flush_work( struct work_struct *work );
int flush_workqueue( struct workqueue_struct *wq );
void flush_scheduled_work( void );
```



&emsp;&emsp;还没有在处理程序当中执行的任务可以被取消。 调用 cancel\_work\_sync 
将会终止队列中的任务或者阻塞任务直到回调结束（如果处理程序已经在处理该任务）。 
如果任务被延迟，可以调用 cancel\_delayed\_work\_sync。

    int cancel_work_sync( struct work_struct *work );
    int cancel_delayed_work_sync( struct delayed_work *dwork );

&emsp;&emsp;最后，可以通过调用 work\_pending 或者 delayed\_work_pending 来确定任务项目是否在进行中。

    work_pending( work );
    delayed_work_pending( work );

&emsp;&emsp;这就是工作队列 API 的核心。在 ./kernel/workqueue.c 中能够找到工作队列 API 的实现方法， 
API 在 ./include/linux/workqueue.h 中定义。 下面我们看一个工作队列 API 的简单例子。

### 工作队列简单例子

&emsp;&emsp;下面的例子说明了几个核心的工作队列 API 函数。 如同微线程的例子一样，为方便起见，可将这个例子部署在内核模块上下文。

&emsp;&emsp;首先，看一下将用于实现 bottom half 的任务结构体和处理程序函数（参见 清单 9）。 
首先您将注意到工作队列结构体参考的定义 （my\_wq）以及 my\_work\_t 的定义。 my\_work\_t 类型定义的头部包括结构体 
work\_struct 和一个代表任务项目的整数。 处理程序（回调函数）将 work\_struct 指针引用改为 
my\_work_t 类型。 发送出任务项目（来自结构体的整数）之后，任务指针将被释放。

#### 清单 9. 任务结构体和 bottom-half 处理程序

```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/workqueue.h>

MODULE_LICENSE("GPL");

static struct workqueue_struct *my_wq;

typedef struct {
  struct work_struct my_work;
  int    x;
} my_work_t;

my_work_t *work, *work2;


static void my_wq_function( struct work_struct *work)
{
  my_work_t *my_work = (my_work_t *)work;

  printk( "my_work.x %d\n", my_work->x );

  kfree( (void *)work );

  return;
}
```

&emsp;&emsp;清单 10 是 init\_module 函数， 该函数从使用 create\_workqueue API 函数生成工作队列开始。 
成功生成工作队列之后，创建两个任务项目（通过 kmalloc 来分配）。 利用 INIT\_WORK 来初始化每个任务项目，
任务定义完成， 接着通过调用 queue_work 将任务安排到工作队列中。 top-half 进程（在此处模拟）完成。
如同清单 10 中所示，任务有时会晚些被处理程序处理。

#### 清单 10. 工作队列和任务创建

```
int init_module( void )
{
  int ret;

  my_wq = create_workqueue("my_queue");
  if (my_wq) {

   /* Queue some work (item 1) */
    work = (my_work_t *)kmalloc(sizeof(my_work_t), GFP_KERNEL);
    if (work) {

      INIT_WORK( (struct work_struct *)work, my_wq_function );

      work->x = 1;

      ret = queue_work( my_wq, (struct work_struct *)work );

    }

    /* Queue some additional work (item 2) */
    work2 = (my_work_t *)kmalloc(sizeof(my_work_t), GFP_KERNEL);
    if (work2) {

      INIT_WORK( (struct work_struct *)work2, my_wq_function );

      work2->x = 2;

      ret = queue_work( my_wq, (struct work_struct *)work2 );

    }

  }

  return 0;
}
```

&emsp;&emsp;最终的元素在 清单 11 中展示。 在模块清理过程中，会清理一些特别的工作队列
（它们将保持阻塞状态直到处理程序完成对任务的处理）， 然后销毁工作队列。

#### 清单 11. 工作队列清理和销毁

```
void cleanup_module( void )
{
  flush_workqueue( my_wq );

  destroy_workqueue( my_wq );

  return;
}
```

## 微线程与工作队列的区别

&emsp;&emsp;从对微线程和工作队列的简短介绍中， 可以发现两个将任务从 top halves 延迟到 bottom halves 
的不同方法。 微线程提供低延迟机制，该方式简单而直接， 而工作队列提供复杂的 API 来允许对多个任务项目进行排队。 
每种方法都在中断上下文延迟任务，但只有微线程采用 run-to-complete 的风格自动运行， 而在此处，
如果需要，工作队列允许处理程序休眠。 为有效实现任务延迟，可根据具体需求来选择相应的方法。



### 更进一步

&emsp;&emsp;这里所探讨的任务延迟方法涉及了历史的和当前的应用在 Linux 内核中的延迟方法（除了计时器之外，
这将在以后的文章中讨论）。 它们当然不是新的 — 事实上，它们在过去已经以其他形式存在 — 
但是它们代表了一种有趣的架构模式，这在 Linux 中和其他地方都很有用。 从软中断到微线程再到任务队列再
到延迟的工作队列，Linux 在提供一致的和兼容的用户空间体验的同时，保持其内核各方面的持续发展。

---

**参考资料**
学习
大部分来自互联网的有关微线程和工作队列的可用信息都有些过时。关于 工作队列 API 返工的介绍，可以参考 LWN.net。还可以从 Linux Device Drivers 一书当中学习一些有用的知识。
I'll do it later: Softirqs, Tasklets, Bottom Halves, Task Queues, Work Queues and Timers（PDF）的作者为 Matthew Wilcox，是介绍 Linux 当中各种延迟机制的重要著作。
这一重要的演示文稿出自 Jennifer Hou（来自伊利诺伊大学厄本那—香槟分校计算机科学系）， 提供了一个对 Linux 内核的精彩论述，对软中断的介绍，以及关于利用微线程实现任务延迟的简短介绍。
这一概要来自于旧金山大学的高级系统编程课程（由 Emeritus 教授和 Allan Cruse 博士讲授），提供了大量的 有关工作延迟的资源 （包括驱动着它们的理念）。
“Linux Device Drivers” 这本富有创意的图书提供了一个 针对任务延迟的有益介绍 （PDF）。在其中一个免费章节（“Timers, Delays, and Deferred Work”）中， 能够找到对微线程和工作队列的深刻（尽管有些过时的）论述。
想获得更多 有关应该在何时何地采用微线程或者工作队列的信息， 可查看位于内核邮件清单中的交流区。
在这一演示文稿当中，哥伦比亚大学讲师 Junfeng Yang 提供了一个优秀的 Linux 内的中断和系统调用简介（PDF）。
这一讲稿由台北清华大学的 Tai-Yi Huang 博士提供， 其中有一个不错的 Linux 内核中断和异常简介（PowerPoint）。 另外，本讲稿还探讨了软中断、微线程和工作队列的相关主题，并提供了样例代码。
在 developerWorks Linux 专区 寻找为 Linux 开发人员（包括 Linux 新手入门）准备的更多参考资料，查阅我们 最受欢迎的文章和教程。
在 developerWorks 上查阅所有 Linux 技巧 和 Linux 教程。
随时关注 developerWorks 技术活动和网络广播。
观看 developerWorks 演示中心，包括面向初学者的产品安装和设置演示，以及为经验丰富的开发人员提供的高级功能。
获得产品和技术
以最适合您的方式 IBM 产品评估试用版软件：下载产品试用版，在线试用产品， 在云环境下试用产品，或者在 IBM SOA Sandbox for People 中花费几个小时来学习如何高效实现 Service Oriented Architecture。
讨论
加入 My developerWorks 社区。查看开发人员参与的博客、论坛、组和 wikis，并与其他 developerWorks 用户交流。

---------

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载](http://www.cnblogs.com/hoys/archive/2012/02/22/2363399.html)
{% endhighlight %}

