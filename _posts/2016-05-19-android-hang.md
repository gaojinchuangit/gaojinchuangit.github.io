---
layout: post
title:  "Android Hange机问题处理方法"
date:   2016-05-19 13:59:12 +0800
categories: Android
tag: hang
---

-----------------------


Hang机即死机，是指手机没有反应，通常屏幕无法点亮或者屏幕卡住不再刷新，另外按键，点击屏幕都不会有反应。
是一种非常严重的bug，必须要引起足够的重视。

当出现Hang机问题时，要保护好现场

+ 千万不要直接拔掉电池，破坏场景

+ 确保持续充电，保存好现场

### 常见的Hang机有以下情况

+ 整个系统已经hang 住, 此时UART4 都不吐LOG, 没有办法获取机
   器的信息,概率非常非常低
+ 底层kernel 运行正常,上层android hang 住,发生的概率较高

### 记录场景

+ 发现hang 机的时间

    如果是发现时,感觉机器早已经hang机,也请说明

+ 复现手法,操作的流程,当时环境

    强调在正常使用到hang机过程中的操作。
    
    环境状态通常包括温度,湿度,网络信号状况。

+ 复现手机情况

    复现的软件版本: 版本号? USER/ENG Build ?
    
    外部设备情况:有插入SD卡?耳机?SIM?
    
    软件开启情况: 开启蓝牙?WIFI?数据服务?GPS?

+ 复现的概率

    多少台手机做过测试,多少台手机可以复现。
    
    前后多少个版本可以复现,从哪个版本开始可以复现。

### Hange机问题处理方法
+ 静等三分钟,确认机器是否重启

    Android 2.3 , SW Watchdog reboot 时间为3分钟
    
    ICS 4.0, SW Watchdog reboot 时间为1分钟
    
    确认重启时,是否是直接闪出动画,而没有开机的logo显示
+ 进一步手机操作,确认Hang 机
    
    Power key, Volume down, Volume up 等按键是否有响应
    
    Home, Menu, Back 等虚拟按键是否有响应
    
    Statusbar 是否可以下拉

+ 插入USB,确认ADB是否可以使用

    首先查看windows的设备管理器里面是否出现对应的设备
    
    在命令行中输入adb devices, 看是否可以打印设备信息,在输入之 前您最好先输入adb kill-server 保证pc 上的adb client 没有卡住
    
    请确保您使用的PC上已经安装ADB,USB端口本身正常

+ ADB能使用情况下,使用ADB确认手机的基本状态

    adb shell 是否可以执行正常,如正常ENG 版本#,USER 版本$
    
    在命令行中输入 adb shell getevent ,然后点击屏幕,操作按键, 看是否有如下图的信息吐出,确认按键和屏幕的驱动是否正常

    尝试使用adb logcat –v time 打印LOG信息,确认是否有LOG 可以 打印。

+ ADB抓取手机状态信息

    确认adb shell 可正常执行,将下面的脚本拷贝到一个bat 文件中,点击执行,抓出手机的状态信息, 提交完整的mtklog 目录 • 确保持续的USB连接,保存好现场。

```
adb devices
@echo "抓出mtklog"
adb pull /mnt/sdcard/mtklog mtklog
@echo "抓出trace"
adb pull /data/anr mtklog/anr
@echo "抓出data aee db"
adb pull /data/aee_exp mtklog/data_aee_exp @echo "抓出NE core"
adb pull /data/core mtklog/data_core
@echo "抓出tombstones"
adb pull /data/tombstones mtklog/tombstones @echo “抓出手机状态”
adb shell ps > mtklog/ps.txt
adb shell dumpstate > mtklog/dumpstate.txt adb shell dumpsys > mtklog/dumpsys.txt
adb shell top -t -d 2 -n 5 > mtklog/top.txt
adb shell service list > mtklog/serviceList.txt
@echo "完成"
```

+ UART4串口连接手机抓取信息
    
    如果手机没有拉长UART4 , 则忽略此过程

    在确认adb无法使用的情况下,尝试UART4连接手机,并将超级终 端设置为catch text,保存好对话信息。

    串口连接后,尝试按power key 是否有LOG 吐出
    
    输入ps命令,是否可以打印出进程信息,如果成功,进一步 
    
    – dumpsys
    
    – dumpstate
    
    – service list

+ 继续用USB 连接机器,保证手机不掉电。

+ 确保该复现手机的版本的symbols 和vmlinux 都还存在


+ 如果串口(UART4) 按power key 都没有任何的LOG 突出,插入 usb 也没有任何的LOG,各种操作都无法得到信息。

+ 此时手机可能处于两种情况:
    
    机器已经完全从硬件上hang住 
    
    机器的电池没电停机了
+ 插入USB,持续20-30分钟,后再按power key 如果能够开机 ,说明前面是电池没电的情况(可进一步查看电量确认),否则是 硬件hang 住的情况。当然如果前面有串口连接手机,我们可以 从串口LOG 里面查看电池的电量情况



