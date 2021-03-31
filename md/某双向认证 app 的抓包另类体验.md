> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1405913-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/001/41/01/98_avatar_middle.jpg)darbra 最近尝试学习某教育 app，却发现连包都抓不到。  
尝试了 SSLUnpinning 等方法均无功而返，遂怀疑本 app 采用了双向认证。  
 ![](https://attach.52pojie.cn/forum/202103/30/164122wp1ec1eg4kpcge10.png)   
使用某开源脚本进行 keystore 的 trace，果不其然，发现本 app 采用了 BKS 证书。  
 ![](https://attach.52pojie.cn/forum/202103/30/164141rwl2bfhwlbarl95k.png)   
ps：脚本地址 https://raw.githubusercontent.com/FSecureLABS/android-keystore-audit/master/fr[IDA](https://www.52pojie.cn/thread-1345176-1-1.html)-scripts/tracer-keystore.js  
果然在 assets 里发现有相关 bks 证书  
 ![](https://attach.52pojie.cn/forum/202103/30/164144mtx6bx600epr9ehh.png)   
接着尝试将 bks 证书导入 charles，却发现只得 p12 证书，（某灵魂 app 双向认证用的就是 p12 证书，想必大家都印象深刻吧）。  
之后尝试 bks 格式证书转换成 p12 的格式证书，转换后却发现无法使用。  
 ![](https://attach.52pojie.cn/forum/202103/30/164147e0fyqrut7tuy77qz.png)   
不可轻易放弃，遂尝试进行 https 到 http 的转换。  
首先 hook java.net.URL 看看是否有相应的 url 呈现。  
 ![](https://attach.52pojie.cn/forum/202103/30/164129ocm1mlz5jpwwv3kv.png)   
接着[脱壳](https://www.52pojie.cn/forum-5-1.html)，反编译 app，全局搜索此 host。  
 ![](https://attach.52pojie.cn/forum/202103/30/164138log1ooogth1fh33o.png)   
我们去到上述代码里探索一番，目标锁定 Global 静态方法，进行动态的 https 到 http 的转换。  
 ![](https://attach.52pojie.cn/forum/202103/30/164126qxxgoxgxpffjjxgl.png)   
 ![](https://attach.52pojie.cn/forum/202103/30/164132cf1jj0n4fe9u41z1.png)   
接着在 charles 就能看到鲜活的数据返回了，喜不自胜。  
 ![](https://attach.52pojie.cn/forum/202103/30/164135deeqkq24e7x894we.png)   
这次是和大家分享一个小方法，希望对大家有所帮助。  
在本人 github.com/darbra/sign 有更多的一些学习思路交流，如果对老师们有所帮助，不甚欣喜。![](https://avatar.52pojie.cn/data/avatar/001/41/01/98_avatar_middle.jpg)darbra

> [lyghost 发表于 2021-3-31 08:12](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=37767330&ptid=1405913)  
> 服务端要是强制 https 访问岂不是完蛋了

强制 https 的情况下 老师可以尝试使用 charles 里的 map remote 功能 又是一个小方法![](https://static.52pojie.cn/static/image/smiley/default/4.gif) ![](https://avatar.52pojie.cn/data/avatar/000/37/41/74_avatar_middle.jpg) vone

> [darbra 发表于 2021-3-30 17:48](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=37760283&ptid=1405913)  
> 搞不来 老师后面成功了告知一下 万分感谢

我也不知道，找不到方法 ![](https://avatar.52pojie.cn/data/avatar/000/53/10/37_avatar_middle.jpg) tanghengvip 这波操作涵盖的知识点不少啊 ![](https://avatar.52pojie.cn/data/avatar/001/66/20/83_avatar_middle.jpg) poiiop 谢谢楼主分享心得![](https://static.52pojie.cn/static/image/smiley/mogu/dyj.gif) ![](https://avatar.52pojie.cn/data/avatar/001/64/99/55_avatar_middle.jpg) jinguangfeiyu 脚本网址打不开呀 ![](https://avatar.52pojie.cn/data/avatar/001/41/01/98_avatar_middle.jpg) darbra

> [jinguangfeiyu 发表于 2021-3-30 17:00](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=37759369&ptid=1405913)  
> 脚本网址打不开呀

老师可以尝试用更科学的方式打开 ![](https://avatar.52pojie.cn/data/avatar/001/54/45/22_avatar_middle.jpg) key_user 知识点挺多的 ![](https://avatar.52pojie.cn/data/avatar/000/37/41/74_avatar_middle.jpg) vone 想问问楼主，bks 怎么转 p12？![](https://avatar.52pojie.cn/data/avatar/000/78/42/02_avatar_middle.jpg)13169456869 感谢分享！！！学习学习 ![](https://avatar.52pojie.cn/data/avatar/001/41/01/98_avatar_middle.jpg) darbra

> [vone 发表于 2021-3-30 17:48](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=37760271&ptid=1405913)  
> 想问问楼主，bks 怎么转 p12？

搞不来 老师后面成功了告知一下 万分感谢![](https://static.52pojie.cn/static/image/smiley/default/42.gif)