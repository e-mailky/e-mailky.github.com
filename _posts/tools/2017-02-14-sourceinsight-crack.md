---
layout: post
title:  破解Source Insight 3.5.0072过程(附：安装软件+注册机)
categories: [工具软件]
tags: [Dev, 破解,]
description: ""
---

注册机及软件下载[地址](http://download.csdn.NET/detail/huhu1544/5330869)

&emsp;&emsp;&emsp;&emsp;上几周灰春哥说在试着破解Source Insight 3.5，一直拿CrackMe做实验的俺也
不免手痒来练练手(顺便拿去吾爱混个邀请码)，刚拖进OD里就看见了GetTickCount函数，还以为里面有反调试呢
(现在想想估计是检查试用天数用的)，当时反反调试还没怎么修炼，就搁置了几天，后来看了几天脱壳常用的
反调试手段就开始动手了，废话不说了，看分析吧。

&emsp;&emsp;&emsp;&emsp;首先运行查看错误提示：

![图1](/images/others/20130504203933761.png)

&emsp;&emsp;&emsp;&emsp;用OD的字符串搜索来查一下“You typed”：

![图2](/images/others/20130504204024334.png)

&emsp;&emsp;&emsp;&emsp;很明显，双击查看代码，向上翻找到关键Call(00408BD9)：

![图3](/images/others/20130504204057223.png)

&emsp;&emsp;&emsp;&emsp;进去这个Call里瞅一下：

![图4](/images/others/20130504204127551.png)

&emsp;&emsp;&emsp;&emsp;先看00448f3a处的第一个Call 00448e53(事实证明这里很可能是一个大坑呀！！！，
被坑了N个小时才出去)，直接上IDA看流程图，比较明了(我已经加上了注释)：

![图5](/images/others/20130504204318922.png)

&emsp;&emsp;&emsp;&emsp;到底跳到哪一个呢？本来想eax最终应该返回1的，
大致看了几眼sub\_449030的内容，跟真的似的，就让sub\_448e53返回1进入sub\_449030跟踪看看，
怎么让sub_448e53函数返回1呢？看代码：

![图6](/images/others/20130504204407608.png)

&emsp;&emsp;&emsp;&emsp;还要继续看sub\_1602，没办法，继续看sub_1602吧(虽然这里是个坑，
但是这个函数后面用到的多，可以看一下代码)：

![图7](/images/others/20130504204503028.png)

&emsp;&emsp;&emsp;&emsp;这段代码很简单，ES3US是参数，输入的前五个字符和ES3US的大写相同的话就返回EAX=1，
就输入前五个是ES3US的假码进入上面提到的sub\_449030看看(最初是在OD里看的，
花费了很多时间才意识到可能是进坑了，只截取部分代码大致看看就行了)：

![图8](/images/others/20130504204542126.png)

&emsp;&emsp;&emsp;&emsp;执行完sub\_448e7e之后返回值应该为0才能继续下面的验证，
但是最后剩余的其它函数执行完毕后返回1才注册成功，这个sub\_448e7e函数内容里是对输入的字符串的各种验证，
符合验证的返回值才是1，与这个函数要求的返回值0矛盾呀，也就是说可以说如果输入任何不满足sub\_448e7e
函数里面验证的假验证码都可以返回0，继续下面的剩下的几个函数的验证，还有好多呢，要通过后面的验证用
随意构造的假码就难了，所以估计这整个函数就是个坑(随意构造的注册码能通过sub_448e7e，返回0，
但是后面的验证估计过不去了，如果有不同见解，欢迎讨论交流哈)，这条路估计行不通了，所以继续回去看第一幅图中的函数：

![图9](/images/others/20130504204617619.png)

&emsp;&emsp;&emsp;&emsp;既然不能走sub\_449030这条道，那就看00448f5c处的sub\_448f5c吧，
其实这个函数和sub_449030函数的验证差不多，但是关键之处不一样呀，进去看看：

![图10](/images/others/20130504204654921.png)

&emsp;&emsp;&emsp;&emsp;是不是有点熟悉呢，就是判断前五位的大写是不是SI3US而已，
输入前五位是SI3US的假码的话就让其跳转到Loc_448f89：

![图11](/images/others/20130504204800475.png)

&emsp;&emsp;&emsp;&emsp;又是函数sub_448e7e，上面的提到的坑里的验证函数，
不过这次是符合函数里的各种验证才能返回1才能继续下面的验证，进去看看里面的验证吧
(还是用IDA的流程图看吧，OD的图太长了不好截)：

![图12](/images/others/20130504204859270.png)

&emsp;&emsp;&emsp;&emsp;首先要存在”-”字符才能继续，OD直接修改自己的假码这一位为”-”就行了，
不过还是推荐重来一次，毕竟后面的路还很长，很麻烦额(⊙o⊙)…，继续：

![图13](/images/others/20130504204936589.png)

&emsp;&emsp;&emsp;&emsp;判断刚才的寻找的”-”位置是不是和SI3US挨着(PS:真折腾，
直接调用返回具体位置的函数就行了呗，还非要strchr()再减去基址，莫非是编译器优化了？？？不确定额)才行，继续：

![图14](/images/others/20130504205117469.png)

&emsp;&emsp;&emsp;&emsp;又是这玩意，,截取”-”字符第二次出现的位置，判断中间那段注册码长度是不是6，
中间用了and XX 0指令把”-”字符第二次出现的位置置为0，相当于把字符截断了(最后又补回去了)，
就只剩余中间的那一段部分(最前面的SI3US已经用指针偏移处理掉了，这里只剩下中间的6位注册码)，
输入类似SI3US-XXXXXX-XXX的注册码再继续看：

![图15](/images/others/20130504205204445.png)

&emsp;&emsp;&emsp;&emsp;具体拷贝的啥不记得了，忘记注释了，现在就不记得了，
就记得strlen是求的最后的注册码长度，看长度是不是5，到此注册码的格式就明了了，
就是SI3US-XXXXXX-XXXXX类型，继续看呀，真折腾：

![图16](/images/others/20130504205304396.png)

&emsp;&emsp;&emsp;&emsp;后五位注册码转化成整型值(存进了eax，最终保存进了arg_8的空间里)，
由于atoi函数处理字符串时存在非数字型的字符就停止转化了，第一个字符不是数字返回0，为了保险，
还是把后五位字符串全输入成数字型的，让它全部转化成数字，”12345”就转化成12345，看它还有神马招数，
终于返回1了，别急，后面还有验证呢，毕竟还没验证中间及最后的注册码正确性呀，执行到函数结束，继续Go：

![图17](/images/others/20130504205352902.png)

&emsp;&emsp;&emsp;&emsp;执行到红框处又跳到00449075，接着执行sub_428e8a，跟进去：

![图18](/images/others/20130504205419823.png)

&emsp;&emsp;&emsp;&emsp;这段代码其实没啥用，挺无语的，就是验证中间的六位注册码第一位是不是和后五位相等，
注意红线处的代码挺有意思的，很多编译器在优化选择语句时，尤其是类似a>b?b:c的语句时经常用到类似的代码
（有兴趣的可以看看《C++反汇编与逆向揭秘》，里面很多讲了很多编译器优化的规则）再继续说这段代码的含义呀，
猜测是不是一个小坑，预防像我这种懒人输入中间6位相同的假注册码，太小瞧人了，骗人也来一个稍微高级点的，
6位全相等能骗得了我输入的假码中间6位是”123456”的吗？(下面紧接着的一个还真是稍微高级点的坑，
我的”123456”还真被坑了╮(╯_╰)╭) 继续看，不闲扯了，上面返回eax为1之后跳转到了448fc1：

![图19](/images/others/20130504205616704.png)

&emsp;&emsp;&emsp;&emsp;看00408fea处有1E=30次比较，比较啥呢？53c468处向上30个4字节数据，
其实是将注册码中间6位由atoi转成的整型值与30个值依次比较，有相等的话就调走返回0，最后导致验证失败，
在OD里看53c468处的数据只能看到些16进制数值，使用IDA的计算器可以计算出这些十进制数，
其中第二次与输入的中间6位注册码比较的就是我输入的”123456”的整型值……中招了，真是猥琐的验证，
(不过没关系，把标志位改一下不让它跳走就得了，但是最后写算法注册机时就要注意了，万一生成的注册码里
真的落在了这30个数之中或是6位全相同的就注册不成了)进行30次比较之后，跳进了红框中最后一个验证的函数，
最后一把了，返回值为1就OK了，跟进去：

![图20](/images/others/20130504205652896.png)

![图21](/images/others/20130504205728171.png)

&emsp;&emsp;&emsp;&emsp;别看代码有点乱，实际上验证的算法好水的，写成的伪代码的除去变量定义就剩下
几行计算的代码，拿我输入的SI3US-123456-54321来说吧，令K=123456(数值)，再将”123456”(字符查)与存放的
几个常量(代码中的Temp数组，有10个，只用到6个)异或下，结果和K*4相加的结果再存进K里，进行6次而已，
(好水，算法简单不说，逻辑清楚点也行呀，一共就六次循环，004f3ed3处的循环变量还考虑超过10时重新赋值成0，瞎搞！)。

&emsp;&emsp;&emsp;&emsp;算法明了了，编写注册机吧，看代码：

![图22](/images/others/20130504205911684.png)

&emsp;&emsp;&emsp;&emsp;随便写的，生成6位随机数，计算出后5位(有时候求后面几位时结果可能不是五位，
不能用，就重来直到结果是5位的)，最后Si3US、中间的6位，结果的5位拼接就完了，当然还有很多小问题，
比如随机数生成的没有很大的数，一般没有超过130000，这样也好，结果就很难在那30个比较的数和
全相等的数范围内了，功能差不多就行了，注册机写完刚好12点钟，本来打算把破文写完就睡呢，结果，呵呵……



{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

[破解Source Insight 3.5.0072过程 附：安装软件+注册机](http://blog.csdn.net/qs_hud/article/details/8884867)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help