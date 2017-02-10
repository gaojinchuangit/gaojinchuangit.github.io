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
