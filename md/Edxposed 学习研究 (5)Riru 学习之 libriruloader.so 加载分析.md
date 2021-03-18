> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/JU-Wcm9NLUF7eqmUzWOTCg)

**一、前言**

  

      **Riru** 功能主要是注入到 **zygote** 进程然后通过 **zygote** 创建的子进程都可以运行 **riru** 模块的代码。在 **Riru v23** 以后，采用设置属性 **ro.dalvik.vm.native.bridge** 的值为 **libriruloader.so** 来实现 **zygote** 加载 **riru** 功能。

**二、安卓系统中 ro.dalvik.vm.native.bridge 处理流程**

      在安卓系统中 **ro.dalvik.vm.native.bridge** 属性操作的逻辑位于 **AndroidRuntime.cpp** 源文件中。该文件路径如下:

```
frameworks/base/core/jni/AndroidRuntime.cpp

```

      该文件中启动 **java** 虚拟机的函数中获取了该属性:

```
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
    ...
      // Native bridge library. "0" means that native bridge is disabled.
    //
    // Note: bridging is only enabled for the zygote. Other runs of
    //       app_process may not have the permissions to mount etc.
    //读取ro.dalvik.vm.native.bridge属性
    property_get("ro.dalvik.vm.native.bridge", propBuf, "");
    if (propBuf[0] == '\0') {
        ALOGW("ro.dalvik.vm.native.bridge is not expected to be empty");
    } else if (zygote && strcmp(propBuf, "0") != 0) {
        //这儿表示ro.dalvik.vm.native.bridge属性设置了值，非空
        //设置nativeBrigdgeLibray为-XX:NativeBridge=xxx.so
        //
        snprintf(nativeBridgeLibrary, sizeof("-XX:NativeBridge=") + PROPERTY_VALUE_MAX,
                 "-XX:NativeBridge=%s", propBuf);
        addOption(nativeBridgeLibrary);
    }
    ...
}

```

      该方法中如果设置了属性参数获取加入到启动参数项目。接下来启动虚拟机的时候会用到。在 runtime.cc 中涉及到相关的逻辑处理。源码路径:  

```
art/runtime/runtime.cc

```

      该文件中涉及处理的方法如下:

```
bool Runtime::Init(RuntimeArgumentMap&& runtime_options_in) {
{
    ...
      {
       std::string native_bridge_file_name = runtime_options.ReleaseOrDefault(Opt::NativeBridge);
        is_native_bridge_loaded_ = LoadNativeBridge(native_bridge_file_name);
    }
    ...
}

```

     以上调用了方法 "LoadNativeBridge"。该方法在源文件"native_bridge_art_interface.cc"。该文件路径如下:  

```
art/runtime/native_bridge_art_interface.cc

```

   该文件中处理 **native bridge** 相关函数代码如下:  

```
bool LoadNativeBridge(const std::string& native_bridge_library_filename) {
  VLOG(startup) << "Runtime::Setup native bridge library: "
      << (native_bridge_library_filename.empty() ? "(empty)" : native_bridge_library_filename);
  return android::LoadNativeBridge(native_bridge_library_filename.c_str(),
                                   &native_bridge_art_callbacks_);
}

```

    以上方法调用了 **android::LoadNativeBridge**。该方法实现源文件如下:

```
system/core/libnativebridge/native_bridge.cc

```

    以下是该方法实现代码:

```
bool LoadNativeBridge(const char* nb_library_filename,
                      const NativeBridgeRuntimeCallbacks* runtime_cbs) {
       ...
      // Try to open the library.
      //原来是调用dlopen来加载设置的动态库
      void* handle = dlopen(nb_library_filename, RTLD_LAZY);
      if (handle != nullptr) {
        callbacks = reinterpret_cast<NativeBridgeCallbacks*>(dlsym(handle,
                                                                   kNativeBridgeInterfaceSymbol));
        if (callbacks != nullptr) {
          if (isCompatibleWith(NAMESPACE_VERSION)) {
            // Store the handle for later.
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
      ...
}

```

     通过以上分析，如果 属性 **ro.dalvik.vm.native.bridge** 设置了动态库，那么在 **zygote** 进程中会进行 dlopen 加载。

**三、riruloader 分析**

     在 R**iru** 源码中 **riruloader** 动态库的实现文件在 **cpp** 目录中的 **loader** 目录下。如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432Yb4zwpoMlKNvMWU3c5nFmTVZGB3bpxgYF8QSzNoMtHE9cy7D7xBmZUIC0GkcicdtYCypZVQfdBFQ/640?wx_fmt=png)

     在 **loader.cpp** 中定义了 **constructor** 属性的函数。****constructor**** 说明:

```
constructor属性指定的函数可以使函数在main()函数之前或者so加载的时候首先得到执行

```

     所以当安卓系统加载 **libriruloader** 的时候，首先执行如下的函数。如下所示:  

```
__used __attribute__((constructor)) void constructor() {
   ...
  }

```

   以下是该函数的一个详细的分析说明:  

```
__used __attribute__((constructor)) void constructor() {
    //getuid获取当进程的uid，如果当前不是root用户返回
    if (getuid() != 0)
        return;
    char cmdline[ARG_MAX + 1];
    //获取当前的进程名称,通过读取/proc/self/cmdline
    get_self_cmdline(cmdline, 0);
    //只能zygote相关进程或者usap相关进程才能继续执行
    if (strcmp(cmdline, "zygote") != 0
        && strcmp(cmdline, "zygote32") != 0
        && strcmp(cmdline, "zygote64") != 0
        && strcmp(cmdline, "usap32") != 0
        && strcmp(cmdline, "usap64") != 0) {
        LOGW("not zygote (cmdline=%s)", cmdline);
        return;
    }
    //该函数功能主要是socket连接rirud进程，测试是否连通
    //刷入riru之后，会存在一个rirud进程，该进程主要是解决由于selinux打开
    //存在某些情况下直接读取/data/adb/riru/native_bridge文件失败的情况。
    WaitForSocket(10);
    //此处加载/system/lib64/libriru.so动态库，libriru.so中将会完成模块的扫描和加载
    dlopen(LIB_PATH "libriru.so", 0);
   //以下代码只有在定义DEBUG宏的情况下才执行，主要是为了测试原系统ro.dalvik.vm.native.bridge属性不为空的情况
   //目前使用的手机系统中暂未遇到设置该属性的情况，可以跳过
   //以下代码主要是备份当前系统ro.dalvik.vm.native.bridge原有设置的值
#ifdef HAS_NATIVE_BRIDGE
    char buf[PATH_MAX]{0};
    int32_t buf_size;
    ssize_t size;
    //连接到rirud进程获取备份的native bridge
    ReadOriginalNativeBridgeFromSocket(buf, buf_size);
    //通过rirud读取失败，以下直接打开文件方式获取
    if (buf_size <= 0) {
        LOGW("socket failed, try file");
        //打开/data/adb/riru/native_bridge
        int fd = open(CONFIG_DIR "/native_bridge", O_RDONLY);
        if (fd == -1) {
            PLOGE("access " CONFIG_DIR "/native_bridge");
            return;
        }
        size = read(fd, buf, PATH_MAX);
        close(fd);
        if (size <= 0) {
            LOGE("can't read native_bridge");
            return;
        }
        buf[size] = 0;
        if (size > 1 && buf[size - 1] == '\n') buf[size - 1] = 0;
    } else {
        size = buf_size;
    }
    LOGI("original native bridge: %s", buf);
    //说明读取失败返回
    if (buf[0] == '0' && buf[1] == 0) {
        return;
    }
    //定义保存原有native bridge的存储空间
    auto native_bridge = buf + size + 1;
    //复制/system/lib64
    strcpy(native_bridge, LIB_PATH);
    //复制native bridge的名称组装成完整的路径
    strncat(native_bridge, buf, size);
    //如果原有的不存在返回
    if (access(native_bridge, F_OK) != 0) {
        PLOGE("access %s", native_bridge);
        return;
    }
    //打开原有的native bridge
    original_bridge = dlopen(native_bridge, RTLD_NOW);
    if (original_bridge == nullptr) {
        LOGE("dlopen failed: %s", dlerror());
        return;
    }
    //获取native bridge中定义的接口NativeBridgeItf
    auto original_NativeBridgeItf = dlsym(original_bridge, "NativeBridgeItf");
    if (original_NativeBridgeItf == nullptr) {
        LOGE("dlsym failed: %s", dlerror());
        return;
    }
    int sdk = 0;
    char value[PROP_VALUE_MAX + 1];
    //获取当前安卓sdk版本
    if (__system_property_get("ro.build.version.sdk", value) > 0)
        sdk = atoi(value);
    auto callbacks_size = 0;
    if (sdk >= 30) {
        callbacks_size = sizeof(android30::NativeBridgeCallbacks);
    } else if (sdk == 29) {
        callbacks_size = sizeof(android29::NativeBridgeCallbacks);
    } else if (sdk == 28) {
        callbacks_size = sizeof(android28::NativeBridgeCallbacks);
    } else if (sdk == 27) {
        callbacks_size = sizeof(android27::NativeBridgeCallbacks);
    } else if (sdk == 26) {
        callbacks_size = sizeof(android26::NativeBridgeCallbacks);
    } else if (sdk == 25) {
        callbacks_size = sizeof(android25::NativeBridgeCallbacks);
    } else if (sdk == 24) {
        callbacks_size = sizeof(android24::NativeBridgeCallbacks);
    } else if (sdk == 23) {
        callbacks_size = sizeof(android23::NativeBridgeCallbacks);
    }
    //备份原有native bridge保存到全局变量NativeBridgeItf中
    memcpy(NativeBridgeItf, original_NativeBridgeItf, callbacks_size);
#endif
}

```

**四、总结**

  **  1. 安卓系统中会检测 **ro.dalvik.vm.native.bridge** 属性是否设置，如果设置了会在 zygote 进程中使用 dlopen 进行 so 加载。**

    **2.**ro.dalvik.vm.native.bridge** 设置为 libriruloader.so 提供了 riru 执行初始化的入口。**  

**如果你对安卓相关的开发学习感兴趣:**

       可加作者的 QQ 群（1017017661), 本群专注安卓方面的技术，欢迎加群技术交流。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |** 查看更多文章**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)