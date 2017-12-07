---
layout: post
title:  史上最全讲解：WiFi知识
categories: [网络]
tags: [TCP/IP, 80211]
description: ""
---

## IE802.11简介

标准号 | 802.11b | 802.11a | 802.11g | 802.11n | 802.11ac | 802.11ax
-----  | -----   | -----   | -----   | -----   | -----   |-----
标准发布时间 | 1999年9月 | 1999年9月 | 2003年6月 | 2009年9月 | 2012年 | 2018年
工作频率范围 | 2.4－2.4835GHz | 5.150－5.350GHz,5.475－5.725GHz,5.725－5.850GHz | 2.4－2.4835GHz ,5.150－5.850GHz | 5.150－5.850GHz | 2.4－2.4835GHz ,5.150－5.850GHz
非重叠信道数 | 3 | 24 | 3 | 15 | 24 | 3
物理速率（Mbps）| 11 | 54 | 54 | 600 | 6933 | 9607
实际吞吐量（Mbps）| 6 | 24 | 24 | 100+ | ~ | ~
频宽 | 20MHz | 20MHz | 20MHz | 20MHz/40MHz | 20MHz/40MHz/80MHz/160MHz | 20MHz/40MHz/80MHz/160MHz
调制方式 | CCK/DSSS | OFDM | CCK/DSSS/OFDM | MIMO-OFDM/DSSS/CCK | MU-MIMO-OFDM/DSSS/CCK | MU-MIMO-OFDM/DSSS/CCK
兼容性 | 802.11b | 802.11a | 802.11b/g | 802.11a/b/g/n | 802.11a/b/g/n/ac | 802.11a/b/g/n/ac/ax


## 频谱划分

2.4G WiFi总共有14个信道，如下图所示：

![T1](/images/networks/80211/337893.jpg)

1. IEEE 802.11b/g标准工作在2.4G频段，频率范围为2.400—2.4835GHz，共83.5M带宽
2. 划分为14个子信道
3. 每个子信道宽度为22MHz
4. 相邻信道的中心频点间隔5MHz 
5. 相邻的多个信道存在频率重叠(如1信道与2、3、4、5信道有频率重叠)
6. 整个频段内只有3个（1、6、11）互不干扰信道

## 接收灵敏度

误码率要求

误码率要求 | 速率 | 最小信号强度
-----  | -----   | -----   
|/8.PER(误码率)不超过8％ | 6Mbps | -82dBm
              9Mbps | -81dBm
              12Mbps | -79dBm
              18Mbps | -77dBm
              24Mbps | -74dBm
              36Mbps | -70dBm
              48Mbps | -66dBm
              54Mbps | -65dBm

## 2.4GHz中国信道划分

&emsp;&emsp;&emsp;&emsp;802.11b和802.11g的工作频段在2.4GHz（2.4GHz-2.4835GHz），
其可用带宽为83.5MHz，中国划分为13个信道，每个信道带宽为22MHz

北美/FCC      2.412-2.461GHz(11信道)
欧洲/ETSI     2.412-2.472GHz(13信道)
日本/ARIB     2.412-2.484GHz(14信道）

### 2.4GHz频段WLAN信道配置表

信道 | 中心频率（MHz） | 信道低端/高端频率
-----  | -----   | -----   
1 | 2412 | 2401/2423
2 | 2417 | 2406/2428
3 | 2422 | 2411/2433
4 | 2427 | 2416/2438
5 | 2432 | 2421/2443
6 | 2437 | 2426/2448
7 | 2442 | 2431/2453
8 | 2447 | 2426/2448
9 | 2452 | 2441/2463
10| 2457 | 2446/2468
11| 2462 | 2451/2473
12| 2467 | 2456/2478
13| 2472 |2461/2483

## SSID和BSSID

### 基本服务集（BSS）

基本服务集是802.11 LAN的基本组成模块。能互相进行无线通信的STA可以组成一个BSS（Basic Service Set） 。
如果一个站移出BSS的覆盖范围，它将不能再与BSS的其它成员通信。

### 扩展服务集（ESS）

多个BSS可以构成一个扩展网络，称为扩展服务集（ESS）网络，一个ESS网络内部的STA可以互相通信，
是采用相同的SSID的多个BSS形成的更大规模的虚拟BSS。连接BSS的组件称为分布式系统（Distribution System，DS）

### SSID

服务集的标识，在同一SS内的所有STA和AP必须具有相同的SSID，否则无法进行通信。

SSID是一个ESS的网络标识(如:TP_Link_1201)，BSSID是一个BSS的标识，BSSID实际上就是AP的MAC地址，
用来标识AP管理的BSS，在同一个AP内BSSID和SSID一一映射。在一个ESS内SSID是相同的，但对于ESS内的
每个AP与之对应的BSSID是不相同的。如果一个AP可以同时支持多个SSID的话，则AP会分配不同的BSSID来对应这些SSID。

![T2](/images/networks/80211/337897.jpg)

## AP种类

FAT AP和FIT AP比较如下图所示：

![T3](/images/networks/80211/3.jpg)

## 无线接入过程三个阶段

STA（工作站）启动初始化、开始正式使用AP传送数据帧前，要经过三个阶段才能够接入
（802.11MAC层负责客户端与AP之间的通讯，功能包括扫描、接入、认证、加密、漫游和同步等功能）：

1. 扫描阶段（SCAN）
2. 认证阶段 (Authentication)
3. 关联（Association）

![T4](/images/networks/80211/337894.png)

### Scanning

802.11 MAC 使用Scanning来搜索AP，STA搜索并连接一个AP，当STA漫游时寻找连接一个新的AP，STA会在在每个可用的信道上进行搜索。

* Passive Scanning（特点：找到时间较长，但STA节电）通过侦听AP定期发送的Beacon帧来发现网络，
  该帧提供了AP及所在BSS相关信息：“我在这里”…
* Active Scanning  （特点：能迅速找到）
  STA依次在13个信道发出Probe Request帧，寻找与STA所属有相同SSID的AP,若找不到相同SSID的AP，则一直扫描下去.

### Authentication

当STA找到与其有相同SSID的AP，在SSID匹配的AP中，根据收到的AP信号强度，选择一个信号最强的AP，
然后进入认证阶段。只有身份认证通过的站点才能进行无线接入访问。AP提供如下认证方法：

1. 开放系统身份认证(open-system authentication)
2. 共享密钥认证(shared-key authentication）
3. WPA PSK认证（ Pre-shared key）
4. 802.1X EAP认证

![T5](/images/networks/80211/337895.png)

### Association

当AP向STA返回认证响应信息，身份认证获得通过后，进入关联阶段。

1. STA向AP发送关联请求
2. AP 向STA返回关联响应

至此，接入过程才完成，STA初始化完毕，可以开始向AP传送数据帧。

![T6](/images/networks/80211/337898.png)

### 认证和关联过程

![T7](/images/networks/80211/337899.png)

### 漫游过程

![T8](/images/networks/80211/337896.png)


{% highlight ruby %}
文档信息
--------------
* 作者:
* 版权声明：自由转载-非商用
* 转载: []
{% endhighlight %}

[史上最全讲解](http://rf.eefocus.com/article/id-IEEE802.11bg?p=1)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
