---
layout: post
title:  "Android Monkey测试"
date:   2016-06-09 20:29:43 +0800
categories: Android
tag: Monkey
---

-----------------------


    Monkey是一个命令行工具 ，可以运行在模拟器里或实际设备中。它向系统发送伪随机的用户事件流，实现对正在开发的应用程序进行压力测试。

    Monkey测试是Android自动化测试的一种手段，Monkey测试就是模拟用户的按键输入，触摸屏输入，手势输入等，看设备多长时间会出异常。

    Monkey程序是由Android系统自带，使用Java语言写成，在Android文件系统中的存放路径是：/system/framework/Monkey.jar

    Monkey.jar程序是由一个名为“Monkey”的Shell脚本来启动执行，shell脚本在Android文件系统中的存放路径是：/system/bin/Monkey;这样就可以通过在CMD窗口中执行：adb shell Monkey {命令参数}来进行Monkey测试了。

### 环境配置

PC端

    OS
         -Windows XP or Vista
         -Linux

    SDK
         -Android Software Development Kit
         -选用与Android系统相匹配版本的SDK

    环境变量
         -设置SDK中文件夹tools的路径到环境变量中


手机端

    SD card
        -建议插入已格式化的SD卡，并保证存储空间足够，以防log被写满

    Screen timeout
         -设置屏幕延时为最大

    USB debugging
         -打开USB调试

    3rd part & GMS application
        -建议移除第三方应用及GMS应用，以先关注系统的稳定性

    Unlock screen
        -执行Monkey测试前，一定要确保屏幕解锁

    TagLog
        -打开MTK的tagLog防止丢失log


### Monkey用法

Monkey基本语法

    adb shell monkey [options] <event-count>

测试内部用的命令

    adb shell "monkey --throttle 200 --ignore-crashes --ignore-timeouts --ignore-security-exceptions --monitor-native-crashes --ignore-native-crashes -v -v -v 10000000 > /mnt/sdcard/monkey-log.txt"

测试单个apk的命令

    adb shell "monkey --throttle 200 -p com.android.dialer -v -v -v 1000 > /mnt/sdcard/monkey-log.txt"

事件命令

    s<seed>  
        伪随机数生成器的seed值。如果用相同的seed值再次运行Monkey，它将生成相同的事件序列。

    throttle<milliseconds>  
        在事件之间插入固定延迟。通过这个选项可以减缓Monkey的执行速度。如果不指定该选项，Monkey将不会被延迟，事件将尽可能快地被产成。

    v        
       命令行的每一个-v将增加反馈信息的级别。

    pct-touch<percent>  
        调整触摸事件的百分比

    pct-motion<percent>  
        调整动作事件的百分比 

    pct-trackball<percent>  
        调整轨迹事件的百分比

    pct-nav<percent>  
        调整“基本”导航事件的百分比

    pct-majornav<percent>
        调整“主要”导航事件的百分比

    pct-syskeys<percent>
        调整“系统”按键事件的百分比

    pct-appswitch<percent>
        调整启动Activity的百分比。在随机间隔里，Monkey将执行一个startActivity()调用，作为最大程度覆盖包中全部Activity的一种方法。

    pct-anyevent<percent>
       调整其它类型事件的百分比。它包罗了所有其它类型的事件，如：按键、其它不常用的设备按钮、等等

约束限制命令


    p<allowed-package-name>
           如果用此参数指定了一个或几个包，Monkey将只允许系统启动这些包里的Activity。
    
    c<main-category>    
            如果用此参数指定了一个或几个类别，Monkey将只允许系统启动被这些类别中的某个类别列出的Activity。如果不指定任何类别，Monkey将默认选择（Intent.CATEGORY_LAUNCHER或Intent.CATEGORY_MONKEY）。
    
    ignore-security-exceptions
            通常，当应用程序发生许可错误(如启动一个需要某些许可的Activity)时，Monkey将停止运行。如果设置了此选项，Monkey将继续向系统发送事件，直到计数完成。

    kill-process-after-error
            通常，当Monkey由于一个错误而停止时，出错的应用程序将继续处于运行状态。当设置了此选项时，将会通知系统停止发生错误的进程。注意，正常的(成功的)结束，并没有停止启动的进程，设备只是在结束事件之后，简单地保持在最后的状态。 

    monitor-native-crashes
           监视并报告Android系统中本地代码的崩溃事件。如果设置了--kill-process-after-error，系统将停止运行。

    wait-dbg
           停止执行中的Monkey，直到有调试器和它相连接。


### 分析工具

**QAAT**

    全称 
    -Quickly Android Analysis Tool

    特性 
    -针对海量Log，快速提取有效讯息

    支持异常类型
    -JE/NE/KE/ME/ANR
    -Sytem Server Crash
    -Low Memory kill info
    -Some important warning

**addr2line**

    定义
    -一个可以将指令的地址和可执行映像转换成文件名、函数名和源代码行数的工具

    用法
    -addr2line -options
    -addr2line -help

    常用的命令
    -addr2line -C -f -e 解析文件 解析地址

**AEE**

    全称
    -Android Exception Engine

    用途
    -捕捉异常
    -收集调试信息
    -输出DB file-db0X.dbg


### LOG类型

**ANR**

    全称 
    -Application Not Response

    分类
    -Dispatch time out
    -Broadcast time out
    -Service time out

**NE**

    全称
    -Native Exception

    分类
    -空指针，野指针，数组越界
    -格式化输出参数错误
    -缓冲区溢出
    -主动溢出抛出
    -中断信号（SIGABRT6：由abort(3)发出的退出指令；SIGKILL9：AEF Kill信号；SIGSEGV11：无效的内存引用）

**KE**

    全称
    -Kernel Exception

### LOG案例分析

**ANR**

![monkey1]({{ '/styles/images/androidmonkey/monkey1.jpg' | prepend: site.baseurl  }})

**NE**

![monkey2]({{ '/styles/images/androidmonkey/monkey2.jpg' | prepend: site.baseurl  }})

![monkey3]({{ '/styles/images/androidmonkey/monkey3.jpg' | prepend: site.baseurl  }})

**KE**

![monkey4]({{ '/styles/images/androidmonkey/monkey4.jpg' | prepend: site.baseurl  }})