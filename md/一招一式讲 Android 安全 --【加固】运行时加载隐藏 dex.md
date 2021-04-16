> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267039.htm)

一招一式讲 Android 安全 --【加固】运行时加载隐藏 dex

 

目录

*            1. 概述
*                    1.1. 背景
*                    1.2. 目的
*                    1.3. 作者
*            2. ClassLoader 机制介绍
*                    2.1. ClassLoader 类关系
*                    2.2. ClassLoader 部分源码
*            3. 初始化 ClassLoader
*                    3.1. 调用链路
*                    3.2. 详细源码
*                            3.2.1. ActivityThread.performLaunchActivity
*                            3.2.2. ActivityThread.createBaseContextForActivity
*                            3.2.3. ContextImpl.createActivityContext
*                            3.2.4. LoadedApk.getClassLoader
*                            3.2.5. LoadedApk.createOrUpdateClassLoaderLocked
*                            3.2.6. ApplicationLoaders.getClassLoader
*                            3.2.7. ClassLoaderFactory.createClassLoader
*            4. findClass 源码路径
*                    4.1. BaseDexClassLoader
*                    4.2. DexPathList
*                    4.3. Element
*                    4.4. DexFile
*                    4.5. dalvik_system_DexFile
*            5. 替换 mCookie
*                    5.1. 关键点
*                    5.2. 思路
*                    5.3. 实现
*                    5.4. 思考
*            6. 替换 ClassLoader
*                    6.1. 覆盖 ClassLoader 成员变量
*                    6.2. 替换 LoadedApk 里 ClassLoader
*            7. 添加到 DexPathList
*            8. 直接加载 dex byte
*                    8.1. 从内存中加载 dex，避免生成 oat
*                            8.1.1. 调用链路
*                            8.1.2. 详细源码

1. 概述
-----

### 1.1. 背景

本系列会把 Android 加固一系列的保护原理和思路讲解一遍，仅作为归档整理。  
包括 dex、so、反调试等。

### 1.2. 目的

本文主要以 Android 9.0 代码为蓝本进行研究。  
希望把加载隐藏 dex 的思路和原理讲明白，详细的细节，各个步奏，等等。  
如有错误的地方，请联系我改正，谢谢。

### 1.3. 作者

本文所有权益归 chinamima@163.com 所有，如需转载请发邮件。  
昵称：chinamima  
邮箱：chinamima@163.com  
经验：从事 Android 行业 10 年，Android 安全 7 年，网络安全 2 年

2. ClassLoader 机制介绍
-------------------

### 2.1. ClassLoader 类关系

![](https://bbs.pediy.com/upload/attach/202104/633423_QGA2RVFTXBRNXSN.jpg)  
![](https://bbs.pediy.com/upload/attach/202104/633423_U7BAGFAU93N352H.jpg)

### 2.2. ClassLoader 部分源码

```
// DexClassLoader.java
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
 
// PathClassLoader.java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
 
    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}

```

*   `DexClassLoader`和`PathClassLoader`都可以加载外部 dex。
*   两个类的区别在于`DexClassLoader`可以指定`optimizedDirectory`，但在 android 8.0 以后`optimizedDirectory`已经废弃掉了。
*   `optimizedDirectory`是`dex2oat`生成的`.oat`、`.odex`文件存放位置。
*   `optimizedDirectory`为空，则会在默认位置生成`.oat`、`.odex`文件，因此源码里用`PathClassLoader`作为`ClassLoader`的实例化类。
    
    ```
    //android 8.0
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        this(dexPath, optimizedDirectory, librarySearchPath, parent, false);
    }
     
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent, boolean isTrusted) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
     
        if (reporter != null) {
            reportClassLoaderChain();
        }
    }
    
    ```
    
    `BaseDexClassLoader`的构造函数并没有把`optimizedDirectory`传递给`DexPathList`。
    

3. 初始化 ClassLoader
------------------

### 3.1. 调用链路

```
ActivityThread.performLaunchActivity //启动activity
 ├─ ActivityThread.createBaseContextForActivity //创建ContextImpl，并作为app的Context实现
 │    └─ ContextImpl.createActivityContext //从LoadedApk获取ClassLoader，并创建ContextImpl
 │        └─ LoadedApk.getClassLoader //判断是否已有ClassLoader，如无则调用createOrUpdateClassLoaderLocked
 │            └─ LoadedApk.createOrUpdateClassLoaderLocked
 │                └─ ApplicationLoaders.getClassLoader
 │                    └─ ClassLoaderFactory.createClassLoader
 │                        └─ new PathClassLoader
 └─ ContextImpl.getClassLoader //LoadedApk.getClassLoader已经生成了ClassLoader，并存于ContextImpl的mClassLoader中

```

### 3.2. 详细源码

#### 3.2.1. ActivityThread.performLaunchActivity

```
package android.app;
 
public final class ActivityThread extends ClientTransactionHandler {
    /**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //...省略，chinamima@163.com
        //创建ContextImpl
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            //从ContextImpl里获取ClassLoader
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
             //...省略，chinamima@163.com
        } catch (Exception e) {
            //...省略，chinamima@163.com
        }
 
        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            //...省略，chinamima@163.com
            if (activity != null) {
                //...省略，chinamima@163.com
                appContext.setOuterContext(activity);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
                //...省略，chinamima@163.com
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                //...省略，chinamima@163.com
                r.activity = activity;
            }
            mActivities.put(r.token, r);
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            //...省略，chinamima@163.com
        }
 
        return activity;
    }
}

```

#### 3.2.2. ActivityThread.createBaseContextForActivity

```
package android.app;
 
public final class ActivityThread extends ClientTransactionHandler {
 
    private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
        int displayId = ActivityManager.getService().getActivityDisplayId(r.token);
        //...省略，chinamima@163.com
        ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
        //...省略，chinamima@163.com
        return appContext;
    }
}

```

#### 3.2.3. ContextImpl.createActivityContext

```
package android.app;
 
class ContextImpl extends Context {
    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
        //...省略，chinamima@163.com
        //从LoadedApk里获取ClassLoader
        ClassLoader classLoader = packageInfo.getClassLoader();
        //...省略，chinamima@163.com
        //创建一个ContextImpl实例
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,
                activityToken, null, 0, classLoader);
        //...省略，chinamima@163.com
        return context;
    }
}

```

#### 3.2.4. LoadedApk.getClassLoader

```
package android.app;
public final class LoadedApk {
 
    private ClassLoader mClassLoader;
 
    public ClassLoader getClassLoader() {
        synchronized (this) {
            if (mClassLoader == null) {
                createOrUpdateClassLoaderLocked(null /*addedPaths*/);
            }
            return mClassLoader;
        }
    }
}

```

调用`createOrUpdateClassLoaderLocked`。

#### 3.2.5. LoadedApk.createOrUpdateClassLoaderLocked

```
package android.app;
public final class LoadedApk {
 
    private void createOrUpdateClassLoaderLocked(List addedPaths) {
        //...省略，chinamima@163.com
 
        // If we're not asked to include code, we construct a classloader that has
        // no code path included. We still need to set up the library search paths
        // and permitted path because NativeActivity relies on it (it attempts to
        // call System.loadLibrary() on a classloader from a LoadedApk with
        // mIncludeCode == false).
        if (!mIncludeCode) {
            if (mClassLoader == null) {
                //...省略，chinamima@163.com
                mClassLoader = ApplicationLoaders.getDefault().getClassLoader(
                        "" /* codePath */, mApplicationInfo.targetSdkVersion, isBundledApp,
                        librarySearchPath, libraryPermittedPath, mBaseClassLoader,
                        null /* classLoaderName */);
                //...省略，chinamima@163.com
            }
 
            return;
        }
        //...省略，chinamima@163.com
    }
} 
```

调用`ApplicationLoaders.getClassLoader`获取`ClassLoader`。

#### 3.2.6. ApplicationLoaders.getClassLoader

```
package android.app;
 
public class ApplicationLoaders {
    public static ApplicationLoaders getDefault() {
        return gApplicationLoaders;
    }
 
    ClassLoader getClassLoader(String zip, int targetSdkVersion, boolean isBundled,
                               String librarySearchPath, String libraryPermittedPath,
                               ClassLoader parent, String classLoaderName) {
        // For normal usage the cache key used is the same as the zip path.
        return getClassLoader(zip, targetSdkVersion, isBundled, librarySearchPath,
                              libraryPermittedPath, parent, zip, classLoaderName);
    }
 
    private ClassLoader getClassLoader(String zip, int targetSdkVersion, boolean isBundled,
                                       String librarySearchPath, String libraryPermittedPath,
                                       ClassLoader parent, String cacheKey,
                                       String classLoaderName) {
        /*
         * This is the parent we use if they pass "null" in.  In theory
         * this should be the "system" class loader; in practice we
         * don't use that and can happily (and more efficiently) use the
         * bootstrap class loader.
         */
        ClassLoader baseParent = ClassLoader.getSystemClassLoader().getParent();
 
        synchronized (mLoaders) {
            if (parent == null) {
                parent = baseParent;
            }
 
            /*
             * If we're one step up from the base class loader, find
             * something in our cache.  Otherwise, we create a whole
             * new ClassLoader for the zip archive.
             */
            if (parent == baseParent) {
                ClassLoader loader = mLoaders.get(cacheKey);
                if (loader != null) {
                    return loader;
                }
                //...省略，chinamima@163.com
                ClassLoader classloader = ClassLoaderFactory.createClassLoader(
                        zip,  librarySearchPath, libraryPermittedPath, parent,
                        targetSdkVersion, isBundled, classLoaderName);
                //...省略，chinamima@163.com
                mLoaders.put(cacheKey, classloader);
                return classloader;
            }
 
            ClassLoader loader = ClassLoaderFactory.createClassLoader(
                    zip, null, parent, classLoaderName);
            return loader;
        }
    }
 
    private final ArrayMap mLoaders = new ArrayMap<>();
 
    private static final ApplicationLoaders gApplicationLoaders = new ApplicationLoaders(); 
```

如`mLoader`已存在，则直接返回。  
如`mLoader`不存在，则调用`ClassLoaderFactory.createClassLoader`创建。

#### 3.2.7. ClassLoaderFactory.createClassLoader

```
package com.android.internal.os;
 
public class ClassLoaderFactory {
 
    public static ClassLoader createClassLoader(String dexPath,
            String librarySearchPath, ClassLoader parent, String classloaderName) {
        if (isPathClassLoaderName(classloaderName)) {
            return new PathClassLoader(dexPath, librarySearchPath, parent); //新建PathClassLoader实例
        } else if (isDelegateLastClassLoaderName(classloaderName)) {
            return new DelegateLastClassLoader(dexPath, librarySearchPath, parent);
        }
 
        throw new AssertionError("Invalid classLoaderName: " + classloaderName);
    }
 
    public static ClassLoader createClassLoader(String dexPath,
            String librarySearchPath, String libraryPermittedPath, ClassLoader parent,
            int targetSdkVersion, boolean isNamespaceShared, String classloaderName) {
 
        final ClassLoader classLoader = createClassLoader(dexPath, librarySearchPath, parent,
                classloaderName);
        //...省略，chinamima@163.com
        return classLoader;
    }
}

```

新建一个 PathClassLoader 实例。

4. findClass 源码路径
-----------------

### 4.1. BaseDexClassLoader

```
package dalvik.system;
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
 
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent, boolean isTrusted) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
        //...省略，chinamima@163.com
    }
    protected Class findClass(String name) throws ClassNotFoundException {
        //从DexPathList里找类
        Class c = pathList.findClass(name, suppressedExceptions);
        //...省略，chinamima@163.com
        return c;
    }   
 
}

```

### 4.2. DexPathList

```
package dalvik.system;
final class DexPathList {
    private Element[] dexElements;
 
    public Class findClass(String name, List suppressed) {
        for (Element element : dexElements) {
            //从Element里找类
            Class clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
        //...省略，chinamima@163.com
        return null;
    }
} 
```

### 4.3. Element

```
static class Element {
    private final File path;
    private final DexFile dexFile;
 
    public Element(DexFile dexFile, File dexZipPath) {
        this.dexFile = dexFile;
        this.path = dexZipPath;
    }
 
    public Class findClass(String name, ClassLoader definingContext,
            List suppressed) {
        //从DexFile里找类
        return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed)
                : null;
    }
} 
```

### 4.4. DexFile

> java/dalvik/system/DexFile.java

```
public final class DexFile {
    private Object mCookie;
 
    public Class loadClassBinaryName(String name, ClassLoader loader, List suppressed) {
        return defineClass(name, loader, mCookie, this, suppressed);
    }
 
    private static Class defineClass(String name, ClassLoader loader, Object cookie,
                                     DexFile dexFile, List suppressed) {
        Class result = null;
        try {
            result = defineClassNative(name, loader, cookie, dexFile);
        } catch (NoClassDefFoundError e) {
        } catch (ClassNotFoundException e) {
        }
        return result;
    }
 
    private static native Class defineClassNative(String name, ClassLoader loader, Object cookie,
                                                  DexFile dexFile);
} 
```

### 4.5. dalvik_system_DexFile

> /art/runtime/native/dalvik_system_DexFile.cc

```
static jclass DexFile_defineClassNative(JNIEnv* env,
                                        jclass,
                                        jstring javaName,
                                        jobject javaLoader,
                                        jobject cookie,
                                        jobject dexFile) {
  std::vector dex_files;
  const OatFile* oat_file;
  if (!ConvertJavaArrayToDexFiles(env, cookie, /*out*/ dex_files, /*out*/ oat_file)) {
    //...省略，chinamima@163.com
    return nullptr;
  }
 
  ScopedUtfChars class_name(env, javaName);
  //...省略，chinamima@163.com
  const std::string descriptor(DotToDescriptor(class_name.c_str()));
  const size_t hash(ComputeModifiedUtf8Hash(descriptor.c_str()));
  for (auto& dex_file : dex_files) {
    const DexFile::ClassDef* dex_class_def =
        OatDexFile::FindClassDef(*dex_file, descriptor.c_str(), hash);
    if (dex_class_def != nullptr) {
      ScopedObjectAccess soa(env);
      ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
      StackHandleScope<1> hs(soa.Self());
      Handle class_loader(
          hs.NewHandle(soa.Decode(javaLoader)));
      ObjPtr dex_cache =
          class_linker->RegisterDexFile(*dex_file, class_loader.Get());
      //...省略，chinamima@163.com
      ObjPtr result = class_linker->DefineClass(soa.Self(),
                                                               descriptor.c_str(),
                                                               hash,
                                                               class_loader,
                                                               *dex_file,
                                                               *dex_class_def);
      class_linker->InsertDexFileInToClassLoader(soa.Decode(dexFile),
                                                 class_loader.Get());
      if (result != nullptr) {
        VLOG(class_linker) << "DexFile_defineClassNative returning " << result
                           << " for " << class_name.c_str();
        return soa.AddLocalReference(result);
      }
    }
  }
  VLOG(class_linker) << "Failed to find dex_class_def " << class_name.c_str();
  return nullptr;
} 
```

`cookie`通过`ConvertJavaArrayToDexFiles(env, cookie, /*out*/ dex_files, /*out*/ oat_file)`转换成`DexFile*`实例。  
再通过`class_linker->DefineClass`从`DexFile*`里加载类。

5. 替换 mCookie
-------------

### 5.1. 关键点

可以看到`DexFile_defineClassNative`里的`ConvertJavaArrayToDexFiles`函数把`cookie`和`dex_files`、`oat_file`关联了起来。

> /art/runtime/native/dalvik_system_DexFile.cc

```
static jclass DexFile_defineClassNative(JNIEnv* env,
                                        jclass,
                                        jstring javaName,
                                        jobject javaLoader,
                                        jobject cookie,
                                        jobject dexFile) {
  std::vector dex_files;
  const OatFile* oat_file;
  if (!ConvertJavaArrayToDexFiles(env, cookie, /*out*/ dex_files, /*out*/ oat_file)) {
    VLOG(class_linker) << "Failed to find dex_file";
    DCHECK(env->ExceptionCheck());
    return nullptr;
  }
  //...省略，chinamima@163.com
} 
```

从这里可以猜测得出，`cookie`和`dex_files`、`oat_file`是一对一关系的。  
因此，只需要把`cookie`替换成另一个 dex 的值，就可以加载另一个 dex 的类。

### 5.2. 思路

替换`cookie`值在 native 层比较难操作，因此我们把目光放到 java 层。

> java/dalvik/system/DexFile.java

```
public final class DexFile {
    private Object mCookie;
 
    DexFile(String fileName, ClassLoader loader, DexPathList.Element[] elements) throws IOException {
        mCookie = openDexFile(fileName, null, 0, loader, elements);
        mInternalCookie = mCookie;
        mFileName = fileName;
    }
    //...省略，chinamima@163.com
}

```

在`DexFile`里也有一个`mCookie`，通过`defineClassNative`传参到`DexFile_defineClassNative`的。  
也就是替换`mCookie`即可达成目的 -- 加载其他 dex。

### 5.3. 实现

根据`DexFile`的构造函数可知。  
调用`DexFile`的`openDexFile`得到一个新的`cookie`，用于替换原始的`mCookie`。

### 5.4. 思考

*   `openDexFile`需要一个 dex 的路径，也就是 dex 需要作为文件释放到某处，容易被截获。
*   art 模式下，`openDexFile`会调用`dex2oat`生成 oat 文件，造成一定的性能损耗。

6. 替换 ClassLoader
-----------------

### 6.1. 覆盖 ClassLoader 成员变量

![](https://bbs.pediy.com/upload/attach/202104/633423_GCSUEZXVFJVSA5K.png)

### 6.2. 替换 LoadedApk 里 ClassLoader

由`初始化ClassLoader`的调用链路可知，`LoadedApk`保存了`ClassLoader`的实例，实际上这个实例就是`findClass`时用到的`PathClassLoader`。  
因此，新 new 一个`ClassLoader`并替换掉`LoadedApk`里的`mClassLoader`即能实现替换 dex 效果。

7. 添加到 DexPathList
------------------

```
package dalvik.system;
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
 
    protected Class findClass(String name) throws ClassNotFoundException {
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            //...省略，chinamima@163.com
        }
        return c;
    }       
}

```

```
package dalvik.system;
final class DexPathList {
    private Element[] dexElements;
 
    public Class findClass(String name, List suppressed) {
        for (Element element : dexElements) {
            Class clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
        //...省略，chinamima@163.com
        return null;
    }
} 
```

从上面代码可知，`ClassLoader`是从`DexPathList pathList`里加载 class 的。  
而`DexPathList`是由`Element[] dexElements`组成的，在`findClass`最先从数组最前面取`Element`。  
因此，我们可以在`DexPathList.dexElements`的前面插入隐藏 dex 的`Element`，从而实现加载隐藏 dex。

8. 直接加载 dex byte
----------------

```
public final class DexFile {
    private Object mCookie;
 
    DexFile(ByteBuffer buf) throws IOException {
        mCookie = openInMemoryDexFile(buf);
        mInternalCookie = mCookie;
        mFileName = null;
    }
 
    private static Object openInMemoryDexFile(ByteBuffer buf) throws IOException {
        if (buf.isDirect()) {
            return createCookieWithDirectBuffer(buf, buf.position(), buf.limit());
        } else {
            return createCookieWithArray(buf.array(), buf.position(), buf.limit());
        }
    }
}

```

发现`DexFile.openInMemoryDexFile`可以直接加载 buffer，因此可以从内存中加载 dex。

### 8.1. 从内存中加载 dex，避免生成 oat

#### 8.1.1. 调用链路

```
DexFile.openInMemoryDexFile
 ├─ DexFile_createCookieWithDirectBuffer
 │  └─ CreateSingleDexFileCookie
 │      └─ CreateDexFile
 │          └─ ArtDexFileLoader::open
 │              └─ ArtDexFileLoader::OpenCommon
 └──────── ConvertDexFilesToJavaArray

```

关键点在于`ArtDexFileLoader::open`，此处传递了一个`kNoOatDexFile`参数，明显是无`oat`文件的意思。

#### 8.1.2. 详细源码

> dalvik_system_DexFile.cc

```
static jobject DexFile_createCookieWithDirectBuffer(JNIEnv* env,
                                                    jclass,
                                                    jobject buffer,
                                                    jint start,
                                                    jint end) {
  //...省略，chinamima@163.com
  std::unique_ptr dex_mem_map(AllocateDexMemoryMap(env, start, end));
  //...省略，chinamima@163.com
  //创建单个DexFile实例，并转成cookie
  return CreateSingleDexFileCookie(env, std::move(dex_mem_map));
} 
```

```
static jobject CreateSingleDexFileCookie(JNIEnv* env, std::unique_ptr data) {
  //创建DexFile
  std::unique_ptr dex_file(CreateDexFile(env, std::move(data)));
  if (dex_file.get() == nullptr) {
    DCHECK(env->ExceptionCheck());
    return nullptr;
  }
  std::vector> dex_files;
  dex_files.push_back(std::move(dex_file));
  //转成cookie
  return ConvertDexFilesToJavaArray(env, nullptr, dex_files);
} 
```

```
static const DexFile* CreateDexFile(JNIEnv* env, std::unique_ptr dex_mem_map) {
  std::string location = StringPrintf("Anonymous-DexFile@%p-%p",
                                      dex_mem_map->Begin(),
                                      dex_mem_map->End());
  std::string error_message;
  const ArtDexFileLoader dex_file_loader;
  //打开DexFile
  std::unique_ptr dex_file(dex_file_loader.Open(location, //Anonymous-DexFile@0xaabbccdd-0xeeff0011
                                                               0,
                                                               std::move(dex_mem_map),
                                                               /* verify */ true,
                                                               /* verify_location */ true,
                                                               &error_message));
  //...省略，chinamima@163.com
  return dex_file.release();
} 
```

> art_dex_file_loader.cc

```
static constexpr OatDexFile* kNoOatDexFile = nullptr;
 
std::unique_ptr ArtDexFileLoader::Open(const uint8_t* base,
                                                      size_t size,
                                                      const std::string& location,
                                                      uint32_t location_checksum,
                                                      const OatDexFile* oat_dex_file,
                                                      bool verify,
                                                      bool verify_checksum,
                                                      std::string* error_msg) const {
  ScopedTrace trace(std::string("Open dex file from RAM ") + location);
  return OpenCommon(base,
                    size,
                    /*data_base*/ nullptr,
                    /*data_size*/ 0u,
                    location,
                    location_checksum,
                    oat_dex_file,         //有oat文件
                    verify,
                    verify_checksum,
                    error_msg,
                    /*container*/ nullptr,
                    /*verify_result*/ nullptr);
}
 
std::unique_ptr ArtDexFileLoader::Open(const std::string& location,
                                                      uint32_t location_checksum,
                                                      std::unique_ptr map,
                                                      bool verify,
                                                      bool verify_checksum,
                                                      std::string* error_msg) const {
  //...省略，chinamima@163.com
  std::unique_ptr dex_file = OpenCommon(map->Begin(),
                                                 map->Size(),
                                                 /*data_base*/ nullptr,
                                                 /*data_size*/ 0u,
                                                 location,
                                                 location_checksum,
                                                 kNoOatDexFile,           //无oat文件！
                                                 verify,
                                                 verify_checksum,
                                                 error_msg,
                                                 std::make_unique(std::move(map)),
                                                 /*verify_result*/ nullptr);
  //...省略，chinamima@163.com
  return dex_file;
} 
```

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 1 天前 被 chinamima 编辑 ，原因：