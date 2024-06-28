> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282311.htm)

> [原创]Pixel6 降级记录

Pixel6 降级记录
===========

因为一些原因，手机升级到了 android15，现在需要降回 android13。 声明：仅做记录及分享，不负任何责任。

![](https://bbs.kanxue.com/upload/attach/202406/950902_EFDDUP5H25TMZJF.jpg)

查看当前两个 bootloader 的版本，使用 `fastboot set_active a` 或者 `fastboot set_active b` 切换 bootloader 可以看到是不一致的，大佬提示当切换到低版本 bootloader 时不要随便开机，很可能开机就变砖，而且这里还有个坑，执行了 fastboot set_active b 后 虽然槽位已经显示切为 b 了，但版本还是显示 a 槽的 15，并没有更新 b 槽的版本，实际上此时 b 槽现在依然是低版本， 所以现在很可能开机就变砖，我的这张 b 槽低版本的图片好像是按 start , 但他没有开机，而是重启了 bootloader，这时它才真正的显示了 b 槽的版本

![](https://bbs.kanxue.com/upload/attach/202406/950902_SAZ7ZJ82EMUGWKF.jpg)

![](https://bbs.kanxue.com/upload/attach/202406/950902_YKGV5EN2H8PFTYF.jpg)

所以现在最好是两个槽位都刷到最新的 15 版本，因为现在是 b 槽位版本低所以执行 `fastboot set_active b` ，然后再刷一遍 android15 [https://flash.android.com/preview/vic-beta2.2?hl=zh-cn](https://flash.android.com/preview/vic-beta2.2?hl=zh-cn) 这里务必注意取消 Lock Bootloader 的勾选，根据我以前刷小米的经验，这里不取消八成就把你的 Bootloader 给锁上了

![](https://bbs.kanxue.com/upload/attach/202406/950902_NHAMHT7NVK64JGJ.jpg)

小结： set bootloader a 刷一次官方包，然后 set bootloader b 再刷一次官方包

![](https://bbs.kanxue.com/upload/attach/202406/950902_M56SEEW8DTKGM5U.jpg)

然后开始准备降级，把 android13 中的 img 拷贝到桌面

![](https://bbs.kanxue.com/upload/attach/202406/950902_6CWGWXVZN73SA9D.jpg)

配置好桌面为环境变量

![](https://bbs.kanxue.com/upload/attach/202406/950902_WE66T3UVEMXJTGY.jpg)

此时我们在桌面路径的 cmd 中执行 adb reboot bootloader 和 fastboot flashall -w 即可开始刷机，但此时提示我们 bootloader 版本不对

![](https://bbs.kanxue.com/upload/attach/202406/950902_S2PRYVWG4MNCVCA.jpg)

所以我们需要去掉这个版本验证 ，删除原本 android-info.txt 中后面的两行

```
require board=slider|whitefin|oriole|raven
# 此行删除   require version-bootloader=slider-1.2-9465321             
# 此行删除   require version-baseband=g5123b-112825-230323-B-9800744

```

此时重新执行 fastboot flashall -w 即可开始降级

![](https://bbs.kanxue.com/upload/attach/202406/950902_Y323CK46QZHAJRD.jpg)

成功降级 android13

![](https://bbs.kanxue.com/upload/attach/202406/950902_7FHB9UBR4FQNVCJ.jpg)

总结：变砖原因有两个，一个是刷了低版本的 bootloader 变砖，另一个是刷机失败，系统自动切到了另一个 bootloader，如果另一个是低版本的，那么也一样会变砖，这也是为什么最好把 a、b 两个槽位都刷成一样的高版本，还有为什么我没使用官方的 bat 刷机脚本，因为官包里面包含旧版的 bootloader 如果哪次刷机的时候不小心忘记删了就会变砖，所以拷贝到桌面只刷 img 是个好主意，另外感谢群里大佬的技术支持

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 14 小时前 被简单的简单编辑 ，原因：

[#系统相关](forum-161-1-126.htm)