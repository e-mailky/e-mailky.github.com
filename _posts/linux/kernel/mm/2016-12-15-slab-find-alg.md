---
layout: post
title:  内存伙伴系统之查找伙伴算法
categories: [Linux]
tags: [Linux, Kernel, 内存管理]
description: ""
---

&emsp;&emsp;&emsp;&emsp;Linux内核中查找伙伴的算法由十分简单的代码构成。

&emsp;&emsp;&emsp;&emsp;如图所示，给定page\_idx时，查看page\_idx的伙伴位置。

![figure1](/images/kernel/MM/slab-1.png)


&emsp;&emsp;&emsp;&emsp;将内存以2^{MAX\_ORDER}单位分割时，page\_idx是2^{MAX\_ORDER}个页帧内的索引。
而对应于page_idx的页将成为相应顺序的第一页，因为统一或查找伙伴时，将该顺序的第一页作为对象以检查顺序。
其余页不具备顺序信息（page->private）。

&emsp;&emsp;&emsp;&emsp;如上图所示，以page\_idx为基准，距-2^{order}大小的①或者距+2^{order}
大小的②将成为page_idx的伙伴。

&emsp;&emsp;&emsp;&emsp;当给定page_idx值时，该如何选择伙伴呢？

&emsp;&emsp;&emsp;&emsp;该答案可以通过page\_idx值确定。由于page\_idx是顺序的第一个页帧号，
所以page_idx值将是顺序的倍数。

&emsp;&emsp;&emsp;&emsp;用位表现的话，在page\_idx值中对应于下级顺序的位被置为顺序值或变成0。
例如，order=2时，对应于page_idx的顺序值将成为下列二者之一。

&emsp;&emsp;&emsp;&emsp;**page_idx = 0b?????????????000          2^{order} = 2^{2}=0b100**

&emsp;&emsp;&emsp;&emsp;**page_idx = 0b?????????????100          2^{order} = 2^{2}=0b100**

&emsp;&emsp;&emsp;&emsp;在page_idx值中，对应于顺序的位均为0时，需要与距+2^(order)的伙伴统一；
对应于顺序的下级位与顺序相同时，需要与距-2^{order}的前方的伙伴统一。

&emsp;&emsp;&emsp;&emsp;总结下：

&emsp;&emsp;&emsp;&emsp;如果对应于page\_idx值顺序下级位与顺序相同，则减少该顺序大小；如果下级为0，
则增加该顺序大小。执行该任务的就是XOR运算。例如，order=2时，按如下所示查找对应于2种page_idx的伙伴。

&emsp;&emsp;&emsp;&emsp;0b?????????????000的伙伴    = 0b?????????????000 ^ 0b100

                                                 = 0b?????????????100

&emsp;&emsp;&emsp;&emsp;0b?????????????100的伙伴    = 0b?????????????100 ^ 0b100

                                                  = 0b?????????????000

&emsp;&emsp;&emsp;&emsp;对应的代码如下：

mm/page_alloc.c
```
/* Authentication algorithms */
buddy_idx = __find_buddy_index(page_idx, order);
buddy = page + (buddy_idx - page_idx);
```
```
static inline unsigned long
__find_buddy_index(unsigned long page_idx, unsigned int order)
{
	return page_idx ^ (1 << order);
}
```
举两个栗子：

- page_idx=1且order=0时
- page_idx=8且order=1时


![figure2](/images/kernel/MM/slab-2.png)

[visio图](/images/kernel/MM/slab.vsd)

参考:

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [内存伙伴系统之查找伙伴算法]
{% endhighlight %}

[内存伙伴系统之查找伙伴算法](http://rick_stone.leanote.com/post/%E5%86%85%E5%AD%98%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E4%B9%8B%E6%9F%A5%E6%89%BE%E4%BC%99%E4%BC%B4)


[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
