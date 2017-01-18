---
layout: post
title:  最常见的Linux用户程序异常----Segment Fault
categories: [Linux]
tags: [Linux]
description: ""
---


## 应用层非法访问地址空间的结果和途径
&emsp;&emsp;&emsp;&emsp;地址空间访问的结果分为三种：

* 分配一个新的页面。 
* 发送SIGSEGV信号给对应进程。
* 内核错误杀死进程。

如图所示：

![图](/images/kernel/segment.png)

&emsp;&emsp;&emsp;&emsp;应用程序访问地址的路径，有五种：

* 应用程序非法访问了内核态地址.
* 应用程序读取了读保护的线性区地址.
* 应用程序写入了写保护的线性区地址.
* 应用程序访问了不存在的用户虚拟地址.
* 应用程序系统调用参数错误.

![图](/images/kernel/segment2.png)

## 如何尽可能捕捉异常打印
### 内核配置`CONFIG\_DEBUG\_USER`

![图](/images/kernel/segment3.png)

![图](/images/kernel/segment4.png)

### 全局变量user_debug 

&emsp;&emsp;&emsp;&emsp;以下是打印具体信号量的code代码行。

![图](/images/kernel/segment5.png)

&emsp;&emsp;&emsp;&emsp;方式1: 修改内核源码

&emsp;&emsp;&emsp;&emsp;方式2: 修改U-boot的bootargs参数

&emsp;&emsp;&emsp;&emsp;解析uboot参数的代码如下所示：

![图](/images/kernel/segment6.png)

### 全局变量print\_fatal_signals

&emsp;&emsp;&emsp;&emsp;print\_fatal_signals的作用点如下所示：

![图](/images/kernel/segment7.png)

print\_fatal_signals的赋值方式同样有两种：

方式1: 修改内核源码

方式2: 修改U-boot的bootargs参数

以下是内核解析print\_fatal_signals的相关代码行：

![图](/images/kernel/segment8.png)

## 异常信息定位

- 根据signal num判断错误原因。
- 获取到进程的虚拟地址空间(/proc/pid/maps)，根据内核dump的pc地址进行定位。
- 使用gdb、objdump进行反汇编辅助定位。

### 实例1 非法访问导致的SIGSEGV 11信号
实验代码如下所示：

![图](/images/kernel/segment9.png)

&emsp;&emsp;&emsp;&emsp;在上面代码中，我们对预留的物理地址进行了映射；
但在访问时，出现了越界，即非法的地址访问。

&emsp;&emsp;&emsp;&emsp;由于打开了debug user，于是出现了以下打印：

![图](/images/kernel/segment10.png)

&emsp;&emsp;&emsp;&emsp;从上面打印可以知道，出现问题时，
触发的信号为signal 11.我们来回顾下之前所描述的，即SIGSEGV：

![图](/images/kernel/segment11.png)

&emsp;&emsp;&emsp;&emsp;另外，PC指针在地址0x19b10908，
而这个地址是属于哪里的呢？我们可以通过/proc/pid/maps一探究竟（pid在打印中有显示，为795）

![图](/images/kernel/segment12.png)

&emsp;&emsp;&emsp;&emsp;原来出错代码在libc-2.15库对应代码段的偏移 
！Offset = 0x19b10908 – 0x19a98000 = 0x78908

&emsp;&emsp;&emsp;&emsp;而/dev/mem的地址空间在19bd5000-19cd5000，
这显然不是操作预留重映射的地址空间了。

&emsp;&emsp;&emsp;&emsp;接下来进行反汇编查看，看下glibc中地址偏移为0x78908究竟作了什么操作，
arm-marvell-linux-gnueabi-objdump -S libc-2.15.so > libc-2.15.asm

![图](/images/kernel/segment13.png)

&emsp;&emsp;&emsp;&emsp;该句汇编的意思是，取r1地址上的数据放到[r0]+1的地址上。

&emsp;&emsp;&emsp;&emsp;实际上，78908偏移上的内容无所谓，
我们知道这个地址在glibc中就可以确认：应用层的确出现了异常内存访问。

### 实例2
&emsp;&emsp;&emsp;&emsp;代码如下所示：

```
#include <stdio.h>
#define __1M 1024*1024
#define __16K 0x4000
void main(void)
{
    void * ptr;
    ptr=malloc(__1M);
    
    printf("alloc virtual addr is 0x%0x\n", ptr);
    memset(ptr, 0x5a, __1M+4);
    while(1);
    return 0;
}
```

&emsp;&emsp;&emsp;&emsp;示例代码malloc申请了1M的内存，但是，使用时，多访问了4个字节。

运行结果:

```
/user_debug_test # ./out_of_range2 &
/user_debug_test # alloc virtual addr is 0x19bbe008
/user_debug_test #
```
![图](/images/kernel/segment14.png)


&emsp;&emsp;&emsp;&emsp;从图中可以确认堆大小为 0x19c9f000-0x19b9b000=0x104000，
而0x19bbe008属于该段地址。

&emsp;&emsp;&emsp;&emsp;因此即使是多访问了4个字节，也是不会报错的。

### 实例3 SIGFPE信号8错误

![图](/images/kernel/segment15.png)

### 实例4用户空间访问异常地址导致的SIGSEGV 错误

&emsp;&emsp;&emsp;&emsp;我们用同样的方法来看另一个问题，代码如下所示：

![图](/images/kernel/segment16.png)

&emsp;&emsp;&emsp;&emsp;如上所示，用户空间线程访问了0地址（不该访问的内核空间），导致奔溃。

![图](/images/kernel/segment17.png)

&emsp;&emsp;&emsp;&emsp;PC地址0x10688在我们示例的代码段中，
而LR地址0x19b80e64在pthread库代码中。

&emsp;&emsp;&emsp;&emsp;使用反汇编也能证明这点。

![图](/images/kernel/segment18.png)

## 总结
&emsp;&emsp;&emsp;&emsp;首先，内核需要打开相关的选项；然后，根据PC和LR地址，
结合出错pid的maps以及出错的信号，定位出错点。

&emsp;&emsp;&emsp;&emsp;现在kernel默认没有了USER_DUBUG选项,所以不用在打开
但是要打开如下开关

* /proc/sys/kernel/print-fatal-signals 
* ulimit -c x
* objdump汇编出源码查看即可

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [最常见的Linux用户程序异常----Segment Fault]
{% endhighlight %}

[最常见的Linux用户程序异常----Segment Fault](http://rick_stone.leanote.com/post/%E6%9C%80%E5%B8%B8%E8%A7%81%E7%9A%84Linux%E7%94%A8%E6%88%B7%E7%A8%8B%E5%BA%8F%E5%BC%82%E5%B8%B8-Segment-Fault-2)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
