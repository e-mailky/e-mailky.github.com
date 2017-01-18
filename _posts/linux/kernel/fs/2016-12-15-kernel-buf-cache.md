---
layout: post
title:  buffer和cache使用率降低的方法（page cache）
categories: [Linux]
tags: [Linux, Kernel, FS]
description: ""
---

**本文基于内核4.2**

## 问题描述
&emsp;&emsp;&emsp;&emsp;长时间运行的服务容易出现cache或者buffer占用内存过多的问题；
另外，在嵌入式设备中，内存相对较少，如果出现大量数据的拷贝，很容易出现拷贝进程被oops杀死的情况。

&emsp;&emsp;&emsp;&emsp;cache：内核用于读优化。简而言之，我们调取read函数，
内核会先查找cache中是否存在该数据，如果命中，那么read系统调用将直接拷贝cache中的数据；
如果没有命中，内核再将数据从存储介质中读出，放入cache。

&emsp;&emsp;&emsp;&emsp;buffer：内核用于写优化。
数据先写入page cache，然后直接返回；由后备存储线程负责数据的刷回操作。

> 注：用户空间看到的buffer和cache对内核而言都是page cache，即address_space；只不过操作函数和使用目的不同而已。

## 具体操作
### 读操作 
&emsp;&emsp;&emsp;&emsp;读取的过程可以参考[这里](http://leanote.com/blog/post/56bfedbbab64416c54004a76) 
中的最后一幅流程图部分

&emsp;&emsp;&emsp;&emsp;在流程图中是裸块设备操作，
因此涉及到的操作都在fs/block\_dev.c中，vfs\_read调用的f->op->read操作为blkdev\_read\_iter
（因为f_op->read在block文件系统中未定义）

```
const struct file_operations def_blk_fops = {
    ……
	.read_iter	= blkdev_read_iter,
	.write_iter	= blkdev_write_iter,
    ……
}; 
```
&emsp;&emsp;&emsp;&emsp;在blkdev\_read\_iter之后，除去一些访问的检查，最终将调用do\_generic\_file_read。

&emsp;&emsp;&emsp;&emsp;do\_generic\_file_read实现查找cache的缓存，即利用基数树进行查找；
正如之前所言，这里会产生两条分支：

* 未找到相应的page cache，那么调用readpage，将数据读入缓存；这里的操作大同小异，
最终目的都是将page cache转换成bio，加入到设备的I/O请求队列中。
* 找到相应的page cache，则将数据拷贝至用户空间copy\_page\_to_iter

> 注：在调用readpage时，page页面已经被lock（lock\_page\_killable）；在提交io之后，
PageUptodate将判断页面是否被读出，如果仍未读出，将再调用lock\_page\_killable，使读操作进入阻塞。

&emsp;&emsp;&emsp;&emsp;PageUptodate的尝试次数将有三次。若三次均未被读出，那么返回失败。

```
static const struct address_space_operations def_blk_aops = {
	.readpage	= blkdev_readpage,
	.readpages	= blkdev_readpages,
	.writepage	= blkdev_writepage,
	.write_begin	= blkdev_write_begin,
	.write_end	= blkdev_write_end,
	.writepages	= generic_writepages,
    ……
};
```
&emsp;&emsp;&emsp;&emsp;以上是读操作的大致过程。

### 写操作
&emsp;&emsp;&emsp;&emsp;与读类似，写操作调用\_\_generic\_file\_write\_iter，如果是bio，
那么该函数会调用generic\_perform\_write，该函数中会调用page cache的方法write\_begin和writepages，
以及balance\_dirty\_pages_ratelimited。

&emsp;&emsp;&emsp;&emsp;最终数据是否同步由函数balance\_dirty\_pages\_ratelimited决定，
该函数适用于所有的文件系统。

&emsp;&emsp;&emsp;&emsp;balance\_dirty\_pages\_ratelimited中，
其主要函数为balance\_dirty\_pages。该函数主要涉及以下结构体，
该结构体控制了page cache刷回的阈值。

```
struct dirty_throttle_control {
#ifdef CONFIG_CGROUP_WRITEBACK
	struct wb_domain	*dom;
	struct dirty_throttle_control *gdtc;	/* only set in memcg dtc's */
#endif
	struct bdi_writeback	*wb;
	struct fprop_local_percpu *wb_completions;
 
	unsigned long		avail;		/* dirtyable */
	unsigned long		dirty;		/* file_dirty + write + nfs */
	unsigned long		thresh;		/* dirty threshold */
	unsigned long		bg_thresh;	/* dirty background threshold */
 
	unsigned long		wb_dirty;	/* per-wb counterparts */
	unsigned long		wb_thresh;
	unsigned long		wb_bg_thresh;
 
	unsigned long		pos_ratio;
};
```
## 方法
&emsp;&emsp;&emsp;&emsp;（待完整）
&emsp;&emsp;&emsp;&emsp;


{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [buffer和cache使用率降低的方法（page cache）]
{% endhighlight %}

[buffer和cache使用率降低的方法（page cache）](http://rick_stone.leanote.com/post/buffer%E5%92%8Ccache%E4%BD%BF%E7%94%A8%E7%8E%87%E9%99%8D%E4%BD%8E%E7%9A%84%E6%96%B9%E6%B3%95)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
