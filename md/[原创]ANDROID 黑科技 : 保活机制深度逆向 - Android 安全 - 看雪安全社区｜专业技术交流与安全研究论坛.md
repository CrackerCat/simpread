> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-290728.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

在 Android 逆向与安全防护的博弈中，进程保活（Keep-Alive）始终是一个充满争议且技术密集的话题。随着 Android 系统的迭代，从早期的 `1 像素 Activity`、`JobScheduler`，到后来的各种同步账号机制，系统对后台进程的容忍度越来越低。

本文将以某头部大厂 APP 中的保活模块（`libundead_native_ability_q.so`）为例，深度剖析其在 Android 10+ (API 29 及以上) 环境下，如何利用 C/C++ 层的双重 Fork 逃逸、Flock 文件锁监控以及纯 Native 层的 Binder 穿透技术，实现高隐蔽性、高存活率的 “不死” 机制。

通过对 APK 的初步分析，我们定位到核心的保活入口位于包 `com.bytedance.platform.ka` 下。针对高版本 Android 系统，应用采用了分层策略，其中针对 Android 10+ 的核心类为 `KaAbilityQ`。

查看其反编译代码：

```
package com.bytedance.platform.ka.ability.q.KaAbilityQ;

public class KaAbilityQ extends 09xQ implements 09xS {
    // 指定加载的 SO 库名称
    public void KaAbilityQ(Application p0, 09xO p1){
       super(p0, p1);
       this.LIZLLL = "undead_native_ability_q"; 
    }

    // 核心 JNI 方法声明
    private native int doKaOnNative(IBinder p0, long[] p1, String p2, long p3, 
                                    String p4, String p5, String p6, String p7, 
                                    String p8, boolean p9, String p10, String p11, 
                                    int p12, int p13);

    // JAVA 调用接口
    public final int LIZ(IBinder p0,long[] p1,String p2,long p3,String p4,String p5,String p6,String p7,String p8,boolean p9,String p10,String p11,int p12,int p13){
       this.LIZIZ();
       if (this.LIZJ != null) {
          UnDeadLog.d("KaAbilityQ", "library load success,invoke doKaOnNative");
          return this.doKaOnNative(p0, p1, p2, p3, p4, p5, p6, p7, p8, p9, p10, p11, p12, p13);
       }else {
          UnDeadLog.e("KaAbilityQ", "library load failed,not doKA");
          return -1;
       }
    }

}


```

从这里可以看出，Java 层主要负责环境准备和参数收集（如当前进程名、各种开关 Flag），真正的硬核逻辑全部通过 `doKaOnNative` 交给了 `libundead_native_ability_q.so` 处理。

在 ServiceImpl 的 LIZIZ 函数中进行了参数初始化并调用了 LIZ 函数。

```
ServiceImpl platform = serviceImpl.platform;
IBinder binder = serviceImpl.o.getBinder();
ServiceImpl g = serviceImpl.g;
ServiceImpl mInstrConfig = serviceImpl.mInstrConfig;
mInstrConfig.getClass();
String str4 = StringBuilderCache.release(StringBuilderCache.get() + mInstrConfig.LJI() + "/enable.flag");
mInstrConfig = serviceImpl.mInstrConfig;
mInstrConfig.getClass();
String str5 = StringBuilderCache.release(StringBuilderCache.get() + mInstrConfig.LJI() + "/top.flag");
mInstrConfig = serviceImpl.mInstrConfig;
mInstrConfig.getClass();
StringBuilderCache.release(StringBuilderCache.get() + mInstrConfig.LJI() + "/mcomm");

if ((iKADepend = serviceImpl.mInstrConfig.LIZ()) != null) {
    daemonProces = iKADepend.getDaemonProcessName();
    if (!TextUtils.isEmpty(daemonProces)) {
        String packageName = serviceImpl.mInstrConfig.LIZIZ.getPackageName();
        b = serviceImpl.mIKADepend.useNativeMode();
        if ((config1 = serviceImpl.mInstrConfig.LIZ.getConfig("instr")) != null && !config1.isEmpty()) {
            str = config1;
        }
        int i1 = platform.LJ.LIZ(binder, g, str1, l2, str4, str5, p1, daemonProces, packageName, b, str, ToolUtils.LJIIIZ(platform.LIZ), startInstrum, serviceImpl.mIKADepend.transactFlags());
    }
}


```

将 SO 拖入 IDA Pro 进行分析，定位到导出函数 `Java_com_bytedance_platform_ka_ability_q_KaAbilityQ_doKaOnNative`，内部调用了真实的实现逻辑 `do_ka`。

该 SO 的核心保活流转经历了三个精妙的阶段：

为了防止系统在杀死主应用时，顺藤摸瓜将子进程 “一锅端”，该组件在 Native 层实现了经典的 Double Fork 逃逸：

```
signal(17, (__sighandler_t)((char * ) & dword_0 + 1));
v27 = fork();
if (!v27) {
    if (a3) {
        v28 = _JNIEnv -> functions -> GetLongArrayElements(_JNIEnv, a3, 0 LL);
        v29 = v28;
        if (v28 && * v28 && v28[1]) {
            v30 = ((__int64( * )(void)) * v28)();
            ((void(__fastcall * )(__int64,
                const char * )) v29[1])(v30, v46);
        }
        v31 = inited;
        prctl(PR_SET_NAME, v46, 0 LL, 0 LL, 0 LL);
        if (fork())
            goto LABEL_42;
    } else {
        v31 = inited;
        v29 = 0 LL;
        if (fork())
            goto LABEL_42;
    }

    // ...

    LABEL_42:
        _exit(0);
}


```

1.  主进程调用 `fork()` 创建**子进程 1**。
2.  **子进程 1** 立即再次调用 `fork()` 创建**孙子进程**（即未来的守护进程）。
3.  **子进程 1** 随即调用 `_exit(0)` 主动退出。
4.  **孙子进程**由于生父死亡，瞬间变为**孤儿进程**，被系统的 `init` 进程（PID 为 1）接管，成功脱离原 APP 的进程组（Process Group）。

_（注：与部分保活方案使用 Pipe 管道阻塞监控不同，这里脱离进程树是为了避免被 ActivityManagerService (AMS) 的 `ProcessRecord.kill()` 连坐查杀。）_

脱离进程组后，守护进程需要一种极低功耗的方式来感知主进程的生死。轮询 `/proc/` 目录显然太耗电且容易被查杀。

```
while (1) {
    fd_2 = open(file, O_RDWR | O_CREAT, 432 LL);
    if (fd_2 >= LOCK_SH) {
        fd = fd_2;
        if (flock(fd_2, (ENUM_LOCK) 6) < 0) {
            v34 = close(fd);
            if ( * (_DWORD * ) __errno(v34) == 11) {
                stream_1 = fopen(filename, "rw");
                if (stream_1) {
                    stream = stream_1;
                    remove(filename);
                    fclose(stream);
                    fflush(stream);
                }
                fd_1 = open(file, O_RDWR | O_CREAT, 432 LL);
                if (fd_1 >= LOCK_SH)
                    flock(fd_1, LOCK_EX);
                if (a5) {
                    if (gettimeofday( & tv, 0 LL))
                        v38 = 1;
                    else
                        v38 = 1000 * tv.tv_sec + tv.tv_usec / 0x3E8 uLL < a5;
                } else {
                    v38 = 0;
                }
                v39 = access(name, F_OK);
                v40 = access(name_1, F_OK);
                if (!v38 && !v39 && v40) {
                    if (!a11)
                        return 0;
                    start_instr();
                }
                LABEL_42:
                    _exit(0);
            }
        } else {
            flock(fd, LOCK_UN);
            close(fd);
        }
    }
    usleep(0x3D0900 u);
}


```

该组件巧妙地利用了 Linux 的 `flock` (File Lock) 机制：

*   主应用在启动时，对一个特定的本地文件（如 `.ka_monitor`）进行加锁 (`LOCK_EX`)。
*   守护进程在其死循环中，尝试对同一个文件进行 `flock`。
*   由于互斥锁的存在，守护进程会被内核挂起阻塞（或者 `usleep` 轮询），不占用 CPU 资源。
*   **一旦主进程被系统 Kill (OOM 或用户划掉)，Linux 内核会强制回收主进程持有的所有文件句柄，该文件锁被瞬间释放。**
*   守护进程的 `flock` 立刻返回成功，从而精准捕获主进程的死亡事件。

这是该方案最为硬核的部分。当守护进程发现主进程死亡后，如果通过常规的 `am start` 命令行去拉起，不仅速度慢，而且极易被高版本 Android 的后台拦截机制阻断。

逆向伪代码显示，该 SO 直接引入了 NDK 的 `AIBinder` API，徒手拼接底层 IPC 数据包（Parcel）：

```
__int64 __fastcall init_instr_internal(__int64 kaAbility,const char * s,const char * s_1,const char * s_2,int a5,int a6) {
    unsigned int v12; // w22
    unsigned int DataPosition; // w24
    unsigned int DataPosition_1; // w23

    if ((unsigned int) AIBinder_prepareTransaction(kaAbility, & ::DataPosition)) {
        return 0;
    } else {
        v12 = 1;
        AParcel_writeInt32();
        strlen(s);
        AParcel_writeString();
        strlen(s_1);
        AParcel_writeString();
        AParcel_writeString();
        AParcel_writeInt32();
        AParcel_writeInt32();
        DataPosition = AParcel_getDataPosition(::DataPosition);
        AParcel_writeInt32();
        AParcel_writeInt32();
        AParcel_getDataPosition(::DataPosition);
        AParcel_writeInt32();
        __strlen_chk("instrumentation_type", 0x15 uLL);
        AParcel_writeString();
        AParcel_writeInt32();
        __strlen_chk("instrumentation_type_ka", 0x18 uLL);
        AParcel_writeString();
        __strlen_chk("source_process", 0xF uLL);
        AParcel_writeString();
        AParcel_writeInt32();
        strlen(s_2);
        AParcel_writeString();
        __strlen_chk("use_native_mode", 0x10 uLL);
        AParcel_writeString();
        AParcel_writeInt32();
        AParcel_writeInt32();
        DataPosition_1 = AParcel_getDataPosition(::DataPosition);
        AParcel_setDataPosition(::DataPosition, DataPosition);
        AParcel_writeInt32();
        AParcel_setDataPosition(::DataPosition, DataPosition_1);
        AParcel_writeStrongBinder(::DataPosition, 0 LL);
        AParcel_writeStrongBinder(::DataPosition, 0 LL);
        AParcel_writeInt32();
        AParcel_writeString();
        dword_410C0 = a5;
        unk_410C4 = a6;::kaAbility = kaAbility;
        byte_410D8 = 1;
    }
    return v12;
}


```

```
bool start_instr(void)
{
  return byte_410D8 && (unsigned int)AIBinder_transact(kaAbility, (unsigned int)dword_410C0, &DataPosition) == 0;
}


```

守护进程提前在内存中组装好了一通向 `ActivityManagerService` 发送 `startInstrumentation` 事务的 Parcel 包。触发时，直接调用 `AIBinder_transact` 将伪造的请求发给 AMS。系统接收到请求后，会主动分配进程资源，拉起该 APP 注册的自定义 `Instrumentation`。应用随之在 `callApplicationOnCreate` 中完成复苏。

守护进程通过 `AIBinder_transact` 向 AMS 发起 `startInstrumentation` 请求，其底层完整的调用链涉及客户端 Binder 代理与服务端实现两个关键环节。

**客户端代理实现**（`android.app.IActivityManager` 的 Stub 代理）：

```
@Override // android.app.IActivityManager
public boolean startInstrumentation(ComponentName componentName, String str, int i, Bundle bundle, IInstrumentationWatcher iInstrumentationWatcher, IUiAutomationConnection iUiAutomationConnection, int i2, String str2) throws RemoteException {
    Parcel parcelObtain = Parcel.obtain(asBinder());
    Parcel parcelObtain2 = Parcel.obtain();
    try {
        parcelObtain.writeInterfaceToken(Stub.DESCRIPTOR);
        parcelObtain.writeTypedObject(componentName, 0);
        parcelObtain.writeString(str);
        parcelObtain.writeInt(i);
        parcelObtain.writeTypedObject(bundle, 0);
        parcelObtain.writeStrongInterface(iInstrumentationWatcher);
        parcelObtain.writeStrongInterface(iUiAutomationConnection);
        parcelObtain.writeInt(i2);
        parcelObtain.writeString(str2);
        this.mRemote.transact(51, parcelObtain, parcelObtain2, 0);
        parcelObtain2.readException();
        return parcelObtain2.readBoolean();
    } finally {
        parcelObtain2.recycle();
        parcelObtain.recycle();
    }
}


```

**服务端真实处理**（`ActivityManagerService.java`）：

```
public boolean startInstrumentation(ComponentName className,
        String profileFile, int flags, Bundle arguments,
        IInstrumentationWatcher watcher, IUiAutomationConnection uiAutomationConnection,
        int userId, String abiOverride) {
    enforceNotIsolatedCaller("startInstrumentation");
    final int callingUid = Binder.getCallingUid();
    final int callingPid = Binder.getCallingPid();
    userId = mUserController.handleIncomingUser(callingPid, callingUid,
            userId, false, ALLOW_FULL_ONLY, "startInstrumentation", null);
    // Refuse possible leaked file descriptors
    if (arguments != null && arguments.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Bundle");
    }
    final IPackageManager pm = AppGlobals.getPackageManager();
    synchronized(this) {
        InstrumentationInfo ii = null;
        ApplicationInfo ai = null;
        boolean noRestart = (flags & INSTR_FLAG_NO_RESTART) != 0;
        try {
            ii = pm.getInstrumentationInfoAsUser(className, STOCK_PM_FLAGS, userId);
            if (ii == null) {
                reportStartInstrumentationFailureLocked(watcher, className,
                        "Unable to find instrumentation info for: " + className);
                return false;
            }
            ai = pm.getApplicationInfo(ii.targetPackage, STOCK_PM_FLAGS, userId);
            if (ai == null) {
                reportStartInstrumentationFailureLocked(watcher, className,
                        "Unable to find instrumentation target package: " + ii.targetPackage);
                return false;
            }
        } catch (RemoteException e) {
        }
        if (ii.targetPackage.equals("android")) {
            if (!noRestart) {
                reportStartInstrumentationFailureLocked(watcher, className,
                        "Cannot instrument system server without 'no-restart'");
                return false;
            }
        } else if (!ai.hasCode()) {
            reportStartInstrumentationFailureLocked(watcher, className,
                    "Instrumentation target has no code: " + ii.targetPackage);
            return false;
        }
        int match = SIGNATURE_NO_MATCH;
        try {
            match = pm.checkSignatures(ii.targetPackage, ii.packageName, userId);
        } catch (RemoteException e) {
        }
        if (match < 0 && match != PackageManager.SIGNATURE_FIRST_NOT_SIGNED) {
            if (Build.IS_DEBUGGABLE && (callingUid == Process.ROOT_UID)
                    && (flags & INSTR_FLAG_ALWAYS_CHECK_SIGNATURE) == 0) {
                Slog.w(TAG, "Instrumentation test " + ii.packageName
                        + " doesn't have a signature matching the target " + ii.targetPackage
                        + ", which would not be allowed on the production Android builds");
            } else {
                String msg = "Permission Denial: starting instrumentation "
                        + className + " from pid="
                        + Binder.getCallingPid()
                        + ", uid=" + Binder.getCallingUid()
                        + " not allowed because package " + ii.packageName
                        + " does not have a signature matching the target "
                        + ii.targetPackage;
                reportStartInstrumentationFailureLocked(watcher, className, msg);
                throw new SecurityException(msg);
            }
        }
        if (!Build.IS_DEBUGGABLE && callingUid != ROOT_UID && callingUid != SHELL_UID
                && callingUid != SYSTEM_UID && !hasActiveInstrumentationLocked(callingPid)) {
            // If it's not debug build and not called from root/shell/system uid, reject it.
            final String msg = "Permission Denial: instrumentation test "
                    + className + " from pid=" + callingPid + ", uid=" + callingUid
                    + ", pkgName=" + mInternal.getPackageNameByPid(callingPid)
                    + " not allowed because it's not started from SHELL";
            Slog.wtfQuiet(TAG, msg);
            reportStartInstrumentationFailureLocked(watcher, className, msg);
            throw new SecurityException(msg);
        }
        boolean disableHiddenApiChecks = ai.usesNonSdkApi()
                || (flags & INSTR_FLAG_DISABLE_HIDDEN_API_CHECKS) != 0;
        boolean disableTestApiChecks = disableHiddenApiChecks
                || (flags & INSTR_FLAG_DISABLE_TEST_API_CHECKS) != 0;
        if (disableHiddenApiChecks || disableTestApiChecks) {
            enforceCallingPermission(android.Manifest.permission.DISABLE_HIDDEN_API_CHECKS,
                    "disable hidden API checks");
        }
        if ((flags & ActivityManager.INSTR_FLAG_INSTRUMENT_SDK_SANDBOX) != 0) {
            return startInstrumentationOfSdkSandbox(
                    className,
                    profileFile,
                    arguments,
                    watcher,
                    uiAutomationConnection,
                    userId,
                    abiOverride,
                    ii,
                    ai,
                    noRestart,
                    disableHiddenApiChecks,
                    disableTestApiChecks,
                    (flags & ActivityManager.INSTR_FLAG_INSTRUMENT_SDK_IN_SANDBOX) != 0);
        }
        ActiveInstrumentation activeInstr = new ActiveInstrumentation(this);
        activeInstr.mClass = className;
        String defProcess = ai.processName;;
        if (ii.targetProcesses == null) {
            activeInstr.mTargetProcesses = new String[]{ai.processName};
        } else if (ii.targetProcesses.equals("*")) {
            activeInstr.mTargetProcesses = new String[0];
        } else {
            activeInstr.mTargetProcesses = ii.targetProcesses.split(",");
            defProcess = activeInstr.mTargetProcesses[0];
        }
        activeInstr.mTargetInfo = ai;
        activeInstr.mProfileFile = profileFile;
        activeInstr.mArguments = arguments;
        activeInstr.mWatcher = watcher;
        activeInstr.mUiAutomationConnection = uiAutomationConnection;
        activeInstr.mResultClass = className;
        activeInstr.mHasBackgroundActivityStartsPermission = checkPermission(
                START_ACTIVITIES_FROM_BACKGROUND, callingPid, callingUid)
                        == PackageManager.PERMISSION_GRANTED;
        activeInstr.mHasBackgroundForegroundServiceStartsPermission = checkPermission(
                START_FOREGROUND_SERVICES_FROM_BACKGROUND, callingPid, callingUid)
                        == PackageManager.PERMISSION_GRANTED;
        activeInstr.mNoRestart = noRestart;
        final long origId = Binder.clearCallingIdentity();
        ProcessRecord app;
        synchronized (mProcLock) {
            if (noRestart) {
                app = getProcessRecordLocked(ai.processName, ai.uid);
            } else {
                // Instrumentation can kill and relaunch even persistent processes
                forceStopPackageLocked(ii.targetPackage, -1, true, false, true, true, false,
                        false, userId, "start instr");
                // Inform usage stats to make the target package active
                if (mUsageStatsService != null) {
                    mUsageStatsService.reportEvent(ii.targetPackage, userId,
                            UsageEvents.Event.SYSTEM_INTERACTION);
                }
                app = addAppLocked(ai, defProcess, false, disableHiddenApiChecks,
                        disableTestApiChecks, abiOverride, ZYGOTE_POLICY_FLAG_EMPTY);
                app.mProfile.addHostingComponentType(HOSTING_COMPONENT_TYPE_INSTRUMENTATION);
            }
            app.setActiveInstrumentation(activeInstr);
            activeInstr.mFinished = false;
            activeInstr.mSourceUid = callingUid;
            activeInstr.mRunningProcesses.add(app);
            if (!mActiveInstrumentation.contains(activeInstr)) {
                mActiveInstrumentation.add(activeInstr);
            }
        }
        if ((flags & INSTR_FLAG_DISABLE_ISOLATED_STORAGE) != 0) {
            // Allow OP_NO_ISOLATED_STORAGE app op for the package running instrumentation with
            // --no-isolated-storage flag.
            mAppOpsService.setMode(AppOpsManager.OP_NO_ISOLATED_STORAGE, ai.uid,
                    ii.packageName, AppOpsManager.MODE_ALLOWED);
        }
        Binder.restoreCallingIdentity(origId);
        if (noRestart) {
            instrumentWithoutRestart(activeInstr, ai);
        }
    }
    return true;
}


```

以上两段代码完整展示了从客户端通过 `Binder` 事务码 `51` 发起 `startInstrumentation` 请求，到服务端 `ActivityManagerService` 进行权限校验、签名匹配、进程强制停止（或复用）以及 `ActiveInstrumentation` 注册的完整流程。守护进程正是利用这一底层通道，在主进程死亡后伪造合法请求，触发 AMS 重新分配进程并拉起自定义 `Instrumentation`，从而实现隐蔽唤醒。

该厂的保活方案代表了目前 Android 端企服 / 核心业务模块在进程驻留方面的一流水平。**“Double Fork 逃避进程树监控 + Flock 零功耗挂起 + Native Binder 直接穿透 AMS”** 的组合拳，巧妙绕过了 Java 层层层加码的系统限制。

面对未来 API 36 (Android 16) 更加严苛的 `Foreground Service` 限制和广播冻结机制，此类纯底层触发 `Instrumentation` 的方式是否还能如鱼得水？系统是否会在内核层面切断非信任进程的 Binder 节点访问权？这些都是非常值得持续跟进和逆向分析的研究方向。

[[培训]Windows 内核深度攻防：从 Hook 技术到 Rootkit 实战！](https://www.kanxue.com/book-section_list-220.htm)

最后于 3 小时前 被易之生生编辑 ，原因：