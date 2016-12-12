---
layout: post
title:  mips地址空间
categories: [Linux]
tags: [Linux, Kernel]
description: ""
---

*在32位MIPS体系结构下*，最多可寻址4GB地址空间。这4GB空间的分配是怎样的呢？让我们看下面这张图：
![Figure 1 MIPS Logical Addressing Space](/images/kernel/32位mips地址空间.png)

Figure 1 MIPS Logical Addressing Space

&emsp;&emsp;&emsp;&emsp;上图是MIPS处理器的逻辑寻址空间分布图。我们看到，2GB以下的地址空间，
也就是从0x00000000到0x7FFFFFFF的这一段空间，为User Space，可以在User Mode下访问，
当然，在Kernel Mode下也是可以访问的。程序在访问User Space的内存时，会通过MMU的TLB，
映射到实际的物理地址上。也就是说，这一段逻辑地址空间和物理地址空间的对应关系，
是由MMU中的TLB表项决定的。

&emsp;&emsp;&emsp;&emsp;从0x80000000到0xFFFFFFFF的一段为Kernel Space，
仅限于Kernel Mode访问。如果在User Mode下试图访问这一段内存，将会引发系统的一个Exception。
MIPS的Kernel Space又可以划分为三部分。首先是通过MMU映射到物理地址的1GB空间，
地址范围从0xC0000000到0xFFFFFFFF。这1GB空间可以用来访问实际的DRAM内存，可以为操作系统的内核所用。

&emsp;&emsp;&emsp;&emsp;MIPS的Kernel Space中，还有两段特殊的地址空间，
分别是从0x80000000到0x9FFFFFFF的Kernel Space Unmapped Uncached和
0xA0000000到0xBFFFFFFF的Kernel Space Unmapped Cached。之所以说它们特殊，
是因为这两段逻辑地址到物理地址的映射关系是硬件直接确定的，不通过MMU，而且两段实际上是重叠的，
均对应 0x00000000到0x20000000的物理地址。那么，为什么一段同样的物理地址有两个逻辑地址对应呢？
它们的区别又在哪里呢？

&emsp;&emsp;&emsp;&emsp;原来，这是MIPS的设计特色之一。软件在访问Kernel Space Unmapped Uncached
这段地址空间的时候，不经过MIPS的Cache。这样，虽然速度会比较慢，但是，对于硬件I/O寄存器来说，
就不存在所谓的Cache 一致性问题。Cache一致性问题，是指硬件将某个地址的内容跳过软件而改变了，
Cache中的内容尚未同步。这样，如果软件读取该地址，有可能从 Cache中获取到错误的内容。
将硬件I/O寄存器设定在这段地址空间，就可以避免Cache一致性带来的问题。MIPS的程序上电启动地址 0xBFC00000，
也落在这段地址空间内。——上电时，MMU和Cache均未初始化，因此，只有这段地址空间可以正常读取并处理。

&emsp;&emsp;&emsp;&emsp;另一段特殊的地址Kernel Space Unapped Cached，与前者类似，
直接映射到0x00000000到0x20000000，与Kernel Space Unmapped Uncached重叠。因为通过Cache，
这段地址空间的访问速度比前者为快。一般地，这段内存空间用于内核代码段，或者内核中的堆栈。

&emsp;&emsp;&emsp;&emsp;显然地，当工程师们换算Kernel Space中的这两段的物理地址和逻辑地址时，
只需要改变地址的高3bit就可以了。

&emsp;&emsp;&emsp;&emsp;那么，什么时候需要使用物理地址，什么时候需要使用逻辑地址呢？我们知道，
逻辑地址是程序中访问的内存地址，譬如，下面的这条指令：

    lw a0, 128(t2)

&emsp;&emsp;&emsp;&emsp;这条指令的内容是从t2寄存器内的地址 + 偏移128字节处，读取一个word (4Byte)到寄存器a0内。
如果t2的值为0x88200100，则最终访问的物理地址为0x88200180。而物理地址，从工程上可以理解为，
将逻辑分析仪连接到内存总线(Memory Bus)上，逻辑分析仪指示的地址，就是物理地址。假如，在上一个例子中，
我们把逻辑分析仪接到处理器的前端内存总线，我们就可以看到，执行该指令时，系 统访问的物理地址为0x08200180。
物理地址和逻辑地址的换算，不仅限于电子工程师在设计硬件线路时需要。在内核工程师编写支持DMA的外部设备驱 动时，
需要将向操作系统申请到的数据缓冲区地址（当然，这是一个逻辑地址）转换为物理地址，并“告诉”相关外设。
这样，外设就可以在收到数据后，使用 DMA模式储存在系统的主存中，并向系统发起一个IRQ。
操作系统在IRQ的handler中，从外设的相应IO寄存器读取到这段内存的地址（当然，是物 理地址）并转换为逻辑地址并处理之。
这个过程中，如果没有正确使用和分辨物理地址和逻辑地址，驱动程序便会导致内核的一个panic错误。

&emsp;&emsp;&emsp;&emsp;物理地址到逻辑地址的映射关系是由什么决定的呢？除了上面提到的两段Unmapped的地址空间，
其余都是由TLB确定的，由MMU来执行。这是后话。

&emsp;&emsp;&emsp;&emsp;Kuseg：0×00000000～0×7FFFFFFF（2G）。这些地址是用户态可用的地址。
在有MMU的机器里，这些地址将一概被转换。除非MMU已经设置好，否则不应该使用这些地址；对于没有MMU的处理器，
这些地址的行为与具体处理器有关。 如果想要你的代码能够移植到无MMU的处理器上，或者能够在无MMU的处理器间移植，
应避免使用这块区域。

&emsp;&emsp;&emsp;&emsp;Kseg0：0×80000000～0×9FFFFFFF（512M）。只要把最高位清零，
这些地址就会转换成物理地址，映射到连续的 低端512M的物理空间上。这段区域的地址几乎总是要通过
高速缓存来存取（write-back，或write-through模式），所以在高速缓存初 始化之前，不能使用。
因为这种转换极为简单，且通过Cache访问，因而常称为这段地址为Unmapped Cached。这个区域在无MMU的系统
中用来存放大多数程序和数据；在有MMU的系统中用来存放操作系统核心。

&emsp;&emsp;&emsp;&emsp;KSeg1：0xA0000000～0xBFFFFFFF（512M）。这些地址通过把最高三位清零，
映射到连续的第算512M的物理地址，即 Kseg1和Kseg0对应同一片物理内存。但是与通过Kseg0的访问不同，
通过Kseg1访问的话，不经过高速缓存。 Kseg1是唯一的在系统加电/复位时，能正常访问的地址空间，
这也是为什么复位入口点（0xBFC00000）放在这个区域的原因，即对应复位物理地址 为0×1FC00000。
因而，一般情况下利用该区域存储Bootlooder；大多数人也把该区域用作IO寄存器（这样可以保证访问时，
是直接访问 IO，而不是访问cache中内容），因而建议IO相关内容也要放在物理地址的512M空间内。

&emsp;&emsp;&emsp;&emsp;KSeg2：0xC0000000～0xFFFFFFFF（1G）。这块区域只能在核心态下使用，
并且要经过MMU的转换，因而在MMU设置好之前，不要存取该区域。除非你在写一个真正的操作系统，否则没有理由使用Kseg2。

&emsp;&emsp;&emsp;&emsp;MIPS CPU运行时有三种状态：
用户模式（User Mode）；核心模式（Kernel Mode）；管理模式（Supervisor Mode）。
其中管理模式不常用。用户模式下，CPU只能访问KUSeg；当需要访问KSeg0、Kseg1和Kseg2时，必须使用核心模式或管理模 式。



--------------
{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [Warp Drive 的时间隧道][Markdown简明教程]
{% endhighlight %}

[mips地址空间](http://blog.chinaunix.net/uid-22547469-id-5056345.html)

[mips地址空间](http://joe.is-programmer.com/posts/17481.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
