> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267048.htm)

目录

*   Xposed ART 原理分析笔记
*            0. 写在前面
*            1. Xposed 结构
*            2. `Xposed` 注入进程
*            3. `Xposed` `hook` 函数
*            4. 被 `hook` 的函数执行流程分析
*            5. 参考资料

Xposed ART 原理分析笔记
=================

0. 写在前面
-------

这篇文章是我在看`Xposed`源码时所做的一些笔记内容，中间也参考了一些前人的分析文章这里都在最后列出了，除此之外还有一些内容纯属个人理解，如果有错误欢迎各位大佬斧正，感谢！

1. Xposed 结构
------------

Xposed 相关源码: https://github.com/rovo89

1.  XposedBridge: Xposed 提供的 jar 文件，app_process 启动过程中会加载该 jar 包，其 他的 Modules 的开发都是基于该 jar 包;
2.  Xposed: Xposed 的 C++ 部 分 ， 主 要 是 用 来 替 换 /system/bin/app_process ， 并 为 XposedBridge 提供 JNI 方法;
3.  XposedInstaller: Xposed 的安装包，提供对基于 Xposed 框架的 Modules 的管理;
4.  android_art: Xposed 修改的 art 部分源码

2. `Xposed` 注入进程
----------------

`Xposed`注入进程的方式是通过`zygote`进程在`fork`出`APP`进程时使之加载`XposedBridge.jar`；

 

根据`Xposed`的`Android.mk`文件可知在 5.0 以上的手机上编译的是`app_main2.cpp`

```
ifeq (1,$(strip $(shell expr $(PLATFORM_SDK_VERSION) \>= 21))) # if Android_Version >= 5.0
  LOCAL_SRC_FILES := app_main2.cpp
  LOCAL_MULTILIB := both
  LOCAL_MODULE_STEM_32 := app_process32_xposed
  LOCAL_MODULE_STEM_64 := app_process64_xposed
else
  LOCAL_SRC_FILES := app_main.cpp
  LOCAL_MODULE_STEM := app_process_xposed
endif

```

这里选取`Android 7.1.1_r6`的`app_main.cpp`文件与`app_main2.cpp`做对比

 

从`main`函数看起

 

![](https://bbs.pediy.com/upload/attach/202104/715334_2CCXG3XP6PGEFPP.png)

 

从文件的对比可以发现，`app_main2.cpp`中多出了对于 `Xposed::handleOptions`的调用，而这个函数主要是对`--xposedversion`的处理以及是否测试模式的启用，真正运行时无需理会

```
bool handleOptions(int argc, char* const argv[]) {
    // version 信息
    parseXposedProp();
 
    if (argc == 2 && strcmp(argv[1], "--xposedversion") == 0) {
        printf("Xposed version: %s\n", xposedVersion);
        return true;
    }
 
    if (argc == 2 && strcmp(argv[1], "--xposedtestsafemode") == 0) {
        printf("Testing Xposed safemode trigger\n");
 
        if (detectSafemodeTrigger(shouldSkipSafemodeDelay())) {
            printf("Safemode triggered\n");
        } else {
            printf("Safemode not triggered\n");
        }
        return true;
    }
 
    // From Lollipop coding, used to override the process name
    argBlockStart = argv[0];
    uintptr_t start = reinterpret_cast(argv[0]);
    uintptr_t end = reinterpret_cast(argv[argc - 1]);
    end += strlen(argv[argc - 1]) + 1;
    argBlockLength = end - start;
 
    return false;
} 
```

在`main`函数的最后，增加了对`Xposed`的加载与`runtimeStart`函数的调用。

 

![](https://bbs.pediy.com/upload/attach/202104/715334_P6QHKFKE8JT7K8P.png)

 

在`initialize`函数最终返回`XposedBridge.jar`是否成功加入环境变量的标志和`XposedInstaller`是否成功运行的标志

 

首先对`xposed`这个结构体变量进行赋值

```
struct XposedShared {
    // Global variables
    bool zygote;
    bool startSystemServer;
    const char* startClassName;
    uint32_t xposedVersionInt;
    bool isSELinuxEnabled;
    bool isSELinuxEnforcing;
    uid_t installer_uid;
    gid_t installer_gid;
 
    // Provided by runtime-specific library, used by executable
    void (*onVmCreated)(JNIEnv* env);
 
#if XPOSED_WITH_SELINUX
    // Provided by the executable, used by runtime-specific library
    int (*zygoteservice_accessFile)(const char* path, int mode);
    int (*zygoteservice_statFile)(const char* path, struct stat* st);
    char* (*zygoteservice_readFile)(const char* path, int* bytesRead);
#endif
};
XposedShared* xposed = new XposedShared;   
xposed->zygote = zygote;
xposed->startSystemServer = startSystemServer;
xposed->startClassName = className;
xposed->xposedVersionInt = xposedVersionInt;
 
#if XPOSED_WITH_SELINUX
    xposed->isSELinuxEnabled   = is_selinux_enabled() == 1;
    xposed->isSELinuxEnforcing = xposed->isSELinuxEnabled && security_getenforce() == 1;
#else
    xposed->isSELinuxEnabled   = false;
    xposed->isSELinuxEnforcing = false;
#endif  // XPOSED_WITH_SELINUX

```

然后在`logcat`中通过`printStartupMarker`函数和`printRomInfo`函数打印一些信息

```
void printStartupMarker() {
    sprintf(marker, "Current time: %d, PID: %d", (int) time(NULL), getpid());
    ALOG(LOG_DEBUG, "XposedStartupMarker", marker, NULL);
}
void printRomInfo() {
    char release[PROPERTY_VALUE_MAX];
    char sdk[PROPERTY_VALUE_MAX];
    char manufacturer[PROPERTY_VALUE_MAX];
    char model[PROPERTY_VALUE_MAX];
    char rom[PROPERTY_VALUE_MAX];
    char fingerprint[PROPERTY_VALUE_MAX];
    char platform[PROPERTY_VALUE_MAX];
#if defined(__LP64__)
    const int bit = 64;
#else
    const int bit = 32;
#endif
 
    property_get("ro.build.version.release", release, "n/a");
    property_get("ro.build.version.sdk", sdk, "n/a");
    property_get("ro.product.manufacturer", manufacturer, "n/a");
    property_get("ro.product.model", model, "n/a");
    property_get("ro.build.display.id", rom, "n/a");
    property_get("ro.build.fingerprint", fingerprint, "n/a");
    property_get("ro.product.cpu.abi", platform, "n/a");
 
    ALOGI("-----------------");
    ALOGI("Starting Xposed version %s, compiled for SDK %d", xposedVersion, PLATFORM_SDK_VERSION);
    ALOGI("Device: %s (%s), Android version %s (SDK %s)", model, manufacturer, release, sdk);
    ALOGI("ROM: %s", rom);
    ALOGI("Build fingerprint: %s", fingerprint);
    ALOGI("Platform: %s, %d-bit binary, system server: %s", platform, bit, xposed->startSystemServer ? "yes" : "no");
    if (!xposed->zygote) {
        ALOGI("Class name: %s", xposed->startClassName);
    }
    ALOGI("SELinux enabled: %s, enforcing: %s",
            xposed->isSELinuxEnabled ? "yes" : "no",
            xposed->isSELinuxEnforcing ? "yes" : "no");
}

```

然后以下函数通过启动`XposedInstaller`和相应`Xposed`服务

```
if (!determineXposedInstallerUidGid() || !xposed::service::startAll()) {
            return false;
}

```

最后如果`XPosed`能够正常加载，那么就通过函数`addJarToClasspath` 将`XposedBridge.jar`文件加入环境变量

 

这里如果想要禁用`Xposed`只需要在`XposedInstaller`的私有目录下创建一个`conf/disabled`文件即可。

```
/** Create a flag file to disable Xposed. */
#if PLATFORM_SDK_VERSION >= 24
#define XPOSED_DIR "/data/user_de/0/de.robv.android.xposed.installer/"
#else
#define XPOSED_DIR "/data/data/de.robv.android.xposed.installer/"
#endif
 
#define XPOSED_LOAD_BLOCKER      XPOSED_DIR "conf/disabled"
void disableXposed() {
    int fd;
    // FIXME add a "touch" operation to xposed::service::membased
    fd = open(XPOSED_LOAD_BLOCKER, O_WRONLY | O_CREAT, S_IRUSR | S_IWUSR);
    if (fd >= 0)
        close(fd);
}

```

如果成功加载则调用`runtimeStart`函数对调用`AndroidRuntime::start()`函数对`de.robv.android.xposed.XposedBridge`类进行启动

```
// XPOSED_CLASS_DOTS_ZYGOTE =  de.robv.android.xposed.XposedBridge
runtimeStart(runtime, isXposedLoaded ? "de.robv.android.xposed.XposedBridge" : "com.android.internal.os.ZygoteInit", args, zygote);

```

而`AndroidRuntime::start()`函数经过观察发现实际上完成了几个工作

 

1.`startVm`函数，启动虚拟机

```
/* start the virtual machine */
  JniInvocation jni_invocation;
  jni_invocation.Init(NULL);
  JNIEnv* env;
  if (startVm(&mJavaVM, &env, zygote) != 0) {
      return;
  }
  onVmCreated(env);

```

2. 反射调用传入的类的`main`函数作为虚拟机的入口点

 

![](https://bbs.pediy.com/upload/attach/202104/715334_K9KXZKSZ2AE45Z8.png)

 

而`Xposed`就是通过替换这个`className`来达到加载自己的`art`虚拟机的效果。这样就成功注入了`XposedBridge.jar`文件，达到任意`APP`在加载时内存空间中总会有`XposedBridge.jar`。

 

而`XposedBridge.jar`文件的`main`函数如下，其实就是一个对`SELinux`的处理以及对`Xposed`模块的加载。

```
protected static void main(String[] args) {
        // Initialize the Xposed framework and modules
        try {
            if (!hadInitErrors()) {
                initXResources();
 
                SELinuxHelper.initOnce();
                SELinuxHelper.initForProcess(null);
 
                runtime = getRuntime();
                XPOSED_BRIDGE_VERSION = getXposedVersion();
 
                if (isZygote) {
          // 暂且忽略
                    XposedInit.hookResources();
                    XposedInit.initForZygote();
                }
            // 模块的加载
                XposedInit.loadModules();
            } else {
                Log.e(TAG, "Not initializing Xposed because of previous errors");
            }
        } catch (Throwable t) {
            Log.e(TAG, "Errors during Xposed initialization", t);
            disableHooks = true;
        }
 
        // Call the original startup code
        if (isZygote) {
            ZygoteInit.main(args);
        } else {
            RuntimeInit.main(args);
        }
    }

```

对模块的加载是通过对`XposedInstaller`私有目录下的`conf/modules.list`文件进行读取, 并分别是通过`BOOTCLASSLOADER`对模块进行加载。

```
/*package*/ static void loadModules() throws IOException {
        final String filename = BASE_DIR + "conf/modules.list";
        BaseService service = SELinuxHelper.getAppDataFileService();
        if (!service.checkFileExists(filename)) {
            Log.e(TAG, "Cannot load any modules because " + filename + " was not found");
            return;
        }
 
        ClassLoader topClassLoader = XposedBridge.BOOTCLASSLOADER;
        ClassLoader parent;
        while ((parent = topClassLoader.getParent()) != null) {
            topClassLoader = parent;
        }
 
        InputStream stream = service.getFileInputStream(filename);
        BufferedReader apks = new BufferedReader(new InputStreamReader(stream));
        String apk;
        while ((apk = apks.readLine()) != null) {
      // 按行加载模块
            loadModule(apk, topClassLoader);
        }
        apks.close();
    }

```

在最终的`loadModule`函数中实际上是通过`DexFile`的构造函数将模块加载并通过`dexFile.loadClass`函数的方式对入口类进行加载。最终达到模块注入的效果

```
private static void loadModule(String apk, ClassLoader topClassLoader) {       
  ...
        DexFile dexFile;
        try {
            dexFile = new DexFile(apk);
        } catch (IOException e) {
            Log.e(TAG, "  Cannot load module", e);
            return;
        }
        // Instant Run的处理
        if (dexFile.loadClass(INSTANT_RUN_CLASS, topClassLoader) != null) {
            Log.e(TAG, "  Cannot load module, please disable \"Instant Run\" in Android Studio.");
            closeSilently(dexFile);
            return;
        }
 
        if (dexFile.loadClass(XposedBridge.class.getName(), topClassLoader) != null) {
            Log.e(TAG, "  Cannot load module:");
            Log.e(TAG, "  The Xposed API classes are compiled into the module's APK.");
            Log.e(TAG, "  This may cause strange issues and must be fixed by the module developer.");
            Log.e(TAG, "  For details, see: http://api.xposed.info/using.html");
            closeSilently(dexFile);
            return;
        }
  ...
}

```

3. `Xposed` `hook`函数
--------------------

我们知道`Xposed` `hook`函数的方式是大致是通过以下模版完成的

```
public class XposedHook implements IXposedHookLoadPackage {
    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam loadPackageParam) throws Throwable {
            Class clasz = loadPackageParam.classLoader.loadClass("xxxx"); //要hook的方法所在的类名
 
            XposedHelpers.findAndHookMethod(clazz, "xxx",String.class,String.class,String.class, new XC_MethodHook() { //要hook的方法名和参数类型，此处为三个String类型
 
                @Override //重写XC_MethodHook()的回调方法
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
 
 
                }
 
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    Log.i("hook after result:",param.getResult().toString()); //打印返回值（String类型）
                }
            });
        }
    }
}

```

其中最关键的执行`hook`逻辑的函数实际上是`XposedHelpers.findAndHookMethod()`函数

 

我们从这个函数跟踪起

 

![](https://bbs.pediy.com/upload/attach/202104/715334_UBZWA4QNUZWPC2K.png)

 

观察这个函数分为两个部分

1.  找到回调`CallBack`和`hook`的函数
2.  调用`XposedBridge.hookMethod()`函数完成`hook`。

第一部分，用于获取回调的部分实际上就是获取参数的最后一个值，而用于寻找`hook`对应函数的`findMethodExact`函数, 如下所示实际上就是通过反射获取，具体代码如下：

```
public static Method findMethodExact(Class... parameterTypes) {
        String fullMethodName = clazz.getName() + '#' + methodName + getParametersString(parameterTypes) + "#exact";
        // 缓存机制
        if (methodCache.containsKey(fullMethodName)) {
            Method method = methodCache.get(fullMethodName);
            if (method == null)
                throw new NoSuchMethodError(fullMethodName);
            return method;
        }
 
        try {
      // 真实逻辑
            Method method = clazz.getDeclaredMethod(methodName, parameterTypes);
            method.setAccessible(true);
            methodCache.put(fullMethodName, method);
            return method;
        } catch (NoSuchMethodException e) {
            methodCache.put(fullMethodName, null);
            throw new NoSuchMethodError(fullMethodName);
        }
    }

```

第二部分，执行`hook`逻辑的函数其内容主要分为三个部分

 

1. 检查要`hook`的函数是否合法，必须同时满足三个条件：第一，是普通函数或者构造函数；第二，所在类不是接口`Interface`；第三，函数不是`abstract`抽象函数。

 

![](https://bbs.pediy.com/upload/attach/202104/715334_DX8EEYBCBN8D4J6.png)  
2. 从缓存中确认函数未被`hook`并将新函数加入缓存。

 

![](https://bbs.pediy.com/upload/attach/202104/715334_RRE4UP8YZ72RRUA.png)

 

3. 执行真实`hook`逻辑，具体代码如下。可以发现真实执行`hook`逻辑的函数交给了`hookMethodNative`函数，而这个函数实际上是一个`native`属性的函数。

 

![](https://bbs.pediy.com/upload/attach/202104/715334_9J5AU6KP4EUN8C4.png)

 

而这个函数的`C++`实现在`libxposed_art.cpp`文件中。观察其函数发现，实际上就是将`Java`层的`Method`转换成了`ArtMethod`，然后通过`ArtMethod`类的`EnableXposedHook`函数执行`hook`。

```
void XposedBridge_hookMethodNative(JNIEnv* env, jclass, jobject javaReflectedMethod,
            jobject, jint, jobject javaAdditionalInfo) {
    // Detect usage errors.
    // ScopedObjectAccess：访问管理对象时候调用，一是将jobject加入LocalReference二是线程切换为kRunnable状态，即离开安全区。
    ScopedObjectAccess soa(env);
    if (javaReflectedMethod == nullptr) {
#if PLATFORM_SDK_VERSION >= 23
        ThrowIllegalArgumentException("method must not be null");
#else
        ThrowIllegalArgumentException(nullptr, "method must not be null");
#endif
        return;
    }
 
    // Get the ArtMethod of the method to be hooked.
    ArtMethod* artMethod = ArtMethod::FromReflectedMethod(soa, javaReflectedMethod);
 
    // Hook the method
    artMethod->EnableXposedHook(soa, javaAdditionalInfo);
}

```

而`EnableXposedHook`函数的实现就在`Xposed`修改的`art`代码中了，打开`android_art`工程，找到对应实现`runtime/art_method.cc`文件。

 

具体实现分为几步

 

1. 备份原先的`Method`并添加标记`kAccXposedOriginalMethod`

```
// Create a backup of the ArtMethod object
 auto* cl = Runtime::Current()->GetClassLinker();
 auto* linear_alloc = cl->GetAllocatorForClassLoader(GetClassLoader());
 ArtMethod* backup_method = cl->CreateRuntimeMethod(linear_alloc);
 backup_method->CopyFrom(this, cl->GetImagePointerSize());
 backup_method->SetAccessFlags(backup_method->GetAccessFlags() | kAccXposedOriginalMethod);

```

2. 创建一个备份方法对应的反射对象

```
// Create a Method/Constructor object for the backup ArtMethod object
  mirror::AbstractMethod* reflected_method;
  if (IsConstructor()) {
    reflected_method = mirror::Constructor::CreateFromArtMethod(soa.Self(), backup_method);
  } else {
    reflected_method = mirror::Method::CreateFromArtMethod(soa.Self(), backup_method);
  }
  reflected_method->SetAccessible(true); 
```

3. 将所有相关内容保存为一个结构体存储

```
XposedHookInfo* hook_info = reinterpret_cast(linear_alloc->Alloc(soa.Self(), sizeof(XposedHookInfo)));
hook_info->reflected_method = soa.Vm()->AddGlobalRef(soa.Self(), reflected_method);
hook_info->additional_info = soa.Env()->NewGlobalRef(additional_info);
hook_info->original_method = backup_method; 
```

4. 准备工作，处理函数的 JIT 即时编译以及其他

```
// suspend线程
 ScopedThreadSuspension sts(soa.Self(), kSuspended);
 // 停止JIT即时编译
 jit::ScopedJitSuspend sjs;
// 防止死锁
 gc::ScopedGCCriticalSection gcs(soa.Self(),
                                 gc::kGcCauseXposed,
                                 gc::kCollectorTypeXposed);
// 用来暂停占用函数的所有线程
 ScopedSuspendAll ssa(__FUNCTION__);
 // 取消所有调用
 cl->InvalidateCallersForMethod(soa.Self(), this);
 
 // 去除JIT编译的结果
 jit::Jit* jit = art::Runtime::Current()->GetJit();
 if (jit != nullptr) {
   jit->GetCodeCache()->MoveObsoleteMethod(this, backup_method);
 }

```

5. 设置被`hook`函数的入口点（关键的 hook 逻辑）

```
// 将hook信息存到entry_point_from_jni这个指针
SetEntryPointFromJniPtrSize(reinterpret_cast(hook_info), sizeof(void*));
// 设替换函数入口点entry_point_from_quick_compiled_code_为自己的art_quick_proxy_invoke_handler
SetEntryPointFromQuickCompiledCode(GetQuickProxyInvokeHandler());
// 设置函数在CodeItem偏移
SetCodeItemOffset(0);
 
// Adjust access flags.
// 更改属性并添加kAccXposedHookedMethod标记
const uint32_t kRemoveFlags = kAccNative | kAccSynchronized | kAccAbstract | kAccDefault | kAccDefaultConflict;
SetAccessFlags((GetAccessFlags() & ~kRemoveFlags) | kAccXposedHookedMethod); 
```

6. 恢复环境

```
// 恢复线程
  MutexLock mu(soa.Self(), *Locks::thread_list_lock_);
  Runtime::Current()->GetThreadList()->ForEach(StackReplaceMethodAndInstallInstrumentation, this);

```

这样就执行完成了函数的`hook`

4. 被`hook`的函数执行流程分析
-------------------

当被`hook`的函数执行时，我们直接从`ArtMethod->Invoke`函数入手。

 

由于被`hook`的函数属性是一个`native`属性, 观察函数中重要逻辑。由于在设置函数时已经设置过函数入口点为`entry_point_from_quick_compiled_code_`不为`false`则会进入`art_quick_invoke_stub`或者`art_quick_invoke_static_stub`跳板函数。

```
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
  ...
      bool have_quick_code = GetEntryPointFromQuickCompiledCode() != nullptr;
    if (LIKELY(have_quick_code)) {
      if (kLogInvocationStartAndReturn) {
        LOG(INFO) << StringPrintf(
            "Invoking '%s' quick code=%p static=%d", PrettyMethod(this).c_str(),
            GetEntryPointFromQuickCompiledCode(), static_cast(IsStatic() ? 1 : 0));
      }
 
      // Ensure that we won't be accidentally calling quick compiled code when -Xint.
      if (kIsDebugBuild && runtime->GetInstrumentation()->IsForcedInterpretOnly()) {
        CHECK(!runtime->UseJitCompilation());
        const void* oat_quick_code = runtime->GetClassLinker()->GetOatMethodQuickCodeFor(this);
        CHECK(oat_quick_code == nullptr || oat_quick_code != GetEntryPointFromQuickCompiledCode())
            << "Don't call compiled code when -Xint " << PrettyMethod(this);
      }
 
      if (!IsStatic()) {
        (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
      } else {
        (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
      }
      ....
    }
    ...
} 
```

这里以`art_quick_invoke_stub`非静态函数为例，最终不管是`arm`还是`arm64`都是以汇编实现的这个函数，只是`arm`在真实执行函数时有一些中间的跳板，最终实现函数为`art_quick_invoke_stub_internal`。

 

以`arm`为例，其实现函数所在文件为`android_art/runtime/arch/arm/quick_entrypoints_arm.S`，汇编中存在着一个`ART_METHOD_QUICK_CODE_OFFSET_32`这个函数的调用。

```
ENTRY art_quick_invoke_stub_internal
    ....
 
#ifdef ARM_R4_SUSPEND_FLAG
    mov    r4, #SUSPEND_CHECK_INTERVAL     @ reset r4 to suspend check interval
#endif
 
    ldr    ip, [r0, #ART_METHOD_QUICK_CODE_OFFSET_32]  @ get pointer to the code
    blx    ip                              @ call the method
 
    mov    sp, r11                         @ restore the stack pointer
    .cfi_def_cfa_register sp
 
    ldr    r4, [sp, #40]                   @ load result_is_float
    ldr    r9, [sp, #36]                   @ load the result pointer
    cmp    r4, #0
    ite    eq
    strdeq r0, [r9]                        @ store r0/r1 into result pointer
    vstrne d0, [r9]                        @ store s0-s1/d0 into result pointer
 
    pop    {r4, r5, r6, r7, r8, r9, r10, r11, pc}               @ restore spill regs
END art_quick_invoke_stub_internal

```

而`ART_METHOD_QUICK_CODE_OFFSET_32`函数定义在`runtime/asm_support.h`文件中实现如下

```
struct PACKED(4) PtrSizedFields {
    // Short cuts to declaring_class_->dex_cache_ member for fast compiled code access.
    ArtMethod** dex_cache_resolved_methods_;
 
    // Short cuts to declaring_class_->dex_cache_ member for fast compiled code access.
    GcRoot* dex_cache_resolved_types_;
 
    // Pointer to JNI function registered to this method, or a function to resolve the JNI function,
    // or the profiling data for non-native methods, or an ImtConflictTable.
    void* entry_point_from_jni_;
 
    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into
    // the interpreter.
    void* entry_point_from_quick_compiled_code_;
 } ptr_sized_fields_;
static MemberOffset EntryPointFromQuickCompiledCodeOffset(size_t pointer_size) {
    return MemberOffset(PtrSizedFieldsOffset(pointer_size) + OFFSETOF_MEMBER(
        PtrSizedFields, entry_point_from_quick_compiled_code_) / sizeof(void*) * pointer_size);
 }
#define ART_METHOD_QUICK_CODE_OFFSET_32 32
ADD_TEST_EQ(ART_METHOD_QUICK_CODE_OFFSET_32,
            art::ArtMethod::EntryPointFromQuickCompiledCodeOffset(4).Int32Value()) 
```

`EntryPointFromQuickCompiledCodeOffset`函数是一个获取`PtrSizedFields`结构体固定偏移的函数，这里就是获取的`entry_point_from_quick_compiled_code_`变量的值。

 

这里由于`Xposed`对这个函数进行了`hook`，实际上就是获取的`art_quick_proxy_invoke_handler`函数的地址。

```
ENTRY art_quick_proxy_invoke_handler
    SETUP_REFS_AND_ARGS_CALLEE_SAVE_FRAME_WITH_METHOD_IN_R0
    mov     r2, r9                 @ pass Thread::Current
    mov     r3, sp                 @ pass SP
    @ 调用函数
    blx     artQuickProxyInvokeHandler  @ (Method* proxy method, receiver, Thread*, SP)
    ldr     r2, [r9, #THREAD_EXCEPTION_OFFSET]  @ load Thread::Current()->exception_
    // Tear down the callee-save frame. Skip arg registers.
    add     sp, #(FRAME_SIZE_REFS_AND_ARGS_CALLEE_SAVE - FRAME_SIZE_REFS_ONLY_CALLEE_SAVE)
    .cfi_adjust_cfa_offset -(FRAME_SIZE_REFS_AND_ARGS_CALLEE_SAVE - FRAME_SIZE_REFS_ONLY_CALLEE_SAVE)
    RESTORE_REFS_ONLY_CALLEE_SAVE_FRAME
    cbnz    r2, 1f                 @ success if no exception is pending
    vmov    d0, r0, r1             @ store into fpr, for when it's a fpr return...
    bx      lr                     @ return on success
1:
    DELIVER_PENDING_EXCEPTION
END art_quick_proxy_invoke_handler

```

故最终在`art_quick_invoke_stub_internal`也就是调用的`art_quick_proxy_invoke_handler`函数。而在这个`art_quick_proxy_invoke_handler`函数中，又再次调用了`artQuickProxyInvokeHandler`函数。

 

在`artQuickProxyInvokeHandler`函数中又看到了一堆`xposed`相关的信息，其具体实现在`android_art/runtime/entrypoints/quick/quick_trampoline_entrypoints.cc`。

 

这里我梳理了一下`Xposed`相关的重要代码列出来，具体如下：

```
extern "C" uint64_t artQuickProxyInvokeHandler(
    ArtMethod* proxy_method, mirror::Object* receiver, Thread* self, ArtMethod** sp)
    SHARED_REQUIRES(Locks::mutator_lock_) {
  const bool is_xposed = proxy_method->IsXposedHookedMethod();
  ...
  if (is_xposed) {
    jmethodID proxy_methodid = soa.EncodeMethod(proxy_method);
    self->EndAssertNoThreadSuspension(old_cause);
    JValue result = InvokeXposedHandleHookedMethod(soa, shorty, rcvr_jobj, proxy_methodid, args);
    local_ref_visitor.FixupReferences();
    return result.GetJ();
  }
}

```

观察发现实际上最重要的就是`InvokeXposedHandleHookedMethod`函数的调用，其具体内容主要分成三部分。

 

1. 处理参数信息

```
if (args.size() > 0 || (target_sdk_version > 0 && target_sdk_version <= 21)) {
    args_jobj = soa.Env()->NewObjectArray(args.size(), WellKnownClasses::java_lang_Object, nullptr);
    if (args_jobj == nullptr) {
      CHECK(soa.Self()->IsExceptionPending());
      return zero;
    }
    for (size_t i = 0; i < args.size(); ++i) {
      if (shorty[i + 1] == 'L') {
        jobject val = args.at(i).l;
        soa.Env()->SetObjectArrayElement(args_jobj, i, val);
      } else {
        JValue jv;
        jv.SetJ(args.at(i).j);
        mirror::Object* val = BoxPrimitive(Primitive::GetType(shorty[i + 1]), jv);
        if (val == nullptr) {
          CHECK(soa.Self()->IsExceptionPending());
          return zero;
        }
        soa.Decode* >(args_jobj)->Set(i, val);
      }
    }
  } 
```

2. 调用`XposedBridge.handleHookedMethod()`函数

```
    const XposedHookInfo* hook_info = soa.DecodeMethod(method)->GetXposedHookInfo();
 
  // Call XposedBridge.handleHookedMethod(Member method, int originalMethodId, Object additionalInfoObj,
  //                                      Object thisObject, Object[] args)
// 利用jvalue传递参数
  jvalue invocation_args[5];
  invocation_args[0].l = hook_info->reflected_method;
  invocation_args[1].i = 1;
  invocation_args[2].l = hook_info->additional_info;
  invocation_args[3].l = rcvr_jobj;
  invocation_args[4].l = args_jobj;
  /*
   #define CLASS_XPOSED_BRIDGE  "de/robv/android/xposed/XposedBridge"
    classXposedBridge = env->FindClass(CLASS_XPOSED_BRIDGE);
    ArtMethod::xposed_callback_class = classXposedBridge;
     methodXposedBridgeHandleHookedMethod = env->GetStaticMethodID(classXposedBridge, "handleHookedMethod",
       "(Ljava/lang/reflect/Member;ILjava/lang/Object;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;");
    ArtMethod::xposed_callback_method = methodXposedBridgeHandleHookedMethod;
  */
  jobject result =
      soa.Env()->CallStaticObjectMethodA(ArtMethod::xposed_callback_class,
                                         ArtMethod::xposed_callback_method,
                                         invocation_args);

```

3. 处理结果并返回

```
// Unbox the result if necessary and return it.
  if (UNLIKELY(soa.Self()->IsExceptionPending())) {
    return zero;
  } else {
    if (shorty[0] == 'V' || (shorty[0] == 'L' && result == nullptr)) {
      return zero;
    }
    // This can cause thread suspension.
    size_t pointer_size = Runtime::Current()->GetClassLinker()->GetImagePointerSize();
    mirror::Class* result_type = soa.DecodeMethod(method)->GetReturnType(true /* resolve */, pointer_size);
    mirror::Object* result_ref = soa.Decode(result);
    JValue result_unboxed;
    if (!UnboxPrimitiveForResult(result_ref, result_type, &result_unboxed)) {
      DCHECK(soa.Self()->IsExceptionPending());
      return zero;
    }
    return result_unboxed;
  } 
```

在这三个部分中最最重要的实际上是第二部分。

 

其中首先通过`GetXposedHookInfo()`函数调用`GetEntryPointFromJniPtrSize()`函数获取`Xposed`在设置函数`hook`时保存在`entry_point_from_jni_`中的`XposedHookInfo`对象信息。

```
const XposedHookInfo* GetXposedHookInfo() {
    DCHECK(IsXposedHookedMethod());
    return reinterpret_cast(GetEntryPointFromJniPtrSize(sizeof(void*)));
}
const XposedHookInfo* hook_info = soa.DecodeMethod(method)->GetXposedHookInfo(); 
```

然后拼接参数并通过`CallStaticObjectMethodA()`函数`JNI`调用`XposedBridge.handleHookedMethod(Member method, int originalMethodId, Object additionalInfoObj,Object thisObject, Object[] args)`函数这样就再次回到`Java`层中。

 

此时再次回到`XposedBridge`的源码, 找到对应`handleHookedMethod`函数的实现会发现

1.  先调用了`CallBack`中的`beforeHookedMethod`函数。

```
// call "before method" callbacks
        int beforeIdx = 0;
        do {
            try {
                ((XC_MethodHook) callbacksSnapshot[beforeIdx]).beforeHookedMethod(param);
            } catch (Throwable t) {
                XposedBridge.log(t);
 
                // reset result (ignoring what the unexpectedly exiting callback did)
                param.setResult(null);
                param.returnEarly = false;
                continue;
            }
 
            if (param.returnEarly) {
                // skip remaining "before" callbacks and corresponding "after" callbacks
                beforeIdx++;
                break;
            }
        } while (++beforeIdx < callbacksLength);

```

2. 然后通过`invokeOriginalMethodNative`调用原函数

```
// call original method if not requested otherwise
        if (!param.returnEarly) {
            try {
                param.setResult(invokeOriginalMethodNative(method, originalMethodId,
                        additionalInfo.parameterTypes, additionalInfo.returnType, param.thisObject, param.args));
            } catch (InvocationTargetException e) {
                param.setThrowable(e.getCause());
            }
        }

```

3. 最后执行所有的`afterHookedMethod`回调。

```
// call "after method" callbacks
        int afterIdx = beforeIdx - 1;
        do {
            Object lastResult =  param.getResult();
            Throwable lastThrowable = param.getThrowable();
 
            try {
                ((XC_MethodHook) callbacksSnapshot[afterIdx]).afterHookedMethod(param);
            } catch (Throwable t) {
                XposedBridge.log(t);
 
                // reset to last result (ignoring what the unexpectedly exiting callback did)
                if (lastThrowable == null)
                    param.setResult(lastResult);
                else
                    param.setThrowable(lastThrowable);
            }
        } while (--afterIdx >= 0);

```

其中`invokeOriginalMethodNative`函数则是通过保存的`reflected_method`反射对象对原函数进行反射调用，也就是通过原`art`函数`InvokeMethod()`函数进行调用。

```
jobject XposedBridge_invokeOriginalMethodNative(JNIEnv* env, jclass, jobject javaMethod,
            jint isResolved, jobjectArray, jclass, jobject javaReceiver, jobjectArray javaArgs) {
    ScopedFastNativeObjectAccess soa(env);
    if (UNLIKELY(!isResolved)) {
        ArtMethod* artMethod = ArtMethod::FromReflectedMethod(soa, javaMethod);
        if (LIKELY(artMethod->IsXposedHookedMethod())) {
            javaMethod = artMethod->GetXposedHookInfo()->reflected_method;
        }
    }
#if PLATFORM_SDK_VERSION >= 23
    return InvokeMethod(soa, javaMethod, javaReceiver, javaArgs);
#else
    return InvokeMethod(soa, javaMethod, javaReceiver, javaArgs, true);
#endif
}

```

5. 参考资料
-------

https://blog.csdn.net/weixin_47883636/article/details/109018440

 

https://egguncle.github.io/2018/02/04/xposed-art-hook-%E6%B5%85%E6%9E%90/

 

https://bbs.pediy.com/thread-257844.htm

[[公告] 2021 KCTF 春季赛 防守方征题火热进行中！](https://bbs.pediy.com/thread-266222.htm)

最后于 23 小时前 被 Simp1er 编辑 ，原因：