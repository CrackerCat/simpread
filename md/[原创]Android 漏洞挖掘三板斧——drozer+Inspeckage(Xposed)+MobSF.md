> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269196.htm)

> [原创]Android 漏洞挖掘三板斧——drozer+Inspeckage(Xposed)+MobSF

Android 漏洞挖掘三板斧——drozer+Inspeckage(xposed)+MobSF
================================================

目录

*   [Android 漏洞挖掘三板斧——drozer+Inspeckage(xposed)+MobSF](#android漏洞挖掘三板斧——drozer+inspeckage(xposed)+mobsf)
*            [一、前言](#一、前言)
*            二、Android APP 漏洞介绍
*            [三、工具的安装和使用](#三、工具的安装和使用)
*                    [1.drozer](#1.drozer)
*                            [（1）drozer 介绍](#（1）drozer介绍)
*                            [（2）安装准备](#（2）安装准备)
*                            [（3）安装](#（3）安装)
*                            [（4）基本使用](#（4）基本使用)
*                    [2.Inspeckage(xposed)](#2.inspeckage(xposed))
*                            [（1）Inspeckage 介绍](#（1）inspeckage介绍)
*                            [（2）安装准备](#（2）安装准备)
*                            [（3）安装](#（3）安装)
*                            [（4）基本使用](#（4）基本使用)
*                    [3.MobSF](#3.mobsf)
*                            [（1）MobSF 基本介绍](#（1）mobsf基本介绍)
*                            [（2）安装准备](#（2）安装准备)
*                            [（3）安装](#（3）安装)
*                            [（4）基本使用](#（4）基本使用)
*            [四、总结](#四、总结)
*            [五、参考网址](#五、参考网址)

[](#一、前言)一、前言
-------------

最近在学习 Android APP 客户端漏洞挖掘过程中，对 Android APP 端漏洞挖掘做了一个基本的梳理总结本节主要是在介绍 Android APP 漏洞挖掘过程中，使用常见的 Android 漏洞挖掘工具的安装和使用办法，帮助 Android 漏洞挖掘人员提供便利。本文里面一部分的介绍采摘与网络博客，大家可以点击对应的网址进行查看。

 

为什么选择 drozer+Inspeckage+Mobsf 三个工具呢，这是因为在我进行 Android APP 漏洞挖掘的过程中，这三个工具很好的提供了自动化的分析和一些对应的 hook 技术，极大的方便了 Android APP 漏洞挖掘工作。下面将依次介绍每一种工具的安装和使用，后续继续会用一些案例来实际操作。

二、Android APP 漏洞介绍
------------------

根据中国互联网协会 APP 数据安全测评服务工作组在 2020 年发布的《移动应用安全形势研究报告》中显示，2020 年度收录存在安全漏洞威胁的 APK 860 万余个，同 一 App 普遍存在多个漏洞。其中存在的 Janus 漏洞风险 App 数量最多，占监测总量的 78.13%；其次是 Java 代码加壳检 测，占总量的 62.45%；排在第三位的是动态注册 Receiver 风险，占总量的 61.16%。

 

![](https://bbs.pediy.com/upload/attach/202109/905443_AFCRFHZQXAYM697.png)

 

根据爱加密的 Android 客户端常见漏洞调研显示，Android 方向的漏洞总共可以分为三个方面：组件安全、业务安全、数据安全。

 

![](https://bbs.pediy.com/upload/attach/202109/905443_TTUX43JVTA4PFC3.png)

 

我们可以看出针对不同的安全问题，我们在进行 Android APP 漏洞挖掘的过程中，也应该有针对性的进行漏洞挖掘。我们更加具体的组件分类，可以大致查看一些 Android APP 常见安全漏洞：https://ayesawyer.github.io/2019/08/21/Android-App%E5%B8%B8%E8%A7%81%E5%AE%89%E5%85%A8%E6%BC%8F%E6%B4%9E/

 

![](https://bbs.pediy.com/upload/attach/202109/905443_6J7WGQ2HGW4RUP4.png)

 

根据 Android 安全分析平台的 Android APP 审计系统显示，这也为我们进行 Android APP 客户端的漏洞挖掘提供了详细的方案：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_Q5T53MV76D62MYJ.png)

 

综上，我们对 Android APP 漏洞挖掘和 Android 安全测试就有了一个初步的认识，接下来我们来看这三个神器具体是怎么安装和使用，帮助我们移动安全分析人员提供便利。

[](#三、工具的安装和使用)三、工具的安装和使用
-------------------------

### 1.drozer

#### [](#（1）drozer介绍)（1）drozer 介绍

```
drozer是一款针对Android系统的安全测试框架，可以分成两个部分：其一是“console”，它运行在本地计算机上；其二是“server”，它是一个安装在目标Android设备上的app，当使用console与Android设备交互时，就是把Java代码输入到运行在实际设备上的drozer代理(agent)中。
根据drozer官方的描述，drozer主要是有助于Android研究人员去测试一些共享的Android漏洞，对于远程攻击，可以生成shellcode来帮助开发人员将drozer Agent 部署为远程管理员工具，从而最大程度利用设备。
drozer是一个全面的安全审计和攻击框架，可以进行更快的Android安全评估，通过自动化繁琐和耗时的工作，帮助减少Android安全评估所花费的时间。还可以针对真实的Android设备进行测试，drozer不需要启用USB调试或其他开发功能，还可以自动化和扩展，测试公共漏洞的暴露程度。

```

#### [](#（2）安装准备)（2）安装准备

```
（1）drozer官方地址：https://labs.f-secure.com/tools/drozer/
（2）drozer github：https://github.com/mwrlabs/drozer
（3）下载及drozer用户手册：https://labs.mwrinfosecurity.com/tools/drozer/

```

#### [](#（3）安装)（3）安装

我们在这里主要列举 Windows 端和 Linux 端的安装：

 

**1.Windows 端：**

 

我们在 Windows 安装时，首先需要配置环境：

```
jdk1.8
python 2.7
android sdk
其中python的版本必须为2.7版本，因为drozer现在只支持python2.7版本

```

​ 我们先从官网上下载 drozer（RPM）和 agent.apk

 

![](https://bbs.pediy.com/upload/attach/202109/905443_5VTVJ2P7GAR8GSK.png)

 

**服务端安装：**

 

我们下载 drozer（RPM）后解压，然后点击 setup 安装，一路默认安装就可以了，它会自动安装到 C:\drozer 文件夹下

 

![](https://bbs.pediy.com/upload/attach/202109/905443_XUM4CR7WJ8UNNBV.png)

 

我们检测是否成功安装，进入 bin 目录下，在 cmd 中执行 drozer.bat，出现下面的图就说明成功安装了

 

![](https://bbs.pediy.com/upload/attach/202109/905443_A4E9GYXNKAPKD89.png)

 

**客户端安装：**

 

我们将 agent.apk 安装到手机上：

```
adb install agent.apk

```

安装完成后手机启动 drozer，点击右下角开启端口转发按钮

 

![](https://bbs.pediy.com/upload/attach/202109/905443_DKKTWKX8HAPD7M4.png)

 

然后我们在电脑的终端中输入命令转发端口：

```
adb forward tcp:31415 tcp:31415

```

最后我们在终端中进入 drozer 的安装目录下，输入命令运行：

```
drozer console connect

```

![](https://bbs.pediy.com/upload/attach/202109/905443_YP58NZAX3EFVUYS.png)

 

此时我们就可以正常使用 drozer 框架，来进行我们的测试工作了。

 

**2.Linux 端安装（Kali）**

 

Linux 上的安装因为需要的库比较多，所以很容易出错，大家最好按照这个步骤一步步来

```
首先，我们需要将python环境配置成python 2.7.0
wget https://github.com/FSecureLABS/drozer/releases/download/2.4.4/drozer-2.4.4-py2-none-any.whl ##下载drozer
apt-get --assume-yes install python-pip
pip2 install wheel
pip2 install pyyaml
pip2 install pyhamcrest
pip2 install protobuf
pip2 install pyopenssl
pip2 install twisted
pip2 install service_identity
pip2 install drozer-2.4.4-py2-none-any.whl
安装jdk8: apt-get install --assume-yes openjdk-8-jdk-headless
安装adb: apt-get --assume-yes install adb ##已经有adb不需要再次安装
下载客户端并安装：https://labs.f-secure.com/tools/drozer/
在客户端中打开31415端口，然后进行端口转发：
adb forward tcp:31415 tcp:31415
最后启动drozer：drozer console connect
参考网址：
https://blog.csdn.net/LoopherBear/article/details/84030567
https://github.com/FSecureLABS/drozer/issues/350
https://github.com/FSecureLABS/drozer/issues/357#issuecomment-652669536

```

一些 drozer 安装常规问题解决方案：https://blog.csdn.net/Jession_Ding/article/details/82528142

#### [](#（4）基本使用)（4）基本使用

drozer 的一些常用指令：

```
> list  //列出目前可用的模块，也可以使用ls
> help app.activity.forintent       //查看指定模块的帮助信息
> run app.package.list      //列出android设备中安装的app
> run app.package.info -a com.android.browser       //查看指定app的基本信息
> run app.activity.info -a com.android.browser      //列出app中的activity组件
> run app.activity.start --action android.intent.action.VIEW --data-uri  http://www.google.com  //开启一个activity，例如运行浏览器打开谷歌页面
> run scanner.provider.finduris -a com.sina.weibo       //查找可以读取的Content Provider
> run  app.provider.query content://settings/secure --selection "    //读取指定Content Provider内容
> run scanner.misc.writablefiles --privileged /data/data/com.sina.weibo     //列出指定文件路径里全局可写/可读的文件
> run shell.start       //shell操作
> run tools.setup.busybox       //安装busybox
> list auxiliary        //通过web的方式查看content provider组件的相关内容
> help auxiliary.webcontentresolver     //webcontentresolver帮助
> run auxiliary.webcontentresolver      //执行在浏览器中以http://localhost:8080即可访问
以sieve示例
> run app.package.list -f sieve         //查找sieve应用程序
> run app.package.info -a com.mwr.example.sieve         //显示app.package.info命令包的基本信息
> run app.package.attacksurface com.mwr.example.sieve   //确定攻击面
> run app.activity.info -a com.mwr.example.sieve         //获取activity信息
> run app.activity.start --component com.mwr.example.sieve com.mwr.example.sieve.PWList     //启动pwlist
> run app.provider.info -a com.mwr.example.sieve        //提供商信息
> run scanner.provider.finduris -a com.mwr.example.sieve        //扫描所有能访问地址
> run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/--vertical  //查看DBContentProvider/Passwords这条可执行地址
> run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "'"   //检测注入
> run app.provider.read content://com.mwr.example.sieve.FileBackupProvider/etc/hosts    //查看读权限数据
> run app.provider.download content://com.mwr.example.sieve.FileBackupProvider/data/data/com.mwr.example.sieve/databases/database.db /home/user/database.db //下载数据
> run scanner.provider.injection -a com.mwr.example.sieve       //扫描注入地址
> run scanner.provider.traversal -a com.mwr.example.sieve
> run app.service.info -a com.mwr.example.sieve         //查看服务

```

我们在测试过程中，常用测试指令：

 

四大组件模块：

```
(1)Activity:
app.activity.forintent -- 找到可以处理已指定的包
app.activity.info -- 获取activity组件信息
app.activity.start -- 开启activity组件
scanner.activity.browsable -- 获取可从web浏览器调用的所有可浏览的activity组件
 
(2)Service:
app.service.info -- 获取service组件信息
app.service.send -- 向服务组件发送消息并显示答复
app.service.start -- 开启service组件
app.service.stop -- 停止service组件
 
(3)Content Provider:
app.provider.columns -- 在内容提供程序中列出列
app.provider.delete -- 在内容提供程序中删除
app.provider.download -- 在内容提供程序中下载支持文件
app.provider.finduri -- 在包中查找引用的内容URIS
app.provider.info -- 获取Content Provider组件信息
app.provider.insert -- 插入到Content Provider组件中
app.provider.query -- 查询Content Provider组件
app.provider.read -- 从支持文件的Content Provider读取
app.provider.update -- 更新Content Provider的记录
scanner.provider.finduris -- 搜索可从上下文中查询的Content Provider
scanner.provider.injection -- 测试Content Provider的注入漏洞
scanner.provider.sqltables -- 查找可通过SQL注入漏洞访问的表
scanner.provider.traversal -- 测试Content Provider的基本目录遍历漏洞
 
(4)Broadcast Receivers:
app.broadcast.info -- 获取有关广播接收器的信息
app.broadcast.send -- 带目的发送广播
app.broadcast.sniff -- 注册一个能嗅出特定意图的广播接收器

```

获取 APP 包信息：

```
app.package.attacksurface------获取包攻击面
 
app.package.backup------列出使用备份API的包(在标记“允许备份”时返回true)
 
app.package.debuggable------查找可调试包
 
app.package.info------获取有关已安装软件包的信息
 
app.package.launchintent------获取包的启动意图
 
app.package.list------列出程序包
 
app.package.manifest------获取包的AndroidManifest.xml
 
app.package.native------查找嵌入在应用程序中的本地库
 
app.package.shareduid------查找具有共享uid的包

```

```
1.连接drozer：
drozer.bat console connect
 
2.列出详细APP信息：
run app.package.info -a com.xxx.xxxx
 
3.查看APP的配置信息
run app.package.manifest com.xxx.xxxx
 
3.分析是否存在攻击攻击点
run app.package.attacksurface com.xxx.xxxx
 
==============================================
Activity测试：
4.获取Activity信息
命令 run app.activity.info -a
示例 run app.activity.info -a com.xxx.xxxx
 
5.Activity跳转
命令：run app.activity.start --component 软件包名 软件包名.对应exported的activtiy
示例：run app.activity.start --component com.example.test com.example.test.Activity
 
==============================================
Service测试：
6.获取service信息
命令 run app.service.info -a
示例 run app.service.info -a com.xxx.xxxx
 
7.发送service服务
命令：run app.service.start --component 软件包名 软件包名.对应exported的activtiy --extra 数据
示例：run app.service.start --component com.example.test com.example.test.Activity --extra string phone 12345678901 --extra string content Hello
 
==============================================
Broadcast Receivers测试：
8.获取Broadcast信息
命令 run app.broadcast.info -a
示例 run app.broadcast.info -a com.xxx.xxxx
 
9.注册一个能嗅出特定意图的广播接收器
命令：app.broadcast.sniff --action "活动"
示例：app.broadcast.sniff --action "ddns.actiton.Token"
 
10.发送广播
命令：run app.broadcast.send --action 广播名 --extra string name lisi
示例：run app.broadcast.send --action org.owasp.goatdroid.fourgoats.SOCIAL_SMS --extra string phoneNumber 1234 --extra string message dog
 
==============================================
contentProvider测试：
11.获取contentProvider信息
命令：run app.provider.info -a
示例：run app.provider.info -a com.xxx.xxxx
 
12.获取所有可访问的Uri
命令 run scanner.provider.finduris -a
示例 run scanner.provider.finduris -a com.xxx.xxxx
 
13.检测SQL注入
命令 run scanner.provider.injection -a
示例 run scanner.provider.injection -a com.xxx.xxxx
 
14.检测目录遍历
命令 run scanner.provider.traversal -a
示例 run scanner.provider.traversal -a com.xxx.xxxx
 
15.进行SQL注入
命令 run app.provider.query [--projection] [--selection]
示例 run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/
 
列出所有表 run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "* FROM SQLITE_MASTER WHERE type='table';--"
获取单表（如Key）的数据 run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "* FROM Key;--"
 
16.读取文件系统下的文件
示例 run app.provider.read content://com.mwr.example.sieve.FileBackupProvider/etc/hosts
 
17.下载数据库文件到本地
示例 run app.provider.download content://com.mwr.example.sieve.FileBackupProvider/data/data/com.mwr.example.sieve/databases/database.db d:/database.db

```

### 2.Inspeckage(xposed)

#### [](#（1）inspeckage介绍)（1）Inspeckage 介绍

```
Inspeckage是一个用来动态分析安卓app的xposed模块。Inspeckage对动态分析很多常用的功能进行了汇总并且内建一个webserver。整个分析操作可以在友好的界面环境中进行。

```

#### [](#（2）安装准备)（2）安装准备

```
Xposed框架安装
Inspeckage模块网址：http://repo.xposed.info/module/mobi.acpm.inspeckage  //也可以直接到Xposed模块库中搜索

```

```
Xposed框架安装：
（1）4.4以下Android版本安装比较简单，只需要两步即可
    1.对需要安装Xposed的手机进行root
    2.下载并安装xposedInstaller,之后授权其root权限，进入app点击安装即可
    但是由于官网不在维护，导致无法直接通过xposedinstaller下载补丁包
（2）Android 5.0-8.0 由于5.0后出现ART，所以安装步骤分成两个部分：xposed.zip 和
    XposedInstaller.apk,zip文件是框架主体，需要进入Recovery后刷入，apk文件用于Xposed管理
    1.完成对手机的root，并刷入reconvery(比如twrp),使用Superroot
    2.下载你对应的zip补丁包，并进入recovery刷入
    3.重启手机，安装xposedInstaller并授予root权限即可
    官网地址：https://dl-xda.xposed.info/framework/
（3）由于Android 8.0后，Xposed官方作者没有再对其更新，我们一般就使用国内大佬riyu的Edxposed框架
    Magisk + riyu + Edxposed
    具体安装过程，大家百度搜索

```

#### [](#（3）安装)（3）安装

我们先下载安装 Inspeckage 模块，在 xposed 中勾选：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_7JNDBXRN6G3SXGS.png)

 

然后手机重启，激活 Xposed 中该模块，就完成正常的安装了

#### [](#（4）基本使用)（4）基本使用

**客户端：**

 

我们进入 Inspeckage 应用程序，可以配置其相关的信息：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_7SFVBNQ8X93ZVPN.png)

```
(1)序号1 Only user app : 默认只显示用户安装的APP,关闭可以显示系统自带的APP
(2)序号2 选择准备要测试的APP列表，这里是我们测试的目标APP
(3)序号3 表示我们的Inspeckage模块是否启动，如果这里为红色，说明可能没有安装Xposed框架，或者没有启动Inspeckage模块
(4)序号4 上面为我们手机的局域网地址，下面为我们USB访问的地址，也是我们电脑上主界面访问地址
(5)序号5 我们在访问主界面前，需要进行端口转发
(6)序号6 启动对应APP 我们可以在主界面查看其相关信息

```

我们查看 APP 的配置界面：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_5UEE8FEEEEP5V8Y.png)

```
（1）序号1 我们连接的地址，我们可以全选，也可以设置成手机局域网或主机地址，这样我们在主界面访问时就需要输入对应ip地址
（2）序号2 服务端端口号 序号3 Web套接字端口号  我们对端口号可以自定义

```

客户端使用步骤：

```
（1）我们首先在choose target中选择目标应用程序，然后我们点击Launch，启动该目标程序
（2）然后我们进行端口转发,转发手机的8008端口到本地：adb forward tcp:8008 tcp:8008
（3）接着我们在电脑上访问 http://127.0.0.1:8008 就可以看到Inspeckage的web界面。(如果web也买你没有输出结果，查看APP is running是否为true，Logcat左边分那个自动刷新按钮是否开启)

```

**服务端：**

 

我们在电脑上访问 http://127.0.0.1:8008，就可以进入服务端的 web 界面

 

首先我们来介绍 Tag 界面：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_QMMPMUB9365Z75H.png)

 

Data dir：数据文件树

 

![](https://bbs.pediy.com/upload/attach/202109/905443_7W6D6BMFKS46QNA.png)

 

主要提供我们 APP 运行过程中一些数据存储的位置

 

主界面的各字段含义依次为：

```
Logcat                               实时查看该app的logcat输出
Tree View                         可以实时浏览app的数据目录并直接下载文件到本地
Package Information     应用基本信息（组件信息、权限信息、共享库信息）
Shared Preferences       LOG：app XML文件读写记录；Files：具体XML写入内容
Serialization                     反序列化记录
Crypto                                常见加解密记录（KEY、IV值）
Hash                                 常见的哈希算法记录
SQLite                               SQLite数据库操作记录
HTTP                                 HTTP网络请求记录
File System                      文件读写记录
Misc.                                  调用Clipboard,URL.Parse()记录
WebView                          调用webview内容                 
IPC                                     进程之间通信记录
+Hooks                             运行过程中用户自定义Hook记录
参考网址：https://blog.csdn.net/tom__chen/article/details/78216732

```

我们进入设置界面：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_Y4N6FF5GH8S7VYD.png)

 

Replace 界面：

 

Replace 界面主要是用来替换 hook 的方法参数和返回值的，所以我们顺便看一下 hook 界面

 

![](https://bbs.pediy.com/upload/attach/202109/905443_7XVMA7JA6WTZX4F.png)

 

点击新建 hook 界面

 

![](https://bbs.pediy.com/upload/attach/202109/905443_DF6JTHR49GXQHFE.png)

 

点击替换界面

 

![](https://bbs.pediy.com/upload/attach/202109/905443_P8GG44NG96UW52Y.png)

 

Location 界面：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_YAPDB2V92VE7DC7.png)

 

这里官方是指可以替换 GPS 位置信息，不过平时在使用过程中用的并不多

 

Fingerprint（指纹信息）界面：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_JD98R9QQ2BCRUH9.png)

 

这个功能界面还是十分强大的，我们可以在这里替换一些我们设备常见的参数值，这样可以在我们进行一些测试工作的时候，可以绕过一些检测，比如我们可以更改 IMEI、IMSI 等等

 

Tip 界面：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_YX3PR5NUTYA3SJQ.png)

 

这里主要是介绍我们在 Android 分析操作过程中的一些检测方法，一些例子

 

Logcat 界面：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_735KQH3YBRVFEXU.png)

 

这里主要提供我们在程序运行过程中的一些日志信息，和我们 ddms 的作用一致

```
总结：
    Inspeckage是一款功能强大的Android安全测试工具，为用户提供了可视化的UI界面，很大程度上帮助用户去进行测试工作，可以对APP应用的基本配置信息进行检测，而且还可以提供用户简单的hook操作，用户可以很方便并且可视化的进行一些hook操作，还可以去修改设备和设备上应用程序的很多基本属性，还可以添加代理，绕过一些证书的固定。
参考网址：
    官方网址：https://github.com/ac-pm/Inspeckage#information-gathering
            https://repo.xposed.info/module/mobi.acpm.inspeckage
    博客网址：https://blog.csdn.net/tom__chen/article/details/78216732

```

### 3.MobSF

#### [](#（1）mobsf基本介绍)（1）MobSF 基本介绍

```
移动安全框架（MobSF）是一种自动化的移动应用程序（Android / iOS / Windows）测试框架，能够执行静态，动态和恶意软件分析。 它可用于 Android，iOS 和 Windows 移动应用程序的有效和快速安全分析，并支持二进制文件（APK，IPA 和 APPX）和压缩源代码。 MobSF 可以在运行时为 Android 应用程序进行动态应用程序测试，并具有由 CapFuzz（一种特定于 Web API 的安全扫描程序）提供支持的 Web API 模糊测试。

```

#### [](#（2）安装准备)（2）安装准备

```
安装MobSF一般有两种方式，一种是使用drozer安装，另外是使用源码安装，我们这里仅显示源码安装案例
官方地址：https://github.com/MobSF/Mobile-Security-Framework-MobSF

```

#### [](#（3）安装)（3）安装

**Windows 安装：**

 

安装环境

```
Windows10
Python 3.7
jdk 1.8.0

```

安装要求

```
安装 Git：https://git-scm.com/download/win
安装 Python3.7（3.8会出现错误)：https://www.python.org/ftp/python/3.7.9/python-3.7.9-amd64.exe
安装 JDK 8+：https://www3.ntu.edu.sg/home/ehchua/programming/howto/JDK_Howto.html
安装  Microsoft Visual C++ Build Tools: https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16
安装  OpenSSL: https://slproweb.com/products/Win32OpenSSL.html
安装  wkhtmltopdf: https://wkhtmltopdf.org/downloads.html //wkhtmltopdf主要是为了将生成的报告转换成pdf版本
wkhtmltopdf 操作指南：https://github.com/JazzCore/python-pdfkit/wiki/Installing-wkhtmltopdf
将包含 wkhtmltopdf 二进制文件的文件夹添加到环境变量PATH

```

安装步骤：

```
步骤1：下载好项目之后，可以重命名项目文件夹名称MobSf，打开cmd窗口进入该项目目录。将项目内的requirements.txt打开，最后一行libsast==1.2.2改为libsast==1.3.4

```

```
步骤2：安装OpenSSL和wkhtmltopdf，并配置好wkhtmltopdf环境后，我们在终端进入文件夹，然后运行run.bat

```

```
步骤3：我们打开浏览器，在输入网址：127.0.0.1:8000，如果需要修改默认端口，可以在run.bat中修改SET conf="0.0.0.0:8000"中的端口号

```

![](https://bbs.pediy.com/upload/attach/202109/905443_FWP8EFJJJ73BKBS.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_M9CSZPCRUFY5ANZ.png)

 

一般我们在运行的时候，会出现一些报错，例如：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_4TR8U2GXSJSHRXN.png)

```
原因解析：
    由于众所周知的网络原因，导致无法访问一些域名如raw.githubusercontent.com这个域名
    解决办法：
    步骤1：通过IPAddress.com首页，输入raw.githubusercontent.com查询到真实IP地址如：185.199.108.133
    步骤2：在本地电脑host文件中添加记录185.199.108.133 raw.githubusercontent.com即可

```

例如：IPAddress.com 首页

 

![](https://bbs.pediy.com/upload/attach/202109/905443_88UWDFEMZCJ7QK8.png)

 

然后我们进入 C:\WINDOWS\system32\drivers\etc , 修改 hosts 文件并保存

 

![](https://bbs.pediy.com/upload/attach/202109/905443_525P9XK45NYUSQY.png)

 

然后我们重新启动 run.bat，再输入 127.0.0.1，就可以发现正常的进入不报错误

 

![](https://bbs.pediy.com/upload/attach/202109/905443_GM2F8E96DJE6HGA.png)

 

同理在后面访问过程中，出现类似错误按照此解决方案就可以解决

 

![](https://bbs.pediy.com/upload/attach/202109/905443_2Q6RRAD78H7MD78.png)

 

但是由于这一般是由于网络受限原因导致，所以可能改了仍然会存在报错，但是一般不会影响正常使用，要彻底解决可以修改代码，大家可以尝试一下

 

**docker 安装：**

```
（1）下载镜像：docker pull opensecurity/mobile-security-framework-mobsf
（2）启动容器：docker run -it -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest

```

```
参考网址：
https://www.mad-coding.cn/2019/10/11/%E4%BD%BF%E7%94%A8docker%E5%AE%89%E8%A3%85%E7%A7%BB%E5%8A%A8%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%EF%BC%88MobSF%EF%BC%89/#0x01-%E5%BC%80%E5%A7%8B%E5%AE%89%E8%A3%85

```

**Linux 安装：**

```
参考网址：https://blog.csdn.net/Alexhcf/article/details/107438583

```

**Mac 安装:**

```
安装环境：
Mac OS 10.14
Python 3.8
java 12.0.2
MobSF v3.1 beta
安装步骤：
参考网址：http://www.51ste.com/share/det-5952.html

```

#### [](#（4）基本使用)（4）基本使用

![](https://bbs.pediy.com/upload/attach/202109/905443_QMA6CRB9HJ9RGTY.png)

 

我们直接拖入一个 APP 开始分析：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_SEFD987U5K6K7ET.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_PQU75PFQDCQNT9U.png)

 

我们可以看见 APP 正在进行逆向分析

 

![](https://bbs.pediy.com/upload/attach/202109/905443_FWD7ZGJZTBTP59A.png)

 

**静态分析：**

 

我们点击最近扫描就可以看见我们最近扫描的一些 APP 情况：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_846RYAJUAR5W9KN.png)

 

我们随便点击一个应用的静态报告，就可以看见详细的静态分析结果

 

![](https://bbs.pediy.com/upload/attach/202109/905443_V457S87RXQPXAP9.png)

 

**动态分析：**

 

我们需要安装 Genymotion 并注册账号，创建一个模拟器，这里我创建的是 7.0 模拟器

```
Genymotion官方地址：https://www.genymotion.com/download/

```

![](https://bbs.pediy.com/upload/attach/202109/905443_84JF444ECRPPPCU.png)

 

我们启动创建的模拟器

 

![](https://bbs.pediy.com/upload/attach/202109/905443_YC89MB6GQM7JECJ.png)

 

然后重新启动 MobSF, 在平台上点击 Dynamic Analyzer 发现新的内容, 点击 MobSFy Android Runtime 给模拟器安装插件

 

![](https://bbs.pediy.com/upload/attach/202109/905443_8MYB5UYEZKR32V4.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_95QC335KTUKB5Z2.png)

 

然后我们点击动态分析，开始进行动态分析

 

在这个过程中，我们可能会遇到各种错误，这里我们依次来解决：

```
问题1：Genymotion模拟器无法联网问题：
    我们需要去检测Genymotion模拟器的相关配置，具体解决方案参考：https://www.jianshu.com/p/ecb88d6bd815
问题2：废话不多说、电脑上的360管家等最好关闭，虽然这里不一定是这个导致，不过作为一名逆向人员，最好不要打开这些管家
问题3：安装Genymotion时，需要将VirusBox安装到默认路径下，不然会报错，安装后重启电脑

```

![](https://bbs.pediy.com/upload/attach/202109/905443_AZ7R5A42RMJWKZD.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_TAFNJ2DTPT2755W.png)

 

点击 start Instrumentation 开始自动化的遍历扫描

 

![](https://bbs.pediy.com/upload/attach/202109/905443_ARSHKDC9QHJXQD8.png)

 

我们还可以实时查看 API 监测情况：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_N2NWBEPW767MQS6.png)

 

然后我们产生动态分析报告

 

![](https://bbs.pediy.com/upload/attach/202109/905443_JBH28WJTK9GWGWW.png)

[](#四、总结)四、总结
-------------

本文针对 Android APP 漏洞做了一个初步的介绍，先调研了当下的一些主流漏洞，以及常见的 Android APP 端漏洞挖掘方式，还结合 APP 漏洞审计系统，详细的列出了当下 Android APP 端漏洞挖掘的初步思路，为漏洞挖掘安全人员提供一个参考。

 

本文还介绍了 Android APP 漏洞挖掘过程中的三种常用的工具 drozer+Inspeckage(Xposed)+MobSF，详细介绍了三种工具的安装和具体使用教程，能很好的帮助 Android 漏洞挖掘和渗透测试人员的工作。

 

本文的知识总结来自个人学习总结和网络上大量博客的收集，所有的博客都会列在参考列表中，可能还有一些总结不足，后续逐步完善以及欢迎各位大佬指正。

[](#五、参考网址)五、参考网址
-----------------

Android APP 漏洞：

```
《移动应用安全形势报告》（2020）：https://www.isc.org.cn/zxzx/xhdt/listinfo-40058.html
https://ayesawyer.github.io/2019/08/21/Android-App%E5%B8%B8%E8%A7%81%E5%AE%89%E5%85%A8%E6%BC%8F%E6%B4%9E/
https://xuanxuanblingbling.github.io/ctf/android/2018/02/12/Android_app_part1/

```

drozer：

```
https://labs.f-secure.com/tools/drozer/
http://rui0.cn/archives/30
http://www.feidao.site/wordpress/?p=3438

```

Inspeckage:

```
https://www.e-learn.cn/topic/3470422
https://blog.csdn.net/tom__chen/article/details/78216732
https://repo.xposed.info/module/mobi.acpm.inspeckage

```

MobSF:

```
https://www.codeleading.com/article/13073244765/
https://blog.csdn.net/Alexhcf/article/details/107438583
https://github.com/MobSF/Mobile-Security-Framework-MobSF
http://www.51ste.com/share/det-5952-3.html
https://bbs.pediy.com/thread-218973.htm

```

[【公告】“雪花” 创作激励计划，3 月 1 日正式开启！](https://bbs.pediy.com/thread-271637.htm)

最后于 2021-9-21 17:01 被随风而行 aa 编辑 ，原因： drozer 指令修改

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#漏洞相关](forum-161-1-123.htm) [#HOOK 注入](forum-161-1-125.htm) [#工具脚本](forum-161-1-128.htm)