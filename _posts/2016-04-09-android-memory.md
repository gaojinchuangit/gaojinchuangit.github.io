---
layout: post
title:  "Android内存相关"
date:   2016-04-09 23:29:51 +0800
categories: Android
tag: Proprity
---

-----------------------

## 简介
    Java内存泄漏指的是进程中某些对象（垃圾对象）已经没有使用价值了，但是它们却可以直接或间接地引用到gc roots导致无法被GC回收。无用的对象占据着内存空间，使得实际可使用内存变小，形象地说法就是内存泄漏了。

    GC的工作流程
    1、标记(Mark)
    找到所有的GC的根结点(GC Root), 将他们放到队列里，然后依次递归地遍历所有的根结点以及引用的所有子节点和子子节点，将所有被遍历到的结点标记成live
    2、计划(Plan)
    3、清理(Sweep)
    4、引用更新(Relocate)
    5、压缩(Compact)

    在应用程序中，只要某对象变得不可达，也就是没有根（root）引用该对象，这个对象就会成为垃圾回收器的目标。

### Android内存相关 

    Android系统对dalvik的vm heapsize作了硬性限制。 
    Android APP是基于java开发的，用到的dalvik虚拟机，每个应用启动后都会运行在一个独立的虚拟机中，每个虚拟机会为应用分配heap空间。当java进程申请的java空间超过阈值时，就会抛出OOM异常（这个阈值可以是48M、24M、16M等，视机型而定），可以通过adb shell getprop | grep dalvik.vm.heapgrowthlimit查看此值。

    也就是说，程序发生OOM并不表示RAM不足，而是因为程序申请的java heap对象超过了dalvik vm heapgrowthlimit。也就是说，在RAM充足的情况下，也可能发生OOM。

    Build.prop文件限制虚拟机大小：
    dalvik.vm.heapstartsize   堆分配的初始大小
    dalvik.vm.heapgrowthlimit   单个应用可用最大内存
    dalvik.vm.heapsize   manifest中指定android:largeHeap为true




![mem1]({{ '/styles/images/androidmem/mem1.jpg' | prepend: site.baseurl  }})

进程具有各自独立的用户空间，但是内核空间是共享的。当两个独立的进程要进行通信的时候，要通过的共享的内核空间来完成。

![mem2]({{ '/styles/images/androidmem/mem2.jpg' | prepend: site.baseurl  }})

memory mapping segment我们使用c库的malloc申请内存时，malloc调用brk()或mmap()实现堆(heap)或映射区域(memory mapping segment)的增长。


### APP常见的OOM原因

+ 注册某个对象后未反注册

+ 资源对象没关闭造成的内存泄露 (Cursor)

+ Bitmap使用后未调用recycle()。

+ 构造adapter没有使用缓存contentview

+ static关键字等

+ 未关闭InputStream/OutputStream



**1.注册某个对象后未反注**
```
    public class MediaPlaybackService extends Service{

    public void onCreate() {
    .....
        registerUnmountReceiver();
        .....
    }
    
    @Override
    public void onDestroy() {
    .....
        if (mUnmountReceiver != null) {
            unregisterReceiver(mUnmountReceiver);
            mUnmountReceiver = null;
        }
    }
 }
```

**2、资源对象没关闭造成的内存泄露 (Cursor)**

```
Cursor cursor = null;
try{
    cursor = mContext.getContentResolver().query(uri,null,null,null,null);
    if(cursor != null){
        cursor.moveToFirst();
    //do something
    }
}catch(Exception e){
    e.printStatckTrace();
}finally{
    if(cursor != null){
        cursor.close();
    }
}

```

    在CursorAdapter中应用的情况，，CursorAdapter在Acivity结束时并没有自动的将Cursor关闭掉，因此，你需要在onDestroy函数中，手动关闭。

    推荐用法:

```
@Override  
protected void onDestroy() 
    {            
        if (mAdapter != null && mAdapter.getCurosr() != null) 
        {          
            mAdapter.getCursor().close();      
        }    
        super.onDestroy();   
    }

```

**3、Bitmap使用后未调用recycle()**

    在用完Bitmap时，要及时的recycle掉
```
if (entry.bitmapTexture != null){
     entry.bitmapTexture.recycle();
}
```
[bitmap相关参考](https://developer.android.com/topic/performance/graphics/manage-memory.html)

**4、构造adapter没有使用缓存contentview**

```
public View getView(intposition, View convertView, ViewGroup parent) {  
　　   View view = null;  
　　     if (convertView != null){  
　　                view = convertView;  
　　                 populate(view, getItem(position));  
　　         } else {  
　　               view = new Xxx(...);  
　　         }  
　　return view;  
　　}  


```

**5、static关键字等**

    static是Java中的一个关键字，当用它来修饰成员变量时，那么该变量就属于该类，而不是该类的实例。

    public class ClassName {  
       private static Context mContext;  
       //省略  
    }   
    以上的代码是很危险的，如果将Activity赋值到么mContext的话。那么即使该Activity已经onDestroy，但是由于仍有对象保存它的引用，因此该Activity依然不会被释放。



### 内存问题调试方法

**DDMS查看系统内存**

    通过ddms的sysInfo,我们可以看到系统内存目前的分布情况,是一个饼状图

![mem3]({{ '/styles/images/androidmem/mem3.jpg' | prepend: site.baseurl  }})


**使用procrank查看进程内存**

    procrank 命令可以获得当前系统中各进程的内存使用快照，这里有PSS，USS，VSS，RSS。

    我们一般观察Uss来反映一个Process的内存使用情况,Uss 的大小代表了只属于本进程正在使用的内存大小,这些内存在此Process被杀掉之后,会被完整的回收掉,Vss和Rss对查看某一Process自身内存状况没有什么价值,因为他们包含了共享库的内存使用,而往往共享库的资源占用比重是很大的,这样就稀释了对Process自身创建内存波动。

    而Pss是按照比例将共享内存分割,某一Process对共享内存的占用情况。

    使用procrank查看进程内存
    VSS - Virtual Set Size 虚拟耗用内存（包含共享库占用的内存）
    RSS - Resident Set Size 实际使用物理内存（包含共享库占用的内存）
    PSS - Proportional Set Size 实际使用的物理内存（比例分配共享库占用的内存）
    USS - Unique Set Size 进程独自占用的物理内存（不包含共享库占用的内存）

![mem4]({{ '/styles/images/androidmem/mem4.jpg' | prepend: site.baseurl  }})


    使用脚本testmem.sh  chmod 775 testmem.s
```    
    #!/bin/bash
 
    while true; do 
    adb shell procrank | grep "com.android.systemui"
    sleep 1
    done
```        

    运行该脚本: ./testmem.sh
    这个脚本的用途是每1秒钟让系统输出一次systemui的内存使用状况

![mem5]({{ '/styles/images/androidmem/mem5.jpg' | prepend: site.baseurl  }})


**定位OOM的位置**

    1、内存分析工具 MAT(Memory Analyzer Tool)，可以作为Eclipse插件下载，也可以作为RCP应用下载.

    2、通过DDMS查看heap

    3、adb shell procrank –u  

    4、Android Studio也有工具可以定位OOM问题


## Native memory leak(APP)

    1.首先要先开启bionic memory debug 1模式  prop可以设置,1是memleak,5和10是内存越界,20是虚拟机(malloc_debug_common.cpp
    
  

    KK及以前版本：
        $adb shell setprop persist.libc.debug.malloc 1  $adb shell reboot

    L及以后版本：
        $ adb shell setprop libc.debug.malloc 1  
        $ adb shell stop   
        $ adb shell start
       
        查看 adb shell getprop libc.debug.malloc


    2. 开启DDMS隐藏的功能需要改一个配置
    
       linux版本：~/.android/ddms.cfg
       windows版本：C:…\$user\.android\ddms.cfg
       
       在ddms.cfg结尾新增一行: native=true 保存后重启ddms
       就可以看到新加的一个‘native heap’的tab了(SDK tools下的ddms.bat)

    3.打开DDMS，选择需要查看native memory的进程

    4.切换到native heap菜单，填入symbols search path(因为是linux路径，所以windows不能用,这导致第5步的结果没有函数名称信息，只有函数地址！)，点击snapshot current native heap usage就可以看到当前native heap的状况

    5.点击保存后，可以导出一个文本文件，打开后，可以看到一个个记录，它们分配大小*分配次数从大到小排列，所以如果有泄露的话，一般看一个记录就够了，拿到调用栈再结合代码分析可能的泄露点

## Native memory leak(Native)

    1、打开内存监控功能(必须使用eng或userdebug版本)
    
    L及之后版本：

    在vendor/mediatek/proprietary/external/aee/config_external/init.aee.customer.rc添加:

 ```
    on init 
    export LD_PRELOAD libsigchain.so:libudf.so
 
 ```   
    重新打包bootimage并下载, 开机后用adb输入:
    adb shell setprop persist.libc.debug.malloc 15
    adb shell setprop persist.libc.debug15.prog xxxxx 
    adb shell setprop persist.debug15.config 0x4a003024
    adb reboot

    其中xxxxx部分修改为要监控的可执行文件名,对应于system/bin目录下的可执行文件名，比如java程序都是app_process或者app_process64。需要监控的应用程序一般是DB通MeaiatekLogView解析开的_exp_main.txt中的Current Executing Process 0x4a003024是对记录调用栈的存储内存的设置，包括属性，分配的size等。

    注意：对于L版本，还需要额外做如下修改(M版本及之后版本不需要)：
    
    将/bionic/libc/bionic/malloc_debug_common.cpp中
```    
    #ifdef _MTK_MALLOC_DEBUG_
    const MallocDebug* __libc_malloc_dispatch = &__libc_malloc_default_dispatch;
    #else
    static const MallocDebug* __libc_malloc_dispatch = &__libc_malloc_default_dispatch;
    #endif
```    
    修改为

``` 
    const MallocDebug* __libc_malloc_dispatch = &__libc_malloc_default_dispatch;
```    
   
    这时会重启手机，再次开机后就进入malloc/free的调试方式，此时overhead会比较重，可能会有一些不预期的anr(出现ANR时请点击等待)，但不影响测试。测试前，请注意保留out/target/product/$project/symbols目录


    2. 检查是否存在内存泄漏。

        打开mtklogger，不断通过adb shell procrank -u > procrank.txt查看当前系统内存的使用情况。
        当看到被监控的进程占用内存(USS字段)超过了正常值很多，则可能存在内存泄漏。

    3. 得到该进程内存使用分布情况($pid为被监控进程pid)
        adb shell dumpsys meminfo $pid >meminfo_$pid.txt
        adb shell procmem $pid > procmem_$pid.txt

    4. 抓取该进程的coredump。
    
          M版本Userdebug load：
          adb shell aee -d coreon
          adb shell aee -d directon
          adb reboot
          adb shell kill -5 $pid

    5. 这种问题可以用E-Consulter（mtk online的工具栏搜索得到）分析，会给出分析报告



## 问题分析

**问题背景**

    用Launcher跑monkey时存在泄露，2个小时USS从65M涨到20
    
**分析过程**

    打开debug 15测试

    adb shell procrank –u
    在测试中用procrank查看uss一直在涨

    C:\Windows\System32>adb shell procrank | findstr launcher3
    1698 2801540K 368192K 257240K 250476K com.android.launcher3 

    用dumpsys抓取的信息如下：C:\Windows\System32>adb shell dumpsys meminfo 1698

```

** MEMINFO in pid 1698 [com.android.launcher3] **                         
Pss      Private Private Swapped Heap     Heap    Heap                           
Total   Dirty     Clean    Dirty        Size        Alloc     Free                         
------     ------     ------      ------         ------       ------     ------
Native Heap     29363  28016     0            0            78080  70795   7284
Dalvik Heap      7911    7768       0            0            64595  61187   3408
Dalvik Other     43303  43072     0            0
Stack                  1244     1244       0            0
Ashmem           155002 154972  0            0
Other dev          4            0             4            0
.so mmap          2307     516         92          0
.apk mmap       1454      0             96          0
.ttf mmap          106        0             28          0
.dex mmap        1112     4              1108     0
.oat mmap         923       0              116       0
.art mmap          3106     2804       24         0
Other mmap      32          8             0           0
EGL mtrack         7989     7989      0            0
GL mtrack           44356   44356    0            0
Unknown           11308   10660    0             0
TOTAL         309520 301409 1468      0            142675 131982 10692

```





    从dump出来的memory info可以看到，总共309520，也即300多M，其中 Ashmem 155002 ，就占了151M，这个应该才是leak的原因。

    Ashmem是通过mmap分配的内存，于是按照以下方法打开mmap debug机制再复测提供mtklog：
    
    在vendor/mediatek/proprietary/external/aee/config_external/init.aee.customer.rc添加:

```
    on init
    export LD_PRELOAD libsigchain.so:libudf.so
```

    重新打包bootimage并下载开机, 用adb输入:
    adb shell setprop persist.debug.mmap.program app_process
    adb shell setprop persist.debug.mmap.config 0x22002010
    adb shell setprop persist.debug.mmap 1
    adb reboot

    复测提供mtklog
    解析再次提供的DB，用工具分析怀疑SoundPoolThread存在内存泄漏：
    == mmap泄漏检查 ==anon mmap: 800848KB
    mmap已分配超过200MB (可能存在内存泄露), 以下列出分配最大尺寸和次数的调用栈:
    大小: 1040384字节, 已分配: 497次
    分配调用栈:
```    
    libudf.so mmap() + 138
    libc.so __allocate_thread() + 454 <bionic/libc/bionic/pthread_create.cpp:155>
    libc.so pthread_create() + 586 <bionic/libc/bionic/pthread_create.cpp:229>
    libutils.so androidCreateRawThreadEtc() + 138 <system/core/libutils/Threads.cpp:157>
    libutils.so androidCreateThreadEtc() + 14 <system/core/libutils/Threads.cpp:296>
    libutils.so android::createThreadEtc() + 150 <system/core/include/utils/AndroidThreads.h:112>
    libutils.so android::Thread::run() + 290 <system/core/libutils/Threads.cpp:698>
    libstagefright_foundation.so android::ALooper::start() + 342 <frameworks/av/media/libstagefright/foundation/ALooper.cpp:123>
    libmediandk.so createAMediaCodec() + 162 <frameworks/av/media/ndk/NdkMediaCodec.cpp:151>
    libsoundpool.so decode() + 698 <frameworks/base/media/jni/soundpool/SoundPool.cpp:537>
    libsoundpool.so android::Sample::doLoad() + 890 <frameworks/base/media/jni/soundpool/SoundPool.cpp:646>
    libsoundpool.so android::SoundPoolThread::doLoadSample() + 46 <frameworks/base/media/jni/soundpool/SoundPoolThread.cpp:108>
    libsoundpool.so android::SoundPoolThread::run() + 78 <frameworks/base/media/jni/soundpool/SoundPoolThread.cpp:86>
```
  
    == 栈结束 ==


    工具解析的结果是未释放
    从backtrace出发，__allocate_thread() + 454  <bionic/libc/bionic/pthread_create.cpp:155>对应的code：

    attr->stack_base = __create_thread_mapped_space(mmap_size, attr->guard_size); 

    也即提示是attr->stack_base没有释放，查找代码看它会在哪里被mummap。

    libutils.so androidCreateRawThreadEtc() + 138 <system/core/libutils/Threads.cpp:157>

    可知是在androidCreateRawThreadEtc()函数中调用pthread_create，该函数对attr的设置如下：
    
    pthread_attr_t attr; pthread_attr_init(&attr); pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED); 

    而对于PTHREAD_CREATE_DETACHED属性的线程所占的资源在pthread_exit 时自动释放,pthread_exit()中可以看到attr->stack_base的释放是通过_exit_with_stack_teardown(thread->attr.stack_base, thread->mmap_size);

    这个函数是汇编代码实现的，而mmap debug机制是通过rehook mmap&mummap 来监测内存的分配和释放，所以对于这种是没有办法监测到释放的过程

    既然是ashmem占用内存过多，但是奇怪的是mmap泄漏检查显示的backtrace根本就没有与之相关的内存块，将所有mmap的backtrace列出来也没有看到，这又是怎么回事呢？review mmap debug机制，mmap_debug.c中:

    原因在这里，mmap debug机制只会记录flags为MAP_ANONYMOUS匿名映射的backtrace，而ashmem是有名字的，并不会记录，所以打开mmap debug机制对于检查ashmem leak的case并没有作用。


    根据DB解析开的PROCESS_MAPS可以看到，
    /dev/ashmem/MemoryHeapBase占的内存过高，在MemoryHeapBase.cpp中加log打印申请和释放的backtrace。
```
    int backtrace(void **buffer, int size); 
    从复现提供过来mobilelog结合DB解析开的PROCESS_FILE_STATE中的fd可知：
    01-01 02:47:37.378898 1568 2358 D MemoryHeapBase: #00 pc 0000000000032220 
    /system/lib64/libbinder.so (android::MemoryHeapBase::MemoryHeapBase(unsigned long, unsigned int, char const*)+416)
    01-01 02:47:37.378974 1568 2358 D MemoryHeapBase: #01 pc 0000000000006814 /system/lib64/libsoundpool.so (android::Sample::doLoad()+84)
    01-01 02:47:37.379019 1568 2358 D MemoryHeapBase: #02 pc 0000000000008b08 /system/lib64/libsoundpool.so (android::SoundPoolThread::doLoadSample(int)+48)
    01-01 02:47:37.379062 1568 2358 D MemoryHeapBase: #03 pc 0000000000008ba0 /system/lib64/libsoundpool.so (android::SoundPoolThread::run()+80)
    01-01 02:47:37.379103 1568 2358 D MemoryHeapBase: #04 pc 0000000000093410 /system/lib64/libandroid_runtime.so (android::AndroidRuntime::javaThreadShell(void*)+96)
    01-01 02:47:37.379144 1568 2358 D MemoryHeapBase: #05 pc 0000000000014fec /system/lib64/libutils.so
    01-01 02:47:37.379184 1568 2358 D MemoryHeapBase: #06 pc 00000000000673c8 /system/lib64/libc.so (__pthread_start(void*)+52)
    01-01 02:47:37.379224 1568 2358 D MemoryHeapBase: #07 pc 000000000001ed44 /system/lib64/libc.so (__start_thread+16)
    
```    
    这个backtrace申请的 MemoryHeapBase最终都没有释放

    结论:
    客制化代码写的有问题，造成sound 的resouce 无法释放




