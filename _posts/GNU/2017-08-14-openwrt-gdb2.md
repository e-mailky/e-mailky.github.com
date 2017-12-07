---
layout: post
title:  Openwrt之gdb调试
categories: [Linux]
tags: [GNU, Linux, Dev, C]
description: ""
---

## 第一种情况：应用层API(用户态)【coredump方法】
&emsp;&emsp;&emsp;&emsp; 路由器： 在路由器/tmp运行命令，使其段错误的时候生成core文件；

    $ ulimit  -c  unlimited; 

把路由器的/tmp/core文件复制到 电脑的openwrt源码编译目录[/home/luo/op]（虚拟机/远程服务器）开始gdb调试：

    $ cd /home/luo/op;
    $  ./staging_dir/toolchain-mipsel_24kec+dsp_gcc-4.8-linaro_uClibc-1.0.x/bin/mipsel-openwrt-linux-uclibc-gdb build_dir/target-mipsel_24kec+dsp_uClibc-1.0.x/xxx/可执行文件     /home/luo/op/core
    $ set sysroot  /home/luo/op/staging_dir/target-mipsel_24kec+dsp_uClibc-1.0.x/root-ralink/
    $ bt
    $ list
    $ frame 1


## 第二种情况：编译进内核,内核奔溃调试(内核态) [看符号表]

1. 看路由器串口信息，在哪里奔溃

    [   1.452000] Call Trace:
    [   1.452000] [<802735e8>]  flash_otp_init+0x8c/0x1e8
    [   1.452000] [<802735e8>]  xxxx
    [   1.452000] [<802735e8>]  xxxx

2. 进入openwrt编译目录调试

    $ cd build_dir/target-mipsel_24kec+dsp_uClibc-1.0.x/linux-ralink_mt7620/linux-3.14.79；
    $ gdb  vmlinux;                # 使用X86/64的gdb，而不是mipsel的 
    $ list *(flash_otp_init+0x8c); # 这个可以定位到哪一个.c文件的哪一行；


## 第三种情况：编译为模块,内核奔溃调试(内核态)  [看符号表]

.ko模块调试；需要做一个准备动作；

```
make menuconfig
Global build settings  --->
Compile the kernel with profiling enabled; [CONFIG_KERNEL_PROFILING]

$ make V=s -j8要重新编译内核，烧写固件；
```


1. 看路由器串口信息，在哪里奔溃

```
[   1.452000] Call Trace:
[   1.452000] [<802735e8>]  xxx_trig_activate+0x2c/0x1e8
[   1.452000] [<802735e8>]  xxxx
[   1.452000] [<802735e8>]  xxxx
```

2. 进入openwrt编译目录调试

    $ cd  build_dir/target-mipsel_24kec+dsp_uClibc-1.0.x/linux-ralink_mt7620/
    $ gdb ledtrig-xxx/ledtrig-xxx.ko;  使用X86/64的gdb，而不是mipsel的
    $ list *(xxx_trig_activate+0x2c); 这个可以定位到哪一个.c文件的哪一行；


{% highlight ruby %}
文档信息
--------------
* 作者:
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

参考文献: 
[Openwrt之gdb调试](http://www.openwrt.pro/post-291.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
