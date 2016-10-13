---
layout: post
title:  "内核使用硬件ip的dma，dma_alloc_coherent 与 dma_alloc_writecombine"
date:   2016-10-13 10:09:35
categories: [编程, Linux, Kernel]
tags: [Linux, Kernel, ]
description: ""
---

内核的dma一般在平台初始化的时候已经分配好了。但是对于一些有内部dma的硬件ip，比如usb ip、video加速ip，他们可能由ip厂商封装好的，没办法绑定到cpu端，这时候在内核使用dma就要注意了，因为dma只认识物理地址哦。

当然，办法还是有的，look

这两天在做 DMA 相关开发， 遇到一对分配 dma buffer 的函数，dma_alloc_coherent 与 dma_alloc_writecombine。 不知其区别。 google 一下也没有得到信息。只好自己看代码。

原来 dma_alloc_coherent 在 arm 平台上会禁止页表项中的 C （Cacheable） 域以及 B (Bufferable)域。

而 dma_alloc_writecombine 只禁止 C （Cacheable） 域.

#define pgprot_noncached(prot)  __pgprot(pgprot_val(prot) & ~(L_PTE_CACHEABLE | L_PTE_BUFFERABLE))
#define pgprot_writecombine(prot) __pgprot(pgprot_val(prot) & ~L_PTE_CACHEABLE)

进一步查找 ARM 书籍， 原来 C 代表是否使用高速缓冲存储器， 而 W 代表是否使用写缓冲区。

这样， dma_alloc_writecombine  分配出来的内存不使用缓存，但是会使用写缓冲区。

而 dma_alloc_coherent 则二者都不适用。


由此，再去理解 LDD3上讲解的一致性 DMA映射 与流式 DMA 映射就比较容易了，一致性 DMA映射(dma_alloc_coherent )调用的是上面的函数， 由于关闭了 cache/buffer,性能自然比较低。 而流式则通过复杂的同步机制，没有付出性能的代价。

所以我们要尽量使用流式DMA来编程。


再去看看 mmc驱动里使用的分散/聚集映射，原来也是一种流式 DMA映射。


【补充】

讲了那么多没用，总结下用法：

1、页对齐内存大小：dma_map_size = PAGE_ALIGN(MY_DATA_SIZE + PAGE_SIZE);

MY_DATA_SIZE是你想分配的大小.

2、调用
A =dma_alloc_writecombine(B,C,D,GFP_KERNEL);

含义：

A: 内存的虚拟起始地址，在内核要用此地址来操作所分配的内存

B: struct device指针，可以平台初始化里指定，主要是dma_mask之类，可参考framebuffer

C: 实际分配大小，传入dma_map_size即可

D: 返回的内存物理地址，dma就可以用。

所以，A和D是一一对应的，只不过，A是虚拟地址，而D是物理地址。对任意一个操作都将改变缓冲区内容。当然要注意操作环境。
 
{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载: ](http://blog.csdn.net/zjujoe/archive/2009/05/15/4189612.aspx)
{% endhighlight %}

