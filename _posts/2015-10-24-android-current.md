---
layout: post
title:  "Android电流功耗问题(2)"
date:   2015-10-22 23:11:21 +0800
categories: Android
tag: current
---

-----------------------

### 如何查看唤醒时长

分析待机平均电流高的问题时，经常需要知道每次wake up起来的时间点，以及唤醒的时长，以此找到一些异常的唤醒。

**MT6572平台**

    （1）查找kernel log中的“Wakeup Succefully”信息
    （2）“往下”查找离这条log最近的带“UTC”的log，UTC log中显示的时间就是唤醒的时间（跟main log的时间一致）
    （3）“往下”查找离这条log最近的“[SPM] Kernel Suspend with”信息，跟“Wakeup Succefully”的时间戳相减就是wake up的时长
 
     eg：
```
<4>[ 2497.698724]-(0)[789:kworker/u:3][PCM WAKEUP NORMAL]CPU WAKE UP BY: EINT :0x10000
<6>[ 2497.901281] (0)[789:kworker/u:3]PM: suspend exit 2013-05-27 00:40:53.186864384 UTC
<2>[ 2539.341088]-(0)[789:kworker/u:3][SPM] Kernel Suspend with cpu_pdn=1, infra_pdn=1
```

    唤醒时间点：00：40：53
    唤醒时长：2539-2497 = 42s


**其他平台**

    （1）查找kernel log中的“wake up by XXX”信息
    （2）“往下”查找离这条log最近的带“UTC”的log，UTC log中显示的时间加8小时就是真实的唤醒时间（跟mainlog的时间一致）
    （3）“往下”查找离这条log最近的“wakesrc”信息，跟“wake up by XXX”的时间戳相减就是wake up的时长

     eg：
```
<5>[ 621.286161] (0)[53:kworker/u:2][Power/Sleep] wake up by EINT (0x20)(0x4)(41695)
<5>[ 621.626419] (0)[310:WindowManagerPo][request_suspend_state]: wakeup (3->0) at612863283411 (2013-05-24 07:50:14.331687541 UTC)
<5>[ 665.287105] (0)[1218:kworker/u:3][Power/Sleep] sec = 300, wakesrc = 0x1ec
```
    唤醒时间点：07(+8)：50：14 = 15：50：14
    唤醒时长：665-621 = 44s


### 关于wifi耗电问题

**关于电流的测量**

    1 在测量wifi电流前，请先确认是否有一些可疑的第三方apk，比如QQ，比如wifi分析仪等，最好能够是拿一只没有安装第三方apk的手机进行测试。
    2 测试电流时，最好是灭屏待机一段时间后，等电流稳定后进行测量。
    3 如果是连接路由器进行测量，请务必不要使路由器接到外网，单独进行测试。
    4 抓取log时，需要同时提供mobilelog和netlog，而且要能够复现完整的过程，且记录测试和结束的时间点。

**耗电问题的log分析**

    如果以上操作，发现电流仍然偏高，就需要分析log，主要从以下几点来分析，

    1 在测量电流的时间段内，从mainlog搜索wakelock，查看是否wakelock被wifi长时间暂用而不释放。
    2 搜索DHCP，查看是否有DHCP的频繁的续租ip地址
    3 下载Wireshark软件，查看netlog，看看测量时间段内，是否有大量的数据包发送，比如tcp/ip包，DNS包，ARP包，ICMP包等

    基本上，通过以上分析，大部分wifi耗电问题的原因都可以找到。




### 如何测试 Mediatek 平台各个场景的功耗数据

**测试功耗数据之前，请先确认以下配置：**

    1、关闭 WIFI/BT/GPS，关闭数据连接，设置飞行模式。 （根据具体测试场景设置）
    2、关闭 mobile log/modem log/net log，打开LOG会增加电流。注意：确认 /sdcard/mtklog （/data/mtklog） 中是否有 LOG 生成，确定关闭成功。
    3、确认各个模块是否已经正常工作，各个模块都会影响功耗，需要在模块工作 OK 之后再测试功耗问题。
    4、测试将所有第三方 APK 删除，排除第三方 APK 问题。

![current8]({{ '/styles/images/androidcurrent/current8.jpg' | prepend: site.baseurl  }})

![current9]({{ '/styles/images/androidcurrent/current9.jpg' | prepend: site.baseurl  }})    
