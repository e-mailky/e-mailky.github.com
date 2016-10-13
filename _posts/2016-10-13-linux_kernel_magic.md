---
layout: post
title:  "魔数常量"
date:   2016-10-13 17:09:35
categories: [编程, Linux, Kernel]
tags: [Linux, Kernel, ]
description: ""
---

	unsigned long hash_long(unsigned long val, unsigned int bits) 
    { 
        unsigned long hash=val *0x9e370001UL; 
        return hash>>(32-bits); 
    } 

0x9e370001=2 654 404 609=2^31+2^29-2^25+2^22-2^19-2^16+1. 
是接近黄金比例的2^32的一个素数。（也称为 “魔数常量”) 

也许你会想常量 0x9e370001（=2654 404 609）究竟是怎么得出来的。这种散列函数是基于表索引乘于一个适当的大数，于是结果溢出，就把留在32位变量中的值作为模数操作的结果。Knuth建议，要得到满意的结果，将2^32做黄金分割，这个大数是最接近黄金分割的素数。

2654 404 609就是接近2^32*（5开方－1）／2的一个素数，这个数可以方便地通过加运算和位移运算得到，因为等于：2^31+2^29-2^25+2^22-2^19-2^16+1.


{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载: ](blog.163.com/cupidove/blog/static/1005662)
{% endhighlight %}

