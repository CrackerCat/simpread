> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270649.htm)

> Android12 应用启动流程分析

最近因为一些需求，需要梳理 Android 应用的启动链路，从中寻找一些稳定的注入点来实现某些特殊功能。本文即为对应用端启动全过程的一次代码分析记录。

> 注: 本文所分析的代码基于 AOSP android_12.0.0_r14

 

目录

*   [前言](#前言)
*   [应用端](#应用端)
*            [Activity](#activity)
*            [Instrumentation](#instrumentation)
*            [ActivityTaskManager](#activitytaskmanager)
*   [服务端](#服务端)
*            [ActivityTaskManagerService](#activitytaskmanagerservice)
*            [ActivityStarter](#activitystarter)
*            [Task](#task)
*   [进程已存在](#进程已存在)
*            [Transaction](#transaction)
*            Launch Activity
*   [启动新进程](#启动新进程)
*            [AMS](#ams)
*            [Zygote](#zygote)
*            [ZygoteServer](#zygoteserver)
*            [子进程初始化](#子进程初始化)
*            启动 Activity
*   [后记](#后记)

前言
==

之前的文章介绍过 [Android 操作系统的启动流程](https://evilpan.com/2020/11/08/android-init/)，从 init 进程开始，一直到 zygote 和 system_server，有助于我们去理解 Android 和 Linux 之间的异同。不过，用户和开发者最直观的感知是 UI、Framework 和四大组件，这些组件虽然大部分由 Java 代码构成，但对其分析的流程也是类似的，核心在于识别出跨进程的 RPC 调用，以及接口和继承引出的多态调用。

应用端
===

做过应用开发的对下面这句应该不会陌生:

```
context.startActivity(new Intent(context, SomeActivity.class));

```

这经常用来启动一个新的界面 (Activity)，可以是当前应用的，也可以是其他应用的:

```
Intent intent = new Intent();
intent.setComponent(new ComponentName("com.android.settings", "com.android.settings.DeviceAdminSettings"));
context.startActivity(intent);

```

其中`Context.startActivity`是个虚函数，Activity 本身也是 Context 的子类，也实现了该方法。

Activity
--------

在 **frameworks/base/core/java/android/app/Activity.java** 中调用路径如下:

*   startActivity(intent)
*   startActivity(intent, null)
*   startActivityForResult(intent, -1)
*   startActivityForResult(intent, -1, null)

关键代码:

```
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
    // ...
    Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(this, mMainThread.getApplicationThread(), mToken, this, intent, requestCode, options);
    if (ar != null) {
        mMainThread.sendActivityResult(mToken, mEmbeddedID, requestCode, ar.getResultCode(), ar.getResultData());
    }
}

```

最终调用了 Instrumentation 的 execStartActivity。当然，启动应用的方法有很多，比如:

*   startActivity[ForResult] - 开发者所能接触到的接口，启动一个应用 (Activity)，可以选择一个回调接收启动的结果；
*   startActivityAsUser[ForResult] - 以指定用户的身份启动应用，这是个隐藏接口，只对预装的应用开放；
*   startActivityAsCaller[ForResult] - 以调用者 (即启动本当前 Activity 的 APP) 的身份启动应用，也是个隐藏接口，主要用于 resolver 和 chooser Activity；

这些接口的实现都和上面代码类似，即最终都会调用到 Instrumentation 的接口。

Instrumentation
---------------

Instrumentation 类主要作用是实现代码追踪的一个收口，当启用跟踪时，该类会在 APP 代码之前被实例化，允许用户监控系统与应用之间的所有交互。代码追踪的配置见 `AndroidManifest.xml` 的 **instrumentation** 参数。

 

execStartActivity 主要有两个作用:

1.  更新所有激活的 ActivityMonitor 对象，调用它们的 onStartActivity 回调以及更新对应的计数信息；
2.  将当前调用分发到 system activity manager 中；

简略的伪代码如下:

```
    @UnsupportedAppUsage
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
  // 1. 更新 ActivityMonitor
  for (int i = 0; i < mActivityMonitors.size(); i++) {
    ActivityMonitor am = mActivityMonitors.get(i);
    am.onStartActivity(intent);
  }
  // 2. 调用 System Activity Manager
  int result = ActivityTaskManager.getService().startActivity();
  checkStartActivityResult(result, intent);
}

```

在代码注释中提到我们可以覆写此方法来观察 APP 启动新 Activity 的事件，并且可以修改启动 Activity 的行为和返回。许多插件化框架就是通过修改此方法来实现 Activity 的拦截。

ActivityTaskManager
-------------------

由于我们关心的是实际的启动链路，因此暂时忽略监控相关的代码，直接进入到 ActivityTaskManager (简称 ATM)。其 getService 返回的实际上是:

```
final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE); // activity_task
return IActivityTaskManager.Stub.asInterface(b);

```

是这里是个常见的远程调用模式:

```
// 1. 获取原始的IBinder对象
IBinder b = ServiceManager.getService("service_name");
// 2. 转换为 Service 的 RPC 接口
IXXInterface in = IXXInterface.Stub.asInterface(b);

```

所以返回的是一个类型为`IActivityTaskManager`的对象，这其实是一个 Service，即 activity_task，其接口为:

```
package android.app;
/**
 * System private API for talking with the activity task manager that handles how activities are
 * managed on screen.
 *
 * {@hide}
 */
public interface IActivityTaskManager extends android.os.IInterface
{
  /** Default implementation for IActivityTaskManager. */
  public static class Default implements android.app.IActivityTaskManager
  {
    //...
  }
  /** Local-side IPC implementation stub class. */
  public static abstract class Stub extends android.os.Binder implements android.app.IActivityTaskManager
  {
    //...
  }
  public static final java.lang.String DESCRIPTOR = "android.app.IActivityTaskManager";
  public int startActivity(...);
}

```

这个类是自动生成的，即使用 AIDL 生成的 RPC 接口，细节可以参考 [Android HAL 与 HIDL 开发笔记](https://evilpan.com/2020/11/01/hidl-basics/)，HIDL 的实现和 AIDL 类似，区别在于 HIDL 面向 **H**ardware，AIDL 面向 **A**pplication。

 

其定义文件为: `frameworks/base/core/java/android/app/IActivityTaskManager.aidl`:

```
interface IActivityTaskManager {
    int startActivity(in IApplicationThread caller, in String callingPackage,
            in String callingFeatureId, in Intent intent, in String resolvedType,
            in IBinder resultTo, in String resultWho, int requestCode,
            int flags, in ProfilerInfo profilerInfo, in Bundle options);
  // ...
}

```

既然到了 AIDL，那在 APP 进程中也就走到头了，下一步需要转换视角，从 RPC 的另一端继续分析。

服务端
===

根据 AIDL 的原理，RPC 的另一端需要实现 `IActivityTaskManager` 接口，一般来说是要继承 `IActivityTaskManager.Stub`，搜索可以发现这个类为 `ActivityTaskManagerService`。

ActivityTaskManagerService
--------------------------

该类在 AOSP 中的路径为: `frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java`，看名称猜测这是一个系统服务。笔者在之前的文章 ([Android 用户态启动流程分析](https://evilpan.com/2020/11/08/android-init/)) 中介绍了从 init 到 zygote 到 system_server 的一套流程，而这个 `ActivityTaskManagerService` 正好也是在 startBootstrapServices 中启动的其中一个服务，代码片段如下:

```
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    // ...
 
            // Activity manager runs the show.
    t.traceBegin("StartActivityManager");
    // TODO: Might need to move after migration to WM.
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mWindowManagerGlobalLock = atm.getGlobalLock();
    t.traceEnd();
 
      // ...
}

```

随之启动的还有 ActivityManagerService。

> 在旧版本中实际上是在 ActivityManagerNative 中调用 ActivityManagerService，不过新版本已经将其 startActivity 标记为过时了。

 

回到代码，ActivityTaskManagerService 的 startActivity 调用经过了以下链路:

*   startActivity
*   startActivityAsUser(..., userId=UserHandle.getCallingUserId())
*   startActivityAsUser(..., validateIncomingUser=true)

后者调用的代码如下:

```
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    assertPackageMatchesCallingUid(callingPackage);
    enforceNotIsolatedCaller("startActivityAsUser");
 
    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");
 
    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setUserId(userId)
            .execute();
 
}

```

通过一系列 builder 获取 starter 并设置对应参数，最终执行 execute。

ActivityStarter
---------------

obtainStarter 返回的类型是 ActivityStarter， 该类定义在: `frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java`，其 execute 函数的作用就是根据前面提供的的请求参数并开始启动应用。

 

关键调用路径:

*   **exexute**: 处理 Activity 启动请求的接口；
*   **executeRequest**: 执行一系列权限检查，对于合法的请求才继续；
*   **startActivityUnchecked**: 调用该方法时表示大部分初步的权限检查已经完成，执行 Trace，以及异常处理；
*   **startActivityInner**: 启动 Activity，并更新全局的 task 栈帧信息；

在 Android 中系统维护了所有应用的状态信息，因此用户才可以在不同应用中无缝切换和返回。同时在处理启动应用请求的时候还需要进行额外的判断，比如当前栈顶是否是同样的 Activity，如果是则根据设置决定是否重复启动等等。

 

忽略 ActivityStack、WindowContainer 等任务窗口管理的代码，只关注其中真正启动应用相关的:

*   mTargetRootTask.startActivityLocked()
*   mRootWindowContainer.resumeFocusedTasksTopActivities()

mTargetRootTask 是 Task 类型， `Task` 是 **WindowContainer** 的子类，用于管理和表示属于同一栈帧的所有 activity，其中每个 activity 使用 **ActivityRecord** 表示。mRootWindowContainer 是 **RootWindowContainer** 类型，也是 WindowContainer 的子类，特别代表根窗口。

 

startActivityLocked 方法的主要作用是判断当前 activity 是否可见以及是否需要为其新建 Task，根据不同情况将 ActivityRecord 加入到对应的 Task 栈顶中。

 

resumeFocusedTasksTopActivities 方法正如其名字一样，将所有聚焦的 Task 的所有 Activity 恢复运行，因为有些刚加入的 Activity 是处于暂停状态的。

Task
----

Task 从宏观来说是一系列 Activity 界面的集合，根据先进后出的栈结构进行组织。注意 task 和 进程 / 线程是不同的概念，大多数 task 可以认为是从桌面点击某个应用开始启动，随着不断点击深入打开其他界面，使对应的 Activity 入栈，在点击返回时将当前 Activity 出栈并销毁，如下所示:

 

![](https://bbs.pediy.com/upload/attach/202112/844554_98A6SQ6QG2KA5E9.jpg)

 

同时，整个 Task 本身也可以被移动当后台，比如当用户点击 HOME 键时，此时 Task 中的所有 Activity 都会停止。Task 中的 Activity 可以同属于一个 APP，也可能属于不同的 APP 和进程。在 Android R (11) 以及 Android S(12) beta 的代码中 (甚至更早的代码之前)，Task 类实际上是 ActivityStack，可以认为 **Task 就是 ActivityStack，ActivityStack 就是 Task**。关于 Task 的更多介绍，可以阅读官方的文档: [Tasks and the back stack](https://developer.android.com/guide/components/activities/tasks-and-back-stack)。

 

resumeFocusedTasksTopActivities 中主要是判断传入的 targetRootTask 是否等于当前栈顶的 Task，不管是否相等，后续都是调用栈顶 Task 的 **resumeTopActivityUncheckedLocked** 方法。

 

其中对 Task 进行了一次判断，如果是非叶子结点，则对所有子结点递归调用本方法，递归结束 (即到达叶子结点) 后才继续实际流程。再次执行了一系列的判断，比如查找当前栈帧中最接近栈顶且可显示和可聚焦的 Activity (next)，判断其暂停状态以及所属用户的权限等，然后进入到 resumeTopActivityInnerLocked。

 

该方法中主要是寻找合适的 ActivityRecord、设置 resume 条件、准备启动目标 Activity。最后，来到了我们的关键逻辑:

```
@GuardedBy("mService")
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
      ActivityRecord next = topRunningActivity(true /* focusableOnly */);
    // ...
      if (next.attachedToProcess()) {
        // Activity 已经附加到进程，恢复页面并更新栈
    } else {
        // Whoops, need to restart this activity!
                    mTaskSupervisor.startSpecificActivity(next, true, true);
    }
}

```

在函数末尾处判断当前 (栈顶) Activity 是否与已有的进程关联，如果已经关联，就在该进程中恢复页面。否则就需要 (重新) 启动目标 Activity。启动新进程是在 startSpecificActivity 中，属于 TaskSupervisor。这是个垃圾类，里面有各种不同功能的代码 (Google 原话)，因此我们不需要对其过多关注。

```
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);
 
    boolean knownToBeDead = false;
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }
 
        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }
 
    r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
 
    final boolean isTop = andResume && r.isTopRunningActivity();
    mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
}

```

startSpecificActivity 主要是判断目标 Activity 所在的应用是否正在运行。如果已经在运行就直接启动，否则就启动新进程。下面我们分别对这两种情况进行分析。

进程已存在
=====

如果待启动的 Activity 所属的 Application 已经在运行中，那么只需要在其对应进程启动 Activity，此时所走的分支是 realStartActivityLocked。该函数的核心是创建事务并分发给生命周期管理器进行处理。

```
// Create activity launch transaction.
final ClientTransaction clientTransaction = ClientTransaction.obtain(
    proc.getThread(), r.appToken);
 
final ActivityLifecycleItem lifecycleItem;
if (andResume) {
    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
} else {
    lifecycleItem = PauseActivityItem.obtain();
}
clientTransaction.setLifecycleStateRequest(lifecycleItem);
 
// Schedule transaction.
mService.getLifecycleManager().scheduleTransaction(clientTransaction);

```

Transaction
-----------

事务 (Transaction) 是包含一系列消息的容器，这些消息最终可能会传送到客户端中。其中包含一组回调以及最终的生命周期状态。setLifecycleStateRequest 用于指定客户端的终态，即事务执行完毕后客户端应当处于的最终状态，其参数类型是 ActivityLifecycleItem，可以理解为发送给客户端的请求。

 

这里的 mService 是 ActivityTaskManagerService，是一个主要用于管理 activity 及其容器 (task、display 等) 的系统服务；getLifecycleManager 返回 ClientLifecycleManager。但这都不重要，因为最终也只是执行了 `transaction.schedule()` 开始调度事务，主要按照以下顺序执行:

1.  客户端调用 preExecute，触发所有需要在真正调度事务前执行完毕的工作；
2.  发送事务的 message 信息到客户端；
3.  客户端调用 TransactionExecutor.execute，执行所有回调以及必要的生命周期事务；

```
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}

```

mClient 为 IApplicationThread 类型，这是个抽象接口，同时也是远程调用接口，其 AIDL 定义在: _frameworks/base/core/java/android/app/IApplicationThread.aidl_ 文件中。因此，最终的调用又会回到 RPC 的实现端，即应用程序所在的客户端，实现类为 **ActivityThread**。其中通过异步的 Handler 分发和调度事件，最终在 执行线程 (主线程) 的 handleMessage 回调中执行服务端传过来的 Transaction。流程大致如下图所示:

```
sequenceDiagram
%% realStartActivityLocked
participant CT as ClientTransaction
participant IA as IApplicationThread
participant AT as ActivityThread
participant H as H (Handler)
participant TE as TransactionExecutor
CT ->> IA: execute
IA ->> AT: scheduleTransaction
AT --> IA: preExecute
AT ->> H: sendMessage
H --> AT: handleMessage (EXECUTE_TRANSACTION)
H ->> TE: execute
TE --> H: executeCallbacks
TE --> H: cycleToPath
TE --> H: performLifecycleSequence
TE ->> AT: handleLaunchActivity(注1)
Note right of AT: performLaunchActivity

```

![](https://bbs.pediy.com/upload/attach/202112/844554_S79YVMG4SZ3YCH9.jpg)

 

值得一提的是，TransactionExecutor 中执行了 mTransactionHandler.handleLaunchActivity，而 mTransactionHandler 是在 TransactionExecutor 的构造函数中传入的，构造正是在 ActivityThread 中:

```
class ActivityThread {
  // ...
  private final TransactionExecutor mTransactionExecutor = new TransactionExecutor(this);
}

```

因此，注 1 中实际调用的是 `ActivityThread` 的 **handleLaunchActivity**。

Launch Activity
---------------

ActivityThread 中的 **performLaunchActivity** 是启动 Activity 的核心实现。其主要伪代码为:

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ContextImpl appContext = createBaseContextForActivity(r);
      java.lang.ClassLoader cl = appContext.getClassLoader();
      activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
      Application app = r.packageInfo.makeApplication(false, mInstrumentation);
      // loadLabel
      // initialize Activity resources
      // activity.attach(appContext, ...)
      mInstrumentation.callActivityOnCreate(activity, ...);
}

```

长话短说，主要流程为:

1.  创建应用上下文 (Context)，获取 ClassLoader；
2.  创建 Activity 对象，实质上是 classLoader.loadClass(name).newInstance()，这里会对 Activity 类进行初始化，调用对象的 `<cinit>` 方法，从而执行目标类里 **static block** 中的代码；
3.  根据应用的 **AndroidManifest.xml** 创建 Application 对象，并调用其 onCreate 回调；
4.  调用对应 Activity 的 onCreate 回调；

调用时序如下图所示:

```
sequenceDiagram
%% performLaunchActivity
participant AT as ActivityThread
participant IN as Instrumentation
participant AF as AppcomonentFactory
participant LA as LoadApk
participant AC as Activity
AT ->> IN: newActivity
IN ->> AF: makeApplication
IN --> AF: instantiateActivity
AT ->> LA: makeApplication
LA ->> IN: callApplicationOnCreate
AT ->> IN: callActivityOnCreate
IN ->> AC: performCreate
Note right of AC: onCreate

```

![](https://bbs.pediy.com/upload/attach/202112/844554_NXUTGZS5Q4JNZ87.jpg)

 

最终，调用到开发者熟知的生命周期函数 **Activity.onCreate**，开始执行 APP 本身的代码。

 

值得一提的是上面的代码都在 APP 的主线程中执行，**Application.onCreate** 仅在应用初次启动时调用一次，并且早于任意 Activity/Service/Receiver 执行，不过 ContentProvider 是个例外。

 

关于 Android 应用的生命周期，可以参考: [The Activity Lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)，一个简略的流程如下图所示:

 

![](https://bbs.pediy.com/upload/attach/202112/844554_E6FZHA42Q2A9QEM.jpg)

启动新进程
=====

分析完了进程已经存在的情况下启动应用 Activity 的流程，现在再翻回头看看进程不存在的情况。此时需要通过 **mService.startProcessAsync** 去启动进程。

AMS
---

mService 也就是前面提到的 **ActivityTaskManagerService**(ATMS)，启动进程通过异步发送消息的方式发送给 AMS，再由其执行启动进程的操作。

```
sequenceDiagram
%% startProcessAsync
participant ATMS as ActivityTaskManagerService
participant H as H(ATMS$H)
participant AMS as ActivityManagerService
participant PL as ProcessList
ATMS ->> H: sendMessage(startProcess)
Note right of H: LocalService
H ->> AMS: startProcess
H --> AMS: startProcessLocked
AMS ->> PL: startProcessLocked
AMS --> PL: startProcess

```

![](https://bbs.pediy.com/upload/attach/202112/844554_Q4QH8PKJ7ZFMN88.jpg)

 

startProcessLocked 中关键的伪代码如下:

```
ProcessRecord startProcessLocked(String processName, ...) {
  ProcessRecord app = getProcessRecordLocked(processName, info.uid);
  final String entryPoint = "android.app.ActivityThread";
  // ...
  startProcess(hostingRecord, entryPoint, app, ...)
}

```

记住这里的 entryPoint 字符串，值为 **android.app.ActivityThread**，后面会用到。

 

ProcessRecord 中包含特定运行中进程的所有信息，在进程启动完成后，再通过 handleProcessStartedLocked 来填充动态的进程 ID 等内容。

 

**startProcess** 根据入参确定实际的启动进程方式:

```
private Process.ProcessStartResult startProcess(...) {
  // mount App data ...
  // data isolation ...
  if (hostingRecord.usesWebviewZygote()) {
        startResult = startWebView(entryPoint, app.processName, uid, ...);
  } else if (hostingRecord.usesAppZygote()) {
        final AppZygote appZygote = createAppZygoteForProcessIfNeeded(app);
        startResult = appZygote.getProcess().start(entryPoint, app.processName, uid, ...);
  } else {
        regularZygote = true;
            Process.start(entryPoint, app.processName, uid, ...);
  }
}

```

可以看到新进程都是从 zygote 衍生的，但实际上有三种不同类型的 zygote:

1.  [regular zygote](https://android.googlesource.com/platform/system/sepolicy/+/master/private/zygote.te): 即常规的 zygote32/zygote64 进程，是所有 Android Java 应用的父进程；
2.  [app zygote](https://android.googlesource.com/platform/system/sepolicy/+/master/private/app_zygote.te): 应用 zygote 进程，与常规 zygote 创建的应用相比受到更多限制；
3.  [webview zygote](https://android.googlesource.com/platform/system/sepolicy/+/master/private/webview_zygote.te): 辅助 zygote 进程，用于创建 isolated_app 进程来渲染不可信的 web 内容，具有最为严格的安全限制；

Zygote
------

对于我们的分析场景，只需要关注常规 zygote 创建进程的情况，其他模式也是大同小异。根据上述代码，创建进程的请求最终会进入到 Process.start，这是个静态函数，主要工作是封装新进程的启动参数 (进程名、UID、GID、appDataDir 等信息) 为字符串，并通过 zygoteWriter 发送给 zygote 进程，通知 zygote 启动新的进程并返回对应的新进程 ID，图示如下:

```
sequenceDiagram
%% Process.start
participant P as Process
participant ZP as ZygoteProcess
participant Z as Zygote
P ->> ZP: start
ZP --> P: startViaZygote
ZP ->> Z: zygoteSendArgsAndGetResult
ZP --> Z: zygoteWriter.write(msgStr);
Note right of ZP: Note right of Z: 启动进程
Z ->> ZP: pid = zygoteInputStream.readInt() 
```

![](https://bbs.pediy.com/upload/attach/202112/844554_C4CCRB86DECKAJ4.jpg)

 

上图中的 Tunnel 是 AMS 进程与 zygote 进程的通信桥梁，二者实质上通过 UNIX socket 进行通信:

```
// android/internal/os/Zygote.java
 
/**
 * Creates a managed LocalServerSocket object using a file descriptor
 * created by an init.rc script.  The init scripts that specify the
 * sockets name can be found in system/core/rootdir.  The socket is bound
 * to the file system in the /dev/sockets/ directory, and the file
 * descriptor is shared via the ANDROID_SOCKET_ environment
 * variable.
 */
static LocalServerSocket createManagedSocketFromInitSocket(String socketName) {
    int fileDesc;
    final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
    try {
        String env = System.getenv(fullSocketName);
        fileDesc = Integer.parseInt(env);
    } catch (RuntimeException ex) {
        throw new RuntimeException("Socket unset or invalid: " + fullSocketName, ex);
    }
 
    try {
        FileDescriptor fd = new FileDescriptor();
        fd.setInt$(fileDesc);
        return new LocalServerSocket(fd);
    } catch (IOException ex) {
        throw new RuntimeException(
            "Error building socket from file descriptor: " + fileDesc, ex);
    }
} 
```

socket 名称通过环境变量传递，key 为 `“ANDROID_SOCKET_” + “zygote"`。

ZygoteServer
------------

前面说了 Zygote 进程收到 AMS 发送的数据后会启动进程并将新进程的 PID 返回，而这些流程正是在 ZygoteServer 中实现的:

```
// frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
class ZygoteServer {
    /**
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     * @param abiList list of ABIs supported by this zygote.
     */
    Runnable runSelectLoop(String abiList) {
           ArrayList socketFDs = new ArrayList<>();
          socketFDs.add(mZygoteSocket.getFileDescriptor());
 
        while (true) {
          pollFDs = new StructPollfd[socketFDs.size()];
          pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
          // ...
          while (--pollIndex >= 0) {
                if ((pollFDs[pollIndex].revents & POLLIN) == 0) continue;
                  ZygoteConnection connection = peers.get(pollIndex);
                // 处理收到的命令，并且根据需要执行 fork，注意该系统调用会返回两次
                 Runnable command = connection.processCommand(this, multipleForksOK);
                if (mIsForkChild) {
                    // We're in the child
              } else {
                                    // We're in the server
              }
                // ...
          }
        }
    }
} 
```

处理和执行 Socket 信息的流程如下图所示:

```
sequenceDiagram
%% ZygoteServer
participant AMS as ActivityManagerService
participant Z as Zygote
participant ZS as ZygoteServer
participant ZC as ZygoteConnection
AMS ->> ZS: write(msgStr)
ZS ->> ZC: processCommand
ZC ->> Z: forkAndSpecialize
ZC --> Z: nativeForkAndSpecialize
Note right of Z: fork(2)
ZC --> Z: handleChildProc
Z ->> AMS: child PID

```

![](https://bbs.pediy.com/upload/attach/202112/844554_YD5Q9ECX64D2VNN.jpg)

 

在 fork 完成后，父进程根据传入的参数设置好子进程的名称、UID 等属性，再将子进程 PID 返回给 AMS，完成新进程的启动流程。当然，其中存在一些细节的优化，比如用 pre-fork (同时 fork N 个进程，监听同一个 socket fd，当收到消息的时候，只有一个进程会被唤醒，来处理这个消息) 机制加快应用启动速度，感兴趣的可以自行探索。

 

在 ZygoteConnection fork 完成后，父进程中的返回会执行 handleParentProc，子进程的返回会执行 handleChildProc。该函数返回的是一个 Runnable 对象，包含子进程待执行的代码。

子进程初始化
------

对于子进程，调用链路为:

*   handleChildProc
*   ZygoteInit.zygoteInit
    
*   RuntimeInit.applicationInit
    
*   RuntimeInit.findStaticMain

```
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
    VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);
    final Arguments args = new Arguments(argv);
    // The end of of the RuntimeInit event (see #zygoteInit).
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    // Remaining arguments are passed to the start class's static main
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}

```

最终调用的实际上是 startClass 的 `static main(argv[])` 方法。还记得前文中提到的 entryPoint 吗？传到此处的 startClass 正是 entryPoint 的值，即 **android.app.ActivityThread**。因此，子进程的入口是 ActivityThread 的 main 方法，该方法的调用链路如下图所示:

```
sequenceDiagram
%% ActivtyThread.main
participant AT as ActivtyThread
participant AM as ActivtyManager
participant H as H
participant I as Instrumentation
Note left of AT: main
AT ->> AM: attachApplication (RPC)
Note left of AM: 设置 APP 初始化状态
AM ->> AT: bindApplication (RPC)
AT ->> H: sendMessage(H.BIND_APPLICATION)
Note right of H: handleBindApplication
H ->> I: onCraete
H ->> I: callApplicationOnCreate
Note left of AT: looper.Loop()

```

![](https://bbs.pediy.com/upload/attach/202112/844554_J2Z36N78M95GRB7.jpg)

 

首先通过 RPC 进程间通信调用到 AMS 服务端，设置一些初始化状态 (如调试状态、启动状态、OOM 优先级等) 后，又通过 RPC 回调到客户端，虽然兜了一圈，但这也是为了实现权限分离，让 AMS 专职负责管理系统应用状态。

 

回到客户端后，将应用信息封装发送给主线程去执行启动流程，这些都在 handleBindApplication 方法中实现，摘取一些关键的代码片段如下:

```
private void handleBindApplication(AppBindData data) {
  // 将当前 UI 线程注册为 JavaVM 的重要线程
    VMRuntime.registerSensitiveThread();
  // 设置调试的跟踪栈深度
  String property = SystemProperties.get("debug.allocTracker.stackDepth");
  VMDebug.setAllocTrackerStackDepth(Integer.parseInt(property));
  // 设置应用的真正名称
  Process.setArgV0(data.processName);
  android.ddm.DdmHandleAppName.setAppName(data.processName, ...);
  VMRuntime.setProcessPackageName(data.appInfo.packageName);
  // 为 ART 设置应用数据的路径
  VMRuntime.setProcessDataDirectory(data.appInfo.dataDir);
  // 如果设置了调试模式，会等待调试器连接，同时显示弹窗信息
  if (data.debugMode != ApplicationThreadConstants.DEBUG_OFF) {
      Debug.changeDebugPort(8100);
    mgr = ActivityManager.getService();
    mgr.showWaitingForDebugger(mAppThread, true);
    Debug.waitForDebugger();
    mgr.showWaitingForDebugger(mAppThread, false);
  }
  // 创建目标 APP 的上下文
  ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
  // 设置 HTTP 代理
  final ConnectivityManager cm = appContext.getSystemService(ConnectivityManager.class);
  Proxy.setHttpProxyConfiguration(cm.getDefaultProxy());
  // 加载 Network Security Config: APP 自定义证书、SSL-Pinning ...
  NetworkSecurityConfigProvider.install(appContext);
  // Applicaiton 初始化以及 ContentProvider 注册
  app = data.info.makeApplication(data.restrictedBackupMode, null);
  ActivityThread.updateHttpProxy(app)；
  installContentProviders(app, data.providers);
  // 启动应用
  mInstrumentation.onCreate(data.instrumentationArgs);
  mInstrumentation.callApplicationOnCreate(app);
  // 预加载字体资源 ...
}

```

可以看到，如果应用启用了调试，那么调试器在 Application 启动之前初始化，而且在应用启动之前还设置了系统的的 HTTP 代理，这也是为什么在 Android 中 native 进程不使用系统代理，因为对于代理是在 ActivityThread 即 Java 应用的 UI 线程中进行初始化的。

 

Instrumentation 类是我们的老朋友了，在上一节进程已存在情况下启动 Activity 就是通过该类最终进入 Activity 的生命周期函数入口。如果读者仔细看了前一节，应该还记得这么一句话:

> **Application.onCreate** 仅在应用初次启动时调用一次，并且早于任意 Activity/Service/Receiver 执行，不过 ContentProvider 是个例外。

 

从上面的代码我们也能看到，ContentProvider 的注册还在 callApplicationOnCreate 之前，这其实属于历史遗留问题，因为有时候 Application 的初始化过程会访问 ContentProvider，感兴趣的可以参考早期的讨论 ([issue#36917845](https://issuetracker.google.com/issues/36917845#comment4))。

 

所以如果我们想要自己的代码尽早执行，可以将其放到 ContentProvider.onCreate 方法中，比如实现一些野路子的加固方案，不过 Android 并没有对这个时序做应用层的承诺就是了。

启动 Activity
-----------

此时，APP 进程已经启动，但似乎 Activity 还没开始？其实前文中我们执行到了 ActivityThread 的 main 函数，其中最后会调用 Looper.loop() 进入循环等待，也是在这里收到 AMS 的 **BIND_APPLICATION** 请求从而进行子进程初始化。除了这个请求，还有一个前面我们见过的请求: **EXECUTE_TRANSACTION**，所以在子进程初始化后，AMS 会进而完成 Activity 的启动处理。

 

之后的流程就和前面进程已存在情况的场景一致了，我们这里就不再分析代码，而是通过动态调试去验证。首先编写一个测试应用，然后在 MainActivity 的 onCreate 方法打上断点，运行之后可以得到下面的栈回溯信息:

```
onCreate:13, MainActivity (com.evilpan.vulnapp)
performCreate:8000, Activity (android.app)
performCreate:7984, Activity (android.app)
callActivityOnCreate:1309, Instrumentation (android.app)
performLaunchActivity:3404, ActivityThread (android.app)
handleLaunchActivity:3595, ActivityThread (android.app)
execute:85, LaunchActivityItem (android.app.servertransaction)
executeCallbacks:135, TransactionExecutor (android.app.servertransaction)
execute:95, TransactionExecutor (android.app.servertransaction)
handleMessage:2066, ActivityThread$H (android.app)
dispatchMessage:106, Handler (android.os)
loop:223, Looper (android.os)
main:7660, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:592, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:947, ZygoteInit (com.android.internal.os)

```

和我们前面的分析基本吻合。至此，应用就完成了漫长的启动流程。

后记
==

对于 Android 应用启动流程，网上已经有很多[相关的分析](https://blog.csdn.net/Luoshengyang/article/details/6689748)，但自己实际看一遍代码才真正理解实际的执行细节。应用启动看似一件简单的事，却在操作系统中跨越多个线程甚至进程，存在许多异步的操作，比如 AIDL/RCP 调用，跨线程的 Handler sendMessage/handleMessage 模式，了解这些设计模式有助于我们顺利跟踪数据流和控制流，避免在庞杂的代码中迷失。

[【公告】看雪团队招聘 CTF 安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

[#系统底层](forum-4-1-2.htm)