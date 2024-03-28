> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281122.htm)

> [原创]【Andorid 取证】微信聊天记录数据库和索引库逆向思路和一点思考

看到 @行简 大佬采用 Hook 的方法对微信数据库解密，深受启发。前段时间正好在做微信相关的项目，这里分享一下我自己逆向的思路，如有谬误，还请大佬批评指正  
注：本文所提供方法仅供逆向学习参考，请勿用于违法犯罪，所产生的的一切后果由用户自行承担责任

前期调研
====

随着微信多次更新，微信数据加密的方式多次更改，应该是从微信的某个大版本开始引入聊天记录数据库 SQLCipher 加密，密钥为 md5（IMEI+UIN）.substring(0,7)，到了微信 8.0 版本，更改了加密方式，猜测是因为 Android10 原有的 API 无法再提供手机的 IMEI 码，便更改了加密方式。聊天记录索引库在微信早期版本命名为 “IndexMicroMsg.db”；6.6 版本后更新命名为 ““FTS5IndexMicroMsg.db”，新增全文检索功能；微信 7.0 版本将命名改为 “FTS5IndexMicroMsg_encrypt.db”，将索引文件也进行了加密，二者密钥并不相同

工具和环境
=====

微信 8.0.19  
Jeb  
DB Browser

逆向思路
====

因为是纯分析代码，可能会有些繁琐，估将一些步骤省略，只贴一些关键函数

文件搜集
----

用户 UIN 位置：/data/data/com.tencent.mm/shared_prefs/auth_info_key_prefs.xml  
![](https://bbs.kanxue.com/upload/attach/202403/927366_43GNNFZ9MUCK552.webp)  
聊天记录数据库文件：'/data/data/com.tencent.mm/MicroMsg/[用户 IDmd5]'  
索引文件：'/data/data/com.tencent.mm/MicroMsg/[用户 IDmd5]/FTS5IndexMicroMsg_encrypt.db'

逆向思路
----

### 数据库文件解密

因为与数据库相关，所以可以直接定位包名为 com.tencent.mm 和 com.tencent.wcdb  
明文搜索 “EnMicroMsg.db” 定位到可疑函数（因为也有个数据库初始化的过程，其中也会有这个明文，注意分析）  
![](https://bbs.kanxue.com/upload/attach/202403/927366_9EMUFZHB5BE9HY6.webp)  
这里还可以看到数据库备份文件相关代码，我们先不管着重分析一下下面的加密代码  
明文变量 key 直接给出，getmessageDeigest 跟踪进去是 md5  
![](https://bbs.kanxue.com/upload/attach/202403/927366_7HVWPPCMGYGMEN2.webp)  
下面我们需要知道的就是其中的两个变量 q.eD(true) 和 arg12  
先看 q.eD(true)  
![](https://bbs.kanxue.com/upload/attach/202403/927366_737MBTCKJ4KUTWK.webp)  
其实从传入参数看就已经知道该函数返回值为 "1234567890ABCDEF"  
后面看了一下上面的函数，发现一步一步往里面跟踪可以找到获取用户 IMEI 的函数，应该是被弃用了。  
再看 arg12，改了一下名为 j2，交叉引用查看，定位到唯一用例,  
![](https://bbs.kanxue.com/upload/attach/202403/927366_QA8ZT6JWAFKSVJ5.webp)  
b 函数也传入了 j2，这里的 j2 是由 a 函数传入的，为第四个参数  
![](https://bbs.kanxue.com/upload/attach/202403/927366_HGCKBTRDHTPS4A3.webp)  
查看函数 a 的交叉引用，也是唯一的  
![](https://bbs.kanxue.com/upload/attach/202403/927366_4C2RVFHVF2H8529.webp)  
由另一个函数 b 传入，继续交叉引用，找到调用  
![](https://bbs.kanxue.com/upload/attach/202403/927366_AU8M2PEYUQM2DTF.webp)  
可以由名字推知 arg12 也就是 j2 就是用户 UIN  
由此可以得出聊天记录数据库文件加密方式：md5("1234567890ABCDEF"+[UIN]).substring(0,7)

### 索引库文件解密

因为索引库肯定是在数据库文件初始化之后才生成，且使用 FTS，可以直接顺着逻辑在 com.tencent.mm.plugin.fts 包中往下找  
![](https://bbs.kanxue.com/upload/attach/202403/927366_TDEUJMC92QUDX35.webp)  
还是三个字符串拼接，第一个最后返回值还是 UIN，第二个是 “1234567890ABCDEF”, 第三个我们跟踪一下  
![](https://bbs.kanxue.com/upload/attach/202403/927366_H7UWBGA93W9HVUD.webp)  
这里注意一下 log，先把代码放在这里

### 数据库查看

先将上面的密钥试着解一下数据库，当然是可以解开的  
![](https://bbs.kanxue.com/upload/attach/202403/927366_PP87GFHHEWQWHY6.webp)  
这里注意，我们看到了 userinfo 的表，结合上面索引库的 Log，便有了一些想法，先看一下这个表  
![](https://bbs.kanxue.com/upload/attach/202403/927366_RR44HSJGDUDXTXB.webp)  
里面有 userid 这个东西，推测一下，第三个字段可能就是用户 id  
这样索引文件的密钥就是 md5([UIN]+"1234567890ABCDEF"+[USERID]).substring(0,7)

后期补充和一些思考
=========

抱着试一试的想法去更改了一下数据库内容，试一下能不能实现聊天记录的篡改，结果成了。。。。具体操作过程就不再提供了（虽然试了一晚上只成功了一次）  
![](https://bbs.kanxue.com/upload/attach/202403/927366_B5P6UF5FXT9MS36.webp)  
对数据库记录进行更改，然后使用 adb pull 指令将原有文件进行替换，重新登陆微信，微信会自动检测到索引文件与聊天记录数据库不符，是否进行修复。如果不进行修复，那么索引文件就不会重新索引，索引文件中的原有记录不会改变；如果修复，则会重新启动微信，根据数据库文件进行重新索引  
虽然目前还没有见到改微信聊天记录的，但确实能通过修改数据库并替换的方式实现聊天记录的篡改，现在微信聊天记录也具有法律效益了，我觉得以目前的取证工作来说，聊天记录的法律上的有效性检测仍需进一步努力  
另外提供一个可以检测篡改的思路，不知可行性怎么样  
微信在更新版本时会生成上文中提到了一个数据库文件的备份，那个的加密方式好像和聊天记录数据库大致但又不完全相同，是用来微信版本更新恢复聊天记录的，可以作为判断聊天记录是否篡改的依据

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

最后于 3 小时前 被 PengLaiDoll 编辑 ，原因： 添加话题，修改了一部分文章

[#逆向分析](forum-161-1-118.htm) [#其他](forum-161-1-129.htm)