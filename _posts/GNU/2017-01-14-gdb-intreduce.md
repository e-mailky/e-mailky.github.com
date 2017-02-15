---
layout: post
title:  玩转GDB
categories: [工具软件]
tags: [GNU, Linux]
description: ""
---

# 启动GDB开始调试

&emsp;&emsp;&emsp;&emsp;写在最前面：GDB是unix相关操作系统中C/C++程序开发必不可少的工具，
它的功能之强大，是其它调试器所不能匹敌的。但是，现实的工作中，有很多开发者因为GDB本身入门门槛比较高，
而被拒之门，与如此强大的失之交臂。笔者在近两年的C/C++开发工作中，对GDB本身的有一点研究，
在这里总结出一系列《手把手教你玩转GDB》的文章，一方面权当是对自己经验的一个总结，
一方面也是真的想能够对刚接触GDB的开发者朋友带去一些帮助，让更多的人来使用如此强大的工具。今天推出第一篇：

## 第一部分 牛刀小试：启动GDB开始调试

### 1. 启动GDB开始调试：

- gdb program       ///最常用的用gdb启动程序，开始调试的方式
- gdb program core  ///用gdb查看core dump文件，跟踪程序core的原因
- gdb program pid   ///用gdb调试已经开始运行的程序，指定pid即可

### 2. 应用程序带命令行参数的情况，可以通过下面两种方法启动：

- 启动GDB的时候，加上–args选项，然后把应用程序和其命令行参数带在后面，具体格式为：gdb –args program args
- 先按1中讲的方法启动GDB, 然后再执行run命令的时候，后面加上参数

### 3. 退出GDB：

- End-of-File(Ctrl+d)
- quit或者q

### 4. 在GDB调试程序的时候执行shell命令：

- shell command args（也可以先执行shell命令，GDB会退出到当前shell, 执行完command后，然后在shell中执行exit命令，便可回到GDB）
- make make-args（等同于shell make make-args）

### 5. 在GDB中获取帮助：

1. 在GDB中执行help命令，可以得到如图1所示的帮助信息：

![图1](/images/gnu/gdb_help.jpg)

图1 GDB帮助菜单

由图1可以看出，GDB中的命令可以分为八类：

别名(aliases)、断点(breakpoints)、数据(data)、文件(files)、内部(internals)、隐含(obscure)、
运行(running)、栈(stack)、状态(status)、支持(support)、跟踪点(tracepoints)和用户自定义(user-defined)。

2. help class-name：查看该类型的命令的详细帮助说明
3. help all：列出所有命令的详细说明
4. help command：列出命令command的详细说明
5. apropos word：列出与word这个词相关的命令的详细说明
6. complete args：列出所有以args为前辍的命令

### 6. info和show：

- info：用来获取和被调试的应用程序相关的信息
- show：用来获取GDB本身设置相关的一些信息

## 第二部分 Breakpoint, Watchpoint和Catchpoint

&emsp;&emsp;&emsp;&emsp;本文是《手把手教你玩转GDB》系列的第二篇，主要内容是用GDB调试程序中比较常用到
的断点（breakpoint）、监视点（watchpoint）和捕捉点（catchpoint）。虽然说这三类point的功能是不一样的，
但它们的用法却极为相似。因此，本文将以断breakpoint为例，进行详细的介绍，关于watchpoint和catchpoint的
介绍就相对比较粗略，相信读者朋友如果能够理解breakpoint的部分，那么便可以触类旁通，学会watchpoint和catchpoint的用法。

### 一. Breakpoint: 作用是让程序执行到某个特定的地方停止运行

#### 设置breakpoint：

1. break function: 在函数funtion入口处设置breakpoint
2. break +offset: 在程序当前停止的行向前offset行处设置breakpoint
3. break –offset: 在程序当前停止的行向衙offset行处设置breakpoint
4. break linenum: 在当前源文件的第linenum行处设置breakpoint
5. break filename:linenum: 在名为filename的源文件的第linenum行处设置breakpoint
6. break filename:function: 在名为filename的源文件中的function函数入口处设置breakpoint
7. break *address: 在程序的地址address处设置breakpoint
8. break … if cond: …代表上面讲到的任意一个可能的参数，在某处设置一个breakpoint, 但且仅但cond为true时，程序停下来
9. tbreak args: 设置一个只停止一次的breakpoints, args与break命令的一样。这样的breakpoint当第一次停下来后，就会被自己删除
10. rbreak regex: 在所有符合正则表达式regex的函数处设置breakpoint

#### info breakpoints [n]：

查看第n个breakpoints的相关信息，如果省略了n，则显示所有breakpoints的相关信息

#### pending breakpoints:

是指设置在程序开始调试后加载的动态库中的位置处的breakpoints

1. set breakpoint pending auto: GDB缺省设置，询问用户是否要设置pending breakpoint
2. set breakpoint pending on: GDB当前不能识别的breakpoint自动成为pending breakpoint
3. set breakpoint pending off: GDB当前不能识别某个breakpoint时，直接报错
4. show breakpoint pending: 查看GDB关于pending breakpoint的设置的行为(auto, on, off)

#### breakpoints的删除：

1. clear: 清除当前stack frame中下一条指令之后的所有breakpoints
2. clear function & clear filename:function: 清除函数function入口处的breakpoints
3. clear linenum & clear filename:linenum: 清除第linenum行处的breakpoints
4. delete [breakpoints] [range…]: 删除由range指定的范围内的breakpoints，range范围是指breakpoint的序列号的范围

####  breakpoints的禁用、启用:

1. disable [breakpoints] [range…]: 禁用由range指定的范围内的breakpoints
2. enable [breakpoints] [range…]: 启用由range指定的范围内的breakpoints
3. enable [breakpoints] once [range…]: 只启用一次由range指定的范围内的breakpoints，
等程序停下来后，自动设为禁用
4. enable [breakpoints] delete [range…]: 启用range指定的范围内的breakpoints，
等程序停下来后，这些breakpoints自动被删除

#### 条件breakpoints相关命令：

1. 设置条件breakpoints可以通过break … if cond来设置，也可以通过condition bnum expression来设置，
在这里首先要通过（1）中介绍的命令设置好breakpoints，然后用condition命令来指定某breakpoint的条件，
该breakpoint由bnum指定，条件由expression指定
2. condition bnum: 取消第bnum个breakpoint的条件
3. ignore bnum count: 第bnum个breakpoint跳过count次后开始生效

####  指定程序在某个breakpoint处停下来后执行一串命令：

1. 格式：commands [bnum]
  
   … command-list …
   
   end
2. 用途：指定程序在第bnum个breakpoint处停下来后，执行由command-list指定的命令串，
  如果没有指定bnum，则对最后一个breakpoint生效
3. 取消命令列表： commands [bnum]
   
   end

例子：

```
break foo if x>0
commands
silent
printf “x is %d\n”,x
continue
end
```

上面的例子含义：当x>0时，在foo函数处停下来，然后打印出x的值，然后继续运行程序

### 二 Watchpoint: 它的作用是让程序在某个表达式的值发生变化的时候停止运行，达到‘监视’该表达式的目的

1. 设置watchpoints:
  + watch expr: 设置写watchpoint，当应用程序写expr, 修改其值时，程序停止运行
  + rwatch expr: 设置读watchpoint，当应用程序读表达式expr时，程序停止运行
  + awatch expr: 设置读写watchpoint, 当应用程序读或者写表达式expr时，程序都会停止运行
2. info watchpoints:
查看当前调试的程序中设置的watchpoints相关信息
3. watchpoints和breakpoints很相像，都有enable/disabe/delete等操作，使用方法也与breakpoints的类似

### 三 Catchpoint: 的作用是让程序在发生某种事件的时候停止运行，比如C++中发生异常事件，加载动态库事件

#### 设置catchpoints:

1. 设置catchpoints:
  + catch event: 当事件event发生的时候，程序停止运行，这里event的取值有：
    1. throw: C++抛出异常
    1. catch: C++捕捉到异常
    1. exec: exec被调用
    1. fork: fork被调用
    1. vfork: vfork被调用
    1. load: 加载动态库
    1. load libname: 加载名为libname的动态库
    1. unload: 卸载动态库
    1. unload libname: 卸载名为libname的动态库
    1. syscall [args]: 调用系统调用，args可以指定系统调用号，或者系统名称
  +  tcatch event: 设置只停一次的catchpoint，第一次生效后，该catchpoint被自动删除
2. catchpoints和breakpoints很相像，都有enable/disabe/delete等操作，使用方法也与breakpoints的类似

## 第三部分 常用命令

&emsp;&emsp;&emsp;&emsp;本文是手把手教你玩转GDB的第三篇，主要内容是介绍一些在程序调试过程中
最常用的GDB命令，废话不多话，开始今天的正题。

### attach process-id/detach

1. attach process-id: 在GDB状态下，开始调试一个正在运行的进程，其进程ID为process-id
2. detach: 停止调试当前正在调试有进程，与attach配对试用

### kill

1. 基本功能：杀掉当前GDB正在调试的应用程序所对应的子进程
2. 如果想不退出GDB而对当前正在调试的应用程序重新编译、链接，
可以在GDB中执行kill杀掉子进程，等编译、链接完后，再重新执行run，GDB便可加载新的可执行程序启动调试

### 多线程程序调试相关：

1. thread threadno：切换当前线程到由threadno指定的线程
2. info threads：查看GDB当前调试的程序的各个线程的相关信息
3. thread apply [threadno] [all] args：对指定（或所有）的线程执行由args指定的命令

### 多进程程序调试相关(fork/vfork)：

1. 缺省方式：fork/vfork之后，GDB仍然调试父进程，与子进程不相关
2. set follow-fork-mode mode：设置GDB行为，mode为parent时，与缺省情况一样；mode为child时，
fork/vfork之后，GDB进入子进程调试，与父进程不再相关
3. show follow-fork-mode：查看当前GDB多进程跟踪模式的设置

### step & stepi

1. step [count]: 如果没有指定count, 则继续执行程序，直到到达与当前源文件不同的源文件中时停止；
如果指定了count, 则重复coun行上面的过程count次
2. stepi [count]: 如果没有指定count, 继续执行下一条机器指令，然后停止；如果指定了count，则重复上面的过程count次
3. step比较常见的应用场景：在函数func被调用的某行代码处设置断点，等程序在断点处停下来后，
可以用step命令进入该函数的实现中，但前提是该函数编译的时候把调试信息也编译进去了，否则step会跳过该函数。

### next & nexti

1. next [count]: 如果没有指定count, 单步执行下一行程序；如果指定了count，单步执行接下来的count行程序
2. nexti [count]: 如果没有指定count, 单步执行下一条指令；如果指定了count, 单步执行接下来的count条执行
3. stepi和nexti的区别：nexti在执行某机器指令时，如果该指令是函数调用，那么程序执行直到该函数调用结束时才停止。

### .continue [ignore-count] 

唤醒程序，继续运行，至到遇到下一个断点，或者程序结束。如果指定ignore-count，
那么程序在接下来的运行中，忽略ignore-count次断点。

### finish & return

1. finish: 继续执行程序，直到当前被调用的函数结束，如果该函数有返回值，把返回值也打印到控制台
2. return [expression]: 中止当前函数的调用，如果指定了expression，把expresson值当做当前函数的返回值；
如果没有，直接结束当前函数调用

### 信号的处理

1. info signals & info handle：打印所有的信号相关的信息，以及GDB缺省的处理方式：

  ![图2](/images/gnu/gdb_signal.jpg)

2. handle signal action: 设置GDB对具体某个信号的处理方式。signal可以为信号整数值，
也可以为SIGSEGV这样的符号。action的取值有
  + stop和nostop: nostop表示当GDB收到指定的信号，不会应用停止程序的执行，只会打印出一条收到信号
    的消息，因此，nostop也暗含了下面的print; 而stop则表示，当GDB收到指定的信号，停止应用程序的执行。
  + print和noprint: print表示如果收到指定的信号，打印出一条信息; noprint与print表示相反的意思
  + pass和nopass：pass表示如果收到指定的信号，把该信号通知给应用程序; nopass表示与pass相反的意思
  + ignore和noignore: ignore与nopass同义，同理，noignore与pass同义


## 第四部分 函数调用栈(call stack)探密

&emsp;&emsp;&emsp;&emsp;本文是GDB系列的第四篇，感兴趣的朋友可以阅读本系列的前三篇。
本文的主要内容是讲如何用GDB来查看C/C++程序中函数调用栈(call stack)的相关信息，通过介绍一些
相关的命令及其用法，让读者朋友能够循序渐进了解调用栈的各个方面，更好的驾驭程序。下面开始今天的内容。

&emsp;&emsp;&emsp;&emsp;我们知道，通常一个程序的运行，不外乎是A函数调用B，B函数调用C等等，
等所有的调用都完成后，整个程序的运行也就ok了。在这个过程中，每当有新的函数调用，系统都会把该函数的
一些信息，包括函数的参数，以及一些寄存器的值等，保存到调用栈(call stack)上。等该函数运行完成后，
这些信息再从调用栈上弹出(pop)。如下图所示，是一个完整的调用栈：

![图3](/images/gnu/callstack.jpg)

在上图中，整体叫做调用栈(call stack)，每一行叫做一桢(frame)。我们来看看桢信息的组成有哪些：

1. 桢号：调用栈中对桢的一个编号，从0开始，依次增大
2. PC：Program counter寄存器，指向当前桢中下一条要执行的指令的地址
3. 函数名：当前桢中被调用的函数的名字
4. 参数及传入的值：当前桢中被调用的函数在调用时传入的参数及其值
5. 源码位置：当前桢执行到的源码位置，格式为 file:linenum

&emsp;&emsp;&emsp;&emsp;这里还有一点需要说明，不知道细心的读者朋友有没有发现，
foo那一桢没有PC的地址，GDB通过这样来标示该桢是当前正在执行到的桢，因此我们通过看调用栈的信息，
便可得知程序执行到哪里了。

&emsp;&emsp;&emsp;&emsp;读者朋友有没有觉得原来函数调用的过程还有这么多信息可以知道啊，下面我就开始介绍
一些GDB命令，通过这些命令你便可以查看到上面介绍的这些信息，甚至更加详细的信息。

###  查看调用栈信息：（具体信息的内容，与上面第二部分中介绍的相同）

1. backtrace: 显示程序的调用栈信息，可以用bt缩写
2. backtrace n: 显示程序的调用栈信息，只显示栈顶n桢(frame)
3. backtrace –n: 显示程序的调用栈信息，只显示栈底部n桢(frame)
4. set backtrace limit n: 设置bt显示的最大桢层数
5. where, info stack：都是bt的别名，功能一样

###  查看桢信息：

1. frame n: 查看第n桢的信息， frame可以用f缩写
2. frame addr: 查看pc地址为addr的桢的相关信息
3. up n: 查看当前桢上面第n桢的信息
4. down n: 查看当前桢下面第n桢的信息

###  查看更加详细的信息：

1. info frame、info frame n或者info frame addr查看指定桢的详细信息，关于详细信息的内容，这里有必要做一个介绍，如下图所示：

![图3](/images/gnu/frame.jpg)

上图中显示的信息有：

- 当前桢的地址: 0xbffff400
- 当前桢PC: eip = 0x8048516
- 当前桢函数： bar (test.cpp:16)
- caller桢的PC: saved eip 0x8048535
- caller桢的地址: called by frame at 0xbffff420
- callee桢的地址: caller of frame at 0xbffff3e0
- 源代码所用的程序的语言(c/c++): source language c++
- 当前桢的参数的地址及值: Arglist at 0xbffff3f8, args: name=0x8048621 “jessie”, myname=0x804861c “jack”
- 当前相中局部变量的地址：Locals at 0xbffff3f8, Previous frame’s sp is 0xbffff400
- 当前桢中存储的寄存器： Saved registers: ebp at 0xbffff3f8, eip at 0xbffff3fc

2. info args：查看当前桢中的参数
3. info locals：查看当前桢中的局部变量
4. info catch：查看当前桢中的异常处理器（exception handlers）



{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [玩转GDB]
{% endhighlight %}

[玩转GDB](http://www.wuzesheng.com/?p=1327)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
