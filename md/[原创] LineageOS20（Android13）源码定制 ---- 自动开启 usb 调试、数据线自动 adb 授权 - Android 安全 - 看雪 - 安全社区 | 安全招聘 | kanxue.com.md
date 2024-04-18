> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281293.htm)

> [原创] LineageOS20（Android13）源码定制 ---- 自动开启 usb 调试、数据线自动 adb 授权

[原创] LineageOS20（Android13）源码定制 ---- 自动开启 usb 调试、数据线自动 adb 授权

6 天前 1202

### [原创] LineageOS20（Android13）源码定制 ---- 自动开启 usb 调试、数据线自动 adb 授权

 [![](http://passport.kanxue.com/upload/avatar/781/899781.png?1712854564)](user-home-899781.htm) [smallnn](user-home-899781.htm) ![](https://bbs.kanxue.com/view/img/rank/5.png) 1  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 6 天前  1202

**网上关于自动 adb 授权的资料很多，但是大部分是安卓低版本的没有关于 Android13 的，如下分析怎么才能在不打开开发者模式的情况下，插入数据线即可授权 adb  
(不修改 ro.adb.secure 等相关安全相关属性，以免被检测)  
分为两部分：**

1.usb 调试按钮自动打开
--------------

在 frameworks/base/services/core/java/com/android/server/adb/AdbService.java systemReady() 函数内  
将

```
Settings.Global.putInt(mContentResolver,
                    Settings.Global.ADB_ENABLED, shouldEnableAdbUsb ? 1 : 0);

```

改为

```
Settings.Global.putInt(mContentResolver,
                    Settings.Global.ADB_ENABLED, 1);

```

2.adb 调试弹框自动授权
--------------

在 frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java onCreate 函数内结尾增加如下

```
public void onCreate(Bundle icicle) {
     
       /**
       *忽略代码，无需注释和改动
       **/
      //增加如下代码
       mAlwaysAllow.setChecked(true);
       onClick(null,-1);
   }

```

以上修改完成，编译刷机，即可实现。

  

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#源码框架](forum-161-1-127.htm)