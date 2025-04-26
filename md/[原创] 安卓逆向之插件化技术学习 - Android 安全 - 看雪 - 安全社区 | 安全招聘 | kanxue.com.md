> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286613.htm)

> [原创] 安卓逆向之插件化技术学习

### **一、引言**​

1.  ​**插件化开发的背景与意义**

*   从 12 年的概念提出，到 16 年的把花齐放，件化技术引领着 Android 技术的进步。
    
*   ​现在出现的 flutter 等等 已经让插件化技术变得没那么重要
    

1.  ​**插件化技术的适用场景**​

*   大型应用模块化拆分、按需加载功能、第三方服务集成
    
*   多开 app，无需安装运行 app 的框架
    

我所了解到 virtualapp,edxposed 等等，可以说实现安卓 app 的多开，那时候我就想自己实现一个简单的 demo，可以多开简单的 app 就够了。

于是开始了的安卓学习之旅，本以为学完了安卓基础，就可以自己完成开发，结果就只能望着 android studio 发呆 ------ 原来这个是高级的开发知识。

继而开始了我的安卓源码学习之路。

读了很多教程与安卓开发的书籍才知道，要实现多开要先学会插件化开发

那我们一起进入这个旅程吧 ------------

源码基于 安卓 12

Activity 在源码中的启动
----------------

从开发者熟悉的的 startActivity 开始

### 定位 Activity 的 startActivity 方法

```
5687 
5688      /**
5689       * Same as {@link #startActivity(Intent, Bundle)} with no options
5690       * specified.
5691       *
5692       * @param intent The intent to start.
5693       *
5694       * @throws android.content.ActivityNotFoundException
5695       *
5696       * @see #startActivity(Intent, Bundle)
5697       * @see #startActivityForResult
5698       */
5699      @Override
5700      public void startActivity(Intent intent) {
5701          this.startActivity(intent, null);
5702      }
```

继续跟踪这个方法

```
5704      /**
5705       * Launch a new activity.  You will not receive any information about when
5706       * the activity exits.  This implementation overrides the base version,
5707       * providing information about
5708       * the activity performing the launch.  Because of this additional
5709       * information, the {@link Intent#FLAG_ACTIVITY_NEW_TASK} launch flag is not
5710       * required; if not specified, the new activity will be added to the
5711       * task of the caller.
5712       *
5713       * 

This method throws {@link android.content.ActivityNotFoundException}
5714       * if there was no Activity found to run the given Intent.
5715       *
5716       * @param intent The intent to start.
5717       * @param options Additional options for how the Activity should be started.
5718       * See {@link android.content.Context#startActivity(Intent, Bundle)}
5719       * Context.startActivity(Intent, Bundle)} for more details.
5720       *
5721       * @throws android.content.ActivityNotFoundException
5722       *
5723       * @see #startActivity(Intent)
5724       * @see #startActivityForResult
5725       */
5726      @Override
5727      public void startActivity(Intent intent, @Nullable Bundle options) {
5728          if (mIntent != null && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN)
5729                  && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY)) {
5730              if (TextUtils.equals(getPackageName(),
5731                      intent.resolveActivity(getPackageManager()).getPackageName())) {
5732                  // Apply Autofill restore mechanism on the started activity by startActivity()
5733                  final IBinder token =
5734                          mIntent.getIBinderExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN);
5735                  // Remove restore ability from current activity
5736                  mIntent.removeExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN);
5737                  mIntent.removeExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY);
5738                  // Put restore token
5739                  intent.putExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN, token);
5740                  intent.putExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY, true);
5741              }
5742          }
5743          if (options != null) {
5744              startActivityForResult(intent, -1, options);
5745          } else {
5746              // Note we want to go through this call for compatibility with
5747              // applications that may have overridden the method.
5748              startActivityForResult(intent, -1);
5749          }
5750      }



```

然后就是 startActivityForResult 方法

```
   public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
5400              @Nullable Bundle options) {
5401          if (mParent == null) {
5402              options = transferSpringboardActivityOptions(options);
5403              Instrumentation.ActivityResult ar =
5404                  mInstrumentation.execStartActivity(
5405                      this, mMainThread.getApplicationThread(), mToken, this,
5406                      intent, requestCode, options);
5407              if (ar != null) {
5408                  mMainThread.sendActivityResult(
5409                      mToken, mEmbeddedID, requestCode, ar.getResultCode(),
5410                      ar.getResultData());
5411              }
5412              if (requestCode >= 0) {
5413                  // If this start is requesting a result, we can avoid making
5414                  // the activity visible until the result is received.  Setting
5415                  // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
5416                  // activity hidden during this time, to avoid flickering.
5417                  // This can only be done when a result is requested because
5418                  // that guarantees we will get information back when the
5419                  // activity is finished, no matter what happens to it.
5420                  mStartedActivity = true;
5421              }
5422 
5423              cancelInputsAndStartExitTransition(options);
5424              // TODO Consider clearing/flushing other event sources and events for child windows.
5425          } else {
5426              if (options != null) {
5427                  mParent.startActivityFromChild(this, intent, requestCode, options);
5428              } else {
5429                  // Note we want to go through this method for compatibility with
5430                  // existing applications that may have overridden it.
5431                  mParent.startActivityFromChild(this, intent, requestCode);
5432              }
5433          }
5434      }
```

主要的代码就是这个玩意了 ：mInstrumentation.execStartActivity

```
1710      public ActivityResult execStartActivity(
1711              Context who, IBinder contextThread, IBinder token, Activity target,
1712              Intent intent, int requestCode, Bundle options) {
1713          IApplicationThread whoThread = (IApplicationThread) contextThread;
1714          Uri referrer = target != null ? target.onProvideReferrer() : null;
1715          if (referrer != null) {
1716              intent.putExtra(Intent.EXTRA_REFERRER, referrer);
1717          }
1718          if (mActivityMonitors != null) {
1719              synchronized (mSync) {
1720                  final int N = mActivityMonitors.size();
1721                  for (int i=0; i= 0 ? am.getResult() : null;
1734                          }
1735                          break;
1736                      }
1737                  }
1738              }
1739          }
1740          try {
1741              intent.migrateExtraStreamToClipData(who);
1742              intent.prepareToLeaveProcess(who);
1743              int result = ActivityTaskManager.getService().startActivity(whoThread,
1744                      who.getOpPackageName(), who.getAttributionTag(), intent,
1745                      intent.resolveTypeIfNeeded(who.getContentResolver()), token,
1746                      target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
1747              checkStartActivityResult(result, intent);
1748          } catch (RemoteException e) {
1749              throw new RuntimeException("Failure from system", e);
1750          }
1751          return null;
1752      } 
```

这里就调用 ActivityTaskManager.getService().startActivity 方法了 这里就开始 与 ams 沟通

AMS 的内部就不记录了

后面会调用 ActivityThread 的 H 对象 进行沟通

```
@Override
public void scheduleTransaction(ClientTransaction transaction) {
    mH.sendMessage(H.EXECUTE_TRANSACTION, transaction);  // 发送消息到应用进程
}
```

```
    case EXECUTE_TRANSACTION:
2213                      final ClientTransaction transaction = (ClientTransaction) msg.obj;
2214                      mTransactionExecutor.execute(transaction);
2215                      if (isSystem()) {
2216                          // Client transactions inside system process are recycled on the client side
2217                          // instead of ClientLifecycleManager to avoid being cleared before this
2218                          // message is handled.
2219                          transaction.recycle();
2220                      }
2221                      // TODO(lifecycler): Recycle locally scheduled transactions.
```

我查看了 TransactionExecutor 的类

```
     /** Transition the client through previously initialized state sequence. */
205      private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
206              ClientTransaction transaction) {
207          final int size = path.size();
208          for (int i = 0, state; i < size; i++) {
209              state = path.get(i);
210              if (DEBUG_RESOLVER) {
211                  Slog.d(TAG, tId(transaction) + "Transitioning activity: "
212                          + getShortActivityName(r.token, mTransactionHandler)
213                          + " to state: " + getStateName(state));
214              }
215              switch (state) {
216                  case ON_CREATE:
217                      mTransactionHandler.handleLaunchActivity(r, mPendingActions,
218                              null /* customIntent */);
219                      break;
220                  case ON_START:
221                      mTransactionHandler.handleStartActivity(r, mPendingActions,
222                              null /* activityOptions */);
223                      break;
224                  case ON_RESUME:
225                      mTransactionHandler.handleResumeActivity(r, false /* finalStateRequest */,
226                              r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
227                      break;
228                  case ON_PAUSE:
229                      mTransactionHandler.handlePauseActivity(r, false /* finished */,
230                              false /* userLeaving */, 0 /* configChanges */, mPendingActions,
231                              "LIFECYCLER_PAUSE_ACTIVITY");
232                      break;
233                  case ON_STOP:
234                      mTransactionHandler.handleStopActivity(r, 0 /* configChanges */,
235                              mPendingActions, false /* finalStateRequest */,
236                              "LIFECYCLER_STOP_ACTIVITY");
237                      break;
238                  case ON_DESTROY:
239                      mTransactionHandler.handleDestroyActivity(r, false /* finishing */,
240                              0 /* configChanges */, false /* getNonConfigInstance */,
241                              "performLifecycleSequence. cycling to:" + path.get(size - 1));
242                      break;
243                  case ON_RESTART:
244                      mTransactionHandler.performRestartActivity(r, false /* start */);
245                      break;
246                  default:
247                      throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
248              }
249          }
250      }
```

这里就进入

```
mTransactionHandler.handleLaunchActivity(r, mPendingActions,
218                              null /* customIntent */);
```

```
/**
3758       * Extended implementation of activity launch. Used when server requests a launch or relaunch.
3759       */
3760      @Override
3761      public Activity handleLaunchActivity(ActivityClientRecord r,
3762              PendingTransactionActions pendingActions, Intent customIntent) {
3763          // If we are getting ready to gc after going to the background, well
3764          // we are back active so skip it.
3765          unscheduleGcIdler();
3766          mSomeActivitiesChanged = true;
3767 
3768          if (r.profilerInfo != null) {
3769              mProfiler.setProfiler(r.profilerInfo);
3770              mProfiler.startProfiling();
3771          }
3772 
3773          if (r.mPendingFixedRotationAdjustments != null) {
3774              // The rotation adjustments must be applied before handling configuration, so process
3775              // level display metrics can be adjusted.
3776              overrideApplicationDisplayAdjustments(r.token, adjustments ->
3777                      adjustments.setFixedRotationAdjustments(r.mPendingFixedRotationAdjustments));
3778          }
3779 
3780          // Make sure we are running with the most recent config.
3781          mConfigurationController.handleConfigurationChanged(null, null);
3782 
3783          if (localLOGV) Slog.v(
3784              TAG, "Handling launch of " + r);
3785 
3786          // Initialize before creating the activity
3787          if (ThreadedRenderer.sRendererEnabled
3788                  && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
3789              HardwareRenderer.preload();
3790          }
3791          WindowManagerGlobal.initialize();
3792 
3793          // Hint the GraphicsEnvironment that an activity is launching on the process.
3794          GraphicsEnvironment.hintActivityLaunch();
3795 
3796          final Activity a = performLaunchActivity(r, customIntent);
3797 
3798          if (a != null) {
3799              r.createdConfig = new Configuration(mConfigurationController.getConfiguration());
3800              reportSizeConfigurations(r);
3801              if (!r.activity.mFinished && pendingActions != null) {
3802                  pendingActions.setOldState(r.state);
3803                  pendingActions.setRestoreInstanceState(true);
3804                  pendingActions.setCallOnPostCreate(true);
3805              }
3806          } else {
3807              // If there was an error, for any reason, tell the activity manager to stop us.
3808              ActivityClient.getInstance().finishActivity(r.token, Activity.RESULT_CANCELED,
3809                      null /* resultData */, Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
3810          }
3811 
3812          return a;
3813      }
```

继续跟进

```
3512      /**  Core implementation of activity launch. */
3513      private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
3514          ActivityInfo aInfo = r.activityInfo;
3515          if (r.packageInfo == null) {
3516              r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
3517                      Context.CONTEXT_INCLUDE_CODE);
3518          }
3519 
3520          ComponentName component = r.intent.getComponent();
3521          if (component == null) {
3522              component = r.intent.resolveActivity(
3523                  mInitialApplication.getPackageManager());
3524              r.intent.setComponent(component);
3525          }
3526 
3527          if (r.activityInfo.targetActivity != null) {
3528              component = new ComponentName(r.activityInfo.packageName,
3529                      r.activityInfo.targetActivity);
3530          }
3531 
3532          ContextImpl appContext = createBaseContextForActivity(r);
3533          Activity activity = null;
3534          try {
3535              java.lang.ClassLoader cl = appContext.getClassLoader();
3536              activity = mInstrumentation.newActivity(
3537                      cl, component.getClassName(), r.intent);
3538              StrictMode.incrementExpectedActivityCount(activity.getClass());
3539              r.intent.setExtrasClassLoader(cl);
3540              r.intent.prepareToEnterProcess(isProtectedComponent(r.activityInfo),
3541                      appContext.getAttributionSource());
3542              if (r.state != null) {
3543                  r.state.setClassLoader(cl);
3544              }
3545          } catch (Exception e) {
3546              if (!mInstrumentation.onException(activity, e)) {
3547                  throw new RuntimeException(
3548                      "Unable to instantiate activity " + component
3549                      + ": " + e.toString(), e);
3550              }
3551          }
3552 
3553          try {
3554              Application app = r.packageInfo.makeApplication(false, mInstrumentation);
3555 
3556              if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
3557              if (localLOGV) Slog.v(
3558                      TAG, r + ": app=" + app
3559                      + ", appName=" + app.getPackageName()
3560                      + ", pkg=" + r.packageInfo.getPackageName()
3561                      + ", comp=" + r.intent.getComponent().toShortString()
3562                      + ", dir=" + r.packageInfo.getAppDir());
3563 
3564              if (activity != null) {
3565                  CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
3566                  Configuration config =
3567                          new Configuration(mConfigurationController.getCompatConfiguration());
3568                  if (r.overrideConfig != null) {
3569                      config.updateFrom(r.overrideConfig);
3570                  }
3571                  if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
3572                          + r.activityInfo.name + " with config " + config);
3573                  Window window = null;
3574                  if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
3575                      window = r.mPendingRemoveWindow;
3576                      r.mPendingRemoveWindow = null;
3577                      r.mPendingRemoveWindowManager = null;
3578                  }
3579 
3580                  // Activity resources must be initialized with the same loaders as the
3581                  // application context.
3582                  appContext.getResources().addLoaders(
3583                          app.getResources().getLoaders().toArray(new ResourcesLoader[0]));
3584 
3585                  appContext.setOuterContext(activity);
3586                  activity.attach(appContext, this, getInstrumentation(), r.token,
3587                          r.ident, app, r.intent, r.activityInfo, title, r.parent,
3588                          r.embeddedID, r.lastNonConfigurationInstances, config,
3589                          r.referrer, r.voiceInteractor, window, r.configCallback,
3590                          r.assistToken, r.shareableActivityToken);
3591 
3592                  if (customIntent != null) {
3593                      activity.mIntent = customIntent;
3594                  }
3595                  r.lastNonConfigurationInstances = null;
3596                  checkAndBlockForNetworkAccess();
3597                  activity.mStartedActivity = false;
3598                  int theme = r.activityInfo.getThemeResource();
3599                  if (theme != 0) {
3600                      activity.setTheme(theme);
3601                  }
3602 
3603                  if (r.mActivityOptions != null) {
3604                      activity.mPendingOptions = r.mActivityOptions;
3605                      r.mActivityOptions = null;
3606                  }
3607                  activity.mLaunchedFromBubble = r.mLaunchedFromBubble;
3608                  activity.mCalled = false;
3609                  if (r.isPersistable()) {
3610                      mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
3611                  } else {
3612                      mInstrumentation.callActivityOnCreate(activity, r.state);
3613                  }
3614                  if (!activity.mCalled) {
3615                      throw new SuperNotCalledException(
3616                          "Activity " + r.intent.getComponent().toShortString() +
3617                          " did not call through to super.onCreate()");
3618                  }
3619                  r.activity = activity;
3620                  mLastReportedWindowingMode.put(activity.getActivityToken(),
3621                          config.windowConfiguration.getWindowingMode());
3622              }
3623              r.setState(ON_CREATE);
3624 
3625              // updatePendingActivityConfiguration() reads from mActivities to update
3626              // ActivityClientRecord which runs in a different thread. Protect modifications to
3627              // mActivities to avoid race.
3628              synchronized (mResourcesManager) {
3629                  mActivities.put(r.token, r);
3630              }
3631 
3632          } catch (SuperNotCalledException e) {
3633              throw e;
3634 
3635          } catch (Exception e) {
3636              if (!mInstrumentation.onException(activity, e)) {
3637                  throw new RuntimeException(
3638                      "Unable to start activity " + component
3639                      + ": " + e.toString(), e);
3640              }
3641          }
3642 
3643          return activity;
3644      }
```

这里进行一个简单的总结

1.  **`startActivity`方法的调用链**

*   开发者常用的`startActivity(Intent intent)`方法，实际上调用的是`startActivity(Intent intent, Bundle options)`，将`options`设为`null`。
    
*   在`startActivity(Intent intent, Bundle options)`方法中，会先处理与自动填充（Autofill）相关的逻辑。若满足特定条件，会在启动的 Activity 上应用自动填充恢复机制。之后，根据`options`是否为`null`，调用`startActivityForResult(intent, -1, options)`或`startActivityForResult(intent, -1)`方法 。
    

1.  **`startActivityForResult`方法的处理**

*   当`mParent`为`null`时，会调用`mInstrumentation.execStartActivity`方法，并传入一系列参数，包括当前上下文、应用线程、Activity 令牌等。同时，如果启动请求需要结果（`requestCode >= 0`），会标记`mStartedActivity`为`true`，以避免在结果返回前 Activity 闪烁。
    
*   当`mParent`不为`null`时，会调用`mParent.startActivityFromChild`方法，同样会根据`options`是否为`null`进行不同的处理。
    

1.  **`execStartActivity`方法与 AMS 交互**

*   在`mInstrumentation.execStartActivity`方法中，首先会处理`ActivityMonitor`相关逻辑，检查是否有匹配的监视器并进行相应操作。
    
*   接着，通过`ActivityTaskManager.getService().startActivity`方法与 Activity 管理服务（AMS）进行跨进程通信，将启动 Activity 的请求发送给 AMS。AMS 接收到请求后，会进行一系列系统层面的处理，如任务调度、Activity 栈管理等，不过网页中未详细记录 AMS 内部的处理过程。
    

1.  **ActivityThread 的消息处理与 Activity 生命周期**

*   AMS 处理完启动请求后，会通过 ActivityThread 的`H`对象发送消息。`H`类接收到`EXECUTE_TRANSACTION`消息后，会调用`mTransactionExecutor.execute(transaction)`方法。
    
*   `performLifecycleSequence`方法根据不同的生命周期状态（如`ON_CREATE`、`ON_START`等），调用`mTransactionHandler`相应的方法来处理 Activity 的生命周期转换。例如，在`ON_CREATE`状态时，会调用`mTransactionHandler.handleLaunchActivity`方法。
    

1.  **`handleLaunchActivity`与`performLaunchActivity`方法**

*   `handleLaunchActivity`方法会进行一些准备工作，如取消垃圾回收任务、处理配置变更等，然后调用`performLaunchActivity`方法来实际启动 Activity。
    
*   `performLaunchActivity`方法负责创建 Activity 实例，首先获取 Activity 的相关信息，如`ActivityInfo`、`PackageInfo`等。接着，使用`ClassLoader`加载 Activity 类，并通过`mInstrumentation.newActivity`方法创建 Activity 实例。然后，创建应用上下文`ContextImpl`，并将 Activity 与上下文、应用等进行关联。最后，调用`mInstrumentation.callActivityOnCreate`方法，触发 Activity 的`onCreate`方法，完成 Activity 的初始化。
    

### 实现 activity 的加载

```
    public static void hook()
    {
        //开启 欺骗 ams
        Method execStartActivity = null;
        try {
            execStartActivity = Instrumentation.class.getDeclaredMethod("execStartActivity",  Context.class,//0
                    IBinder.class,//1
                    IBinder.class,//2
                    Activity.class,//3
                    Intent.class,//4
                    int.class,//5
                    Bundle.class);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        CalvinBridge.hookMethod(execStartActivity, new CA_MethodHook() {
            @Override
            public void beforeHookedMethod(MethodHookParam param) throws Throwable {
                Log.e("hook", "hook 测试成功！"+param.args[3]);
                Intent oriIntent =(Intent) param.args[4];
                Intent intent = new Intent((Activity) param.args[3], ProxyActivity.class);
                intent.setComponent(new ComponentName(BaseData.oriAppName,BaseData.proxyActivity));
                intent.putExtra("oriIntent",oriIntent);
                param.args[4] = intent;
            }
        });
 
 
        //欺骗 app
        try {
            Class aClass = ReflectUtils.loadClass("android.app.ActivityThread");
            Class ActivityClientRecord = ReflectUtils.loadClass("android.app.ActivityThread");
            CalvinBridge.hookAllMethods(aClass, "performLaunchActivity", new CA_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Log.e(TAG, "createBaseContextForActivity " + param.args[0].getClass());
                    Object arg = param.args[0];
                    Object activityInfo = ReflectUtils.getField(arg, "activityInfo");
                    Object LoadedApk = ReflectUtils.getField(arg, "packageInfo");
                    ReflectUtils.printFieldValues(LoadedApk);
                    ReflectUtils.setField(LoadedApk, "mApplication", null);
                    ReflectUtils.setField(LoadedApk, "mApplicationInfo", packageInfo.applicationInfo);
                    //设置资源
//                        fs.setFieldValue(LoadedApk, "mResources", apkResources);
                    //设置 Intent Intent intent = Intent { cmp=com.ywl.fva/.ui.ProxyActivity (has extras) }
                    Intent FakeIntent =(Intent) ReflectUtils.getField(arg, "intent");
                    if(FakeIntent != null){
                        Intent oriIntent = (Intent)FakeIntent.getParcelableExtra("oriIntent");
                        if(oriIntent != null){
                            Log.e(TAG, "替换 Intent intent = " + oriIntent);
                            ReflectUtils.setField(arg, "intent", oriIntent);
                        }
                    }
                    //设置 主题 theme
//                        ReflectUtils.setFieldValue(activityInfo, "theme", packageInfo.applicationInfo.theme);
                    ReflectUtils.setField(activityInfo, "theme", packageInfo.applicationInfo.theme);
 
//
                    super.beforeHookedMethod(param);
                }
            });
 
        } catch (Exception e) {
 
        }
 
 
 
        // 处理包名
        Method getPackageName = ReflectUtils.getMethod(context.getClass(), "getPackageName");
        CalvinBridge.hookMethod(getPackageName, new CA_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
            }
 
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
//                    Log.e(TAG, "getPackageName " + param.getResult());
//                    param.setResult("com.crack.testdemo");
                param.setResult(packageInfo.packageName);
                super.afterHookedMethod(param);
            }
        });
    }
```

我使用侵入式 hook ，这样才会显得我不专业，但也显得 我们的攻击方式很明确

欺上瞒下的思路。

如何加载插件 Dex 的呢？
--------------

### **DexClassLoader 的作用**

*   与 PathClassLoader 不同，DexClassLoader 支持从 APK、JAR 或 ZIP 文件中加载 Dex 文件，无需预装到系统
    
*   通过构造函数指定插件 Dex 路径、优化目录、父加载器等参数，实现动态加载：
    

```
DexClassLoader loader = new DexClassLoader(dexPath, optimizedDir, null, getClassLoader());
```

​

### ​**类加载流程**

*   ​**双亲委派模型**​：优先通过父加载器加载类，避免重复加载。若未找到，则调用`findClass`方法从 Dex 中解析类
    
*   ​**Dex 文件解析**​：Dex 文件包含所有类的索引信息，通过`DexPathList`将 Dex 转换为 Element 数组，逐个加载类
    

从这里可以了解到，要实现 dex 的加载 可以利用双亲委派机制，或者修改 dexClassloader 的`DexPathList`, 我这里有两个思路：

思路一： 我觉得可以加载 dexclassloader 直接把 原来的 dexclassloader 的父亲 classloader 修改为插件 classloader 以链表的样子插入进去

```
  public static void addPluginApkDex(File apk) {
        // 优化后的输出目录
        File optimizedDir = context.getDir("oapk", Context.MODE_PRIVATE);
        DexClassLoader dexClassLoader = new DexClassLoader(
                apk.getAbsolutePath(), // 替换为 APK 文件路径
                optimizedDir.getAbsolutePath(),
                null,
                context.getClassLoader()
        );
        Field Field_parent;
        try {
            Field_parent = ClassLoader.class.getDeclaredField("parent");
            Field_parent.setAccessible(true);
            Object parent = Field_parent.get(context.getClassLoader());
 
            Field_parent.set(context.getClassLoader(), dexClassLoader);
            Field_parent.set(dexClassLoader,parent );
        } catch (Exception e) {
//            throw new RuntimeException(e);
            Log.e(TAG, "addPluginApkDex: ", e);
        }
        Class aClass = null;
        try {
            aClass = context.getClassLoader().loadClass("com.crack.testdemo.MainActivity");
        } catch (ClassNotFoundException e) {
 
        }
        Log.d(TAG, "addPluginApkDex: "+aClass);
 
        //我认为这样做 可以优先加载 插件的 类 这个样 在插件中 就不会出现与 宿主app 拥有相同的sdk 冲突了
    }
```

思路二： 可以让直接 将插件 dexclassloader 的`DexPathList` copy 到 程序的 dexclassloader 的`DexPathList`中去

```
public static void addDexs(File dir) {
        //合并 dex 文件
//        File dir = new File(context.getFilesDir(), appItem.getPackageName());
        File[] dex_s = dir.listFiles(new FilenameFilter() {
            @Override
            public boolean accept(File dir, String name) {
                return name.contains(".dex");
            }
        });
        // 优化后的输出目录
        File optimizedDir = context.getDir("odex", Context.MODE_PRIVATE);
 
        ArrayList allElements = new ArrayList<>();
        int ElementsLen = 0;
        Field pathList = null;
        for (File dex : dex_s) {
            // 创建DexClassLoader
            DexClassLoader dexClassLoader = new DexClassLoader(
                    dex.getAbsolutePath(),
                    optimizedDir.getAbsolutePath(),
                    null,
                    context.getClassLoader()
            );
            // 获取BaseDexClassLoader中的DexPathList字段
            Class aClass = fs.getClass("dalvik.system.BaseDexClassLoader");
 
            try {
                pathList = aClass.getDeclaredField("pathList");
            } catch (NoSuchFieldException e) {
                throw new RuntimeException(e);
            }
            pathList.setAccessible(true);
            Object pathList_obj = null;
            try {
                pathList_obj = pathList.get(dexClassLoader);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
            Object[] dexElements = (Object[])fs.getFieldValue(pathList_obj, "dexElements");
 
            allElements.add(dexElements);
            ElementsLen += dexElements.length;
        }
        //加上原本的
        ClassLoader oriclassLoader = context.getClassLoader();
        Object pathList_obj =null;
        try {
            pathList_obj = pathList.get(oriclassLoader);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
 
        Object[] dexElements = (Object[])fs.getFieldValue(pathList_obj, "dexElements");
        //app 自身的dex
        allElements.add(dexElements);
        ElementsLen += dexElements.length;
 
        Object[] NewDexElements =(Object[]) Array.newInstance(
                dexElements.getClass().getComponentType(),
                ElementsLen
        );
        int index = 0;
 
        for (Object[] allElement : allElements) {
            System.arraycopy(allElement, 0, NewDexElements, index, allElement.length);
            index += allElement.length;
        }
        //添加完毕 开始设置 变量
        try {
            Field dexElements1 = pathList_obj.getClass().getDeclaredField("dexElements");
            dexElements1.setAccessible(true);
            dexElements1.set(pathList_obj, NewDexElements);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        Log.d(TAG, "handle: dexElements 变量设置完毕！");
    } 
```

这里 加载外部 dex 结束

资源加载
----

这里还有资源的加载 我写到另一个文章中去了

[https://flowus.cn/e421c3f8-12b3-4190-902f-52e4c2889eb6](elink@fe7K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6X3L8r3!0%4N6i4y4Q4x3X3g2U0L8W2)9J5c8X3f1@1x3U0q4U0x3$3j5^5i4K6u0V1x3e0u0T1x3#2)9J5k6o6b7I4z5e0m8Q4x3X3b7&6x3o6u0X3i4K6u0V1y4e0u0W2y4r3x3J5z5o6R3&6k6h3t1$3)

这里我出现的问题好多

比如 资源冲突等等

so 加载的问题呢？
----------

其实这里可以使用 io 重定向解决是比较好的方式 ，但是 楼主 为了简单省事，我的处理方案就是 提前加载 so 文件

```
public static void hook(File dir) {
      //加载so文件   思路 预加载so文件
      File soDir = new File(dir, "lib/arm64-v8a");
      if(soDir.exists()){
          File[] files = soDir.listFiles(new FilenameFilter() {
              @Override
              public boolean accept(File dir, String name) {
                  return name.contains(".so");
              }
          });
          if(files != null){
              for (File file : files) {
                  System.load(file.getAbsolutePath());
                  Log.d(TAG, "so 加载："+file.getAbsolutePath());
              }
          }else {
              Log.d(TAG, "so 文件不存在");
          }
          //拦截 so加载函数
          try {
              Class System = context.getClassLoader().loadClass("java.lang.System");
              Method loadLibrary = System.getDeclaredMethod("loadLibrary", String.class);
              CalvinBridge.hookMethod(loadLibrary, new CA_MethodHook() {
                  @Override
                  protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                      Log.d(TAG, "loadLibrary: "+param.args[0]);
                      param.args[0] = "c";
 
                  }
              });
          } catch (Exception e) {
              throw new RuntimeException(e);
          }
      }
  }
```

Service 的加载呢？
-------------

这里 Service 的加载 与 activity 差不的，只不过 service 没有 生命周期，与 ams 交互 感觉就是走个形式，那这里 我也就走个形式 ：

```
    public static  void hook(){
        //开启 欺骗 ams
        //startService(Intent service)  类 android.app.
        Method startService = ReflectUtils.getMethod(ContextWrapper.class, "startService", Intent.class);
        CalvinBridge.hookMethod(startService, new CA_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                Log.e(TAG, "startService: " );
                ReflectUtils.printFieldValues(param.args[0]);
                Intent oriintent =(Intent) param.args[0];
//                intentList.contains(oriintent)
                map.put("ori", oriintent);
                Intent Fakeintent = new Intent(BaseData.baseActivity, ProxyService.class);
                Fakeintent.setComponent(new android.content.ComponentName(BaseData.oriAppName,"com.crack.vapp.service.ProxyService"));
                Fakeintent.putExtra("oriIntent",oriintent);
                param.args[0] = Fakeintent;
                ReflectUtils.printFieldValues(param.args[0]);
                super.beforeHookedMethod(param);
            }
        });
 
 
        // hookService  handleCreateService
        Class ActivityThread = ReflectUtils.loadClass("android.app.ActivityThread");
        CalvinBridge.hookAllMethods(ActivityThread, "handleCreateService", new CA_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                Log.e(TAG, "handleCreateService: " );
                ReflectUtils.printFieldValues(param.args[0]);
                Object info = ReflectUtils.getField(param.args[0], "info");
                ReflectUtils.printFieldValues(info);
                if(info != null){
                   ReflectUtils.setField(info, "name", map.get("ori").getComponent().getClassName());
                   ReflectUtils.setField(info, "packageName", map.get("ori").getComponent().getPackageName());
                }
                super.beforeHookedMethod(param);
            }
        });
 
 
    }
```

这样做 只能启动一个 service 因为会把之前的覆盖掉！

反正我也只是学习 涨涨见识 。 就这样无论吞枣的过去吧 哈哈。

后面还有 广播 内容提供者的插件化 这些 我了解原理了 就不写了

网络上还有一堆 好文 关于插件化的。

结束
--

就这样， 劈里啪啦的代码一顿乱写，就把这个残疾版的玩具多开 app 写出来了。也是 对我半个月学习的总结吧。当然 这个项目写的很垃圾 完全不能与 网上大佬相比。

要提升 我的想法也很多，比如

加入 去签名的逻辑 我之前写的文章，

在 用 io 重定向将文件进行隔离，设置好 activityThread 类的对象 ，

这里 是我 Vapp 的项目地址 ：

[https://github.com/ywl20020421/Vapp](elink@b09K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6&6N6$3H3J5x3o6l9J5x3o6b7J5x3g2)9J5c8W2k6S2M7s2l9`.)

支持 安卓 12 ，可以加载 安卓 android studio 编写出来的小 demo 实际测试 很多 app 都加载不了

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#基础理论](forum-161-1-117.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm)