> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1382688-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/000/69/65/07_avatar_middle.jpg)LoLik  
**------------------------------------------------------------------------------** **环境:**  
手机:           pixel 1  
系统:           aosp 8.0                         编译 aosp 的方式此处不展开  
openssl :    使用 Git 中自带的              [https://git-scm.com/downloads](https://git-scm.com/downloads)  
**------------------------------------------------------------------------------**  
**起因:**  
因为新的安全机制, 系统不再信任用户证书,  
所以本次通过编译 ROM 的方式, 直接将 Charles 证书放置到安卓系统中.  
  
**------------------------------------------------------------------------------**  
**正文:**  
**1 charles 导出证书为 xxxx 文件, 格式为 pem**  
 **![](https://attach.52pojie.cn/forum/202103/03/220607q4uxre3dhi3ue4i3.png)**   
  
**2 使用 openssl 命令得到一个文件名**  
**openssl x509 -subject_hash_old -in xxxx.pem**  
 **![](https://attach.52pojie.cn/forum/202103/03/220609io9m0x907yox9mku.png)**   
  
**3 在 aosp8.0 工程中,/system/ca-certificates/files / 目录下,**  
**创建文本文件, 文件名为上面命令得到的那个数字, 以及配上一个 ".0" 后缀**  
 **![](https://attach.52pojie.cn/forum/202103/03/220611mccscq4g1t4hkh19.png)**   
  
**4 用编辑器打开 xxxx.pem, 将内容复制到上步 aosp 新创建的文本文件中.**  
**仅此一处修改, 接着正常编译 aosp 即可.**  
 **![](https://attach.52pojie.cn/forum/202103/03/220614di4dki0yeiistyif.png)**   
  
**5 编译成功后, 将 rom 刷入手机,**  
**查看 / system/etc/security/cacerts 是否存在 charles 证书文件,**  
**顺利的话应该存在, 此时 rom 已经内置了 charles 证书**  
 **![](https://attach.52pojie.cn/forum/202103/03/220616pqkkwa8x828prrpq.jpg)**   
  
**6 最后是常规的 charles 和手机配置流程**  
 **![](https://attach.52pojie.cn/forum/202103/03/220618neeegllvze6pxl0b.jpg)**   
  
 **![](https://attach.52pojie.cn/forum/202103/03/220620zzy62zuwjy927es5.jpg)**   
  
 **![](https://attach.52pojie.cn/forum/202103/03/220622ejb0s7vkkk7k0nu7.png)**   
  
 **![](https://attach.52pojie.cn/forum/202103/03/220624epmznidqr8ptsdoo.png)**   
  
**7 最后测试 HTTPS 抓包成功**  
 **![](https://attach.52pojie.cn/forum/202103/03/220626b6dd6n5ndpuohvsv.png)**   
  
**------------------------------------------------------------------------------**  
**另一种免编译, 免框架添加证书方法:**  
1 代 pixel Lineage 9.0 下, 添加抓包工具证书, 到系统信任  
[https://www.52pojie.cn/thread-1148892-1-1.html](https://www.52pojie.cn/thread-1148892-1-1.html)![](https://avatar.52pojie.cn/data/avatar/000/49/35/99_avatar_middle.jpg)djzhao 楼上说了，直接放到对应目录就可以，另外，你这个只是针对一台设备的 charls 生效，做一次编译太小题大做了  
免 root 抓包可以试试设置 app 访问用户证书  
[信任用户证书 (CA)，实现 Android7 及以上 HTTPS 抓包](https://blog.csdn.net/djzhao627/article/details/108658332?spm=1001.2014.3001.5502) ![](https://avatar.52pojie.cn/data/avatar/000/97/67/82_avatar_middle.jpg) jiebozhang 不 root 的手机也没法用 root 了的手机也不用这么麻烦![](https://avatar.52pojie.cn/data/avatar/001/58/30/76_avatar_middle.jpg)转身鬼魅 root 的手机直接放进去证书目录就好了 ![](https://avatar.52pojie.cn/data/avatar/000/25/28/00_avatar_middle.jpg) a5112075 root 后直接放入对应目录就行了 ![](https://avatar.52pojie.cn/data/avatar/000/96/03/70_avatar_middle.jpg) digitalhouse root 後直接放入對應目錄 + 1![](https://avatar.52pojie.cn/data/avatar/000/37/06/26_avatar_middle.jpg)repobor 过分复杂 Root 放入目录即可 ![](https://avatar.52pojie.cn/data/avatar/001/61/34/13_avatar_middle.jpg) PpaPingggg root 了放目录就行了![](https://avatar.52pojie.cn/data/avatar/001/53/36/80_avatar_middle.jpg)听风没有雨 放目录不香吗 ![](https://avatar.52pojie.cn/data/avatar/000/81/00/29_avatar_middle.jpg) 8438 文字教程看不明白，楼主能否录制个视频 ![](https://avatar.52pojie.cn/data/avatar/001/49/74/68_avatar_middle.jpg) cjc3528 非常好的教程，谢谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/23/35/86_avatar_middle.jpg) frankB4 虽然看不懂 但是看着真 np 啊