> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [gist.github.com](https://gist.github.com/5ec1cff/b2a88d23c3d5f40a2c23fc785e80fc5f)

> 在 Android 11+ 中开启 data 和 Android/data 隔离. GitHub Gist: instantly share code, notes, and snippets.

探测包名是 Android 应用安全措施的重要方法，而手段也多种多样。常规的通过 PMS 查询包名的方法已经有 [HMA](https://github.com/Dr-TSNG/Hide-My-Applist) 帮忙隐藏，且随着 Android 版本更新也逐渐[限制应用任意地获取包名](https://developer.android.com/about/versions/11/privacy/package-visibility?hl=zh-cn)（虽然可以声明需要查询的类型），但还有一种更阴险的方式，就是利用 /data/data 目录或 /sdcard/Android/data 的漏洞——它们下面的目录都是以包名命名的应用数据目录，尽管这两个目录不能直接列出，但是任何应用都具有这个目录的 x 权限（否则无法访问自己的数据目录），因此如果已知需要探测的包名，就可以通过 stat 等系统调用判断目录是否存在，进而确定包名的存在（成功或 Permission Denied 都表明文件存在）。这种方法并不能通过简单的 hook 防止（尤其是绕过库函数直接 syscall 的情况）。

不过 Android 11 实际上还引入了一些措施来防止包名的泄露，那就是通过文件系统层面的措施，将 data 和 Android/data 下的包名直接隔离，每个应用的进程基本上只能看到属于自己 uid 的数据目录。

检查下面两个 prop ：

```
getprop persist.zygote.app_data_isolation
getprop persist.sys.vold_app_data_isolation_enabled


```

如果返回值都是 1 ，那么恭喜你，你的系统可能已经开启了 data 和 Android/data 隔离。不过还没完，因为此时它可能并不如你想象的那样正常工作，我们还是在源码中探索一下内部细节：

先看看 Zygote ：

```
// frameworks/base/core/java/com/android/internal/os/Zygote.java
    static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
            int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
            int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir,
            boolean isTopApp, String[] pkgDataInfoList, String[] allowlistedDataInfoList,
            boolean bindMountAppDataDirs, boolean bindMountAppStorageDirs)

```

这里末尾有两个参数 `bindMountAppDataDirs`, `bindMountAppStorageDirs` ，看起来就和这两个目录相关， `allowlistedDataInfoList` 可能就是它们的白名单。

一路往上跟踪，来到 ProcessList.startProces ，在这里我们可以看到参数的来历：

```
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java
            Map<String, Pair<String, Long>> pkgDataInfoMap;
            Map<String, Pair<String, Long>> allowlistedAppDataInfoMap;
            boolean bindMountAppStorageDirs = false;
            boolean bindMountAppsData = mAppDataIsolationEnabled
                    && (UserHandle.isApp(app.uid) || UserHandle.isIsolated(app.uid))
                    && mPlatformCompat.isChangeEnabled(APP_DATA_DIRECTORY_ISOLATION, app.info);

            // Get all packages belongs to the same shared uid. sharedPackages is empty array
            // if it doesn't have shared uid.
            final PackageManagerInternal pmInt = mService.getPackageManagerInternal();
            final String[] sharedPackages = pmInt.getSharedUserPackagesForPackage(
                    app.info.packageName, app.userId);
            final String[] targetPackagesList = sharedPackages.length == 0
                    ? new String[]{app.info.packageName} : sharedPackages;

            pkgDataInfoMap = getPackageAppDataInfoMap(pmInt, targetPackagesList, uid);
            if (pkgDataInfoMap == null) {
                // TODO(b/152760674): Handle inode == 0 case properly, now we just give it a
                // tmp free pass.
                bindMountAppsData = false;
            }

            // Remove all packages in pkgDataInfoMap from mAppDataIsolationAllowlistedApps, so
            // it won't be mounted twice.
            final Set<String> allowlistedApps = new ArraySet<>(mAppDataIsolationAllowlistedApps);
            for (String pkg : targetPackagesList) {
                allowlistedApps.remove(pkg);
            }

            allowlistedAppDataInfoMap = getPackageAppDataInfoMap(pmInt,
                    allowlistedApps.toArray(new String[0]), uid);
            if (allowlistedAppDataInfoMap == null) {
                // TODO(b/152760674): Handle inode == 0 case properly, now we just give it a
                // tmp free pass.
                bindMountAppsData = false;
            }

            int userId = UserHandle.getUserId(uid);
            StorageManagerInternal storageManagerInternal = LocalServices.getService(
                    StorageManagerInternal.class);
            if (needsStorageDataIsolation(storageManagerInternal, app)) {
                // We will run prepareStorageDirs() after we trigger zygote fork, so it won't
                // slow down app starting speed as those dirs might not be cached.
                if (pkgDataInfoMap != null && storageManagerInternal.isFuseMounted(userId)) {
                    bindMountAppStorageDirs = true;
                } else {
                    // Fuse is not mounted or inode == 0,
                    // so we won't mount it in zygote, but resume the mount after unlocking device.
                    app.setBindMountPending(true);
                    bindMountAppStorageDirs = false;
                }
            }

            // If it's an isolated process, it should not even mount its own app data directories,
            // since it has no access to them anyway.
            if (app.isolated) {
                pkgDataInfoMap = null;
                allowlistedAppDataInfoMap = null;
            }

                startResult = Process.start(/* ... */,
                        allowlistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                        new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()});

```

其中 bindMountAppStorageDirs 和 mVoldAppDataIsolationEnabled 相关，bindMountAppsData 和 mAppDataIsolationEnabled 相关

```
    private boolean needsStorageDataIsolation(StorageManagerInternal storageManagerInternal,
            ProcessRecord app) {
        final int mountMode = app.getMountMode();
        return mVoldAppDataIsolationEnabled && UserHandle.isApp(app.uid)
                && !storageManagerInternal.isExternalStorageService(app.uid)
                // Special mounting mode doesn't need to have data isolation as they won't
                // access /mnt/user anyway.
                && mountMode != Zygote.MOUNT_EXTERNAL_ANDROID_WRITABLE
                && mountMode != Zygote.MOUNT_EXTERNAL_PASS_THROUGH
                && mountMode != Zygote.MOUNT_EXTERNAL_INSTALLER
                && mountMode != Zygote.MOUNT_EXTERNAL_NONE;
    }

        mAppDataIsolationEnabled =
                SystemProperties.getBoolean(ANDROID_APP_DATA_ISOLATION_ENABLED_PROPERTY, true);
        mVoldAppDataIsolationEnabled = SystemProperties.getBoolean(
                ANDROID_VOLD_APP_DATA_ISOLATION_ENABLED_PROPERTY, false);


```

它们都和系统属性相关：

```
    // A system property to control if app data isolation is enabled.
    static final String ANDROID_APP_DATA_ISOLATION_ENABLED_PROPERTY =
            "persist.zygote.app_data_isolation";

    // A system property to control if obb app data isolation is enabled in vold.
    static final String ANDROID_VOLD_APP_DATA_ISOLATION_ENABLED_PROPERTY =
            "persist.sys.vold_app_data_isolation_enabled";

```

那么只要把这两个属性都设置成 true 就行了？我们来试试：

```
setprop persist.zygote.app_data_isolation 1
setprop persist.sys.vold_app_data_isolation_enabled 1


```

persist 属性可以在重启后保留，因此直接 setprop 即可。

重启后可以观察到 Android/data 的隔离生效了，无法通过 stat 找到 /sdcard/Android/data 下自己以外的包名（stat 对应包名路径提示文件不存在），但是 /data/data 下的似乎还没生效，仍然能得到 stat 结果。此处测试了三个终端：终端模拟器 (API 22), JuiceSSH(API 29), MT 管理器 (API 26) ，得到的结果一致。

注意到 `bindMountAppsData` 为 true 还需要 `mPlatformCompat.isChangeEnabled(APP_DATA_DIRECTORY_ISOLATION, app.info);` 为 true 。经过跟踪可以发现，这里的值实际上取自 `/etc/compatconfig` 。

详细源码在此：

```
frameworks/base/services/core/java/com/android/server/compat/PlatformCompat.java
frameworks/base/services/core/java/com/android/server/compat/CompatConfig.java
frameworks/base/services/core/java/com/android/server/compat/CompatChange.java


```

APP_DATA_DIRECTORY_ISOLATION 是一个常量 `143937733` ，在 `/etc/compatconfig/services-platform-compat-config.xml` 中可以找到：

```
<compat-change description="Apps have no access to the private data directories of any other app, even if the other app has made them world-readable." enableAfterTargetSdk="29"  />


```

看来要 API 29 以上（Android 11 中这里的取法是 > 29）才会进行 data 隔离，不过我们可以用 Magisk 模块修改掉这里，也就是用一个 `enableAfterTargetSdk="0"` 的文件替换掉。

这样替换之后果然对于任何 target API 的 App 都会进行 data 隔离了。

最后简单来研究一下隔离的原理，大部分都是在 zygote 中利用 bind mount 实现的。

`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`

data 隔离实际上就是给 /data/data 挂载了一层 tmpfs ，然后在上面创建需要的目录，把原先的 app 数据目录绑定挂载上去。

不过有人会问了，既然 /data/data 已经挂了一层 tmpfs ，就看不到里面的数据目录了，还怎么把这些目录 bind mount 呢？

所以 Android 11 开始根目录就多了一个 /data_mirror 目录，`/data_mirror/cur_profiles` ，里面是 /data/data 的镜像挂载。这个目录的权限是 700 ，所属者是 root ，因此 App 没法通过这里检测包名的存在，而 zygote 可以从这里找到原先的数据目录并进行 bind mount 。

实际上过程相当复杂，具体实现可以参考源码中的函数 isolateAppData ，，除了 /data/data 是传统的 CE (凭据加密) 存储目录要隔离，还有 /data/user 多用户的数据目录，/data/user_de 设备加密存储目录，/mnt/expand 扩展存储目录。并且对于 CE 存储还要考虑开机处于加密状态，需要通过 inode 而不是路径名字查找目录。甚至还有 isolateJitProfile 函数，处理了 `/data/misc/profiles/cur` 下泄露包名的问题。看来 Android 在这方面比我们要想得周到得多。

类似地，Android/data 目录也是通过 bind mount 隔离的，函数 BindMountStorageDirs 进行了这个工作。

不过需要注意的是，不同的 mount_external 类型会挂载不同的 /mnt 下的目录到 /storage ，这些目录有的给 Android/data 目录 bind mount 了在 /data/media 下的原始目录（如 /mnt/androidwritable），有些则没有（如 /mnt/installer），这种情况下也不会启用 bind mount 隔离 Android/data ，不过仍然能起到隔离的效果。推测是这时候通过 fuse 访问 Android/data ，由 fuse 处理了隔离。在这种情况下，su 之后就能看到 Android/data 的内容。