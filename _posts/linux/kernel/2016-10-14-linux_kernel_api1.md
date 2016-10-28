---
layout: post
title:  "内核API"
date:   2016-10-14 17:09:35
categories: [Linux]
tags: [Linux, Kernel, ]
description: ""
---

第 3 部分: 2.6 内核中的计时器和列表
---

### （Linux）时间的起源

&emsp;&emsp;在 Linux 内核中，时间由一个名为 jiffies 的全局变量衡量，
该变量标识系统启动以来经过的滴答数。在最低的级别上，计算滴答数的方式取决于正在运行的特定硬件平台；
但是，滴答计数通常在一次中断期间仍然继续进行。滴答速率（jiffies 的最不重要的位）可以配置，
但在最近针对 x86 的 2.6 内核中，一次滴答等于 4ms（250Hz）。jiffies 全局变量在内核中广泛使用，
目的有几个，其中之一是提供用于计算一个计时器的超时值的当前绝对时间（稍后将展示一个例子）。

### 内核计时器

&emsp;&emsp;最近的 2.6 内核中有几个不同的计时器模式，其中最简单、最不精确（但适用于大多数实例）
的模式就是计时器 API。这个 API 允许构造在 jiffies 域（最低 4ms 超时）中运行的计时器。
还有一个高精确度计时器 API，它允许构造在以纳秒定义的时间中运行的计时器。
根据您的处理器和处理器运行的速度，您的里程（mileage）可能会不同，但这个 API 的确提供了一种方法来在
 jiffies 滴答间隔下调度超时。

### 标准计时器

&emsp;&emsp;标准计时器 API 作为 Linux 内核的一部分已经有很长一段时间了（自从 Linux 内核的早期版本开始）。尽管它提供的精确性比高精确度计时器要低，但它对于在处理物理设备时提供错误覆盖的传统驱动程序超时来说比较理想。在很多情况下，这些超时实际上从不触发，而是被启动，然后被删除。

&emsp;&emsp;简单内核计时器使用计时器轮（timer wheel） 实现。这个主意是由 Finn Arne Gangstad 在 1997 年首次引入的。它不理睬管理大量计时器的问题，而是很好地管理数量合理的计时器 — 典型情况。（原始计时器实现只是按照过期顺序将计时器实现双重链接。尽管在概念上比较简单，但这种方法是不可伸缩的。）时间轮是一个 buckets 集合，其中每个 bucker 表示将来计时器过期的一个时间块。这些 buckets 使用基于 5 个 bucket 的对数时间定义。使用 jiffies 作为时间粒度，定义了几个组，它们表示将来的过期时段（其中每个组通过一列计时器表示）。计时器插入使用具有 O(1) 复杂度的列表操作发生，过期发生在 O(N) 时间内。计时器过期以串联的形式出现，其中计时器被从高粒度 buckets 删除，然后随着它们的过期时间的下降被插入到低粒度 buckets 中。现在我们查看一下针对这个计时器实现的 API。

### 计时器 API

&emsp;&emsp;Linux 提供了一个简单的 API 来构造和管理计时器。它包含一些函数（和助手函数），
用于创建、取消和管理计时器。

&emsp;&emsp;计时器通过 timer\_list 结构定义，该结构包括实现一个计时器所需的所有数据
（其中包括列表指针和在编译时配置的可选计时器统计数据）。从用户角度看，timer\_list 包含一个过期时间，
一个回调函数（当/如果计时器过期），以及一个用户提供的上下文。用户必须初始化计时器，可以采取几种方法，
最简单的方法是调用 setup\_timer，该函数初始化计时器并设置用户提供的回调函数和上下文。
或者，用户可以设置计时器中的这些值（函数和数据）并简单地调用 init\_timer。
注意，init\_timer 由 setup_timer 内部调用。

```
void init_timer( struct timer_list *timer );
void setup_timer( struct timer_list *timer, 
                     void (*function)(unsigned long), unsigned long data );
```

&emsp;&emsp;拥有一个经过初始化的计时器之后，用户现在需要设置过期时间，这通过调用 mod\_timer 来完成。
由于用户通常提供一个未来的过期时间，他们通常在这里添加 jiffies 来从当前时间偏移。
用户也可以通过调用 del_timer 来删除一个计时器（如果它还没有过期）：

```
int mod_timer( struct timer_list *timer, unsigned long expires );
void del_timer( struct timer_list *timer );
```

最后，用户可以通过调用 timer_pending（如果正在等待，将返回 1）来发现计时器是否正在等待（还没有发出）：

    int timer_pending( const struct timer_list *timer );

### 计时器示例

&emsp;&emsp;我们来检查一下这些 API 函数的实际运行情况。清单 1 提供了一个简单的内核模块，
用于展示简单计时器 API 的核心特点。在 init\_module 中，您使用 setup\_timer 初始化了一个计时器，
然后调用 mod\_timer 来启动它。当计时器过期时，将调用回调函数 my\_timer\_callback。最后，
当您删除模块时，计时器删除（通过 del\_timer）发生。（注意来自 del_timer 的返回检查，它确定计时器是否还在使用。）

#### 清单 1. 探索简单计时器 API

```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/timer.h>

MODULE_LICENSE("GPL");

static struct timer_list my_timer;

void my_timer_callback( unsigned long data )
{
  printk( "my_timer_callback called (%ld).\n", jiffies );
}

int init_module( void )
{
  int ret;

  printk("Timer module installing\n");

  // my_timer.function, my_timer.data
  setup_timer( &my_timer, my_timer_callback, 0 );

  printk( "Starting timer to fire in 200ms (%ld)\n", jiffies );
  ret = mod_timer( &my_timer, jiffies + msecs_to_jiffies(200) );
  if (ret) printk("Error in mod_timer\n");

  return 0;
}

void cleanup_module( void )
{
  int ret;

  ret = del_timer( &my_timer );
  if (ret) printk("The timer is still in use...\n");

  printk("Timer module uninstalling\n");

  return;
}
```

您可以在 ./include/linux/timer.h 中进一步了解计时器 API。尽管简单计时器 API 简单有效，但它并不能提供实时应用程序所需的准确性。为此，我们来看一下 Linux 最近新增的功能，该功能用于支持精确度更高的计时器。

### 高精确度计时器

&emsp;&emsp;高精确度计时器（简称 hrtimers）提供一个高精确度的计时器管理框架，这个框架独立于此前讨论过的计时器框架，原因是合并这两个框架太复杂。尽管计时器在 jiffies 粒度上运行，hrtimers 在纳秒粒度上运行。

&emsp;&emsp;hrtimer 框架的实现方式与传统计时器 API 不同。hrtimer 不使用 buckets 和串联操作，而是维护一个按时间排序的计时器数据结构（按时间顺序插入计时器，以最小化激活时的处理）。这个数据结构是一个 “红-黑” 树，对于注重性能的应用程序很理想（且恰好作为内核中的一个库普遍可用）。

&emsp;&emsp;hrtimer 框架作为内核中的一个 API 可用，用户空间应用程序也可以通过 nanosleep、itimers 和 Portable Operating System Interface (POSIX)-timers interface 使用它。hrtimer 框架被主线化（mainlined）到 2.6.21 内核中。

### 高精确度计时器 API

&emsp;&emsp;hrtimer API 与传统 API 有些相似，但它们之间的一些根本差别是它能够进行额外的时间控制。
应该注意的第一点是：时间不是用 jiffies 表示的，而是以一种名为 ktime 的特殊数据类型表示。
这种表示方法隐藏了在这个粒度上有效管理时间的一些细节。hrtimer API 正式确认（formalize）
了绝对时间和相对时间之间的区别，要求调用者指定类型。

&emsp;&emsp;与传统的计时器 API 类似，高精确度计时器通过一个结构表示 — 这里是 hrtimer。
这个结构从用户角度定义定时器（回调函数、过期时间等）并包含了管理信息
（其中计时器存在于 “红-黑” 树、可选统计数据等中）。

&emsp;&emsp;定义过程首先通过 hrtimer\_init 初始化一个计时器。这个调用包含计时器、
时钟定义和计时器模式（one-shot 或 restart）。使用的时钟在 ./include/linux/time.h 中定义，
表示系统支持的各种时钟（比如实时时钟或者单一时钟，后者只表示从一个起点[比如系统启动]开始的时间）。
计时器被初始化之后，就可以通过 hrtimer\_start 启动。这个调用包含过期时间（在 ktime_t 中）
和时间值的模式（绝对或相对值）。

    void hrtimer_init( struct hrtimer *time, clockid_t which_clock, 
            enum hrtimer_mode mode );
    int hrtimer_start(struct hrtimer *timer, ktime_t time, const 
            enum hrtimer_mode mode);


&emsp;&emsp;hrtimer 启动后，可以通过调用 hrtimer\_cancel 或 hrtimer\_try\_to\_cancel 来取消。
每个函数都包含将被停止的计时器的 hrtimer 引用。这两个函数的区别在于：hrtimer\_cancel 函数试图取消计时器，
但如果计时器已经发出，那么它将等待回调函数结束；hrtimer\_try\_to\_cancel 函数也试图取消计时器，
但如果计时器已经发出，它将返回失败。

    int hrtimer_cancel(struct hrtimer *timer);
    int hrtimer_try_to_cancel(struct hrtimer *timer);

&emsp;&emsp;可以通过调用 hrtimer\_callback\_running 来检查 hrtimer 是否已经激活它的回调函数。
注意，这个函数由 hrtimer\_try\_to_cancel 内部调用，以便在计时器的回调函数被调用时返回一个错误。

    int hrtimer_callback_running(struct hrtimer *timer);

### 一个 hrtimer 示例

&emsp;&emsp;hrtimer API 的使用方法非常简单，如 清单 2 所示。在 init\_module 中，
首先定义针对超时的相对时间（本例中为 200ms）。然后，通过调用 hrtimer\_init 来初始化您的
 hrtimer（使用单一时钟），并设置回调函数。最后，使用此前创建的 ktime 值启动计时器。当计时器发出时，
 将调用 my\_hrtimer\_callback 函数，该函数返回 HRTIMER\_NORESTART，以避免计时器自动重新启动。
 在 cleanup\_module 函数中，通过调用 hrtimer_cancel 来取消计时器。

#### 清单 2. 探索 hrtimer API

```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/hrtimer.h>
#include <linux/ktime.h>

MODULE_LICENSE("GPL");

#define MS_TO_NS(x) (x * 1E6L)

static struct hrtimer hr_timer;

enum hrtimer_restart my_hrtimer_callback( struct hrtimer *timer )
{
  printk( "my_hrtimer_callback called (%ld).\n", jiffies );

  return HRTIMER_NORESTART;
}

int init_module( void )
{
  ktime_t ktime;
  unsigned long delay_in_ms = 200L;

  printk("HR Timer module installing\n");

  ktime = ktime_set( 0, MS_TO_NS(delay_in_ms) );

  hrtimer_init( &hr_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL );
  
  hr_timer.function = &my_hrtimer_callback;

  printk( "Starting timer to fire in %ldms (%ld)\n", delay_in_ms, jiffies );

  hrtimer_start( &hr_timer, ktime, HRTIMER_MODE_REL );

  return 0;
}

void cleanup_module( void )
{
  int ret;

  ret = hrtimer_cancel( &hr_timer );
  if (ret) printk("The timer was still in use...\n");

  printk("HR Timer module uninstalling\n");

  return;
}
```

&emsp;&emsp;关于 hrtimer API，还有许多内容这里没有涉及到。一个有趣的方面是它能够定义回调函数的
执行上下文（比如在 softirq 或 hardiirq 上下文中）。您可以在 ./include/linux/hrtimer.h 文件中进一步了解 hrtimer API。

----

## 内核列表

&emsp;&emsp;如本文此前所述，列表是有用的结构，内核提供了一个有效的通用使用实现。另外，
您将在我们此前讨论过的 APIs 下面发现列表。理解这个双重链接的列表 API 有助于使用这个有效的数据结构进行开发，
您会发现，代码在这个利用列表的内核中是多余的。现在我们来快速了解一下这个内核列表 API。

&emsp;&emsp;这个 API 提供一个 list\_head 结构，用于表示列表头（锚点）和结构内（in-structure）列表指针。
我们来检查一个包含列表功能的样例结构（参见 清单 3）。注意，清单 3 添加了 list\_head 结构，
该结构用于对象链接（object linkage）。注意，可以在您的结构中的任意位置添加这个 list\_head 结构 — 
通过一些 GCC（list\_entry 和 container\_of，在 ./include/kernel/kernel.h 中定义）— 
可以取消从列表指针到超对象的引用。

#### 清单 3. 带有列表引用的样例结构

```
struct my_data_structure {
    int value;
    struct list_head list;
};
```

&emsp;&emsp;与其他列表实现一样，需要一个列表头来充当列表的锚点。这通常通过 LIST\_HEAD 宏来完成，
这个宏提供列表的声明和初始化。这个宏创建一个结构 list_head 对象，可以在该对象上添加其他一些对象。

    LIST_HEAD( new_list )

也可以通过使用 LIST\_HEAD_INIT 宏手动创建一个列表头（例如，您的列表头位于另一个结构中）。
主初始化完成后，可以使用 list\_add 和 list_del 等函数来操纵列表。
下面，我们将跳到示例代码，以便更好地解释这个 API 的使用方法。

### 列表 API 示例

&emsp;&emsp;清单 4 提供一个简单的内核模块来探索几个列表 AIO 函数（./include/linux/list.h 中包含更多函数）。
这个示例创建了两个列表，使用 init\_module 函数来填充它们，然后使用 cleanup_module 函数来操纵这两个列表。

&emsp;&emsp;一开始，您创建了您的数据结构（my\_data\_struct），该结构包含一些数据和两个列表头。
这个示例展示可以将一个对象同时插入到多个列表中。然后，您创建了两个列表头（my\_full\_list 和 my\_odd_list）。

&emsp;&emsp;在 init\_module 函数中，您创建了 10 个数据对象，并使用 list\_add 函数将它们加载到列表中
（所有对象加载到 my\_full\_list 中，所有奇值对象加载到 my\_odd\_list) 中）。这里要注意一点：list\_add 
接受两个参数，一个是将用到的对象中的列表引用，另一个是列表锚点。这个示例展示了将一个数据对象插入多个
列表的能力，方法是使用内核的内部机制来识别包含列表引用的超级对象。

&emsp;&emsp;cleanup\_module 函数提供了这个 API 的其他几个功能，其中之一是 list\_for\_each 宏，
这个宏简化了列表迭代。对于这个宏，您提供了一个对当前对象（pos）的引用以及将被迭代的列表引用。
对于每次迭代，您接收一个 list\_head 引用，list\_entry 接收这个引用以识别容器对象（您的数据结构）。
指定您的结构和结构之内的列表变量，后者用于在内部取消引用，返回容器。

&emsp;&emsp;为发出奇值列表（odd list），要使用另一个名为 list\_for\_each\_entry 的迭代宏。这个宏更简单，
因为它自动提供数据结构，无需再执行一个 list\_entry 函数。

&emsp;&emsp;最后，使用 list\_for\_each\_safe 来迭代列表，以便释放已分配的元素。这个宏将迭代列表，
但阻止删除列表条目（删除列表条目是迭代操作的一部分）。您使用 list\_entry 来获取您的数据对象
（以便将它释放回内核池），然后使用 list_del 来释放列表中的条目。

#### 清单 4. 探索列表 API

```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/list.h>

MODULE_LICENSE("GPL");

struct my_data_struct {
  int value;
  struct list_head full_list;
  struct list_head odd_list;
};

LIST_HEAD( my_full_list );
LIST_HEAD( my_odd_list );


int init_module( void )
{
  int count;
  struct my_data_struct *obj;

  for (count = 1 ; count < 11 ; count++) {

    obj = (struct my_data_struct *)
            kmalloc( sizeof(struct my_data_struct), GFP_KERNEL );

    obj->value = count;

    list_add( &obj->full_list, &my_full_list );

    if (obj->value & 0x1) {
      list_add( &obj->odd_list, &my_odd_list );
    }

  }

  return 0;
}


void cleanup_module( void )
{
  struct list_head *pos, *q;
  struct my_data_struct *my_obj;

  printk("Emit full list\n");
  list_for_each( pos, &my_full_list ) {
    my_obj = list_entry( pos, struct my_data_struct, full_list );
    printk( "%d\n", my_obj->value );
  }

  printk("Emit odd list\n");
  list_for_each_entry( my_obj, &my_odd_list, odd_list ) {
    printk( "%d\n", my_obj->value );
  }

  printk("Cleaning up\n");
  list_for_each_safe( pos, q, &my_full_list ) {
    struct my_data_struct *tmp;
    tmp = list_entry( pos, struct my_data_struct, full_list );
    list_del( pos );
    kfree( tmp );
  }

  return;
}
```
&emsp;&emsp;还有很多其他函数，它们的用途包括在列表末尾而不是头部添加数据（list\_add\_tail）、
连接列表（list\_splice）、测试列表的内容（list\_empty）等。请参见 参考资料 详细了解内核列表函数。

### 结束语

&emsp;&emsp; 本文探索了几个 API，展示了在必要时隔离功能的能力（计时器 API 和高精确度 hrtimer API）
以及编写通用代码以实现代码重用的能力（列表 API）。传统计时器为典型的驱动程序超时提供了一种有效的机制，
而 hrtimer 为更精确的计时器功能提供了更高水平的服务质量。列表 API 提供了一个非常通用，
但功能丰富的高效接口。当您编写内核代码时，您将遇到一个或多个这样的 API，因此它们肯定值得深入研究。



---------

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载](http://www.cnblogs.com/hoys/archive/2012/02/22/2363399.html)
{% endhighlight %}

