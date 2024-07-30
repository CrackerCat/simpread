> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282677.htm)

> [原创]Android 应用启动过程源码分析

这是我在看雪的第二篇文章，内容是 Android 系统下应用启动的流程，涉及从 Lanucher 的点击到应用的 MainActivity.onCreate 的全流程。

上一篇文章被加精了，也收获了很多前辈和同行者的认可，感觉受宠若惊，看到了部分前辈的补充，也第一时间进行了补习。现在来到第二篇，希望能给各位同好带来点帮助，也希望各位前辈不吝赐教，因为我也是刚开始研究，不免有错误。

环境

```
源码版本
android-7.0.0_r1
  
链接：https://pan.baidu.com/s/1yla9fqd4EbxSBSemrYVjsA?pwd=ukvt
提取码：ukvt
--来自百度网盘超级会员V3的分享

```

前言
==

其实我觉得学习应用启动流程最应该好好读是这一个部分，也就是前言这个部分。

整个启动流程还是非常庞大的，虽然下图看起来并不是非常复杂，但是其实隐去了很多在启动过程当中的细节，例如：Zygote 如何 fork 一个新进程，Instrumentation 在启动过程当中有什么作用，各进程之间使用什么对象通信且如何获得整个对象，应用信息何时被加载等等问题。**这些都是需要阅读源码并且耐心看资料才能搞明白的，这也作证应用启动流程的复杂性。**

虽然它庞大且复杂，但我崇尚的观点是 **只要把复杂的整体给分化细化成小的部分，并对小的部分作出作用性的解释， 就能好好消化下来**。（我把作用性解释理解成更多偏向于更加笼统地对代码块整体作用解释。与之相对应的是描述型解释是我以前使用的，需要对以代码为单位做解释，这种解释方法不太适合启动流程的复杂主题的学习。在我看来更适合小规模的逆向，和一些关键部分的分析。）

在我自己的学习过程中，运用这种观点可以使得源码学习和研究得到真正效能地提升。而在编写这篇文章帮助一些源码初学者理解这部分的时候，我也大体采用这种思路：**大整体分化成小部分，大部分小部分作出作用性的解释，小部分关键代码作描述性的解释（如 5.2，6.2）**。

而前言这部分，其实主要的作用是**介绍我如何将大整体分化成小部分**，建立知识模型，并不断破坏重塑扩展这个模型，这在我看是首要的。同时，因为篇幅还是比较长的，读的时候可能会有点不知道目前处于哪一个步骤（也可能是我写的有点流水账）, 所以建议多回头看看前言，不至于迷失。

下面是我理解下的整体框架和主要功能，涉及的部分例如（2.1-2.13）就是本篇文章对应部分。（可能不是很完整，笔者目前也是初入源码研究，形成的理解还不到位，大家做个借鉴就可以，可以多参考几篇博客，形成自己的理解，常学常新。）

```
用户触发启动流程
    (1.1)用户点击应用图标:Launcher处理点击的事件。
系统服务处理启动请求
    (1.2-1.3)启动请求传递:准备Intent，获取接口。
    (2.1-2.13)AMS处理请求:处理中手机整理App的基本信息,让Launcher先Paused
进程创建与初始化
    (3.1-3.8,3.13)进程创建:主要涉及到zygote如何fork进程，以及fork之后怎么做。
    (3.9-3.12)应用进程初始化:目标进程相关的初始化
App进程Binder通信对象传递
    (4.1-5.4)通信对象传递到ActivityManagerService:然后对app的部分成员进行初始化
AMS传递信息给App进程
    (6.1-6.4)传递信息给App进程：告诉它一切准备就绪，可以真正执行Activity的启动操作，也是来到了oncreate

```

冒号后面的总结并不完善，这部分**建议看正文**。

而下面是应用启动的流程经典的结构图，里面标记了启动所需的四个进程和关键的步骤。其实大体上也对应了我写这篇文章的思路，所以我也是按照这张图，进行**拆解**，写下这篇的笔记的。

![](https://bbs.kanxue.com/upload/attach/202407/993634_3SEC9F7AMKMFYTG.jpg)

参考资料

```
https://blog.csdn.net/luoshengyang/article/details/6689748/
https://blog.csdn.net/shift_wwx/article/details/85626448
https://blog.csdn.net/wcsbhwy/article/details/105965932
https://blog.csdn.net/mafei852213034/article/details/117049651
https://blog.csdn.net/mba16c35/article/details/137226452
https://www.cnblogs.com/linhaostudy/p/18277752
https://blog.csdn.net/gh1026385964/article/details/80615453
https://cloud.tencent.com/developer/article/1005769
https://www.jianshu.com/p/91197047f780
https://www.cnblogs.com/Robin132929/p/12166563.html
https://blog.csdn.net/qq_30993595/article/details/82747738
https://lishuaiqi.top/2016/03/19/Process2-theCreateOfProcess/#3-3-1-3-1-Daemons-start
https://ly669966.top/2019/09/03/Android9.0%E6%A0%B9activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/
《Android框架解密》
文心一言

```

省略了 zygote fork 阶段的代码图

![](https://bbs.kanxue.com/upload/attach/202407/993634_EW4ZMJSP9WWZ27C.jpg)

好了，前面应该有个大体的认识了，可以进入到我们细化的小部分了。

1 Launcher 进程
=============

这块涉及这一步，从 Launcher 进入到 system_server 的 AMS。

```
总结：
1.用户在设备的桌面上点击应用程序的快捷方式，这个动作会被Launcher应用捕获。
2.准备Intent
3.获取远程接口，准备与AMS进行通信

```

![](https://bbs.kanxue.com/upload/attach/202407/993634_CBPHKGT9DYK6AG5.jpg)

1.1 Launcher.java
-----------------

对于 Launcher 我们可以看作是一个提供桌面的应用程序，上面显示着我们需要启动应用的图标，点击这个图标，Launcher 准备让这个对应的应用程序启动，而下面这个 onClick 就是入口，其中 startActivitySafely 需要继续跟。

```
public void onClick(View v) {
        // Make sure that rogue clicks don't get through while allapps is launching, or after the
        // view has detached (it's possible for this to happen if the view is removed mid touch).
        if (v.getWindowToken() == null) {
            return;
        }
 
        if (!mWorkspace.isFinishedSwitchingState()) {
            return;
        }
 
        Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            // Open shortcut
            final Intent intent = ((ShortcutInfo) tag).intent;
            int[] pos = new int[2];
            v.getLocationOnScreen(pos);
            intent.setSourceBounds(new Rect(pos[0], pos[1],
                    pos[0] + v.getWidth(), pos[1] + v.getHeight()));
 
            //重要的是这里
            boolean success = startActivitySafely(v, intent, tag);
 
            if (success && v instanceof BubbleTextView) {
                mWaitingForResume = (BubbleTextView) v;
                mWaitingForResume.setStayPressed(true);
            }
        } else if (tag instanceof FolderInfo) {
            if (v instanceof FolderIcon) {
                FolderIcon fi = (FolderIcon) v;
                handleFolderClick(fi);
            }
        } else if (v == mAllAppsButton) {
            if (isAllAppsVisible()) {
                showWorkspace(true);
            } else {
                onClickAllAppsButton(v);
            }
        }
    }

```

startActivitySafely 里面还需要继续跟 startActivity

```
boolean startActivitySafely(View v, Intent intent, Object tag) {
        ...
            success = startActivity(v, intent, tag);
        ...
        return success;
    }

```

来到这里，可以看到`intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)`，设置要在一个新的 Task 当中启动 Activity，而这个 Task 有点像任务栈，里面是一串的 Activity。

然后可以看到里面有一串判断，是用来判断有无动画，启动用户是否相同，不同的分支会进行一些不同的设置，但最后都会殊途同归。我们就假设没有动画，继续跟。

```
boolean startActivity(View v, Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
 
        try {
            // Only launch using the new animation if the shortcut has not opted out (this is a
            // private contract between launcher and may be ignored in the future).
            boolean useLaunchAnimation = (v != null) &&
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
            UserHandle user = (UserHandle) intent.getParcelableExtra(ApplicationInfo.EXTRA_PROFILE);
            LauncherApps launcherApps = (LauncherApps)
                    this.getSystemService(Context.LAUNCHER_APPS_SERVICE);
            if (useLaunchAnimation) {
                ActivityOptions opts = ActivityOptions.makeScaleUpAnimation(v, 0, 0,
                        v.getMeasuredWidth(), v.getMeasuredHeight());
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    // Could be launching some bookkeeping activity
                    startActivity(intent, opts.toBundle());
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(),
                            opts.toBundle());
                }
            } else {
                //没有动画直接走着
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent);
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(), null);
                }
            }
            return true;
        } catch (SecurityException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Launcher does not have the permission to launch " + intent +
                    ". Make sure to create a MAIN intent-filter for the corresponding activity " +
                    "or use the exported attribute for this activity. "
                    + "tag="+ tag + " intent=" + intent, e);
        }
        return false;
    }

```

这里调用的是父类 Activity 的`startActivity(Intent intent)`方法，所以我们要进入 Activity.java

1.2 Activity.java
-----------------

`frameworks/base/core/java/android/app/Activity.java`

先调用上面的，然后跟进下面的。

```
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
 
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        //之前我们假设没有动画 跟这里
        startActivityForResult(intent, -1);
    }
}

```

来到这里

```
public void startActivityForResult(Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }
//跟到下面这个
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            //这里重要
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            ...
    }

```

上面这块代码中重要的是下面这行，其中 mInstrumentation 在应用的任何代码执行前被初始化，目的是来**监控应用程序和进程的交互**。

```
Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);

```

上面的代码参数当中有一个需要注意的`mMainThread.getApplicationThread()`，这里 mMainThread 代表的是一个应用程序进程，而不是一个线程，是一个 final 的类。**当我们启动一个应用程序的时候，进程里面就夹在一个 ActivityThread 类实例**，而这个实例就保存在这个 Activity 成员变量 mMainThread 里面。

ok，那`mMainThread.getApplicationThread()`返回的是什么呢？是一个他内部的类型为 ApplicationThread 的 Binder 本地对象，也就是说这个参数传出取得是一个 Binder 对象，而这个对象恰恰就是它用来**和 AMS 进程进行通信**的，可以先理解为下图 “发送 " 二字所需要的必备接口。

![](https://bbs.kanxue.com/upload/attach/202407/993634_T5EXUXZ7NXPDAZ9.jpg)

1.3 Instrumentation.java
------------------------

`frameworks/base/core/java/android/app/Instrumentation.java`

这里进行了一些准备工作，然后就调用了 AMS 的代理对象的 startActivity 方法 (ActivityManagerNative.getDefault() 方法**返回的就是 ActivityManagerService 的代理对象**)

```
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }

```

2 AMS 进程
========

```
总结
1.处理中手机整理App的基本信息
2.task的处理
3.让Launcher先Paused、
4.通过socket把一些参数写入，发送给zygote让它进行fork进程

```

![](https://bbs.kanxue.com/upload/attach/202407/993634_EDNGFQQZT2TRJSA.jpg)

2.1 ActivityManagerService.java
-------------------------------

`ActivityManagerService.java`

```
@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
跟进去
@Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, bOptions, false, userId, null, null);
    }

```

上面的代码当中做了一些设置，然后把参数转发给 startActivityMayWait 了。

2.2 ActivityStarter.java
------------------------

这里和罗老师那片博客里面的不太一样，那里是 ActivityStarter.java

`frameworks\base\services\core\java\com\android\server\am\ActivityStarter.java`

函数逻辑比较长，我截了我觉得比较重要的部分。

```
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, IActivityManager.WaitResult outResult, Configuration config,
            Bundle bOptions, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
        // Refuse possible leaked file descriptors
        ...
        //下面这块，是解析MainActivity的信息 并收集保存在aInfo里面，就比如是是com.example.11.MainActivity 是AndroidManifest.xml里面配置的
        ...
               //下面这块是我们后面需要跟的
            int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                    aInfo, rInfo, voiceSession, voiceInteractor,
                    resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                    options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                    inTask);
 
            ...
            return res;
        }
    }

```

下面是 startActivityLocked，这里的函数更是重量级，我做了一些删减。

```
final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
            TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
 
    //首先传进来的参数caller得到调用者的进程信息，然后存放在callerApp（这里其实是Launcher的信息）
        ProcessRecord callerApp = null;
        if (caller != null) {
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            } else {
                ...
            }
        }
 
        ...
 
        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        ...
            //后面还以一块代码，确定sourceRecord和resultRecord，sourceRecord代表请求启动当前activity；后者表示需要返回结果的ActivityRecord。一般，如果sourceRecord的activity使用startActivityForResult启动当前activity并且requestCode>=0时，则resultRecord=sourceRecord
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, mSupervisor, container,
                options, sourceRecord);
        ...
           //要跟的
         err = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                    true, options, inTask);
    }

```

startActivityUnchecked

```
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {
 
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);
 
    // 要跟的
        computeLaunchingTaskFlags();
 
        computeSourceStack();
 
        ...
           
             
            if (mDoResume) {
            if (!mLaunchTaskBehind) {
                // TODO(b/26381750): Remove this code after verification that all the decision
                // points above moved targetStack to the front which will also set the focus
                // activity.
                mService.setFocusedActivityLocked(mStartActivity, "startedActivity");
            }
            final ActivityRecord topTaskActivity = mStartActivity.task.topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                // If the activity is not focusable, we can't resume it, but still would like to
                // make sure it becomes visible as it starts (this will also trigger entry
                // animation). An example of this are PIP activities.
                // Also, we don't want to resume activities in a task that currently has an overlay
                // as the starting activity just needs to be in the visible paused state until the
                // over is removed.
                mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
                // Go ahead and tell window manager to execute app transition for this activity
                // since the app transition will not be triggered through the resume channel.
                mWindowManager.executeAppTransition();
            } else {
              //要跟的 假如activity没获得焦点，无法进行resume操作。或者满足条件后开始resume。
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
    }

```

要跟是 resumeFocusedStackTopActivityLocked，下面这个跟踪分析的主线函数，但是其中内容值得说说 computeLaunchingTaskFlags

```
private void computeLaunchingTaskFlags() {
        // If the caller is not coming from another activity, but has given us an explicit task into
        // which they would like us to launch the new activity, then let's see about doing that.
        ...
 
            // If task is empty, then adopt the interesting intent launch flags in to the
            // activity being started.
            if (root == null) {
                final int flagsOfInterest = FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_MULTIPLE_TASK
                        | FLAG_ACTIVITY_NEW_DOCUMENT | FLAG_ACTIVITY_RETAIN_IN_RECENTS;
                mLaunchFlags = (mLaunchFlags & ~flagsOfInterest)
                        | (baseIntent.getFlags() & flagsOfInterest);
                mIntent.setFlags(mLaunchFlags);
                mInTask.setIntent(mStartActivity);
                mAddingToTask = true;
 
                // If the task is not empty and the caller is asking to start it as the root of
                // a new task, then we don't actually want to start this on the task. We will
                // bring the task to the front, and possibly give it a new intent.
            } else if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
                mAddingToTask = false;
 
            } else {
                mAddingToTask = true;
            }
 
            mReuseTask = mInTask;
        } else {
            mInTask = null;
            // Launch ResolverActivity in the source task, so that it stays in the task bounds
            // when in freeform workspace.
            // Also put noDisplay activities in the source task. These by itself can be placed
            // in any task/stack, however it could launch other activities like ResolverActivity,
            // and we want those to stay in the original task.
            if ((mStartActivity.isResolverActivity() || mStartActivity.noDisplay) && mSourceRecord != null
                    && mSourceRecord.isFreeform())  {
                mAddingToTask = true;
            }
        }
 
        //判断task是否为空，把flag设为New Task，不为空则判断标志位是否为New Task。
        if (mInTask == null) {
            if (mSourceRecord == null) {
                // This activity is not being started from another...  in this
                // case we -always- start a new task.
                if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0 && mInTask == null) {
                    Slog.w(TAG, "startActivity called from non-Activity context; forcing " +
                            "Intent.FLAG_ACTIVITY_NEW_TASK for: " + mIntent);
                    mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
                }
            } else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
                // The original activity who is starting us is running as a single
                // instance...  this new activity it is starting must go on its
                // own task.
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            } else if (mLaunchSingleInstance || mLaunchSingleTask) {
                //Launchmodel设定的singleInstance和newtask真正确定launchflag,这后面会根据这一步的条件选择是创建或复用task
                //对于task的复用，如果一个activity启动的时候创建了一个task，那么这个activity就是某task的root activity，其中一些属性指向这个root activity
                //还有关于后续的AMS中的mhistory查找task，后面我就不跟了，还是回到主要的跟踪
                // The activity being started is a single instance...  it always
                // gets launched into its own task.
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            }
        }
    }

```

2.3 ActivityStackSupervisor.java
--------------------------------

resumeFocusedStackTopActivityLocked 逻辑比较简单

```
boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || r.state != RESUMED) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        }
        return false;
    }

```

resumeTopActivityUncheckedLocked 是需要进去看的

2.4 ActivityStack.java
----------------------

resumeTopActivityUncheckedLocked

在这个. java 里面完成了一些基础判断，走到 pauseactivity

```
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        // 关注这一行
            result = resumeTopActivityInnerLocked(prev, options);
...
    }
 
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
        //这个函数先执行onpause，再考虑onresume。
        //关注这里
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
       ...
}
 
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        ...
            try {
               //获取来源activity prev的Binder对象，通过ipc告诉prev该pause了 EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                mService.updateUsageStats(prev, false);
               //要跟的地方 prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
           ...
    }

```

2.5 ActivityThread.java
-----------------------

内部发送了一条 message，消息的发送和处理是在 H 类中，H 是 handler 的子类且它是 ActivityThread 的一个内部类

```
public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges, boolean dontReport) {
            int seq = getLifecycleSeq();
            if (DEBUG_ORDER) Slog.d(TAG, "pauseActivity " + ActivityThread.this
                    + " operation received seq: " + seq);
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? USER_LEAVING : 0) | (dontReport ? DONT_REPORT : 0),
                    configChanges,
                    seq);
        }

```

我们找到他的 handle 方法

```
case PAUSE_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                    SomeArgs args = (SomeArgs) msg.obj;
                    handlePauseActivity((IBinder) args.arg1, false,
                            (args.argi1 & USER_LEAVING) != 0, args.argi2,
                            (args.argi1 & DONT_REPORT) != 0, args.argi3);
                    maybeSnapshot();
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;

```

先去**执行 pause 操作**，执行完毕后会通知 AMS。

经过一系列的调用最终调用了 Instrumentation.callActivityOnPause

```
private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport, int seq) {
        ActivityClientRecord r = mActivities.get(token);
        if (DEBUG_ORDER) Slog.d(TAG, "handlePauseActivity " + r + ", seq: " + seq);
        if (!checkAndUpdateLifecycleSeq(seq, r, "pauseActivity")) {
            return;
        }
        if (r != null) {
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                performUserLeavingActivity(r);
            }
 
            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(token, finished, r.isPreHoneycomb(), "handlePauseActivity");//执行pause ，从这里往下跟
 
            // Make sure any pending writes are now committed.
            if (r.isPreHoneycomb()) {
                QueuedWork.waitToFinish();
            }
 
            // Tell the activity manager we have paused.
            if (!dontReport) {
                try {
                    ActivityManagerNative.getDefault().activityPaused(token);//执行完后通知AMS当前Activity已经pause
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            }
            mSomeActivitiesChanged = true;
        }
    }
 
 
 
 
 
接着跟
final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState, reason) : null;
    }
 
调用重载 接着跟
final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState, String reason) {
        if (r.paused) {
            if (r.activity.mFinished) {
                // If we are finishing, we won't call onResume() in certain cases.
                // So here we likewise don't want to call onPause() if the activity
                // isn't resumed.
                return null;
            }
            RuntimeException e = new RuntimeException(
                    "Performing pause of activity that is not resumed: "
                    + r.intent.getComponent().toShortString());
            Slog.e(TAG, e.getMessage(), e);
        }
        if (finished) {
            r.activity.mFinished = true;
        }
 
        // Next have the activity save its current state and managed dialogs...
        if (!r.activity.mFinished && saveState) {
            callCallActivityOnSaveInstanceState(r);
        }
 
        performPauseActivityIfNeeded(r, reason);// 这里执行pause
 
        // Notify any outstanding on paused listeners
        ArrayList listeners;
        synchronized (mOnPauseListeners) {
            listeners = mOnPauseListeners.remove(r.activity);
        }
        int size = (listeners != null ? listeners.size() : 0);
        for (int i = 0; i < size; i++) {
            listeners.get(i).onPaused(r.activity);
        }
 
        return !r.activity.mFinished && saveState ? r.state : null;
    }
接着跟
private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
        if (r.paused) {
            // You are already paused silly...
            return;
        }
 
        try {
            r.activity.mCalled = false;
           //重点关注这里
        mInstrumentation.callActivityOnPause(r.activity);
            EventLog.writeEvent(LOG_AM_ON_PAUSE_CALLED, UserHandle.myUserId(),
                    r.activity.getComponentName().getClassName(), reason);
            if (!r.activity.mCalled) {
                throw new SuperNotCalledException("Activity " + safeToComponentShortString(r.intent)
                        + " did not call through to super.onPause()");
            }
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to pause activity "
                        + safeToComponentShortString(r.intent) + ": " + e.toString(), e);
            }
        }
        r.paused = true;
    } 
```

2.6 Instrumentation.java
------------------------

```
public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
     
final void performPause() {
    mDoReportFullyDrawn = false;
    mFragments.dispatchPause();
    mCalled = false;
    onPause();//调用activity的生命周期函数onPause()
    mResumed = false;
    if (!mCalled && getApplicationInfo().targetSdkVersion
            >= android.os.Build.VERSION_CODES.GINGERBREAD) {
        throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onPause()");
    }
    mResumed = false;
}

```

到现在，当前的 activity 处于了 onpause 状态，然后我们在 2.5 当中可以看到已经通知给了 AMS，调用`ActivityManagerNative.getDefault().activityPaused(token);`

2.8 ActivityStack.java
----------------------

负责 pause 完成

```
ActivityManagerService.java中的
@Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }//调用了ActivityStack得activityPausedLocked
        }
        Binder.restoreCallingIdentity(origId);
    }
 
 
final void activityPausedLocked(IBinder token, boolean timeout) {
    if (DEBUG_PAUSE) Slog.v(TAG_PAUSE,
        "Activity paused: token=" + token + ", timeout=" + timeout);
 
    final ActivityRecord r = isInStackLocked(token);
    if (r != null) {
        mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
        if (mPausingActivity == r) {
            if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                    + (timeout ? " (due to timeout)" : " (pause complete)"));
            mService.mWindowManager.deferSurfaceLayout();
            try {
                completePauseLocked(true /* resumeNext */, null /* resumingActivity */);//1 pause完成
            } finally {
                mService.mWindowManager.continueSurfaceLayout();
            }
            return;
        }
//...
}
继续跟
private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
//...
mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
 
//...
}

```

2.9 ActivityStackSupervisor.java
--------------------------------

经过一些条件的判断，但是最后还会调用 resumeTopActivityUncheckedLocked（注意这是**第二次**来到这个函数）

```
boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || r.state != RESUMED) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        }
        return false;
    }

```

2.11 ActivityStackSupervisor
----------------------------

跟到这里

getProcessRecordLocked 返回值为 null（我们没有 AndroidManifest.xml 特别设置的话，process 默认就是包名，uid+process 创建了一个 ProcessRecord）

```
ActivityStack.java中的
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {//至于为啥走到resumeTopActivityInnerLocked，就可以看2.4，是一样的，只是这边的出口不一样
        mStackSupervisor.startSpecificActivityLocked(next, true, false);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        }
 
 
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        ...
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid);
        ...
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }

```

2.12 ActivityManagerService.java
--------------------------------

经过一系列的调用重载来到下面这个 startProcessLocked

重点 Process.start

```
private final void startProcessLocked(ProcessRecord app, String hostingType,
           String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    ...
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
           Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                   app.processName);
           checkTime(startTime, "startProcess: asking zygote to start proc");
           Process.ProcessStartResult startResult = Process.start(entryPoint,
                   app.processName, uid, uid, gids, debugFlags, mountExternal,
                   app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                   app.info.dataDir, entryPointArgs);
    ...
     
     
}

```

2.13 Process.java
-----------------

看得比较舒服

在这个. java 算是 AMS 做最后的告诉 **Zygote 要初始化的操作**了

```
public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            //要跟这里，看名字可以知道终于准备到zygote了
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }
 
//跟，不是很复杂
private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
         
     
    //创建进程的参数给到argsForZygote，然后一起给zygote初始化去
    synchronized(Process.class) {
            ArrayList argsForZygote = new ArrayList();
 
            // --runtime-args, --setuid=, --setgid=,
            // and --setgroups= must go first
            argsForZygote.add("--runtime-args");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            if ((debugFlags & Zygote.DEBUG_ENABLE_JNI_LOGGING) != 0) {
                argsForZygote.add("--enable-jni-logging");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {
                argsForZygote.add("--enable-safemode");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
                argsForZygote.add("--enable-debugger");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {
                argsForZygote.add("--enable-checkjni");
            }
            if ((debugFlags & Zygote.DEBUG_GENERATE_DEBUG_INFO) != 0) {
                argsForZygote.add("--generate-debug-info");
            }
            if ((debugFlags & Zygote.DEBUG_ALWAYS_JIT) != 0) {
                argsForZygote.add("--always-jit");
            }
            if ((debugFlags & Zygote.DEBUG_NATIVE_DEBUGGABLE) != 0) {
                argsForZygote.add("--native-debuggable");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {
                argsForZygote.add("--enable-assert");
            }
            if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
                argsForZygote.add("--mount-external-default");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
                argsForZygote.add("--mount-external-read");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
                argsForZygote.add("--mount-external-write");
            }
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);
 
            //TODO optionally enable debuger
            //argsForZygote.add("--enable-debugger");
 
            // --setgroups is a comma-separated list
            if (gids != null && gids.length > 0) {
                StringBuilder sb = new StringBuilder();
                sb.append("--setgroups=");
 
                int sz = gids.length;
                for (int i = 0; i < sz; i++) {
                    if (i != 0) {
                        sb.append(',');
                    }
                    sb.append(gids[i]);
                }
 
                argsForZygote.add(sb.toString());
            }
 
            if (niceName != null) {
                argsForZygote.add("--nice-name=" + niceName);
            }
 
            if (seInfo != null) {
                argsForZygote.add("--seinfo=" + seInfo);
            }
 
            if (instructionSet != null) {
                argsForZygote.add("--instruction-set=" + instructionSet);
            }
 
            if (appDataDir != null) {
                argsForZygote.add("--app-data-dir=" + appDataDir);
            }
 
            argsForZygote.add(processClass);
 
            if (extraArgs != null) {
                for (String arg : extraArgs) {
                    argsForZygote.add(arg);
                }
            }
//zygoteSendAndGetPid 这里进一步操作
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
 
 
//看名字也可以大概看到个作用，发送fork的参数和获取结果
private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList args)
            throws ZygoteStartFailedEx {
        try {
            /**
             * See com.android.internal.os.ZygoteInit.readArgumentList()
             * Presently the wire format to the zygote process is:
             * a) a count of arguments (argc, in essence)
             * b) a number of newline-separated argument strings equal to count
             *
             * After the zygote process reads these it will write the pid of
             * the child or -1 on failure, followed by boolean to
             * indicate whether a wrapper process was used.
             */
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;
 
            //这里writer是一个socket写入流，不深挖了，暂时理解为现在已经得到所有参数，要给zygote的server端socket写数据了
            writer.write(Integer.toString(args.size()));
            writer.newLine();
 
            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                if (arg.indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx(
                            "embedded newlines not allowed");
                }
                writer.write(arg);
                writer.newLine();
            }
 
            writer.flush();
 
            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();
            result.pid = inputStream.readInt();
            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            result.usingWrapper = inputStream.readBoolean();
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    } 
```

3 Zygote 进程
===========

这部分干的事情比较多，我放在 3.8 和 3.12 的阶段性总结了。

![](https://bbs.kanxue.com/upload/attach/202407/993634_4Q68ZEQVYZ72E7C.jpg)

这里是怎么监听到 AMS 的信息的呢

如下，是之前运行了一个 runSelectLoop

![](https://bbs.kanxue.com/upload/attach/202407/993634_7J44CQ78RTDGPBY.jpg)

那我们就从 runSelectLoop 入手，看看这个函数究竟要怎么**处理 AMS 的请求**

3.1 ZygoteInit.java
-------------------

1.  将 `sServerSocket` 的文件描述符添加到 `fds` 列表中，并添加一个 `null` 到 `peers` 列表中
2.  主循环使用 `while (true)` 进行无限循环
3.  使用 `Os.poll()` 方法对 `pollFds` 进行阻塞式轮询，等待文件描述符的事件发生
4.  如果 `i == 0`，表示监听的主服务器套接字有可读事件，调用 `acceptCommandPeer(abiList)` 方法接受新的 Zygote 连接，并将新的连接和其文件描述符添加到 `peers` 和 `fds` 列表中，否则，对于其他索引 `i`，运行 `peers.get(i).runOnce()` 方法执行一次 Zygote 连接处理。如果连接处理完成 (`done` 为 true)，则从 `peers` 和 `fds` 列表中移除对应的连接和文件描述符。

所以 peers.get(i).runOnce() 就是下一步要跟的

```
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList fds = new ArrayList();
        ArrayList peers = new ArrayList();
 
        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);
 
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    } 
```

3.2 ZygoteConnection.java
-------------------------

同样代码比较长，我只留下了目前只需要的函数

1.  这里先解析客户端发送过来的参数列表
2.  然后调用 Zygote.forkAndSpecialize 方法

注意我们后面先看 **forkAndSpecialize**，后面还需回到这里的 **handleChildProc**

```
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
 
        ...
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                    parsedArgs.appDataDir);
        ...
    }
 
try {
        if (pid == 0) {
            //子进程执行
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;
            //进入子进程流程
            handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
            return true;
        } else {
            //父进程执行
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }

```

3.3 Zygote.java
---------------

1.  fork 一个新的 vm 实例， 必须使用 - Xzygote 标志启动当前 VM
2.  新实例保留所有 root 功能

```
public static int forkAndSpecialize(int uid, int gid, int[] gids, int debugFlags,
          int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
          String instructionSet, String appDataDir) {
    //先调用 VM_HOOKS.preFork().这里不跟进去了，只要知道里面停止zygote进程的四个daemon子线程，分别是ReferenceQueueDaemon，FinalizerDaemon，FinalizerWatchdogDaemon，HeapTaskDaemon，以保证zygote单线程，提高fork效率
        VM_HOOKS.preFork();
    //跟这里
        int pid = nativeForkAndSpecialize(
                  uid, gid, gids, debugFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
                  instructionSet, appDataDir);
        // Enable tracing as soon as possible for the child process.
        if (pid == 0) {
            Trace.setTracingEnabled(true);
 
            // Note that this event ends at the end of handleChildProc,
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");
        }
        VM_HOOKS.postForkCommon();
        return pid;
    }

```

3.4 com_android_internal_os_Zygote.cpp
--------------------------------------

这玩意是 native 的函数, 把 fork 的一些准备做了，可以看看代码，我做了一些注释

```
static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
        JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits,
        jint mount_external, jstring se_info, jstring se_name,
        jintArray fdsToClose, jstring instructionSet, jstring appDataDir) {
    ...
 
    return ForkAndSpecializeCommon(env, uid, gid, gids, debug_flags,
            rlimits, capabilities, capabilities, mount_external, se_info,
            se_name, false, fdsToClose, instructionSet, appDataDir);
}
 
跟
     
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jstring instructionSet, jstring dataDir) {
  //设置子进程的signal信号处理
    SetSigChldHandler();
 
#ifdef ENABLE_SCHED_BOOST
  SetForkLoad(true);
#endif
//关键的fork
  pid_t pid = fork();
 
  if (pid == 0) {
    // The child process. //进入子进程
    gMallocLeakZygoteChild = 1;
 
    // Clean up any descriptors which must be closed immediately关闭并清除文件描述符
    DetachDescriptors(env, fdsToClose);
 
    ...
 
    SetGids(env, javaGids);//设置设置group
 
    SetRLimits(env, javaRlimits);//设置资源limit
 
    if (use_native_bridge) {
      ScopedUtfChars isa_string(env, instructionSet);
      ScopedUtfChars data_dir(env, dataDir);
      android::PreInitializeNativeBridge(data_dir.c_str(), isa_string.c_str());
    }
 
    int rc = setresgid(gid, gid, gid);
    if (rc == -1) {
      ALOGE("setresgid(%d) failed: %s", gid, strerror(errno));
      RuntimeAbort(env, __LINE__, "setresgid failed");
    }
 
    rc = setresuid(uid, uid, uid);
    if (rc == -1) {
      ALOGE("setresuid(%d) failed: %s", uid, strerror(errno));
      RuntimeAbort(env, __LINE__, "setresuid failed");
    }
 
    if (NeedsNoRandomizeWorkaround()) {
        // Work around ARM kernel ASLR lossage (http://b/5817320).
        int old_personality = personality(0xffffffff);
        int new_personality = personality(old_personality | ADDR_NO_RANDOMIZE);
        if (new_personality == -1) {
            ALOGW("personality(%d) failed: %s", new_personality, strerror(errno));
        }
    }
 
    SetCapabilities(env, permittedCapabilities, effectiveCapabilities);
 
    SetSchedulerPolicy(env);////设置调度策略
 
    const char* se_info_c_str = NULL;
    ScopedUtfChars* se_info = NULL;
    if (java_se_info != NULL) {
        se_info = new ScopedUtfChars(env, java_se_info);
        se_info_c_str = se_info->c_str();
        if (se_info_c_str == NULL) {
          RuntimeAbort(env, __LINE__, "se_info_c_str == NULL");
        }
    }
    const char* se_name_c_str = NULL;
    ScopedUtfChars* se_name = NULL;
    if (java_se_name != NULL) {
        se_name = new ScopedUtfChars(env, java_se_name);
        se_name_c_str = se_name->c_str();
        if (se_name_c_str == NULL) {
          RuntimeAbort(env, __LINE__, "se_name_c_str == NULL");
        }
    }
    rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);////selinux上下文
    if (rc == -1) {
      ALOGE("selinux_android_setcontext(%d, %d, \"%s\", \"%s\") failed", uid,
            is_system_server, se_info_c_str, se_name_c_str);
      RuntimeAbort(env, __LINE__, "selinux_android_setcontext failed");
    }
 
    // Make it easier to debug audit logs by setting the main thread's name to the
    // nice name rather than "app_process".
    if (se_info_c_str == NULL && is_system_server) {
      se_name_c_str = "system_server";//设置线程名为system_server，方便调试
    }
    if (se_info_c_str != NULL) {
      SetThreadName(se_name_c_str);
    }
 
    delete se_info;
    delete se_name;
 
    UnsetSigChldHandler();
 
    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                              is_system_server, instructionSet);
    if (env->ExceptionCheck()) {
      RuntimeAbort(env, __LINE__, "Error calling post fork hooks.");
    }
  } else if (pid > 0) {
    // the parent process
 
#ifdef ENABLE_SCHED_BOOST
    // unset scheduler knob
    SetForkLoad(false);
#endif
 
  }
  return pid;
}

```

3.5 fork.cpp
------------

我们可以先看一下 zygote 进行 fork 时候的示意图

![](https://bbs.kanxue.com/upload/attach/202407/993634_BUQM372GWK4DAGG.jpg)

这是我在《Android 框架揭秘》抄来的图片，先有个大致概念。

![](https://bbs.kanxue.com/upload/attach/202407/993634_NWA2TZ2QQHB99FE.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/993634_QEH8J4VY8HV65E6.jpg)

```
int fork() {
  __bionic_atfork_run_prepare();
 
  pthread_internal_t* self = __get_thread();
 
  // Remember the parent pid and invalidate the cached value while we fork.
    //获取父进程pid
  pid_t parent_pid = self->invalidate_cached_pid();
 
#if defined(__x86_64__) // sys_clone's last two arguments are flipped on x86-64.
  int result = syscall(__NR_clone, FORK_FLAGS, NULL, NULL, &(self->tid), NULL);
#else
  int result = syscall(__NR_clone, FORK_FLAGS, NULL, NULL, NULL, &(self->tid));
#endif
  if (result == 0) {
    self->set_cached_pid(gettid());
    __bionic_atfork_run_child();
  } else {
    self->set_cached_pid(parent_pid);
    __bionic_atfork_run_parent();
  }
  return result;
}

```

结束后会在 ForkAndSpecializeCommon 函数中调用 CallStaticVoidMethod。这就是**反射调用 zygote.callPostForkChildHooks()**

3.6 Zygote.java
---------------

```
private static void callPostForkChildHooks(int debugFlags, boolean isSystemServer,
            String instructionSet) {
        VM_HOOKS.postForkChild(debugFlags, isSystemServer, instructionSet);
    }
     
//跟,这里是ZygoteHooks.java的内容
public void postForkChild(int debugFlags, boolean isSystemServer, String instructionSet) {
        nativePostForkChild(token, debugFlags, isSystemServer, instructionSet);
 
    //这里负责以当前时间设置新进程随机数种子  
    Math.setRandomSeedInternal(System.currentTimeMillis());
    }

```

nativePostForkChild 通过 jni 调用到 native 的下面这个函数

```
dalvik_system_ZygoteHooks.cc中的
static void ZygoteHooks_nativePostForkChild(JNIEnv* env,
                                            jclass,
                                            jlong token,
                                            jint debug_flags,
                                            jboolean is_system_server,
                                            jstring instruction_set) {
    ////此处token是由nativePreFork()创建的，记录着当前线程
  Thread* thread = reinterpret_cast(token);
  //设置新进程的主线程id
  thread->InitAfterFork();
  EnableDebugFeatures(debug_flags);
 
  ...
      if (instruction_set != nullptr && !is_system_server) {
    ScopedUtfChars isa_string(env, instruction_set);
    InstructionSet isa = GetInstructionSetFromString(isa_string.c_str());
    Runtime::NativeBridgeAction action = Runtime::NativeBridgeAction::kUnload;
    if (isa != kNone && isa != kRuntimeISA) {
      action = Runtime::NativeBridgeAction::kInitialize;
    }
    Runtime::Current()->InitNonZygoteOrPostFork(
        env, is_system_server, action, isa_string.c_str());
  } else {
    Runtime::Current()->InitNonZygoteOrPostFork(
        env, is_system_server, Runtime::NativeBridgeAction::kUnload, nullptr);
  }
  } 
```

l 来到这里

```
void Runtime::InitNonZygoteOrPostFork(
    JNIEnv* env, bool is_system_server, NativeBridgeAction action, const char* isa) {
  is_zygote_ = false;
 
  if (is_native_bridge_loaded_) { // native bridge 相关逻辑！
    switch (action) {
      case NativeBridgeAction::kUnload:
        UnloadNativeBridge();
        is_native_bridge_loaded_ = false;
        break;
 
      case NativeBridgeAction::kInitialize:
        InitializeNativeBridge(env, isa);
        break;
    }
  }
 
  // 创建堆处理的线程池
  heap_->CreateThreadPool();
  // Reset the gc performance data at zygote fork so that the GCs
  // before fork aren't attributed to an app.
  heap_->ResetGcPerformanceInfo();
 
  // 非 SystemServer 进程会进入这里
  if (!is_system_server &&
      !safe_mode_ &&
      (jit_options_->UseJitCompilation() || jit_options_->GetSaveProfilingInfo()) &&
      jit_.get() == nullptr) {
    // 创建 JIT
    CreateJit();
  }
 
  // 设置信号处理函数
  StartSignalCatcher();
 
  // 启动 JDWP 线程，当命令行 debuger 的 flags 指定"suspend=y"时，则暂停 runtime
  Dbg::StartJdwp();
}

```

3.7 ZygoteHooks
---------------

OK, 我们现在已经成功 fork 了，现在需要回到 3.3 里面的 postForkCommon

就是重新启动我们之前的四个 Daemon 线程启动起来

```
public void postForkCommon() {
        Daemons.start();
    }

```

注意 这个方法会在父进程 Zygote 和子进程中各调用一次，也就是说，**在父进程 Zygote 中是恢复 4 个 Daemon 线程，而在子进程中，是启动 4 个 Daemon 线程**

3.8 zygote 阶段总结
---------------

```
Zygote.forkAndSpecialize
    ZygoteHooks.preFork  // 停掉 zygote 的子线程，为 fork 做准备
        Daemons.stop
        ZygoteHooks.nativePreFork
            dalvik_system_ZygoteHooks.ZygoteHooks_nativePreFork//调用Linux的fork()子进程，设置新进程的主线程id，重置gc性能数据，设置信号处理函数等功能
                Runtime::PreZygoteFork
                    heap_->PreZygoteFork()
                     
    Zygote.nativeForkAndSpecialize   // fork 阶段，该这个方法会返回 2 次
        com_android_internal_os_Zygote.ForkAndSpecializeCommon
            fork()
            Zygote.callPostForkChildHooks
                ZygoteHooks.postForkChild
                    dalvik_system_ZygoteHooks.nativePostForkChild
                        Runtime::InitNonZygoteOrPostFork
                         
    ZygoteHooks.postForkCommon    // 重新启动四个daemon子线程
        Daemons.start

```

还有一点要注意

forkAndSpecialize 执行结束，即创建新进程后，会有两次返回，**第一次**回到 ZygoteConnection.runOnce 方法，执行子进程 handleChildProc 方法；**第二次**回到该方法，执行 zygote 进程的 handleParentProc 方法

3.9 ZygoteConnection.java
-------------------------

我们回头看 3.2，是不是 fork 之后需要回调 handleChildProc 啊，ok 继续，下面主要要对子进程进行一些初始化了

1.  处理子进程的 post-fork 设置，并根据需要关闭套接字，根据需要重新打开 stdio
2.  如果成功则最终抛出 MethodAndArgsCaller，如果失败则返回。

```
private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {
        /**
         * By the time we get here, the native code has closed the two actual Zygote
         * socket connections, and substituted /dev/null in their place.  The LocalSocket
         * objects still need to be closed properly.
         */
//关闭了两个实际的Zygote socket连接 替换了/ dev / null。 LocalSocket对象仍需要正确关闭
        closeSocket();
        ZygoteInit.closeServerSocket();
 
        if (descriptors != null) {
            try {
                Os.dup2(descriptors[0], STDIN_FILENO);
                Os.dup2(descriptors[1], STDOUT_FILENO);
                Os.dup2(descriptors[2], STDERR_FILENO);
 
                for (FileDescriptor fd: descriptors) {
                    IoUtils.closeQuietly(fd);
                }
                newStderr = System.err;
            } catch (ErrnoException ex) {
                Log.e(TAG, "Error reopening stdio", ex);
            }
        }
////设置进程名
        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }
 
        // End of the postFork event.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.remainingArgs);
        } else {
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
        }
    }

```

3.10 RuntimeInit.java
---------------------

我感觉这个 zygoteInit 的命名有点问题，这里其实是**目标进程相关的初始化**，而非 Zygote 进程的初始化。

```
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");
 
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();//重定向log输出
 
        commonInit();//通用的一些初始化 我们跟进去看看
        nativeZygoteInit();// zygote初始化 重点跟
        applicationInit(targetSdkVersion, argv, classLoader);// 应用初始化 也要跟
    }
第一个
private static final void commonInit() {
        if (DEBUG) Slog.d(TAG, "Entered RuntimeInit!");
 
        /* 设置默认的未捕捉异常处理方法 */
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());
 
        /*
         * 时区的设置
         */
        TimezoneGetter.setInstance(new TimezoneGetter() {
            @Override
            public String getId() {
                return SystemProperties.get("persist.sys.timezone");
            }
        });
        TimeZone.setDefault(null);
 
        /*
         重置log
         */
        LogManager.getLogManager().reset();
        new AndroidConfig();
 
        /*
         * 设置默认的HTTP User-agent格式
         */
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);
 
        /*
         * Wire socket tagging to traffic stats.
         */
        NetworkManagementSocketTagger.install();
 
        /*
         设置socket的tag
         */
        String trace = SystemProperties.get("ro.kernel.android.tracing");
        if (trace.equals("1")) {
            Slog.i(TAG, "NOTE: emulator trace profiling enabled");
            Debug.enableEmulatorTraceOutput();
        }
 
        initialized = true;
    }
 
 
第二个nativeZygoteInit(); native函数
private static final native void nativeZygoteInit();
在app_main里面找到
    virtual void onZygoteInit()
    {
        sp proc = ProcessState::self();//调用open()打开/dev/binder驱动设备，再利用mmap()映射内核的地址空间，将Binder驱动的fd赋值ProcessState对象中的变量mDriverFD，用于交互操作。
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();//启动新binder线程
    }
 
 
第三个
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        // 应用程序退出时不调用AppRuntime.onExit()，否则会在退出前调用
        nativeSetExitWithoutCleanup(true);
 
        // 置虚拟机的内存利用率参数值为0.75
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
 
        final Arguments args;
        try {
            args = new Arguments(argv);//解析参数
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }
 
        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
 
        // 调用startClass的static方法 main() 此处args.startClass为”android.app.ActivityThread 注意这里传进来的classLoader为null，可以看3.9最后
    //这里也是需要跟下去的
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
 
 
RuntimeInit.invokeStaticMain
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class cl;
 
        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }
 
        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }
 
        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }
 
        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    } 
```

我们需要重点看最后的方法，通过反射获取了 Activity 的 main 方法，然后抛出一个 MethodAndArgsCaller，里面两个参数 m 和 argv，其中 m 为上面的 main 方法，argv 是 AMS 发送过来需要 zygote 进行 fork 的一系列参数。

那么这个异常如何处理呢，之前有研究过异常处理机制的兄弟应该知道会一直往上找调用方法，知道某 catch 到了这个异常。具体是哪里我们看下面

![](https://bbs.kanxue.com/upload/attach/202407/993634_HQ8PJWCE9HXQYT3.jpg)

原来是在 ZygoteInit 里面的 main 里面捕获到了

好，那我们接下来就看这个 run 是啥了。

3.11 ZygoteInit.java
--------------------

```
public static class MethodAndArgsCaller extends Exception
            implements Runnable {
        /** method to call */
    //在上面的方法可知是main方法
        private final Method mMethod;
 
    ///** 参数列表 */
        /** argument array */
        private final String[] mArgs;
 
        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }
 
        public void run() {
            try {
                ////反射调用，由上面可知这里会调用到ActivityThread.main()方法
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }

```

可以看到这里终于要进入到 App 进程的 ActivityThread.main 了

3.12 ZygoteConnection.java(回调 zygote 进程部分)
------------------------------------------

理论上来说这一 part 就要进入到 App 进程了，但是上次知道除了回调到 handlechildproc 之外还会回调到 zygote 进程，我们索性全部看完。

也比较简单，总体作用是通知 system server 进程，新进程已经创建完成

```
private boolean handleParentProc(int pid,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, Arguments parsedArgs) {
 
        if (pid > 0) {
            setChildPgid(pid);
        }
       ....
        try {
         
            mSocketOutStream.writeInt(pid);
            mSocketOutStream.writeBoolean(usingWrapper);
        } catch (IOException ex) {
            Log.e(TAG, "Error writing to command socket", ex);
            return true;
        }
 
        return false;
    }

```

3.13 zygote 阶段总结 2
------------------

1.  APP 或者桌面启动，需要**使用 Binder 机制告诉 AMS**（因为启动方有自己的进程，AMS 在 system server 进程），AMS 请求 zygote 进程是通过 Socket
2.  Process.start 方法是阻塞的，只有**等进程创建完成才会返回**
3.  zygote 进程 fork 进程一次，会有两次返回，在 zygote 进程和子进程分别返回一次，这样子线程就能执行 ZygoteConnection.handleChildProc 及以后的逻辑；zygote 进程通过 ZygoteConnection.handleParentProc 通知 system server 进程新进程已经创建完成

4 App 进程
========

这 part 就一步，非常短，可以总结为

```
成功fork之后，ActivityThread通过Binder进程间通信机制将一个ApplicationThread类型的Binder对象传递给ActivityManagerService，以便以后ActivityManagerService能够通过这个Binder对象和它进行通信。

```

![](https://bbs.kanxue.com/upload/attach/202407/993634_RJ6YNHDPM6ACR49.jpg)

4.1 ActivityThread.java
-----------------------

```
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();
 
        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);
 
        Environment.initForCurrentUser();
 
        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());
 
        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);
 
        Process.setArgV0("");
 
        Looper.prepareMainLooper();
 
    //我们暂且先关注这里thread.attach(false);
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
 
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
 
        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }
 
        // End of event ActivityThreadMain.
        //是在这里收到 AMS 的 BIND_APPLICATION 请求从而进行子进程初始化。除了这个请求，还有一个前面我们见过的请求: EXECUTE_TRANSACTION，所以在子进程初始化后，AMS 会进而完成 Activity 的启动处理。
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();
 
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
 
继续跟这个attach
     
private void attach(boolean system) {
        ...
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                //暂且省略别的代码，关注这个attachApplication，就是我们上面总览图上的那个函数了
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
    } 
```

可以看到在最后显示通过 ActivityManagerNative.getDefault(); 获得了一个 ActivityManagerService 的远程接口，是 ActivityManagerProxy 的 attachApplication 函数，这边传入的参数 mAppThread，是 **Binder 对象，用来 App 和 AMS 进程间的通信**。

5 AMS 进程
========

总结

```
1.然后对app的部分成员进行初始化
2.其他环境相关参数的设置，这里可以看看5.2，我写的比较多

```

![](https://bbs.kanxue.com/upload/attach/202407/993634_BM87BNPHDAYWZ53.jpg)

上回说到调用 ActivityManagerService 的远程接口 ActivityManagerProxy 的 attachApplication 函数

5.1 ActivityManagerNative.java
------------------------------

写在这了

```
public void attachApplication(IApplicationThread app) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
    //这
        data.writeStrongBinder(app.asBinder());
        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }

```

通过 Binder 驱动程序，最后进入 ActivityManagerService 的 attachApplication 函数

5.2 ActivityManagerService.java
-------------------------------

在 attachApplication 转发给了 attachApplicationLocked(thread, callingPid);

```
@Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }

```

这玩意长了

```
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
 
        // 我们知道在此之前我们已经成功创建了一个ProcessRecord
           //这里我们用pid把他取出来，放在app里面
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        } else {
            app = null;
        }
        //app为 null，总之就是错误情况
        if (app == null) {
           ...
        }
 
        // 这个application attched在其他进程，就清除掉
        if (app.thread != null) {
            handleAppDiedLocked(app, true, true);
        }
 
        // Tell the process all about itself.
 
        if (DEBUG_ALL) Slog.v(
                TAG, "Binding process pid " + pid + " to record " + app);
 
        //设置死亡监听
        final String processName = app.processName;
        try {
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            app.resetPackageList(mProcessStats);
            startProcessLocked(app, "link fail", processName);
            return false;
        }
 
        EventLog.writeEvent(EventLogTags.AM_PROC_BOUND, app.userId, app.pid, app.processName);
        //激活应用进程
        app.makeActive(thread, mProcessStats);
        // 重置调整值和调度组
        app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
        app.curSchedGroup = app.setSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        //重置其他状态 包括是否强制到前台、是否显示过UI、是否处于调试模式、是否已缓存、是否被AMS杀死等
        app.forcingToForeground = null;
        updateProcessForegroundLocked(app, false, false);
        app.hasShownUi = false;
        app.debugging = false;
        app.cached = false;
        app.killedByAm = false;
 
        //设置用户解锁状态
        app.unlocked = StorageManager.isUserKeyUnlocked(app.userId);
        //移除启动超时消息
        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
        //处理provider 判断系统是否处于正常模式 如果是，则生成并返回该应用的所有provider信息
        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        List providers = normalMode ? generateApplicationProvidersLocked(app) : null;
 
        //设置provider发布超时
        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }
 
        if (!normalMode) {
            Slog.i(TAG, "Launching preboot mode app: " + app);
        }
        //检查是否启用了全局调试
        if (DEBUG_ALL) Slog.v(
            TAG, "New app record " + app
            + " thread=" + thread.asBinder() + " pid=" + pid);
        try {
            int testMode = IApplicationThread.DEBUG_OFF;
            if (mDebugApp != null && mDebugApp.equals(processName)) {
                //如果是，则记录应用进程的一些基本信息，如应用记录（app）、线程绑定器（thread.asBinder()）和进程ID（pid）
                testMode = mWaitForDebugger
                    ? IApplicationThread.DEBUG_WAIT
                    : IApplicationThread.DEBUG_ON;
                app.debugging = true;
                if (mDebugTransient) {
                    //根据mDebugApp（当前设置为调试模式的应用名）和processName（当前进程名）的比较结果，来决定是否将当前应用设置为调试模式。如果当前应用被设置为调试模式，还会根据mWaitForDebugger的值来决定是等待调试器连接（IApplicationThread.DEBUG_WAIT）还是直接开启调试（IApplicationThread.DEBUG_ON）。如果设置了mDebugTransient（临时调试模式），则在设置完成后会恢复到之前的调试设置。
                    mDebugApp = mOrigDebugApp;
                    mWaitForDebugger = mOrigWaitForDebugger;
                }
            }
            String profileFile = app.instrumentationProfileFile;
            ParcelFileDescriptor profileFd = null;
            //性能分析
            int samplingInterval = 0;
            boolean profileAutoStop = false;
            if (mProfileApp != null && mProfileApp.equals(processName)) {
                mProfileProc = app;
                profileFile = mProfileFile;
                profileFd = mProfileFd;
                samplingInterval = mSamplingInterval;
                profileAutoStop = mAutoStopProfiler;
            }
            boolean enableTrackAllocation = false;
            //内存分配跟踪设置
            if (mTrackAllocationApp != null && mTrackAllocationApp.equals(processName)) {
                enableTrackAllocation = true;
                mTrackAllocationApp = null;
            }
 
            // 备份/恢复模式设置
            boolean isRestrictedBackupMode = false;
            if (mBackupTarget != null && mBackupAppName.equals(processName)) {
                isRestrictedBackupMode = mBackupTarget.appInfo.uid >= Process.FIRST_APPLICATION_UID
                        && ((mBackupTarget.backupMode == BackupRecord.RESTORE)
                                || (mBackupTarget.backupMode == BackupRecord.RESTORE_FULL)
                                || (mBackupTarget.backupMode == BackupRecord.BACKUP_FULL));
            }
 
            // 检查app对象的instrumentationClass是否非空
            if (app.instrumentationClass != null) {
                notifyPackageUse(app.instrumentationClass.getPackageName(),
                                 PackageManager.NOTIFY_PACKAGE_USE_INSTRUMENTATION);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION, "Binding proc "
                    + processName + " with config " + mConfiguration);
            //接下来，通过app.instrumentationInfo（如果存在）或app.info来确定应用的ApplicationInfo。
            ApplicationInfo appInfo = app.instrumentationInfo != null
                    ? app.instrumentationInfo : app.info;
            //设置兼容性信息
            app.compat = compatibilityInfoForPackageLocked(appInfo);
            if (profileFd != null) {
                profileFd = profileFd.dup();
            }
            ProfilerInfo profilerInfo = profileFile == null ? null
                //性能分析文件描述符
                    : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
            //通过thread.bindApplication方法将应用绑定到其进程。这个方法接受许多参数，包括进程名、ApplicationInfo、内容提供者列表等等。
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, null);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            // 第一次看这里还以为啥WTF呢。。
            //
            Slog.wtf(TAG, "Exception thrown during bind of " + app, e);
        //异常时 调用app.resetPackageList(mProcessStats)来重置应用的包列表
            app.resetPackageList(mProcessStats);
            //app.unlinkDeathRecipient()方法移除应用的死亡接收者（Death Recipient）
            app.unlinkDeathRecipient();
            //通过startProcessLocked(app, "bind fail", processName)方法重新启动应用进程
            startProcessLocked(app, "bind fail", processName);
            return false;
        }
 
         
        mPersistentStartingProcesses.remove(app);
        if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG_PROCESSES,
                "Attach application locked removing on hold: " + app);
        mProcessesOnHold.remove(app);
 
        boolean badApp = false;
        boolean didSomething = false;
 
        // 这个值可能表示系统是否处于正常操作模式
        if (normalMode) {
            try {
                //这里需要继续跟下去
                if  (mStackSupervisor.attachApplicationLocked(app))
                   //调用mStackSupervisor.attachApplicationLocked(app)尝试将应用（app）附加到系统中，以便启动其顶部的可见活动  {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
 
        // 启动服务
        if (!badApp) {
            try {
                //mServices.attachApplicationLocked(app, processName)尝试在指定的进程（processName）中启动任何应该运行的服务
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }
 
        // 处理待处理的广播 检查当前进程（通过pid）是否有待处理的广播
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                //如果有待处理的广播，并且应用（app）没有被标记为“坏的”，它会尝试通过sendPendingBroadcastsLocked(app)发送这些广播
                didSomething |= sendPendingBroadcastsLocked(app);
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
                badApp = true;
            }
        }
 
        // 检查是否有一个备份目标
        if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
            if (DEBUG_BACKUP) Slog.v(TAG_BACKUP,
                    "New app is backup target, launching agent for " + app);
            notifyPackageUse(mBackupTarget.appInfo.packageName,
                             PackageManager.NOTIFY_PACKAGE_USE_BACKUP);
            try {
                thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                        compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                        mBackupTarget.backupMode);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown creating backup agent in " + app, e);
                badApp = true;
            }
        }
 
        if (badApp) {
            //bad 时候的情况
            app.kill("error during init", true);
            handleAppDiedLocked(app, false, true);
            return false;
        }
 
        if (!didSomething) {
            //应用没有被标记为“坏的”，但也没有执行任何操作（如启动活动、服务或广播接收器），那么系统会调用updateOomAdjLocked();。这个方法可能用于更新应用进程的OOM（Out-Of-Memory，内存溢出）调整值。OOM调整值是Android系统用来决定在内存不足时哪个进程应该被首先终止的一个指标。通过更新这个值，系统可以优化内存使用，确保关键应用或服务的稳定性。
            updateOomAdjLocked();
        }
 
        return true;
    } 
```

跟罗升阳老师文章当中不一样的是这里 android7 做了封装

![](https://bbs.kanxue.com/upload/attach/202407/993634_CRFEVWYZUN88KJA.jpg)

不是直接的 realStartActivityLocked，而是需要进上面这个函数

5.3 ActivityStackSupervisor.java
--------------------------------

主要目的是

1.  检查系统中是否有等待启动的活动（Activity）
2.  即将附加的应用相匹配，并尝试启动这些活动。

```
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFocusedStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked();
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {//这里要跟
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                  + hr.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        }
        return didSomething;
    } 
```

还是这个. java 的

这里就不详细分析了

主要是下面注释中的关键代码，通过 app.thread 进入到 ApplicationThreadProxy 的 scheduleLaunchActivity 函数中

其中第二个参数，是一个 ActivityRecord 类型的 Binder 对象，用来作来这个 Activity 的 token 值。

```
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
 
        if (!allPausedActivitiesComplete()) {
            // While there are activities pausing we skipping starting any new activities until
            // pauses are complete. NOTE: that we also do this for activities that are starting in
            // the paused state because they will first be resumed then paused on the client side.
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    "realStartActivityLocked: Skipping start of r=" + r
                    + " some activities pausing...");
            return false;
        }
 
        if (andResume) {
            r.startFreezingScreenLocked(app, 0);
            mWindowManager.setAppVisibility(r.appToken, true);
 
            // schedule launch ticks to collect information about slow apps.
            r.startLaunchTickingLocked();
        }
 
        // Have the window manager re-evaluate the orientation of
        // the screen based on the new activity order.  Note that
        // as a result of this, it can call back into the activity
        // manager with a new orientation.  We don't care about that,
        // because the activity is not currently running so we are
        // just restarting it anyway.
        if (checkConfig) {
            Configuration config = mWindowManager.updateOrientationFromAppTokens(
                    mService.mConfiguration,
                    r.mayFreezeScreenLocked(app) ? r.appToken : null);
            mService.updateConfigurationLocked(config, r, false);
        }
 
        r.app = app;
        app.waitingToKill = null;
        r.launchCount++;
        r.lastLaunchTime = SystemClock.uptimeMillis();
 
        if (DEBUG_ALL) Slog.v(TAG, "Launching: " + r);
 
        int idx = app.activities.indexOf(r);
        if (idx < 0) {
            app.activities.add(r);
        }
        mService.updateLruProcessLocked(app, true, null);
        mService.updateOomAdjLocked();
 
        final TaskRecord task = r.task;
        if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE ||
                task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV) {
            setLockTaskModeLocked(task, LOCK_TASK_MODE_LOCKED, "mLockTaskAuth==LAUNCHABLE", false);
        }
 
        final ActivityStack stack = task.stack;
        try {
            if (app.thread == null) {
                throw new RemoteException();
            }
            List results = null;
            List newIntents = null;
            if (andResume) {
                results = r.results;
                newIntents = r.newIntents;
            }
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                    "Launching: " + r + " icicle=" + r.icicle + " with results=" + results
                    + " newIntents=" + newIntents + " andResume=" + andResume);
            if (andResume) {
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY,
                        r.userId, System.identityHashCode(r),
                        task.taskId, r.shortComponentName);
            }
            if (r.isHomeActivity()) {
                // Home process is the root process of the task.
                mService.mHomeProcess = task.mActivities.get(0).app;
            }
            mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
                                      PackageManager.NOTIFY_PACKAGE_USE_ACTIVITY);
            r.sleeping = false;
            r.forceNewConfig = false;
            mService.showUnsupportedZoomDialogIfNeededLocked(r);
            mService.showAskCompatModeDialogLocked(r);
            r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
            ProfilerInfo profilerInfo = null;
            if (mService.mProfileApp != null && mService.mProfileApp.equals(app.processName)) {
                if (mService.mProfileProc == null || mService.mProfileProc == app) {
                    mService.mProfileProc = app;
                    final String profileFile = mService.mProfileFile;
                    if (profileFile != null) {
                        ParcelFileDescriptor profileFd = mService.mProfileFd;
                        if (profileFd != null) {
                            try {
                                profileFd = profileFd.dup();
                            } catch (IOException e) {
                                if (profileFd != null) {
                                    try {
                                        profileFd.close();
                                    } catch (IOException o) {
                                    }
                                    profileFd = null;
                                }
                            }
                        }
 
                        profilerInfo = new ProfilerInfo(profileFile, profileFd,
                                mService.mSamplingInterval, mService.mAutoStopProfiler);
                    }
                }
            }
 
            if (andResume) {
                app.hasShownUi = true;
                app.pendingUiClean = true;
            }
            app.forceProcessStateUpTo(mService.mTopProcessState);
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
 
            if ((app.info.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
                // This may be a heavy-weight process!  Note that the package
                // manager will ensure that only activity can run in the main
                // process of the .apk, which is the only thing that will be
                // considered heavy-weight.
                if (app.processName.equals(app.info.packageName)) {
                    if (mService.mHeavyWeightProcess != null
                            && mService.mHeavyWeightProcess != app) {
                        Slog.w(TAG, "Starting new heavy weight process " + app
                                + " when already running "
                                + mService.mHeavyWeightProcess);
                    }
                    mService.mHeavyWeightProcess = app;
                    Message msg = mService.mHandler.obtainMessage(
                            ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                    msg.obj = r;
                    mService.mHandler.sendMessage(msg);
                }
            }
 
        } catch (RemoteException e) {
            if (r.launchFailed) {
                // This is the second time we failed -- finish activity
                // and give up.
                Slog.e(TAG, "Second failure launching "
                      + r.intent.getComponent().flattenToShortString()
                      + ", giving up", e);
                mService.appDiedLocked(app);
                stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                        "2nd-crash", false);
                return false;
            }
 
            // This is the first time we failed -- restart process and
            // retry.
            app.activities.remove(r);
            throw e;
        }
 
        r.launchFailed = false;
        if (stack.updateLRUListLocked(r)) {
            Slog.w(TAG, "Activity " + r + " being launched, but already in LRU list");
        }
 
        if (andResume) {
            // As part of the process of launching, ActivityThread also performs
            // a resume.
            stack.minimalResumeActivityLocked(r);
        } else {
            // This activity is not starting in the resumed state... which should look like we asked
            // it to pause+stop (but remain visible), and it has done so and reported back the
            // current icicle and other state.
            if (DEBUG_STATES) Slog.v(TAG_STATES,
                    "Moving to PAUSED: " + r + " (starting in paused state)");
            r.state = PAUSED;
        }
 
        // Launch the new version setup screen if needed.  We do this -after-
        // launching the initial activity (that is, home), so that it can have
        // a chance to initialize itself while in the background, making the
        // switch back to it faster and look better.
        if (isFocusedStack(stack)) {
            mService.startSetupActivityLocked();
        }
 
        // Update any services we are bound to that might care about whether
        // their client may have activities.
        if (r.app != null) {
            mService.mServices.updateServiceConnectionActivitiesLocked(r.app);
        }
 
        return true;
    } 
```

5.4 ApplicationThreadNative.java
--------------------------------

好了 终于是进入到了这个代理类的 scheduleLaunchActivity，也是到 ApplicationThread 的 scheduleLaunchActivity 函数前的最后一步

```
////这个函数最终通过Binder驱动程序进入到ApplicationThread的scheduleLaunchActivity函数中
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List pendingResults, List pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        intent.writeToParcel(data, 0);
        data.writeStrongBinder(token);
        data.writeInt(ident);
        info.writeToParcel(data, 0);
        curConfig.writeToParcel(data, 0);
        if (overrideConfig != null) {
            data.writeInt(1);
            overrideConfig.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        compatInfo.writeToParcel(data, 0);
        data.writeString(referrer);
         
        data.writeStrongBinder(voiceInteractor != null ? voiceInteractor.asBinder() : null);
        data.writeInt(procState);
        data.writeBundle(state);
        data.writePersistableBundle(persistentState);
        data.writeTypedList(pendingResults);
        data.writeTypedList(pendingNewIntents);
        data.writeInt(notResumed ? 1 : 0);
        data.writeInt(isForward ? 1 : 0);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    } 
```

6 App 进程
========

到了最后一个 part 了，后面的逻辑总体上都是属于我们创建出的这个所谓的 App 进程里面的了。

总结

```
1.告诉App进程一切准备就绪
2.让他开始启动Activity，并告诉它如何启动
3.最后来到oncreate了

```

![](https://bbs.kanxue.com/upload/attach/202407/993634_JZRMZC2G8DFRZTV.jpg)

6.1 ActivityThread.java
-----------------------

不是很长

这里就不墨迹了，直接看最后。

我们的信息**整理到 r 里面**，然后 sendmessage 出去了

```
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List pendingResults, List pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
 
            updateProcessState(procState, false);
 
            ActivityClientRecord r = new ActivityClientRecord();
 
            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;
 
            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;
 
            r.startsNotResumed = notResumed;
            r.isForward = isForward;
 
            r.profilerInfo = profilerInfo;
 
            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);
 
            sendMessage(H.LAUNCH_ACTIVITY, r);
        } 
```

可以看到调用了 handleLaunchActivity。

```
public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
 
                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
              ...
            }

```

handleLaunchActivity 就是下面这个样子

```
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;
 
        if (r.profilerInfo != null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }
 
        // Make sure we are running with the most recent config.
        handleConfigurationChanged(null, null);
 
        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);
 
        // Initialize before creating the activity
        WindowManagerGlobal.initialize();
 
        Activity a = performLaunchActivity(r, customIntent);
 
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
 
           ...
    }

```

对上面的这个函数 做一个解释吧

1.  取消 GC 计划
2.  **标记活动已更改**
3.  **设置和启动性能分析器**
4.  **处理配置变化**：调用`handleConfigurationChanged`方法确保使用最新的配置信息。
5.  **初始化 WindowManager**：调用`WindowManagerGlobal.initialize()`初始化窗口管理器。
6.  **启动 Activity**：调用`performLaunchActivity`方法根据`ActivityClientRecord`和自定义 Intent 启动 Activity，并获取启动后的 Activity 实例。
7.  **处理 Activity 的启动后逻辑**：
    *   如果 Activity 成功启动，保存当前的配置信息到`r.createdConfig`。
    *   调用`reportSizeConfigurations`方法报告 Activity 的大小配置。
    *   如果 Activity 之前的状态（`oldState`）存在，根据 Activity 的当前状态和启动参数调用`handleResumeActivity`方法处理 Activity 的恢复或启动逻辑。
    *   如果 Activity 未完成且被标记为`startsNotResumed`（即 Activity 需要显示但不在前台），则调用`performPauseActivityIfNeeded`方法执行必要的暂停逻辑，但不需要执行完整的暂停周期（如冻结状态等），因为 Activity 管理器假设它可以保留当前状态。

6.2 ActivityThread.java (performLaunchActivity)
-----------------------------------------------

ok，重点就是 **performLaunchActivity**，看他怎么启动 activity

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");
 
    //从 ActivityClientRecord（r）中获取 ActivityInfo（aInfo），它包含了关于要启动的 Activity 的各种信息，如名称、包名、主题等。
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            //如果 r.packageInfo 为空，则通过调用 getPackageInfo 方法获取与 Activity 关联的 PackageInfo
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
    //尝试从 r.intent 中获取 ComponentName
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }
    //如果 ActivityInfo（aInfo）中的 targetActivity 不为空，说明该 Activity 有一个目标 Activity 需要被启动。在这种情况下，会创建一个新的 ComponentName，其包名为 ActivityInfo 中的包名，类名为 targetActivity 指定的类名。然后，这个新的 ComponentName 会被用作启动的目标。
        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
 
        Activity activity = null;
        try {
            //从 r.packageInfo 中获取与包相关联的 ClassLoader。 后面学加固的时候这里会涉及到,fart就有
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            //使用 mInstrumentation.newActivity 方法创建 Activity 的实例。
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            //通过 r.intent.setExtrasClassLoader(cl) 设置 Intent 中额外数据的 ClassLoader。
            r.intent.setExtrasClassLoader(cl);
            //调用 r.intent.prepareToEnterProcess() 来准备 Intent 进入目标进程
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                //如果 r.state 不为空（这通常表示 Activity 是从一个已保存的状态中恢复的），则通过 r.state.setClassLoader(cl) 设置其 ClassLoader。
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
 
        try {
            //创建或获取与 Activity 关联的应用程序上下文（Application 实例）
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
 
            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());
 
            if (activity != null) {//通过 createBaseContextForActivity(r, activity) 方法为 Activity 创建一个基础上下文（Context）
                Context appContext = createBaseContextForActivity(r, activity);
                //使用 r.activityInfo.loadLabel(appContext.getPackageManager()) 加载 Activity 的标题
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                //调用 activity.attach(...) 方法将 Activity 附加到其应用程序上下文、Activity 线程（ActivityThread）、Instrumentation、令牌（用于内部标识）、标识符、应用程序实例、Intent、ActivityInfo、标题、父 Activity、嵌入 ID、上次非配置实例、配置、引荐者（referrer）和语音交互器等。
                // 这是首个activity的生命周期的起点，它设置了 Activity 运行所需的所有关键组件和状态。
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);
 
                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                //获取 Activity 的主题资源 ID
                int theme = r.activityInfo.getThemeResource();
                //设置主题
                if (theme != 0) {
                    activity.setTheme(theme);
                }
             
                //标记 Activity 的 onCreate 方法是否已调用
                activity.mCalled = false;
                //调用 onCreate 方法
                //根据 Activity 是否是可持久化的（isPersistable()），Instrumentation 对象会调用不同的 onCreate 方法。
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                //检查 onCreate 方法是否已调用
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                //记录 Activity 实例并更新
                r.activity = activity;
                r.stopped = true;
                        "Activity " + r.intent.getComponent().toShortString() +
                //
                ...
                //检查 Activity 是否完成
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    //重置 mCalled 标志并调用 onPostCreate
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;
 
            mActivities.put(r.token, r);
 
        ...
 
        return activity;
    }

```

好通过 callActivityOnCreate 方法要到我们的 oncreate 了

**这里不是直接调用 MainActivity 的 onCreate 函数，而是通过 mInstrumentation 的 callActivityOnCreate 函数来间接调用，前面我们说过，mInstrumentation 在这里的作用是监控 Activity 与系统的交互操作**

6.3 Instrumentation.java
------------------------

既然写了，我们就刨根就地跟

```
其中一个
 
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }

```

暂且不管 prePerformCreate postPerformCreate 我们看 activity.performCreate(icicle);

6.4 Activity.java
-----------------

好了，终于到尾声了， onCreate(icicle); 根 Activity 就启动了，即应用程序就启动了。

```
final void performCreate(Bundle icicle) {
        restoreHasCurrentPermissionRequest(icicle);
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }

```

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 18 小时前 被 C0rax 编辑 ，原因：

[#基础理论](forum-161-1-117.htm)