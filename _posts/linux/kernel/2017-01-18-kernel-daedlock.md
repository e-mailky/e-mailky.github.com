---
layout: post
title:  Linux内核死锁检测机制
categories: [Linux]
tags: [Linux, Kernel, 进程管理]
description: ""
---

&emsp;&emsp;&emsp;&emsp;死锁就是多个进程（线程）因为等待别的进程已占有的自己所需要的资源而陷入阻塞的一种状态，
死锁状态一旦形成，进程本身是解决不了的，需要外在的推动，才能解决，最重要的是死锁不仅仅影响进程业务，
而且还会占用系统资源，影响其他进程。所以内核中设计了内核死锁检测机制，一旦发现死锁进程，就重启OS，
快刀斩乱麻解决问题。之所以使用重启招数，还是在于分布式系统中可以容忍单点崩溃，不能容忍单点进程计算异常，
否则进行死锁检测重启OS就得不偿失了。

&emsp;&emsp;&emsp;&emsp;内核提供自旋锁、信号量等锁形式的工具，具体不再赘述。

&emsp;&emsp;&emsp;&emsp;Linux内核死锁主要分为分为两种：D状态死锁和R状态死锁。

## 一、D状态死锁检测

&emsp;&emsp;&emsp;&emsp;D状态死锁：进程长时间处于TASK\_UNINTERRUPTIBLE而不恢复的状态。
进程处于TASK\_UNINTERRUPTIBLE状态，不响应其他信号(kill -9)，保证一些内核原子操作不被意外中断。
但这种状态时间长就表示进程异常了，需要处理。

内核D状态死锁检测就是hung\_task机制，主要代码就在kernel/hung\_task.c文件。

具体实现原理：

1. 创建Normal级别的khungtaskd内核线程，在死循环中每隔sysctl\_hung\_task\_timeout\_secs时间后check一下，
用schedule_timeout定时（节约定时器浪费的CPU）。
2. 调用do\_each\_thread,while\_each\_thread宏遍历所有的进程信息，如果有D状态进程，则检查最近切换次数
和task计算是否一致，即最近是否有调度切换，如果一致，则没有切换，打印相关信息，并根据sysctl\_hung\_task
\_panic开关决定是否重启。

对应用户态控制的proc接口有：

**/proc/sys/kernel/hung\_task\_timeout\_secs，hung\_task\_panic等。**

## 二、R状态死锁检测

&emsp;&emsp;&emsp;&emsp;R状态死锁：进程长时间处于TASK_RUNNING 状态抢占CPU而不发生切换，一般是，
进程关抢占后一直执行任务，或者进程关抢占后处于死循环或者睡眠，此时往往会导致多个CPU互锁，整个系统异常。

补充：lockdep不是所谓的死锁。

内核R状态死锁检测机制就是lockdep机制，入口即是lockup\_detector\_init函数。

1. 通过cpu\_callback函数调用watchdog\_enable，在每个CPU core上创建SCHED_FIFO级别的实时线程watchdog，
其中使用了hrtimer定时器，控制检查周期。
2. hrtimer定时器调用watchdog\_timer\_fn进行清狗的时间检查，而线程则每次重置清狗时间，
如果watchdog\_timer\_fn发现狗的重置时间已经和当前时间差出危险值，则根据开关进行panic处理。

对应用户态控制的proc接口有：

**/proc/sys/kernel/watchdog_thresh，softlockup_panic等。**

&emsp;&emsp;&emsp;&emsp;整个死锁检测机制比较简单，但cpu_callback函数结构性设计巧妙，可以在很多地方参考使用。

```
static int __cpuinit
cpu_callback(struct notifier_block *nfb, unsigned long action, void *hcpu)
{
    int hotcpu = (unsigned long)hcpu;
 
    switch (action) {
    case CPU_UP_PREPARE:
    case CPU_UP_PREPARE_FROZEN:
        watchdog_prepare_cpu(hotcpu);
        break;
    case CPU_ONLINE:
    case CPU_ONLINE_FROZEN:
        if (watchdog_enabled)
            watchdog_enable(hotcpu);
        break;
#ifdef CONFIG_HOTPLUG_CPU
    case CPU_UP_CANCELED:
    case CPU_UP_CANCELED_FROZEN:
        watchdog_disable(hotcpu);
        break;
    case CPU_DEAD:
    case CPU_DEAD_FROZEN:
        watchdog_disable(hotcpu);
        break;
#endif /* CONFIG_HOTPLUG_CPU */
    }
 
    /*
     * hardlockup and softlockup are not important enough
     * to block cpu bring up.  Just always succeed and
     * rely on printk output to flag problems.
     */
    return NOTIFY_OK;
}
```

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [OENHAN]
{% endhighlight %}

[Linux内核死锁检测机制](http://www.oenhan.com/kernel-deadlock-check)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
