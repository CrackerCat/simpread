> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/138x_kF5keABM50URvoQtA)

**1. 前言**

 在安卓 **jni** 编程中，常常接触到 **JavaVM** 和 **JNIEnv**。比如 **jni** 动态注册中的 JNI_OnLoad 方法是我们需要实现的。使用的时候需要通过 **JavaVM** 获取 **JNIEnv**。参考如下:

```
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env = NULL;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return -1;
    }
    LOGD("JNI Loaded");
    return JNI_VERSION_1_6;
}

```

     除了动态注册的时候，在编写 **java** 层 **native** 方法在 **c** 层的实现时候，会使用到 **JNIEnv** 中提供的很多 **jni** 接口函数，比如 **FindClass**、**CallObjectMethod** 等。

     以下将通过源码中追踪 **JavaVM**&**JNIEnv** 创建和初始化流程分析理解 **JNI** 层的一个大概实现机制。

**2.**JavaVM** 和 **JNIEnv** 创建流程分析  
**

  安卓中 **zygote** 是所有 **Java** 层进程的鼻祖。**zygote** 启动的时候会创建 **art** 虚拟机。**JavaVM** 和 **JNIEnv** 初始化的流程也会在 **zygote** 启动的时候完成。

**2.1  zygote 启动流程**

  **(1).init 进程解析 init.rc**

       安卓系统用户空间第一个进程 **init** 进程启动的时候会解析 **init.rc** 文件来确认使用那个一个 **init.zygotexx.rc** 文件来启动 zygote 进程。init.rc 中相关内容如下:

```
...
import /init.${ro.zygote}.rc
...

```

     由以上内容可知需要根据 **ro.zygote** 属性来确认加载具体的 **zygote** 启动配置的 **rc** 文件来启动 **32** 位还是 **64** 位的 **zygote** 进程。在文件路径 "**system\core\rootdir**" 存在多个 zygote 启动的配置文件，如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/3sPGriaR212FYBBHsVFKAuRIF9YPOiaZVKhqvrRicKIHkkkvvzH0510W3s1qay8u67IiaRDUEsQ4AIYxqdic8xBjFLQ/640?wx_fmt=png)

 **ro.zygote** 属性会有四种不同的取值：

*   **zygote32**：代表 32 位模式运行 zygote 进程  
    
*   **zygote32_64**：代表 32 为主，64 位为辅的方式启动 zygote 进程
    
*   **zygote64**：代表 64 位方式启动 zygote 进程
    
*   **zygote64_32**：代表 64 位为主，32 位为辅的方式启动 zygote 进程
    

以下是一加 3 手机在安卓 10 系统中的 ro.zygote 属性的值。如下所示:

```
C:\Users\Qiang>adb shell getprop |findstr "ro.zygote"
[ro.zygote]: [zygote64_32]

```

 根据以上 **ro.zygote** 值为 zygote64_32，当前手机将会加载 **init.zygote64_32.rc** 文件。**init.zygote64_32.rc** 内容如下:

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    ...
service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
    class main
    priority -20
    user root
    ...

```

      该配置文件中配置启动了两个 **zygote** 进程，一个是执行 **app_process64**, 一个是执行 **app_process32**。接下来分析 app_process 进程的相应逻辑代码。

  **(2).app_process 启动过程**

     在安卓系统源码中，app_process64 和 app_process32 使用的是同一份源代码。源码路径位于目录:

```
frameworks\base\cmds\app_process

```

     安卓系统在编译的时候根据 Android.mk 文件配置编译成了两个平台并重命名。如下是相应的 Android.mk 文件中配置情况。  

```
LOCAL_PATH:= $(call my-dir)
...
include $(CLEAR_VARS)
...
# 指定编译模块名称app_process
LOCAL_MODULE:= app_process
# 指定编译手机平台both表示32和64都要都要编译
LOCAL_MULTILIB := both
# 指定32位情况下编译名称变为app_process32
LOCAL_MODULE_STEM_32 := app_process32
# 指定64位情况下编译名称变为app_process64
LOCAL_MODULE_STEM_64 := app_process64
...
include $(BUILD_EXECUTABLE)
...

```

      **app_process** 模块中只有一个源文件 **app_main.cpp**。**app_main.cpp** 文件中和 **zygote** 启动相关的逻辑代码如下:  

```
//AppRuntime继承AndroidRuntime类
class AppRuntime : public AndroidRuntime
{
   ...
}
//zygote进程入口函数
int main(int argc, char* const argv[])
{
    ...
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    ...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } 
    ...
}

```

   以上代码中关键逻辑变成了 **AndroidRuntime->start**, 接下来分析一下 **AndroidRuntime** 中的相关逻辑。

 **2.2  AndroidRuntime 中的处理流程分析**

 AndroidRuntime 类文件路径如下:

```
frameworks\base\core\jni\AndroidRuntime.cpp

```

 在该类中的 **start** 方法如下

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ...
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
    ...
}

```

      以上方法中调用了 **startVm** 方法创建 **JavaVM 和 JNIEnv**。startVm 中的关键逻辑如下:  

```
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
   ...
   if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
   ...
}

```

     以上方法中执行 **JNI_CreateJavaVM** 方法来创建虚拟机。接下来分析 **JNI_CreateJavaVM** 中的处理逻辑。

**2.3 **JNI_CreateJavaVM** 中的处理流程分析**

     **JNI_CreateJavaVM** 方法定义在 "jni.h" 文件中。具体实现在文件 "**libnativehelper\JniInvocation.cpp**" 中。该文件中 **JNI_CreateJavaVM** 实现如下:

```
MODULE_API jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
   ...
  return JniInvocationImpl::GetJniInvocation().JNI_CreateJavaVM(p_vm, p_env, vm_args);
}

```

以上方法中使用了 **JniInvocationImpl** 类的 **JNI_CreateJavaVM** 方法。该类位于路径 "**libnativehelper\JniInvocation.cpp**" 中，他的 **JNI_CreateJavaVM** 方法实现如下:

```
jint JniInvocationImpl::JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
  return JNI_CreateJavaVM_(p_vm, p_env, vm_args);
}

```

      在以上实现中调用了 **JNI_CreateJavaVM_**方法, 该方法是一个函数指针，定义为如下:

```
jint (*JNI_CreateJavaVM_)(JavaVM**, JNIEnv**, void*);

```

    在 **JniInvocationImpl** 类中函数指针 **JNI_CreateJavaVM_**赋值相关逻辑实现如下:

```
//该方法主要是根据打开的libart.so来查找函数符号
bool JniInvocationImpl::FindSymbol(FUNC_POINTER* pointer, const char* symbol) {
  *pointer = GetSymbol(handle_, symbol);
  ...
  return true;
}
//
bool JniInvocationImpl::Init(const char* library) {
  ...
  //获取需要加载的so，此处为libart.so
  library = GetLibrary(library, buffer);
  //打开libart.so
  handle_ = OpenLibrary(library);
  ...
  //此处表示获取libart.so中的JNI_CreateJavaVM函数地址并赋值给JNI_CreateJavaVM_
  if (!FindSymbol(reinterpret_cast<FUNC_POINTER*>(&JNI_CreateJavaVM_),
                  "JNI_CreateJavaVM")) {
    return false;
  }
  ...
}

```

    以上分析可知最终调用的是 **libart.so** 中的 JNI_CreateJavaVM 方法来实现创建虚拟机。接下来分析 **art** 中的 JNI_CreateJavaVM 方法逻辑。

**2.4 art 中虚拟机创建分析**  

   在 **art** 源码中，**JNI_CreateJavaVM** 方法实现源文件路径为:  

```
art\runtime\jni\java_vm_ext.cc

```

   在该文件中 JNI_CreateJavaVM 的实现代码逻辑如下:

```
extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
  ...
  //使用了Runtime::create创建虚拟机
  if (!Runtime::Create(options, ignore_unrecognized)) {
    return JNI_ERR;
  }
  ...
  Runtime* runtime = Runtime::Current();
  //启动虚拟机
  bool started = runtime->Start();
  ...
  //初始化JNIEnv
  *p_env = Thread::Current()->GetJniEnv();
  //初始化JavaVM
  *p_vm = runtime->GetJavaVM();
  return JNI_OK;
}

```

     以上代码分析核心流程变成了 **Runtime::Create**。接下来分析 **art** 中的 **Runtime** 类中的处理逻辑。  

**2.5 Runtime 中的处理逻辑**  

     Runtime 类位于文件路径:  

```
art\runtime\runtime.cc

```

     在该类中 **Create** 调用的处理逻辑如下:

```
//调用1
bool Runtime::Create(const RuntimeOptions& raw_options, bool ignore_unrecognized) {
  RuntimeArgumentMap runtime_options;
  return ParseOptions(raw_options, ignore_unrecognized, &runtime_options) &&
      Create(std::move(runtime_options));
}
//调用2
bool Runtime::Create(RuntimeArgumentMap&& runtime_options) {
  ...
  instance_ = new Runtime;
  ...
  //调用了Init方法初始化
  if (!instance_->Init(std::move(runtime_options))) {
    ...
  }
  return true;
}

```

    以上调用中流程走到 **Init** 方法中，**Init** 方法实现如下:  

```
bool Runtime::Init(RuntimeArgumentMap&& runtime_options_in) {
  ...
  std::string error_msg;
  //这个地方真正创建JavaVM
  java_vm_ = JavaVMExt::Create(this, runtime_options, &error_msg);
  if (java_vm_.get() == nullptr) {
    LOG(ERROR) << "Could not initialize JavaVMExt: " << error_msg;
    return false;
  }
  ...
  // 这个地方线程附加
  Thread* self = Thread::Attach("main", false, nullptr, false);
  ...
  }

```

   以上逻中完成 JavaVM 的初始化，接下来继续看一下 **JNIEnv** 的初始化流程。  

**2.6  Thread 类中的分析**  

   **Thread** 类源码路径:  

```
art\runtime\runtime.cc

```

    在该类中 **Attach** 方法逻辑如下:  

```
//调用1
Thread* Thread::Attach(const char* thread_name,
                       bool as_daemon,
                       jobject thread_group,
                       bool create_peer) {
      ...
      //调用另一个Attach
 return Attach(thread_name, as_daemon, create_peer_action);
 }
 //调用2
 template <typename PeerAction>
Thread* Thread::Attach(const char* thread_name, bool as_daemon, PeerAction peer_action) {
  Runtime* runtime = Runtime::Current();
  ...
  Thread* self;
  {
    ...
    if (runtime->IsShuttingDownLocked()) {
     ...
    } else {
      Runtime::Current()->StartThreadBirth();
      self = new Thread(as_daemon);
      //初始化JNIEnv相关逻辑
      bool init_success = self->Init(runtime->GetThreadList(), runtime->GetJavaVM());
      ...
    }
  }
 ...
}

```

    以上代码中核心进入 **Thread->Init** 方法，**Init** 方法实现如下:  

```
bool Thread::Init(ThreadList* thread_list, JavaVMExt* java_vm, JNIEnvExt* jni_env_ext) {
  ...
  if (jni_env_ext != nullptr) {
   ...
  } else {
    std::string error_msg;
    //创建JNIEnv
    tlsPtr_.jni_env = JNIEnvExt::Create(this, java_vm, &error_msg);
    if (tlsPtr_.jni_env == nullptr) {
      LOG(ERROR) << "Failed to create JNIEnvExt: " << error_msg;
      return false;
    }
  }
  ...
}

```

**3. 总结**  

 **JavaVM 创建关键:**

```
JavaVMExt::Create(this, runtime_options, &error_msg);

```

 **JNIEnv 创建关键:**

```
JNIEnvExt::Create(this, java_vm, &error_msg);

```

    本篇主要分析 **JavaVM&JNIEnv** 在 zygote 启动过程中的创建和初始化**。**后面会通过 **JNIEnv->FindClass** 的接口调用分析 **JNI** 相关接口的内部实现机制。  

**如果你对安卓相关的开发学习感兴趣:**

 **可加作者的 QQ 群（1017017661), 本群专注安卓方面的技术，欢迎加群技术交流。**

![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif) 点击屏末 ****| ****阅****读****原****文**** |** 获取更多文章列表信息**

扫描下方二维码关注公众号

![](https://mmbiz.qpic.cn/mmbiz_png/3sPGriaR212HJoY7wDaAklzxvoFerMCk6YKFSaEcpYTf6zSPHibqzibGr7hk91Vh8iaXY4wc5z7zZYfeu1hlyvNNGQ/640?wx_fmt=png)