---
layout: post
title:  Linux Kernel中断机制2——中断分类
categories: [Linux]
tags: [Linux, Interrupt, C, Kernel, 硬件]
description: ""
---

&emsp;&emsp;&emsp;&emsp;在《Linux Kernel中断机制1——中断概念》一节中描述了中断的基本概念，
其中professional linux kernel architecture中提到中断可以有异常(exception)和错误(error)产生，
本节研究中断的分类和硬件相关的部分。本节主要来源于Understanding the Linux Kernel 3rd.


## 一、同步中断和异步中断
&emsp;&emsp;&emsp;&emsp;同步中断和异步中断的概念经常被提到(understanding the linux kernel 3rd)：

同步中断是当指令执行时由 CPU 控制单元产生，之所以称为同步，
是因为只有在一条指令执行完毕后 CPU 才会发出中断，而不是发生在代码指令执行期间，比如系统调用。

异步中断是指由其他硬件设备依照 CPU 时钟信号随机产生，即意味着中断能够在指令之间发生，例如键盘中断。

&emsp;&emsp;&emsp;&emsp;Intel微处理器手册称同步中断为“异常(Exception)”,称异步中断为“中断(Interrupt)”，
而平时所说的中断，两者都包含。

## 二、中断和异常
&emsp;&emsp;&emsp;&emsp;Intel手册中Interrupt（实际上就是我们平时说的异步中断）又可以分为
可屏蔽中断（maskable interrupt）和不可屏蔽中断(unmaskable interrupt)。可屏蔽中断通常由外部IO设备发出，
处理器根据该中断是否设置屏蔽位。中断的执行逻辑如下图所示：

![T1](/images/kernel/irq/interrupt1.png)

**可屏蔽中断（Maskable interrupts）**

&emsp;&emsp;&emsp;&emsp;所有的I/O设备产生的中断请求都是可屏蔽中断，可屏蔽中断有两种状态，
masked和unmasked;只要一个中断还处于maked状态，控制单元将忽略它发出的中断请求。

**不可屏蔽中断（Nonmaskable interrupts）**

&emsp;&emsp;&emsp;&emsp;只有极少数的关键事件，像硬件错误等，会发出不可屏蔽中断，不可屏蔽总是被CPU识别

而异常又分为故障（fault）、陷阱(trap)和终止(abort)三类：

&emsp;&emsp;&emsp;&emsp;处理器侦测异常（Processor-detected exceptions），
当CPU正在执行某条指令时侦测到异常状况时，产生的异常，这些异常又根据EIP的值分为3类，
CPU的控制单元产生异常时EIP的值被保存在内核栈中

* 故障（Faults）

能被纠正，一旦发生，程序允许在不影响连贯性的情况下重启.EIP中保存的值是引起故障的指令，
因此当异常处理程序终止时，该指令可以被重新执行。

![T2](/images/kernel/irq/fault.png)

* 陷阱（Traps）

在陷阱指令执行后立即报告；在内核把控制权返回给程序后，程序还可以不影响连贯性的继续执行。
EIP中保存的值是陷阱指令后应该执行的指令的地址。只有在没有必要重复执行已经终止的指令时，
才出发陷阱。陷阱的主要功能是调试，在这种情况下，中断信号的作用是通知debugger
一条特殊的指令已经被执行（例如，到断点位置了）。一旦用户检查过debugger提供的数据，
她很可能会继续执行程序的下一条指令。

![T2](/images/kernel/irq/traps.png)

* 终止（Aborts）

严重错误发生了，导致控制单元无法将引起异常的指令地址存入EIP，Abort就是被用于报告这些错误的，
例如硬件故障和系统表中非法或不一致的值。控制单元发送的中断信号是紧急信号，用来把控制权切换到相应
的异常终止处理程序，该异常终止处理程序除了强制受影响的进程终止，别无他法

![T3](/images/kernel/irq/abort.png)

编程异常（Programmed exceptions），由编程者请求发出，通常使用int或者int0/int3指令；
into(检查溢出)和bound（检查地址边界）在他们检测的结果不为真的时候也会引发编程异常。
编程异常被控制单元当作陷阱处理，它们经常称作software interrupts。这类异常通常有两类用处：
实现系统调用和通知debugger一个特殊的时间发生。

## 三、总结

&emsp;&emsp;&emsp;&emsp;中断，Interrupt在x86架构下，根据发生中断的时刻处理器上是否执行完毕当前指令，
分为同步中断(synchronous interrupt)和异步中断(asynchronous interrupt),Intel通常将同步中断成为异常（Exception），
而将异步中断称为中断（Interrupt）。同步中断多为CPU上正在执行的指令产生直接产生，而异步中断来自于I/O设备。如下图所示

![T3](/images/kernel/irq/interrupt.jpg)

&emsp;&emsp;&emsp;&emsp;Intel的“异常”（Exception）又根据严重程度和EIP存储哪条指令分为三类：
故障（Fault）、陷阱（Trap）和终止（Abort），而这三类异常的处理方法不同：

类别 | 原因 | 同步/异步 | 返回的行为
----- | ----- | ----- | -----  
陷阱（Trap）|	有意的异常	| 同步	| 总是返回到下一条指令 
终止（Abort）|	不可恢复的错误	| 同步	| 不会返回
故障（Fault）|	潜在可恢复的错误 |	同步 |	返回到当前指令
中断（Interrupt）|	来自I/O设备的电信号	| 异步	| 总是返回到下一条指令



{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [Rock3的Linux博客]
{% endhighlight %}

[Rock3的Linux博客](http://rock3.info/)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
