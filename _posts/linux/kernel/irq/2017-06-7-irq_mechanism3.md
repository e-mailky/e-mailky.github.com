---
layout: post
title:  Linux Kernel中断机制3——硬件支撑
categories: [Linux]
tags: [Linux, Interrupt, C, Kernel, 硬件]
description: ""
---

&emsp;&emsp;&emsp;&emsp;中断（Interrupt）包括中断和异常两种类型，异常通常由CPU上执行的指令直接触发，
而中断是由外设发出的电信号触发的，但是那么是否所有的外设都直接接在CPU的中断PIN脚上触发中断？
CPU有多少负责中断的PIN脚？CPU如何区别可屏蔽中断和非可屏蔽中断？CPU如何区别Faults、Traps、Aborts？
本篇文章主要来搞懂这些问题。

## 一、中断控制器
&emsp;&emsp;&emsp;&emsp;首先，CPU肯定不会为允许所有的外设都直接接在其上（否则要总线干什么），
即使是通知发生中断这一种功能。应该有专门的中间设备/元器件负责这个工作，这个中间设备就是中断控制器。
我刚刚学习中断的时候，甚至都不清楚中断控制器是个硬件还是软件，是否是CPU的一部分，
那么这个地方先给出两个中断控制器的外观图片：

![T1](/images/kernel/irq/Intel-P8259A-300x173.jpg)
![T2](/images/kernel/irq/ich10-300x203.jpg)
![T3](/images/kernel/irq/82093aa-300x225.jpg)

&emsp;&emsp;&emsp;&emsp;以上三张图片分别是Intel的IP8259A芯片、intel的ICH10南桥芯片以及Intel的S82093AA芯片，
他们都有中断控制器的功能，其中8259A芯片比较老式，目前已经基本淘汰；Intel平台流行的做法是
高级可编程将中断控制器(APIC)集成到南桥芯片（I/O Controller Hub）中，而较老的S82093AA芯片是最初的APIC的形态。

&emsp;&emsp;&emsp;&emsp;可编程控制器（Programmable Interrupt Controller）是通常由两片 8259A 风格的
外部芯片以“级联”的方式连接在一起。每个芯片可处理多达 8 个不同的 中断请求。因为从 PIC 的 INT 
输出线连接到主 PIC 的 IRQ2 引脚，所以可用 IRQ 线的个数达到 15 个，如下图所示（示意图，并不代表一定这么连接）：

![T4](/images/kernel/irq/8259A.gif)

&emsp;&emsp;&emsp;&emsp;8259A除了起到向CPU引入多个外部中断源的作用外，
还有一些基本功能，如中断分级、中断屏蔽，中断管理（存储中断向量）等。

&emsp;&emsp;&emsp;&emsp;随着SMP架构的发展，Intel在2000年左右的时候率先引入一种名为
高级可编程控制器的新组件（Advanced Programmable Interrupt Controller），来替代老式的 8259A
可编程中断控制器。APIC包括两部分：一是“本地 APIC（Local APIC）”，主要负责传递中断信号到指定的处理器，
本地APIC通常集成到CPU内部，之所以成为Local，是相对CPU而言。
另外一个重要的部分是 I/O APIC，主要是收集来自 I/O 设备的 Interrupt 信号且将中断时发送信号到本地 APIC。

&emsp;&emsp;&emsp;&emsp;每个本地 APIC 都有 32 位的寄存器，一个内部时钟，一个本地定时设备以及为
本地中断保留的两条额外的 IRQ 线 LINT0 和 LINT1。所有本地 APIC 都连接到 I/O APIC，
形成一个多级 APIC 系统，如下图所示（示意图，并不代表一定这么连接）：

![T5](/images/kernel/irq/ioapic.gif)

&emsp;&emsp;&emsp;&emsp;当然，本地APIC除了接收来自IO APIC的中断信号，还可以接收其他来源的中断，
比如接在CPU LINT0和LINT1管脚上的中断、IPI中断（核间中断）、APIC定时器产生中断、性能监视计数器中断、
热传感器中断、APIC内部错误中断等。无论是PIC还是APIC，都通过某种方式与CPU相连（有的时候并不直接相连），
这解决两个问题：

1. CPU对多个外设的中断的管理
2. 多CPU的中断管理（APIC）

&emsp;&emsp;&emsp;&emsp;当然，APIC自有一套硬件逻辑去实现这些功能，Intel也提供了相关的用户手册，
这里就不深究了，有兴趣可以参考资料3——Intel® 64 and IA-32 Architectures Software Developer’s Manual 
Volume 3 Chapter 10 advanced programmable interrupt controller(APIC).

以下为ULK中关于IRQ线以及中断控制器的工作逻辑的描述：

&emsp;&emsp;&emsp;&emsp;每个能够发出中断请求的硬件设备控制器都有一条名为IRQ（Interrupt ReQuest）的输出线。
所有现有的IRQ线都与PIC的硬件段路的输入引脚相连，PIC执行下列动作：

1. 监视IRQ线，检查产生的信号。如果有两条或两条以上的IRQ线产生信号，就选择引脚编号较小的IRQ线。
2. 如果一个引发信号出现在IRQ线上：
   * 把接收到的引发信号转换成对应的向量。
   * 把这个向量存放在中断控制器的一个I/O端口，从而允许CPU通过数据总线读此向量。
   * 把引发信号发送到处理器的INTR引脚，产生一个中断。
   * 等待，直到CPU把这个中断信号写进PIC的一个I/O端口来确认它；当这种情况发生时，清INTR线。
 3. 返回到第一步。
   
## 二、Intel x86 CPU中断管脚
&emsp;&emsp;&emsp;&emsp;APIC系统主要作用是管理外设产生的异步中断，而对Intel x86架构下的各种异常，
如故障、陷阱以及终止，系统是如何管理的那？这得看Intel手册：Intel® 64 and IA-32 Architectures
 Software Developer’s Manual Volume 3 的Chaper 2 system architecture overview 和Chapter 6 
 interrupt and exception以及Chapter 10 APIC。

&emsp;&emsp;&emsp;&emsp;Intel x86架构提供LINT0和LINT1两个中断引脚，他们通常与Local APIC相连，
用于接收Local APIC传递的中断信号，另外，当Local APIC被禁用的时候，LINT0和LINT1即被配置为INTR和NMI管脚，
即外部IO中断管脚和非屏蔽中断管脚。INTR引脚负责向处理器通知发生了外部中断，处理器从系统总线上读出
由外部中断控制器提供的中断向量（Interrupt Vector）号，例如从8259a提供。NMI中断使用2号中断向量。

&emsp;&emsp;&emsp;&emsp;通常一个INTR引脚能够接收64个中断源，至于CPU内部怎么通过LINT0和LINT1进行中断的处理，
就不管了，抑或还有其他的中断引脚什么的...

## 三、Intel x86架构对中断的支持——中断向量和IDT表

&emsp;&emsp;&emsp;&emsp;Intel采用中断向量表(Interrupt Describtor Table)的方式去管理中断。
对于中断、陷阱、故障以及终止，有一些是Intel自己在设计CPU架构的时候就能够预知的，例如在执行除零时就会出现异常，
在页式管理机制就可能出现缺页异常，有一些是Intel无法预估的，比如一个未来设计的设备产生的中断。
对于前者，Intel称之为Intel 64 and IA-32 architectures for architecture-defined exceptions and interrupts，
即Intel64和IA-32架构定义的架构相关的异常和中断，姑且简称为架构相关的中断和异常。

&emsp;&emsp;&emsp;&emsp;为了区别这些中断和异常，Intel给出了中断向量（Interrupt Vector），
即使用一个数字代表一个特殊的中断或者异常类型，Intel规定中断向量号的范围是0~255，0~31号为架构相关的异常和中断，
32~255为User Defined Interrupt(保护模式)：

中断向量号 | 助记符 | 描述 | 类型
----- | ----- | ----- | -----  
0 |	#DE	Divide Error	 | Fault
1 |	#DB	| RESERVED	     | Fault/ Trap
2 |	—	| NMI Interrupt	 | Interrupt
3 |	#BP	| Breakpoint（INT 3）| 	Trap
4 |	#OF	| Overflow（INTO 0） | 	Trap
5 |	#BR	| BOUND Range Exceeded	|  Fault
6 |	#UD	| Invalid Opcode (Undefined Opcode)	 | Fault
7 |	#NM	| Device Not Available (No Math Coprocessor) | 	Fault
8 |	#DF	| Double Fault	|  Abort
9 |	 -  | Coprocessor Segment Overrun (reserved) | Fault


&emsp;&emsp;&emsp;&emsp;中断向量是中断向量表（IDT）的索引，而中断向量表存在于内存的某个位置，
由Intel的寄存器IDTR负责记录其基址（线性地址）和大小。IDT表中包含了操作系统中注册的外部IO中断的处理程序的入口地址，
以及其他操作系统实现的架构相关的中断和异常的处理函数入口地址(这些地址又存放在所谓的gate destribtor中)。
INTR和IDT的关系如下图所示：

![T6](/images/kernel/irq/IDTRandIDT.png)

gate在Linux内核（3.11.1）中数据结构如下（32bits）：

```
struct desc_struct {
union {
struct {
unsigned int a;
unsigned int b;
};
struct {
u16 limit0;
u16 base0;
unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1;
unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
};
};
} __attribute__((packed));
```

64bits的gate describtor：

```
/* 16byte gate */
struct gate_struct64 {
u16 offset_low;
u16 segment;
unsigned ist : 3, zero0 : 5, type : 5, dpl : 2, p : 1;
u16 offset_middle;
u32 offset_high;
u32 zero1;
} __attribute__((packed));
```

IDT中的Gate可以以某种方式获取到具体的中断/异常处理函数的入口地址，从而可以执行该中断/异常处理函数：

![T7](/images/kernel/irq/interrupt_procedure_call.png)

&emsp;&emsp;&emsp;&emsp;GDT和LDT分别为全局描述符表(Global Destribtor Table)和本地描述附表(Local Destribtor Table),
其中存储的内容为整个操作系统对各segment的描述，GDT、LDT、segment牵扯到内存管理的机制将在《Linux Kernel内存管理》
系列中被具体学习，此处关心gate整个概念。

## 四、中断过程
&emsp;&emsp;&emsp;&emsp;PIC和APIC是用于解决多设备中断管理问题的硬件设备，而APIC还可以解决SMP架构中断管理的问题。
PIC通常指两片8259a级联，而APIC通常包括两个部分Local APIC和IO APIC，Local APIC通常集成在CPU内部（Intel），
外部与IO APIC相连，内部与CPU管脚LINT0和LINT1相连。

&emsp;&emsp;&emsp;&emsp;Intel 64和IA-32架构CPU对外提供中断向量和中断描述符表（IDT）的机制来处理中断和异常。
中断按照下列逻辑触发并被执行(以键盘为例)：

1. 用户按下键盘按键；
2. 电平信号变化通知中断控制器发生了一次中断；
3. 中断控制器通知CPU此次中断的中断向量号；
4. CPU判定是否要处理此次中断，如果要处理，转5，否则退出；
5. 从IDTR寄存器读取IDT的基址+中断向量号，找到对应的中断处理函数入口地址；
6. CPU按照某种软件策略执行该中断处理函数；

## 五、中断硬件处理细节（IA32/Intel 64）

&emsp;&emsp;&emsp;&emsp;对于第四节中断过程的步骤4、步骤5的一些细节，描述如下（摘自ULK）：

&emsp;&emsp;&emsp;&emsp;当执行了一条指令后，cs和eip这对寄存器包含了下一条要执行的指令的逻辑地址。
在处理那条指令之前，控制单元会检查在运行前一条指令时是否存在已经发生了一个中断或者异常。
如果发生了一个中断或异常，那么控制单元执行下列操作：

1. 确定与中断或异常关联的向量i（i的范围为0～255）。
2. 读由idtr寄存器指向的IDT表中的第i项（在下面的描述中，我们假定IDT表项中包含的是一个中断门或一个陷阱门）。
3. 从gdtr寄存器获得GDT的基地址，并在GDT中查找，以读取IDT表项中的选择符标识的段描述符。这个描述符指定中断或异常处理程序所在段的基地址。
4. 确信中断是授权的（中断）发生源发出的。首先将当前特权级CPL（存放在cs寄存器的低两位）与段描述符（存放在GDT中）
   的描述符特权级DPL比较，如果CPL小于DPL，就产生一个“General Protection”异常，因为中断处理程序的特权不能低于
   引起中断的程序的特权。对于编程异常，则做进一步的安全检查：比较CPL与处于IDT中的门描述符的DPL，如果DPL小于CPL，
   就产生一个“General Protection”异常。这最后一个检查可以避免用户程序访问特殊的陷阱门或中断门。
5. 检查是否发生了特权级的变化，也就是说，CPL是否不同于所选择的段描述符的DPL。如果是，
   控制单元必须开始使用与新的特权级相关的栈。通过执行以下步骤来做到这点：
   * 读tr寄存器，以访问运行进程的TSS段。
   * 用与新特权级相关的栈段和桟指针的正确值装载ss和esp寄存器。这些值可以在TSS中找到。
   * 在新的栈中保存ss和esp以前的值，这些值定义了与旧特权级相关的栈的逻辑地址。
6. 如果发生的是“故障（Fault）”，用引起异常的指令地址装载cs和eip寄存器，从而使得这条指令能再次被执行。
7. 在栈中保存eflag、cs及eip的内容。
8. 如果硬件产生了一个错误码，则将它保存在栈中。
9. 装载cs和eip寄存器，其值分别是IDT表中的第i项门描述符的段选择符和偏移量字段。这些值给出了中断或者异常处理程序的第一条指令的逻辑地址。

&emsp;&emsp;&emsp;&emsp;控制单元所执行的最后一步就是跳转到中断或异常处理程序。还句话说，处理完中断信号后，
控制单元所执行的指令就是被选中处理程序的第一条指令。
中断或异常被处理后，相应的处理程序必须产生一条iret指令，把控制权转交给被中断的进程，这将迫使控制单元：

1. 用保存的栈中的值状态cs、eip或eflag寄存器。如果一个硬件出错码曾被压入栈中，并且在eip内容的上面，那么，执行iret指令前必须先弹出这个硬件出错码。
2. 检查处理程序的CPL是否等于cs中的最低两位的值（这意味着被中断的进程与处理程序运行在同一特权级）。如果是，iret终止执行；否则，转入下一步。
3. 从栈中装载ss和esp寄存器，因此，返回到与旧特权级相关的栈。
4. 检查ds、es、fs以及gs寄存器的内容，如果其中一个寄存器包含的选择符是一个段描述符，并且DPL值小于CPL，
   那么，清相应的段寄存器。控制单元这么做是为了禁止用户态的程序（CPL=3）利用内核以前所用的段寄存器（DPL=0）。
   如果不清这个寄存器，怀有恶意的用户态程序就能利用他们来访问内核地址空间。


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
