> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271122.htm)

> [原创]Android APP 漏洞之战（7）——信息泄露漏洞详解

Android APP 漏洞之战（7）—— 信息泄露漏洞详解
==============================

目录

*   Android APP 漏洞之战（7）—— 信息泄露漏洞详解
*            [一、前言](#一、前言)
*            [二、基础知识](#二、基础知识)
*                    [1. 文件存储](#1.文件存储)
*                            [（1）内部存储](#（1）内部存储)
*                            [（2）外部存储](#（2）外部存储)
*                                    <1> 私有目录
*                                    <2> 公共目录
*                                    [（3）系统存储目录](#（3）系统存储目录)
*                            [（4）文件存储的读写方式](#（4）文件存储的读写方式)
*                    [2.SharedPreferences](#2.sharedpreferences)
*                            [（1）SharedPreferences 数据的存取](#（1）sharedpreferences数据的存取)
*                    3. SQLite 数据库存储
*                            [（1）数据库的创建](#（1）数据库的创建)
*                            [（2）数据库的更新](#（2）数据库的更新)
*                            [（3）数据库的基本操作](#（3）数据库的基本操作)
*                    4. ContentProvider
*                    [5. 网络存储](#5.网络存储)
*            [三、信息泄露漏洞的安全场景和分类](#三、信息泄露漏洞的安全场景和分类)
*                    [1. 漏洞的安全场景](#1.漏洞的安全场景)
*                            [（1）产品敏感信息](#（1）产品敏感信息)
*                            [（2）用户敏感信息](#（2）用户敏感信息)
*                    [2. 漏洞的分类](#2.漏洞的分类)
*            [四、信息泄露漏洞的原理分析和复现](#四、信息泄露漏洞的原理分析和复现)
*                    1. LogCat 输出敏感信息漏洞
*                            [（1）原理分析](#（1）原理分析)
*                            [（2）漏洞案例——DIVA.apk 例题 1](#（2）漏洞案例——diva.apk例题1)
*                            [（3）安全防护](#（3）安全防护)
*                    [2. 硬编码问题漏洞](#2.硬编码问题漏洞)
*                            [（1）原理分析](#（1）原理分析)
*                            [（2）漏洞案例——DIVA.apk 例题 2（java 层）](#（2）漏洞案例——diva.apk例题2（java层）)
*                            [（3）漏洞案例——DIVA.apk 例题 12（so 层）](#（3）漏洞案例——diva.apk例题12（so层）)
*                            [（4）安全防护](#（4）安全防护)
*                    3. Shared Preference 全局可读写漏洞
*                            [（1）原理分析](#（1）原理分析)
*                            [（2）漏洞案例——DIVA.apk 例题 3](#（2）漏洞案例——diva.apk例题3)
*                            [（3）安全防护](#（3）安全防护)
*                    [4. 数据库存储漏洞](#4.数据库存储漏洞)
*                            [（1）原理分析](#（1）原理分析)
*                            [（2）漏洞案例——DIVA.apk 例题 4](#（2）漏洞案例——diva.apk例题4)
*                            [（3）安全防护](#（3）安全防护)
*                    [5. 临时文件或 SD 卡漏洞](#5.临时文件或sd卡漏洞)
*                            [（1）原理分析](#（1）原理分析)
*                            [（2）漏洞案例——DIVA.apk 例题 5 与例题 6](#（2）漏洞案例——diva.apk例题5与例题6)
*                            [（3）安全防护](#（3）安全防护)
*                    6. http 明文传输漏洞
*                            [（1）漏洞原理](#（1）漏洞原理)
*                            [（2）漏洞案例——XX 音乐存在任意文件下载漏洞](#（2）漏洞案例——xx音乐存在任意文件下载漏洞)
*                            [（3）安全防护](#（3）安全防护)
*            [五、实验总结](#五、实验总结)
*            [六、参考文献](#六、参考文献)

[](#一、前言)一、前言
-------------

好久没有更新帖子了，最近一直都比较忙，这里首先祝大家腊八节快乐。本文主要围绕 Android APP 漏洞中的信息泄露漏洞展开描述，因为挖掘 Android APP 信息泄露漏洞的思路各有差异，所以本文只是基于 Android APP 中较为基础的信息泄露的漏洞实例开始讲述。

 

本文第二节主要讲述 Android 中存储的基本方式

 

本文第三节主要讲述信息泄露漏洞的分类

 

本文第四节主要讲述漏洞的原因和具体复现

[](#二、基础知识)二、基础知识
-----------------

APP 信息泄露漏洞往往和 Android APP 的数据存储方式有关，所以我们这里首先详细的了解 Android 的数据存储方式。我们知道 Android 中数据存储的方式共有五种，分别为：文件存储、SharedPreferences、SQLite 数据库存储、ContentProvider、网络存储。

### 1. 文件存储

![](https://bbs.pediy.com/upload/attach/202201/905443_N7VN9HEHA24NXPF.png)

 

下面我们将结合上面的思维导图依次讲解

#### [](#（1）内部存储)（1）内部存储

内部存储一般存储一些应用的数据，如 apk、shareprefence、database 数据、webview 缓存和图片缓存等等，内部存储一般存储在`/data/`下面，这些都需要用户获得 root 权限后才能访问到

 

![](https://bbs.pediy.com/upload/attach/202201/905443_ZKETVRHPN677TGG.png)

 

我们以 root 权限的模式进入：

 

![](https://bbs.pediy.com/upload/attach/202201/905443_JHD9USNK9E36FEW.png)

 

下面我们介绍常见的一些内部存储目录：

```
/data/app/    存储着我们手机上安装的apk文件
/data/data/包名/share_prefs  存储对应的应用程序中的shareprefence存储文件
/data/data/包名/cache    存储对应的应用程序中的cache缓存文件
/data/data/包名/databases  存储对应的应用程序中的数据库文件
/data/data/包名/files      存储对应的应用程序中的资源文件

```

内部存储的特点：

```
内部存储的文件和目录只能被我们的app自己所访问，别的app不能访问。
内部存储中的私有目录，当用户卸载app之后，改文件目录中关于该应用的信息就会被删除。
内部存储是可用的。
内部存储大小有限，不适合存储大量数据。
只有root的手机，才能从手机文件管理器看见，否则都是隐藏着的。

```

#### [](#（2）外部存储)（2）外部存储

Android4.4 以前，手机自身的存储就叫内部存储，插入 SD 卡的存储叫外部存储，然而 Android 4.4 以后，手机自带的存储很大，因此现在的外部存储分为两部分：SD 卡和扩展卡内存。外部存储一般分为两类，私有目录和公有目录，私有目录里面的数据会随着应用的卸载而删除，公有目录并不会

 

自身的外部存储目录：`/storage/emulated/0/Android/data/packagename/files`

 

存储卡的存储目录：`/storage/extSdCard/Android/data/packagename/files`

 

![](https://bbs.pediy.com/upload/attach/202201/905443_868NJKEA68EMZ78.png)

 

外部存储的特点：

```
公有目录任何程序都可以访问，私有目录自身可以访问。
并不一定是可用的，因为SD卡会被挂载。
外部存储中的私有目录中的数据会随着应用的卸载而删除，公有目录则不会。

```

##### <1> 私有目录

私有目录，在 Android 4.4 以上，不需要注册和用户授权 SD 读写的权限，就可以在应用的私有目录进行读写文件，文件不能被其他应用访问，用户删除应用时，对应的应用的私有目录也会被删除

 

私有目录地址：`/storage/emulated/0/Android/data/packagename`

 

相关 API:

 

私有目录访问的 API 都在`ContextWrapper`对象上，可以直接通过 Activity 或 Context 进行调用

```
getExternalCacheDir()： 访问/storage/emulated/0/Android/data/应用包名/cache目录，该目录用来存放应用的缓存文件，当我们通过应用删除缓存文件的时候，该目录下的文件会被清除
getExternalFilesDir(): 访问/storage/emulated/0/Android/data/应用包名/files 目录,该目录用来存放应用的数据

```

##### <2> 公共目录

公共目录必须需要用户授权读写的权限，就意味需要在`AndroidManifest.xml`中注册用户权限

Android 6.0 系统之后需要申请用户权限，并获得用户授权，才能读写文件，公共目录相对开放，我们可以访问其他 APP 存在公共目录下的文件，并且当 APP 被删除时，并不会删除应用存在公共目录下的文件

 

相关 API:

 

公共目录可以通过 Environment 对象，访问读写公共目录的文件

```
Environment.getExternalStorageDirectory() 访问外部存储设备公共根目录
Environment.getExternalStorageState() 获得外部存储SD卡的状态

```

##### [](#（3）系统存储目录)（3）系统存储目录

```
getRootDirectory()：对应获取系统分区根路径:/system
getDataDirectory()：对应获取用户数据目录路径:/data
getDownloadCacheDirectory()：对应获取用户缓存目录路径:/cache

```

补充：

```
（1）升级应用程序后的apk文件在哪？
一般我们从服务器端下载的app需要放到外部存储目录下面，而不是内部存储目录，即/storage/emulated/0/Android/data/packagename下
（2）清除数据和清除缓存的区别？
清除数据清除的是保存在app中所有数据，就是上面提到的位于packagename下面的所有文件，包含内部存储(/data/data/packagename/)和外部存储(/storage/emulated/0/Android/data/packagename/)，但不会影响SD卡的数据
缓存是程序运行时的临时存储空间，缓存文件存放在getCacheDir()或者 getExternalCacheDir()路径下

```

#### [](#（4）文件存储的读写方式)（4）文件存储的读写方式

写入文件：

```
public void save(){
      String data = "Data to save";
      FileOutputStream out = null;
      ButteredWriter writer = null;
      try{
            out = openFileOutput("data",Context.MODE_PRIVATE);  //MODE_PRIVATE（默认）：覆盖、MODE_APPEND：追加
            writer = new ButteredWriter(new OutputSreamWriter(out));
            writer.write(data);
      }catch(IOException e){
            e.printStackTrace();
      }finally{
            try{
                  if(writer!=null){
                        writer.close();
                  }
            }catch(IOException e){
                  e.printStackTrace();
       }
}

```

读取文件：

```
public String load(){
      FileInputStream in = null;
      ButteredReader reader = null;
      StringBuilder builder = new StringBuilder();
      try{
            in = openFileInput("data");
            reader = new ButteredReader(new InputStreamReader(in));
            String line= "";
            while((line = reader.readline()) != null){
                   builder.append();
            }
      }catch(IOException e){
            e.printStackTrace();
      }finally{
            if(reader != null){
                    try{
                          reader.close();
                    }catch(IOException e){
                          e.printStackTrace();
                    }
             }
      }
}

```

### 2.SharedPreferences

SharedPreference 是 Android 平台上一个轻量级的存储类，主要是保存一些常用的配置比如窗口状态，是使用键值对的方式来存储数据，这样就可以支持多种不同的数据类型存储，进行数据持久化就会比文件方便很多

 

默认存储路径：`/data/data/packageName/shared_prefs`

 

获取 SharedPreferences 对象的方法

```
Context的getSharedPreferences()方法，参数一是文件名，参数二是操作模式
Activity的getPreferences()方法，参数为操作模式，使用当前应用程序包名为文件名
PreferenceManager的getDefaultSharedPreferences()静态方法，接收Context参数，使用当前应用程序包名为文件名

```

#### [](#（1）sharedpreferences数据的存取)（1）SharedPreferences 数据的存取

SharedPreference 的存储：

```
(1)根据Context获取SharedPreferences对象
(2)利用edit()方法获取Editor对象
(3)ditor对象存储key-value键值对数据
(4)apply()提交数据

```

```
public class MainActivity extends Activity {    
 @Override
     public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
 
        //获取SharedPreferences对象
        Context ctx = MainActivity.this;       
        SharedPreferences sp = ctx.getSharedPreferences("SP", MODE_PRIVATE); //MODE_PRIVATE（默认）：只有当前的应用程序才能对文件进行读写、MODE_MULTI_PROCESS：用于多个进程对同一个SharedPreferences进行读写
        //存入数据
        Editor editor = sp.edit();
        editor.putString("name", "Tom");
        editor.putInt("age", 28);
        editor.putBoolean("married", true);
        editor.apply();
 
        //返回STRING_KEY的值
        Log.d("SP", sp.getString("name", "none"));
        //如果NOT_EXIST不存在，则返回值为"none"
        Log.d("SP", sp.getString("NOT_EXIST", "none"));
     }
 
}

```

SharedPreference 数据的读取：

```
SharedPreferences pref = getSharedPreferences("data",MODE_PRIVATE);
String name = pref.getString("name");
int age = pref.getInt("age");
boolean isMarried = pref.getBoolean("isMarried");

```

### 3. SQLite 数据库存储

SharedPreferences 对象与 SQLite 数据库相比，免去了创建数据库、创建表、写 SQL 语句等操作，但是其只能存储 boolean，int，float，long 和 String 五种简单的数据类型，而且 SharedPreferences 是以明文的形式存储密钥信息，往往存在一定的安全隐患。为此 Android 还提供了一个轻量级的数据库 SQLite 数据库，SQLite 是轻量级嵌入式数据库引擎，它支持 SQL 语言，并且只利用很少的内存就有很好的性能

 

默认存储路径：`/data/data/packagename/databases`

#### [](#（1）数据库的创建)（1）数据库的创建

Android 提供了 SQLiteOpenHelper 类帮助创建和升级数据库，SQLiteOpenHelper 子类至少需要实现三个方法：

```
1.构造函数，调用父类SQLiteOpenHelper的构造函数。需要四个参数（上下文环境、数据库名称、查询数据的游标Cursor(通常为null)、当前数据库的版本号）
2.onCreate（）方法，它需要一个 SQLiteDatabase 对象作为参数，根据需要对这个对象填充表和初始化数据
3.onUpgrage() 方法，它需要三个参数，一个 SQLiteDatabase 对象，一个旧的版本号和一个新的版本号，这样就可以方便的实现数据库的升级

```

继承 SQLiteOpenHelper 创建数据库

```
public class MyDatabaseHelper extends SQLiteOpenHelper {
    //1.构造方法
  MyDatabaseHelper(Context context, String name, CursorFactory cursorFactory, int version) 
  {     
    super(context, name, cursorFactory, version);     
     }     
 
     @Override    
     public void onCreate(SQLiteDatabase db) {     
         // TODO 创建数据库后，对数据库的操作     
     }     
 
     @Override    
 public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {     
         // TODO 升级数据库版本
     }     
 
 @Override    
 public void onOpen(SQLiteDatabase db) {     
         super.onOpen(db);       
         // TODO 每次成功打开数据库后首先被执行     
     }     
 }

```

#### [](#（2）数据库的更新)（2）数据库的更新

```
public class MyDatabaseHelper extends SQLiteOpenHelper{ 
    ......
    //当打开数据库时传入的版本号与当前的版本号不同时会调用该方法 
    @Override 
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {  
          db.execSQL("drop table if exists Book");
          onCreate(db):
    }   
}

```

只要我们在 MainActivity 中将 version 改为大于原来版本号，就可以让`onUpgrade()`方法得到执行

```
MyDatabaseHelper helper = new MyDatabaseHelper(this,"BookStore.db",null,2);
helper.getWritableDatabase();

```

#### [](#（3）数据库的基本操作)（3）数据库的基本操作

添加数据：

```
SQLiteDatabase db = helper.getWritableDatabase();
ContentValues values = new ContentValues();
values.put("name","The Book Name");
values.put("author","chen");
values.put("pages",100);
values.put("price",200);
db.insert("Book",null,values);//参数一 表名 参数二 未指定添加数据的情况下为NULL 参数三 ContentValues对象

```

更新数据：

```
SQLiteDatabase db = helper.getWritableDatabase();
ContentValues values = new ContentValues();
values.put("price",120);
db.update("Book",values,"name= ?",new String[]{"The Book Name"});  //参数一 表名 参数二 ContentValues对象 参数三、四是去约束更新某一行或某几行的数据，不指定默认更新所有

```

删除数据：

```
SQLiteDatabase db = helper.getWritableDatabase();
db.delete("Book","pages> ?",new String[]{"100"}); //参数一是表名，参数二、三是去约束删除某一行或某几行的数据，不指定默认删除所有

```

查询数据：

```
SQLiteDatabase db = helper.getWritableDatabase();
//query()方法，参数一是表名，参数二是指定查询哪几列，默认全部，参数三、四是去约束查询某一行或某几行的数据，不指定默认查询所有，参数五是用于指定需要去group by的列，参数六是对group by的数据进一步的过滤，参数七是查询结果的排序方式
Cursor cursor = db.query("Book",null,null,null,null,null,null);
if(cursor.moveToFirst()){
      do{
            String name = cursor.getString(cursor.getColumnIndex("name");
            String author = cursor.getString(cursor.getColumnIndex("author");
            int pages = cursor.getString(cursor.getColumnIndex("pages");
            double price = cursor.getString(cursor.getColumnIndex("price");
       }while(cursor.moveToNext());
}
cursor.close();

```

SQL 语句操作数据库：

```
//添加数据
db.execSQL("insert into Book(name,author,pages,price) values(?,?,?,?) "
            ,new String[]{"The Book Name","chen",100,20});
//更新数据
db.execSQL("update Book set price = ? where name = ?",new String[]
            {"10","The Book Name"});
//删除数据
db.execSQL("delete from Book where pages > ?",new String[]{"100"});
//查询数据
db.execSQL("select * from Book",null);

```

使用事务操作：

```
SQLiteDatabase db = helper.getWritableDatabase();
db.beginTransaction();  //开启事务
try{
      ......
      db.insert("Book",null,values);
      db.setTransactionSuccessful();  //事务成功执行
}catch(SQLException e){
      e.printStackTrace();
}finally{
      db.endTransaction();  //结束事务
}

```

当然 Android 数据库中还包括`LitePal`数据库更加方便的操作，由于本文主要是介绍漏洞，所以这里就简单介绍到这里

### 4. ContentProvider

前面三种方式是 Android 中基本的存储方式，但是由于都存在一个公共的缺点：不能实现不同应用程序之间进行数据共享，大家都知道 Android 是采用沙箱的管理机制，不同的应用程序之间都是独立隔离开的，这一定程度上也是为了 Android 应用之间的安全性考虑，但是如果应用之间不能很好的进行交互，那么很显然就带来了明显的不便，因此为了解决这个问题，内容提供器——ContentProvider 就孕育而生了，由于前面我们有专门的章节讲述这一组件，所以本文只是简单描述，不详细了解，请参考 [Android APP 漏洞之战（4）——Content Provider 漏洞详解](https://bbs.pediy.com/thread-269447.htm)

 

主要作用：用于不同的程序之间实现数据共享的功能，并通过 ContentResolver 进行操作

 

ContentResolver 使用方法：

 

（1）内容 URI

```
//包名为com.example.app的表table1访问路径
Uri uri  = Uri.parse("content://com.example.app.provider/table1");

```

（2）使用 URI 对象进行数据操作

 

**查询**

```
Cursor cursor = getContentResolver().query(uri,null,null,null,null);
if(cursor != null){
      while(cursor.moveToNext()){
            String column1 = cursor.getString(cursor.getColumnIndex("column1"));
            String column2 = cursor.getString(cursor.getColumnIndex("column2"));
      }
      cursor.close();
}

```

**插入**

```
ContentValues values = new ContentValues();
values.put("column1","text");
values.put("column2",1);
getContentResolver().insert(uri,values);

```

### 5. 网络存储

Android 网络存储主要是通过网络来实现数据的存储和获取，这里主要调用 WebService 返回的数据或是解析 HTTP 协议实现网络数据交互，具体需要熟悉 java.net._，Android.net._ 这两个包的内容，因为不是 Android 的主流存储方式，这里主要参考相应文档即可，也不是本文漏洞所关注的主要对象，就不做过多叙述了

 

网络存储需要开启权限：

具体可参考本文参考文献

[](#三、信息泄露漏洞的安全场景和分类)三、信息泄露漏洞的安全场景和分类
-------------------------------------

### 1. 漏洞的安全场景

信息泄露漏洞往往是指 APP 开发过程中一些不安全的开发问题导致敏感信息的泄露，那我们首先可以将敏感信息进行分类：产品敏感信息和用户敏感信息

#### [](#（1）产品敏感信息)（1）产品敏感信息

产品敏感信息：登录密码、后台登录及数据库地址、服务器部署的绝对路径、内部 ip、地址分配规则、网络拓扑、页面注释信息等

#### [](#（2）用户敏感信息)（2）用户敏感信息

用户的个人隐私信息泄露导致被恶意人员利用获取不当信息，例如用户的密码，账户信息等等

 

这些敏感信息泄露往往是由于信息未加密或存储位置不当造成：

```
代码中明文使用敏感信息，比如：服务器地址、数据库信息等
数据库中明文保存敏感信息，比如：账号、密码、银行卡等
SD卡中保存敏感信息或隐私信息,比如：聊天记录、通讯录等
日志打印敏感信息,比如：账号、密码
明文传输敏感信息

```

### 2. 漏洞的分类

综上我们将 Android 中的信息泄露漏洞大致可分为：

 

![](https://bbs.pediy.com/upload/attach/202201/905443_EXW48JWXPFUHJHU.png)

[](#四、信息泄露漏洞的原理分析和复现)四、信息泄露漏洞的原理分析和复现
-------------------------------------

本文为了简单演示各个漏洞的复现情况，将会使用样本 DIVA.apk 和一些实际漏洞的场景，样本将放到附件中

### 1. LogCat 输出敏感信息漏洞

#### [](#（1）原理分析)（1）原理分析

在 APP 的开发过程中，为了方便调试，开发者通常会用 logcat 输出 info、debug、error 等信息。如果在 APP 发布时没有去掉 logcat 信息，可能会导致攻击者通过查看 logcat 日志获得敏感信息

 

一般来说，LogCat 敏感信息输出漏洞包括：

```
应用层Log敏感信息输出
应用层System.out.println敏感信息输出
系统bug异常导致Log输出
Native层敏感Log输出

```

#### （2）漏洞案例——DIVA.apk 例题 1

首先，我们安装样本 DIVA.apk

 

![](https://bbs.pediy.com/upload/attach/202201/905443_PHCKNF27Y9396P7.png)

 

然后我们打开例题 1，并开启日志监控

```
adb logcat |grep diva

```

![](https://bbs.pediy.com/upload/attach/202201/905443_RB9RTMPSZQBF4KR.png)

 

我们在 app 表单总输入内容，check out 后查看相关日志

 

![](https://bbs.pediy.com/upload/attach/202201/905443_BDAXJENHMNJDT7T.png)

 

我们就可以发现我们输入的密钥信息，然后我们根据日志信息，可以找到漏洞代码在 LogActivity.class 文件，我们查看相关代码：

 

![](https://bbs.pediy.com/upload/attach/202201/905443_QEBF7JMQWRH4ZDS.png)

 

这里相关代码就是将敏感信息给泄露，当然我们真实的 app 中不会这么简单，但这确实一个很好的思路，比如我们对我们分析的 app 中的敏感信息的函数进行 hook 或者通过插桩日志的信息，我们就可以成功获得敏感信息了

#### [](#（3）安全防护)（3）安全防护

防护建议：

```
1.Android Studio中配置ProGuard实现release版apk自动删除Log.d()/v()等代码
2.使用自定义LogCat类，上线前关闭LogCat开关

```

```
public class LG{
    //是否开启debug
    public static boolean isDebug = true;
 
    public static void e(Class clazz,String msg){
        if (isDebug){
            //Log.e
        }
    }
 
     public static void i(Class clazz,String msg){
        if (isDebug){
            //Log.i
        }
    }
 
     public static void w(Class clazz,String msg){
        if (isDebug){
            //Log.w
        }
    }
 
     public static void d(Class clazz,String msg){
        if (isDebug){
            //Log.d
        }
    }
}

```

### 2. 硬编码问题漏洞

#### [](#（1）原理分析)（1）原理分析

一些开发人员，在开发时使用硬编码的方式，导致存在一定的安全风险，硬编码一般是指将输出或输入的相关参数以常量的方式编写在源代码中，这样导致逆向分析人员可以直接通过分析源码就可以获得敏感信息

#### （2）漏洞案例——DIVA.apk 例题 2（java 层）

我们打开样本 app 的例题 2

 

![](https://bbs.pediy.com/upload/attach/202201/905443_PAPDKYR5EVCP4HS.png)

 

我们分析对应 app 相关的逆向代码：

 

![](https://bbs.pediy.com/upload/attach/202201/905443_4DYRUMP48G9ATTF.png)

 

我们可以发现相应的敏感信息被直接用来判断，采用硬编码的方式，我们直接将密钥输入，发现可以成功破译

 

![](https://bbs.pediy.com/upload/attach/202201/905443_9PJT94WU4NHUGYF.png)

#### （3）漏洞案例——DIVA.apk 例题 12（so 层）

我们打开例题 12

 

![](https://bbs.pediy.com/upload/attach/202201/905443_48Q7M99YAT86YY8.png)

 

我们分析对应的代码段

 

![](https://bbs.pediy.com/upload/attach/202201/905443_EVYFDWH74XXF5MY.png)

 

说明 key 被保存在 libsoName.so 库中，我们将文件打开

 

![](https://bbs.pediy.com/upload/attach/202201/905443_HSHRH78ZDPNT2BD.png)

 

![](https://bbs.pediy.com/upload/attach/202201/905443_HTH9MQ8TJSPDQSE.png)

 

经过分析，我们确定这就是我们的键值，我们输入

 

![](https://bbs.pediy.com/upload/attach/202201/905443_GMFTSKW9JT3E7QU.png)

 

说明 key 放在 so 层中也是不安全的

 

密钥硬编码案例：

 

下面为乌云平台一些 APP 漏洞案例，详细可做参考：

 

[密钥硬编码](436b34f44b9f95fd3aa8667f1ad451b16a683c5957c8d733b3809de3444b7c6fffae9f78c9c65e6f292dd695bd96daa0c3f153ebdaeb4f1b582e5c13e28f97c3601018d9e8a589033217ff27785e473dbcc50607f5530dd0a63dd6f512327a46effe31c0ed8b01752287563eebc03bf2495aa785bf000246363bd83dc65459799e5b2af9e74fcc8cb84b8c8b55ba087b6c184ae39c9f8a03cef3622956a4767be83bc3c2f93d8931185a3a9c47e527aed5b7fe1c570c1cc866fc201b62a8c5ba0ca75d7a1d0fd6dd53c3b3d342a74ce414334ae0d1bb980becf489b072cfaaaf00f6d3ac37f973c7970274827c26ecbb177cc4fa5d38d9ece06ab2b5a4bf7147)

#### [](#（4）安全防护)（4）安全防护

```
1.对明文传输的密钥进行加密传输
2.采用变量的方式去读取，不采用硬解码的方式
3.对java层中进行混淆，对so层中进行ollvm控制流混淆

```

### 3. Shared Preference 全局可读写漏洞

#### [](#（1）原理分析)（1）原理分析

Shared Preferences 存储安全风险在于：开发者在创建文件时没有正确的选择合适的创建模式（MODE_PRIVATE、MODE_WOELD_READABLE 以及 MODE_WORLD_WRITEABBLE）进行权限控制，导致将一些用户信息、密码等敏感信息存储在 Shared Preferences 文件中，攻击者可以通过 root 来查看敏感信息

#### （2）漏洞案例——DIVA.apk 例题 3

我们进入例题 3：

 

![](https://bbs.pediy.com/upload/attach/202201/905443_H783ZSTR8RU7XHB.png)

 

我们分析对应的逆向代码：

 

![](https://bbs.pediy.com/upload/attach/202201/905443_YZ2KWVHSRUEQF9V.png)

 

经过我们前文的分析，很明显这里采用的是 SharedPreference 的存储方式

 

我们输入相关的账号和密码，然后我们可以进入 shared_prefs 查看相关的文件

 

![](https://bbs.pediy.com/upload/attach/202201/905443_E5PWZ5YUPUJSJDW.png)

#### [](#（3）安全防护)（3）安全防护

防护意见：

```
1.避免使用MODEWORLDWRITEABLE和MODEWORLDREADABLE模式创建进程间通信的文件，此处即为Shared Preferences
2.不要将密码等敏感信息存储在Shared Preferences等内部存储中
3.避免滥用"Android:sharedUserId"属性
4.不要在使用“android:sharedUserId”属性的同时，对应用使用测试签名，否则其他应用拥有“android:sharedUserId"属性值和测试签名是，将会访问到内部存储文件数据
5.使用Secure Preferences第三方加固库进行存储

```

### 4. 数据库存储漏洞

#### [](#（1）原理分析)（1）原理分析

Database 配置模式安全风险源于：

 

开发者在创建数据库（Database）时没有正确的选取合适的创建模式（MODE_PRIVATE、MODE_WORLD_READABLE）进行权限控制，从而导致数据库（Database）内容被恶意读写，造成账户密码、身份信息、以及其他敏感信息泄露，甚至攻击者进一步实施恶意攻击

#### （2）漏洞案例——DIVA.apk 例题 4

我们进入例题 4

 

![](https://bbs.pediy.com/upload/attach/202201/905443_X6D2XECHY8HQG5A.png)

 

然后我们输入相关信息并保存

 

![](https://bbs.pediy.com/upload/attach/202201/905443_NSJ3ZT6DJWE8WKN.png)

 

我们将数据库给拉取下来，然后使用 SQLite 查看

 

![](https://bbs.pediy.com/upload/attach/202201/905443_WWTWB94CH77HXX9.png)

 

我们就可以发现敏感信息，我们可以分析对应的代码段

 

![](https://bbs.pediy.com/upload/attach/202201/905443_EZ8NF5QDWFM44D2.png)

 

我们上文的存取数据库名也是从对应代码出找到

#### [](#（3）安全防护)（3）安全防护

防护建议：

```
1.敏感信息在进行数据库存储时，对数据进行加密存储，并且应该避免弱加密或者是不安全的加密方式
2.对于敏感的数据库文件，不得使用MODE_WORLD_READABLE或者是MODE_WORLD_WRITEABLE进行创建

```

### 5. 临时文件或 SD 卡漏洞

#### [](#（1）原理分析)（1）原理分析

经过上文我们讲述的文件存储，现在的手机很多的公共目录都是自身自带的存储空间，Android 系统的文件一般都存储在 sdCard 和应用的私有目录下，任何在 Android Manifest 中声明读写 sdcard 权限的应用都可以对 sdcard 进行读写。

#### （2）漏洞案例——DIVA.apk 例题 5 与例题 6

我们打开例题 5

 

![](https://bbs.pediy.com/upload/attach/202201/905443_ZUX6TBZWQ7XNP3X.png)

 

我们分析对应部分的相关代码：

 

![](https://bbs.pediy.com/upload/attach/202201/905443_QJ3R2WXN8PJJDEY.png)

 

根据代码，我们知道这里采用文件存储的方式，所以我们直接找到该文件，查看即可

 

![](https://bbs.pediy.com/upload/attach/202201/905443_NPXZZTQZKQZ3CSA.png)

 

同理，我们打开例题 6，查看对应的代码

 

![](https://bbs.pediy.com/upload/attach/202201/905443_CCHT6CEP7G594PJ.png)

 

我们可以发现是存储在 sd 卡下，然后我们输入相关信息，之后直接去 sdcard 下查找，这里我们需要注意，需要给应用存取权限，否则是保存不了的

 

![](https://bbs.pediy.com/upload/attach/202201/905443_XZ5AXWMDA4CMNZB.png)

#### [](#（3）安全防护)（3）安全防护

防护意见：

```
1.不要将敏感信息存入本地固定的文件中，哪怕是加密存储也可能面临暴力破解的风险
2.对于保存信息的代码段进行混淆加密，使其难以被逆向人员简单分析获取

```

### 6. http 明文传输漏洞

#### [](#（1）漏洞原理)（1）漏洞原理

开发人员在开发时对网络连接的一些敏感数据往往采用 http 明文传输，这样就十分容易导致恶意攻击者通过一些抓包工具进行捕获，获取敏感信息，导致信息泄露的风险

#### [](#（2）漏洞案例——xx音乐存在任意文件下载漏洞)（2）漏洞案例——XX 音乐存在任意文件下载漏洞

这段时间我查看了一下国内主流的音乐软件平台，发现很多音乐软件都存在 http 明文传输的问题，这样势必会导致升级劫持、信息泄露、任意文件下载等等漏洞，这里我们列举一个有 http 明文泄露导致的任意文件下载问题

 

首先我们随便选择几首需要 VIP 下载的歌曲

 

等他

 

![](https://bbs.pediy.com/upload/attach/202201/905443_SXGVZ9PUPYU4BN9.png)

 

我们发现这些音乐都需要开通 VIP 才能下载

 

接下来我们用抓包软件抓取

 

![](https://bbs.pediy.com/upload/attach/202201/905443_PYDXWRECSYJKWMN.png)

 

我们发现了音乐的 url，我们直接访问，发现可以直接下载

 

还有一些 http 明文传输漏洞导致任意登录，一些 APP 也会把登录成功的 Cookie 保存在本地，那么只要找到相关文件复制下来这个 Cookie，就可以任意登录了。

#### [](#（3）安全防护)（3）安全防护

防护建议：

```
1.采用https加密传输
2.敏感信息使用http传输，那么对敏感信息进行加密，并且使用非对称加密，或者公认的强加密算法
3.对以下字段进行加密处理：密码、手机号、快捷支付手机号、Email、身份证、银行卡、CVV码、有效期等

```

[](#五、实验总结)五、实验总结
-----------------

本文对 Android App 中信息泄露漏洞的常见形式做了一个基本的讲述，当然实际过程中很多 APP 的信息泄露漏洞可能要比这个复杂，这里只是初步的为大家介绍一下信息泄露漏洞的基础知识，本文的样例我会上传到 github 上，这里放一个传送门：

 

[github 地址](https://github.com/guoxuaa/Android-Vulnerability-Mining)

[](#六、参考文献)六、参考文献
-----------------

文件存储：

```
https://juejin.cn/post/6844903778227847182#heading-10
https://juejin.cn/post/6844904013515718664#heading-6

```

其他存储：

```
https://www.jianshu.com/p/536ca489a7f4
https://cloud.tencent.com/developer/article/1045171

```

网络存储：

```
https://www.cnblogs.com/doodle777/p/4937594.html
https://blog.csdn.net/weixin_43689040/article/details/103761411

```

漏洞挖掘参考：

```
https://www.anquanke.com/post/id/84603
https://www.anquanke.com/post/id/86057

```

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#漏洞相关](forum-161-1-123.htm) [#程序开发](forum-161-1-124.htm)

上传的附件：

*   [diva-beta.apk](javascript:void(0)) （1.43MB，4 次下载）