---
layout: post
title:  嵌入式系统linux 记录内存panic
categories: [Linux]
tags: [GNU, Linux, Dev, Kernel, GDB, OpenWrt]
description: ""
---

## 简介
&emsp;&emsp;&emsp;&emsp;在内存发生panic时，需要把panic的日志保存下来。以方便日后进行分析。
目前有三种记录的方式： kdump; mtdoops; crashlog(这是openwrt特别的功能，正式linux内核中没有)
大家对kdump比较了解。它主要使用于x86系统。因为它使用占用大量内存和硬盘。
mtdoops和crashlog主要用于嵌入式的环境。也只是记录文本日志。

mtdoop功能在发生oops时,把msg区写入特定的mt分区。写入过程，不支持文件系统。直接二进制文本写。
它需要由mtd驱动的支持，就是mtd驱动支持mtd_panic_write。也就是原子式写入。不能被中断。一般flash不支持。
在mtdoops.c文件，并在标准内核了。

crashlog功能在发生oops时，把msg写入之前分配好的mem区。并加上magic。重启后，判断有magic后，
把上次的日志放到/sys/kernel/debug/crashlog.

因为在发生oops，是把日志写入内存区，这个很容易保存是原子操作。但它不在标准内核中，它是openwrt里有patch。
因为不同的架构，对于在上电来保存一块内存的方式也不太一样。要使用它时，需要程序员增加几行代码
给这个功能事先分配一个物理地址固定（明确）的内存区。并且这个区域在热重启时，不被uboot等使用。

## mtdoops
&emsp;&emsp;&emsp;&emsp;指定写入哪个分区（分区名或是分区号），可写入的size。
使用此函数 kmsg_dump_register注册一个回调函数，当发生OOPS时，回调一下cxt->dump.dump。
cxt->dump.dump函数则把读取kmsg区的内容，直接调用mtd->write的驱动进行写操作。

## carshlog
&emsp;&emsp;&emsp;&emsp;在内核中有一个叫crashlog的东东，它完成如下操作:

1. 在linux内核启动时，保留一64K内存。用于记录panic日志。
2. 使用kmsg_dump_register，注册一个回调函数，当发生panic,oops时，把日志记到保留内存。
3. linux内核上电后，把保留内存的内容写入文件。 它是输出在/sys/kernel/debug/crashlog中。

## kmsg_dump机制
&emsp;&emsp;&emsp;&emsp;以上两种方案都使用到了kmsg_dump的注册机制。
注册很简单，就是把一个全局变量结构挂到一个全局list中。
kmsg_dump是oops时进入kmsg_dump的入口。由panic,die,oops_exit等函数调用。它会一一调用回调函数。
每一个回调函数都会用到kmsg_dump_get_buffer。我没有看懂这个函数，从注释来看，
它先是计算dump还有多少空间，然后把kmsg中最后的一部分写进去。




{% highlight ruby %}
文档信息
--------------
* 作者:
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

**参考**

[嵌入式系统linux 记录内存panic](https://blog.csdn.net/wbd880419/article/details/70241130)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
