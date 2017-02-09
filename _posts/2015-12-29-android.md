---
layout: post
title:  "OTA升级权限问题"
date:   2015-12-29 22:13:41 +0800
categories: Android
tag: selinux
---

-----------------------

    在本地升级的过程中，升级失败，发现主要是avc的问题。 
    https://onlinesso.mediatek.com/Pages/FAQ.aspx?List=SW&FAQID=FAQ11486

    android KK 4.4 版本后，Google 默认启用了SELinux, 并会把SELinux 审查异常打印在kernel log或者android log(L 版本)中，对应的关键字是: "avc:  denied" 或者"avc: denied"
    如一行LOG：
    <5>[ 17.285600].(0)[503:idmap]type=1400 audit(1356999072.320:202): avc:  denied  { create } for  pid=503 comm="idmap" name="overlays.list" scontext=u:r:zygote:s0 tcontext=u:object_r:resource_cache_data_file:s0 tclass=file
    即表明idmap 这个process, 使用zygote 的source context, 访问/data/resource_cache 目录，并创建文件时，被SELinux 拒绝访问。
 
    [Keyword]
    android, SELinux, avc:  denied, audit
 
# [Solution]

    KK 版本, Google 只有限制的启用SELinux, 即只有针对netd, installd, zygote, vold 以及它们直接fork 出的child process 使用enforcing mode, 但不包括zygote fork的普通app.
    L  版本以及以后版本, Google 全面开启SELinux, 几乎所有的process 都使enforcing mode， 影响面非常广.
 
    目前所有的SELinux check 失败，在kernel log 或者android log(L版本后)中都有对应的"avc:  denied" 或者 "avc: denied"的LOG 与之对应。反过来，有此LOG，并非就会直接失败，还需要确认当时SELinux 的模式, 是enforcing mode 还是 permissve mode.
    首先, 务必确认对应进程访问系统资源是否正常， 是否有必要 ？如果本身是异常非法访问，那么就要自行消除访问。
    其次, 如果确认访问是必要，并且正常的，那么就要对对应的process/domain 增加新的policy.

### 1). 简化方法
 
     1.1 提取所有的avc LOG.   如 adb shell "cat /proc/kmsg | grep avc" > avc_log.txt
     1.2 使用 audit2allow tool 直接生成policy. audit2allow -i avc_log.txt  即可自动输出生成的policy
     1.3 将对应的policy 添加到selinux policy 规则中，对应MTK Solution, 您可以将它们添加在KK: mediatek/custom/common/sepolicy, L: device/mediatek/common/sepolicy 下面，如
      allow zygote resource_cache_data_file:dir rw_dir_perms;
      allow zygote resource_cache_data_file:file create_file_perms;
      ===> mediatek/custom/common/sepolicy/zygote.te (KK)
      ===> device/mediatek/common/sepolicy/zygote.te (L)
     注意audit2allow 它自动机械的帮您将LOG 转换成policy, 而无法知道你操作的真实意图，有可能出现权限放大问题，经常出现policy 无法编译通过的情况。
 
### 2). 按需确认方法
     此方法需要工程人员对SELinux 基本原理，以及SELinux Policy Language 有了解. 
    2.1 确认是哪个进程访问哪个资源，具体需要哪些访问权限，read ? write ? exec ? create ? search ?
    2.2 当前进程是否已经创建了policy 文件？ 通常是process 的执行档.te，如果没有，并且它的父进程即source context 无须访问对应的资源，则创建新的te 文件.
          在L 版本上, Google 要求维护关键 security context 的唯一性, 比如严禁zygote, netd, installd, vold, ueventd 等关键process 与其它process 共享同一个security context.
    2.3 创建文件后，关联它的执行档，在file_contexts 中, 关联相关的执行档.
          比如 /system/bin/idmap 则是 /system/bin/idmap u:object_r:idmap_exec:s0
    2.4 填写policy 到相关的te 文件中

     如果沿用原来父进程的te 文件，则直接添加.
     如果是新的文件，那么首先：
      ==============================================
      # Type Declaration
      ==============================================
      type idmap, domain;
      type idmap_exec, exec_type, file_type;
  
      #==============================================
      # Android Policy Rule
      #==============================================
      #permissive idmap;
      domain_auto_trans(zygote, idmap_exec, idmap);
  
      然后添加新的policy
  
      # new policy
      allow idmap resource_cache_data_file:dir rw_dir_perms;
      allow idmap resource_cache_data_file:file create_file_perms;
 
### 3). 权限放大情况处理
  
    如果直接按照avc: denied 的LOG 转换出SELinux Policy, 往往会产生权限放大问题. 比如因为要访问某个device, 在这个device 没有细化SELinux Label 的情况下, 可能出现:
    <7>[11281.586780] avc:  denied { read write } for pid=1217 comm="mediaserver" name="tfa9897" dev="tmpfs" ino=4385 scontext=u:r:mediaserver:s0 tcontext=u:object_r:device:s0 tclass=chr_file permissive=0
    如果直接按照此LOG 转换出SELinux Policy:  allow mediaserver device:chr_file {read write};  那么就会放开mediaserver 读写所有device 的权限. 而Google 为了防止这样的情况, 使用了neverallow 语句来约束, 这样你编译sepolicy 时就无法编译通过.
    为了规避这种权限放大情况, 我们需要细化访问目标(Object) 的SELinux Label, 做到按需申请. 通常会由三步构成

     3.1 定义相关的SELinux type.
       比如上述案例, 在 device/mediatek/common/sepolicy/device.te 添加
       type tfa9897_device, dev_type;
     3.2 绑定文件与SELinux type.
       比如上述案例, 在 device/mediatek/common/sepolicy/file_contexts 添加
       /dev/tfa9897(/.*)? u:object_r:tfa9897_device:s0
     3.3 添加对应process/domain 的访问权限.
       比如上述案例, 在 device/mediatek/common/sepolicy/mediaserver.te 添加
       allow mediaserver tfa9897_device:chr_file { open read write };
 
       那么哪些访问对象通常会遇到此类呢？(以L 版本为例)
  
        * device  
        -- 类型定义: external/sepolicy/device.te;device/mediatek/common/sepolicy/device.te 
        -- 类型绑定: external/sepolicy/file_contexts;device/mediatek/common/sepolicy/file_contexts
  
        * File 类型: 
        -- 类型定义: external/sepolicy/file.te;device/mediatek/common/sepolicy/file.te
        -- 绑定类型: external/sepolicy/file_contexts;device/mediatek/common/sepolicy/file_contexts
 
        * 虚拟File 类型: 
        -- 类型定义: external/sepolicy/file.te;device/mediatek/common/sepolicy/file.te
        -- 绑定类型: external/sepolicy/genfs_contexts;device/mediatek/common/sepolicy/genfs_contexts
 
        * Service 类型:
        -- 类型定义: external/sepolicy/service.te; device/mediatek/common/sepolicy/service.te
        -- 绑定类型：external/sepolicyservice_contexts;device/mediatek/common/sepolicy/service_contexts
        
        * Property 类型:
        -- 类型定义: external/sepolicy/property.te;device/mediatek/common/sepolicy/property.te
        -- 绑定类型: external/sepolicy/property_contexts;device/mediatek/common/sepolicy/property_contexts;
        
      通常我们强烈反对更新google default 的policy, 大家可以更新mediatek 下面的相关的policy.
 
 
### 4). 从android m1 (6.1) 后, MTK sepolicy 进行了分层划分。

    从android m1 (6.1) 后, 根据不同的项目,平台, 客户需求进行配置, 消除sepolicy 冗余, 降低安全风险.

    /device/mediatek/common/sepolicy 下面是ALL platform 都需要.
    /device/mediatek/[platform]/sepolicy 则是某个具体的平台.
    /device/[customer]/[project]/sepolicy 目前没有默认配置, 客户可根据更新BroadConfig.mk 中BOARD_SEPOLICY_DIRS 以新增目录.

    Basic 版本                 导入 /device/mediatek/common|[platfrom]/sepolicy/basic
    Bsp 版本                   导入/device/mediatek/common|[platfrom]/sepolicy/basic|bsp
    TK 版本(通常情况下)    导入/device/mediatek/common|[platfrom]/sepolicy/basic|bsp|full

### 分析结果
  
      首先是生成串口log，使用附件-通讯-超级终端来抓取串口log，然后使用使用 audit2allow tool 直接生成policy. audit2allow -i avc_log.txt 即可自动输出生成的policy。
  
      生成的结果如：allow recovery media_rw_data_file:dir read

      然后添加到recovery.te中
      allow recovery media_rw_data_file:dir { open read search write remove_name }; 
      allow recovery media_rw_data_file:file { open read }; 
      allow recovery app_data_file:dir search;
      重新编译bootimage、recoveryimage、systemimage