---
layout: post
title:  "Android电流功耗问题(1)"
date:   2015-10-20 20:23:29 +0800
categories: Android
tag: current
---

-----------------------

## 背景知识

    电流和功耗是非常容易出问题，也很难理清的部分，因为涉及到APK/驱动/硬件各种不确定因素。所以在遇到这类问题的时候一定要遵循一个处理原则：
    
    先花时间把现象理清楚，问题复现的环境是什么，多做几次交叉实验，得到清晰的问题描述，问题复现条件和相应的电流波形图。

### 手机的模式

**Sleep mode**

    为了降低功耗，系统在处理完需要处理的事情之后就会处于idle状态进入睡眠模式（当用户将手机放置一段时间后，或者按Power键，手机都会进入到睡眠模式），睡眠模式下系统时钟会由26M切换到32K,部分外部设备的电源会被切断，系统所需的相关内核电压会被切换到一个较低的值。

    系统在sleep mode 下，一般我们需要关注底电流和待机平均电流

**Active mode**

    泛指系统在wake up时的各种应用场景，包含如下场景：

    (1)Talking mode:手机通话模式
    (2)MultiMedia：多媒体模式，包括play MP3/mp4 camera preview/record等
    (3)WCN：各种connectivity 的turn on/off场景，包括FM/BT/WIFI/GPS等
    (4)other Application：浏览网页，发送email，观看在线视频等

### 平台参考电流

    平台参考电流，一般是由基带根据MTK给出的文档进行测试之后给出参考标准


### 低功耗

**Deep idle**

    deep idle是idle线程的一种最优化方式，为了改善低功耗的表现
    在没有任务需要处理的时候，idle线程就会启动，CPU就进入到WFI（wait for interrupt）状态

**SODI**

    Screen on deep idle一种省电模式，平台一般是关闭的，可能会导致开机花屏

**PerfService（performance service）**

    ＭＴＫ开发，用来动态主动调整CPU/GPU资源的服务


![current1]({{ '/styles/images/androidcurrent/current1.jpg' | prepend: site.baseurl  }})

**DVFS**

    Dynamic Voltage and Frequency Scaling
    动态电压和频率调节用来降低功耗节省电量,预估下次Cpu loading所需的power

**CPU hot-plug**

    linux内核支持在不重启手机的情况下开关CPU

![current2]({{ '/styles/images/androidcurrent/current2.jpg' | prepend: site.baseurl  }})


## 抓取Log和电流波形图

    (1)如果与数据连接有关的电流异常，请先把手机全频段校准写入合法的IMEI，保证手机天线连接并且性能良好， SIM卡没有欠费，不是特殊SIM卡，手机Modem基本功能正常

    (2)连接Power Monitor 点击run然后开机

    (3)将手机时间调整到当前时间

    (4)清空old log

    (5)打开mobilelog，关闭其他Log.如果要抓取的Log是和数据连接或者wifi相关的那么还需要再打开netlog. modemlog除非MTK开口要，否则务必要关闭，因为他会频繁的唤醒系统，导致电流偏大.

    (6)等待手机自动灭屏

    (7)等待电流降下来，稳定后点击Power Monitor stop然后再run,抓取电流波形图并记录此时的时间点

    (8)等待20分钟后，停止抓取电流波形图，记录时间点并save电流波形图

    (9)使用adb pull导出Mtklog


## debug和Log的分析

### 唤醒源

    可以在kernel log中搜索关键字 wake up by查找唤醒源

![current3]({{ '/styles/images/androidcurrent/current3.jpg' | prepend: site.baseurl  }})

    wake up by xxx这个xxx就代表了唤醒源
    
    系统在进入到suspend之后会由SPM接手控制，那么从suspend状态中resume回来的前提自然是需要先把CPU唤醒。
    
    所谓的唤醒源其实也就是一些系统的irq资源，通过设定SPM寄存器可以选择哪些irq可以被被SPM处理并且做为系统的唤醒源，而哪些是不可以的，这个可以灵活设定，但一般不允许用户修改。
    
    suspend的唤醒源可以在mt_spm_sleep.c中找到，全部的唤醒源定义在mt_spm.h,不过其中有很多都是SPM自己使用的，不会把信号发给CPU，我们这里所说的唤醒源其实只是其中的一小部分。

**我们一般需要关注的有以下几个唤醒源：**

    (1)KP
       键盘，如果用到侧键唤醒，需要打开这个唤醒源，但这个不是功耗要讨论的重点
    (2)EINT： External interrupt
       外部中断，其中最重要的是PMIC的中断(pwrkey/charger/RTC,alarm/wifi EINT/C2K EINT),PMIC(Power Management IC)电源管理集成电路
    (3)CONN2AP
       wcn chip的唤醒,包括BT/WIFI/GPS等各种connectivity
    (4)CCIF0_MD CCIF1_MD
       旧的架构中使用的modem唤醒源，或者第二颗modem的唤醒源是modem和AP之间的通道的唤醒
    (5)CLDMA_MD
      新的架构中使用的modem的唤醒源，是modem和AP之间的通道的唤醒
    (6)SEJ
       目前只有指纹识别模块可能用到唤醒源是定位待机功耗的关键.


### 唤醒时间点

![current4]({{ '/styles/images/androidcurrent/current4.jpg' | prepend: site.baseurl  }})

    也可以直接搜索离wake up by xxx 最近的关键字UTC
    
    UTC关键字前面有具体的时间，这个时间加上8小时就是对应的main log中的时间，我们可以根据这个时间去main log中查看这个段时间是哪个具体的唤醒源，唤醒了系统。然后进一步定位，异常电流产生的原因。

### APK Alarm唤醒类
  
    1.wake up by EINT后面有出现rtc_int_handler或者rtc_tasklet_handler或者RG_INT_STATUS_RTC

    2.找到最近的UTC或者suspend exit确认时间点

    3.sys log/main log中搜索关键字wakeup alarm或者AlarmManager:sending alarm Alarm.

    4.把kernel log和system log中的时间点对应起来，就是对应的alarm默认alarm可能没开，如果没有看到，可以使用adb命令打开 
    adb shell dumpsys alarm log on    （eng版本）    

![current5]({{ '/styles/images/androidcurrent/current5.jpg' | prepend: site.baseurl  }})

### APK后台发送数据类

    在main log中搜索关键字Posix_connect Debug

    有些第三方APK，会透过设置alarm到RTC，定时唤醒系统，然后透过数据连接发送接收数据，这个会造成modem很大的电流

    Modem端在接收QQ的消息时，会透过CLMDA或者CCCI唤醒AP来完成消息的接收.APK如果发送消息就会dump出Posix_connect Debug. 

![current6]({{ '/styles/images/androidcurrent/current6.jpg' | prepend: site.baseurl  }})

### Android wake lock

    上层持锁，造成系统无法进入睡眠

    可以在system log中搜索关键字PARTIAL_WAKE_LOCK

    或者也可以搜索关键字onWakeLockAcquired唤醒持锁开始,关键字releaseWakeLockInternal此关键字应用释放锁其后一般跟应用持锁的total_time如果应用持锁时间过长，则可能导致电流偏大    


### Native and kernel wake lock 

    底层持锁造成系统无法进入休眠

    可以在测试电流波形结束后读取节点:
    adb shell dumpsys batterystats > batterystats.log(user userdebug eng版本)
    adb shell cat /sys/kernel/debug/wakeup_sources > wakeup_sources.log (eng)


![current7]({{ '/styles/images/androidcurrent/current7.jpg' | prepend: site.baseurl  }})
