---
layout: post
title:  "Android增加系统属性"
date:   2014-08-29 10:21:21 +0800
categories: Android
tag: Proprity
---

-----------------------


    由于在设置中添加了一个新的选项，用来控制打电话时是否关闭屏幕的选项，需要记录是否关闭的选项。在这里用到了系统属性，下面记录添加的过程。
    首先列出修改的文件列表：

```
frameworks/base/core/java/android/provider/setting.java
frameworks/base/packages/SettingsProvider/res/values/default.java
frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
```

    其中在第一个文件中setting.java添加如下内容：

```
/**
  * add by chuange
  * Value to specify if the user prefers the proximity
  * 1=yes, 0=no
  * @hide
  */
public static final String ENABLE_PSENSOR = "enable_psensor";
............
public static final String[] SETTINGS_TO_BACKUP = {
   ......
	    ENABLE_PSENSOR,                
   ......
}
```

    在default.java添加如下代码：

```
    <!--add tyd chuange default value psensor 20140829-->
    <integer name="def_enable_psensor">1</integer>
```

    最后一个文件是DatabaseHelper.java，代码如下:
```
    //add tyd chuange default value psensor 20140829
    loadSetting(stmt, Settings.System.ENABLE_PSENSOR, 1);
```

    通过添加以上三个文件的内容，就可以在系统中获取或者设置系统属性了，具体例子如下：
	
	
```
    packages/apps/Settings/src/com/android/settings/DisplaySetting.java
    packages/apps/InCallUI/src/com/android/incallui/ProximitySensor.java
```

    首先是DisplaySetting.java 
    获取系统属性：
```
    int isPenable = Settings.System.getInt(getContentResolver(),     	ENABLE_PSENSOR , 1);
```

    设置系统属性：
  
   
```
if(isPsensor){

    	 Settings.System.putInt(getContentResolver(), ENABLE_PSENSOR, 1);
     	 Xlog.d(TAG, "tyd get the value of isPsensor " + isPsensor);
 	
    }else{
  
    	 Settings.System.putInt(getContentResolver(), ENABLE_PSENSOR, 0);

	 }
```


    如果某一个属性在被赋值之后是不可改变的，则需要使用ro.tyd_psensor_control。其使用方法和之前的不同，需要修改的文件如下：

```
    packages/apps/Settings/src/com/mediatek/settings/FeatureOption.java
    device/mediatek/common/device.mk
    device/hsimobile/hf7_tns_sima/ProjectConfig.mk
```

    首先在FeatureOption.java中添加如下：FeatureOption只是一个工具类可以不要的，直接在使用的地方调用,还有要特别注意FeatureOption类引用的位置。

```
public static final boolean TYD_PSENSOR_CONTROL = SystemProperties.get("ro.tyd_psensor_control").equals("1");

```

```
    /*add by chuange 20140829 for tyd proximity sensor*/
    public static final boolean TYD_PSENSOR_CONTROL = getValue("ro.tyd_psensor_control");
    ......
    private static boolean getValue(String key) {
        return SystemProperties.get(key).equals("1");
    }

```

    在device.mk中添加如下：

```
    **#add by chuange 20140829 for tyd proximity sensor.**
    ifeq ($(strip $(TYD_PSENSOR_CONTROL)),yes)
    PRODUCT_PROPERTY_OVERRIDES += ro.tyd_psensor_control=1
    endif
```

    在ProjectConfig.mk中设置yes或者no

```
    **#add by chuange 20140829 for tyd proximity sensor**
    TYD_PSENSOR_CONTROL = no
```

    在DisplaySetting.java中使用

```
    import com.mediatek.settings.FeatureOption;

    boolean isPSensorControl = FeatureOption.TYD_PSENSOR_CONTROL;
    Log.d(TAG, "tyd isPSensorControl is " +isPSensorControl);
```


