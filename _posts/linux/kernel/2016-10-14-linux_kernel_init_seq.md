---
layout: post
title:  "内核初始化过程中的调用顺序"
date:   2016-10-14 17:09:35
categories: [Linux]
tags: [Linux, Kernel, ]
description: ""
---

&emsp;&emsp;所有的\_\_init函数在区段.initcall.init中还保存了一份函数指针，在初始化时内核会通过这些函数指针调用这些__init函数指针，并在整个初始化完成后，释放整个init区段（包括.init.text，.initcall.init等）。
**注意** &emsp;这些函数在内核初始化过程中的调用顺序只和这里的函数指针的顺序有关，和1）中所述的这些函数本身在.init.text区段中的顺序无关。在2.4内核中，这些函数指针的顺序也是和链接的顺序有关的，是不确定的。在2.6内核中，initcall.init区段又分成7个子区段，分别是

```
.initcall1.init  
.initcall2.init  
.initcall3.init  
.initcall4.init  
.initcall5.init  
.initcall6.init  
.initcall7.init
```
当需要把函数fn放到.initcall1.init区段时，只要声明

	core_initcall(fn);

即可。

其他的各个区段的定义方法分别是：
```
core_initcall(fn) --->.initcall1.init  
postcore_initcall(fn) --->.initcall2.init  
arch_initcall(fn) --->.initcall3.init  
subsys_initcall(fn) --->.initcall4.init  
fs_initcall(fn) --->.initcall5.init  
device_initcall(fn) --->.initcall6.init  
late_initcall(fn) --->.initcall7.init
```

&emsp;&emsp;而与2.4兼容的initcall(fn)则等价于device_initcall(fn)。各个子区段之间的顺序是确定的，即先调用.initcall1.init中的函数指针，再调用.initcall2.init中的函数指针，等等。**而在每个子区段中的函数指针的顺序是和链接顺序相关的，是不确定的。**
&emsp;&emsp;在内核中，不同的init函数被放在不同的子区段中，因此也就决定了它们的调用顺序。这样也就解决了一些init函数之间必须保证一定的调用顺序的问题。按照include/linux/init.h文件所写的，我在驱动里偿试了这样两种方式：

    __define_initcall("7", fn); 
    late_initcall(fn);

都可以把我的驱动调整到最后调用。实际上上面两个是一回事：

    #define late_initcall(fn) __define_initcall("7", fn)

linux 2.6中把initcall又分成了若干种类，主要用来区别不同的initcall的调用次序，由于initcall中的调用次序是随机的，所以不能保证某些重要的初始化先运行。

* pure_initcall：最先运行的，不依赖于任何其他初始化函数。
* core_initcall
* core\_initcall_sync
* postcore_initcall
* postcore\_initcall_sync
* arch_initcall
* arch\_initcall_sync
* subsys_initcall
* subsys\_initcall_sync
* fs_initcall
* fs\_initcall_sync
* rootfs_initcall
* device_initcall
* device\_initcall_sync
* late\_initcall
* late\_initcall_sync

&emsp;&emsp;可以看一下x86中pci初始化的代码，首先是pci\_access\_init，这个函数是arch\_initcall的，然后是
pci\_legacy\_init，这个函数是subsys\_initcall的，然后是pcibios\_irq_init，这个函数也是
subsys\_initcall的，然后是pcibios\_assign\_resources这个函数是fs\_initcall的，最后才是
pci\_init，这个函数是device\_initcall的，这样就把整个pci初始化过程分开了。


{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载: ](http://www.cnblogs.com/hoys/archive/2012/02/22/2363399.html)
{% endhighlight %}

