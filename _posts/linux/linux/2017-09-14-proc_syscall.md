---
layout: post
title:  找出进程当前系统调用
categories: [Linux]
tags: [Linux, Dev, C, FS, 进程管理]
description: ""
---

&emsp;&emsp;&emsp;&emsp;当一个程序发生故障时，有时候想通过了解该进程正在执行的系统调用来排查问题。
通常可以用 strace 来跟踪。但是当进程已经处于 D 状态（uninterruptible sleep）时，strace 也帮不上忙。这时候可以通过

    cat /proc/<PID>/syscall

来获取当前的系统调用以及参数。

&emsp;&emsp;&emsp;&emsp;这里用最近排查的一个问题为例。碰到的问题是，
发现一台服务器在执行 pvcreate 创建物理卷的时候卡死，进程状态为 D

```
# ps aux|grep pvcreate
root      8443  0.0  0.0  27096  2152 ?        D    Apr04   0:00 pvcreate /dev/sddlmac
...
```

D 状态实际是在等待系统调用返回。那么来看看究竟在等待什么系统调用

    B0313010:~ # cat /proc/8443/syscall
    0 0x7 0x70f000 0x1000 0x0 0x7f33e1532e80 0x7f33e1532ed8 0x7fff3a6b8718 0x7f33e128cf00

&emsp;&emsp;&emsp;&emsp;第一个数字是系统调用号，后面是参数。不同的系统调用所需的参数个数不同。
这里的字段数是按最大参数数量来的，所以不一定每个参数字段都有价值。那么怎么知道系统调用号对应
哪个系统调用呢？在头文件 /usr/include/asm/unistd_64.h 中都有定义。也可以用个小脚本来快速查找：

```
#!/bin/bash
# usage: whichsyscall <syscall_nr>
nr="$1"
file="/usr/include/asm/unistd_64.h"
gawk '$1=="#define" && $3=="'$nr'" {sub("^__NR_","",$2);print $2}' "$file"
```

&emsp;&emsp;&emsp;&emsp;对于不同的系统调用的参数，可以通过 man 2 <系统调用名> 查阅。如 man 2 read。
对刚才那个例子来说，0 就对应了 read 调用。而 read 调用的第一个参数是文件描述符。

&emsp;&emsp;&emsp;&emsp;之后用 lsof 找到 7 对应的是什么文件

```
#  lsof -p 8443
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF     NODE NAME
......
pvcreate 8443 root    5u   CHR 10,236      0t0    19499 /dev/mapper/control
pvcreate 8443 root    6u   BLK  253,1   0t8192 36340797 /dev/dm-1
pvcreate 8443 root    7u   BLK  253,5      0t0 35667968 /dev/dm-5
```


&emsp;&emsp;&emsp;&emsp;结果发现是个 device mapper 的设备文件。最后顺藤摸瓜，
发现这个文件是 multipathd 创建的。而系统应当使用的是存储厂商提供的多路径软件。
问题是由于同时开启了 multipathd 造成冲突导致的。

&emsp;&emsp;&emsp;&emsp;/proc/<PID>/syscall 对排查 D 状态进程很有用。不过在 2.6.18 内核上并不支持，
具体从哪个内核版本开始有这个功能，还没查到。不过至少从在 2.6.32 以上版本都是支持的。



{% highlight ruby %}
文档信息
--------------
* 作者:
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

[神仙的仙居](http://xiezhenye.com/tag/syscall)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
