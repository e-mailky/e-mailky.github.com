---
layout: post
title:  "内核抢占与preempt_count"
date:   2016-10-13 10:09:35
categories: [编程, Kernel]
tags: [Linux, Kernel,]
description: ""
---

## 1 相关数据结构 

	struct thread_info {
    	struct task_struct  *task;      /* main task structure */
    	struct exec_domain  *exec_domain;   /* execution domain */
    	__u32           flags;      /* low level flags */
    	__u32           status;     /* thread synchronous flags */
    	__u32           cpu;        /* current CPU */
    	int         preempt_count;  /* 0 => preemptable,
                           <0 => BUG */
    	mm_segment_t        addr_limit;
    	struct restart_block    restart_block;
    	void __user     *sysenter_return;
	#ifdef CONFIG_X86_32
    	unsigned long           previous_esp;   /* ESP of the previous stack in
                           case of nested (IRQ) stacks
                        */
		__u8            supervisor_stack[0];
	#endif
    	int         uaccess_err;
	};
	/* how to get the thread information struct from C */
	static inline struct thread_info *current_thread_info(void)
	{
    	return (struct thread_info *)
        	(current_stack_pointer & ~(THREAD_SIZE - 1));
	}
	#define preempt_count() (current_thread_info()->preempt_count)

##2 以下内容转自：内核抢占 

与其他大部分Unix变体和其他大部分的操作系统不同， Linux完整地支持内核抢占。

在不支持内核抢占的内核中，内核代码可以一直执行，到它完成为止。也就是说，调度程序没有办法在一个内核级的任务正在执行的时候重新调度 – 内核中的各任务是协作方式调度的，不具备抢占性。

在2.6版的内核中，内核引入了抢占能力；现在，只要重新调度是安全的，那么内核就可以在任何时间抢占正在执行的任务。

那么，什么时候重新调度才是安全的呢?只要没有持有锁，内核就可以进行抢占。锁是非抢占区域的标志。由于内核是支持SMP的，所以，如果没有持有锁，那么正在执行的代码就是可重新导入的，也就是可以抢占的。

为了支持内核抢占所作的第一处变动就是每个进程的thread_info引入了 preempt_count(thread_info.preempt_count)计数器。该计数器初始值为0，每当使用锁的时候数值加1，释放锁的时候数值减1。当数值为0的时候，内核就可执行抢占。从中断返回内核空间的时候，内核会检查flag和preempt_count的值。如果flag中TIF_NEED_RESCHED被设置，并且preempt_count为0的话，这说明有一个更为重要的任务需要执行并且可以安全地抢占，此时，调度程序就会调度(抢占当前进程)。如果preempt_count不为0，说明当前任务持有锁，所以抢占是不安全的。这时，就会像通常那样直接从中断返回当前执行进程。 如果当前进程所持有的所有的锁都被释放了,那么preemptcount就会重新为0。此时，释放锁的代码会检查need_resched是否被设置。如果是的话，就会调用调度程序。有些内核代码需要允许或禁止内核抢占。

如果内核中的进程被阻塞了，或它显式地调用了schedule()，内核抢占也会显式地发生。这种形式的内核代码从来都是受支持的，因为根本无需额外的逻辑来保证内核可以安全地发生被抢占。如果代码显式的调用了schedule()，那么它应该清楚自己是可以安全地被抢占的。

内核抢占发生在: 
当"从中断处理程序"正在执行，且返回内核空间之前 
内核代码再一次具有可抢占性的时候 
如果内核中的任务显式的调用schedule() 
如果内核中的任务阻塞(这同样也会导致调用schedule()) 

 注:
	current->threadinfo.flags中TIF_NEED_RESCHED为1，表示当前进程需要执行schedule()释放CPU控制权 
	current->threadinfo.preemptcount的值不为0，表示当前进程持有锁不能释放CPU控制权(不能被抢占)

当从内核态返回到用户态的时候，要检查是否进行调度，而调度要看两个条件：

__1. preempt_count是否为0__

__2. rescheduled是否置位__

检查preempt_count的时候，是统一检查是否为0，也就是说，__有4个条件限制，可能不能够进行调度。__

1. preempt_disable()

    #define preempt_disable() \
		do { \
			inc_preempt_count(); \
    		barrier(); \
    	while (0)

	#define inc_preempt_count() add_preempt_count(1)
 
会在preempt_enable中释放
 
2. add\_preempt\_count(HARDIRQ_OFFSET)
是在irq_enter()中调用的，记录进入硬件中断处理的次数（计数），会在irq_exit()中释放

3. \_\_local\_bh\_disable((unsigned long)\_\_builtin\_return\_address(0));
在do_softirq中调用的，

	static inline void __local_bh_disable(unsigned long ip)
	{
 		add_preempt_count(SOFTIRQ_OFFSET);
		barrier();
	}
 
4. 总开关，第一位。

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载: ](http://blog.csdn.net/zjujoe/archive/2009/05/15/4189612.aspx)
{% endhighlight %}

