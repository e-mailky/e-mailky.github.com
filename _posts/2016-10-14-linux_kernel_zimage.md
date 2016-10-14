---
layout: post
title:  "S3C2410平台上运行为例，讲解内核的解压过程"
date:   2016-10-14 17:09:35
categories: [编程, Linux, Kernel]
tags: [Linux, Kernel, ]
description: ""
---

&emsp;&emsp;内核编译完成后会生成zImage内核镜像文件。关于bootloader加载zImage到内核，并且跳转到zImage开始地址运行zImage的过程，相信大家都很容易理解。但对于zImage是如何解压的过程，就不是那么好理解了。本文将结合部分关键代码，讲解zImage的解压过程。
&emsp;&emsp;先看看zImage的组成吧。在内核编译完成后会在arch/arm/boot/下生成zImage。

在arch/armboot/Makefile中：

```
$(obj)/zImage: $(obj)/compressed/vmlinux FORCE
    $(call if_changed,objcopy)
```

由此可见，zImage的是elf格式的arch/arm/boot/compressed/vmlinux二进制化得到的

在arch/armboot/compressed/Makefile中：

```
$(obj)/vmlinux: $(obj)/vmlinux.lds $(obj)/$(HEAD) $(obj)/piggy.o /
                              $(addprefix $(obj)/, $(OBJS)) FORCE
    $(call if_changed,ld)

$(obj)/piggy.gz: $(obj)/../Image FORCE
    $(call if_changed,gzip)

$(obj)/piggy.o: $(obj)/piggy.gz FORCE
```

其中Image是由内核顶层目录下的vmlinux二进制化后得到的。注意：arch/arm/boot/compressed/vmlinux是位置无关的，这个有助于理解后面的代码。，链接选项中有个 –fpic参数：

    EXTRA_CFLAGS := -fpic

**总结一下zImage的组成，它是由一个压缩后的内核piggy.o，连接上一段初始化及解压功能的代码（head.o misc.o）,组成的。**

&emsp;&emsp;下面就要看内核的启动了，那么内核是从什么地方开始运行的呢？这个当然要看lds文件啦。
zImage的生成经历了两次大的链接过程：
&emsp;&emsp;一次是顶层vmlinux的生成，由arch/arm/boot/vmlinux.lds（这个lds文件是由arch/arm/kernel/vmlinux.lds.S生成的）决定；
&emsp;&emsp;另一次是arch/arm/boot/compressed/vmlinux的生成，是由arch/arm/boot/compressed/vmlinux.lds（这个lds文件是由arch/arm/boot/compressed/vmlinux.lds.in生成的）决定。zImage的入口点应该由arch/arm/boot/compressed/vmlinux.lds决定。从中可以看出入口点为‘_start’

```
OUTPUT_ARCH(arm)

ENTRY(_start)

SECTIONS

{

        . = 0;

       _text = .;

       .text : {

       _start = .;

       *(.start)

       *(.text)

        ……

}
```

在arch/arm/boot/compressed/head.S中找到入口点。

看看head.S会做些什么样的工作：

* 对于各种Arm CPU的DEBUG输出设定，通过定义宏来统一操作；
* 设置kernel开始和结束地址，保存architecture ID；
* 如果在ARM2以上的CPU中，用的是普通用户模式，则升到超级用户模式，然后关中断
* 分析LC0结构delta offset，判断是否需要重载内核地址(r0存入偏移量，判断r0是否为零)。
* 需要重载内核地址，将r0的偏移量加到BSS region和GOT table中的每一项。
  对于位置无关的代码，程序是通过GOT表访问全局数据目标的，也就是说GOT表中中记录的是全局数据目标的绝对地址，所以其中的每一项也需要重载。
* 清空bss堆栈空间r2－r3
* 建立C程序运行需要的缓存
* 这时r2是缓存的结束地址，r4是kernel的最后执行地址，r5是kernel境象文件的开始地址
* 用文件misc.c的函数decompress_kernel()，解压内核于缓存结束的地方(r2地址之后)。

可能大家看了上面的文字描述还是不清楚解压的动态过程。还是先用图表的方式描述下代码的搬运解压过程。然后再针对中间的一些关键过程阐述。

假定zImage在内存中的初始地址为0x30008000(这个地址由bootloader决定，位置不固定)

1. 初始状态

<table>
    <thead>
        <tr>
            <th>.text</th>
            <th>0x30008000开始，包含piggydata段（即压缩的内核段）</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>. got</td>
            <td>?</td>
        </tr>
        <tr>
            <td>. data</td>
            <td>?</td>
        </tr>
        <tr>
            <td>.bss</td>
            <td>?</td>
        </tr>
        <tr>
            <td>.stack</td>
            <td>4K大小</td>
        </tr>
    </tbody>
</table>

2. head.S调用misc.c中的decompress_kernel刚解压完内核后

<table>
    <thead>
        <tr>
            <th>.text</th>
            <th>0x30008000开始，包含piggydata段（即压缩的内核段）</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>. got</td>
            <td>?</td>
        </tr>
        <tr>
            <td>. data</td>
            <td>?</td>
        </tr>
        <tr>
            <td>.bss</td>
            <td>?</td>
        </tr>
        <tr>
            <td>.stack</td>
            <td>4K大小</td>
        </tr>
        <tr>
            <td>.解压函数所需缓冲区</td>
            <td>64K大小</td>
        </tr>
        <tr>
            <td>解压后的内核代码</td>
            <td>64K大小</td>
        </tr>
    </tbody>
</table>

3. 此时会将head.S中的部分代码重定位

<table>
    <thead>
        <tr>
            <th>.text</th>
            <th>0x30008000开始，包含piggydata段（即压缩的内核段）</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>. got</td>
            <td>?</td>
        </tr>
        <tr>
            <td>. data</td>
            <td>?</td>
        </tr>
        <tr>
            <td>.bss</td>
            <td>?</td>
        </tr>
        <tr>
            <td>.stack</td>
            <td>4K大小</td>
        </tr>
        <tr>
            <td>解压函数所需缓冲区</td>
            <td>64K大小</td>
        </tr>
        <tr>
            <td>解压后的内核代码</td>
            <td>小于4M</td>
        </tr>
        <tr>
            <td>head.S中的部分重定位代码代码</td>
            <td>reloc_start至reloc_end</td>
        </tr>
    </tbody>
</table>

4. 跳转到重定位后的reloc\_start处，由reloc\_start至reloc\_end的代码复制解压后的内核代码到0x30008000处，并调用call_kernel跳转到0x30008000处执行。

<table>
    <thead>
        <tr>
            <th>解压后的内核</th>
            <th>0x30008000开始</th>
        </tr>
    </thead>
</table>


**在通过head.S了解了动态过程后，大家可能会有几个问题：**

* 问题1：zImage是如何知道自己最后的运行地址是0x30008000的？
* 问题2：调用decompress_kernel函数时，其4个参数是什么值及物理含义？
* 问题3：解压函数是如何确定代码中压缩内核位置的？

先回答第1个问题

&emsp;&emsp;这个地址的确定和Makefile和链接脚本有关，在arch/arm/Makefile文件中的

&emsp;&emsp;textaddr-y := 0xC0008000 这个是内核启动的虚拟地址
&emsp;&emsp;TEXTADDR := $(textaddr-y)

在arch/arm/mach-s3c2410/Makefile.boot中

&emsp;&emsp;zreladdr-y := 0x30008000 这个就是zImage的运行地址了

在arch/arm/boot/Makefile文件中

&emsp;&emsp;ZRELADDR := $(zreladdr-y)

在arch/arm/boot/compressed/Makefile文件中

&emsp;&emsp;zreladdr=$(ZRELADDR)

在arch/arm/boot/compressed/Makefile中有

&emsp;&emsp;.word zreladdr @ r4

内核就是用这种方式让代码知道最终运行的位置的

接下来再回答第2个问题

```
decompress_kernel(ulg output_start, ulg free_mem_ptr_p, ulg free_mem_ptr_end_p,int arch_id)

output_start：指解压后内核输出的起始位置，此时它的值参考上面的图表，紧接在解压缓冲区后；

free_mem_ptr_p：解压函数需要的内存缓冲开始地址；

ulg free_mem_ptr_end_p：解压函数需要的内存缓冲结束地址，共64K；

arch_id ：architecture ID，对于SMDK2410这个值为193；
```

最后回答第3个问题

首先看看piggy.o是如何生成的，在arch/arm/boot/compressed/Makefie中

&emsp;&emsp;$(obj)/piggy.o: $(obj)/piggy.gz FORCE

Piggy.o是由piggy.S生成的，咱们看看piggy.S的内容：

```
    .section .piggydata,#alloc
    .globl input_data

input_data:

    .incbin "arch/arm/boot/compressed/piggy.gz"
    .globl input_data_end

input_data_end:
```
再看看misc.c中decompress\_kernel函数吧，它将调用gunzip()解压内核。gunzip()在lib/inflate.c中定义，它将调用NEXTBYTE()，进而调用get_byte()来获取压缩内核代码。

在misc.c中

    #define get_byte() (inptr < insize ? inbuf[inptr++] : fill_inbuf())

查看fill_inbuf函数

```
int fill_inbuf(void)

{
    if (insize != 0)
        error("ran out of input data");

    inbuf = input_data;

    insize = &input_data_end[0] - &input_data[0];

    inptr = 1;

    return inbuf[0];

}
```

发现什么没？这里的input\_data不正是piggy.S里的input_data吗？这个时候应该明白内核是怎样确定piggy.gz在zImage中的位置了吧。

时间关系，可能叙述的不够详细，大家可以集合内核代码和网上的其它相关文章，理解启动解压过程。

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载](http://www.cnblogs.com/hoys/archive/2012/02/22/2363399.html)
{% endhighlight %}

