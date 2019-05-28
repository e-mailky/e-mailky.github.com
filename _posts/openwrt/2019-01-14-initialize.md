---
layout: post
title:  openwrt 内核完成后的初始过程
categories: [其他]
tags: [其他]
description: ""
---

启动图解如下:

![T1](/images/linux/openwrt/20160706150126332.png)

linux内核启动完成后，执行的第一个程序中/etc/preinit。此时环境变量PREINIT为空，所以马上执行/sbin/init
/sbin/init是由procd/init.c编译而来。主要过程如下:

1. /sbin/kmodloader 加载内核模块
2. 执行一些early cmdline等
3. 执行preinit()函数,设置PREINIT=1, 再次fork了/etc/preinit。

/etc/preinit执行完成后，进程消失后，调用了回调函数spawn_procd
spawn_procd则execp("procd"),至此0号进程/sbin/init被1号进程/sbin/procd替换,成为真正的用户空间的父进程
主要过程如下:

1. 安装hostplug_preinit.json()
	* 打开watchdog
	* 开始hotplug
	* udevtrigger
2. 再去执行/etc/init.d/\*文件,启动各个服务。


**第二次执行 /etc/preinit的过程：**

preinit功能由几个脚本组成，主脚本是/etc/preinit，它会读取其它的脚本【其中hook_XX函数库在/lib/functions/preinit.sh。
其它功能性的脚本在/lib/preinit/\*】。它定义了一些函数挂到hook上.当运行时，这些hook们会启动函数按函数加入的顺序。

hook点如下：

* preinit_essential
* preinit_main
* failsafe
* initramfs
* preinit_mount_root

这些hook点说白了就是一个保存一些函数名+空格的字符串。如preinit_essentail的hook就是变量：preinit_essentail_hook
使用boot_hook_add把一些函数名的字符串加入相关变量中。示例：

    boot_hook_add preinit_main define_default_set_state  ## export -n preinit_main_hook=define_default_set_state

使用 boot_run_hook时，把从hook的变量中取出函数来并一一执行。
示例代码：

    boot_run_hook preinit_main ## export -n PI_RAN_define_default_set_state=1 export -n PI_RAN_define_default_set_state=1

**procd启动各服务**

* procd: - early -   //初始化看门狗。
* procd: - watchdog -
* procd: - ubus -
* procd: - init -

如上日志表示了procd的初始化过程。
procd有几个state。state_enter函数为状态机处理入口。

STATE_NONE -->STATE_EARLY -->STATE_UBUS-->STATE_INIT-->STATE_RUNNING

* STATE_NONE ：什么也不干。
* STATE_EARLY ：初始化看门狗等。
* STATE_UBUS：与ubusd建立socket.
* STATE_INIT:读取/etc/initab中的条目，为每一个条目建议一个action（其中有cb处理函数）。再依次执行respawn，askconsole，askfirst，sysinit的action.

执行完成sysinit后则进程了STAT_RUNNING状态。



{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
