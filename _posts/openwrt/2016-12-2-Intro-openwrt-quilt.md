---
layout: post
title: openwrt quilt学习笔记 
category : Linux
tagline: 
tags : Linux, OpenWrt
---

# 安装

quilt 是用来管理代码树中的 patch 的, 嵌入式内核开发利器!
请通过软件包管理器安装 quilt. 然后写入以下配置文件到 ~/.quiltrc

```
QUILT\_DIFF_ARGS="--no-timestamps --no-index -pab --color=auto"
QUILT\_REFRESH_ARGS="--no-timestamps --no-index -pab"
QUILT\_PATCH_OPTS="--unified"
QUILT\_DIFF_OPTS="-p"
EDITOR="vi"
```

最后一句EDITOR="xxx",xxx是你自己喜欢的编辑器

# 准备工作

假定 ~/openwrt 是你的开发目录, 建议直接在 git branch 下开发.
假设之前已经编译过 bin, 有完整的 .config 和 toolchain.
建立开发 branch: (git 教程自行脑补)

    ~/openwrt$ git checkout -b add-tl-mr10u-support 

清理 tmp 目录!!!!!! 这个是大个坑. 直接删除.

    ~/openwrt$ rm -rvf tmp/ 

修改 OpenWrt 代码
进入设备硬件目录:

    ~/openwrt$ cd target/linux/ar71xx 

修改以下文件, 里面涉及到 TL-MR10U 固件中设备ID的部分, 是 0x00100101, 这个值从 tp-link 官方网站下载的固件中可以获得.

```
    - base-files/etc/diag.sh
    - base-files/etc/uci-defaults/02_network
    - base-files/lib/ar71xx.sh
    - base-files/lib/upgrade/platform.sh
    - config-3.8
    - generic/profiles/tp-link.mk
    - image/Makefile
```

新建文件: files/arch/mips/ath79/mach-tl-mr10u.c , 内容参考 TL-WR703N 设备的文件 mach-tl-wr703n.c, 修改所有出现 wr703n, WR703N 等等大小写混合的部分, emacs 无痛苦完成

# 添加 Linux patch

这里就完成了 OpenWrt 的设备支持代码. 为了支持我们的设备, Linux 代码树的部分文件也需要做改动, OpenWrt 采用了 patch 的方式实现.
回退到根目录 ~/openwrt .
清理并准备 patch 树:

    ~/openwrt$ make target/linux/{clean,prepare} # 后面可加 V=s QUILT=1 参数, 表示静默无输出 

进入内核代码目录(其中版本号可能与你的不一致):

    ~/openwrt$ cd build_dir/target-mips_r2_uClibc-0.9.33.2/linux-ar71xx_generic/linux-3.8.12/ 

这里就是内核代码树了, 里面的代码是已经打过所有 patch 的, 可以用 quilt push 检查看是不是这样:

    $ quilt push
    File series fully applied, ends at patch platform/902-unaligned\_access_hacks.patch 

这条输入也告诉我们, 当前最顶的 patch 是 platform/902 (这个是坑啊, 官方文档不带 platform 前缀, 是错的).
为我们的 TL-MR10U 新建个 patch:

    $ quilt new platform/920-add-tl-mr10u-support.patch 

选择的数字需要大于刚才的那个 902, 然后 quilt 会自动把这个 patch 设置为当前 patch, 所有的改动都针对这个 patch.
然后就是增加代码了

```
$ quilt edit arch/mips/ath79/Kconfig
$ quilt edit arch/mips/ath79/Makefile
$ quilt edit arch/mips/ath79/machtypes.h 
```

至于怎么改, 参考这些文件里其他硬件的配置, 基本上说 copy TL-WR703N 的就可以了. 保证不重不漏.
然后验证下修改的内容:

```
$ quilt diff # 查看 diff
$ quilt refresh # 保存所有 diff 到 patch 文件 
```

这个时候我们的 patch 文件还在 build_dir 里, 大概位置是 patches/platform/ 下. 需要同步到 OpenWrt 代码树.
退回到顶层工作目录, 执行:

    ~/openwrt$ make target/linux/update V=s 

同步完成后, patch 文件会出现在 target/linux/ar71xx/patches-3.8/ 下.

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* 转载: [http://blog.chinaunix.net/uid-22547469-id-5163307.html]
{% endhighlight %}

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
