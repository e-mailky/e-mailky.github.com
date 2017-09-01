---
layout: post
title:  Makefile调试
categories: [工具软件]
tags: [GNU, Linux, MakeFile]
description: ""
---

## Markdown调试

### warning函数

&emsp;&emsp;&emsp;&emsp;warning函数非常适合用来调试难以捉摸的makefile。因为warning函数会被扩展成空字符串，
所以它可以放在makefile 中的任何地方：开始的位置、工作目标或必要条件列表中以

及命令脚本中。这让你能够在最方便查看变量的地方输出变量的值。例如：

```
$(warning A top-level warning)

FOO := $(warning Right-hand side of a simple variable)bar
BAZ = $(warning Right-hand side of a recursive variable)boo

$(warning A target)target: $(warning In a prerequisite list)makefile
$(BAZ)
 $(warning In a command script)
 ls
$(BAZ):

这会产生如下的输出：

$ make
makefile:1: A top-level warning
makefile:2: Right-hand side of a simple variable
makefile:5: A target
makefile:5: In a prerequisite list
makefile:5: Right-hand side of a recursive variable
makefile:8: Right-hand side of a recursive variable
makefile:6: In a command script
ls
makefile
```

请注意，warning函数的求值方式是按照make标准的立即和延后求值算法。
虽然对BAZ的赋值动作中包含了一个warning函数，但是直到BAZ在必要条件列表中被求值后，这个信息才
会被输出来。

“可以在任何地方安插warning调用”的这个特性，让它能够成为一个基本的调试工具。

### 命令行选项

三个最适合用来调试的命令行选项：
--just-print（-n）
--print-database（-p）
--warn-undefined-variables。

### remake 工具
[remake](https://github.com/rocky/remake)

{% highlight ruby %}
文档信息
--------------
* 作者:
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

[Markdown简明教程](http://www.ruanyifeng.com/blog/2014/06/git_remote.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
