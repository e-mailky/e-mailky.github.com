---
layout: post
title:  深入理解Linux网络技术内幕—通知链
categories: [Linux]
tags: [Linux, Dev, Kernel,]
description: ""
---

&emsp;&emsp;&emsp;&emsp;内核的很多子系统之间具有很强的相互依赖关系，其中一个子系统发现的或产生的事件，
其他子系统可能都有兴趣。为了实现这种交互需求，Linux使用了通知链（Notification Chain）机制。

&emsp;&emsp;&emsp;&emsp;通知链这种机制只在内核的子系统之间使用，内核和用户空间之间的通知信息有
其他更多的机制。在网络子系统中通知链机制使用频繁，如管理路由表的子系统会根据网络设备子系统的通知来更新路由表。

&emsp;&emsp;&emsp;&emsp;通知链其实就是一份简单的函数列表，当给定事件发生时被执行，
每个函数让另一个子系统知道调用此函数的子系统内所发生的一个事件或者子系统侦测到的一个事件。
每条通知链都有被动端（被通知者）和主动端（通知者），也就是所谓的发布-订阅模型（publish-and-subscribe）：

* 被通知者（notified）就是接收某个事件的子系统，并提供回调函数；
* 通知者（notifier）就是侦测到某个事件后调用注册子系统的回调函数；

通知链允许每个子系统和其他子系统共享发生的事件，而无需知道就是是哪些子系统产生事件以及为何感兴趣。
通知链中的每个列表元素的类型是：

```
struct notifier_block { 
    int (*notifier_call)(struct notifier_block *, unsigned long, void *); //回调函数 
    struct notifier_block __rcu *next; //列表中下一个元素 
    int priority; //优先级，优先级高的先执行，但一般都没有指定优先级，因此也就是根据注册先后顺序调用 
};

notifier_block的实例常见名称有：xxx_chain、xxx_notifier_chain以及xxx_notifier_list。
```

当一个内核组件对给定通知链的事件感兴趣时，可以调用通用函数notifier\_chain\_register予以注册：

**注册**

通用函数：

    static int notifier_chain_register(struct notifier_block **nl,
                  struct notifier_block *n)

注册由读写锁同步保护的链：

    int blocking_notifier_chain_register(struct blocking_notifier_head *nh,
                     struct notifier_block *n)

注册由自旋锁保护的链：

    int atomic_notifier_chain_register(struct atomic_notifier_head *nh, 
                     struct notifier_block *n)

注册没有保护的链，需要用户自己保证访问同步：

    int raw_notifier_chain_register(struct raw_notifier_head *nh,
                      struct notifier_block *n)

注册inetaddr_chain通知链(读写锁保护)：

    int register_inetaddr_notifier(struct notifier_block *nb)

注册inet6addr_chain通知链(自旋锁保护)：

    int register_inet6addr_notifier(struct notifier_block *nb)

注册netdev_chain通知链(原始链)：

    int register_netdevice_notifier(struct notifier_block *nb)

**注销**

```
static int notifier_chain_unregister(struct notifier_block **nl, struct notifier_block *n)

int blocking_notifier_chain_register(struct blocking_notifier_head *nh, struct notifier_block *n)

int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh, struct notifier_block *n)

int raw_notifier_chain_unregister(struct raw_notifier_head *nh,  struct notifier_block *n)

int unregister_inetaddr_notifier(struct notifier_block *nb)

int unregister_inet6addr_notifier(struct notifier_block *nb)

int unregister_netdevice_notifier(struct notifier_block *nb)
```

**调用**

&emsp;&emsp;&emsp;&emsp;nl是通知链表头；val传递给回调函数的参数，表示发生事件类型；
v传递给回调函数的参数指针；nr\_to\_call将要调用的元素个数；nr_calls实际调用的回调元素个数。
最后一个回调函数返回返回值

```
nl是通知链表头；val传递给回调函数的参数，表示发生事件类型；v传递给回调函数的参数指针；
nr_to_call将要调用的元素个数；nr_calls实际调用的回调元素个数。最后一个回调函数返回返回值

static int __kprobes notifier_call_chain(struct notifier_block **nl, 
              unsigned long val, void *v, int nr_to_call,    int *nr_calls)
int atomic_notifier_call_chain(struct atomic_notifier_head *nh,
                  unsigned long val, void *v)
int blocking_notifier_call_chain(struct blocking_notifier_head *nh, 
                      unsigned long val, void *v)
int raw_notifier_call_chain(struct raw_notifier_head *nh,
                 unsigned long val, void *v)
```

不同保护机制的通知链的链表头结构：

```
struct atomic_notifier_head { 
    spinlock_t lock; 
    struct notifier_block __rcu *head; 
};
struct blocking_notifier_head { 
    struct rw_semaphore rwsem; 
    struct notifier_block __rcu *head; 
};
struct raw_notifier_head { 
    struct notifier_block __rcu *head; 
};
```

&emsp;&emsp;&emsp;&emsp;这三种通知链有专门的链表头，而通用函数操作的通知链的链表头就是一个元素。

```
static int __kprobes notifier_call_chain(struct notifier_block **nl, 
                    unsigned long val, void *v, 
                    int nr_to_call,    int *nr_calls) 
{ 
    int ret = NOTIFY_DONE; 
    struct notifier_block *nb, *next_nb;
    nb = rcu_dereference_raw(*nl);
    while (nb && nr_to_call) { 
        next_nb = rcu_dereference_raw(nb->next);
        ret = nb->notifier_call(nb, val, v);
        if (nr_calls) 
            (*nr_calls)++;
        if ((ret & NOTIFY_STOP_MASK) == NOTIFY_STOP_MASK) //检查回调函数返回值 
            break; 
        nb = next_nb; 
        nr_to_call--; 
    } 
    return ret; 
}
```

回调函数返回值：

```
#define NOTIFY_DONE        0x0000        /* Don't care */ 
#define NOTIFY_OK        0x0001        /* Suits me */ 
#define NOTIFY_STOP_MASK    0x8000        /* Don't call further */ 
#define NOTIFY_BAD        (NOTIFY_STOP_MASK|0x0002) 
                        /* Bad/Veto action */ 
/* 
* Clean way to return from the notifier and stop further calls. 
*/ 
#define NOTIFY_STOP        (NOTIFY_OK|NOTIFY_STOP_MASK)
```

&emsp;&emsp;&emsp;&emsp;内核定义了很多种不同的通知链，网络子系统中常用的两个通知链主要是：

* inetaddr\_chain：发送有关本地接口上的IPv4地址的插入、删除以及变更的通知信息。IPv6使用类似的inet6addr_chain；
* netdev_chain：发送有关网络设备注册状态的通知信息。


&emsp;&emsp;&emsp;&emsp;某些网络接口设备驱动程序也许会注册reboot_notifer_list链，当系统重启时，会得到通知。

在net/ipv4/fib_frontend.c中注册通知链的实例：

```
static struct notifier_block fib_inetaddr_notifier = { 
    .notifier_call = fib_inetaddr_event, 
};
static struct notifier_block fib_netdev_notifier = { 
    .notifier_call = fib_netdev_event, 
};
void __init ip_fib_init(void) 
{ 
                         …………………………
    register_netdevice_notifier(&fib_netdev_notifier); 
    register_inetaddr_notifier(&fib_inetaddr_notifier);
                         …………………………
}
```


{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [visualfan的ChinaUnix博客]
{% endhighlight %}

[深入理解Linux网络技术内幕—通知链 ](http://tekkamanninja.blog.chinaunix.net/uid-14518381-id-3413968.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
