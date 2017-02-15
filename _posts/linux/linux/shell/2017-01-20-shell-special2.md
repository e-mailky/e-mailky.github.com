---
layout: post
title:  Bash参数展开(Parameter Expansion)
categories: [Linux]
tags: [Linux, Shell]
description: ""
---


1. ${parameter} 取parameter的值
2. ${parameter:-word} 如果parameter为空，则用word的值做parameter的缺省值
3. ${parameter:=word} 在2的基础上，把word的值赋给parameter
4. ${parameter?=word} 如果parameter为空，word作为错误信息输出。
5. ${parameter+=word} 在parameter不为空的情况下，输出word的值。
6. ${parameter:offset} parameter的从第offset个字符开始的substring

    ${parameter:offset:length} parameter的从第offset个字符开始的，长度为length的substring

7. ${!prefix*} 所有的以prefix开始的变量名的展开，由IFS分隔

    ${!prefix@}

8. ${!name[@]}

    ${!name[*]}

  如果name为一个数组变量，那么结果是该数组的所有下标的列表。如果name不是数组，那么，如果name为空，
  结果就为空，如果name不为空，结果就为0.

9. ${#parameter} 取parameter的长度为值
10. ${parameter#word} 从前最短匹配
  
  ${parameter##word} 最长匹配
  
  在这里word是一个模式(pattern), 如果parameter的开始匹配word模式，那么第一个的结果是最短匹配，第二个的结果是最长匹配
11. ${parameter%word} 从后最短匹配
  
  ${parameter%%word} 最长匹配

  在这里word也是一个模式，如果parameter的结尾匹配word模式，那么第一个的结果是最短匹配，第二个的结果是最长匹配

12. ${parameter/pattern/string}

  ${parameter//pattern/string}
  
  在这里pattern也是一个模式，parameter展开后最长匹配的部分被string替换。第一种情况只替换首次匹配，第二种情况替换所有匹配。

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}


[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
