> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.canyie.top](https://blog.canyie.top/2020/08/18/nbinjection/)

之前研究 art 的时候发现了 native bridge，简单来说这东西是主要作用就是为了能运行不同指令集的 so（比如 x86 的设备运行 arm 的 app），而 arm 设备上这个东西一般都是关闭的，研究了一下后发现这东西挺适合动手脚的，刚好自己在用的 [Riru](https://github.com/RikkaApps/Riru) 被针对了，所以有了这篇博客。把对应的示例代码传到了 github：[NbInjection](https://github.com/canyie/NbInjection)，接下来我们聊一下这个小玩具。

[](#源码分析 "源码分析")源码分析
--------------------

大家都知道的，zygote 对应的可执行文件就是 app_process，它的 main 函数代码如下（已精简）：

```
int main(int argc, char* const argv[])
{
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    
    
    argc--;
    argv++;

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

AppRuntime 继承自 AndroidRuntime，而 AndroidRuntime 的代码大概是这样的：

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);

    


    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    
}
```

这个函数做的最重要一件事就是把虚拟机启动起来（startVm），然后调用传入类的 main 方法。  
追踪这个 startVm 方法你会发现调用到了 Runtime::Init 初始化 runtime，这个函数很长，截取了一段对我们来说最重要的：

```
bool Runtime::Init(RuntimeArgumentMap&& runtime_options_in) {
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  {
    std::string native_bridge_file_name = runtime_options.ReleaseOrDefault(Opt::NativeBridge);
    is_native_bridge_loaded_ = LoadNativeBridge(native_bridge_file_name);
  }
  
}
```

在 Runtime::Init 里会加载 native bridge，LoadNativeBridge() 函数是这样实现的：

```
bool LoadNativeBridge(const char* nb_library_filename,
                      const NativeBridgeRuntimeCallbacks* runtime_cbs) {
  
  

  if (nb_library_filename == nullptr || *nb_library_filename == 0) {
    CloseNativeBridge(false);
    return false;
  } else {
    if (!NativeBridgeNameAcceptable(nb_library_filename)) {
      CloseNativeBridge(true);
    } else {
      
      void* handle = dlopen(nb_library_filename, RTLD_LAZY);
      if (handle != nullptr) {
        callbacks = reinterpret_cast<NativeBridgeCallbacks*>(dlsym(handle,
                                                                   kNativeBridgeInterfaceSymbol));
        if (callbacks != nullptr) {
          if (isCompatibleWith(NAMESPACE_VERSION)) {
            
            native_bridge_handle = handle;
          } else {
            callbacks = nullptr;
            dlclose(handle);
            ALOGW("Unsupported native bridge interface.");
          }
        } else {
          dlclose(handle);
        }
      }

      
      
      if (callbacks == nullptr) {
        CloseNativeBridge(true);
      } else {
        runtime_callbacks = runtime_cbs;
        state = NativeBridgeState::kOpened;
      }
    }
    return state == NativeBridgeState::kOpened;
  }
}
```

发现了什么没有！！是我们熟悉的 dlopen！！dlopen 会执行目标库的. init_array 中的所有函数，而让自己的函数进入. init_array 实际上只需要声明`__attribute__((constructor))`就好了，完全没有难度啊！  
hey，先冷静一下，我们还有一个问题不知道答案：这个 native bridge 是从哪传进来的？答案很简单，回过头看一下 AndroidRuntime::startVm() 就明白了：

```
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote, bool primary_zygote)
{
    JavaVMInitArgs initArgs;
    

    
    
    
    
    property_get("ro.dalvik.vm.native.bridge", propBuf, "");
    if (propBuf[0] == '\0') {
        ALOGW("ro.dalvik.vm.native.bridge is not expected to be empty");
    } else if (zygote && strcmp(propBuf, "0") != 0) {
        snprintf(nativeBridgeLibrary, sizeof("-XX:NativeBridge=") + PROPERTY_VALUE_MAX,
                 "-XX:NativeBridge=%s", propBuf);
        addOption(nativeBridgeLibrary);
    }
    
    initArgs.version = JNI_VERSION_1_4;
    initArgs.options = mOptions.editArray();
    initArgs.nOptions = mOptions.size();
    initArgs.ignoreUnrecognized = JNI_FALSE;

    






    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }

    return 0;
}
```

原来是读取的`ro.dalvik.vm.native.bridge`这个系统属性啊，等等，这个属性名字是以`.ro`开头的，也就代表着这个属性是只读的，一旦设置不能修改…… 另一个问题是，这个属性定义在 default.prop 中，而非常规的 build.prop，这个文件改不了，每次开机都会重新读取，那还玩啥啊，拜拜……  
等等！谁说这条属性就只能由厂商修改了？

[](#利用 "利用")利用
--------------

我拿来测试的设备是一台 Google Pixel 3（Android 10，Magisk 20.4），因为有 magisk 所以直接写成了 magisk 模块；没有 magisk 的话可以考虑修改 ramdisk.img（此方法同样适用于模拟器），将 default.prop 中的 ro.dalvik.vm.native.bridge 修改为我们的 so 文件名就好了（注意文件必须在系统的 lib 下面）  
这里就当你把环境配置好了吧，让我们继续：  
写一个函数，往里面写入代码，加上`__attribute__((constructor))`，编译，放 / system/lib64 和 / system/lib 下面，修改`ro.dalvik.vm.native.bridge`为我们的文件名，重启，成功，完结撒花……

当然不可能这么容易，此时虽然你已经把代码成功注入到了 zygote 进程，但是还有一些问题要处理，让我们来细数一下。

### [](#系统原有的native-bridge被覆盖 "系统原有的native bridge被覆盖")系统原有的 native bridge 被覆盖

native bridge 这东西对 arm 设备上来说基本没啥用，然而对 x86 设备来说，没有这玩意你就没法用只支持 arm 的 app，也就是说你连微信都用不了……  
要解决这个问题，还是得看源码，看看系统是怎么调用的 native bridge 里的函数：

```
void* NativeBridgeGetTrampoline(void* handle, const char* name, const char* shorty,
                                uint32_t len) {
  if (NativeBridgeInitialized()) {
    return callbacks->getTrampoline(handle, name, shorty, len);
  }
  return nullptr;
}
```

是用的一个叫 callbacks 的全局变量啊，看下这个 callbacks 是啥：

```
struct NativeBridgeCallbacks {
  
  uint32_t version;

  bool (*initialize)(const struct NativeBridgeRuntimeCallbacks* runtime_cbs,
                     const char* private_dir, const char* instruction_set);

  void* (*loadLibrary)(const char* libpath, int flag);

  void* (*getTrampoline)(void* handle, const char* name, const char* shorty, uint32_t len);
  
}



static const NativeBridgeCallbacks* callbacks = nullptr;
```

原来是一个指向 NativeBridgeCallbacks 的指针，这个叫做 NativeBridgeCallbacks 的结构体里包含函数指针，运行时会找到对应的函数指针然后调用。  
这个变量是在哪初始化的呢：

```
static constexpr const char* kNativeBridgeInterfaceSymbol = "NativeBridgeItf";

bool LoadNativeBridge(const char* nb_library_filename,
                      const NativeBridgeRuntimeCallbacks* runtime_cbs) {
      
      void* handle = dlopen(nb_library_filename, RTLD_LAZY);
      if (handle != nullptr) {
        callbacks = reinterpret_cast<NativeBridgeCallbacks*>(dlsym(handle,
                                                                   kNativeBridgeInterfaceSymbol));
        if (callbacks != nullptr) {
          if (isCompatibleWith(NAMESPACE_VERSION)) {
            
            native_bridge_handle = handle;
          } else {
            callbacks = nullptr;
            dlclose(handle);
            ALOGW("Unsupported native bridge interface.");
          }
        } else {
          dlclose(handle);
        }
      }
    return state == NativeBridgeState::kOpened;
  }
}
```

是从 native bridge 的 so 库中找到的，对应符号是`NativeBridgeItf`。  
既然系统是这样做的，那我们就顺着系统来，在合适的时候偷梁换柱一下。  
首先声明一个对应类型的变量 NativeBridgeItf：

```
__attribute__ ((visibility ("default"))) NativeBridgeCallbacks NativeBridgeItf;
```

注：如果你使用 c++，记得加上`extern "C"`。  
然后，在系统 dlopen 我们的库时，会执行`.init_array`里的函数，我们可以在这里动手脚：

```
if (real_nb_filename[0] == '\0') {
    LOGW("ro.dalvik.vm.native.bridge is not expected to be empty");
} else if (strcmp(real_nb_filename, "0") != 0) {
    LOGI("The system has real native bridge support, libname %s", real_nb_filename);
    const char* error_msg;
    void* handle = dlopen(real_nb_filename, RTLD_LAZY);
    if (handle) {
        void* real_nb_itf = dlsym(handle, "NativeBridgeItf");
        if (real_nb_itf) {
            
            memcpy(&NativeBridgeItf, real_nb_itf, sizeof(NativeBridgeCallbacks));
            return;
        }
        errro_msg = dlerror();
        dlclose(handle);
    } else {
        errro_msg = dlerror();
    }
    LOGE("Could not setup NativeBridgeItf for real lib %s: %s", real_nb_filename, error_msg);
}
```

简单解释一下：系统是通过读取我们的`NativeBridgeItf`这个变量来获取要执行的对应函数的，那我们就可以仿照系统，从真正的 native bridge 中读取这个变量，覆盖掉我们暴露出去的那个`NativeBridgeItf`，这样就会走真实的 native bridge callbacks。  
注：这里还有个坑，NativeBridgeCallbacks 这个结构体的大小在其他系统版本是不同的，如果只复制固定大小，要么复制不全要么越界；所以这里需要按照版本判断一下。

### [](#无法驻留在内存中 "无法驻留在内存中")无法驻留在内存中

当你兴致勃勃地写好了代码，运行时你会发现各种奇怪的 bug，排查 N 遍后你才发现，你写好的这个 so 在内存中不知道什么时候消失了？？  
让我们看看系统的那个 LoadNativeBridge：

```
void* handle = dlopen(nb_library_filename, RTLD_LAZY);
if (handle != nullptr) {
    callbacks = reinterpret_cast<NativeBridgeCallbacks*>(dlsym(handle, kNativeBridgeInterfaceSymbol));
    if (callbacks != nullptr) {
      if (isCompatibleWith(NAMESPACE_VERSION)) {
        
        native_bridge_handle = handle;
      } else {
        callbacks = nullptr;
        dlclose(handle);
        ALOGW("Unsupported native bridge interface.");
      }
    } else {
      dlclose(handle);
    }
}
```

如果`isCompatibleWith`这个函数返回 false，那么就会 close 掉我们的 so 库。

```
static bool isCompatibleWith(const uint32_t version) {
  
  
  if (callbacks == nullptr || callbacks->version == 0 || version == 0) {
    return false;
  }

  
  if (callbacks->version >= SIGNAL_VERSION) {
    return callbacks->isCompatibleWith(version);
  }

  return true;
}
```

是通过`callbacks->version`和`callbacks->isCompatibleWith`这个函数指针判断的。  
那我们需要在系统没有 native bridge 时设置一下这些东西。（如果系统有 native bridge 那么在上面`NativeBridgeItf`就已经被覆盖了）  
你需要把 callbacks 里面的东西都设置一下，以免发生其他问题；还好还好，那些函数只需要写个空实现就行，需要注意的是版本，比如 5.0 就只接受 v1 版本的 native bridge，而 7.0 时只接受 v3 及以上版本。

把这些设置好了以后，你的 so 库能成功驻留在 zygote 进程的内存中了；然而，你在应用进程中找不到这个 so 库，这是因为新进程 fork 出来以后，如果不需要 native bridge，系统会卸载它：

```
static void ZygoteHooks_nativePostForkChild(JNIEnv* env,
                                            jclass,
                                            jlong token,
                                            jint runtime_flags,
                                            jboolean is_system_server,
                                            jboolean is_zygote,
                                            jstring instruction_set) {
  
  if (instruction_set != nullptr && !is_system_server) {
    ScopedUtfChars isa_string(env, instruction_set);
    InstructionSet isa = GetInstructionSetFromString(isa_string.c_str());
    Runtime::NativeBridgeAction action = Runtime::NativeBridgeAction::kUnload;
    if (isa != InstructionSet::kNone && isa != kRuntimeISA) {
      action = Runtime::NativeBridgeAction::kInitialize;
    }
    runtime->InitNonZygoteOrPostFork(env, is_system_server, is_zygote, action, isa_string.c_str());
  } else {
    runtime->InitNonZygoteOrPostFork(
        env,
        is_system_server,
        is_zygote,
        Runtime::NativeBridgeAction::kUnload,
         nullptr,
        profile_system_server);
  }
}

void Runtime::InitNonZygoteOrPostFork(
    JNIEnv* env,
    bool is_system_server,
    
    
    
    bool is_child_zygote,
    NativeBridgeAction action,
    const char* isa,
    bool profile_system_server) {
  if (is_native_bridge_loaded_) {
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
  
}
```

这个过程我们很难干预，然而其实我们可以换个思路：既然系统要卸载这个 so 库，那我们就让它卸载；我们已经可以在 zygote 里执行任意代码了，那么写个新 so 库把主要逻辑放里面，在这个假的 native bridge 里 dlopen() 这个新库，假的 native bridge 直接当个 loader 不就好了嘛！而且这样的话实际上我们不用实现那堆函数，只需要把 version 设置成一个无效的值（比如 0），这样系统检测到版本无效就会自动关闭我们的假 native bridge 库，也不用担心那些回调函数会被调用~

[](#总结 "总结")总结
--------------

利用 native bridge 可以实现比较简单的 zygote 注入，实际用起来需要费点功夫，不过都是体力活，比如每个版本中`NativeBridgeCallbacks`这个结构体的大小之类的；以后可能会把这东西应用在我的 Dreamland 上。  
文末再放一下示例代码链接：[NbInjection](https://github.com/canyie/NbInjection)  
[QQ 群：949888394](https://shang.qq.com/wpa/qunwpa?idkey=25549719b948d2aaeb9e579955e39d71768111844b370fcb824d43b9b20e1c04)，欢迎一起来玩~  
文章可能有疏漏，也可能有更好的办法；欢迎交流讨论~

  

> 博客内容遵循 署名 - 非商业性使用 - 相同方式共享 4.0 国际 (CC BY-NC-SA 4.0) 协议
> 
> 本文永久链接是：[https://blog.canyie.top/2020/08/18/nbinjection/](https://blog.canyie.top/2020/08/18/nbinjection/)