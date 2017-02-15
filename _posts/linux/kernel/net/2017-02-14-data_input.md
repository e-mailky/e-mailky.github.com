---
layout: post
title:  linux 内核网络--数据接收流程图 
categories: [网络]
tags: [Linux, Kernel, TCP/IP]
description: ""
---

## 4.3 数据接收流程图

![约翰.格鲁伯](/images/kernel/receive.gif)

&emsp;&emsp;&emsp;&emsp;各层主要函数以及位置功能说明：

1. sock\_read:初始化msghdr{}的结构类型变量msg，并且将需要接收的数据存放的地址传给msg.msg\_iov->iov_base.(net/socket.c)
2. sock\_recvmsg: 调用函数指针sock->ops->recvmsg()完成在INET Socket层的数据接收过程.
   其中sock->ops被初始化为inet\_stream\_ops,其成员recvmsg对应的函数实现为inet_recvmsg()函数. (net/socket.c)
3. sys\_recv()/sys\_recvfrom():分别对应着面向连接和面向无连接的协议两种情况.(net/socket.c)
4. inet\_recvmsg:调用sk->prot->recvmsg函数完成数据接收,这个函数对于tcp协议便是tcp\_recvmsg (net/ipv4/af_net.c)
5. tcp\_recvmsg:从网络协议栈接收数据的动作,自上而下的触发动作一直到这个函数为止,
   出现了一次等待的过程.函数tcp\_recvmsg可能会被动地等待在sk的接收数据队列上,也就是说,
   系统中肯定有其他地方会去修改这个队列使得tcp_recvmsg可以进行下去.入口参数sk是这个网络连接
   对应的sock{}指针,msg用于存放接收到的数据.接收数据的时候会去遍历接收队列中的数据,找到序列号合适的.
   但读取队列为空时tcp\_recvmsg就会调用tcp\_v4\_do\_rcv使用backlog队列填充接收队列.
6. tcp\_v4\_rcv:tcp\_v4\_rcv被ip\_local\_deliver函数调用,是从IP层协议向INET Socket层提交的"数据到"请求,
   入口参数skb存放接收到的数据,len是接收的数据的长度,这个函数首先移动skb->data指针,让它指向tcp头,
   然后更新tcp层的一些数据统计,然后进行tcp的一些值的校验.再从INET Socket层中已经建立的sock{}结构变量中
   查找正在等待当前到达数据的哪一项.可能这个sock{}结构已经建立,或者还处于监听端口、等待数据连接的状态。
   返回的sock结构指针存放在sk中。然后根据其他进程对sk的操作情况,将skb发送到合适的位置.调用如下:

    TCP包接收器(tcp\_v4\_rcv)将TCP包投递到目的套接字进行接收处理. 当套接字正被用户锁定,TCP包将暂时
    排入该套接字的后备队列(sk\_add\_backlog).这时如果某一用户线程企图锁定该套接字(lock\_sock),
    该线程被排入套接字的后备处理等待队列(sk->lock.wq).当用户释放上锁的套接字时(release\_sock,
    在tcp\_recvmsg中调用),后备队列中的TCP包被立即注入TCP包处理器(tcp\_v4\_do\_rcv)进行处理,然后
    唤醒等待队列中最先的一个用户来获得其锁定权. 如果套接字未被上锁,当用户正在读取该套接字时, 
    TCP包将被排入套接字的预备队列(tcp\_prequeue),将其传递到该用户线程上下文中进行处理.
    如果添加到sk->prequeue不成功,便可以添加到 sk->receive\_queue队列中(用户线程可以登记到预备队列,
    当预备队列中出现第一个包时就唤醒等待线程.)   (net/tcp_ipv4.c)
7. ip\_rcv、ip\_rcv\_finish:从以太网接收数据，放到skb里，作ip层的一些数据及选项检查，
   调用ip\_route\_input()做路由处理,判断是进行ip转发还是将数据传递到高一层的协议.
   调用skb->dst->input函数指针,这个指针的实现可能有多种情况,如果路由得到的结果说明这个
   数据包应该转发到其他主机,这里的input便是ip\_forward;如果数据包是给本机的,那么input指针初始化为
   ip\_local\_deliver函数.(net/ipv4/ip_input.c)
8. ip\_local\_deliver、ip\_local\_deliver\_finish:入口参数skb存放需要传送到上层协议的数据,
   从ip头中获取是否已经分拆的信息,如果已经分拆,则调用函数ip\_defrag将数据包重组。然后通过调用
   ip\_prot->handler指针调用tcp\_v4\_rcv(tcp)。ip\_prot是inet\_protocol结构指针,是用来ip层登记协议的，
   比如由udp,tcp,icmp等协议。 (net/ipv4/ip_input.)




{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

[linux 内核网络，数据接收流程图](http://blog.chinaunix.net/uid-27057175-id-5181382.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
