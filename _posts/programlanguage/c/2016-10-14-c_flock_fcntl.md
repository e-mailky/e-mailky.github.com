---
layout: post
title:  "linxu c语言 fcntl函数和flock函数区别说明"
date:   2016-10-13 17:09:35
categories: [编程语言]
tags: [Linux, C, ]
description: ""
---

### flock和fcntl都有锁的功能，但他们还有一点小小的区别：

1. flock只能加全局锁，fcntl可以加全局锁也可以加局部锁。
2. 当一个进程用flock给一个文件加锁时，用另一个进程再给这个文件加锁，它会阻塞或者也可以返回加锁失败（可以自己设置）。
3. 当一个进程用fcntl给一个文件加锁时，用另一个进程去读或写文件时必须先获取加锁的信息，然后在给这个文件加锁。
3. 当给一个文件加fcntl的独占锁后，再给这个文件加flock的独占锁，其会进入阻塞状态。
4. 当给一个文件加flock的独占锁后，用fcntl去获取这个锁信息获取不到，再用fcntl仍然可以给文件加锁。

### linux下C语言中的flock函数用法 

表头文件  #include<sys/file.h>
定义函数  int flock(int fd,int operation);
函数说明  flock()会依参数operation所指定的方式对参数fd所指的文件做各种锁定或解除锁定的动作。此函数只能锁定整个文件，无法锁定文件的某一区域。
参数  operation有下列四种情况:

* LOCK_SH 建立共享锁定。多个进程可同时对同一个文件作共享锁定。
* LOCK_EX 建立互斥锁定。一个文件同时只有一个互斥锁定。
* LOCK_UN 解除文件锁定状态。
* LOCK\_NB 无法建立锁定时，此操作可不被阻断，马上返回进程。通常与LOCK\_SH或LOCK\_EX 做OR(|)组合。

&ensp;单一文件无法同时建立共享锁定和互斥锁定，而当使用dup()或fork()时文件描述词不会继承此种锁定。
返回值  返回0表示成功，若有错误则返回-1，错误代码存于errno。
 
&emsp;&emsp;flock只要在打开文件后，需要对文件读写之前flock一下就可以了，用完之后再flock一下，前面加锁，后面解锁。其实确实是这么简单，但是前段时间用的时候发现点问题，问题描述如下：
一个进程去打开文件，输入一个整数，然后上一把写锁（LOCK＿EX），再输入一个整数将解锁（LOCK＿UN），另一个进程打开同样一个文件，直接向文件中写数据，发现锁不起作用，能正常写入（我此时用的是超级用户）。google了一大圈发现flock不提供锁检查，也就是说在用flock之前需要用户自己去检查一下是否已经上了锁，说明白点就是读写文件之前用一下flock检查一下文件有没有上锁，如果上锁了flock将会阻塞在那里(An attempt to lock the file using one of these file descriptors may be denied by a lock that the calling process has already placed via another descriptor ).

### linxu c语言 fcntl函数说明

功能描述：根据文件描述词来操作文件的特性。 
文件控制函数
&ensp;fcntl -- file control


头文件：
&ensp;#include \<fcntl.h\>; 
&ensp;int fcntl(int fd, int cmd); 
&ensp;int fcntl(int fd, int cmd, long arg); 
&ensp;int fcntl(int fd, int cmd, struct flock *lock); 

[描述]
&ensp;Fcntl()针对(文件)描述符提供控制.参数fd是被参数cmd操作(如下面的描述)的描述符.
&ensp;针对cmd的值,fcntl能够接受第三个参数int arg
fcntl函数有5种功能： 

1. 复制一个现有的描述符（cmd=F_DUPFD）. 
2. 获得／设置文件描述符标记(cmd=F\_GETFD或F_SETFD). 
3. 获得／设置文件状态标记(cmd=F\_GETFL或F_SETFL). 
4. 获得／设置异步I/O所有权(cmd=F\_GETOWN或F_SETOWN). 
5. 获得／设置记录锁(cmd=F\_GETLK,F\_SETLK或F_SETLKW).

cmd值： 
&ensp;F_DUPFD            返回一个如下描述的(文件)描述符:

* 最小的大于或等于arg的一个可用的描述符
* 与原始操作符一样的某对象的引用
* 如果对象是文件(file)的话,返回一个新的描述符,这个描述符与arg共享相同的偏移量(offset)
* 相同的访问模式(读,写或读/写)
* 相同的文件状态标志(如:两个文件描述符共享相同的状态标志)
* 与新的文件描述符结合在一起的close-on-exec标志被设置成交叉式访问execve(2)的系统调用
     
    
&ensp;F\_GETFD            取得与文件描述符fd联合close-on-exec标志,类似FD_CLOEXEC.如果返回值和FD\_CLOEXEC进行与运算结果是0的话,文件保持交叉式访问exec(),否则如果通过exec运行的话,文件将被关闭(arg被忽略)
&ensp;F\_SETFD            设置close-on-exec旗标。该旗标以参数arg的FD_CLOEXEC位决定。     
&ensp;F_GETFL            取得fd的文件状态标志,如同下面的描述一样(arg被忽略)
&ensp;F\_SETFL            设置给arg描述符状态标志,可以更改的几个标志是： O\_APPEND，O\_NONBLOCK，O\_SYNC和O_ASYNC。
&ensp;F_GETOWN             取得当前正在接收SIGIO或者SIGURG信号的进程id或进程组id,进程组id返回成负值(arg被忽略)
&ensp;F_SETOWN            设置将接收SIGIO和SIGURG信号的进程id或进程组id,进程组id通过提供负值的arg来说明,否则,arg将被认为是进程id
&ensp;命令字(cmd)F\_GETFL和F_SETFL的标志如下面的描述:

* O_NONBLOCK            非阻塞I/O;如果read(2)调用没有可读取的数据,或者如果write(2)操作将阻塞,read或write调用返回-1和EAGAIN错误
* O\_APPEND                    强制每次写(write)操作都添加在文件大的末尾,相当于open(2)的O_APPEND标志
* O_DIRECT                    最小化或去掉reading和writing的缓存影响.系统将企图避免缓存你的读或写的数据.如果不能够避免缓存,那么它将最小化已经被缓存了的数据造成的影响.如果这个标志用的不够好,将大大的降低性能
* O_ASYNC                    当I/O可用的时候,允许SIGIO信号发送到进程组,例如:当有数据可以读的时候

&ensp;在修改文件描述符标志或文件状态标志时必须谨慎，先要取得现在的标志值，然后按照希望修改它，最后设置新标志值。不能只是执行F\_SETFD或F_SETFL命令，这样会关闭以前设置的标志位。
fcntl的返回值 与命令有关。如果出错，所有命令都返回－1，如果成功则返回某个其他值。下列三个命令有特定返回值：F\_DUPFD,F\_GETFD,F\_GETFL以及F_GETOWN。第一个返回新
的文件描述符，第二个返回相应标志，最后一个返回一个正的进程ID或负的进程组ID。
控制fd的例程 

/**三个存取方式标志(O\_RDONLY,O\_WRONLY,以及O\_RDWR)并不各占1位。(这三种标志的值各是0、1和2，由于历史原因。这三种值互斥—一个文件只能有这三种值之一。)因此首先必须用屏蔽字O_ACCMODE取得存取方式位，然后将结果与这三种值相比较。
****/

switch(var & O_ACCMODE) 
{ 
case O_RDONLY : cout<<"Read only.."<
.获得／设置记录锁的功能： (cmd=F\_GETLK,F\_SETLK或F_SETLKW).
&ensp;F\_GETLK          通过第三个参数arg(一个指向flock的结构体)取得第一个阻塞lock description指向的的锁.取得的信息将覆盖传到fcntl()的flock结构的信息.如果没有发现能够阻止本次锁(flock)生成的锁,这个结构将不被改变,除非锁的类型被设置成F_UNLCK.  
&ensp;F\_SETLK          按照指向结构体flock的指针的第三个参数arg所描述的锁的信息设置或者清除一个文件segment锁.F\_SETLK被用来实现共享(或读)锁 (F\_RDLCK)或独占(写)锁(F\_WRLCK),同样可以去掉这两种锁(F_UNLCK).如果共享锁或独占锁不能被设置,fcntl()将立即返回EAGAIN.
&ensp;F\_SETLKW          除了共享锁或独占锁被其他的锁阻塞这种情况外,这个命令和F\_SETLK是一样的.如果共享锁或独占锁被其他的锁阻塞,进程将等待直到这个请求能够完成.当fcntl()正在等待文件的某个区域的时候捕捉到一个信号,如果这个信号没有被指定SA_RESTART,fcntl将被中断.
&ensp;当一个共享锁被set到一个文件的某段的时候,其他的进程可以set共享锁到这个段或这个段的一部分.共享所阻止任何其他进程set独占锁到这段保护区域的任何部分.如果文件描述符没有以读的访问方式打开的话,共享锁的设置请求会失败

&ensp;独占锁阻止任何其他的进程在这段保护区域任何位置设置共享锁或独占锁.如果文件描述符不是以写的访问方式打开的话,独占锁的请求会失败

结构体flock的指针 ：

```
struct flcok 
{ 
short int l_type; /* 锁定的状态*/
//这三个参数用于分段对文件加锁，若对整个文件加锁，则：l_whence=SEEK_SET,l_start=0,l_len=0;
short int l_whence;/*决定l_start位置*/ 
off_t l_start; /*锁定区域的开头位置*/ 
off_t l_len; /*锁定区域的大小*/

pid_t l_pid; /*锁定动作的进程*/ 
};

l_type 有三种状态: 
F_RDLCK 建立一个供读取用的锁定 
F_WRLCK 建立一个供写入用的锁定 
F_UNLCK 删除之前建立的锁定

l_whence 也有三种方式: 
SEEK_SET 以文件开头为锁定的起始位置。 
SEEK_CUR 以目前文件读写位置为锁定的起始位置 
SEEK_END 以文件结尾为锁定的起始位置。 
```
**对文件锁部分的相关说明及使用方法**

* 当写文件的时候首先给相应的文件加锁:

1. 定义flock结构体
2. 加锁指定cmd为：  F_SETLK 
3. 写数据
4. 解锁指定cmd为：F_SETLKW 
 
* 当有其他线程写同一文件的时候： 

1. 判断文件是否已经被加锁指定cmd为：F\_GETLK，获取flock结构体，查看flock.l_type; /* 锁定的状态*/
2. 判断是否加锁，加锁就提示，没加锁继续....
2. 加锁指定cmd为：  F_SETLK 
3. 写数据
4. 解锁指定cmd为：F_SETLKW 

### lockf() 使用方法

```
#include <unistdh.>
int lockf(fd,cmd,size)
int fd,cmd;
long size;
```
这个方法对知识锁定文件而言是最简单的

首先再看其function prototype:
&ensp;int lockf(fd,cmd,size)
&ensp;int fd,cmd;
&ensp;long size;
&ensp;其中fd为文件的handle number ，cmd 为所要使用lockf的功能指定，size則是执行这个功能所要影响的范围大小。

```
#define F_ULOCK 0   /* Unlock a previously locked section */
#define F_LOCK  1   /* lock a section for exclusive use */
#define F_TLOCK 2   /* test and lock a section(non-blocking) */
#define F_TEST  3   /* test section for other process' locks */
```
 其中F_TEST并没有lock的功能，其主要目的只是在测试文件是否正被其它 process
锁住而已，当已经被锁住时，lockf 返回-1，否則返回0 ；
而F\_LOCK和F\_TLOCK 的功能也是将文件锁住，不同之处在于如果这个文件原先已被其它process 锁上时，F\_LOCK会block 住，并等候此文件直至被解除锁定为止，至于F_TLOCK 则并不会将自己 block
住，而会马上返回-1，至于F_TLOCK 等不等于下列程序呢？
        while (lockf(fd,F_TEST,0L)==-1) {}
        lockf(fd,F_LOCK,0L);
结果当然是不等于了！因为这就必须考虑到critical section等问题了！当然，对于一个单纯的系统而言，F\_LOCK便可以达到目的，但对于大系统而言，由于许多难以预测的因素，如果某process 沒有把文件unlock，则使用F_LOCK便将因为等不到文件可以上锁而使整个系统陷入starvation，這是程序审计者必需先考虑清楚的风险！

那么lockf 的最后一个参数size又要如何使用呢？其实size的值指的就是从文件目前的读取位置开始，要往前或往后多少位置做lock或unlock的动作，而假若是要对整个文件做lock或unlock的话，则并不需要特別去量测size大小，只要直接输入0 即可，lockf 便会知道所要影响的范围为整个文件了！



{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载: ](blog.163.com/cupidove/blog/static/1005662)
{% endhighlight %}

