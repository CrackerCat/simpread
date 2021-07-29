> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1483253-1-1.html)

> 出于解密技术学习的目的，对某款阅读软件进行了学术上的研究操作。

![](https://avatar.52pojie.cn/data/avatar/000/35/89/70_avatar_middle.jpg)Light 紫星 _ 本帖最后由 Light 紫星 于 2021-7-27 09:19 编辑_  
出于解密技术学习的目的，对某款阅读软件进行了学术上的研究操作。  
本文仅提供思路，不提供任何成品软件等。  
首先，定位到本软件下载的图书目录，在 Android/data / 包名 / files/books 下面有几个文件夹，找一下就可以定位到你下载的那本书了。  
然后，反编译 apk，拖入 jadx，搜索 openbook，找到相关代码，最后经过定位，真正的 openbook 在 libjdxxxxreadingengine.so 里面。对应的函数是 Java_com_xx_read_engine_jni_DocView_OpenBookInternal  
通过对此函数分析，发现最后是调用的 xxdecompress::decrypt 进行解密文件，解密之前先实例化了 xxdecompress 对象，传入 key，这里的 key 可以通过 hook 获得，每本书的 key 都不一样，好像换了设备也会变，这里我没有尝试更换设备。  
如果发现此 so 文件不好分析，可以下载一个旧版的该 app，旧版的 so 是 xxxdrm.so，然后函数名称是一样的。  
这里分享一个 github 链接，该项目有部分开源的代码，是关于这个 app 的 ，这里面的 xxxdrm.so 可以直接拿来调用。  
[https://github.com/a-running-snail/read-android](https://github.com/a-running-snail/read-android)  
最后使用 androidemu，写一个 python 脚本调用 so 文件进行最终的解密操作  
部分解密代码如下：  
![](https://attach.52pojie.cn/forum/202107/27/090839u7htbhb5b7ow6c5x.png)

**image.png** _(44.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxNjMxMnxjYWU2OTA4M3wxNjI3NTI3MTY0fDIxMzQzMXwxNDgzMjUz&nothumb=yes)

2021-7-27 09:08 上传

  
最终的解密效果如图：  
![](https://attach.52pojie.cn/forum/202107/27/090725wagpuaiqr9lcrq3r.png)

**image.png** _(700.74 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxNjMxMXw1YTViMjE0M3wxNjI3NTI3MTY0fDIxMzQzMXwxNDgzMjUz&nothumb=yes)

2021-7-27 09:07 上传

  
至此，又解开了一个阅读软件的加密，好像这些软件都是一个套路，aes 加密，然后找个地方放 key，用的时候再解密回来。![](https://avatar.52pojie.cn/data/avatar/000/35/89/70_avatar_middle.jpg)Light 紫星

> [wanyanlong 发表于 2021-7-28 16:14](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=39453404&ptid=1483253)  
> 请问里面的电子书能导出来原来的 PDF 格式文件吗，注意是原来的，不是那种截图一张张保存的

是 epub 格式的，可以进一步转换成 pdf，和原来的是一样的，不是图片，是文字 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) sdieedu

> [randwong 发表于 2021-7-29 09:35](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=39461207&ptid=1483253)  
> 破解之后一看竟然是英文书籍，然后打开翻译软件慢慢看！果然还是要提高英文阅读能力，赞赞：）

能否给个详细介绍过程，或者成品？![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)CapitalIze 江湖上有这么一句话 -- “有问题，找星佬！” yyds！！！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)biostu 学习了，收藏。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)cshk8 真不错，学习了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) wuyue321 这个方法太实用了，技术就是生产力！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)hikaruyin 雖然不知是什麼讀書 app![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)dafei2599 很不错，非常棒，学习了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) water666 星佬，YYDS！ [QQ 图片 20210727092215.gif](forum.php?mod=attachment&aid=MjMxNjMzMXwyYTRmZWUzZHwxNjI3NTI3MTY0fDIxMzQzMXwxNDgzMjUz&nothumb=yes) _(24.36 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxNjMzMXwyYTRmZWUzZHwxNjI3NTI3MTY0fDIxMzQzMXwxNDgzMjUz&nothumb=yes)

2021-7-27 09:28 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202107/27/092809h44ppii4nnk7kixk.gif) ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)smallfiveya yyds 膜拜 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) RootMe 膜拜大佬，学习学习