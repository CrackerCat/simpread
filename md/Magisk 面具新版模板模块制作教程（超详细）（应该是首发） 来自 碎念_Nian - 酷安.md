> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.coolapk.com](https://www.coolapk.com/feed/16056941?shareKey=YWI0MDFiYWE1Y2E3NWUyYzA3ODc~&shareUid=1124169&shareFrom=com.coolapk.market_10.0.1)

> [链接]Magisk 模块模板

[[链接]Magisk 模块模板](http://docq.cn/files/4kap3kp)

链接 2（蓝奏云）：

[[链接]Magisk 模块模板](https://wwx.lanzoux.com/b06lmwf1i)

![](http://image.coolapk.com/feed/2020/0124/10/1315644_b80f1054_1590_6608@1380x2913.png.m.jpg)

模板首页

修改模块配置的话打开第一个进去修改（META-INF/com/google/android/update-binary）  
第二个文件不用管他

![](http://image.coolapk.com/feed/2020/0124/10/1315644_808ca46e_1590_661@1380x2913.png.m.jpg)

模板配置

这个是安装的时候提示的信息  
$pounds 是星号，就是这个 *****，一般是用来隔行的  
$MODNAME 是模块名称，对应 module.prop 里面的 name

![](http://image.coolapk.com/feed/2020/0124/10/1315644_324cff0e_1590_6612@1380x2913.png.m.jpg)

安装时的提示

这里是文件权限，如果想对单个文件进行权限设置，需要去 customize.sh 里面把 SKIPUNZIP=0 改为 SKIPUNZIP=1  
其他的，注释已经说的明明白白了，我就不解释了，不懂可以评论或者私信来问我

![](http://image.coolapk.com/feed/2020/0124/10/1315644_f68a6eb3_1590_6614@1380x2913.png.m.jpg)

模板权限

这里是替换文件的，假如你要替换开机动画，就在这个文件夹里面（system 里面）新建一个 media 文件夹，然后把开机动画文件（bootanimation.zip）放到 media 里面，因为开机动画的文件是在系统的 system/media 里面，所以模板里面也是对应这个路径，这样讲你们应该可以听懂吧

![](http://image.coolapk.com/feed/2020/0124/10/1315644_296b51f4_1590_6616@1380x2913.png.m.jpg)

要替换的文件

这个是模块脚本配置，注释里面也说的明明白白  
对了  
这个 REPLACE 不是替换文件![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)  
这个 REPLACE 不是替换文件![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)  
这个 REPLACE 不是替换文件![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)  
这个 REPLACE 不是替换文件![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)  
这个 REPLACE 不是替换文件![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)  
重要的事情说五遍，我看现在 90% 的教程都是把 REPLACE 说成了替换文件![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)他其实是删除文件的，删除路径文件夹内的所有文件（不会删除文件夹只会删除文件夹内所有文件）  
刚开始我也是以为是替换文件，后来问了一些大佬才知道这个原来是删除文件的![](http://static.coolapk.com/emoticons/v9/c_coolb.png)![](http://static.coolapk.com/emoticons/v9/c_coolb.png)![](http://static.coolapk.com/emoticons/v9/c_coolb.png)我自己用虚拟机试了一下，也确实是删除文件，而不是替换文件！！！![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)

![](http://image.coolapk.com/feed/2020/0124/10/1315644_1017d78b_1590_6617@1380x2913.png.m.jpg)

脚本配置

另外这俩个图我觉得应该不用讲了，不懂的话评论或者私信问我

![](http://image.coolapk.com/feed/2020/0124/10/1315644_e3c1958f_1590_6619@1380x2913.png.m.jpg)

模块信息

![](http://image.coolapk.com/feed/2020/0124/10/1315644_6f86a4ea_1590_6621@1380x2913.png.m.jpg)

修改 build.prop

总结一下，如果你不想折腾，做替换文件的模块，只需要修改  
META-INF/com/google/android/update-binary 里面的安装信息  
system 里面的替换文件  
module.prop 里面的模块信息  
就可以了

最后提醒一句，刷机有风险

，请严格按照教程来制作模块，不懂请及时问楼主，避免不必要的变砖![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_64_shounuehuaji.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_64_shounuehuaji.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_64_shounuehuaji.png)

本图文如有侵权请联系我，我马上删![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)

最后我再说一遍，REPLACE 不是替换文件，是删除文件！！！![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_56_dogexiaoku.png)

教程就这些，如果喜欢的话就点个关注吧![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_37_doge.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_37_doge.png)![](http://static.coolapk.com/emoticons/v9/coolapk_emotion_37_doge.png)