> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [gist.github.com](https://gist.github.com/5ec1cff/5f2cac77f4fc3679927dc8248101efec)

> MIUI 12.5 下使用 Magisk v24.2 随机包名. GitHub Gist: instantly share code, notes, and snippets.

最近把 Magisk 从 alpha 升级到了 24.2 ，不过 Alpha 频道有一个提醒：

[https://t.me/magiskalpha/457](https://t.me/magiskalpha/457)

> MIUI 设备在升级 官方 v24.2 后可能会无法隐藏 / 还原 Magisk app，这是 MIUI 系统本身的问题，请向 MIUI 反馈 PackageInstaller API ( [https://developer.android.com/reference/android/content/pm/PackageInstaller](https://developer.android.com/reference/android/content/pm/PackageInstaller) ) 无法正常工作。  
> 临时解决办法：关闭 MIUI 优化后重试。

自己用的刚好是 MIUI 12.5 ，试了一下，尝试随机包名安装后一直提示失败。

查了一下源码，引入这个改进是这个提交：

[Rewrite app installation · topjohnwu/Magisk@baa19f0](https://github.com/topjohnwu/Magisk/commit/baa19f0ccf85610249db1f5259377af6ee07fa01)

想起某个频道说 MIUI 破坏了重要的安装器 API ，难道是这个导致的？

[https://t.me/vvb2060Channel/597](https://t.me/vvb2060Channel/597)

```
[回覆 南宫雪珊]
在迁移到新 ( ? since Android 5 ) API时发现，MIUI破坏了SDK。

日志1：
android.content.ActivityNotFoundException: No Activity found to handle Intent { act=android.content.pm.action.CONFIRM_INSTALL flg=0x10000000 pkg=com.miui.packageinstaller (has extras) }

MIUI的默认软件包安装器没有实现 CONFIRM_INSTALL。但关闭MIUI优化时会禁用com.miui.packageinstaller，启用Google安装器，所以还是通过了CTS。

日志2：
E/PKMSImpl: MIUILOG- assertCallerAndPackage: uid=10018, installerPkg=com.topjohnwu.magisk, msg=Permission denied

鉴于Android12以后，这个API有强大的诱惑力（能静默更新自己，无需用户确认安装），相信大量开发者会开始跟进，希望MIUI能尽快修复。
（整个 PackageInstaller API都坏掉，MIUI是真厉害。

更新：根据MIUI的回复，PackageInstaller API (https://developer.android.com/reference/android/content/pm/PackageInstaller)（Android 5.0加入公开SDK）在 MIUI 12.5 前不受支持，无法使用(即日志1)，是故意为之。
但是，据报告12.5以后依旧无法使用。

更新2：根据MIUI的回复，12.5有正常过一段时间，然后又被改坏(即日志2)，已经在修了。是framework方面的问题，只能通过OTA推送。变通方案可关闭MIUI优化，绕过这段逻辑。

更新3：MIUI已停止修复。目前状态为使用此API即会使app崩溃。


```

一开始并没有理解，以为只是简单调用了 action.VIEW 安装 apk ，而 MIUI 特别照顾了自家的安装器，既然这样我们就让它打开 google 的好了。

MIUI 并非没有 google installer ，只是在启动的时候把它卸载了，在 priv-app 下可以看到 com.google.android.packageinstaller 。

把它装回来：

```
pm install-existing --user 0 com.google.android.packageinstaller


```

然后 `am start -a android.intent.action.VIEW -d content://a.apk -n android/com.android.internal.app.ResolverActivity` ，打开系统选择器，选择 google 安装器，勾选「默认用此应用打开」，这样就能盖住 MIUI 的特殊照顾了。

然而并没有任何用处，安装还是失败。

frida 抓了一下 intent ，发现它发送的 action 并非是 VIEW ，而是 CONFIRM_INSTALL (`android.content.pm.action.CONFIRM_INSTALL`) ，并且 component 也被解析成 com.miui.packageinstaller 。

于是故技重施，发了一遍这个 action 的 intent ，让系统选择器设置默认，但抓取 intent 发现还是 com.miui.packageinstaller ，而自己 pm resolve-activity 似乎又没问题，正确解析到了 google installer 。

仔细研究了源码，发现原来这个 intent 实际上是来自系统广播的，大概流程是，使用 PackageInstaller 打开 session ，会向 app 发送广播（action 是 session id），广播包含一个进行用户确认的 Activity intent (CONFIRM_INSTALL) ，由应用发出，而这个 intent 的 package 被设置成了系统唯一指定的 installer 。

这个唯一指定的 installer 实际上是开机时确定的，系统会检查能够安装 apk 的 priv app ，并且确保有且仅有一个（否则会导致 crash ，因此 miui installer 和 google installer 只能留一个），然后把包名保存到 PMS.mRequiredInstallerPackage 里面。

显然开机的时候只有 miui installer ，而它又无法处理 CONFIRM_INSTALL ，因此即使加上了 google installer 也无济于事。

难道真的只能临时关闭 MIUI 优化吗？回想起自己曾经关闭 MIUI 优化用了一个多月的各种诡异经历，比如之前给应用授权过的权限失效，应用消失…… 而且自己反编译过，发现实际上开启和关闭过程都会一定程度上破坏功能，因此这个开关十分危险。

拿到随机包名后的 apk 自行安装也很麻烦，因为首先 patch apk 这个操作似乎是直接在内存中完成的（patch 后直接发给 installer），就算拿到了，也要让 magisk 认识 app 的包名才行（虽然就是改一下 db 的事？）

所以我选择了相对方便的方法：frida hook ，这样不用动危险开关，甚至不用重启！

```
// frida-inject -n system_server -s xxx.js
Java.perform(() => {
    const IPMS = Java.use('android.content.pm.IPackageManager$Stub');
    const binder = Java.use('android.os.ServiceManager').getService('package');
    const pms = Java.cast(IPMS.asInterface(binder), Java.use('com.android.server.pm.PackageManagerService'));
    console.log(pms.mRequiredInstallerPackage.value);
    pms.mRequiredInstallerPackage.value = "com.google.android.packageinstaller";
})

```

安装 google installer ，然后直接修改这个 mRequiredInstallerPackage 的值，果然成功了！

[![](https://user-images.githubusercontent.com/56485584/156790619-4b39af8f-eaa2-44bf-ac04-0da939951d1d.png)](https://user-images.githubusercontent.com/56485584/156790619-4b39af8f-eaa2-44bf-ac04-0da939951d1d.png)

[![](https://user-images.githubusercontent.com/56485584/156790662-28f77a77-15fc-49b2-aa75-29056b4368be.png)](https://user-images.githubusercontent.com/56485584/156790662-28f77a77-15fc-49b2-aa75-29056b4368be.png)

总之，是成功在 MIUI 上给 v24.2 随机包名了（本来都打算干脆卸载 Magisk App 来逃避检测了），不过每次都这样操作也很麻烦（有可能 Magisk App 更新也要这样？），可以考虑写个 xposed 模块修正（可以是修改 intent ），当然如果 MIUI 能够把对这个 API 的支持给修正了才是最好的。