> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1434932-1-1.html)  ![](https://avatar.52pojie.cn/data/avatar/000/06/75/58_avatar_middle.jpg) 微笑一刀

IDA Pro 的一个小插件 StrongCC(IDA Strong Charset Code)

主要就是用来处理一下中文显示的问题.

目前实现以下效果

选中状态  
![](https://attach.52pojie.cn/forum/202105/07/140512pnjlal1iz10pm0wl.jpg)

悬停状态  
![](https://attach.52pojie.cn/forum/202105/07/140514egvdg5h4dg4zr440.jpg)

热键 Ctrl+alt+3 处理后  
![](https://attach.52pojie.cn/forum/202105/07/140509hkm3mt76da8g4tmb.jpg)

IDA String Window 效果 (记得自己设置一下)  
![](https://attach.52pojie.cn/forum/202105/07/140517bo11uerqui1r12po.jpg)

初始化热键 Ctrl+Alt+Q

自动处理热键

ANSI Ctrl+Alt+1

UTF8 Ctrl+Alt+2

UTF16LE Ctrl+Alt+3 (对应 IDA String Type 里的 UNICODE)

感谢 海风月影 Hmily

放到 IDA\PLUGINS 目录下即可.

注意!!! 插件只会查找没被定义为字符串的类型数据. 因此可能会有遗漏.

没做太多测试. 目 & 猜 + 测 = 有不少 BUG. 欢迎反馈.

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[Plugin.zip](forum.php?mod=attachment&aid=MjI4Mzk1OXw5ZjQ2MjY2Y3wxNjIwNTk3MTQ2fDB8MTQzNDkzMg%3D%3D)

21.6 KB, 下载次数: 130, 下载积分: 吾爱币 -1 CB

售价: **4 CB 吾爱币**  [[记录](forum.php?mod=misc&action=viewattachpayments&aid=2283959)]

![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg)Hmily 一刀 666，大家发现有不能识别的字符串例子，可以放到网盘给一刀一个链接看看，好完善更新。![](https://avatar.52pojie.cn/data/avatar/000/16/13/92_avatar_middle.jpg)hack、小楠 我记得之前一刀有发过，这次终于又更新了 6666666666![](https://avatar.52pojie.cn/data/avatar/001/63/08/72_avatar_middle.jpg)djxding  
谢谢分享。  
好插件。  
![](https://avatar.52pojie.cn/data/avatar/001/61/84/23_avatar_middle.jpg)jucan123 谢谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/66/85/87_avatar_middle.jpg) surprises 好东西谢谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/53/91/75_avatar_middle.jpg) Zimin 感谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/61/91/50_avatar_middle.jpg) pentium315 辛苦了，感谢楼主分享。 ![](https://avatar.52pojie.cn/data/avatar/001/63/54/61_avatar_middle.jpg)xwhat 好插件，谢谢分享 ![](https://avatar.52pojie.cn/data/avatar/000/64/84/95_avatar_middle.jpg) 395552895 学到了。![](https://avatar.52pojie.cn/data/avatar/000/84/63/01_avatar_middle.jpg)加菲猫_1999 感谢分享 ![](https://avatar.52pojie.cn/data/avatar/000/72/89/30_avatar_middle.jpg) jinwenming 谢谢分享。。。。。。。。。。。