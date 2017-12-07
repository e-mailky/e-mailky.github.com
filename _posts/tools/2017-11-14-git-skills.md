---
layout: post
title:  git 技巧
categories: [工具软件]
tags: [GNU, SCM, Git]
description: ""
---

## git log

A---B---E---F <==(master)
    \---C---D <==(experimet)

### 两点(..)

表示查看experiment上还没有合并到master的commit，换句话说：所有experiment能读取到但master读取不到的commit对象

```
$ git log master..experiment
D
C
```

反过来就是E、F这两个master分支上的还没有合并到experiment的commit对象

```
$ git log experiment..master
F
E
```

所以，下面就是HEAD本地还没有推送到服务器的commit对象啦

    $ git log origin/master..HEAD

符号..的两边有一边缺失，Git会自动用HEAD代替

    git log origin/master..


![T1](/images/others/git/2995088-0297801e735e0255.png)

这三个命令等价，都是显示test有而master没有的commit，也可以说是没有合并到master的test上的commit
符号^ 和 选项--not都表示不显示存在于该分支上的commit

```
$ git log master..test
$ git log ^master test
$ git log test --not master
```

![T2](/images/others/git/2995088-e1734a8f4cfb4220.png)


使用符号^ 和 选项--not，可以写多个branch，这是..符号做不到的，用这两种符号，
可以查看多个branch，比如上面的命令显示test和master分支的commit且不显示experiment能够触及的commit

$ git log test master --not experiment
$ git log test master ^experiment

![T3](/images/others/git/2995088-628e5b901ab942a3.png)

### 三点(...)

三个点...的符号，表示显示test、master的所有commit，但不显示两个分支共有的commit

   $ git log test...master

![T3](/images/others/git/2995088-69c86f1b2aab499b.png)

这个命令中--left-right，就是用来指出列出的commit所属的分支，图里面可以看到，
每个commit和SHA-1值之间都有>和<箭头，表示commit属于哪一边

   $ git log --left-right test...master

![T4](/images/others/git/2995088-1c0d65dcc9510328.png)

## git add

### -i/--interactive

打开hello.txt，修改为下面的样子

```
1.(master1)
2.(master2)
3.(master3)
4.(master4)
5.
6.
7.
8.
```

打开morning.txt文件，修改为下面的样子

```
1.
4.
5.(master5)
6.
7.
8.
9.(master9)
10.(master10)
```

打开afternoon.txt，修改为下面的样子

```
1.(master1)
2.
3.
4.
5.(master5)
6.
7.
8.
9.
10.(master10)
```


    $ git add -i
    $ git add  --interactive

![T5](/images/others/git/2995088-d4974946cb4acddc.png)

进入交互暂存文件模式，显示的信息比git status详细一点,
左边 staged 显示文在是否在staging（暂存区，也叫Index）进行存储
右边 unstaged path 显示文件被修改的行数和在仓库中的路径
可以看到，afternoon、hello、morning三个文件都有增加或减少啦几行
目前没有一个文件加入Index区域

    2
    u

![T6](/images/others/git/2995088-460933d15552d1a3.png)

输入2或者u，列出已经修改，可以添加到Index的文件

    1,2

![T7](/images/others/git/2995088-c5d6985852f3f026.png)

输入序号，选需要添加到Index的文件
输出信息中，1,2两个文件左边有\*，表示这两个文件被选中，会被stage（就是添加到Index）

    Enter

![T8](/images/others/git/2995088-ef09a1e80f52bc10.png)

直接回车，刚才选中的文件就会被staged，输出信息，显示2个文件被添加啦

    1

![T9](/images/others/git/2995088-5bac50bb621a7022.png)

输入1查看当前状态，左边staged一栏，序号为：1、2的两个文件已经staged
右边unstaged path中，两个文件状态都变成nothing

    3 or r

![T10](/images/others/git/2995088-a7399a20e8de148b.png)

输入3或者r，进入Revert状态，列出文件列表

    1

![T11](/images/others/git/2995088-1815282087452798.png)

输入1将那个文件取消staged状态，输出列表里，被选的文件的序号左边有\*号

    Enter

![T12](/images/others/git/2995088-a2488d724d9d49cc.png)

输出信息，将选中的文件从Index区域移除


&emsp;&emsp;&emsp;&emsp;

    1

![T13](/images/others/git/2995088-beed639bf6b47a7b.png)

然后再选1来查看状态，现在只有hello.txt添加到index区域啦

    6 or d

![T14](/images/others/git/2995088-433659b8724cd07c.jpg)

选择6或者d，列出已经staged的文件

    1

![T15](/images/others/git/2995088-bfe58a8e48b0cca3.png)

选择第一个文件，跟使用 git diff --cached是差不多的，就是查看当前staged的文件
与HEAD所指向branch的最后一个commit中的同一文件进行比较。

### git rebase

### --onto 跨越分支rebase

A---B---C---D <==(master)
    \E--F---G <==(server)
     \I---J <==(client)

    $ git rebase --onto master server client

将C8和C9的修改直接应用到master分支上。
结果如图五所示，注意E的内容不会被应用到master上，只有I和J的修改会。

         (master)
A---B---C---D---I'---J' <==(client)
    \E--F---G <==(server)
     \I---J

![T16](/images/others/git/2995088-a2488d724d9d49cc.png)

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
