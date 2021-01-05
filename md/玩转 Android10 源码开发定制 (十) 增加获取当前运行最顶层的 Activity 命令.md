> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483918&idx=1&sn=8894d70d8b62b4ab2f7d9f8424e1e642&chksm=ce07534bf970da5db8c3ed60ec881b88340f67af661c6e1a6c9eb30b0e7769f0b9bd177d5910&mpshare=1&scene=1&srcid=0105witG20vE1BQAaRdWr7Ng&sharer_sharetime=1609813342339&sharer_shareid=1f5aef717fb078f219d80bd9057d2404&key=6ef8794b749be7352e95e7b8e995a121a8f539a2f458edb39bd38999393474fd109d9c7d923ca911931744d1c6063de48b0e964f915218c0b9ffef75c49295cbb67f9412962eb083c3f1ddefdb0eb28904488b916c64f03ac0ccf56d923040c25c375457a6bd336207a3cfdbb8fdbf2b2bccda2a5ee99d44f63e71c51eec4d81&ascene=1&uin=MTEzMjI1NDQxNg%3D%3D&devicetype=Windows+10+x64&version=62090538&lang=zh_CN&exportkey=A9jd9tmx5gowsmy6fvl0TxM%3D&pass_ticket=4tucVLBWCpF%2FwZ3RqWcCTgXVFln7PzWYuRB3EZAkUSXNI4rUPsYkYlyPX7FayTjY&wx_header=0)

一、获取当前运行的顶层 Activity 的几种方式
--------------------------

在 Android 开发中，由于某些需求常常需要获取当前顶层的 Activity 信息。比如 App 中获取顶层 Activity 界面信息来判断某一个 app 是否在前台运行、统计某一个 app 的使用时长、更有恶意程序通过监听界面伪造 app 进行盗号以及欺诈、自动化开发中通过顶层 Activity 进行页面元素定位点击 (比如基于辅助功自动化、uiautomator 自动化) 等等操作。在逆向工程中，获取当前运行 app 运行顶层 activity 也比较常用。通过顶层 Activity 可以快速定位界面中的功能在哪一个页面。

在 Android app 开发中，获取顶层 Activity 有以下方式:

1.  低于 Android 5.0, 情况下使用 getRunningTasks
    

首先需要在 Androidmanifest 中添加权限

```
<uses-permission  android:name = "android.permission.GET_TASKS"/>


```

代码如下：

```
ActivityManager mAm = (ActivityManager)getSystemService(Context.ACTIVITY_SERVICE);
String activity_name = mAm.getRunningTasks(1).get(0).topActivity.getClassName();


```

2.  高于安卓 5.0 使用 UsageStatsManager 获取到应用程序的运行情况。
    

在 AndroidManifest 文件中添加权限:

```
<uses-permission android:/> 


```

启动授权页面，需要用户授权 app 获取应用使用情况统计权限。跳转授权页面参考:

```
Intent intent = new Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS);
context.startActivity(intent);


```

获取顶层 activity 参考代码:

```
public static getTopActivity()
{
  long endTime = System.currentTimeMillis();
  long beginTime = endTime - 10000;
  UsageStatsManager sUsageStatsManager =null;
  if (sUsageStatsManager == null) {
  sUsageStatsManager = (UsageStatsManager) context.getSystemService(Context.USAGE_STATS_SERVICE);
  }
   String result = "";
   UsageEvents.Event event = new UsageEvents.Event();
   UsageEvents usageEvents = sUsageStatsManager.queryEvents(beginT ime, endTime);
  while (usageEvents.hasNextEvent()) {
     usageEvents.getNextEvent(event);
  if (event.getEventType() == UsageEvents.Event.MOVE_TO_FOREGROUND) {
   result = event.getPackageName()+"/"+event.getClassName();
    }
  }
  if (!android.text.TextUtils.isEmpty(result)) {
  return result;
  }
  }
  return "";
}


```

除了 app 中使用代码获取顶层 Activity，也可以通过 adb 命令获取当前顶层的 Activity 信息，比如 windows 系统下 Android 10 手机可以用如下 adb 命令:

```
C:\Users\Qiang>adb shell dumpsys activity |findstr "mResume"
mResumedActivity: ActivityRecord{12f93fd u0 com.sohu.inputmethod.sogou/.SogouIMEHomeActivity t50}


```

由于 ActivityManager->getRunningTasks 获取顶层 Activity 比较准确一些，以下将采用 ActivityManager->getRunningTasks 方式来进行研究实现。

二、源码追踪 ActivityManager getRunningTasks 调用
-----------------------------------------

ActivityManager 源码路径位于:

> frameworks/base/core/java/android/app/ActivityManager.java

ActivityManager 中 getRunningTask 调用如下:

```
// getRunningTasks 方法实现
// 最终调用 getTaskService().getTasks
public List<RunningTaskInfo> getRunningTasks(int maxNum) throws SecurityException {
   try {
       return getTaskService().getTasks(maxNum);
   } catch (RemoteException e) {
         throw e.rethrowFromSystemServer();
    }
}

# getTaskService 方法实现
private static IActivityTaskManager getTaskService() {
return ActivityTaskManager.getService();
}


```

由以上代码分析可知，最终调用的是 ActivityTaskManager.getService().getTasks。接下来分析 ActivityTaskManager 中的调用情况。

ActivityTaskManager 源码路径:

> frameworks/base/core/java/android/app/ActivityTaskManager.java

ActivityTaskManager 中 getService 实现如下:

```
//getService 实现代码
public static IActivityTaskManager getService() {
   return IActivityTaskManagerSingleton.get();
}

//IActivityTaskManagerSingleton 初始化代码
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =new Singleton<IActivityTaskManager>() {
@Override
protected IActivityTaskManager create() {
    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
    return IActivityTaskManager.Stub.asInterface(b);
}
};


```

ActivityTaskManager.getService().getTasks 调用最终变成了 IActivityTaskManager.getTasks。在安卓源码中没有找到 IActivityTaskManager.java 文件，只找到 IActivityTaskManager.aidl 文件，说明 IActivityTaskManager.getTasks 调用最终转化了 binder 的进程间调用。安卓中 aidl 是用于 binder 通信定义服务器和客户端通信接口的一种描述语言，源码编译的时候会将 aidl 文件转化为 java 文件。

IActivityTaskManager.aidl 文件路径如下:

> frameworks/base/core/java/android/app/IActivityTaskManager.aidl

在源码中搜索关键字 “extends IActivityTaskManager" 找到 ActivityTaskManagerService 文件使用了。那么说明 ActivityTaskManagerService 最终实现了 getTasks 的函数功能。以下追踪 ActivityTaskManagerService 中的调用情况:

ActivityTaskManagerService 代码路径:

> frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java

ActivityTaskManagerService 中 getTasks 代码如下:

```
//getTasks 实现代码，调用了 getFilteredTasks
@Override
public List<ActivityManager.RunningTaskInfo> getTasks(int maxNum) {
     return getFilteredTasks(maxNum, ACTIVITY_TYPE_UNDEFINED, WINDOWING_MODE_UNDEFINED);
}

// getFilteredTasks 实现代码
@Override
public List<ActivityManager.RunningTaskInfo> getFilteredTasks(int maxNum,@WindowConfiguration.ActivityType int ignoreActivityType,@WindowConfiguration.WindowingMode int ignoreWindowingMode) {
  final int callingUid = Binder.getCallingUid();
  final int callingPid = Binder.getCallingPid();
  final boolean crossUser = isCrossUserAllowed(callingPid, callingUid);
  final int[] profileIds = getUserManager().getProfileIds(UserHandle.getUserId(callingUid), true);
  ArraySet<Integer> callingProfileIds = new ArraySet<>();
  for (int i = 0; i < profileIds.length; i++) {
    callingProfileIds.add(profileIds[i]);
  }
  ArrayList<ActivityManager.RunningTaskInfo> list = new ArrayList<>();
  synchronized (mGlobalLock) {
  if (DEBUG_ALL) Slog.v(TAG, "getTasks: max=" + maxNum);
  final boolean allowed = isGetTasksAllowed("getTasks", callingPid, callingUid);
  mRootActivityContainer.getRunningTasks(maxNum, list, ignoreActivityType,
ignoreWindowingMode, callingUid, allowed, crossUser, callingProfileIds);
}
  return list;
}


```

通过以上分析, 已经知道系统如何实现 getRunningTask 函数功能的流程。借鉴 getRunningTask 实现，要增加一个获取顶层 Activity 的接口，只需要在 IActivityTaskManager.aidl 中增加一个接口，仿照 ActivityTaskManagerService 中 getTasks 的实现方式即可。

ActivityManager getRunningTasks 大概的一个调用流程总结如下, 图中省略了许多 binder 交互的细节:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew54333pKWxIQnpthKhz8ZqztaMqJ1rJUjSkyJ5d4w8NibM07UyMb8AE0icZagk2txpEd7XTIkdryQ2jV4A/640?wx_fmt=png)

三、为系统增加 getTopActivity 接口
-------------------------

1.  在 IActivityTaskManager.aidl 文件中添加 getTopActivity。添加内容如下:
    

```
    ///ADD START
    String getTopActivity();
    ///ADD END


```

2.  在 ActivityTaskManagerService 中重写 getTopActivity 方法，完成真正的功能，由于我们提供的接口是 adb shell 调用，所以此处没有处理权限检测，默认情况下 adb shell 有权限访问的。代码如下:
    

```
///ADD START
@Override
public String getTopActivity()
{
  List <ActivityManager.RunningTaskInfo> appTasks = getTasks(1);
  if (null != appTasks && !appTasks.isEmpty()) {
    return appTasks.get(0).topActivity.getPackageName()+"/"+appTasks.get(0).topActivity.getClassName();
 }
   return "";
}
///ADD END


```

以上工作做好之后，我们就可以通过如下代码调用获取顶层 activity 了。代码如下：

```
String topString=ActivityTaskManager.getService().getTopActivity();


```

四、系统增加 gettopactivity 命令
------------------------

1.  创建目录 frameworks/base/cmds/gettopactivity, 用来存放命令源码
    

```
> qiang@ubuntu:~/lineageOs$ mkdir -p frameworks/base/cmds/gettopactivity


```

2.  在 gettopactivity 目录创建调用新增 getTopActivity 方法的源码文件 src/com/android/commands/atm/Atm.java，在 Atm.java 中添加如下代码:
    

```
package com.android.commands.atm;

import android.app.ActivityTaskManager;
import android.app.IActivityTaskManager;
import android.os.RemoteException;
import android.os.ResultReceiver;
import android.os.SELinux;
import android.os.ServiceManager;
import android.util.AndroidException;
import java.io.File;
import java.io.FileDescriptor;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.PrintStream;
public class Atm  {

    private IActivityTaskManager mAtm;
    public static void main(String[] args) {
        (new Atm()).run();
    } 
    public void run() throws Exception {

        //获取ActivityTaskManager对象
        mAtm = ActivityTaskManager.getService();
        if (mAtm == null) {
            System.err.println(NO_SYSTEM_ERROR_CODE);
            throw new AndroidException("Can't connect to activity task manager; is the system running?");
        }
     try {
            //调用getTopActivity方法，获取顶层运行activity信息
            String  topString=mAtm.getTopActivity();
         System.out.println("TopActivity:"+topString);
        } catch (RemoteException e) {
            System.err.println(NO_SYSTEM_ERROR_CODE);
            throw new AndroidException("Can't call activity task manager; is the system running?");
        }
  
    }

}


```

3.  在 gettopactivity 目录下增加 gettopactivity shell 脚本, 作为 adb shell 调用的命令。脚本内容如下:
    

```
#!/system/bin/sh
base=/system
export CLASSPATH=$base/framework/gettopactivity.jar
exec app_process $base/bin com.android.commands.atm.Atm "$@"


```

4.  在 gettopactivity 目录下添加预编译可执行程序 Android.mk 脚本，内容如下:
    

```
#///ADD START
#///ADD END
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)
LOCAL_SRC_FILES := \
    $(call all-java-files-under, src) 
LOCAL_MODULE := gettopactivity
include $(BUILD_JAVA_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := gettopactivity
LOCAL_SRC_FILES := gettopactivity
LOCAL_MODULE_CLASS := EXECUTABLES
LOCAL_MODULE_TAGS := optional
include $(BUILD_PREBUILT)



```

5.  将 gettopactivity 模块加入到编译文件链接中。在之前的文章中已经讨论过了预编译二进制模块加入编译链放入文件路径:**make/target/product/base_system.mk** 中。加入之后的部分内容如下:
    

```
#///ADD START
# add frida server to system
#///ADD END

# Base modules and settings for the system partition.
PRODUCT_PACKAGES += \
    myfridaserverarm64 \
    gettopactivity  \
    ...(省略)


```

五、编译系统测试效果
----------

源码根目录执行编译：

> source build/envsetup.sh  
> breakfast oneplus3  
> brunch oneplus3  

刷机测试效果:

```
C:\Users\Qiang>adb shell gettopactivity
TopActivity:com.starbucks.cn/com.starbucks.cn.ui.StarbucksLaunchActivity


```

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。关注公众号第一时间接收更新文章。**![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew54333pKWxIQnpthKhz8ZqztaMDAKKicC7l8EicDlSFJsOUUfFfGlb3XpIDDVgM82z5qvribloeFfhibAzAw/640?wx_fmt=jpeg)

  

---