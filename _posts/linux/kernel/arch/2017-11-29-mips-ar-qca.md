---
layout: post
title:  AR/QCA SPI 启动原理和 ART 地址定位原理
categories: [Linux]
tags: [Linux, Dev, C, Kernel, 内存管理]
description: ""
---

&emsp;&emsp;&emsp;&emsp;本贴主要讲解 Bootloader 是如何在使用 SPI Flash 的 AR/QCA 的芯片上启动的，
以及 OpenWrt 代码 ar71xx 的 mach 文件中类似于 u8 \*art = KSEG1ADDR(0x1fff0000) 中 0x1fff0000 是如何得来的。
楼主之前在 U-Boot 编译教程中进行过简单的描述，但是因为实在是太简略了，所以打算写一个详细版的。

下面楼主将按照必要的顺序依次讲解。

## 1. CPU 地址、总线地址、映射

对于 MIPS CPU 来说，只有一种地址，即 CPU 地址。32 位 CPU 的寻址范围称为它的地址空间，也就是 4GB，
从 0x00000000 到 0xffffffff。而总线，则是用来连接外设的，外设的寄存器、RAM、ROM等，都要通过总线来访问。
总线也有自己的地址空间。

CPU 地址跟总线地址是相互独立的。因此，要让 CPU 能够访问到外设，就必须要让 CPU 地址跟总线地址产生某种关联，这就是映射。
这就好比数学里面的映射。MIPS CPU 上的映射是从 CPU 上一段地址空间到总线上一段地址空间的一一映射。

有了这样的关联后，就很容易通过对 CPU 地址的操作来变成对总线上外设的操作了。

## 2. AR/QCA 的总线地址布局

以 AR9344 为例，如图：

![T1](/images/kernel/92a31490322453.png)

可以看到 AR/QCA 的 CPU 使用了总线地址的 0x00000000 ~ 0x1fffffff 共 512MB 的地址空间。
注意到内存也是通过总线来访问的。由图可知 AR/QCA 只有最大 256MB 的内存寻址能力。


## 3. MIPS 的内存布局

MIPS32 (即 32 位) 的内存模型都是一致的，如图：

![T2](/images/kernel/39a81490322434.png)

(图片来自 MIPS32 74K Processor Core Family Software User’s Manual)

由于 Bootloader 跟 Linux 内核都运行在 Kernel Mode 下，所以这里也只就 Kernel Mode 进行讨论。

由图可以看到，Kernel Mode 下，CPU 的地址空间被分为了5段： kuseg kseg0 kseg1 kseg2 kseg3。
kuseg 空间为 2GB，是用于用户态程序访问用的，kseg2 kseg3 是用作内核的内存分页用的。
这里只重点讨论 kseg0 和 kseg1。kseg0 和 kseg1 都占用 512MB 的地址空间。

由 MIPS 手册可知，kseg0 和 kseg1 都映射到了总线地址上的相同区域，也就是总线地址的 0x00000000 ~ 0x1fffffff 共 512MB 的空间。
这个映射是固定映射，也就是不会经过 MMU 的转换，访问 kseg0 跟 kseg1 都会直接被映射到总线上 0x00000000 ~ 0x1fffffff 的对应位置。

由于 AR/QCA 使用的总线地址空间也是这个范围，因此通过 kseg0 或 kseg1 就都能访问到整个总线地址空间。
如图：

![T3](/images/kernel/39a81490322434-2.png)

从这里可以看到，kseg0 的范围是 0x80000000 ~0x9fffffff，kseg1 的范围是 0xa0000000 ~ 0xbfffffff。
又，总线地址空间范围是 0x00000000 ~ 0x1fffffff，那么：

1. 从总线地址映射到 kseg0 的方法是：在保证总线地址有效 (即总线地址在 0x00000000 ~ 0x1fffffff 内) 的情况下，
   将总线地址加上 kseg0 的起始地址，即：
   kseg0(addr) = 0x80000000 + addr
2. 从总线地址映射到 kseg1 的方法与从总线地址映射到 kseg0 的方法类似，即：
   kseg1(addr) = 0xa0000000 + addr
3. 从 kseg0 和 kseg1 映射到总线的方法是：将 CPU 地址除以 kseg0 或 kseg1 段长度，取余，得到的就是总线地址，即：
   bus(addr) = addr % 0x20000000

实际上，在 Linux 内核中，arch/mips/include/asm/addrspace.h 提供了相应的宏来进行上述的地址转换，简化后如下：

```
#define virt_to_bus(_virt) ((_virt) & 0x1fffffff)
#define KSEG0ADDR(_addr) ((_addr & 0x1fffffff) | 0x80000000)
#define KSEG1ADDR(_addr) ((_addr & 0x1fffffff) | 0xa0000000)
```

virt_to_bus 即为总 CPU 地址转换到总线地址， 与 0x1fffffff (即低 512MB 的掩码) 进行按位与操作，
相当于除以 0x20000000 取余，即舍弃掉 512MB 之上的部分，只剩下低 512MB 的部分；
KSEG0ADDR 与 KSEG1ADDR 都先保证输入地址是有效的，再进行转换，这里的按位或运算，在结果上等同于加法运算

虽然 kseg0 跟 kseg1 都映射在总线相同的地址空间上，但是，它们的作用却并不相同：

*kseg0 经过了缓存，kseg1 没有经过缓存。*

kseg0 经过了缓存，也就是说从这个段读取出来的数据，可能是缓存过的，
      跟总线上实际的数据可能不同；向其写入的数据，可能会被延迟写入总线。
kseg1 则没有经过缓存，对这个段的任何读写操作都将立即反映在总线上。

*所以：*
*kseg0 主要用于需要加速的内存访问和 ROM 访问*
*kseg1 主要用于设备寄存器的访问*

## 4. MIPS 上的启动地址

MIPS 的 CPU 在复位后，会从 CPU 地址的 0xbfc00000 开始执行，也就是总线地址的 0x1fc00000。
可以看出 CPU 是在 kseg1 段上开始执行的，这是因为在 CPU 复位后，缓存还没用初始化，可能并不能使用。
在不能保证 kseg0 段一定能使用的情况下，那么肯定只有从 kseg1 段开始执行了。

## 5. SPI Flash 和映射

这里的 SPI Flash 特指 SPI 接口的 NOR Flash （当然也有 SPI NAND Flash）。

NOR Flash 有个特点，即给出一个确切的地址，那么它就能连续输出从这个地址开始的数据。这个特点跟 DRAM 类似，因此它可以被用作启动设备。
这个特性被称为 XIP （eXecute-In-Place)。

为了实现这个特性，就需要 CPU 能够通过 CPU 地址空间访问到 Flash 上的数据。
由于 Flash 是一个外设，因此对它的访问是通过总线来进行的。

这里，又用到了映射：
Flash 有自己的地址空间，即存储数据的地址。
由于 Flash 是外设，因此它的地址空间会被映射在总线上面。这又是一个映射，只不过是从总线到 Flash 的映射。

那么，通过 CPU 访问 Flash，实际上经过了两次映射：第一次是 CPU 地址到总线地址的映射，第二次是总线地址到 Flash 地址的映射。
这样，CPU 就能够直接读取 Flash 的数据了。

## 6. AR/QCA 的 CPU 在 SPI Flash 上的启动

以 AR9344 为例，其它的型号也都基本相同

AR9344 遵循 MIPS 的要求，CPU 复位后，从 0xbfc00000 开始执行。

那么，总线地址 0x1fc00000 对应了什么外设呢？
从第 2 节可以看出， 0x1f000000 ~ 0x1fffffff 对应的正是 SPI Flash。

但是，Flash 是从 0x1f000000 开始映射的，那么 0x1fc00000 处映射的又是什么呢？

实际上，Flash 确实是从 0x1f000000 开始映射的，总共可以映射 16MB 的 Flash 空间。
为了兼容 MIPS 要求的 0xbfc00000 的复位地址，AR9344 在 0x1fc00000 处又重新对 Flash 进行了一次映射，映射了 Flash 开头的 4MB

但是，如果 0x1fc00000 处映射了 Flash， 那么从 0x1f000000 处就只能访问 12MB 的 Flash 空间了。
因此 AR9344 专门设置了一个寄存器，用来控制是否从 0x1fc00000 处重新映射 Flash。
这样的话，当 Bootloader 完成了启动过程，就可以关闭在 0x1fc00000 的重新映射，这样就能从 0x1f000000 
处访问到完整的 16MB 的 Flash 空间了。

## 7. 如果 Flash 容量小于 16MB，那么会怎样

上一节已经说了，从 0x1f000000 可以映射 16MB 的 Flash 空间，那么如果 Flash 容量小于 16M，会怎么样？

实际上，Flash 对于地址有个回绕功能，也就是说，如果给定的地址超过了 Flash 的地址范围，
那么 Flash 会自动让地址变成 0，即又从头开始。

*Flash 的内部处理方式是将地址中超出容量的那些位 置零(或者说溢出)，效果上相当于用给出的地址除以 Flash 的容量取余数*

这样对 Flash 来说，地址总是有效的，对外界来说，就像是 Flash 的数据在循环。

例如：某个 Flash 的容量是 4MB，那么它的地址范围就是 0x000000 ~ 0x3fffff。
如果访问 0x400000，就变成了访问 0x000000；如果访问 0x7f0000，就变成了访问 0x3f0000；如果访问 0xff0000，也是变成了访问 0x3f0000。

回到 AR9344 的 16MB 映射，如果 Flash 容量小于 16MB，那么在这 16MB 上的体现就是 Flash 数据重复映射了多次
例如：Flash 是 8MB 的，那么它就在 0x1f000000 和 0x1f800000 上各映射了一次；
如果 Flash 是 4MB 的，那么它就在 0x1f000000、0x1f400000、0x1f800000、0x1fc00000 上各映射了一次；

## 8. OpenWrt 的 mach 文件中，ART 数据地址的来历

在上面几节说来这么多之后，对于类似于 u8 \*art = KSEG1ADDR(0x1fff0000) 这样的地址就应该很好理解了：

假设 ART 位于 Flash 的最后 64KB，那么它在 Flash 中的地址就是 Flash 容量 - 64KB。

例如：
Flash 是 4MB 的，那么 ART 在 Flash 中的起始地址就是 0x400000 - 0x10000 = 0x3f0000；
Flash 是 8MB 的，那么 ART 在 Flash 中的起始地址就是 0x800000 - 0x10000 = 0x7f0000；
Flash 是 16MB 的，那么 ART 在 Flash 中的起始地址就是 0x1000000 - 0x10000 = 0xff0000；

那么，这个地址在总上的映射就是 0x1f000000 + ART 地址，即：
Flash 是 4MB 的，那么 ART 在总线中的起始地址就是 0x1f000000 + 0x3f0000 = 0x1f3f0000；
Flash 是 8MB 的，那么 ART 在总线中的起始地址就是 0x1f000000 + 0x7f0000 = 0x1f7f0000；
Flash 是 16MB 的，那么 ART 在总线中的起始地址就是 0x1f000000 + 0xff0000 = 0x1fff0000；

现在，假设 OpenWrt 并不知道 Flash 的容量：
如果在 mach 中使用 8MB Flash 的 ART 地址 0x1f7f0000，如果 Flash 是 16MB 的，那么通过 0x1f7f0000 就无法获取正确的 ART 数据。

那么解决方法是什么呢：
那就是假设 Flash 都是 16MB 的，这样的话就会使用 0x1fff0000 这个地址。
当 Flash 容量小于 16MB 时，根据上一节所说的地址回绕功能，通过 0xff0000 访问到的就是 Flash 实际最后 64KB 的内容。

这就是 mach 文件中这个地址的来历。

## 9. (补充) 为什么从 AR7240 开始 U-Boot 的基址是 0x9f000000

如果编译过 U-Boot，那么可能会发现 AR7161/AR913X 之类的 CPU，其启动地址，即 U-Boot 的基址 (TEXT_BASE) 是 0xbf000000；而之后的 CPU (从 AR7240 开始)，其 U-Boot 的基址是 0x9f000000。
但是如第6节所说，CPU 复位后的执行地址是 0xbfc00000，那么为什么 U-Boot 能从这两个地址启动呢？

就 AR7161/AR913X 来说，这很容易根据解释：
首先必须明确，无论 U-Boot 在编译时指定的基址是什么，CPU 总是从 0xbfc00000 开始执行的。
根据第6节所述，0xbf000000 跟 0xbfc00000 映射了相同的 Flash 地址，也就是说这两个地址最多能映射4MB相同的数据。
那么 U-Boot 在执行时，就可以根据 (当前执行地址 - 0xbfc00000 + 0xbf000000) 的方式跳转到与 0xbf000000 相对应的地址来继续执行。
在执行跳转后，U-Boot 会立即关闭 0x1fc00000 处 Flash 的重映射，因此 U-Boot 必须在进入 C 环境 (即使用栈帧) 之前完成跳转，否则栈中会保留 0xbfc00000 相关的地址，在禁用 Flash 重映射后会失效。

就 AR7240 开始的 CPU 来说，就要考虑到缓存了：
从 AR7240 开始，CPU 支持 Flash 映射的读缓存和指令缓存。但是根据第4节的描述，CPU 初始化时，kseg0 段的缓存并未初始化，不能访问。
因此 U-Boot 会先将缓存初始化，然后使用和 AR7161/AR913X 相同的方式 (当前执行地址 - 0xbfc00000 + 0x9f000000) 来跳转到与 0x9f000000 相对应的地址 ，即 kseg0 段来执行。

AR7161/AR913X 不支持 Flash 映射的指令缓存，虽然可以从 0x9f000000 读取 Flash 数据，但是从 0x9f000000 执行则会导致 CPU 异常。

从 kseg0 执行 U-Boot 的好处显而易见：有缓存，能加快 U-Boot 的执行速度。

## 10. 后记

嵌入式设备开发需要掌握很多的知识，并不是只会编程就行了。例如还需要学习计算机组成原理、操作系统原理等等。
很多开发者对这些并不了解，因此对于这些文件都会有相当多的疑问。
楼主尽量将文章内容简化，以使其变得能够容易理解，并希望能够起到抛砖引玉的作用。
希望通过此文让开发者能过对嵌入式设备开发有更深入的了解，消除疑惑。


{% highlight ruby %}
文档信息
--------------
* 作者:
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

[AR/QCA SPI 启动原理和 ART 地址定位原理](http://www.openwrt.pro/post-81.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
