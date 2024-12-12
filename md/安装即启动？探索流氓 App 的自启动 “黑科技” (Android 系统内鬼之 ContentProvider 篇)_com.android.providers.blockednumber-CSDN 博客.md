> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u011387817/article/details/132260923)

前段时间发现了一个神奇的 app，它居然可以在安装之后立即自启动：

![](https://i-blog.csdnimg.cn/blog_migrate/70757896e057621c38121f1fe575d726.gif)

看到没有，在提示安装成功大概 1 到 2 秒后，就直接弹出 [Toast](https://so.csdn.net/so/search?q=Toast&spm=1001.2101.3001.7020) 和通知了！ 好神奇啊，在没有第三方 app 帮忙唤醒的前提下，它是怎么做到首次安装即自启动的呢？

初步分析
----

难道它监听了应用安装的广播，在收到广播之后立即启动后台服务？  
用 [jadx](https://so.csdn.net/so/search?q=jadx&spm=1001.2101.3001.7020) 打开一看，确实有监听应用安装和卸载的 BroadcastReceiver：

![](https://i-blog.csdnimg.cn/blog_migrate/d1696c671b8e13241b371538d8375f93.png)

但是从截图上来看，这个 receiver 只有 2 个常见的属性: `enable`和`exported`，甚至`intent-filter`都没有设置优先级，分明就是一个很普通的 receiver 嘛。  
而且按常理，在 android 系统上，新安装的 app 如果没有主动运行过一次，那么它所有的 BroadcastReceiver 都是不会生效的，例如监听应用安装卸载、监听设备开机、熄屏亮屏等。  
就算它有办法绕过这个限制，那它真的能接收到**自身的安装广播**吗？（反正这种操作我是第一次见）

不过我还是仿照它的做法，写 demo 测试了一下……

**得到的结果是：** 接收不到任何广播。  
这就说明这个 app 的【安装完自启动】**并不是**通过监听自身的安装广播来实现的。

那么，**它到底是怎么启动的呢，会是谁启动了它呢？**

也许我们可以使用 debug 法来进行分析（当然，debug 系统进程需要手机获取 root 权限，或者直接刷入一个 user-debug/eng 系统，这不在本文的讨论范围内）。

有同学可能会说，**可以在 AMS 的`attachApplication`方法里打断点，因为这是 app 进程启动的必经之路。**  
emmmm，这是必经之路没错，但如果在这里打断点已经迟了，因为这时候进程已经启动，依然无法得知是由哪个进程发起的。  
所以我们应该尽量在靠近启动源头的地方打断点。

寻找启动源头
------

先来复习一下常规应用进程的启动流程：

![](https://i-blog.csdnimg.cn/blog_migrate/35c52df0db454a75a5b50c4e8dee4340.png)

### 

[查看大图](https://i-blog.csdnimg.cn/blog_migrate/35c52df0db454a75a5b50c4e8dee4340.png)

可以看到，向 zygote 发起 fork 请求的是 system_process 进程，我们可以在 system_process 这条线上的任意一个方法打断点，比如 ZygoteProcess.`start`方法：

![](https://i-blog.csdnimg.cn/blog_migrate/a7845f3839de19fc8305a330ba295788.png)

等下就可以顺着堆栈去找到启动的源头了。

> 如果你的手机不是 user-debug/eng 系统但有 root 权限（现在获取 root 权限基本上都是刷 magisk 了吧？），可以直接在 shell 中通过以下命令来临时 (重启后失效) 开启全局 debug：  
> `magisk resetprop ro.debuggable 1&&stop;start`

好，attach 上 system_process 进程：

![](https://i-blog.csdnimg.cn/blog_migrate/8422d93309dd9334b1d36c0265dbca43.png)

![](https://i-blog.csdnimg.cn/blog_migrate/b4583af4e769ffc21d5e7c49ec83310d.png)

现在卸载重新安装一遍（等它自启动）：

![](https://i-blog.csdnimg.cn/blog_migrate/46c9413b0941ad07e26bfa781500a237.png)

来了来了，就是这个`com.fg`！来看下调用链的前半段（注意选中的那个 [lambda](https://edu.csdn.net/cloud/houjie?utm_source=highword&spm=1001.2101.3001.7020)）：

![](https://i-blog.csdnimg.cn/blog_migrate/b40e9a5af96c549ae6c1918afa65423b.png)

原来这里有个 Handler.`post`，我们在它外面再打一个断点，这样就能看到`post`之前的调用链了：

![](https://i-blog.csdnimg.cn/blog_migrate/d7eb0f24c06e70510d630996eabbc1c9.png)

好，再次卸载重新安装（等它自启动）：

![](https://i-blog.csdnimg.cn/blog_migrate/3076b79b75e8c43777d8930192a945b1.png)

咦？？？为什么源头是 AMS 的`getContentProvider`方法啊？  
看下变量面板：

![](https://i-blog.csdnimg.cn/blog_migrate/474c458d181156e6d2866ab5f3e96915.png)

这个`callingPackage`就是本次调用`getContentProvider`方法的进程包名；  
`name`即目标 ContentProvider 在 AndroidManifest 中声明的 authorities（系统唯一）；

现在可以得出结论：  
app 在安装之后，com.android.providers.blockednumber 进程会通过`getContentProvider`获取 com.fg.account.kp.provider 而间接启动了进程！

那么，**为什么 blockednumber 进程要获取这个 provider 呢？**

还是继续 debug 根据堆栈来溯源吧：

![](https://i-blog.csdnimg.cn/blog_migrate/99e45f111af60158f9b1601b9acffdf3.png)

咦？奇怪，居然没有 com.android.providers.blockednumber 进程。  
很有可能是它修改了进程名。 我们现在已经知道了它的包名，可以通过`pm path`命令来得到对应 apk 的路径：

```
:~$ adb shell pm path com.android.providers.blockednumber
        package:/system/priv-app/BlockedNumberProvider/BlockedNumberProvider.apk
```

把它 pull 上来然后拖进 as 看下 AndroidManifest：

```
:~$ adb pull /system/priv-app/BlockedNumberProvider/BlockedNumberProvider.apk .
        /system/priv-app/BlockedNumberProvider...ed. 12.6 MB/s (303518 bytes in 0.023s)
```

![](https://i-blog.csdnimg.cn/blog_migrate/06cce4ca51651be56cf3f21f321fe154.png)

emmmm，果然没猜错，进程名改为`android.process.acore`了，也就是上图中的第二个进程。  
赶紧 attach 上，然后给 IActivityManager 的`getContentProvider`方法打上断点：

![](https://i-blog.csdnimg.cn/blog_migrate/b2c47f11d315ff5b0deebe06b1821a8c.png)

再把那个 apk 继续重安装一遍（等它自启动）：

![](https://i-blog.csdnimg.cn/blog_migrate/1971dd5d1677653670ae01cde9f73e96.png)

断点到了！把调用链整理一下：

```
android.app.IActivityManager$Stub$Proxy.getContentProvider() -->
android.app.ActivityThread.acquireProvider() -->
android.content.ContextImpl$ApplicationContentResolver.acquireUnstableProvider() -->
android.content.ContentResolver.acquireUnstableProvider() -->
android.content.ContentResolver.query() -->
com.android.providers.contacts.ContactDirectoryManager.queryDirectoriesForAuthority() -->
com.android.providers.contacts.ContactDirectoryManager.updateDirectoriesForPackage() -->
com.android.providers.contacts.ContactDirectoryManager.onPackageChanged() -->
com.android.providers.contacts.ContactsProvider2.onPackageChanged() -->
com.android.providers.contacts.ContactsPackageMonitor.onPackageChanged() -->
com.android.providers.contacts.ContactsPackageMonitor.onPerformTask() -->
com.android.providers.contacts.ContactsTaskScheduler$MyHandler.handleMessage() -->
android.os.Handler.dispatchMessage() -->
android.os.Looper.loop() -->
android.os.HandlerThread.run()
```

原来`getContentProvider`是因为 ContactDirectoryManager.`queryDirectoriesForAuthority`里面调用了 ContentResolver.`query`方法而间接调用到的。  
继续往下看，是连续三个`onPackageChanged`，根据方法名再结合刚刚安装 apk 的现象，就很容易能猜到它是监听了应用安装的广播。  
好，现在用 jadx 打开刚刚 pull 上来的 BlockedNumberProvider.apk，看下它这几个类的代码：

![](https://i-blog.csdnimg.cn/blog_migrate/afcdf72e2dc7d943ff51aedef69b2dad.png)

咦？？为什么没有这些类呢？ 甚至都没看到 com.android.providers.contacts 包名！  
再看一眼 Manifest:

![](https://i-blog.csdnimg.cn/blog_migrate/b665f61c90a77013fd7a32cbde7b5051.png)

它居然指定了 sharedUserId 为`android.uid.shared`！这样看来，很可能不止它一个 app 在用这个 sharedUserId。了解过 sharedUserId 的同学都知道，如果不同的 app 声明了相同的 sharedUserId 和相同的进程名，那么这些 app 就会运行在同一个进程中！  
所以我们前面 debug 时看到的`com.android.providers.contacts`这些包名的 class，很可能就在另外一个 app 上。  
**有什么办法可以查到还有哪些 app 跟它使用了同样的 sharedUserId 呢？**

很简单，只需要运行`adb shell dumpsys package com.android.providers.blockednumber`：

![](https://i-blog.csdnimg.cn/blog_migrate/dd8946a3a11ff9bff34ae773326cc946.png)

看第二个: `com.android.providers.contacts`，这不刚好就是上面调用了 ContentResolver.`query`方法的包名吗？

用前面的方法把它 pull 上来用 jadx 看看吧：

![](https://i-blog.csdnimg.cn/blog_migrate/464be63d889a6a8ffa2ad24b6165f5c9.png)

上面调用链里出现的类，在这里都找到了。  
再确认一下 Manifest：

![](https://i-blog.csdnimg.cn/blog_migrate/ce9739776b34829fc0c4afc6bec2629f.png)

看到没? `sharedUserId`和`process`都跟 BlockedNumberProvider.apk 是一样的，这就证明了这两个 apk 是运行在同一进程中的。

代码分析
----

先回顾一下之前断点到的调用链：

```
android.app.IActivityManager$Stub$Proxy.getContentProvider() -->
android.app.ActivityThread.acquireProvider() -->
android.content.ContextImpl$ApplicationContentResolver.acquireUnstableProvider() -->
android.content.ContentResolver.acquireUnstableProvider() -->
android.content.ContentResolver.query() -->
com.android.providers.contacts.ContactDirectoryManager.queryDirectoriesForAuthority() -->
com.android.providers.contacts.ContactDirectoryManager.updateDirectoriesForPackage() -->
com.android.providers.contacts.ContactDirectoryManager.onPackageChanged() -->
com.android.providers.contacts.ContactsProvider2.onPackageChanged() -->
com.android.providers.contacts.ContactsPackageMonitor.onPackageChanged() -->
com.android.providers.contacts.ContactsPackageMonitor.onPerformTask() -->
com.android.providers.contacts.ContactsTaskScheduler$MyHandler.handleMessage() -->
android.os.Handler.dispatchMessage() -->
android.os.Looper.loop() -->
android.os.HandlerThread.run()
```

最后是在 ContactDirectoryManager 的`queryDirectoriesForAuthority`方法里调用 ContentResolver.`query`方法，看下它的代码：

```
protected void queryDirectoriesForAuthority(ArrayList<DirectoryInfo> arrayList, ProviderInfo providerInfo) {
    Cursor cursor = null;
    try {
        cursor = this.mContext.getContentResolver().query(new Uri.Builder().scheme("content")
        .authority(providerInfo.authority).appendPath("directories").build(), DirectoryQuery.PROJECTION, null, null, null);
        if (cursor == null) {
            ......
        } else {
            while (cursor.moveToNext()) {
                DirectoryInfo directoryInfo = new DirectoryInfo();
                directoryInfo.packageName = providerInfo.packageName;
                directoryInfo.authority = providerInfo.authority;
                directoryInfo.accountName = cursor.getString(0);
                directoryInfo.accountType = cursor.getString(1);
                directoryInfo.displayName = cursor.getString(2);
                ......
                arrayList.add(directoryInfo);
            }
        }
    } catch (Throwable th) {
        ......
    }
}
```

大致的逻辑就是把查询出来的 Provider 信息放进一个 ArrayList 里面。  
注意：上面调用`getContentResolver().query`的时候，如果要查询的 Provider 进程不在运行中，AMS 会尝试启动这个 Provider 所在进程！

好，接下来看看在什么情况下它会调用这个`queryDirectoriesForAuthority`方法：

```
private List<DirectoryInfo> updateDirectoriesForPackage(PackageInfo packageInfo, boolean z) {
    ......
    ArrayList<DirectoryInfo> newArrayList = Lists.newArrayList();
    ProviderInfo[] providerInfoArr = packageInfo.providers;
    if (providerInfoArr != null) {
        for (ProviderInfo providerInfo : providerInfoArr) {
            // 这里
            if (isDirectoryProvider(providerInfo)) {
                queryDirectoriesForAuthority(newArrayList, providerInfo);
            }
        }
    }
    ......
}
```

原来是通过`isDirectoryProvider`方法来判断的，看下它的代码：

```
static boolean isDirectoryProvider(ProviderInfo providerInfo) {
     if (providerInfo == null) return false;
     Bundle metaData = providerInfo.metaData;
     if (metaData == null) return false;

     Object obj = metaData.get("android.content.ContactDirectory");
     return obj != null && Boolean.TRUE.equals(obj);
}
```

它是判断这个 provider 的`metaData`中的`"android.content.ContactDirectory"`属性是否为 true！

**还记得前面 debug 看到的那个被拉起的 provider 叫什么吗？**  
没错就是`com.fg.account.kp.provider`，那么现在我们来看下它在 AndroidManifest 中的声明：

![](https://i-blog.csdnimg.cn/blog_migrate/38f8a82f5cb9b8ee425fdffb43ad7cef.png)

妈耶！！！它`meta-data`里的`"android.content.ContactDirectory"`属性就是 true！

真的只有这么简单吗？**只需要在 provider 里面设置这个 meta-data 属性为 true 就可以实现安装自启动？**  
我们来写个 demo 来验证下叭！

效果验证
----

首先写一个 ContentProvider，并在`onCreate`方法里打印日志：

```
class AutoStartProvider : ContentProvider() {

    override fun onCreate(): Boolean {
        Log.e("AutoStartProvider", "process started")
        return true
    }

    override fun query(uri: Uri?, projection: Array<out String>?, selection: String?, selectionArgs: Array<out String>?, sortOrder: String?) = null

    override fun getType(uri: Uri?) = null

    override fun insert(uri: Uri?, values: ContentValues?) = null

    override fun delete(uri: Uri?, selection: String?, selectionArgs: Array<out String>?) = 0

    override fun update(uri: Uri?, values: ContentValues?, selection: String?, selectionArgs: Array<out String>?) = 0
}
```

然后在 AndroidManifest 里声明一下，并加上`"android.content.ContactDirectory"`属性：

```
<provider
    android:
    android:authorities="AutoStartProvider"
    android:exported="true">
    <meta-data
        android:
        android:value="true" />
</provider>
```

再加个前台服务，跟随 app 一起启动：

```
class AutoStartService : Service() {

    override fun onCreate() {
        super.onCreate()
        setForeground()
        Toast.makeText(this, "Service started", Toast.LENGTH_LONG).show()
    }

    private fun setForeground() {
        val channelId = "auto_start"
        (getSystemService(NOTIFICATION_SERVICE) as NotificationManager).createNotificationChannel(NotificationChannel(channelId, channelId, NotificationManager.IMPORTANCE_HIGH).apply {
            setSound(null, null)
            setShowBadge(false)
        })
        startForeground(
            1, Notification.Builder(this, channelId)
                .setContentTitle("Service started")
                .setSmallIcon(R.drawable.ic_launcher_foreground)
                .build()
        )
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

好，push 到测试机上安装看看：

![](https://i-blog.csdnimg.cn/blog_migrate/fa4eda01a51d607c990027945dfb30cc.gif)

哈哈哈哈哈，成功了！居然真的就这么简单！

好了，最后我们来总结一下叭：

总结
--

1.  我们发现了一个 "神奇" 的 app 之后，准备搞清楚它的原理；
    
2.  首先是进行了初步的猜测: 是否监听了自身的安装广播。但在动手验证之后发现并不是；
    
3.  接着通过 debug 法，发现原来是`com.android.providers.blockednumber`进程调用了`getContentProvider`获取`com.fg.account.kp.provider`的实例时，从而间接启动了进程；
    
4.  当我们准备 debug `com.android.providers.blockednumber`时却发现在 running app list 没有这个进程；
    
5.  经查看它 apk 的 AndroidManifest.xml 文件发现原来是进程名改为`android.process.acore`了；
    
6.  但当我们试图进一步查看反编译之后的 class 代码时，居然没有找到先前 debug 时调用堆栈的那些类；
    
7.  后面发现原来有好几个跟它声明了相同`sharedUserId`和`process`的其他 app；
    
8.  经过分析正确 app 的代码发现，原来只需要在 provider 的 meta-data 里面设置`"android.content.ContactDirectory"`的属性值为 true 即可；
    
9.  最后我们自己动手写了 demo 并验证通过。
    

**（以上内容仅供学习交流，不要用来干坏事噢~）**

**文章到此结束，有错误的地方请指出，谢谢大家！**