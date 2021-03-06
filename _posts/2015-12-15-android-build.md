---
layout: post
title:  "AndroidL编译流程"
date:   2015-12-15 19:43:41 +0800
categories: Android
tag: build
---

-----------------------

**1、编译的第一步为 . build/envsetup.sh**

    envsetup.sh是一个脚本，位置在源码根目录下的build目录下，主要作用是初始化一些命令，如lunch、make、mmm、mm、resgrep等。脚本中简单说明如下：

```
Invoke ". build/envsetup.sh" from your shell to add the following functions to your environment:
- lunch:   lunch <product_name>-<build_variant>
- tapas:   tapas [<App1> <App2> ...] [arm|x86|mips|armv5|arm64|x86_64|mips64] [eng|userdebug|user]
- croot:   Changes directory to the top of the tree.
- m:       Makes from the top of the tree.
- mm:      Builds all of the modules in the current directory, but not their dependencies.
- mmm:     Builds all of the modules in the supplied directories, but not their dependencies.
           To limit the modules being built use the syntax: mmm dir/:target1,target2.
- mma:     Builds all of the modules in the current directory, and their dependencies.
- mmma:    Builds all of the modules in the supplied directories, and their dependencies.
- cgrep:   Greps on all local C/C++ files.
- ggrep:   Greps on all local Gradle files.
- jgrep:   Greps on all local Java files.
- resgrep: Greps on all local res/*.xml files.
- sgrep:   Greps on all local source files.
- godir:   Go to the directory containing a file.
```
    在工作中比较常用的命令有lunch、m、mm、mmm、make、jgrep、resgrep、godir。命令m就是对make命令的简单封装，并且是用来对整个Android源代码进行编译，而命令mm和mmm都是通过make命令来对Android源码中的指定模块进行编译；jgrep用来搜索包含某字符串的java文件；resgrep用来搜索包含某字符串的xml文件；godir可以用来查找某文件的路径。

**2、编译的第二步为lunch**

    lunch函数提供了一个菜单，可以选择需要编译的项目，并做一些检查，设置环境变量。
    lunch命令输出了一个菜单，列出了当前Android源码支持的所有设备型号及其编译类型。如编译类型有“eng”、“user”、“userdebug”三种

![build1]({{ '/styles/images/androidbuild/build1.jpg' | prepend: site.baseurl  }})

    输入你要编译的项目前面的数字即可。

![build2]({{ '/styles/images/androidbuild/build2.jpg' | prepend: site.baseurl  }})

    选择之后可以完成Android编译环境的初始化过程，并且会打印出主要的环境变量：

![build3]({{ '/styles/images/androidbuild/build3.jpg' | prepend: site.baseurl  }})


**3、编译的第三步为make**
    常用的命令：make  -j8 2>&1 | tee build.log  

 - -j8 这里的 8 指的是线程数量，就是你要用几个线程去编译这个工程
 - 2>&1中2是标准错误，&1是标准输出，2>&1意思就是将标准错误输出到标准输出中
 - tee的作用同时输出到控制台和文件

    Make命令在执行的时候，默认会在当前目录找到一个Makefile文件，然后根据Makefile文件中的指令来对代码进行编译，在源码的跟目录下有一个Makefile文件，在make  -j8 2>&1 | tee build.log时候首先找到该文件，该文件比较简单，内容如下:

![build3]({{ '/styles/images/androidbuild/build4.jpg' | prepend: site.baseurl  }})

    调用了build/core下面的main.mk

    (1)根据ANDROID_BUILD_SHELL来选择编译系统用到的Shell

```
# Only use ANDROID_BUILD_SHELL to wrap around bash.
# DO NOT use other shells such as zsh.
ifdef ANDROID_BUILD_SHELL
SHELL := $(ANDROID_BUILD_SHELL)
else
# Use bash, not whatever shell somebody has installed as /bin/sh
# This is repeated in config.mk, since envsetup.sh runs that file
# directly.
SHELL := /bin/bash
endif
```
    (2)对MAKE_VERSION的检查

```
# Check for broken versions of make.
# (Allow any version under Cygwin since we don't actually build the platform there.)
ifeq (,$(findstring CYGWIN,$(shell uname -sm)))
ifneq (1,$(strip $(shell expr $(MAKE_VERSION) \>= 3.81)))
$(warning ********************************************************************************)
$(warning *  You are using version $(MAKE_VERSION) of make.)
$(warning *  Android can only be built by versions 3.81 and higher.)
$(warning *  see https://source.android.com/source/download.html)
$(warning ********************************************************************************)
$(error stopping)
endif
endif

```
    (3)设定第一个目标：DEFAULT_GOAL := droid,在用户输入make之后，如果不加任何参数，那么默认的目标就是droid

```
# This is the default target.  It must be the first declared target.
.PHONY: droid
DEFAULT_GOAL := droid
$(DEFAULT_GOAL):
```
    (4)包含config.mk：include $(BUILD_SYSTEM)/config.mk,config.mk是一个总括性的文件，它里面定义了各种module编译所需要使用的HOST工具以及如何来编译各种模块

```
# Set up various standard variables based on configuration
# and host information.
include $(BUILD_SYSTEM)/config.mk
```
    (5)检查当前编译的绝对路径是否纯在空格：ifneq ($(words $(shell pwd)),1)若是存在空格，会抛出警告并退出。

```
# Make sure that there are no spaces in the absolute path; the
# build system can't deal with them.
ifneq ($(words $(shell pwd)),1)
$(warning ************************************************************)
$(warning You are building in a directory whose absolute path contains)
$(warning a space character:)
$(warning $(space))
$(warning "$(shell pwd)")
$(warning $(space))
$(warning Please move your source tree to a path that does not contain)
$(warning any spaces.)
$(warning ************************************************************)
$(error Directory names containing spaces not supported)
endif

```

    (6)TARGET_BUILD_VARIANT 在buildspec.mk设定,这个参数决定了要安装的模块

```
ifneq ($(filter-out $(INTERNAL_VALID_VARIANTS),$(TARGET_BUILD_VARIANT)),)
$(info ***************************************************************)
$(info ***************************************************************)
$(info Invalid variant: $(TARGET_BUILD_VARIANT)
$(info Valid values are: $(INTERNAL_VALID_VARIANTS)
$(info ***************************************************************)
$(info ***************************************************************)
$(error stopping)
endif

```
    (7)在这里调用findleaves.py查找在subdirs的包含的子目录下所有的 Android.mk文件。

```
ifneq ($(dont_bother),true)
#
# Include all of the makefiles in the system
#

# Can't use first-makefiles-under here because
# --mindepth=2 makes the prunes not work.
#tyd niechunmiao modify 2015-04-20 ignore tyd folder  
subdir_makefiles := \
	$(shell build/tools/findleaves.py --prune=$(OUT_DIR) --prune=.repo --prune=.git --prune=tyd $(subdirs) Android.mk)

$(foreach mk, $(subdir_makefiles), $(info including $(mk) ...)$(eval include $(mk)))

endif # dont_bother
```
    (8)提取用户定义的定义的模块，用户定义的模块，不一定在系统的所有模块中有定义，分为: known_custom_modules:用户定义了，并且在系统中有定义。 unknown_custom_modules:用户没有定义，但系统系统中有定义。
```
# -------------------------------------------------------------------
# All module makefiles have been included at this point.
# -------------------------------------------------------------------


# -------------------------------------------------------------------
# Fix up CUSTOM_MODULES to refer to installed files rather than
# just bare module names.  Leave unknown modules alone in case
# they're actually full paths to a particular file.
known_custom_modules := $(filter $(ALL_MODULES),$(CUSTOM_MODULES))
unknown_custom_modules := $(filter-out $(ALL_MODULES),$(CUSTOM_MODULES))
CUSTOM_MODULES := \
	$(call module-installed-files,$(known_custom_modules)) \
	$(unknown_custom_modules)


```

    (9)这里调用了module-installed-files和add-required-deps函数，设定各个模块对各种库的依赖

```
define add-required-deps
$(1): | $(2)
endef

$(foreach m,$(ALL_MODULES), \
  $(eval r := $(ALL_MODULES.$(m).REQUIRED)) \
  $(if $(r), \
    $(eval r := $(call module-installed-files,$(r))) \
    $(eval t_m := $(filter $(TARGET_OUT_ROOT)/%, $(ALL_MODULES.$(m).INSTALLED))) \
    $(eval h_m := $(filter $(HOST_OUT_ROOT)/%, $(ALL_MODULES.$(m).INSTALLED))) \
    $(eval t_r := $(filter $(TARGET_OUT_ROOT)/%, $(r))) \
    $(eval h_r := $(filter $(HOST_OUT_ROOT)/%, $(r))) \
    $(eval t_m := $(filter-out $(t_r), $(t_m))) \
    $(eval h_m := $(filter-out $(h_r), $(h_m))) \
    $(if $(t_m), $(eval $(call add-required-deps, $(t_m),$(t_r)))) \
    $(if $(h_m), $(eval $(call add-required-deps, $(h_m),$(h_r)))) \
   ) \
 )

```


