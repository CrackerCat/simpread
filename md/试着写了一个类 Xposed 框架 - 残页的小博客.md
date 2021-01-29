> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.canyie.top](https://blog.canyie.top/2020/02/03/a-new-xposed-style-framework/)

新人第一次写博客，勿喷..  
本文也发布在[知乎](https://zhuanlan.zhihu.com/p/104871958)上

Xposed 框架在 Android 上是神器般的存在，它给了普通用户随意定制系统的能力，各种骚操作层出不穷。随着咱对 Android 的了解越来越深（其实一点都不深..），逐渐冒出了自己写一个类 Xposed 框架的想法，最终搞出了这个勉强能用的半成品。  
代码在这：[Dreamland](https://github.com/canyie/Dreamland) & [Dreamland Manager](https://github.com/canyie/DreamlandManager) ，代码写的很辣鸡，求轻喷 QAQ  
接下来会介绍一下实现细节与遇到的问题。

[](#注入-zygote-进程 "注入 zygote 进程")注入 zygote 进程
--------------------------------------------

我们想实现 Xposed 那样在目标进程加载自己的模块，就必须把我们自己的代码注入到目标进程，而且我们的代码执行的时机还需要足够早，一般来说都是选择直接注入到 zygote 进程。

先来看看其他框架的实现：

*   [Xposed](https://github.com/rovo89/Xposed) ：Xposed for art 重新实现了 app_process，libart.so 等重要系统库，安装时会替换这些文件，而各大厂商几乎没有不修改它们的，一旦被替换很可能变砖，导致 Xposed 在非原生系统上的稳定性很差。
*   [EdXposed](https://github.com/ElderDrivers/EdXposed) : EdXp 依赖 [Riru](https://github.com/RikkaApps/Riru) 而 Riru 是通过替换 libmemtrack.so 来实现，这个 so 库会在 zygote 进程启动时被加载，并且比 libart 轻得多（只有 10 个导出函数），然后就可以在 zygote 进程里执行任意代码。
*   [太极阳](https://github.com/taichi-framework/TaiChi) : 太极阳通过一种我看不懂的魔法（看了一下只发现 libjit.so，但 weishu 表示 Android 系统里并没有一个这样一个库，所以并不是简单替换 so）注入进 zygote（以前是替换 libprocessgroup.so）

可以看出，其他框架几乎都通过直接替换系统已有的 so 库实现，而替换已有 so 库则需要尽量选择较轻的库，以避免厂商的修改导致的问题。然而，我们没法避免厂商在 so 里加料，如果厂商修改了这个 so 库，我们直接把我们自己以 AOSP 为蓝本写的 so 替换上去，则会导致严重的问题。  
有没有别的什么办法？下面介绍梦境的实现方式。  
（注：如无特别说明，本文中的 AOSP 源码都是 [7.0.0_r36](http://aospxref.com/android-7.0.0_r36) 对应的代码）  
我们知道，在 android 中，所有的应用进程都是由 zygote 进程 fork 出来的，而 zygote 对应的可执行文件就是 app_process（具体可以看 init.rc）  
app_process 的 main 方法如下：

```
int main(int argc, char* const argv[])
{
  
  AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
  
  if (zygote) {
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
  } else if (className) {
    runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
  } else {
    fprintf(stderr, "Error: no class name or --zygote supplied.\n");
    app_usage();
    LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    return 10;
  }
}
```

可以发现，是通过 AppRuntime 启动的，而 AppRuntime 继承自 AndroidRuntime，start 方法的实现在 AndroidRuntime 里

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
  
  
  JniInvocation jni_invocation;
  jni_invocation.Init(NULL);
  JNIEnv* env;
  if (startVm(&mJavaVM, &env, zygote) != 0) {
    return;
  }
  onVmCreated(env);

  


  if (startReg(env) < 0) {
    ALOGE("Unable to register all android natives\n");
    return;
  }
  
}
```

注意 startVm 这个方法，我们点进去看看。

```
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
  
  






   if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
      ALOGE("JNI_CreateJavaVM failed\n");
      return -1;
   }
   return 0;
}
```

接下来看 JNI_CreateJavaVM 方法：

```
extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
  ScopedTrace trace(__FUNCTION__);
  const JavaVMInitArgs* args = static_cast<JavaVMInitArgs*>(vm_args);
  if (IsBadJniVersion(args->version)) {
    LOG(ERROR) << "Bad JNI version passed to CreateJavaVM: " << args->version;
    return JNI_EVERSION;
  }
  RuntimeOptions options;
  for (int i = 0; i < args->nOptions; ++i) {
    JavaVMOption* option = &args->options[i];
    options.push_back(std::make_pair(std::string(option->optionString), option->extraInfo));
  }
  bool ignore_unrecognized = args->ignoreUnrecognized;
  if (!Runtime::Create(options, ignore_unrecognized)) {
    return JNI_ERR;
  }

  
  
  android::InitializeNativeLoader();

  Runtime* runtime = Runtime::Current();
  bool started = runtime->Start();
  if (!started) {
    delete Thread::Current()->GetJniEnv();
    delete runtime->GetJavaVM();
    LOG(WARNING) << "CreateJavaVM failed";
    return JNI_ERR;
  }

  *p_env = Thread::Current()->GetJniEnv();
  *p_vm = runtime->GetJavaVM();
  return JNI_OK;
}
```

这个函数不长，我就直接全部贴出来了。注意看 android::InitializeNativeLoader()，这个函数直接调用了 g_namespaces->Initialize()，而 g_namespaces 是一个 LibraryNamespaces 指针，继续看下去，我们发现了宝藏：

```
void Initialize() {
  std::vector<std::string> sonames;
  const char* android_root_env = getenv("ANDROID_ROOT");
  std::string root_dir = android_root_env != nullptr ? android_root_env : "/system";
  std::string public_native_libraries_system_config =
          root_dir + kPublicNativeLibrariesSystemConfigPathFromRoot;

  LOG_ALWAYS_FATAL_IF(!ReadConfig(public_native_libraries_system_config, &sonames),
                      "Error reading public native library list from \"%s\": %s",
                      public_native_libraries_system_config.c_str(), strerror(errno));

  

  
  ReadConfig(kPublicNativeLibrariesVendorConfig, &sonames);

  
  
  
  
  
  
  
  
  for (const auto& soname : sonames) {
    dlopen(soname.c_str(), RTLD_NOW | RTLD_NODELETE);
  }

  public_libraries_ = base::Join(sonames, ':');
}
```

public_native_libraries_system_config=/system/etc/public.libraries.txt，而 ReadConfig 方法很简单，读取传进来的文件路径，按行分割，忽略空行和以 #开头的行，然后把这行 push_back 到传进来的 vector 里。  
所以这个函数做了这几件事：

1.  读取 / system/etc/public.libraries.txt 和 / vendor/etc/public.libraries.txt
2.  **挨个 dlopen 这两个 txt 文件里提到的所有 so 库**

注：这里分析的源码是 7.0.0 的，在 7.0 往后的所有版本（截至本文发布）你都能找到类似的逻辑。

知道了这件事注入 zygote 就好办多了嘛！只要把我们自己写的 so 库扔到 / system/lib 下面（64 位是 / syste/lib64），然后在 / system/etc/public.libraries.txt 里把我们自己的文件加上去，这样 zygote 启动的时候就会去加载我们的 so 库，然后我们写一个函数，加上__attribute__((constructor))，这样这个函数就会在 so 库被加载的时候被调用，我们就完成了注入逻辑；而且这个文件是一个 txt 文件，只需要追加一行文件名就行，即使厂商做了修改也不用担心，稳定性棒棒哒！

（注 1：此方法是我看一篇博客时看见的，那篇博客吐槽 “在 public.libraries.txt 里加上的 so 库竟然会在 zygote 启动时被加载，每次修改都要重启手机才能生效，多不方便调试”，但是他抱怨的特性却成为了我的曙光，可惜找不到那篇博客了，没法贴出来…）  
（注 2：在我大致完成了核心逻辑之后，我在 EdXp 的源码里发现了[这个文件](https://github.com/ElderDrivers/EdXposed/blob/master/edxp-whale/template_override/system/etc/public.libraries-edxp.txt) ；这个部分看起来是使用 whale 进行 java hook 的方案，但是我从来没有听说过有使用纯 whale 进行 java hook 的 EdXp 版本，并且我在 install.sh 中没有看见操作 public.libraries.txt，所以不太懂他想用这个文件干什么 :( ）

ok，现在我们完成了注入 zygote 进程的逻辑，刚完成的时候我想，完成了注入部分，ART Hook 部分也有很多开源库，那么实现一个 xposed 不是很简单的事吗？果然我还是太年轻…

[](#监控应用进程启动 "监控应用进程启动")监控应用进程启动
--------------------------------

前面我们注入了 zygote 进程，然而这样还不够，我们还需要监控应用进程启动并在应用进程执行代码才行。

刚开始我的想法很简单：直接在 zygote 里随便用 art hook 技术 hook 掉几个 java 方法；不过在我们的 so 库被加载的时候 Runtime 还没启动完成，没法拿到 JNIEnv（就算拿到也用不了），这个也好办，native inline hook 掉几个会在 Runtime 初始化完成时调用的函数就行，然并卵，提示无法分配可执行内存。

wtf？？为什么分配内存会失败？内存满了？没对齐？最后终于发现这样一条 log：

`type=1400 audit(0.0:5): avc: denied { execmem } for scontext=u:r:zygote:s0 tcontext=u:r:zygote:s0 tclass=process permissive=0`

上网查了一下，这条 log 代表 zygote 进程的 context（u:r:zygote:s0）不允许分配可执行的匿名内存。这就麻烦了呀，很多事都做不了了（包括 java 方法的 inline hook），想过很多办法（比如替换 sepolicy），最后都被我否决了。那怎么办？最后打算去看 EdXp 的处理方式，没看见任何有关 SELinux 的部分，似乎是让 magisk 处理，不过我的是模拟器，没法装 magisk。

这个问题困扰了我很久，最后，在 Riru 的源码里发现了另一种实现方案：通过 GOT Hook 拦截 jniRegisterNativeMethods，然后就可以替换部分关键 JNI 函数。

简单来说，当发生跨 ELF 的函数调用时，会去. got 表里查这个函数的绝对地址，然后再跳转过去，所以我们直接改这个表就能达到 hook 的目的，更多实现细节可以看 [xhook 的说明文档](https://github.com/iqiyi/xHook/blob/master/docs/overview/android_plt_hook_overview.zh-CN.md)。

这种方式的好处是不需要直接操作内存中的指令，不需要去手动分配可执行内存，所以不会受到 SELinux 的限制；缺点也很明显，如果人家不查. got 表，就无法 hook 了。所以这种方式一般用于 hook 系统函数，比如来自 libc 的 malloc, open 等函数。

好了，GOT Hook 并不是重点，接下来 Riru 使用 xhook hook 了 libandroid_runtime.so 对 jniRegisterNativeMethods 方法（来自 libnativehelper.so）的调用，这样就能拦截一部分的 JNI 方法调用了。为什么说是**一部分**？因为另一部分 JNI 函数的实现在 libart 里，这一部分函数直接通过 env->RegisterNativeMethods 完成注册，所以无法 hook。

之后 riru 在被替换的 jniRegisterNativeMethods 中动了一点小手脚：如果正在注册来自 Zygote 类的 JNI 方法，那么会把 nativeForkSystemServer 和 nativeForkAndSpecialize 替换成自己的实现，这样就能拦截 system_server 与应用进程的启动了！

Riru 的这个方案非常好，但是还有优化空间：nativeForkAndSpecialize 这个函数基本上每个版本都会变签名，而且各个厂商也会做修改，Riru 的办法很简单：比对签名，如果签名不对那么就不会替换。不过，实际上我们并不需要那么精密的监控进程启动，让我们来找一下有没有其他的 hook 点。

大家都知道的，zygote 进程最终会进入 ZygoteInit.main，在 main 方法里 fork 出 system_server，然后进入死循环接收来自 AMS 的请求 fork 出新进程。

```
public static void main(String argv[]) {
    
    if (startSystemServer) {
        startSystemServer(abiList, socketName);
    }
    
    Log.i(TAG, "Accepting command socket connections");
    runSelectLoop(abiList);
    
}
```

在 startSystemServer 里会 fork 出 system_server，runSelectLoop 中会进入死循环等待创建进程请求，然后 fork 出应用进程。

```
private static boolean startSystemServer(String abiList, String socketName)
        throws MethodAndArgsCaller, RuntimeException {
    
    
    pid = Zygote.forkSystemServer(
            parsedArgs.uid, parsedArgs.gid,
            parsedArgs.gids,
            parsedArgs.debugFlags,
            null,
            parsedArgs.permittedCapabilities,
            parsedArgs.effectiveCapabilities);
    
    if (pid == 0) { 
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        handleSystemServerProcess(parsedArgs);
    }
}
```

点进去 handleSystemServerProcess 里看看：

```
private static void handleSystemServerProcess(
         ZygoteConnection.Arguments parsedArgs)
         throws ZygoteInit.MethodAndArgsCaller {
     
     if (parsedArgs.invokeWith != null) {
         
     } else {
         ClassLoader cl = null;
         if (systemServerClasspath != null) {
             cl = createSystemServerClassLoader(systemServerClasspath,
                                                parsedArgs.targetSdkVersion);
             Thread.currentThread().setContextClassLoader(cl);
         }
         


         RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
     }
 }
```

最终会进入 RuntimeInit.zygoteInit（7.x，对于 8.x 与以上在 ZygoteInit.zygoteInit），记住这一点

然后，应用进程的创建是在 runSelectLoop() 里，最后会通过 ZygoteConnection.runOnce 进行处理

```
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
    
    pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                parsedArgs.appDataDir);
    
    if (pid == 0) { 
        
        IoUtils.closeQuietly(serverPipeFd);
        serverPipeFd = null;
        handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

        
        
        return true;
    } else {
        
    }
}
```

```
private void handleChildProc(Arguments parsedArgs,
        FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
        throws ZygoteInit.MethodAndArgsCaller {
    
    if (parsedArgs.invokeWith != null) {
        
    } else {
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                parsedArgs.remainingArgs, null );
    }
}
```

最后也是通过 RuntimeInit.zygoteInit（7.x，对于 8.x 与以上在 ZygoteInit.zygoteInit）完成。点进去看看有没有 hook 点：

```
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    redirectLogStreams();

    commonInit();
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv, classLoader);
}
```

注意到那个 nativeZygoteInit 没有！！很明显是个 native 方法，而且是我们可以 hook 到的 native 方法！！这样子我们就可以直接在 jniRegisterNativeMethods 里替换掉这个方法了！而且这个方法从 7.0 到 10.0 也只有过一次改变：从 RuntimeInit 搬到 ZygoteInit，比 nativeForkAndSpecialize 稳得多。  
（然而现在看来某些操作还是需要比较精细的监控的，以后再改吧）

[](#加载Xposed模块 "加载Xposed模块")加载 Xposed 模块
----------------------------------------

这一部分其实是最简单的，目前已经有很多开源的 ART Hook 库，拿来就能用，需要自己写的地方也不需要跟太久的系统函数调用。  
目前是选择了 [SandHook](https://github.com/ganyao114/SandHook) 作为核心 ART Hook 库，主要是已经提供好了 Xposed API，很方便。  
然后是模块管理，因为没有那么多时间去弄，所以只是简单的把对应的配置文件设置成谁都能读，当然以后会优化。

[](#未来 "未来")未来
--------------

注：本段内容没有什么营养，可以直接跳过。

### [](#编译打包自动化 "编译打包自动化")编译打包自动化

对，目前连自动化打包都没实现，还是手动拿 dex 等等然后手动压缩进去…

### [](#支持magisk安装 "支持magisk安装")支持 magisk 安装

现在只支持通过安装脚本直接修改 / system，如果能够支持 magisk 模块式安装会少很多麻烦，比如如果变砖了只需要用 mm 管理器把模块给删了就好了

### [](#支持重要系统进程加载模块 "支持重要系统进程加载模块")支持重要系统进程加载模块

由于 SELinux 限制，目前不支持关键系统进程（如 Zygote 和 system_server）加载模块，我这边没有什么很好的解决办法，求各位大佬赐教 :)  
顺便列出一些其他的限制：  
android 9，隐藏 API 访问限制，这个好办，绕过方式有很多，就不细讲了。  
Android 9 及以上，zygote&system_server 还有其他系统应用会 SetOnlyUseSystemOatFiles()，然后就不能加载不在 / system 下面的 oat 文件了，如果违反这个策略就会直接 abort 掉。  
android zygote fork 新进程时，如果有不在白名单中的文件描述符，会进到 ZygoteFailure 里，然后整个进程 abort 掉。

### [](#配置文件加载部分 "配置文件加载部分")配置文件加载部分

因为在目标进程里加载模块肯定需要获取对应的配置，目前的做法是，把对应的配置文件设置成谁都能读，然后直接读这个文件就行，这样做当然不妥，所以计划以后去优化，比如优化成单独跑一个配置守护进程，只有这个进程能去读写配置，其他应用只能通过跨进程交互的方式拿到配置。

### [](#重新实现Xposed-API "重新实现Xposed API")重新实现 Xposed API

目前梦境的 Xposed API 是 SandHook 自带的 [xposedcompat](https://github.com/ganyao114/SandHook/tree/master/xposedcompat)，通过 DexMaker 动态创建新的 dex 实现适配，这么做没有什么兼容性问题，但是有很大的效率问题，如果是第一次创建这个方法，需要生成 dex，一套流程走下来可能就要用上 100+ms。  
在 [xposedcompat_new](https://github.com/ganyao114/SandHook/tree/master/xposedcompat_new) 中，有另一种实现方案：通过动态代理动态生成方法，然后把这个方法设置成 native，对应的 native 函数也是通过 libffi 动态生成的，在这个 native 方法里跳到分发函数执行。这个方案对我来说很不错，至少不会太慢。（当然稳定性存疑）

[](#结语 "结语")结语
--------------

目前梦境框架还有非常多的不足，目前只能当个 PoC 用，如果你有兴趣，不妨一起来玩 ^_^  
核心：[Dreamland](https://github.com/canyie/Dreamland)  
配套的管理器 [Dreamland Manager](https://github.com/canyie/DreamlandManager)  
[QQ 群：949888394](mqqwpa://im/chat?chat_type=group&uin=949888394&version=1)