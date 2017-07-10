---
layout: post
title:  GDB + CoreDump 调试记录 
categories: [Linux]
tags: [GNU, Linux, Dev, C]
description: ""
---


&emsp;&emsp;&emsp;&emsp;今天遇到个问题，某人的代码有断错误，导致我的工作无法展开，
抱怨的就不多说了，正好让我解决了一个gdb的操作问题！ 现在说下gdb+coredump的调试流程

&emsp;&emsp;&emsp;&emsp;在实机上先打开产生core文件的设置，ulimit -c unlimited ，
这将使程序在遇到断错误的时候保留下当时的堆栈信息，而这个core文件的大小没有进行限制，
当然，还可以更改core文件的产生路径，具体可以google下。 之后运行断错误程序，获取产生的core文件。

&emsp;&emsp;&emsp;&emsp;运用平台指定的gdb程序  调用arm-linux-gdb -c core

&emsp;&emsp;&emsp;&emsp; bt下

&emsp;&emsp;&emsp;&emsp;这时会看到#0  0x2ae1cc9e in ?? ()之类的堆栈信息，因为没有指定带调试信息的调试程序，
所以无法读书函数的符号表，所以显示不出当时的堆栈运行到那个函数。

&emsp;&emsp;&emsp;&emsp;输入file exec ，显示出
Reading symbols from /media/disk/work/ngi/work_space/ngi/out/target/product/high/system/usr/bin/testmode...done
说明加载进了符号表，在bt下就应该能看到堆栈的函数名了

```
        (gdb) bt
#0  0x2ae1cc9e in ?? ()
#1  0x2ac02600 in ?? ()
#2  0x2ac02600 in ?? ()
```

&emsp;&emsp;&emsp;&emsp;奇怪 为什么还是没呢？  原来这时候程序是运行到动态库中去了，那既然如此当然需要把动态库中的符号表信息读入gdb咯！

&emsp;&emsp;&emsp;&emsp;事实是否如我猜测呢？ 输入info share

```
(gdb) info share
warning: .dynamic section for "/media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/libm.so.6" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/libc.so.6" is not at the expected address (wrong library or version mismatch?)
warning: Could not load shared library symbols for 60 libraries, e.g. /lib/libncore.so.
Use the "info sharedlibrary" command to see the complete listing.
Do you need "set solib-search-path" or "set sysroot"?
From        To          Syms Read   Shared Object Library
                        No          /lib/libncore.so
                        No          /lib/libicuuc.so
                        No          /lib/liblog.so
                        No          /lib/libncutility.so
                        No          /lib/libncplatform.so
                        No          /lib/libncdevice.so
                        No          /lib/libnhcommon.so
                        No          /lib/libnhdiag.so
                        No          /lib/libnhucom.so
                        No          /lib/libnhkey.so
                        No          /lib/libnhsysctrl.so
                        No          /lib/libzeromq.so
                        No          /lib/libbinder.so
                        No          /lib/libnceventsys.so
                        No          /usr/lib/libncgstreamer.so
                        No          /lib/libGLESv1_CM.so
                        No          /lib/libEGL.so
                        No          /lib/libGAL.so
                        No          /usr/lib/libncgraphics.so
                        No          /usr/lib/libGraphicsCore.so
                        No          /usr/lib/liboverlay.so
                        No          /usr/lib/libtuner.so
                        No          /usr/lib/libsourceusbapi.so
                        No          /usr/lib/libsourceusb.so
                        No          /usr/lib/libnhbtproxy.so
                        No          /usr/lib/libgmock.so
                        No          /lib/libncstorage.so
                        No          /lib/libnhvehicle.so
                        No          /lib/libnhcanbus.so
0x2ae83e80  0x2ae91fdc  No          /media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/libpthread.so.0
0x2adb56a0  0x2adb8e58  No          /media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/librt.so.1
0x2adc38fc  0x2adc4614  No          /media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/libdl.so.2
                        No          /lib/libstdc++.so.6
0x2b0f61a8  0x2b15a9b8  No          /media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/libm.so.6
                        No          /lib/libgcc_s.so.1
0x2b179440  0x2b262044  No          /media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/libc.so.6
0x2aaef780  0x2ab07a20  No          /media/disk/work/ngi/work_space/ngi/prebuilts/gcc-4.6.2-glibc-2.13-linaro-multilib-2011.12/fsl-linaro-toolchain/arm-fsl-linux-gnueabi/multi-libs/lib/ld-linux.so.3
                        No          /lib/libutils.so
```

&emsp;&emsp;&emsp;&emsp;可以看到很多库的符号表都没有被读入，看来果然如此。

&emsp;&emsp;&emsp;&emsp;输入用set solib-search-path + 库的路径 指定所要读取的符号表动态库所在位置，
结果会看到很多done之类的信息，说明加载成功，  
现在bt下就能看到堆栈的详细信息了。

```
(gdb) bt
#0  nutshell::SourceUSB_ScannerThread::setScannerState (this=0x0, 
    state=nutshell::SourceUSB_ScannerThread::SCANNERSTATE_START)
    at system/MediaLibrary/SourceUSB/src/Thread_Scanner/SourceUSB_ScannerThread.cpp:138
#1  0x2ae1ce1a in nutshell::SourceUSB_Scanner_EventHandler::onHandle (this=0xaf4a0, cEvt=...)
    at system/MediaLibrary/SourceUSB/src/Thread_Scanner/SourceUSB_Scanner_EventHandler.cpp:50
#2  0x2ae17018 in nutshell::SourceUSB_EventHandlerBase::onHandle (this=, cEvt=)
    at system/MediaLibrary/SourceUSB/src/include/base/SourceUSB_EventHandlerBase.h:35
#3  0x2abe85b6 in nutshell::PI_EventHandlerThreadImp::run (this=0xaef88)
    at system/MediaLibrary/common/TsmThreadLib/src/PI_EventHandlerThreadImp.cpp:93
#4  0x2ac04e34 in ?? ()
#5  0x2ac04e34 in ?? ()
```


{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
{% endhighlight %}

参考
[GDB + CoreDump 调试记录 ](http://blog.chinaunix.net/uid-24922718-id-3489839.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
