> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1148892-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/000/69/65/07_avatar_middle.jpg)LoLik  
------------------------------------------------------------------------------ **环境:**  
手机:           pixel 1  
系统:           Lineage OS 9.0              https://download.lineageos.org/  
权限:           已 ROOT                         同样由 Lineage 官网提供  
抓包工具:    burpsuite_pro_v2020.2  [https://www.52pojie.cn/thread-1038295-1-1.html](https://www.52pojie.cn/thread-1038295-1-1.html)  
openssl :    使用 Git 中自带的              https://git-scm.com/downloads  
命令行:       Powershell                      shell 也行, 但是 cmd 不行  
------------------------------------------------------------------------------  
**起因:**  
因为新的安全机制,  
系统不再信任用户证书,  
所以现在抓包,  
要把 brup/charles 的证书,  
弄到系统信任中,  
不再是用户信任.  
------------------------------------------------------------------------------  
**准备工作:**  
安装 Git, 找到 openssl:  
 ![](https://attach.52pojie.cn/forum/202004/05/135621tgpovfpe4avrg9r7.png)   
------------------------------------------------------------------------------  
**过程:**  
1 获取 burp 的证书  
----> 打开 burp---->proxy---->options---->export CA certificate----> 导出证书到 der 文件.  
 ![](https://attach.52pojie.cn/forum/202004/05/135035yu2m4uyi6c234y6j.png)   
  
2 将 der 证书转换为 crt 证书  
openssl x509 -inform DER -in 1111.der -out 2222.crt  
  
  
3 依次执行下面命令, 生成两个 " 哈希值. 0" 命名的证书文件  
(1) 计算 crt 证书的两个哈希值  
openssl x509 -inform PEM -subject_hash       -in 2222.crt    // 只关注第一行内容, 为哈希值: 7bf17d07, 这个值后面用来给新的文件起名  
openssl x509 -inform PEM -subject_hash_old -in 2222.crt   // 只关注第一行内容, 为哈希值: 9a5ba575, 这个值后面用来给新的文件起名  
  
(2) 生成  
// cat 命令 + crt 证书文件 + 刚才获得的哈希值 + ".0" ,  
cat 2222.crt > 7bf17d07.0  
cat 2222.crt > 9a5ba575.0  
(3) 进一步操作  
openssl x509 -inform PEM -text -in 2222.crt -out /dev/null >> 7bf17d07.0  
openssl x509 -inform PEM -text -in 2222.crt -out /dev/null >> 9a5ba575.0  
(4) 终于搞完, 文件是这样的.  
 ![](https://attach.52pojie.cn/forum/202004/05/135144ysfws9qewqjfhq8v.png)   
4 将这两个证书文件复制到手机目录 /system/etc/security/cacerts  
依次执行下面命令  
adb root  
adb disable-verity  
adb reboot  
  
adb root  
adb remount  
adb push 7bf17d07.0 /system/etc/security/cacerts/  
adb push 9a5ba575.0 /system/etc/security/cacerts/  
  
  
5 启用证书  
----> 手机上  
----> 设置  
----> 安全性  
----> 加密与凭证  
----> 信任的凭证  
----> 系统  
----> 往下翻  
----> 找到两个 PortSwigger(没找到的话, 也可能已经信任了, 只是列表里没显示)  
----> 启用  
----> 抓包检验是否成功安装证书  
------------------------------------------------------------------------------  
  
**更新:**  
另有一种方式, 在编译 ROM 时, 直接将 Charles 证书放置到系统根证书区, 参考:  
[https://www.52pojie.cn/forum.php?mod=viewthread&tid=1382688&page=1&extra=#pid37183043](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1382688&page=1&extra=#pid37183043)  
  
  
  
  
![](https://avatar.52pojie.cn/data/avatar/000/69/65/07_avatar_middle.jpg)LoLik

> [丶那年如此年少 o 发表于 2020-4-5 10:43](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=31137259&ptid=1148892)  
> 大佬，openssl 文件上传下，还有输出证书的名字命名是随便取还是有要求的？

1 WIN10 安装 GIT,GIT 安装后就提供了 Openssl 工具.  
2 证书的名字不能随便起, 要按照那个步骤生成.  
![](https://avatar.52pojie.cn/data/avatar/000/69/65/07_avatar_middle.jpg)LoLik

> [testhcom 发表于 2020-4-30 11:36](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=31669897&ptid=1148892)  
> 有解决么 访问慢的问题，我也碰到这个问题了。

我有 2 个想法,  
1 编译 AOSP, 向系统添加证书.  
2 编译 AOSP, 修改系统信任用户证书.  
不过需要花时间琢磨一下怎么弄. ![](https://avatar.52pojie.cn/data/avatar/000/29/83/91_avatar_middle.jpg) 至尊丶夏末之刃 这下亲儿子都不好使了。 爱。![](https://avatar.52pojie.cn/data/avatar/000/15/33/54_avatar_middle.jpg)lovxyj 谢谢分享 1![](https://avatar.52pojie.cn/data/avatar/001/33/30/75_avatar_middle.jpg)leezc 优秀，学习了![](https://avatar.52pojie.cn/data/avatar/000/66/66/95_avatar_middle.jpg)【开★心快乐】 能来个视频嘛  
小白看不不明白 &#128514;&#128514;&#128514;  
我想把黄鸟可以弄得嘛 ![](https://avatar.52pojie.cn/data/avatar/001/37/16/78_avatar_middle.jpg) 15295828305 ![](https://static.52pojie.cn/static/image/smiley/laohu/laohu3.gif)谢谢你的分享，给大家带来更多的方法。![](https://avatar.52pojie.cn/data/avatar/000/42/28/88_avatar_middle.jpg)拍拍熊 谢谢分享，学习到了 ![](https://avatar.52pojie.cn/data/avatar/000/30/55/83_avatar_middle.jpg) kenda 问题是需要手机 root![](https://static.52pojie.cn/static/image/smiley/default/8.gif) 华为手机难 root![](https://avatar.52pojie.cn/data/avatar/001/35/59/43_avatar_middle.jpg)Linshengqiang 越来越难了啊![](https://avatar.52pojie.cn/data/avatar/000/81/38/94_avatar_middle.jpg)丶那年如此年少 o 大佬，openssl 文件上传下，还有输出证书的名字命名是随便取还是有要求的？