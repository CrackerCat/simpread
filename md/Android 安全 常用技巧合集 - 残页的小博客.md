> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.canyie.top](https://blog.canyie.top/2026/06/09/android-security-tricks/)

> 一些自己写 PoC 的时候的常用技巧。

一些自己写 PoC 的时候的常用技巧。

[](#Binder "Binder")Binder
--------------------------

### [](#oneway-调用的数据大小限制 "oneway 调用的数据大小限制")oneway 调用的数据大小限制

这个技巧最早是我跟 WeiMin Cheng 一起写 CVE-2025-48570 的 PoC 的时候用到的，后面在 BlackHat Asia 2026 讲 [VsyncBreaker: Subverting Screen Trust via State Disruption and ONE-WAY Flooding](https://blackhat.com/asia-26/briefings/schedule/#vsyncbreaker-subverting-screen-trust-via-state-disruption-and-one-way-flooding-50006) 的时候也作为一个核心技巧进行讲解。

简单来说，binder 调用有一种 oneway mode，通过在 AIDL 里指明 oneway 关键字或者调用 `IBinder.transact()` 的时候加上 `FLAG_ONEWAY` 来使用，在这种模式下 caller 不会等待 callee 执行完毕。  
我们知道，binder 调用有一个大约 1MB 的大小限制，但是很少有人知道，对于 oneway 调用，这个限制只有常规调用的一半，见  
[https://cs.android.com/android/kernel/superproject/+/common-android17-6.18-2026-06:common/drivers/android/binder_alloc.c;l=951;bpv=0](https://cs.android.com/android/kernel/superproject/+/common-android17-6.18-2026-06:common/drivers/android/binder_alloc.c;l=951;bpv=0)

这里就有一个有趣的点可以利用，假设存在 `App A -> 系统组件 B -> 系统组件 C` 这样的调用链，`B->C` 是 oneway 调用且传递的数据受我们控制，就可以让这个数据大小刚好比 binder 总限制小但是比 oneway 限制大，从而让 `A->B` 这个调用成功但是 `B->C` 失败，从而打破系统的逻辑造成异常行为。

### [](#oneway-调用的-pid "oneway 调用的 pid")oneway 调用的 pid

oneway 调用还有一个很有趣的特点，就是被调用者使用 `Binder.getCallingPid()` 收到的 pid 是 0。如果有代码是比对 pid 来鉴权，就有可能出问题。CVE-2025-48635 就是这样，只比对了 pid 就直接返回了 activity token，所以可以拿到死亡进程的 Activity Token。

### [](#如何利用-IApplicationThread "如何利用 IApplicationThread")如何利用 IApplicationThread

泄露出其他进程的 IApplicationThread 后，可以直接调用 `performReceiver()` 传递伪造的 ActivityInfo 来触发代码执行。但是，从 Android 17 开始，[系统禁止了非 system uid 直接调用 IApplicationThread 提供的方法](https://cs.android.com/android/_/android/platform/frameworks/base/+/3a9b1c451a0d4a2e5cea997364ecba98912f85fb)。虽然我没有实验过，但是我觉得还是可以把 IApplicationThread 传递给 AMS 来绕过这个限制。

同时，系统里面还有非常多的方法使用 IApplicationThread 进行鉴权，比如 `startActivity()`，还有 Michal Bednarski [提到](https://github.com/michalbednarski/LeakValue#additional-note-other-ways-to-use-iapplicationthread)的 `grantUriPermission()`。

### [](#利用-ReadWriteHelper-任意反序列化 "利用 ReadWriteHelper 任意反序列化")利用 ReadWriteHelper 任意反序列化

在 Android 13 之前，想要强行让其他进程反序列化一个指定的对象很简单，在 Bundle 里塞一个无关的键值对，把 bundle 传过去然后等待目标进程从里面取元素，往里面放元素，甚至只是调用 `size()` 触发 bundle 被反序列化即可。这是因为 Bundle 会一次性把所有元素都反序列化。

然而，Android 13 开始，Lazy Bundle 优化改变了这一点。Bundle 现在只会反序列化需要的元素，同时系统里大量使用了新添加的带类型的 `readParcelable()` 和 `getParcelable()` 系列方法，会直接拒绝非预期的类型，减少攻击面。这里给出一个代替方案，Parcel 中存在一种名为 ReadWriteHelper 的东西，在有 ReadWriteHelper 的 Parcel 上读 Bundle 会直接导致所有元素都被立刻反序列化：  
[https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/os/BaseBundle.java;l=1916-1927](https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/os/BaseBundle.java;l=1916-1927)

```
if (parcel.hasReadWriteHelper()) {
    
    
    
    
    synchronized (this) {
        mOwnsLazyValues = false;
        initializeFromParcelLocked(parcel,  false, isNativeBundle);
    }
    mHasIntent = parcel.readBoolean();
    return;
}
```

什么时候 Parcel 会有 ReadWriteHelper？其中一个情况是 RemoteViews：  
[https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/widget/RemoteViews.java;l=2896-2909](https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/widget/RemoteViews.java;l=2896-2909)

```
case BUNDLE:
    
    
    
    
    
    if (in.hasReadWriteHelper()) {
        this.mValue = in.readBundle();
    } else {
        in.setReadWriteHelper(ALTERNATIVE_DEFAULT);
        this.mValue = in.readBundle();
        in.setReadWriteHelper(null);
    }
    break;
```

那么问题就转化为，怎么才能给其他进程塞任意 RemoteViews。以 system_server 为例，AppWidgetManager 就提供了一个方法 [setWidgetPreview()](https://developer.android.com/reference/android/appwidget/AppWidgetManager#setWidgetPreview(android.content.ComponentName,%20int,%20android.widget.RemoteViews)) 可以任意指定 RemoteViews，而不需要发送通知（会被用户看见）或者安装桌面小部件（需要用户交互）。甚至还有一个配套方法 [getWidgetPreview()](https://developer.android.com/reference/android/appwidget/AppWidgetManager#getWidgetPreview(android.content.ComponentName,%20android.os.UserHandle,%20int)) 来取回这个 RemoteViews。

这个特性也会引发一些问题，比如正常情况下，把一个指定 Parcelable 类型的数组比如 `Intent[]` 放进 Bundle 中，然后使用带类型的 `getParcelableArray()` 方法指定好类型为 `Intent.class`，读出来的会是一个 `Intent[]` 对象。但是在 ReadWriteHelper 开启的时候，比如把一个 View 放进 RemoteViews 里然后通过反射调用一个接收 Bundle 的方法，在里面使用带类型的 `getParcelableArray()`，即使指定了类型，读出来的也会是 `Parcelable[]`，尝试把它转换成 `Intent[]` 会抛异常。这个问题的原因是 ReadWriteHelper 导致所有元素被提早反序列化，此时 Bundle 还不知道这个数组应该是 `Parcelable[]` 还是 `Intent[]`，导致类型变为 `Parcelable[]`，后续 `getParcelableArray()` 直接返回了内存中的 `Parcelable[]` 而忽略了类型。

### [](#递归调用 "递归调用")递归调用

一个很少人知道的特性是，如果 binder 同步调用过程中发生嵌套 binder 调用，那么这个事务会被分发到同一个线程上执行。什么意思？假设进程 A 的线程 T1 向进程 B 发起一个同步 Binder 调用，B 挑选一个线程 T2 处理这个调用的过程中又同步调用回了进程 A，那么这个时候会是 T1 去执行 A 里面的代码。  
[https://source.android.com/docs/core/architecture/ipc/binder-threading?hl=zh-cn#nested](https://source.android.com/docs/core/architecture/ipc/binder-threading?hl=zh-cn#nested)

### [](#ParceledListSlice "ParceledListSlice")ParceledListSlice

ParceledListSlice 是一个特殊的 Parcelable，它本来是用于分片传输大数据的，但是有一个非常有趣的特性，构造 parcel 数据可以让它在被反序列化时立刻向指定 binder 发起同步调用：  
[https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/content/pm/BaseParceledListSlice.java;l=95-117](https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/content/pm/BaseParceledListSlice.java;l=95-117)

```
@SuppressWarnings("unchecked")
BaseParceledListSlice(Parcel p, ClassLoader loader) {
    
    final IBinder retriever = p.readStrongBinder();
    while (i < N) {
        if (DEBUG) Log.d(TAG, "Reading more @" + i + " of " + N + ": retriever=" + retriever);
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInt(i);
        try {
            retriever.transact(IBinder.FIRST_CALL_TRANSACTION, data, reply, 0); 
            
        } catch (RemoteException e) {
            throw new BadParcelableException(
                    "Failure retrieving array; only received " + i + " of " + N, e);
        } finally {
            reply.recycle();
            data.recycle();
        }
    }
}
```

一个利用方法是利用它去争抢其他进程内的锁然后回调自身，让某些代码在精确的时机执行，或者阻塞同样需要这个锁的其他线程。假设 system server 内存在如下 binder 调用：

```
public void myTransact(Bundle data) {
    synchronized (myLock) {
        data.getParcelable("xxx");
        
    }
}
```

那么就可以在 Bundle 内放入 ParceledListSlice，`myLock` 被获得后我们的进程就会收到通知，并且在我们返回正确这个锁不会被释放。

另一种思路是结合上面的嵌套调用去强制一个线程重入某个没有预料到会有这种情况发生的方法，造成非预期的行为。CVE-2025-48583 及 [Project Zero 利用的无限递归 A-466091304](https://project-zero.issues.chromium.org/issues/465827985) 都是例子，虽然 Project Zero 给的 PoC 是利用的另一个对象。

[](#Intent "Intent")Intent
--------------------------

### [](#只检查-intent-目标是危险的 "只检查 intent 目标是危险的")只检查 intent 目标是危险的

我们经常看见这种代码：

```
Intent incomingIntent = ...; 
if (isActivityAllowedToStart(incomingIntent)) {
    startActivity(incomingIntent);
}
```

这种代码不一定安全。如果 `isActivityAllowedToStart()` 允许启动调用者所有的 Activity，但没有清理 flags，攻击者就有可能指定 `FLAG_GRANT_READ_URI_PERMISSION` `FLAG_GRANT_WRITE_URI_PERMISSION` 获得权限以受害 app 的身份访问 content provider。如果受害 app 存在 FileProvider，还可以转换成文件写入。

另一种情况是，`isActivityAllowedToStart()` 允许启动 exported 且没有权限保护的 Activity，这种代码模式在某些系统 app 里经常做到。但是，没有权限保护的 Activity 并不一定全都安全，比如 `ChooserActivity` `ResolverActivity` 等会使用调用者的身份去发生另一个 intent，`SearchResultTrampoline` 会校验调用者是 Settings 然后用系统权限发送 intent；除此之外，CertInstaller 允许静默安装证书，PackageInstaller 允许绕过未知来源权限，这些都是潜在的危险。更多请查看[我之前对 CVE-2024-40676 的分析](https://blog.canyie.top/2024/11/07/self-changing-data-type/)。

即使以上情况都覆盖了，`incomingIntent` 也可能是 `Intent` 子类，造成 Parcel Creator Mismatch 问题。关于这类问题，请查看我们的论文 Parcel Mismatch Demystified: Addressing a Decade-Old Security Challenge in Android

### [](#通过-action-进行-URI-grant "通过 action 进行 URI grant")通过 action 进行 URI grant

有些时候，我们可以部分控制受害 app 启动的 Intent 但无法控制 flags （比如 flags 已经被明确去除了，或者 intent 来自 `Intent.parseIntent()` 这种不允许指定 flags 的情况），可以控制 action 进行 URI grant。这是因为 `startActivity()` 过程中 `Intent.migrateExtraStreamToClipData()` 有可能会把 grant flags 给加回来：  
[https://cs.android.com/android/platform/superproject/+/android16-qpr2-release:frameworks/base/core/java/android/content/Intent.java;l=13419-13607;bpv=0](https://cs.android.com/android/platform/superproject/+/android16-qpr2-release:frameworks/base/core/java/android/content/Intent.java;l=13419-13607;bpv=0)

```
public boolean migrateExtraStreamToClipData(Context context) {
    
    if (mExtras != null && mExtras.isParcelled()) return false;

    
    if (getClipData() != null) return false;

    final String action = getAction();
    
    } else if (isImageCaptureIntent()) {
        Uri output;
        try {
            output = getParcelableExtra(MediaStore.EXTRA_OUTPUT, Uri.class);
        } catch (ClassCastException e) {
            return false;
        }

        if (output != null) {
            
            if (isMissingGrantFlag(FLAG_GRANT_READ_URI_PERMISSION)) {
                int grantType;
                if (android.security.Flags
                        .implicitUriGrantsRestrictedForSendmultipleImagecaptureActions()) {
                    Log.e(TAG,
                            "Skipping implicit URI read grants for ImageCapture action "
                                    + "because it is restricted");
                    grantType = IMPLICIT_URI_GRANT_EVENT_REPORTED__GRANT_TYPE__RESTRICTED;
                } else {
                    addFlags(FLAG_GRANT_READ_URI_PERMISSION);
                    grantType = IMPLICIT_URI_GRANT_EVENT_REPORTED__GRANT_TYPE__GRANTED;
                }
            }
            if (isMissingGrantFlag(FLAG_GRANT_WRITE_URI_PERMISSION)) {
                int grantType;
                if (android.security.Flags
                        .implicitUriGrantsRestrictedForSendmultipleImagecaptureActions()) {
                    Log.e(TAG,
                            "Skipping implicit URI write grants for ImageCapture action "
                                    + "because it is restricted");
                    grantType = IMPLICIT_URI_GRANT_EVENT_REPORTED__GRANT_TYPE__RESTRICTED;
                } else {
                    addFlags(FLAG_GRANT_WRITE_URI_PERMISSION);
                    grantType = IMPLICIT_URI_GRANT_EVENT_REPORTED__GRANT_TYPE__GRANTED;
                }
            }
            return true;
        }
```

想要利用这一点需要满足以下条件：

*   Intent 的 action 受控，需要是 `ACTION_SEND` `ACTION_SEND_MULTIPLE` 或者 5 个特定的 image capture action
*   不能有 Clipdata
*   extras 不能是 parcelled 状态，也就是说，刚从 extras 里通过 `getParcelable()` 拿到的 intent。对 extras 进行任何操作，比如 `getXxxExtra()` 或者 `putExtra()` 都会打破这个状态。对于不通过 Parcelable 机制传输的 intent，比如通过 `Intent.parseUri()` 或者 `Intent.parseIntent()` 解析的 intent 这个条件天然满足。

可惜这么好用的路子要在 Android 18 被堵上了，见 [限制隐式 URI 授权](https://developer.android.com/about/versions/17/behavior-changes-all?hl=zh-cn#restrict-implicit-uri-grants)

### [](#mutable-PendingIntent-URI-grant "mutable PendingIntent URI grant")mutable PendingIntent URI grant

经常能看见 base intent 只有 action 的 mutable PendingIntent，如果攻击者拿到这些 PendingIntent 可以使用 `PendingIntent.send()` 在 `fillInIntent` 中将 package 指向自身，设置 data/clipdata 为需要访问的 URI，同时添加 URI grant flags，在 AndroidManifest.xml 中注册相同 action 的 intent-filter，当恶意应用被打开的瞬间就会获得指定 URI 的读写权限。更多可以见 [PendingIntent 重定向：一种针对安卓系统和流行 App 的通用提权方法——BlackHat EU 2021 议题详解 （下）](https://mp.weixin.qq.com/s/ifrErL88_8wN36WT8QbLvg)

### [](#getCallingPackage-伪造 "getCallingPackage() 伪造")getCallingPackage() 伪造

以下代码模式在 Activity 中经常出现：

```
if (isCallerAllowed(getCallingPackage())) {
    executePrivilegedAction();
}
```

然而， `getCallingPackage()` 和 `getCallingActivity()` 返回的不一定真的是启动当前 activity 的那个 app，它实际上返回的是谁会收到 `setResult()` 的结果。因此：

*   如果调用者没有使用 `startActivityForResult()`，被调用者只能拿到 null
*   使用 `FLAG_ACTIVITY_FORWARD_RESULT` 或者 `startIntentSenderForResult()` 时，拿到的 caller 会跟预期不同

### [](#从任意-app-调用-startIntentSenderForResult "从任意 app 调用 startIntentSenderForResult()")从任意 app 调用 startIntentSenderForResult()

上面提到，如果要伪造 `getCallingPackage()` 的返回值，需要使用 `FLAG_ACTIVITY_FORWARD_RESULT` 或者 `startIntentSenderForResult()`。经常使用的是前者，但是前者并不是万能的。考虑该代码：`if ("com.xxx.trustedapp".equals(getCallingPackage()))`，要让 `getCallingPackage()` 返回 `com.xxx.trustedapp`，如果使用前者，那么 `com.xxx.trustedapp` 需要使用 `startActivityForResult()` 来启动攻击者的 app。如果它用了一些可以拦截的隐式意图那还好办，可如果没有呢？

这里我提供一个代替方法：实现一个 Autofill Service 让用户选为默认，在 `onFillRequest()` 里通过 [FillResponse.Builder.setAuthentication()](https://developer.android.com/reference/android/service/autofill/FillResponse.Builder#setAuthentication(android.view.autofill.AutofillId[],%20android.content.IntentSender,%20android.service.autofill.Presentations)) 设置好 IntentSender 然后作为响应返回，当用户点按弹出的候选项时就会触发 IntentSender 在目标 app 里被发送。因为[这部分代码](https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/view/autofill/AutofillClientController.java;l=586)在 framework 里，所以几乎所有拥有可被填充 View 的 app 都可以被利用。缺点就是必须要用户授权 + 用户交互。

### [](#TOCTTOU-race-condition "TOCTTOU race condition")TOCTTOU race condition

即使 intent 校验全都通过，如果 intent 是隐式的，那么发送它时实际打开的目标还是有可能解析到不同的 activity。典型例子包括通过 `getType()` 接收回调然后返回不同的 mime type（[Self-changing Data Type](https://blog.canyie.top/2024/11/07/self-changing-data-type/)）和启用 / 禁用组件等方法修改会影响 Activity 解析的全局状态（[BadResolve](https://media.defcon.org/DEF%20CON%2033/DEF%20CON%2033%20presentations/Qidan%20He%20-%20Dead%20Made%20Alive%20Again%20Bypassing%20Intent%20Destination%20Checks%20and%20Reintroducing%20LaunchAnyWhere%20Privilege%20Escalation.pdf)）。

[](#组件安全 "组件安全")组件安全
--------------------

### [](#ActivityManager-openContentUri "ActivityManager.openContentUri()")ActivityManager.openContentUri()

有时你会在某个导出的 provider 里看见这样的代码：

```
public class MyProvider extends ContentProvider {
    @Override public ParcelFileDescriptor openFile(Uri uri, String mode) {
        getContext().enforceCallingPermission(xxx); 
        
    }
}
```

在 2023-08 CVE-2023-21292 被修复之前，可以使用 `ActivityManagerService.openContentUri()` 让系统以自己的身份去 openFile 然后拿到返回的 fd，参见  
[https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=7390](https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=7390)

最初来自  
[https://twitter.com/i/status/1555594672571031552](https://twitter.com/i/status/1555594672571031552)

### [](#系统权限触发-ContentProvider "系统权限触发 ContentProvider")系统权限触发 ContentProvider

有些时候被保护的 provider 的 `getType()` `query()` `openFile()` 等方法内会有一些其他逻辑，虽然非特权 app 不能直接调用，但是可以设法让系统帮我们触发它们，比如塞进 icon 里让系统去显示，这个过程就会触发 `openFile()`。

[https://twitter.com/_bagipro/status/2026798826871247233](https://twitter.com/_bagipro/status/2026798826871247233)

### [](#ContentProvider-openFile-只读模式可以使用-truncate-flag "ContentProvider.openFile() 只读模式可以使用 truncate flag")ContentProvider.openFile() 只读模式可以使用 truncate flag

在 2026-03 CVE-2025-48619 被修复前，`rt` 和 `ra` 这样的 mode 是合法的，而且 ContentProvider 只会检查读权限，可能导致非预期行为。Android 17 开始，这种用法会抛异常。

### [](#ContentProvider-openFile-对-rw-只校验写权限 "ContentProvider.openFile() 对 rw 只校验写权限")ContentProvider.openFile() 对 rw 只校验写权限

当使用 rw 模式 `openFile()` 时，只有 write permission 会被检查，read permission 会被忽略：  
[https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/content/ContentProvider.java;l=804-818](https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/base/core/java/android/content/ContentProvider.java;l=804-818)

```
private void enforceFilePermission(@NonNull AttributionSource attributionSource,
        Uri uri, String mode)
        throws FileNotFoundException, SecurityException {
    if (mode != null && mode.indexOf('w') != -1) {
        if (enforceWritePermission(attributionSource, uri)
                != PermissionChecker.PERMISSION_GRANTED) {
            throw new FileNotFoundException("App op not allowed");
        }
    } else {
        if (enforceReadPermission(attributionSource, uri)
                != PermissionChecker.PERMISSION_GRANTED) {
            throw new FileNotFoundException("App op not allowed");
        }
    }
}
```

如果底层实现不清楚这个行为，可能会导致问题。

### [](#主线程错误鉴权 "主线程错误鉴权")主线程错误鉴权

有些代码长这样：

```
public class MyService {
    @Override public IBinder onBind(Intent intent) {
        enforceCallingOrSelfPermission(xxx);
        return binder;
    }
}
```

这种写法完全错误，因为 `onBind()` 在 app 主线程而非 binder 线程执行，实际上只会检查 app 自己而非调用者的权限。`Service.onBind()` 是最常出现这种错误的地方，其他变体还有写在 `Activity.onCreate()` 等。

### [](#createPackageContext-gt-代码执行 "createPackageContext() -> 代码执行")createPackageContext() -> 代码执行

假设受害 app 调用了 `createPackageContext(attackerPackageName, CONTEXT_INCLUDE_CODE | CONTEXT_IGNORE_SECURITY)`，如何把它变成任意代码执行？  
只需要在 AndroidManifest.xml 里设置 `android:appComponentFactory`，你指定的类就会在 `context.getClassLoader()` 被调用时被自动实例化，从而触发构造函数的执行。

[](#Android-Binary-XML "Android Binary XML")Android Binary XML
--------------------------------------------------------------

Android 里，系统服务一般使用 `Xml.resolveSerializer()` 来存储数据，在 Android 12+ 上默认会使用 BinaryXmlSerializer。

### [](#字符串长度限制 "字符串长度限制")字符串长度限制

观察 BinaryXmlSerializer 的源码，你会发现它使用 FastDataOutput 写数据，而它有一个限制，就是写进去的字符串长度不能超过 65535，否则就会抛出 IOException：  
[https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/libs/modules-utils/java/com/android/modules/utils/FastDataOutput.java;l=150-155](https://cs.android.com/android/platform/superproject/+/android-16.0.0_r1:frameworks/libs/modules-utils/java/com/android/modules/utils/FastDataOutput.java;l=150-155)

```
@Override
public void writeUTF(String s) throws IOException {
    final int len = (int) ModifiedUtf8.countBytes(s, false);
    if (len > MAX_UNSIGNED_SHORT) {
        throw new IOException("Modified UTF-8 length too large: " + len);
    }
    
}
```

（注：实际上在 art 上运行的时候会使用子类 ArtFastDataOutput 写数据，里面也有一样的逻辑，感兴趣的可以自己去看看，这里不赘述）  
如果某个系统服务存储安全相关的设置，比如某个权限是否被授予，同时它还存储某个我们受控的数据，也就是说伪代码大概长这样：

```
FileOutputStream fos = null;
try {
    fos = mSessionsFile.startWrite();

    final TypedXmlSerializer out = Xml.resolveSerializer(fos);
    out.startDocument(null, true);
    
    out.attributeBoolean(null, "permission-granted", isPermissionGranted);
    out.attribute(null, "extraData", attackerControlledString);
    
    out.endDocument();

    mSessionsFile.finishWrite(fos);
    return true;
} catch (IOException e) {
    if (fos != null) {
        mSessionsFile.failWrite(fos);
    }
}
```

那么攻击者就可以使用超长字符串让整个存储过程失败，导致更改被回滚。典型的流程是：用户打开某个权限，权限状态被写入到文件里，恶意 app 插入超长字符串，此时用户如果关闭权限，页面上会显示已经关闭，但是写文件的过程静默失败，新的状态没有被保存，下次开机的时候读到的还是旧的启用状态，表现就是权限自动打开，用户无法撤销授权。

[](#常用代码片段 "常用代码片段")常用代码片段
--------------------------

### [](#纯-java-无外部依赖-关闭隐藏-API-访问限制 "纯 java 无外部依赖 关闭隐藏 API 访问限制")纯 java 无外部依赖 关闭隐藏 API 访问限制

不知道从什么时候开始，提交的 PoC 不让引用 HiddenApiBypass 这种外部库了，让用 adb 关闭限制，但是如果你真的让他们用 adb 又有可能给你评 Low Quality  
[https://github.com/michalbednarski/LeakValue/blob/b0a2e05c079d2cf8a1e6af208870db1885ac9064/app/src/main/java/com/example/leakvalue/MiscUtils.java#L228-L247](https://github.com/michalbednarski/LeakValue/blob/b0a2e05c079d2cf8a1e6af208870db1885ac9064/app/src/main/java/com/example/leakvalue/MiscUtils.java#L228-L247)

### [](#简单字符串生成 "简单字符串生成")简单字符串生成

常用于资源耗尽类问题，生成多个不重复的字符串

```
final int LENGTH = (1 << 16) - 1; 
final int CHAR_RANGE = 'z' - 'A'; 
char[] chars = new char[LENGTH];
Arrays.fill(chars, 'A');
for (int i = 0;i < size;i++) {
    String str = new String(chars);
    chars[i / CHAR_RANGE]++;
}
```

### [](#调用摄像头拍照 "调用摄像头拍照")调用摄像头拍照

常用于演示在后台使用 while in use 权限  
使用已被弃用的 camera 1 API，所以会有警告，只是为了演示漏洞利用的话直接忽略掉即可

```
@SuppressWarnings("deprecation") public static void takePhoto(String outputPath) {
    Log.e("PoC", "take photo start");
    android.hardware.Camera camera;
    SurfaceTexture texture = new SurfaceTexture(10);
    try {
        camera = android.hardware.Camera.open();
        camera.setPreviewTexture(texture);
        camera.startPreview();
        camera.takePicture(null, null, (data, camera1) -> {
            try (FileOutputStream fos = new FileOutputStream(outputPath)) {
                fos.write(data);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }

            camera1.release();
            texture.release();
            Log.e("PoC", "take photo end");
        });
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

### [](#延时任务 "延时任务")延时任务

常用于演示退到后台后 30 秒 仍可以弹出 activity 或使用 while in use 权限，由于系统优化，实际回调时间可能比设定的时间稍长，如果需要精确延时可以申请权限并使用 exact alarm  
需要创建一个 BroadcastReceiver，不使用 Handler 的原因是 Android 14 以上应用退到后台会被系统冻结导致任务在后台不执行

```
int delay = 30 * 1000; 
PendingIntent pendingIntent = PendingIntent.getBroadcast(
        this,
        0,
        new Intent(this, MyReceiver.class),
        PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_ONE_SHOT
);
AlarmManager alarmManager = (AlarmManager) getSystemService(ALARM_SERVICE);
alarmManager.setAndAllowWhileIdle(
        AlarmManager.ELAPSED_REALTIME_WAKEUP,
        SystemClock.elapsedRealtime() + delay,
        pendingIntent
);
```

### [](#获取-Class-对象所属的-dex "获取 Class 对象所属的 dex")获取 Class 对象所属的 dex

需要禁用隐藏 API 限制

```
public static String getDexLocation(Class<?> cls) {
    try {
        Class<?> DexCache = Class.forName("java.lang.DexCache");
        Field dexCache = Class.class.getDeclaredField("dexCache");
        dexCache.setAccessible(true);
        Object cache = dexCache.get(cls);
        if (cache == null) return null;
        Field location = DexCache.getDeclaredField("location");
        location.setAccessible(true);
        return (String) location.get(cache);
    } catch (ReflectiveOperationException e) {
        throw new RuntimeException(e);
    }
}
```

  

> 博客内容遵循 署名 - 非商业性使用 - 相同方式共享 4.0 国际 (CC BY-NC-SA 4.0) 协议
> 
> 本文永久链接是：[https://blog.canyie.top/2026/06/09/android-security-tricks/](https://blog.canyie.top/2026/06/09/android-security-tricks/)