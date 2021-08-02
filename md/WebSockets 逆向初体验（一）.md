> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1487207-1-1.html)

> 首先打开我们的网址，进行抓包记得调对位置是 WS（WebSockets）不懂这一块的可以自己先看看网上教程，然后学一些，后面我会带一点知识出来点进去看看调用栈 ... WebSockets 逆向初体验（一）......

![](https://avatar.52pojie.cn/data/avatar/001/44/93/85_avatar_middle.jpg)QingYi. 首先打开我们的网址，进行抓包记得调对位置是 WS（WebSockets）  
![](https://attach.52pojie.cn/forum/202108/02/153024n6lz4b2ul4u4404k.png)

**1.png** _(111.34 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgwMHw1ZmVmMmZiYnwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:30 上传

  
不懂这一块的可以自己先看看网上教程，然后学一些，后面我会带一点知识出来  
点进去看看调用栈  
![](https://attach.52pojie.cn/forum/202108/02/153150e3sbcd86suvv3edi.png)

**2.png** _(178.46 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgwM3wyYzJkYTBlYXwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:31 上传

  
这里需要注意，仔细看看少了什么东西？我们今天来干的就是这个发送消息的。也就是 send  
![](https://attach.52pojie.cn/forum/202108/02/154509t9p6v8f9jjhvnhbv.png)

**3.png** _(108.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgxMnw0NjFmMGI5MHwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:45 上传

  
但是他有很多个，难不成我们一个一个去断点吗？  
![](https://attach.52pojie.cn/forum/202108/02/155010k5dms8hpmnm2zuvd.png)

**4.png** _(168.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgxM3xjYmQzMjlhYnwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:50 上传

  
好，我们对他进行 Hook，钓鱼  
因为肯定是 p 调用的，所以我们断点要断在这里  
![](https://attach.52pojie.cn/forum/202108/02/153654gxf3ggg7ttzmx7om.png)

**5.png** _(115.77 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgwOHxiMmEyYTE2YnwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:36 上传

  
好，我们把它勾住了  
![](https://attach.52pojie.cn/forum/202108/02/155215e2bhupw7tnl2u22b.png)

**6.png** _(94.22 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgxNnxlMTljNmM1ZnwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:52 上传

  
我们来看调用栈，顺便打个断点  
![](https://attach.52pojie.cn/forum/202108/02/155251pspcspcpyssnhppv.png)

**7.png** _(159.78 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgxN3wyY2Y1OGM0OHwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:52 上传

  
我们现在要回去 p 那边，把钩子取下来  
![](https://attach.52pojie.cn/forum/202108/02/155419fqir3pxqc3qixeuc.png)

**8.png** _(160.52 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgxOHw1NjZhOTNkZHwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:54 上传

  
发送的时候断住了  
![](https://attach.52pojie.cn/forum/202108/02/155525s61o7bcwh7nj5duc.png)

**9.png** _(296.25 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyMHxlZjVlNGZjOHwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:55 上传

  
追栈，发现这个就是主体  
![](https://attach.52pojie.cn/forum/202108/02/155628pwbxlbcclssbjxvu.png)

**10.png** _(276.65 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyMXw1MTBlMWFlOXwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:56 上传

  
我们可以看到，他就是 d 搞出来的名堂，我们把 d 给抓出来  
![](https://attach.52pojie.cn/forum/202108/02/155814l2jfecpceg39pag2.png)

**11.png** _(110.41 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyMnxhYmU4ODhjY3wxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 15:58 上传

  
改写，顺带把当前可以看到的数据先拿出来  
![](https://attach.52pojie.cn/forum/202108/02/160910k7ujl083t63s0q0l.png)

**12.png** _(317.21 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyNHwyMmJmNmQ3MHwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:09 上传

  
然后我们发现还差一个 p.Wup，这个 p 又是什么，wup 又是神马？  
其实它在上面，仔细看下这个流程  
![](https://attach.52pojie.cn/forum/202108/02/161856thop006aozo7psqa.png)

**14.png** _(75 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyNnw4NjYzYWJjYXwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:18 上传

  
我们现在点进去看看这个函数是什么东西  
![](https://attach.52pojie.cn/forum/202108/02/162203k2p3crpy3r7eeeya.png)

**15.png** _(107.76 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyN3xjMmZkMTRjZXwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:22 上传

  
随便找一行来搜索一下  
![](https://attach.52pojie.cn/forum/202108/02/162245ljz2n7nxkxw6xjdi.png)

**16.png** _(113.39 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyOXw2N2FiMDZjNnwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:22 上传

  
他其实是通过这个方法来的  
![](https://attach.52pojie.cn/forum/202108/02/162326gnx96nx58md79ox9.png)

**17.png** _(141.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgzMHwyNjQwNDViNXwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:23 上传

  
仔细想一下这个玩意是不是在哪里似曾相识  
![](https://attach.52pojie.cn/forum/202108/02/162458ajc8ce9cg2ttcidj.png)

**18.png** _(70.45 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgzMXwyMTFjNzYzMnwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:24 上传

  
好，仔细想一下。  
我们现在是要用 p，用 p 去点出方法但是 wIU9 这个东西怎么拿出来呢？  
其实这是一个 webpack 的包（具体可以看我这个帖子：[https://www.52pojie.cn/thread-1469095-1-1.html](https://www.52pojie.cn/thread-1469095-1-1.html)）  
我们要对他进行改写  
![](https://attach.52pojie.cn/forum/202108/02/161605r7s3s5dcon6suozk.png)

**13.png** _(130.7 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgyNXxhMmJiZmYwNXwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:16 上传

  
是不是有这个东西?  
![](https://attach.52pojie.cn/forum/202108/02/163636jsnc1nscocnyz15o.png)

**19.png** _(251.95 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODgzNnw2NGYwZTg1OHwxNjI3OTAwNjAyfDIxMzQzMXwxNDg3MjA3&nothumb=yes)

2021-8-2 16:36 上传

  
好，本次教程还没完，后续的下次写，你们先看着。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)gubs 期待后续的教程~![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)ForJayChou 厉害啊 膜拜 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) Qq76761043 宫后小猪多是~![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)wf84674227 哈哈哈哈哈，我是来看高手的