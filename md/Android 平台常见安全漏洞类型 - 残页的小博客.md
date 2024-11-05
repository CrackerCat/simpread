> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.canyie.top](https://blog.canyie.top/2024/11/05/android-platform-common-vulnerabilities/)

> 本文适用于已对 Android 开发有基础了解，希望了解 Android 系统层常见安全漏洞的人。

本文适用于已对 Android 开发有基础了解，希望了解 Android 系统层常见安全漏洞的人。祝大家写代码无 bug，挖洞天天挖到 Critical RCE 漏洞链。  
本文开始创作时间：2024-02-19 完成时间：2024-02-29 发布时间：2024-11-05

~过年了，不要再讨论什么 CVE、CNVD、CNNVD 之类的了。你的漏洞们不能给你带来任何实质性作用，朋友们兜里掏出一大把钱吃喝玩乐，你默默的在家里打开你的 Test PLMN 。亲威朋友吃饭问你收获了什么，你说我的漏洞被谷歌评级高危了，Android 14 最新安全补丁都能用，亲戚们懵逼了，你还在心里默默嘲笑他们，笑他们不懂你的 BAL，不懂什么是 BG-FGS，不懂怎么利用 confused deputy 类型漏洞进行跨用户读取，不懂 LaunchAnywhere，不懂 PendingIntent 要 FLAG_IMMUTABLE，笑他们手机上的拼多多。你父母的同事都在说自己的子女一年的收获，儿子买了个房，女儿买了个车，姑娘升职加薪了，你的父母默默无言，说我的女儿天天在家里对着电脑上的一堆英文发呆，嘴里念叨谷歌怎么还不回我，有时候还给我们发一堆乱码的文件。~

[](#权限，没有你我怎么活啊权限 "权限，没有你我怎么活啊权限")权限，没有你我怎么活啊权限
-----------------------------------------------

### [](#Android-权限机制概述 "Android 权限机制概述")Android 权限机制概述

权限肯定是每个 Android 开发者都用过的东西，比如应用要访问网络，就要像这样加上网络权限：

```
<uses-permission android: />
```

每个应用在被安装时都会被赋予一个 UID，包括系统应用。一般情况下，每个应用都会有一个独一无二的 UID，除非应用使用 `sharedUserId` 与其他应用共享 UID（这里忽略配置了 `isolatedProcess=true` 的服务进程，它们使用随机 UID 且几乎没有任何权限）。Android 系统内部也使用 UID 区分调用者。

Android 中通过 `getSystemService()` 可以拿到一堆的 `XxxManager`，而几乎所有的这些 `XxxManager` 都会和 system_server 暴露出来的一个 `XxxManagerService` 进行交互。应用调用这些系统服务提供的接口时，如果需要权限，系统会首先校验权限。以接口 `ConnectivityManager.getActiveNetwork()` 为例，它需要 `android.permission.ACCESS_NETWORK_STATE` 这个权限，所以 `ConnectivityService` 就会主动校验这个权限：

```
private void enforceAccessPermission() {
    mContext.enforceCallingOrSelfPermission(
            android.Manifest.permission.ACCESS_NETWORK_STATE,
            "ConnectivityService");
}

@Override
@Nullable
public Network getActiveNetwork() {
    enforceAccessPermission();
    return getActiveNetworkForUidInternal(mDeps.getCallingUid(), false);
}
```

这里的 `enforceCallingOrSelfPermission` 内部就会调用 `Binder.getCallingUid()` 拿到调用者的 UID 然后检查这个 UID 是否具有对应权限。所以实际上，鉴权用的是调用者 UID 而不是包名。

权限还有一种特殊类型，叫做特殊权限，定义时在 `protectionLevel` 中加入 `appop` 的就是。特殊权限描述一组对系统有特殊意义的权限，如悬浮窗权限和修改系统设置权限。设置中有一个 “特殊应用权限” 的页面专门用来控制这些特殊权限。要检查这些特殊权限的授权状态需要用到 `AppOpsManager`。

系统中还有很多别的安全机制，如 SELinux、capability 和 seccomp。这里略过不讲，感兴趣的可以自行搜索。

想要以一个系统开发者的身份了解更多关于权限的知识，可以查看谷歌官方说明文档：[Android permissions for system developers](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/permission/Permissions.md)

### [](#常见漏洞类型：主动鉴权不当 "常见漏洞类型：主动鉴权不当")常见漏洞类型：主动鉴权不当

#### [](#CVE-2023-40094：鉴权缺失 "CVE-2023-40094：鉴权缺失")CVE-2023-40094：鉴权缺失

补丁链接：[Require permission to unlock keyguard](https://android.googlesource.com/platform/frameworks/base/+/1120bc7e511710b1b774adf29ba47106292365e7)

此种类型的漏洞可谓是最简单最经典的漏洞，对 Android 权限模型足够熟悉的人可能看一眼就能知道有问题，且漏洞危害性往往较高，如此例的 CVE-2023-40094，允许任意 app 调用特权 API 无密码解除锁屏，Google 也给出了高危评级。这种漏洞简单但可遇不可求，要在 AOSP 上百 GB 的源码仓库中发现未被适当保护的特权 API，可谓是大海捞针，捞到一个白嫖一个 CVE，偷着乐去。近几个月安全补丁中大概 1~3 个月就会有这种漏洞被公开，说明这种漏洞可能还存在不少，可能集中在新加的 API 和被重构过的服务中，还有各个 OEM 添加的自定义 API。

#### [](#CVE-2021-0554：鉴权在客户端 "CVE-2021-0554：鉴权在客户端")CVE-2021-0554：鉴权在客户端

补丁链接：[Enforce BACKUP permission on Service end.](https://android.googlesource.com/platform/frameworks/base/+/4ab5af1eb569a46f2117377382aac7dc6535cfc5)

Android 很多系统服务向应用暴露了自己的接口，允许应用调用。一般来说，应用这边使用的 API 叫 XxxManager，system_server 里会有一个对应的 XxxManagerService。应用这边的 XxxManager 基本上什么都不干，就只是调用 system_server 里的 XxxManagerService 而已，真正干活的是 XxxManagerService。

发现了什么吗？以 ActivityManager 为例，虽然 ActivityManager 和 ActivityManagerService 都是系统的类，但是应用调用 API 的时候，ActivityManager 这个类里的代码是运行在调用者进程的，只有 ActivityManagerService 是在系统进程的！在调用者进程意味着调用者可以随便干扰，所以所有鉴权操作都应该放在系统进程里的 ActivityManagerService 以避免被干扰。而在这个漏洞中，代码刚好写反了，app 调用 BackupManager 的时候，BackupManager 在当前进程检查一遍权限，而 BackupManagerService 中的接口却没有被保护。无论是利用反射、hook，或者直接调用 IBackupManager 都可以轻易绕过权限检查。

这个其实和上一个没什么区别了，没有人这么写代码，近几年也没再看见过类似漏洞，参考价值不大，纯当乐子看就好。

#### [](#CVE-2015-6624-amp-CVE-2015-6625：特殊接口-dump "CVE-2015-6624 & CVE-2015-6625：特殊接口 dump")CVE-2015-6624 & CVE-2015-6625：特殊接口 dump

CVE-2015-6624 补丁：[Add DUMP permission check to ResourceManagerService.](https://android.googlesource.com/platform/frameworks/av/+/f86a441cb5b0dccd3106019e578c3535498e5315%5E%21/#F0)

CVE-2015-6625 补丁：[Add DUMP permission check to WifiScanner service.](https://android.googlesource.com/platform/frameworks/opt/net/wifi/+/29fa7d2ffc3bba55173969309e280328b43eeca1%5E%21/#F0)

这两个漏洞本质上都还是接口缺权限校验，但漏洞点位于特殊接口 dump 内。dump 是 binder 内置的接口，很少被开发者注意到，仅用于在调试时（如 dumpsys 或 cmd xxx dump）输出信息。为了保护这些敏感数据，调用者应该具有 `android.permission.DUMP` 权限，而大部分接口也确实检查了权限，但仍然不排除有开发者失误的情况。识别这种漏洞可以尝试使用自动化工具，多设备批量对所有系统服务进行测试。不过现在 CTS 测试会确保没权限的时候调用 dump 任何服务都不会返回正确信息，所以应该也没什么挖掘价值。

#### [](#CVE-2020-0109：特殊接口-shellCommand "CVE-2020-0109：特殊接口 shellCommand")CVE-2020-0109：特殊接口 shellCommand

补丁链接：[Fix notification shell commands](https://android.googlesource.com/platform/frameworks/base/+/adc39de3a148a2058d63bd7a1b8b71ee0a3524ac%5E%21/#F0)

和上一条类似，不过这里是特殊接口 shellCommand，该接口用于 adb shell 命令行调试时调用 `cmd xxx yyy`。和上一条的 dump 相比，这一项更为复杂一点：

1.  Binder.java 中默认的 `onShellCommand` 实现会保证调用者 UID 为 root 或 shell，不满足直接拒绝，然后调用 `handleShellCommand` 实现具体操作，但 AOSP 中存在大量服务类直接重写了 `onShellCommand` 而非 `handleShellCommand`，使得这个校验被跳过；
2.  即使服务没有使用 `handleShellCommand`，如果 `onShellCommand` 内正确检查了调用者权限或调用者 UID，那也是安全的；
3.  即使未直接检查调用者权限，如果 `ShellCommand` 内调用的都是会检查权限的公开接口（且没有 clear calling id），那也可以认为安全。

在这个漏洞中，部分操作未正确检查调用者权限，使得攻击者可以无权限调用系统内部私有接口造成漏洞。我发现的 CVE-2024-31318 就属于这种漏洞。这种漏洞近几年也不多见，了解一下就行。

#### [](#CVE-2021-0683：不正确的-ShellCommand-实现 "CVE-2021-0683：不正确的 ShellCommand 实现")CVE-2021-0683：不正确的 ShellCommand 实现

补丁链接：[Fix side effects of trace-ipc and dumpheap commands](https://android.googlesource.com/platform/frameworks/base/+/4241ab5ee435ee3c5e6496c001b2cf5bc827cfc4%5E%21/#F0)

本例中，`ActivityManagerShellCommand` 会在 `ActivityManagerService.onShellCommand` 中被调用，所以这里的代码运行在 system_server 进程，有着系统权限，直接调用 `file.delete()` 是以系统特权执行的，而很明显原本调用者不应该有删除系统文件的权限，造成漏洞。可能你会说，就算不删除文件，底下 system_server 一样要打开这个文件才能往里写入内容啊，这不是一样会覆盖原文件？请注意这里使用的是 `openFileForSystem`，这个函数实际上不会自己打开文件，而是使用调用者传递的 `ShellCallback`，让调用者进程打开文件然后返回文件描述符，所以实际上还是用调用者本身的权限。其他系统服务中，直接接受文件路径的接口（如 `pm install /path/to/apk`）也都应该这样实现，避免越权。

#### [](#CVE-2021-0327-amp-CVE-2021-0398：一行-clearCallingIdentity-引发的惨案 "CVE-2021-0327 & CVE-2021-0398：一行 clearCallingIdentity 引发的惨案")CVE-2021-0327 & CVE-2021-0398：一行 clearCallingIdentity 引发的惨案

CVE-2021-0327 补丁：[Ensure caller identity is restored in CP quick-path.](https://android.googlesource.com/platform/frameworks/base/+/69eaa90b0e4cc78fa2f518a50182bc9e4c9e87f3%5E%21/#F0)

CVE-2021-0398 补丁：[Set mAllowWhileInUsePermissionInFgs correctly when bindService() from background.](https://android.googlesource.com/platform/frameworks/base/+/86bd39db3595842bae77abe7e768226e412591c8)

还记得上面提到的 Binder IPC 鉴权过程吗？`enforceCallingOrSelfPermission` 等 API 实际都是依赖 `Binder.getCallingUid()` 返回的调用者 UID 进行鉴权的，而另一个 API `clearCallingIdentity` 允许让 `getCallingUid` 返回自己的 UID，以此绕过部分权限检查。有时这种行为是有意且安全的，而有时则能造成安全漏洞。Android 发展十几年，API 之间的调用关系错综复杂，如果有一点没考虑到，就有可能造成漏洞。要发现这种漏洞，可以使用[静态代码分析工具](https://github.com/michalbednarski/OrganizerTransaction?tab=readme-ov-file#bindergetcallinguid-that-always-returns-system-uid)，但仍然需要人工一个一个筛选可用项，还是体力活。

#### [](#CVE-2015-6623：鉴权，但是手滑写错了 "CVE-2015-6623：鉴权，但是手滑写错了")CVE-2015-6623：鉴权，但是手滑写错了

补丁：[Ask for system permission to enable ePNO](https://android.googlesource.com/platform/frameworks/opt/net/wifi/+/a15a2ee69156fa6fff09c0dd9b8182cb8fafde1c%5E%21/#F0)

太戏剧了一看就懂，应该不用多说……

除了纯粹手滑，这种漏洞比较多的出现在大版本迭代行为变更时，改了一个但是没改完的情况，如 CVE-2015-3833，Android 5.0 废弃 `getRecentsTask` 方法，只在调用者拥有 `android.permission.REAL_GET_TASKS` 特权时才返回其他进程信息，但是 `getRunningAppProcesses()` 没任何权限检查，调用这个接口可以变相达成 `getRecentsTask` 的效果。

#### [](#CVE-2020-0107-amp-CVE-2020-0246-amp-CVE-2023-21092：调用者伪造-package-name "CVE-2020-0107 & CVE-2020-0246 & CVE-2023-21092：调用者伪造 package name")CVE-2020-0107 & CVE-2020-0246 & CVE-2023-21092：调用者伪造 package name

CVE-2020-0107 补丁：[Check UID in getUiccCardsInfoSecurity](https://android.googlesource.com/platform/packages/services/Telephony/+/a39e6c1efb02ff9c19fb91beae9b548f5c1ecc78%5E%21/#F0)

CVE-2020-0246 补丁：[Add package checking with Uid in EuiccController#getEid](https://android.googlesource.com/platform/frameworks/opt/telephony/+/cfaf9f980aa8d3ca51cd8555ca27cd0ef561cb02%5E%21/#F0)

CVE-2023-21092 补丁：[Checking if package belongs to UID before registering broadcast receiver](https://android.googlesource.com/platform/frameworks/base/+/cdd30b5c040ba7ebd0a1cc6009183ff602434fc0%5E%21/#F0)

虽然大部分情况下鉴权只需要一条 `enforceCallingOrSelfPermission`，但有些时候仍然需要调用者的包名，而 Binder 默认只支持返回调用者的 UID 和 PID（PID 不可靠，调用者调用到一半被杀了，或者调用者指定 `FLAG_ONEWAY` 表示此次调用是异步调用时会是 0），因此遇上这种情况时，通常都是在参数中加一个 packageName 让调用者传递自己的包名。注意这里所谓的 packageName 是调用者主动传递所以完全受控于调用者，服务端必须检查传递的包名属于调用者 UID，而万一有一个地方粗心大意漏了，就有可能造成漏洞。实际上不只是 AOSP，OEM 代码中也出现过类似漏洞，如三星的 [SVE-2021-23076 (CVE-2021-25510, CVE-2021-25511)](https://wrlus.com/android-security/samsung/sve-2021-23076/)。

#### [](#CVE-2021-0319：验证调用者包名，但是没完全验 "CVE-2021-0319：验证调用者包名，但是没完全验")CVE-2021-0319：验证调用者包名，但是没完全验

补丁：[Fix CDM package check](https://android.googlesource.com/platform/frameworks/base/+/0c17049d39b5a8867f030f6f36433564140e124a%5E%21/#F0)

上回书说到，服务端必须保证调用者传入的包名属于调用者 UID，这种验证一般有三种方式，第一张是调用 `AppOpsManager.checkPackage()`，它在不匹配时会抛出 SecurityException；第二种是主动调用 `PackageManager.getPackageUid()` 与 calling uid 相比对，第三种是通过 `isSameApp`。

初看漏洞代码：

```
private void checkCallerIsSystemOr(String pkg, int userId) throws RemoteException {
    if (isCallerSystem()) {
        return;
    }
    checkArgument(getCallingUserId() == userId,
            "Must be called by either same user or system");
    mAppOpsManager.checkPackage(Binder.getCallingUid(), pkg);
}
```

用了上面提到的第一种方式，看起来好像并没有什么问题，对吧？我个人觉得即使是随便找一个安全研究员，跟他明确说这个文件里有校验包名不正确的漏洞，大概率也不会有人留意这里。

其实关键点在这里：

```
private IAppOpsService mAppOpsManager;
```

别被 `mAppOpsManager` 这个名字骗啦，它的类型不是 `android.app.AppOpsManager`，而是内部的 `com.android.internal.app.IAppOpsService`！虽然它们都有 `checkPackage()` 方法，但是前者在检查失败时会抛出异常，而后者只是返回错误而已！当初这段代码的作者应该也是觉得自己在用前者，结果实际上是后者，才造成漏洞。这种漏洞很少见，可以通过静态代码搜索找到所有对 `IAppOpsService.checkPackage()` 的调用然后一个个检查。

#### [](#鉴权虽好，可不要漏信息哦 "鉴权虽好，可不要漏信息哦")鉴权虽好，可不要漏信息哦

此类漏洞通常形式：攻击者传入其他应用的包名，然后通过微小的行为差异绕过 Android 11 中的 [“软件包可见性”（package visibility）](https://developer.android.com/training/package-visibility) 判断指定应用是否已经安装。

例子 1：[CVE-2021-0321](https://android.googlesource.com/platform/frameworks/base/+/b57d1409c52478d37f006145949be8b4591b9898%5E%21/#F0)，`getPackageUid` 在对应包名不存在时会返回 `Process.INVALID_UID`，因此满足下面的 if 直接返回，调用者拿到空列表；而对应包名存在时，会调用 `enforceCallingPermission(android.Manifest.permission.DUMP, function)`，调用者收到 SecurityException。这个微小的行为差异造成了信息泄漏。

例子 2：[CVE-2021-0975](https://cs.android.com/android/_/android/platform/frameworks/base/+/bf0d59726a0d9973f6867faedac0fe476c81fe8b)，包名存在时抛出的异常信息为 `"package " + packageName + " does not match caller's uid " + uid` 而不存在时是 `"package " + packageName + " not found"`，微小的信息差异造成信息泄漏。

这种漏洞非常非常多，我简单搜索了一下，仅仅 Android 14 一个大版本中就修复了至少 37 个类似的漏洞，已被修复的类似漏洞预计已达上百个，可以想象 AOSP 中还有多少。此种漏洞谷歌一般评级仅为 Moderate，获得的赏金上限仅为 $250 且没有 CVE 编号，不值得专门去找，适合 code review 时顺手提交上去赚赏金。

![](https://blog.canyie.top/data/image/android-platform-common-vulnerabilities/android-14-package-side-channel.jpg)

### [](#设备管理与多用户 "设备管理与多用户")设备管理与多用户

Android 从 4.2 开始加入了[多用户功能](https://source.android.com/docs/devices/admin/multi-user)，允许多个用户公用一台设备，每个用户可以安装各自的 app，数据互相隔离，一个用户通常情况下无法跨越用户边界读取或操作其他完全用户的数据（但管理员可以操作设备部分功能是否对其他用户开放）。跨用户操作通常需要 `INTERACT_ACROSS_USERS` `INTERACT_ACROSS_USERS_FULL` 等系统权限，如果有方法能绕过跨用户限制，就会被视为安全漏洞。部分关键系统应用可以[只在主用户运行](https://source.android.com/docs/devices/admin/multiuser-apps)，其他用户访问到的只是主用户的实例。

Android 同时支持[设备管理](https://source.android.com/docs/devices/admin)。系统层提供 [DevicePolicyManager](https://developer.android.com/reference/android/app/admin/DevicePolicyManager) 用于设备管理员控制设备，常用的有 [lock task mode](https://developer.android.com/work/dpc/dedicated-devices/lock-task-mode) 可用于锁定设备或限制设备只能运行某几个应用（专用设备等用途，如自动售货机）和 [User Restrictions](https://developer.android.com/reference/android/app/admin/DevicePolicyManager#addUserRestriction(android.content.ComponentName,%20java.lang.String)) 用于阻止用户操作特定设备功能，如禁用 WiFi、禁用蓝牙等。如果有方法绕过这些限制也会被视为安全漏洞。

#### [](#CVE-2019-2098-amp-CVE-2021-0686：API-缺失跨用户检查 "CVE-2019-2098 & CVE-2021-0686：API 缺失跨用户检查")CVE-2019-2098 & CVE-2021-0686：API 缺失跨用户检查

上文提到，为了支持多用户，很多 API 的参数里都加上了一个 userId，而跨用户操作需要系统权限，这需要服务端主动鉴权，API 这么多总会有一两个漏掉的。另一种形式是要求调用者传入 UID，但调用者可以传入属于其他用户的 UID。下面两个漏洞就是很经典的漏跨用户检查。

CVE-2019-2098 补丁：[Add cross user permission check - areNotificationsEnabledForPackage](https://android.googlesource.com/platform/frameworks/base/+/9162b8635bb6358be4f8fa2f6aa0188d6cddd641%5E%21/#F0)

CVE-2021-0686 补丁：[Add cross-user check for getDefaultSmsPackage().](https://android.googlesource.com/platform/frameworks/base/+/7f39ba09b8ccad2ae50874d3643cdc93746391ea)

#### [](#CVE-2023-21107：组件缺失跨用户 "CVE-2023-21107：组件缺失跨用户")CVE-2023-21107：组件缺失跨用户

漏洞补丁：[Enforce INTERACT_ACROSS_USERS_FULL permission for NotificationAccessDetails](https://android.googlesource.com/platform/packages/apps/Settings/+/179e5ce2a521710992b5ebdb2d88e0c3b3f2c12b%5E%21/#F0)

很多时候我们关注跨用户只关注系统提供的带有 userId 参数的 API，而忽略了系统应用。作为一个关键系统应用，系统设置拥有跨用户权限，此漏洞中的一个 Fragment 接收外部传递的 user handle 而没有任何输入校验和权限检查，然后直接开始使用这个 user handle，这样就使得攻击者可以利用一个低权限的恶意应用，打开 Settings 并传入其他用户的 handle，让高权限的 Settings 错误访问和操作其他用户的数据。这种 “低权限个体欺骗高权限个体执行特权操作” 的攻击模式被称为 confused deputy，虽然此例也可以说 missing permissions check 或者 missing input validation。  
事实上在编写这篇文章的过程中，我顺手搜了一下该漏洞中用到的 `Intent.EXTRA_USER_HANDLE`，然后就发现 Settings 中的另一个被称为 AppInfoBase 的 fragment 也存在一模一样的漏洞，喜提天上掉下来的高危漏洞 CVE-2024-43088。

#### [](#CVE-2023-21123：缺失-User-Restrictions-检查 "CVE-2023-21123：缺失 User Restrictions 检查")CVE-2023-21123：缺失 User Restrictions 检查

漏洞补丁：[Add DISALLOW_DEBUGGING_FEATURES check](https://android.googlesource.com/platform/packages/apps/Traceur/+/0cbd2103660d24abec9064f6a343151eb405b156)

`DISALLOW_DEBUGGING_FEATURES` 是一个 user restriction，可以由设备管理员设置以关闭调试相关的功能。Tracur 是一个系统内置的应用，和 trace 相关，而 trace 是调试功能。此例中，Tracur 没有检查 `DISALLOW_DEBUGGING_FEATURES`，虽然设置中开发者选项打不开，但是利用一个应用可以直接调起 Tracur 从而操作 trace。

个人观察到的是 Android 12-13 刚发布的时候出现了很多这种 restriction bypass 的漏洞，可能是 Android 12 中将系统设置、SystemUI 等应用程序重构为新的 Material You 设计风格时进行了较大的代码改动所导致。笔者也曾经提交过类似漏洞，只获得了 Moderate 评级，摸不清 Google 心情。

### [](#CVE-2021-0691：SELinux-权限配置不当 "CVE-2021-0691：SELinux 权限配置不当")CVE-2021-0691：SELinux 权限配置不当

上文说，Android 除了使用 UID 等 Linux 传统的 DAC 机制，还使用 SELinux 这一 MAC 机制进一步缩减进程权限，确保系统安全。而本节中提到的 CVE-2021-0691 就是 SELinux 权限配置过大导致。补丁链接：[system_app: remove adb data loader permissions](https://android.googlesource.com/platform/system/sepolicy/+/972b000898a21a9b9eb43d209246dc671b3d815b%5E%21/#F0)

可以看见，原本的策略允许 system_app 写入 apk_data_file 即已安装应用的 apk 文件，可以覆盖其他应用的 dex/so 等文件进而向其他应用持久注入代码。虽然攻击只能由系统的 system_app 发起，这满足了 [分级调节规则](https://source.android.com/docs/security/overview/updates-resources?hl=zh-cn#rating_modifiers) 中【需要作为特权上下文运行才能执行攻击】一条，严重程度被降为 moderate，但如果把这个漏洞和 system_app 中的其他漏洞如路径穿越写漏洞结合起来，就能组成一条非常有威力的漏洞链。事实上，此漏洞本身就是 “魔形女” 漏洞链的重要一环。想了解这一漏洞链可以参考[这篇文章](https://wrlus.com/android-security/android-11-mystique-exploit/)。历史上也出现过权限配置错误造成的严重程度更高的漏洞，通杀联发科设备的 MediaTek-SU 漏洞（CVE-2020-0069）就属于此类。

虽然为了安全，Android 的 SELinux 中存在大量 neverallow 规则保证 OEM 不会添加太过离谱的规则，修改 neverallow 会导致无法通过 CTS，但我们确实发现有一些 OEM 确实允许了被 neverallow 的条目。只能对 OEM 质量一声叹息。

[](#组件与意图 "组件与意图")组件与意图
-----------------------

每个 Android 程序员都知道的 Android 四大组件：Activity Service BroadcastReceiver ContentProvider。Intent 则是与组件交互的桥梁。有时，不经意间的不当使用，也许就会造成漏洞。

### [](#CVE-2021-0693：组件权限配置不当 "CVE-2021-0693：组件权限配置不当")CVE-2021-0693：组件权限配置不当

漏洞补丁：[Don’t export HeapDumpProvider.](https://android.googlesource.com/platform/frameworks/base/+/e9a6ebf59258a4ca14f83b74f10113aaddaf2b33%5E%21/#F0)

组件的 exported 属性表示该组件是否可被外部应用访问，若没有设置则有 intent-filter 的组件默认导出（target Android 12+ 的 app 如果出现这种情况会被直接拒绝安装）。即使导出了组件，也可以设置 `android:permission` 等属性来限制只有拥有对应权限的应用才能访问此组件。本例中受害 provider 设置 exported=true 即导出，同时没有设置访问权限，任何应用都能直接访问，而访问 HeapDumpProvider 能获取到调试用的其他 app 的 heap dump，显然是敏感信息，从而造成非常明显的安全漏洞。

这种漏洞前几年比较多，早期甚至出现过把 `android:exported="true"` 误写成 `exported="true"` 这种漏洞（CVE-2013-6272），随着系统的日益完善成熟，近几年观察到的少了。后续挖掘重点可以集中在 OEM 自定义的组件上。

### [](#LaunchAnywhere：危险的-Intent-Redirection "LaunchAnywhere：危险的 Intent Redirection")LaunchAnywhere：危险的 Intent Redirection

大名鼎鼎的 LaunchAnywhere 应该是最早也最经典的 Intent Redirection 类型漏洞，可惜的是年代久远没有找到 CVE 编号，只有一个 Bug ID A-7699048。

这个洞网上的详细分析已经有很多了，这里不赘述，只简单介绍一下它的基本原理：应用 A 通过 AccountManager API 添加账号时，AccountManagerService 会请求 account 的 authenticator（也是一个应用），而这个 authenticator 可能需要向用户显示一个界面，所以可能返回一个 intent，此时 AccountManagerService 把 intent 返回给 A，在 A 中运行的系统代码会直接调用 startActivity 启动这个 intent，这里用的是 A 的身份，从而可以无限制访问应用内部 Activity 组件。如果 A 是 Settings，它拥有系统权限，此时就可以访问所有应用的所有 Activity，无视它是否导出或是否被权限保护。

Intent Redirection 是 Android 平台最经典的漏洞类型，已在 Android 系统、定制系统、三方应用程序等中多次出现过。其基本特征为收到别人发送的 intent，随后将其转发出去，常见 API 有 `startActivity`、`sendBroadcast`、`setResult` 等。前两个很明显，可以访问内部组件，而 `setResult` 虽然不能直接访问内部组件，但攻击者可以通过指定 intent 的 data 为应用内部受保护的 URI，并在 intent flags 中指定 `FLAG_GRANT_READ/WRITE_URI_PERMISSION`，当受害应用使用恶意 intent 调用 `setResult()` 时，就会不知不觉授予恶意应用读写自身内部 content provider 的权限。

在 Android 12 中，为了规避这类问题，Strict mode 引入了新的功能，可以检测到应用收到（getParcelable）别人发来的 intent 并将其立即转发的情况，一定程度上帮助了开发者识别此类漏洞。但是，这个工具并不能扫描到所有危险，而且直接转发 intent 并不是 intent redirection 的唯一一种类型，CVE-2022-20550 & CVE-2024-0015 就是一个例子，特权应用接收不可信的 ComponentName 然后直接创建指向该 ComponentName 的 intent 并 startActivity，也能够越权访问组件。补丁在这里：[Fix vulnerability that allowed attackers to start arbitary activities](https://android.googlesource.com/platform/frameworks/base/+/2ce1b7fd37273ea19fbbb6daeeaa6212357b9a70%5E%21/#F0)。

### [](#CVE-2014-8609（BroadcastAnywhere）：危险的-Pending-Intent "CVE-2014-8609（BroadcastAnywhere）：危险的 Pending Intent")CVE-2014-8609（BroadcastAnywhere）：危险的 Pending Intent

上文提到的 Intent 其实还漏了一种特殊形式，即 PendingIntent。简单来说，PendingIntent 由应用主动创建，代表某项特殊的操作，可以传送给别的应用，别的应用发送这个 PendingIntent 时就以 PendingIntent 创建者的身份发送。创建时可以指定 PendingIntent 是否可变，如果可变则允许发送者修改未被显式指定的 intent 字段，如 action、data、flags 等。为了安全起见，默认 selector 和 ComponentName 是不允许被修改的。

回过头来看漏洞补丁：[SECURITY: Don’t pass a usable Pending Intent to 3rd parties.](https://android.googlesource.com/platform/packages/apps/Settings/+/f5d3e74ecc2b973941d8adbe40c6b23094b5abb7%5E%21/#F0)

很明显，这里的 `mPendingIntent` 创建时使用了一个空的、啥都没有的 intent，同时未指定 `FLAG_IMMUTABLE`（虽然实际原因是这玩意在 Android 6.0 才加，那个年代还没有这玩意），这使得攻击者拿到这个 PendingIntent 之后可以任意改写里面的 intent 再发送，同时由于这个 PendingIntent 是 Settings 创建的，具有系统权限，攻击者发送 PendingIntent 时会以创建者身份发送，也就同样是以系统权限。这里 PendingIntent 是用的 `getBroadcast`，最后 `send` 的时候也会以广播方式发送，比如攻击者改写 action 字段为 `android.intent.action.MASTER_CLEAR`，广播发送出去后就会触发 `MasterClearReceiver` 进行恢复出厂设置的操作。

事实上，就算是开发者记得填充 action，有时候也不能避免漏洞的出现。恶意应用可以在自己的 AndroidManifest.xml 中注册相同 action 的 intent-filter，改写 package 指向恶意应用自己，flags 添加授权 flags，data/clipdata uri 指向受害应用私有的或者可访问的 ContentProvider 并发送，此时恶意应用就会被授予权限。更多可以查看 OPPO 的这篇文章：[PendingIntent 重定向：一种针对安卓系统和流行 App 的通用提权方法——BlackHat EU 2021 议题详解 （下）](https://mp.weixin.qq.com/s/ifrErL88_8wN36WT8QbLvg)

（注：为了安全考虑，如果受害者是 root/system UID，授权时要求 URI 必须是几个特定 authority 否则会被拒绝，但对于系统中及应用市场中的海量应用，它们仍可能被攻击）

为了保证安全性，Android 6.0 添加了 `FLAG_IMMUTABLE` 用于指定 PendingIntent 不可变，而 Android 12 添加了 `FLAG_MUTABLE`，target Android 12+ 的应用创建 PendingIntent 时必须显式指定可变性，不再让应用开发者随手写下的 0 变成安全漏洞，自此这一类漏洞销声匿迹。但有部分场景 PendingIntent 仍然是可变的，此时就要万分小心。

### [](#CVE-2017-0639：危险的-URI "CVE-2017-0639：危险的 URI")CVE-2017-0639：危险的 URI

URI 在 Android 中非常常用，如常见的分享文件操作，就是创建一个 ACTION_SEND 的 intent，把要分享的文件的 URI 放到里面，然后 startActvity。常见的有安全影响的 URI 有 content uri 和 file uri（Android 7.0+ 已弃用）等。

这里其实有一点要注意的，就是接收到的 URI 可能指向原本发送者不应该有权限访问的数据，如果受害应用不加检查直接使用，就有可能造成信息泄漏。本例的 CVE-2017-0639 就是这样一个漏洞，漏洞补丁：[OPP: Restrict file based URI access to external storage](https://android.googlesource.com/platform/packages/apps/Bluetooth/+/f196061addcc56878078e5684f2029ddbf7055ff%5E%21/#F0)

作者写的[文章](https://xz.aliyun.com/t/143?time__1311=CqjhqGxmx5PBqZD2mDB7%3DPD%3DAZWD&alichlgref=https%3A%2F%2Fxz.aliyun.com%2Fu%2F593)也可以看一下。这里提一个点就是，禁止向外传出 file:// URI 这个功能只是 Strict mode 的限制，只需要把 strict mode 关闭就好。

file:// URI 从 Android 7.0 开始被弃用，像这样直接构造指向私有文件的 file URI 的这种利用手法近几年应该是几乎绝迹，但像这样缺失 URI 检查的漏洞其实还有很多，最近比较多的是高权限应用程序接收 URI 时没有检查该 URI 是否指向其他用户，从而允许一个用户越权读取到另一个用户的媒体。content URI 正常格式为 `content://authority/path/id`，跨用户的 content URI 会在 authority 前面加上用户 ID，形如 `content://10@authority/path/id`。

### [](#广播权限与保护广播 "广播权限与保护广播")广播权限与保护广播

上文中说到四大组件都可以定义权限，限制只能有对应权限的个体访问，但其实对于广播来说，它还有额外的保护机制。

我们常用的 `BOOT_COMPLETED` 等广播都是只有系统能发送，如果应用发送了这种广播，会发生什么？答案是会被系统拒绝发送。这一系列的广播对系统极为重要，如果第三方应用可以随意发送，比如发送 `android.intent.action.REBOOT`，系统就重启了；发送 `android.intent.action.MASTER_CLEAR`，系统就会开始恢复出厂设置。当时为了解决这个问题，一种方法是限制所有的系统广播接收器必须加上权限，只有拥有系统权限才能访问，但是难搞的是，这些广播中有一些是粘性广播（sticky broadcasts），它在系统中会长时间存留，直到后一个覆盖前一个，这样应用仍然有机会覆盖粘性广播的数据；另一个选择是，给这些广播都定义上权限，只有有权限的个体才能发送，但是给这些广播一个一个定义专用权限显然不现实，于是保护广播便诞生了，还是限制只有系统能发送，但是没有像传统一样定义特别的权限。很显然，这么多危险的广播，如果有一个被漏掉忘记被保护，或者定义保护广播没有生效，就会出现严重的安全漏洞。事实上，历史上确实发生过类似的漏洞：[CVE-2017-0601](https://xz.aliyun.com/t/177?time__1311=CqjhqGxmxAxfODBqPx2mDBDRg%2BLm3QKx&alichlgref=https%3A%2F%2Fxz.aliyun.com%2Fu%2F593) 和 [CVE-2020-0391](https://wrlus.com/android-security/system-apps-and-cve-2020-0391/)。

另外一点，系统内部也大量应用了广播机制传递信息，广播这玩意和 Activity Service Provider 都不同，一次广播可能被多个个体接收，如果说系统内部发送广播传递内部信息的时候没有指定接收者，那就有可能被第三方应用拿到敏感信息。但是，我们不可能把所有已知会接收这个广播的可信应用硬编码在代码里，由此就引入了另一个安全机制，不仅广播接收者可以指定要有某某权限才能给我发广播，发送者也可以指定接收者必须有某某权限才能收到广播，利用现有的权限机制解决了问题。这里可能出现的漏洞就是写代码的人粗心大意忘记指定权限，导致信息被其他不相关应用接收。

（题外话，其实隐式 intent 启动 Activity 也有可能出现类似问题，三方应用定义一个相同的 intent-filter 就有机会被系统启动，不过这种漏洞偏少，启动 activity 只会启动一个，可以通过在 intent-filter 配置大于 0 的优先级来解决。不过也不是没有，[CVE-2020-0396](https://wrlus.com/android-security/cve-2020-0396/)）。

### [](#Parcel-Mismatch "Parcel Mismatch")Parcel Mismatch

其实这一段不应该放在组件与意图的，但是真的不知道该放哪。

除了传统的 Java 序列化方法，android 还为跨进程通信专门设计了一套轻量级的序列化方案：Parcelable。和 Serializable 不同，使用 Parcelable 要求开发者手动往 parcel 写入或读取数据。以下展示了一个典型的 Parcelable 实现类：

```
public class Person implements Parcelable {
    private String name;
    private int age;
    private int money;
    public static final Creator<Person> CREATOR = new Creator<Person>() {
        @Override public Person createFromParcel(Parcel in) {
            Person obj = new Person();
            obj.name = in.readString();
            obj.age = in.readInt();
            obj.money = in.readInt();
            return obj;
        }

        @Override public Person[] newArray(int size) {
            return new Person[size];
        }
    };

    @Override public int describeContents() {
        return 0;
    }

    @Override public void writeToParcel(@NonNull Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(age);
        dest.writeInt(money);
    }
}
```

假如这里程序员手滑了，write 写入的数据和 read 读取的数据不匹配，比如说一边写了 long 另一半读了 int 或者忘记读某个字段，会发生什么问题吗？

查看以上代码，两次 readInt 调用同一个方法能读取到不同的数据，而且读取顺序和写入顺序完全相同，这说明 Parcel 内部必定存在着一个偏移值，每次 read 读取当前偏移的数据然后自增偏移。假如 read 没有完全消费它写入的所有值，那么 parcel 中残留的值可能会影响后续值的解析。

举个例子，假如有如下 aidl 函数定义：

```
void registerPerson(Person person, int flags);
```

假如 Person 中最后一行 `obj.money = in.readInt()` 缺失了，那么 Person 对象反序列化完之后，还会有一个 int 残留在 parcel 内，接下来尝试读取 flags 调用 readInt 时实际上读取到的是 Person 残留的 money 而不是正确的 flags。

再回过头来看一下之前 LaunchAnywhere 漏洞，当时的修复补丁是，authenticator 返回 intent 后 AccountManagerService 检查 intent 要调起的 activity 的包的签名是否与 authenticator 自身匹配，匹配才发送给调用者让它打开。这里有一个点，authenticator 返回的数据类型实际上是 Bundle，intent 也是放进 bundle 里存储的。Bundle 这里可以简单理解为一个 map，存储着键值对，里面值的类型除了可以是基本类型和 String，还可以是任意 Parcelable 对象。

这两点结合起来，会碰撞出怎样的火花呢？注意上面检查 bundle 是在 AccountManagerService 中，它在 system_server 中运行，而实际调起 activity 的操作在调用者进程中，所以这里 bundle 从 system_server 到 app 还要经过一次序列化和反序列化。我们已经知道 bundle 是一个 map 且可以存储任意 Parcelable 对象，那假如我们在 bundle 中任意放一个反序列化错位的对象，就有机会污染它后面的键值对，通过精心构造内存布局，我们甚至能让 AccountManagerService 检查时找不到 intent，而经过一次序列化和反序列化传输到 app 后却能找到 intent，绕过 LaunchAnywhere 漏洞的修复！由于 Bundle 接受任意 Parcelable，所以实际上任何有问题的 Parcelable 都能拿来这样利用！这种技术有一个特别的名字，叫 Self-changing Bundle。2023 年国内某电商软件大肆在野利用的其中一个漏洞 [CVE-2023-20963](https://android.googlesource.com/platform/frameworks/base/+/266b3bddcf14d448c0972db64b42950f76c759e3%5E%21/#F0) 就属于此类漏洞。获得了以系统权限启动任意 Activity 的能力后，可以利用系统内部一些 activity 的 intent redirection，设置 data 和 flags 拿到内部 content provider 如一些 file provider 的读写权限，可以越权读写系统关键文件，改写系统配置；另一种利用方案是攻击其他应用未导出的组件，读写应用内部数据甚至向其他应用注入恶意代码。

由于 Google 已经意识到这类漏洞的强大破坏力，在 Android 13 中引入了多个机制缓解此类漏洞，如 [Lazy Bundle](https://cs.android.com/android/_/android/platform/frameworks/base/+/9ca6a5e21a1987fd3800a899c1384b22d23b6dee) 和 [Parcel.enforceNoDataAvail()](https://developer.android.com/reference/android/os/Parcel#enforceNoDataAvail())。另外 Google 在 AccountManagerService 引入了一个新的 `checkKeyIntentParceledCorrectly` 函数，校验 bundle 中的 intent 反序列化前后是否一致，并把该补丁一路向下 backport 到了 Android 11，算是基本封死了这类利用 bundle 的方法。

更多关于这类漏洞的细节，可以参考以下文章：

*   [Android 反序列化漏洞攻防史话](https://evilpan.com/2023/02/18/parcel-bugs/)
*   [Creator Mismatch](https://konata.github.io/posts/creator-mismatch/)
*   [Clang 裁缝店：LaunchAnyWhere 补丁绕过：Android Bundle Mismatch 系列漏洞 复现与分析](https://xuanxuanblingbling.github.io/ctf/android/2024/04/13/launchanywhere02/)

同时强烈建议查看 [Michał Bednarski 的 GitHub 主页](https://github.com/michalbednarski)，大部分都是 parcel mismatch 类漏洞且每一个都有很详细的 writeup。

### [](#组件启动的后台限制 "组件启动的后台限制")组件启动的后台限制

Android 系统对在后台运行的应用施加严格限制，如果应用有办法绕过这些限制，就可能被视为安全漏洞。本节我们主要关注以下限制：

*   [从后台启动 activity 的限制](https://developer.android.com/guide/components/activities/background-starts)
*   [从后台启动前台服务的限制](https://developer.android.com/develop/background-work/services/foreground-services#bg-access-restrictions)

同时我们定义以下缩写：

*   BAL：Background activity launch，绕过上面的第一个限制
*   BG-FGS：Background-Foreground Service start，绕过上面的第二个限制
*   WIU：while-in-use 权限，简称前台权限，指用户仅允许应用 “在使用中才能获得” 的权限

恶意应用利用 BAL 漏洞可以在后台随意弹出广告，严重影响用户对手机的正常使用。能够绕过 BAL 限制的漏洞一般至少都会被授予 High 等级，而虽然绕过 BG-FGS 限制并不被直接视为是安全漏洞，但如果能从后台获取 WIU 权限，仍然会被认为是高危。部分此类漏洞具有相似的模式，从近几个月的安全广告来看也确实每隔一两个月都有类似漏洞出现。本节我们主要介绍利用 PendingIntent 实现 BAL。

以 Android 14 为例，系统判断 BAL 是否被允许的逻辑在 [BackgroundActivityStartController.checkBackgroundActivityStart()](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r28:frameworks/base/services/core/java/com/android/server/wm/BackgroundActivityStartController.java;l=193) 里，判断是否能启动前台服务的逻辑则在 [ActiveServices.canStartForegroundServiceLocked()](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r28:frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;l=7494)。

以 BAL 为例，我们这里就不分析完整判断逻辑了，实在是太长了，详细分析绝对不是这里能写完的。我们主要关注这一小段：

```
final BackgroundStartPrivileges balAllowedByPiSender =
        PendingIntentRecord.getBackgroundStartPrivilegesAllowedByCaller(
                checkedOptions, realCallingUid, realCallingPackage);

final boolean logVerdictChangeByPiDefaultChange = checkedOptions == null
        || checkedOptions.getPendingIntentBackgroundActivityStartMode()
                == ComponentOptions.MODE_BACKGROUND_ACTIVITY_START_SYSTEM_DEFINED;
final boolean considerPiRules = logVerdictChangeByPiDefaultChange
        || balAllowedByPiSender.allowsBackgroundActivityStarts();
final String verdictLogForPiSender =
        balAllowedByPiSender.allowsBackgroundActivityStarts() ? VERDICT_ALLOWED
                : VERDICT_WOULD_BE_ALLOWED_IF_SENDER_GRANTS_BAL;

@BalCode int resultIfPiSenderAllowsBal = BAL_BLOCK;
if (realCallingUid != callingUid && considerPiRules) {
    resultIfPiSenderAllowsBal = checkPiBackgroundActivityStart(callingUid, realCallingUid,
        backgroundStartPrivileges, intent, checkedOptions,
        realCallingUidHasAnyVisibleWindow, isRealCallingUidPersistentSystemProcess,
        verdictLogForPiSender);
}
if (resultIfPiSenderAllowsBal != BAL_BLOCK
        && balAllowedByPiSender.allowsBackgroundActivityStarts()
        && !logVerdictChangeByPiDefaultChange) {
    
    
    return resultIfPiSenderAllowsBal;
}
```

如果调用者实际上不是自己 startActivity，而是发送了由其他 UID 创建的 PendingIntent （或 IntentSender，实际上是 PendingIntent 内部实现），则 `realCallingUid != callingUid` 会成立，然后会调用 `checkPiBackgroundActivityStart`：

```
private @BalCode int checkPiBackgroundActivityStart(int callingUid, int realCallingUid,
        BackgroundStartPrivileges backgroundStartPrivileges, Intent intent,
        ActivityOptions checkedOptions, boolean realCallingUidHasAnyVisibleWindow,
        boolean isRealCallingUidPersistentSystemProcess, String verdictLog) {
    final boolean useCallerPermission =
            PendingIntentRecord.isPendingIntentBalAllowedByPermission(checkedOptions);
    if (useCallerPermission
            && ActivityManager.checkComponentPermission(
                            android.Manifest.permission.START_ACTIVITIES_FROM_BACKGROUND,
            realCallingUid, -1, true) == PackageManager.PERMISSION_GRANTED) {
        return logStartAllowedAndReturnCode(BAL_ALLOW_PENDING_INTENT,
             false, callingUid, realCallingUid, intent,
            "realCallingUid has BAL permission. realCallingUid: " + realCallingUid,
            verdictLog);
    }

    
    
    if (realCallingUidHasAnyVisibleWindow) {
        return logStartAllowedAndReturnCode(BAL_ALLOW_PENDING_INTENT,
                 false, callingUid, realCallingUid, intent,
                "realCallingUid has visible (non-toast) window. realCallingUid: "
                        + realCallingUid, verdictLog);
    }
    
    
    if (isRealCallingUidPersistentSystemProcess
            && backgroundStartPrivileges.allowsBackgroundActivityStarts()) {
        return logStartAllowedAndReturnCode(BAL_ALLOW_PENDING_INTENT,
                 false, callingUid, realCallingUid, intent,
                "realCallingUid is persistent system process AND intent "
                        + "sender allowed (allowBackgroundActivityStart = true). "
                        + "realCallingUid: " + realCallingUid, verdictLog);
    }
    
    if (mService.isAssociatedCompanionApp(
            UserHandle.getUserId(realCallingUid), realCallingUid)) {
        return logStartAllowedAndReturnCode(BAL_ALLOW_PENDING_INTENT,
                 false, callingUid, realCallingUid, intent,
                "realCallingUid is a companion app. "
                        + "realCallingUid: " + realCallingUid, verdictLog);
    }
    return BAL_BLOCK;
}
```

如果 `realCallingUid` 即 PendingIntent/IntentSender 的发送者拥有可见窗体，或者是需要持续运行的系统重要进程，那么就有机会被允许。常见的是很多 OEM 在 system uid 的进程中实现手势导航等自定义功能，导致系统认为 system uid 具有可见窗口，暴露攻击面。而系统中有很多接收 PendingIntent/IntentSender 的接口，一般用于异步操作完成后向调用者发送返回值，如果忘记在 options 内指定参数禁止 BAL，那就有可能被我们利用。

以 [CVE-2023-21081 & CVE-2023-21099](https://android.googlesource.com/platform/frameworks/base/+/6aba151873bfae198ef9eceb10f943e18b52d58c%5E%21/#F0) 为例，系统中 PackageManager 多个 API 接受一个 IntentSender 作为回调接口，而发送该 IntentSender 时没有指定 options 导致其为默认的 null。解决方法就是加一个 options 并且 `setPendingIntentBackgroundActivityLaunchAllowed(false)`。

这里有一点，Android 14 中对 BAL 限制做出了一些增强，其中 “应用会收到来自其他可见应用发送的 PendingIntent” 一项中加了一个对我们很重要的限制：

> Note: Starting from Android 14, apps targeting Android 14 or higher must opt in to allow background activity launch when sending a PendingIntent. To opt in, the app should pass an ActivityOptions bundle with setPendingIntentBackgroundActivityStartMode(ActivityOptions.MODE_BACKGROUND_ACTIVITY_START_ALLOWED)

[getDefaultBackgroundStartPrivileges](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r28:frameworks/base/services/core/java/com/android/server/am/PendingIntentRecord.java;l=395) 在 options 为空时返回的默认值也有更改，可能不允许 BAL：

```
public static BackgroundStartPrivileges getDefaultBackgroundStartPrivileges(
        int callingUid, @Nullable String callingPackage) {
    if (UserHandle.getAppId(callingUid) == Process.SYSTEM_UID) {
        
        
        
        
        
        return BackgroundStartPrivileges.ALLOW_BAL;
    }
    boolean isChangeEnabledForApp = callingPackage != null ? CompatChanges.isChangeEnabled(
            DEFAULT_RESCIND_BAL_PRIVILEGES_FROM_PENDING_INTENT_SENDER, callingPackage,
            UserHandle.getUserHandleForUid(callingUid)) : CompatChanges.isChangeEnabled(
            DEFAULT_RESCIND_BAL_PRIVILEGES_FROM_PENDING_INTENT_SENDER, callingUid);
    if (isChangeEnabledForApp) {
        return BackgroundStartPrivileges.ALLOW_FGS;
    } else {
        return BackgroundStartPrivileges.ALLOW_BAL;
    }
}
```

再加上大部分这种漏洞都已经被挖完了，这种漏洞在 2024 年之后基本在 AOSP 中绝迹。

想要了解更多的话，可以查看 OPPO 的这篇文章：[恶意 App 后台弹窗技术手法分析](https://mp.weixin.qq.com/s/hGx8FNjI37trnZehY839bQ)

[](#骗！偷袭！不讲武德！ "骗！偷袭！不讲武德！")骗！偷袭！不讲武德！
--------------------------------------

上面介绍了很多技术漏洞，这一节从人出发，介绍几个原理很简单朴素的漏洞。

### [](#CVE-2018-9432：一个小小的字符串能有什么坏心思呢？ "CVE-2018-9432：一个小小的字符串能有什么坏心思呢？")CVE-2018-9432：一个小小的字符串能有什么坏心思呢？

假如说，你是系统开发者，你自定义了一个特权操作，比如从用户绑定的钱包里划走一百块，同时允许应用调用你定义的接口请求这个操作，那肯定得需要得到用户明确允许才行。一般的做法是设计一个对话框，如下所示，其中 `<appname>` 代表发起请求的应用名称：

```
<appname> 正在请求消费 100 元人民币，此笔款项将会从您的钱包中扣除。您确认要支付吗？

取消    确认
```

而如果我们将应用名设置的特别长，会怎么样呢？我们精心设置一个超长的应用名，它会这样显示：

```
<XX 应用需要您同意隐私协议。
XX 应用隐私协议：
1. 本应用由 xxx 公司开发。
2. 本应用在运行过程中，需要以下权限运行：
(1). 网络权限。本应用需要此权限以连接服务器、向服务器发送数据并拉取数据。您必须授予此权限。
(2). 存储卡权限。本应用部分功能需要读写您的照片、笔记等内容，因此需要存储卡权限。我们不会滥用通过此权限获得的数据，也不会将其持久化存储或上传到网络。我们只在您使用特定功能时请求此权限，您也可以随时拒绝或撤销此权限。不授予此权限不会对其他功能产生影响。
.......


如您同意此协议，请点击“确认”继续运行。如您拒绝此协议，本应用无法运行并将自动退出。




> 正在请求消费 100 元人民币，此笔款项将会从您的钱包中扣除。您确认要支付吗？

取消    确认
```

这里用 <> 括起来的一大段其实都是应用名。而受限于屏幕大小限制，对话框显示的时候只能显示前面的应用名，真正的 “正在请求消费” 信息被挤下去了。如果用户没有注意到对话框文本可以滑动（事实上就算注意到了，估计也会认为剩下的全是又臭又长的协议），直接点了确认，就会不知不觉间损失财产。

实际生活中的支付对话框当然不可能设计的这么简单，但历史上确实出现过此类漏洞，如 CVE-2015-3878 屏幕录制授权欺骗漏洞、CVE-2017-13242 蓝牙配对欺骗漏洞、CVE-2018-9432: 蓝牙通讯录访问欺骗漏洞等。感兴趣的可以查看这篇文章：[Android 中的特殊攻击面（一）——邪恶的对话框](https://mp.weixin.qq.com/s/mN5M9-P0g6x_4NqTKbO2Sg)

高版本 Android 系统对话框做了一些更改，如权限请求对话框的 “允许”“拒绝” 选项现在和对话框的内容放在一起，用户必须完全滚动到最下方才能点到按钮，算是基本杜绝了此类漏洞。

### [](#CVE-2021-0314：UI-覆盖与点按劫持 "CVE-2021-0314：UI 覆盖与点按劫持")CVE-2021-0314：UI 覆盖与点按劫持

Android 有一个功能，允许应用在其他应用之上显示内容，通常简称为 “悬浮窗”。早在 Android 4.x 时代，恶意开发者就已经滥用这个权限制作恶意软件，如在设备开机时显示全屏悬浮窗阻止用户使用手机，从而勒索钱财，即所谓的 “锁机软件”。通过这个功能，还可以实现很多有意义的攻击。还是以上面的对话框为例，这次我们不使用超长应用名称，而是利用悬浮窗功能，在对话框显示的同时覆盖我们自己的内容到对话框的文本上面，用户很容易就会被误导点击确认。

这种类型的攻击叫做点按劫持攻击（Tapjacking Attack）。对大部分情况来说，能够被其他应用覆盖[并不是安全问题](https://bughunters.google.com/learn/invalid-reports/android-platform/5148417640366080/bugs-with-negligible-security-impact#tapjacking-overlay-system_alert_window-vulnerability-on-a-non-security-critical-screen)，而对于敏感对话框，可以通过申请 `HIDE_NON_SYSTEM_OVERLAY_WINDOWS` 权限并对当前 window 添加 `SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS` 这个属性或调用 [setHideOverlayWindows](https://developer.android.com/reference/android/view/Window#setHideOverlayWindows(boolean)) 来隐藏所有的非系统悬浮窗，还可以通过 `filterTouchesWhenObscured` `onFilterTouchEventForSecurity` 等 API 过滤可能被劫持的输入事件。但实际上，Android 系统自身也出现过多个忘记对敏感对话框使用防御措施的漏洞，如此例的 CVE-2021-0314 便是卸载应用的确认对话框。

想要了解更多，可以查看以下文章：

*   [Google 官方对点按劫持的介绍](https://developer.android.com/privacy-and-security/risks/tapjacking)
*   [不可忽视的威胁：Android 中的点击劫持攻击](https://mp.weixin.qq.com/s/04pcy3V3mhyru4YCwiGJLg)

[](#宁为玉碎-——-拒绝服务类攻击 "宁为玉碎 —— 拒绝服务类攻击")宁为玉碎 —— 拒绝服务类攻击
-----------------------------------------------------

Android 的漏洞分类中，还有一种特殊的漏洞，既不能像影视剧里的黑客一样敲敲键盘入侵敌国核弹系统，也不能泄漏别人的银行卡密码，它能做的只有使手机工作出现异常。它就是拒绝服务类漏洞。虽然不像 RCE、EoP、ID 那么亮眼，但也不能小看这类一不小心就让你手机变砖头的漏洞。在 Android 的漏洞严重程度定义中，有如下内容：

> 严重（Critical）：设备遭到远程发起的持久性拒绝服务攻击（永久性损坏、需要重新刷写整个操作系统或恢复出厂设置）  
> 高（High）：设备遭到本地发起的持久性拒绝服务攻击（永久性损坏、需要重新刷写整个操作系统或恢复出厂设置）；攻击者可以在没有用户互动的情况下远程阻止对移动网络或 Wi-Fi 服务的访问（例如，用格式不正确的数据包使移动网络无线装置服务崩溃）  
> 中（Moderate）：设备遭到远程发起的设备暂时性拒绝服务攻击（远程挂起或重新启动设备）

能达到 “严重” 程度的 DoS 漏洞很少见，看见过的几个都是 TextView 文字渲染的崩溃或者死循环。我们主要瞄准严重程度 “高” 的漏洞。

想要 “持久性拒绝服务攻击”，比如让手机系统崩溃开不了机无限重启，除了利用系统本身的缺陷，很容易能想到的还有传统 DoS 中的 “资源耗尽”，简单来说，占用系统大量资源使其停止工作。但是，让 system_server 崩溃一次最多只会造成系统软重启一次，并不算持久。怎么样才能持久呢？

“持久”，这两个字能描述的东西，还有数据。如果能够将恶意的数据保存下来，系统每次启动尝试去读取它的时候就都会崩溃，陷入死局。而所谓 “能让系统崩溃的恶意数据” 除了精心构造的、利用代码本身问题的数据，很容易能想到的还有超大量的数据，在系统处理的过程中耗尽系统内存资源触发崩溃。以 [CVE-2021-0934](https://android.googlesource.com/platform/frameworks/base/+/cf62c760d4c002f562ddd5f372abe5bccda8a6ad%5E%21/#F0) 为例，Account 就是要被持久化存储的数据，虽然已经考虑到系统资源负担，对 Account 内字符串的大小及 Account 数量做出限制，然而字符串大小限制在客户端，能被绕过。这里注意，binder 一次能传输数据的大小也是有限制的，大概在 1mb 左右，所以还不能一次传太大，只能一个一个传。像这样的漏洞还有很多，系统开发者设计接口时一不小心忘记加上限制就有可能变成一个 CVE，如 [CVE-2022-20494](https://github.com/Supersonic/CVE-2022-20494)。这种一点一点增大系统资源负载的攻击很像成语 “压死骆驼的最后一根稻草” 的故事，因此也被称为稻草攻击（Straw Attack）。复旦大学有一篇论文[《Exploit the Last Straw That Breaks Android Systems》](https://www.computer.org/csdl/proceedings-article/sp/2022/131600a542/1FlQCuVjqIU)，发表在 IEEE Symposium on Security and Privacy. 2022 上，专门介绍这类攻击手法，感兴趣的可以看一下。英文不太好的同学也可以看看这篇译文[《稻草攻击：压死安卓系统的最后一根稻草》](https://www.inforsec.org/wp/?p=5909)。~（这算是把中文翻译成英文又翻译成中文吗？）~

[](#扩展阅读 "扩展阅读")扩展阅读
--------------------

上面的文章里放了很多文章，这里再放一些同样很优秀的扩展类文章。强烈建议阅读 OPPO 几个微信公众号发布的系列文章。

*   [小路：Android 系统安全和黑灰产对抗](https://wrlus.com/android-security-learning/android-system-security-intro/)
*   [OPPO 安珀实验室：Android Java Framework 漏洞挖掘快速上手方法](https://mp.weixin.qq.com/s/_qEG6NGa78RP80eeg62Bwg)
*   [OPPO 安珀实验室：Android 本地服务漏洞挖掘技术](https://mp.weixin.qq.com/s/skfpZZiEGj4MNFzw2UGGGg) 这里只放了上篇的链接，剩下两篇自己去微信公众号看吧……

[](#结语 "结语")结语
--------------

本文介绍了大量 Android 系统中的漏洞，并尝试总结出一些常见类型，希望能给对 Android 系统安全感兴趣的人一点帮助。当然，如果你想成为一名专业的 Android 安全研究员，传统二进制漏洞如 buffer overflow、use-after-free 等也肯定是需要了解的。我个人觉得，入门学习 Android 安全最好的时间是 2023 年之前。在 2023 年之前，只要附上一份高质量的漏洞描述，单个 moderate 级别漏洞就能获得 $2000 奖励，High/Critical 更高，而且只要是有效漏洞，Google 给予的漏洞分级一般都不会低于 moderate。如果提交了补丁且被接受，就有机会额外再加 $1000。然而，从 2023 年 5 月开始，moderate 级别漏洞奖金大幅缩水，[最高只有 $250](https://bughunters.google.com/about/rules/6171833274204160/android-and-google-devices-security-reward-program-rules#qualifying-exploit-chains)，且大部分此级别漏洞不再被分配 CVE。笔者曾经提交过 Moderate Severity + Medium Quality 的 bug report，得到的回复是不符合奖励标准。即使获得奖金，Google 的动作也实在是有些慢。笔者曾经询问过奖金进度，得到的回复是这样的：

> Hello,  
> Thanks for reaching out. Rewards are processed at 90 days after submission and once it has been processed, you will receive an email with details on the next steps to collect the reward.  
> Best Regards,  
> Android Security Team

嗯，90 天，Only Google can do。而对于一个被确认的安全漏洞的生命周期是这样的：

> 1.  Initial severity rating assessment (subject to change after review by component owners) (3)
> 2.  Development of an update
> 3.  Assignment of CVE
> 4.  Shared under NDA, as part of coordinated disclosure, to Android partners for remediation
> 5.  Release in a public Android security bulletin
> 6.  Android Security Rewards payment (if applicable)

而 Google 的慢会体现在每一步上，第一步的评级就有可能花几个月。即使内部已经写出来了修复，Google 也要首先向所有合作伙伴共享该漏洞，然后至少等一个月才会在每月安全公告中发布。不过值得高兴的一点是，大部分时候漏洞赏金都会在漏洞被修复之前就给你。很明显，如果你是一个独立的安全研究员，指望靠挖洞吃饭，先不论你能找到多少洞获得多少钱，就这个速度，估计你在钱到手上之前就饿死了。当然，如果你背后有着专业的安全公司或者团队，或者就是对安全感兴趣想来试试，那可以当我没说。我相信人的兴趣来了没有人能挡住，就像以前的我一样 ~（你先把你脑子治好再说话才能让人信服）~。如果你仍然愿意投入时间精力研究，那我相信 Google 不会亏待你。正所谓：天道酬勤。