> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281030.htm)

> Android Root 环境下动态注入 Java 和 Native 代码的实践

Android Root 环境下动态注入 Java 和 Native 代码的实践

3 天前 1572

### Android Root 环境下动态注入 Java 和 Native 代码的实践

 [![](http://passport.kanxue.com/upload/avatar/838/849838.png?1553410502)](user-home-849838.htm) [WindStormy](user-home-849838.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 3  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 3 天前  1572

博客原文：[Android Root 环境下动态注入 Java 和 Native 代码的实践](https://windysha.github.io/2024/03/22/%E4%B8%80%E7%A7%8DAndroid-Root%E7%8E%AF%E5%A2%83%E4%B8%8B%E5%8A%A8%E6%80%81%E6%B3%A8%E5%85%A5Java%E5%92%8CNative%E4%BB%A3%E7%A0%81%E7%9A%84%E6%96%B9%E6%A1%88/)

背景
==

在 Android 逆向开发中，我们通常会使用 Frida 工具在命令行中动态注入 JavaScript 代码到目标应用，编写 JavaScript 对 Android 新手来说可能会有些困难，假如能用 Java 代码 Hook Java 层方法，c/c++ 代码 Hook native 层函数指令，用起来可能会更顺手。  
   
在 Android 正向开发中，我们往往需要在 Release 包上进行性能诊断或复杂问题的分析，然而，这并不是一件容易的事情。原因在于，Release 包通常不方便调试，大型 App 的编译过程需要消耗大量时间，修改框架或第三方 SDK 代码也相对困难。因此，有时候我们就需要利用逆向工具，比如 Xposed 或者 Frida，通过代码注入的方式来进行问题调试。  
   
那么，如何实现在命令行中将 Xposed 插件动态地注入到目标应用中呢？  
本文将探讨在 Android Root 设备上将 Xposed 插件动态注入到目标应用的一种实践。

思路分析
====

实现过程主要包括以下几部分：

*   注入过程的实现；
*   ART Hook 框架的封装；
*   插件 APK 的加载，包括插件中动态库的加载；
*   native 进入 Java 的入口实现；

具体的思路如下：

1.  注入的过程可以参考 Frida，使用 ptrace 实现；
2.  ART Hook 库使用有点过时但还能用的 SandHook，和加载插件功能一起打包成 dex 和 so；
3.  在 ptrace 注入的 native 动态库中实现 dex 的加载并调用其入口方法；

实现过程
====

以上介绍了工具的实现思路，下面详细介绍其实现过程。  
整体的流程图如下：  
![](https://bbs.kanxue.com/upload/attach/202403/849838_ZDHZ3SNQJXQ6B8Y.jpg)  
整体功能分为三层：

*   ** 注入层：** 在命令行中，运行可执行文件，使用 ptrace 将指定 so 文件注入到目标进程；
*   ** 胶水层：** 在目标进程启动时，完成 dex 和 so 文件合并到 App 主 ClassLoader 中，并使用合并后的 ClassLoader 调用 dex 文件中的入口方法；
*   ** 插件加载层：** 完成 ART HOOK 库的初始化以及外部 Xposed Apk 插件的加载；

注入层
---

注入层的目的是将一个二进制可执行文件注入到目标进程中并运行其中的函数；  
   
linux 平台上，使用 ptrace 进行进程注入的技术方案已经非常成熟，刚好笔者之前专门研究过，并开源了一个相对稳定可靠的 android 平台的 ptrace 注入库：  
[XInjector](https://github.com/WindySha/XInjector)  
   
这里，直接使用这个仓库编译出来的动态可执行文件，即可轻松实现将动态库注入到指定应用中。  
 

胶水层
---

胶水层主要是完成 dex 和 so 文件合并到 App 主 ClassLoader 中，并根据合并后的 ClassLoader 调用 dex 文件中的入口方法。  
这一层最大的难点是：如何在 native 世界撬开 Java 世界的大门。  
   
核心流程包含：

> 1.  获取当前线程的 JNIEnv 指针；
> 2.  寻找插件加载层初始化时机；
> 3.  执行插件加载层的入口方法；

   
这一部分三个步骤来讲解。

### 初始化时机

执行插件加载层初始化的时机选择是本方案面临的主要挑战之一，原因如下：

*   插件加载层的初始化应尽早执行，这样插件 apk 能 Hook 住更多的方法；
*   由于 ptrace 注入 so 的时机具有不确定性，过早注入可能会出现应用的包信息未被解析，LoadedApk 对象未创建，因此无法构造 Context 对象，无法进入 Java 世界；
*   若在 ptrace 后的流程中，直接执行插件加载层的初始化，可能会引发未知的崩溃问题。比如，若 ptrace 到一个被 CriticalNative 注解的 jni 方法中，在这个方法的流程中使用 JNIEnv 指针可能导致意外崩溃。

   
对此，Frida 库在开发过程中可能也会遇到类似的问题。它的解决方案是在注入代码时创建一个子线程来运行 hook 代码，从而成功地绕过了第三个问题。然而，由于代码是在子线程中执行的，这使得执行时机具有一定的不确定性。这可能会导致注入时机稍微延后，结果会导致一些方法被 hook 前可能已经被执行。  
   
基于以上分析，运行插件加载层的初始化逻辑选择以下三个时机：

1.  如果 ptrace 注入到无法使用 JNIEnv 指针的 jni 方法中，使用 ALooper 将初始化逻辑发送到主线程 Handler 中执行，其缺点是，执行 hook 逻辑时机较晚，此时 Application 的 attachBaseContext 和 onCreate 已经执行完成，但可以作为一种兜底方案；
    
2.  如果 ptrace 注入时机非常早，此时 ActivityThread 的 mLoadedApk 成员还未创建出来，无法进入 Java 世界，因此，需要延迟初始化。这里选择使用 plt hook libcutils 中`atrace_update_tags`函数，在此函数中执行初始化逻辑，选择这个函数是因为它在启动时只执行一次，并且此时应用的 mLoadedApk 已经被创建；
    
3.  非以上两种情况的话，直接在 Ptrace 注入的流程中执行初始化逻辑，完成 dex 和 so 合并到 App 的主 ClassLoader，并调用 dex 中的入口方法；  
       
    伪代码如下：
    

```
static void OnAtraceFuncCalled() {
       void* current_thread_ptr = runtime::CurrentThreadFunc();
       JNIEnv* env = runtime::GetJNIEnvFromThread(current_thread_ptr);
 
       if (!env) {
           LOGE("Failed to get JNIEnv, Inject Xposed Module failed.");
           return;
       }
       InjectXposedLibraryInternal(env);
   }
    
   int DoInjection(JNIEnv* env) {
       runtime::InitRuntime();
 
       void* current_thread_ptr = nullptr;
       if (!env) {
           current_thread_ptr = runtime::CurrentThreadFunc();
           env = runtime::GetJNIEnvFromThread(current_thread_ptr);
       }
       if (!env) {
           LOGE("Failed to get JNIEnv !!");
           return -1;
       }
 
       // ptrace到JNI方法Java_java_lang_Object_wait时，会出现由于等锁导致的env->FindClass卡死的问题，这里Handler中加载xposed模块
       // ptrace到JNIT方法Java_com_android_internal_os_ClassLoaderFactory_createClassloaderNamespace时，正在构造classLoader此时调用FindClass会卡死
       // 这里绕过这两个方法，发送任务到主线程Handler执行；
       void* art_method = runtime::GetCurrentMethod(current_thread_ptr, false, false);
       if (art_method != nullptr) {
           std::string name = runtime::JniShortName(art_method);
           if (strcmp(name.c_str(), "Java_java_lang_Object_wait") == 0
               || strcmp(name.c_str(), "Java_com_android_internal_os_ClassLoaderFactory_createClassloaderNamespace") == 0) {
               // load xposed modules after in the main message handler, this is later than application's attachBaseContext and onCreate method.
               InjectXposedLibraryByHandler(env);
               return 0;
           }
       }
 
       // If the inject time is very early, then, the loadedapk info and the app classloader is not ready, so we try to hook atrace_set_debuggable function to make sure
       // the injection is early enough and the classloader has also been created.
       jobject loaded_apk_obj = jni::GetLoadedApkObj(env);
       LOGD("Try to get the app loaded apk info, loadedapk jobject: %p", loaded_apk_obj);
 
       if (loaded_apk_obj == nullptr) {
           // load xposed modules after atrace_set_debuggable or atrace_update_tags is called.
           HookAtraceFunctions(OnAtraceFuncCalled);
       }
       else {
           // loadedapk and classloader is ready, so load the xposed modules directly.
           InjectXposedLibraryInternal(env);
       }
       return 0;
   }

```

虽然在主线程中执行代码注入能确保其发生的时机足够早，但在实际应用中，我们发现这个方案会导致注入后的应用启动崩溃或者卡死的概率非常高。这种现象可能是由于虚拟机的限制所导致，虚拟机的某些 JNI 函数内部无法运行 Java 代码。  
   
因此，我们最终还是默认选择使用 Frida 的解决方案，在子线程中初始化和执行插件的入口方法。  
这应该是一种更为稳定和可靠的方案。  
代码如下：

```
static void InjectXposedLibraryAsync(JNIEnv* jni_env) {
      JavaVM* javaVm;
      jni_env->GetJavaVM(&javaVm);
      std::thread worker([javaVm]() {
          JNIEnv* env;
          javaVm->AttachCurrentThread(&env, nullptr);
 
          int count = 0;
          while (count < 10000) {
              jobject app_loaded_apk_obj = jni::GetLoadedApkObj(env);
              // wait here until loaded apk object is available
              if (app_loaded_apk_obj == nullptr) {
                  usleep(100);
              } else {
                  break;
              }
              count++;
          }
          InjectXposedLibraryInternal(env);
          if (env) {
              javaVm->DetachCurrentThread();
          }
      });
      worker.detach();
  }

```

### 合并 Classloader

ptrace 成功后，就可以执行我们的 native 代码，但为了能加载外置插件，我们需要使用 Java 代码来实现。为了能进入 Java 世界，需要在 native 层构造相关 Classloader 然后调 Java 层入口方法。  
   
一般来说，使用 Classloader 加载 Java 代码有两种策略：

1.  依据目录 dex 文件路径，单独构造 dex 文件对应的 Classloader，使用此 Classloader 加载入口类；
2.  将目标 dex 文件合并在 App 的主 Classloader 中，使用 App 的 Classloader 加载入口类；  
       
    考虑到实现的难易程度，这里选择了第二种策略：合并 Classloader 策略。  
    主要的实现思路是对 App 的 PathClassLoader 所对应的 pathList 成员变量进行修改。我们利用目标 dex 文件的路径，构造一个新的 DexElement 实例，并将这个实例加入到 pathList 的 dexElements 数组中。同时，我们将目标 so 文件的路径融入到 pathList 的 nativeLibraryDirectories 数组中。  
       
    具体实现起来，难点有：
3.  需要纯 native 代码实现；
4.  不同 Android 版本 PathClassLoader 中的成员变量和数据结构差异较大，需要针对各版本做好兼容；  
       
    解决办法是，先用 java 代码实现好完整的功能，并兼容好不同的 Android 版本，然后再 “翻译” 为 c++ 代码的实现。

### 调用入口方法

将 dex 和 so 合并到 App 主 ClassLoader 之后，使用当前线程的 JNIEnv 指针反射调用 dex 文件中的入口类和入口方法时，会抛出类找不到的异常，原因暂不可知。  
   
为了规避这个问题，这里使用一种 “破釜沉舟” 的办法：  
直接使用 App 的 ClassLoader 调用 loadClass 方法加载目标类，然后再通过 getDeclaredMethod 获取目标方法，最后再调用 Method.invoke() 执行目标方法。  
   
伪代码如下：

```
void CallStaticMethodByJavaMethodInvoke(JNIEnv* env, const char* class_name, const char* method_name) {
       ScopedLocalRef app_class_loader(env, jni::GetAppClassLoader(env));
 
       const char* classloader_class_name = "java/lang/ClassLoader";
       const char* class_class_name = "java/lang/Class";
       const char* method_class_name = "java/lang/reflect/Method";
 
       // Class clazz = appClassLoader.loadClass("class_name");
       ScopedLocalRef classloader_jclass(env, env->FindClass(classloader_class_name));
       auto loadClass_mid = env->GetMethodID(classloader_jclass.get(), "loadClass", "(Ljava/lang/String;)Ljava/lang/Class;");
       ScopedLocalRef class_name_jstr(env, env->NewStringUTF(class_name));
       ScopedLocalRef clazz_obj(env, env->CallObjectMethod(app_class_loader.get(), loadClass_mid, class_name_jstr.get()));
 
       // get Java Method mid: Class.getDeclaredMethod()
       ScopedLocalRef class_jclass(env, env->FindClass(class_class_name));
       auto getDeclaredMethod_mid = env->GetMethodID(class_jclass.get(), "getDeclaredMethod",
           "(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;");
 
       // Get the Method object
       ScopedLocalRef method_name_jstr(env, env->NewStringUTF(method_name));
       jvalue args[] = { {.l = method_name_jstr.get()},{.l = nullptr} };
       ScopedLocalRef method_obj(env, env->CallObjectMethodA(clazz_obj.get(), getDeclaredMethod_mid, args));
 
       ScopedLocalRef method_jclass(env, env->FindClass(method_class_name));
       // get Method.invoke jmethodId
       auto invoke_mid = env->GetMethodID(method_jclass.get(), "invoke",
           "(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;");
 
       //Call Method.invoke()
       jvalue args2[] = { {.l = nullptr},{.l = nullptr} };
       env->CallObjectMethodA(method_obj.get(), invoke_mid, args2);
   } 
```

插件加载层
-----

插件加载层的最终产物是 Android App 工程编译出来的 dex 文件和 so 文件。  
   
这个 Android 工程主要包含基于 SandHook 的 Xposed Api 库，包含一个入口方法，用于初始化 SandHook 以及加载 Xposed 插件 Apk。  
   
核心逻辑有：

*   ART Hook 的初始化：构造 App Context 对象，并用 context 对象调用 SandHook 的初始化方法；
*   预处理插件 Apk：根据插件 apk 文件路径，将 Apk 中的 so 文件解压到 App 私有目录下；
*   加载插件 Apk：根据插件 Apk 文件路径，so 文件路径，构造插件 ClassLoader，加载插件入口类，并调用入口方法；  
       
    下面简单介绍这三个步骤的实现要点。

### ART Hook 的初始化

由于 SandHook 库的初始化时需要传入 context 对象，但由于插件加载层被注入的时机很早，此时的应用的 Application 和 App Context 对象很可能并未创建出来，因此，需要自行创建一个应用的 context 对象。  
   
ContextImpl 类中有一个方法 createAppContext 可以用于创建 context 对象，参数包含两个，一个是当前进程的 ActivityThread 对象，另一个是 LoadedApk 对象。前者是个单例，通过反射很容易得到，后者是保存在 ActivityThread 的 mBoundApplication 成员变量中，也可以反射获取。  
   
完整的构造 context 代码如下：

```
public static Context createAppContext() {
//        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, mLoadedApk);
        try {
            Class activityThreadClass = Class.forName("android.app.ActivityThread");
            Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
            currentActivityThreadMethod.setAccessible(true);
 
            Object activityThreadObj = currentActivityThreadMethod.invoke(null);
 
            Field boundApplicationField = activityThreadClass.getDeclaredField("mBoundApplication");
            boundApplicationField.setAccessible(true);
            Object mBoundApplication = boundApplicationField.get(activityThreadObj);   // AppBindData
 
            Field infoField = mBoundApplication.getClass().getDeclaredField("info");   // info
            infoField.setAccessible(true);
            Object loadedApkObj = infoField.get(mBoundApplication);  // LoadedApk
 
            Class contextImplClass = Class.forName("android.app.ContextImpl");
            Method createAppContextMethod = contextImplClass.getDeclaredMethod("createAppContext", activityThreadClass, loadedApkObj.getClass());
            createAppContextMethod.setAccessible(true);
 
            Object context = createAppContextMethod.invoke(null, activityThreadObj, loadedApkObj);
 
            if (context instanceof Context) {
                return (Context) context;
            }
        } catch (ClassNotFoundException | NoSuchMethodException | IllegalAccessException | InvocationTargetException | NoSuchFieldException e) {
            e.printStackTrace();
        }
        return null;
    }

```

### 预处理插件 Apk

   
为了实现在 xposed 插件中自动加载 native 代码，我们需对插件内的动态 so 库进行特别处理。  
具体的处理方式是：先将插件 APK 中的 so 文件提取到指定的目录下，然后在构造插件的`DexClassLoader`时，将 so 文件的目录传入。这样，插件 Apk 在调用`System.loadLibrary()`时就可以成功加载插件的 so 库了。  
   
在整个流程中，关键环节在于提取插件 Apk 中的 so 文件。  
   
最常见的提取方式是首先解压 apk 压缩包，然后复制解压后的 so 文件到指定的目录下。  
   
如果你对 Android 系统源码进行过详细了解，会发现源码中已经集成了提取 Apk 文件中 so 的相关功能。这个功能是由`NativeLibraryHelper.java`这个工具类提供的。  
该工具主要用途是，在安装 Apk 过程中，如果 Apk 的 manifest 文件中配置了`android:extractNativeLibs = true`，系统会自动将 apk 文件中的 so 文件提取到指定目录下。然后在 app 启动时，构造 App 的 ClassLoader 并将该目录作为参数传入，从而实现 native 库的自动加载。  
 

提取 so 文件的源码如下：  
`com/android/internal/content/NativeLibraryHelper.java`

```
/**
 * Copies native binaries to a shared library directory.
 *
 * @param handle APK file to scan for native libraries
 * @param sharedLibraryDir directory for libraries to be copied to
 * @return {@link PackageManager#INSTALL_SUCCEEDED} if successful or another
 *         error code from that class if not
 */
public static int copyNativeBinaries(Handle handle, File sharedLibraryDir, String abi) {
    for (long apkHandle : handle.apkHandles) {
        int res = nativeCopyNativeBinaries(apkHandle, sharedLibraryDir.getPath(), abi,
                handle.extractNativeLibs, handle.debuggable);
        if (res != INSTALL_SUCCEEDED) {
            return res;
        }
    }
    return INSTALL_SUCCEEDED;
}

```

这里，只需要先反射构造一个 Handle 对象，然后再反射调用 copyNativeBinaries 这个方法即可实现将 so 文件提取到指定目录下。  
 

### 加载插件 Apk

加载插件 Apk 的逻辑比较简单，参考 Xposed 原生代码的实现即可。  
主要分为两个步骤：

1.  构造 DexClassLoader，读取 Apk 的 assets/xposed_init 目录下配置的创建入口类的类全名；
2.  再次构造 DexClassLoader，根据入口类的全名加载入口类，构造类的实例，并调用入口方法。  
       
    伪代码如下：

```
// use system classloader to load asset to avoid the app has file assets/xposed_init
ClassLoader assetLoader = new DexClassLoader(moduleApkPath, moduleOdexDir, moduleLibPath, ClassLoader.getSystemClassLoader());
InputStream is = assetLoader.getResourceAsStream("assets/xposed_init");
if (is == null) {
    Log.i(TAG, "assets/xposed_init not found in the APK");
    return false;
}
 
ClassLoader mcl = new DexClassLoader(moduleApkPath, moduleOdexDir, moduleLibPath, appClassLoader);
BufferedReader moduleClassesReader = new BufferedReader(new InputStreamReader(is));
 
String moduleClassName;
while ((moduleClassName = moduleClassesReader.readLine()) != null) {
        moduleClassName = moduleClassName.trim();
        if (moduleClassName.isEmpty() || moduleClassName.startsWith("#"))
            continue;
 
        Class moduleClass = mcl.loadClass(moduleClassName);
 
        final Object moduleInstance = moduleClass.newInstance();
 
        if (moduleInstance instanceof IXposedHookLoadPackage) {
            IXposedHookLoadPackage.Wrapper wrapper = new IXposedHookLoadPackage.Wrapper((IXposedHookLoadPackage) moduleInstance);
            XposedBridge.CopyOnWriteSortedSet xc_loadPackageCopyOnWriteSortedSet = new XposedBridge.CopyOnWriteSortedSet<>();
            xc_loadPackageCopyOnWriteSortedSet.add(wrapper);
            XC_LoadPackage.LoadPackageParam lpparam = new XC_LoadPackage.LoadPackageParam(xc_loadPackageCopyOnWriteSortedSet);
            lpparam.packageName = currentApplicationInfo.packageName;
            lpparam.processName = getCurrentProcessName(currentApplicationInfo);;
            lpparam.classLoader = appClassLoader;
            lpparam.appInfo = currentApplicationInfo;
            lpparam.isFirstApplication = true;
            XC_LoadPackage.callAll(lpparam);
        }
} 
```

至此，完成插件 Apk 的加载。

产物目录
----

下面是最终产物文件目录结构，以及每个文件的用途：

```
├── README.md          // 说明文档
├── glue              // 此文件夹中是胶水层的注入到目标进程的so文件，用于插件加载层的dex和so的注入和加载
│   ├── arm64-v8a
│   │   └── libinjector-glue.so
│   └── armeabi-v7a
│       └── libinjector-glue.so
├── injector          // 此文件夹中是注入层的可执行文件
│   ├── arm64-v8a
│   │   └── xinjector
│   └── armeabi-v7a
│       └── xinjector
├── plugin_loader     // 此文件夹中是插件加载层的编译出的dex和so文件，实现ART Hook和Xposed插件Apk的加载
│   ├── classes.dex
│   ├── libsandhook-64.so
│   └── libsandhook.so
└── start.sh          // 启动脚本(暂时只支持macOS)，用于复制所有的文件到Android Root设备的data/local/tmp目录下，并运行xinjector可执行文件

```

使用方法
====

基本用法
----

1.  下载 [zip 压缩包](https://github.com/WindySha/Poros/releases/download/v1.1/v1.1.zip)并解压；
    
2.  打开命令行，cd 到解压后的目录下；
    
3.  执行以下命令，即可实现重启目标 app，并将 xposed 插件注入到目标 app 进程中：
    

```
$ ./start.sh -p com.android.settings  -f xposed_module_sample.apk

```

启动的 App 是: com.android.settings  
注入的 xposed 插件 apk 路径是：xposed_module_sample.apk  
 

其他用法
----

性能模式：使用参数 `-q`  
默认情况下，会拷贝 glue 和 plugin_loader 目录下的 dex 和 so 文件到目标 app 的 data/data / 目录下，使用 - q 后便不拷贝这些文件，对同一个 app 进行第二次注入时使用此参数，可以提升注入性能。

```
$ ./start.sh -p com.android.settings -f xposed_module_sample.apk -q

```

   
另外，如果对同一 app 第二次注入相同的 xposed 插件时，也可以省`-f xposed_module_sample.apk` 参数，此时会使用上次注入过的插件。  
   
使用 - h 参数: 可在控制台输出帮助信息：

```
$ ./start.sh -h

```

控制台输出结果：

```
[ Java And C/C++ Code Injection for Android Root Device]
 
Usage: ./start.sh -p package_name -f xxx.apk  -h -q
 
Options:
  -p [package name]      the package name to be injected
  -f [apk path]          the xposed module apk path
  -h                     display this help message
  -q                     use quick mode, do not inject the xposed dex and so file, this can only used when it is not the first time to inject target app.

```

已知问题
====

由于本工具实现比较仓促，目前还存在一些短期内无法修复的问题，主要有：

1.  偶现 ptrace 注入失败的情况，会导致 app 启动崩溃或者卡死，目前正在尝试修复中，可能会更换非 ptrace 的注入方案，比如此方案：[Code injection on Android without ptrace](https://erfur.github.io/blog/dev/code-injection-without-ptrace)，这个方案也可以规避反 ptrace 的应用；
2.  ART Hook 框架目前使用的是 SandHook，在某些机型中可能存在稳定性问题，未来可能会更换成持续维护的 Lsposed 库；
3.  暂时只支持注入代码到应用的主进程，由于应用的子进程启动时机不确定，暂时无法统一支持；
4.  在 android13 及以上设备上，由于 SankHook 的缺陷，hook 未解析类的静态方法可能会失败，一种规避的方法是调用下面的 resolveStaticMethod() 方法来提前解析这个静态方法所在的类，然后再执行 hook：

```
public static void resolveStaticMethod(Member method) {
    try {
        if (method instanceof Method && Modifier.isStatic(method.getModifiers())) {
            ((Method) method).setAccessible(true);
            ((Method) method).invoke(new Object(), getFakeArgs((Method) method));
        }
    } catch (Throwable throwable) {
    }
}
 
private static Object[] getFakeArgs(Method method) {
    Class[] pars = method.getParameterTypes();
    if (pars == null || pars.length == 0) {
        return new Object[]{new Object()};
    } else {
        return null;
    }
}

```

源码和规划
=====

源码
--

[https://github.com/WindySha/Poros](https://github.com/WindySha/Poros)  
有能力的可提交 PR，欢迎共建。

未来规划
----

*   优化 Java Hook 库的稳定性和兼容性或者更换为其他更稳定库；
*   注入流程支持非 ptrace 方案；
*   支持在 windows，linux 平台上编译和执行注入；
*   支持子进程注入；
*   支持多插件注入；

参考
==

*   [Frida](https://github.com/frida/frida)
*   [XInjector](https://github.com/WindySha/XInjector)
*   [linjector-rs](https://github.com/erfur/linjector-rs)

写在结尾
====

写这个工具的动机是，笔者的工作是做某大型 App 的底层优化，经常要改 ART 虚拟机或者其他系统动态库的一些功能，由于大型 App 编译安装太耗时，通过这种注入的方式进行测试和开发确实能提升效率，因此就顺手撸了这个工具。  
   
其实很久之前就写好了这个工具，最开始只是自己用，没打算开源出来，后来想想，如其让代码烂在电脑里，不如开源出来，说不定能给其他人带来一些帮助。  
   
世界因开源而更加美好。

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#基础理论](forum-161-1-117.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm) [#工具脚本](forum-161-1-128.htm)