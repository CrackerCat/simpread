> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg5MzU3NzkxOQ==&mid=2247484452&idx=1&sn=c7a2082632ea2c6532c7c119a2168db9&chksm=c02dfbf4f75a72e2a18a0fb8d8c23c5278487ff22bd4052bf38b7ac3f5401ebce08478a3de8c&cur_album_id=1799542483832324102&scene=190#rd)

**1.JNIEnv 初始化**  

 在文章 **"** **[安卓 10 源码学习开发定制 (36)Art 虚拟机中 JavaVM &JNIEnv 的初始化流程](http://mp.weixin.qq.com/s?__biz=Mzg5MzU3NzkxOQ==&mid=2247484351&idx=1&sn=86291a3dd19a619900aae18d856b7c60&chksm=c02dfc6ff75a757966df5de543ce67e084a0b6c2444334f33af9a603b00a09806056b5a7c474&scene=21#wechat_redirect) "** 中已经分析了 **JNIEnv** 的初始化构造流程。最终 **JNIEnv** 初始化构造是在文件 **art\runtime\thread.cc** 中的类中进行构造，关键代码如下:

```
bool Thread::Init(ThreadList* thread_list, JavaVMExt* java_vm, JNIEnvExt* jni_env_ext) {
  ...
  if (jni_env_ext != nullptr) {
    ...
    tlsPtr_.jni_env = jni_env_ext;
  } else {
    std::string error_msg;
    //创建JNIEnv结构并存储到线程局部存储变量中
    tlsPtr_.jni_env = JNIEnvExt::Create(this, java_vm, &error_msg);
    ...
  }
  ...
}

```

     接下来分析 **JNIEnvExt** 中的 **JNINativeInterface** 是如何初始化的。

**2.JNINativeInterface 初始化流程分析**

 JNIEnvExt 类的文件路径为:

```
art\runtime\jni\jni_env_ext.cc

```

   **(1).JNIEnvExt 中的调用分析**

          在以上分析中 **JNIEnv** 通过调用 **JNIEnvExt::Create** 进行创建，在 **JNIEnvExt** 中的 **Create** 方法实现逻辑如下:

```
JNIEnvExt* JNIEnvExt::Create(Thread* self_in, JavaVMExt* vm_in, std::string* error_msg) {
  std::unique_ptr<JNIEnvExt> ret(new JNIEnvExt(self_in, vm_in, error_msg));
  ...
}

```

      在以上代码中使用了 **new JNIEnvExt(self_in, vm_in, error_msg)** 进行初始化，下面是 **JNIEnvExt** 构造函数实现:

```
JNIEnvExt::JNIEnvExt(Thread* self_in, JavaVMExt* vm_in, std::string* error_msg)
    : self_(self_in),
      vm_(vm_in),
      local_ref_cookie_(kIRTFirstSegment),
      locals_(kLocalsInitial, kLocal, IndirectReferenceTable::ResizableCapacity::kYes, error_msg),
      monitors_("monitors", kMonitorsInitial, kMonitorsMax),
      critical_(0),
      check_jni_(false),
      runtime_deleted_(false) {
  ...
  //这个地方进行了JNINativeInterface的初始化，functions 定义在JNIEnv中
  functions = GetFunctionTable(check_jni_);
}

```

     在以上代码调用中调用了 **GetFunctionTable**() 方法初始化 **JNIEnv** 中的 **functions** 变量，**JNIEnv** 中的 **functions** 定义如下:

```
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;
    ...
}

```

     接下来分析一下 **GetFunctionTable** 方法中的实现。**GetFunctionTable** 逻辑代码如下:

```
const JNINativeInterface* JNIEnvExt::GetFunctionTable(bool check_jni) {
  ...
  //当前会根据chck_jni调用不同的JNINativeInterface初始化接口
  return check_jni ? GetCheckJniNativeInterface() : GetJniNativeInterface();
}

```

 **check_jni** 在编译 **eng** 版本的时候会被打开, 所以这个地方我们只分析 **user**、**userdebug** 版本的流程。所以接下来将会执行 **GetJniNativeInterface** 方法。

    根据源码定位 **GetJniNativeInterface** 方法定义在如下文件:

```
art\runtime\jni\jni_internal.h

```

   定义的代码内容如下:  

```
const JNINativeInterface* GetJniNativeInterface();

```

    该方法实现文件路径位于:

```
art\runtime\jni\jni_internal.cc

```

   接下来分析 **jni_internal.cc** 中的流程。  

 **(2).**jni_internal.cc** 中的调用分析**

       在该文件中 GetJniNativeInterface 方法实现如下:

```
const JNINativeInterface* GetJniNativeInterface() {
  return &gJniNativeInterface;
}

```

     有以上代码分析可知返回的是 **gJniNativeInterface** 变量的值，该变量定义如下:

```
const JNINativeInterface gJniNativeInterface = {
  nullptr,  // reserved0.
  nullptr,  // reserved1.
  nullptr,  // reserved2.
  nullptr,  // reserved3.
  JNI::GetVersion,
  JNI::DefineClass,
  JNI::FindClass,
  ...
}

```

         有以上代码可以知道该变量为常量，使用一些列 **JNI** 类中的方法填充 **JNINativeInterface** 接口，从而完成 **JNIEnv** 中的 **JNINativeInterface** 初始化。

    在 **jni_internal.cc** 中可以找到类 **JNI** 的定义实现。如下:

```
class JNI {
 public:
  static jint GetVersion(JNIEnv*) {
    return JNI_VERSION_1_6;
  }
  static jclass DefineClass(JNIEnv*, const char*, jobject, const jbyte*, jsize) {
    LOG(WARNING) << "JNI DefineClass is not supported";
    return nullptr;
  }
  static jclass FindClass(JNIEnv* env, const char* name) {
    CHECK_NON_NULL_ARGUMENT(name);
    Runtime* runtime = Runtime::Current();
    ClassLinker* class_linker = runtime->GetClassLinker();
    std::string descriptor(NormalizeJniClassDescriptor(name));
    ScopedObjectAccess soa(env);
    ObjPtr<mirror::Class> c = nullptr;
    if (runtime->IsStarted()) {
      StackHandleScope<1> hs(soa.Self());
      Handle<mirror::ClassLoader> class_loader(hs.NewHandle(GetClassLoader(soa)));
      c = class_linker->FindClass(soa.Self(), descriptor.c_str(), class_loader);
    } else {
      c = class_linker->FindSystemClass(soa.Self(), descriptor.c_str());
    }
    return soa.AddLocalReference<jclass>(c);
  }
  ...
 }

```

      以上代码可以看到 **JNI** 中真正实现了 **JNIEnv** 结构体中 JNINativeInterface 接口，我们平时调用的接口比如 **FindClass** 最终调用的是 **JNI::FindClass**。

**3. 总结**

 art 虚拟机中的 **JNIEnv** 中的各种接口最终实现位于 **"jni_internal.cc"** 中的 **class  JNI** 中实现。

    如果我们需要在系统中进行 JNI trace，就可以在 **jni_internal.cc** 中进行日志输出。

如果你对安卓相关的开发学习感兴趣:

   **可加作者的 QQ 群（1017017661), 本群专注安卓方面的技术，欢迎加群技术交流。**

![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif) 点击屏末 ****| ****阅****读****原****文**** |** 获取更多文章列表信息**

扫描下方二维码关注公众号

![](https://mmbiz.qpic.cn/mmbiz_png/3sPGriaR212HJoY7wDaAklzxvoFerMCk6YKFSaEcpYTf6zSPHibqzibGr7hk91Vh8iaXY4wc5z7zZYfeu1hlyvNNGQ/640?wx_fmt=png)