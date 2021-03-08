> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484916&idx=1&sn=e09e4724d41cfaec2bc5d141cb32d661&chksm=ce0754b1f970dda73e112f2de91d062fbd3a768f2c6a9109c20958e822a256df591e87f64f71&scene=178&cur_album_id=1681422395766538242#rd)

**一、前言**

  

    在 **Android App** 开发中，如果涉及到 **jni** 开发，常常会使用 **System.loadLibray** 来加载生成的 **so** 文件。以下将通过安卓 10 的源码，追踪 **System.loadLibrary** 的内部流程。

**二、****System 类调用分析**

     **System** 类源码路径如下:

```
libcore\ojluni\src\main\java\java\lang\System.java

```

  在该类中 **loadLibrary** 函数代码如下:  

```
   public static void loadLibrary(String libname) {
        Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
    }

```

    有以上代码可分析到调用类转到了 **Runtime** 类中。  

**三、**Runtime** 类调用分析**

      **Runtime** 类源码路径:  

```
libcore\ojluni\src\main\java\java\lang\Runtime.java

```

     **loadLibrary0** 函数实现如下:

```
 //
 void loadLibrary0(Class<?> fromClass, String libname) {
        ClassLoader classLoader = ClassLoader.getClassLoader(fromClass);
        loadLibrary0(classLoader, fromClass, libname);
    }
private synchronized void loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname) {
        if (libname.indexOf((int)File.separatorChar) != -1) {
            throw new UnsatisfiedLinkError(
    "Directory separator should not appear in library name: " + libname);
        }
        String libraryName = libname;
        // Android-note: BootClassLoader doesn't implement findLibrary(). http://b/111850480
        // Android's class.getClassLoader() can return BootClassLoader where the RI would
        // have returned null; therefore we treat BootClassLoader the same as null here.
        if (loader != null && !(loader instanceof BootClassLoader)) {
            String filename = loader.findLibrary(libraryName);
            if (filename == null) {
                // It's not necessarily true that the ClassLoader used
                // System.mapLibraryName, but the default setup does, and it's
                // misleading to say we didn't find "libMyLibrary.so" when we
                // actually searched for "liblibMyLibrary.so.so".
                throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                               System.mapLibraryName(libraryName) + "\"");
            }
            String error = nativeLoad(filename, loader);
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }
        // We know some apps use mLibPaths directly, potentially assuming it's not null.
        // Initialize it here to make sure apps see a non-null value.
        getLibPaths();
        String filename = System.mapLibraryName(libraryName);
        String error = nativeLoad(filename, loader, callerClass);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
    }

```

     不具体讨论函数中的详细逻辑流程，**loadLibrary0** 在 **so** 未加载情况下会调用 **nativeLoad** 方法。**nativeLoad** 函数如下:  

```
 private static String nativeLoad(String filename, ClassLoader loader) {
        return nativeLoad(filename, loader, null);
    }
private static native String nativeLoad(String filename, ClassLoader loader, Class<?> caller);

```

     最终调用的是 **native** 方法 **nativeLoad**，所以接下来需要找到该方法的 **jni** 层实现。**Runtime.java** 中的 **jni** 方法是由 **Runtime.c** 注册实现的。接下来分析 **Runtime.c** 中的流程。      

**四、Runtime.c 中的流程分析**

    ** Runtime.c** 源码路径位于:

```
libcore\ojluni\src\main\native\Runtime.c

```

    在该文件中 **java** 层 **native** 方法 **nativeLoad** 对应的 **jni** 层实现函数为 **Runtime_nativeLoad**, 函数实现如下:  

```
JNIEXPORT jstring JNICALL
Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
                   jobject javaLoader, jclass caller)
{
    return JVM_NativeLoad(env, javaFilename, javaLoader, caller);
}

```

     有以上代码逻辑中可分析调用了 **JVM_NativeLoad** 函数。通过强大的 **xxgrep** 搜索命令，搜到该方法被定义在 **jvm.h** 文件中, 实现文件在 **OpenjdkJvm.cc** 中。

 **jvm.h** 文件路径:

```
libcore\ojluni\src\main\native\jvm.h

```

    **OpenjdkJvm**.cc 文件路径:

```
art/openjdkjvm/OpenjdkJvm.cc

```

     接下来分析 **OpenjdkJvm** 的实现。

**五、**OpenjdkJvm**.cc 中的流程分析**

    在 **OpenjdkJvm.cc** 中 **JVM_NativeLoad** 的实现代码如下:

```
JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jclass caller) {
  ScopedUtfChars filename(env, javaFilename);
  if (filename.c_str() == nullptr) {
    return nullptr;
  }
  std::string error_msg;
  {
    art::JavaVMExt* vm = art::Runtime::Current()->GetJavaVM();
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         caller,
                                         &error_msg);
    if (success) {
      return nullptr;
    }
  }
  // Don't let a pending exception from JNI_OnLoad cause a CheckJNI issue with NewStringUTF.
  env->ExceptionClear();
  return env->NewStringUTF(error_msg.c_str());
}

```

      以上代码中核心调用变为了 **JavaVMExt->LoadNativeLibrary**。  

**六、******JavaVMExt** 中的流程分析**

    

      **JavaVMExt**.cc 源码路径位于:

```
art\runtime\jni\java_vm_ext.cc

```

      该文件中 **LoadNativeLibrary** 的函数实现如下:

```
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jclass caller_class,
                                  std::string* error_msg) {
  //..。省略     
  //使用OpenNativeLibrary打开so库                           
  void* handle = android::OpenNativeLibrary(
      env,
      runtime_->GetTargetSdkVersion(),
      path_str,
      class_loader,
      (caller_location.empty() ? nullptr : caller_location.c_str()),
      library_path.get(),
      &needs_native_bridge,
      &nativeloader_error_msg);
  VLOG(jni) << "[Call to dlopen(\"" << path << "\", RTLD_NOW) returned " << handle << "]";
   //...省略
   //查找JNI_OnLoad方法，所以我们在jni开发中动态注册就需要重写这个方法是有道理额
  void* sym = library->FindSymbol("JNI_OnLoad", nullptr);
  if (sym == nullptr) {
    VLOG(jni) << "[No JNI_OnLoad found in \"" << path << "\"]";
    was_successful = true;
  } else {
    // Call JNI_OnLoad.  We have to override the current class
    // loader, which will always be "null" since the stuff at the
    // top of the stack is around Runtime.loadLibrary().  (See
    // the comments in the JNI FindClass function.)
    ScopedLocalRef<jobject> old_class_loader(env, env->NewLocalRef(self->GetClassLoaderOverride()));
    self->SetClassLoaderOverride(class_loader);
    VLOG(jni) << "[Calling JNI_OnLoad in \"" << path << "\"]";
    using JNI_OnLoadFn = int(*)(JavaVM*, void*);
    JNI_OnLoadFn jni_on_load = reinterpret_cast<JNI_OnLoadFn>(sym);
    //主动去调用JNI_OnLoad方法
    int version = (*jni_on_load)(this, nullptr);
    //...省略
  }
   //...省略
  }

```

      以上方法中会打开 **so** 库并调用 **JNI_OnLoad** 方法。以上打开 **so** 使用了 "**android::OpenNativeLibrary**", 下面准备追踪一下该方法的内部实现机制。  

**七、OpenNativeLibrary 实现**

       在源码中搜索，找到 **OpenNativeLibrary** 实现源码路径如下:

```
system\core\libnativeloader\native_loader.cpp

```

       该方法逻辑实现代码如下:

```
void* OpenNativeLibrary(JNIEnv* env, int32_t target_sdk_version, const char* path,
                        jobject class_loader, const char* caller_location, jstring library_path,
                        bool* needs_native_bridge, char** error_msg) {
#if defined(__ANDROID__)
  UNUSED(target_sdk_version);
  if (class_loader == nullptr) {
    *needs_native_bridge = false;
    if (caller_location != nullptr) {
      android_namespace_t* boot_namespace = FindExportedNamespace(caller_location);
      if (boot_namespace != nullptr) {
        const android_dlextinfo dlextinfo = {
            .flags = ANDROID_DLEXT_USE_NAMESPACE,
            .library_namespace = boot_namespace,
        };
        void* handle = android_dlopen_ext(path, RTLD_NOW, &dlextinfo);
        if (handle == nullptr) {
          *error_msg = strdup(dlerror());
        }
        return handle;
      }
    }
    void* handle = dlopen(path, RTLD_NOW);
    if (handle == nullptr) {
      *error_msg = strdup(dlerror());
    }
    return handle;
  }
  std::lock_guard<std::mutex> guard(g_namespaces_mutex);
  NativeLoaderNamespace* ns;
  if ((ns = g_namespaces->FindNamespaceByClassLoader(env, class_loader)) == nullptr) {
    // This is the case where the classloader was not created by ApplicationLoaders
    // In this case we create an isolated not-shared namespace for it.
    std::string create_error_msg;
    if ((ns = g_namespaces->Create(env, target_sdk_version, class_loader, false /* is_shared */,
                                   nullptr, library_path, nullptr, &create_error_msg)) == nullptr) {
      *error_msg = strdup(create_error_msg.c_str());
      return nullptr;
    }
  }
  return OpenNativeLibraryInNamespace(ns, path, needs_native_bridge, error_msg);
#else
 //...省略不相干的
#endif
}

```

     以上代码中会根据 **class_loader** 、**caller_location** 进行判断，是直接使用 **dlopen** 还是 **android_dlopen_ext**、**OpenNativeLibraryInNamespace** 来进行加载 so。

**八、总结**

    **（1）通过分析 System.loadLibrary 之后就可以非常清楚为什么我们动态注册的时候需要重写 JNI_OnLoad 方法。**

  **（2）System.loadLibrary 流程总结如下:**

```
System(Java).loadLibrary->
Runtime(Java).loadLibrary0->
Runtime(Java).nativeLoad->
Runtime_nativeLoad(Runtime.cc)->
JVM_NativeLoad(OpenjdkJvm.cc)->
LoadNativeLibrary(java_vm_exe.cc)->
OpenNativeLibrary(native_loader.cpp)

```

![图片](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432Gds0tic05EGIQOUnAlHicnibrgNo4Tibk8gsK3BicMJBibdbpHdaMovDlkJEdIx8gTcaXAH05rSUv3D3A/640?wx_fmt=png)

**如果你对安卓系统相关的开发学习感兴趣:**

       可加作者的 QQ 群（1017017661), 本群专注安卓系统方面的技术，欢迎加群技术交流。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |** 查看更多文章**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)