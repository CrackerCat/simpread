> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282594.htm)

> [原创]aosp10 的类加载流程源码注释

目录

*                    [版本信息](#版本信息)
*                    [前置知识点](#前置知识点)
*                            [1. `ClassLoader`](#1-classloader)
*                            [2. `BootClassLoader`](#2-bootclassloader)
*                            [3. `BaseDexClassLoader`](#3-basedexclassloader)
*                            [4. `PathClassLoader`](#4-pathclassloader)
*                            [5. `DexClassLoader`](#5-dexclassloader)
*                    [分析流程](#分析流程)
*                            [ClassLoader](#classloader)
*                            [DexClassLoader](#dexclassloader)
*                            [BaseDexClassLoader](#basedexclassloader)
*                            [DexPathList](#dexpathlist)
*                                    [element.findClass](#elementfindclass)
*                                    [loadClassBinaryName](#loadclassbinaryname)
*                            [DexFile](#dexfile)
*                                    [defineClassNative](#defineclassnative)
*                            [dalvik_system_DexFile](#dalvik_system_dexfile)
*                                    [DefineClass](#defineclass)
*                                    [SetupClass](#setupclass)
*                                    [LoadClass](#loadclass)

### 版本信息

aosp 版本: android-10.0.0_r47

在线查看 aosp 代码平台 (版本很全):[https://cs.android.com/(需要特殊的才能进去)](https://cs.android.com/(%E9%9C%80%E8%A6%81%E7%89%B9%E6%AE%8A%E7%9A%84%E6%89%8D%E8%83%BD%E8%BF%9B%E5%8E%BB))

### 前置知识点

#### 1. `ClassLoader`

`ClassLoader` 是所有类加载器的基类，定义了类加载器的基本行为。它的主要职责是提供类加载的基本机制。

关键方法：

*   `loadClass(String name)`: 加载指定名称的类。
*   `findClass(String name)`: 查找类的定义。
*   `findLoadedClass(String name)`: 查找已经加载的类。

#### 2. `BootClassLoader`

`BootClassLoader` 是用于加载 Android 系统类的类加载器。它是 `ClassLoader` 的内部类，通常由虚拟机启动时初始化，并且开发者无法直接调用。

用途：

*   加载核心库类，如 `java.lang` 包和 `android.*` 包的类。
*   由引导类路径（bootstrap class path）指定的类。

#### 3. `BaseDexClassLoader`

`BaseDexClassLoader` 是一个抽象类，继承自 `ClassLoader`。它提供了加载 DEX 文件的基本功能，是 `PathClassLoader` 和 `DexClassLoader` 的父类。

关键属性：

*   `DexPathList dexPathList`: 管理 DEX 文件和优化的 DEX 文件（ODEX）。
*   `findClass(String name)`: 在 DEX 文件中查找类。

#### 4. `PathClassLoader`

`PathClassLoader` 继承自 `BaseDexClassLoader`，通常用于加载 APK 文件中的类，包括应用程序自身的类和第三方库的类。

用途：

*   加载 APK 文件中的应用程序类和资源。
*   加载第三方库。

#### 5. `DexClassLoader`

`DexClassLoader` 也是继承自 `BaseDexClassLoader`，主要用于动态加载，能够加载指定路径的 APK、JAR、ZIP 和 DEX 文件。因此，很多热修复和插件化方案都会使用 `DexClassLoader`。

用途：

*   动态加载 DEX 文件、JAR 文件和 APK 文件。
*   支持热修复和插件化。

### 分析流程

#### ClassLoader

由上面的前置知识点可知`ClassLoader` 是所有类加载器的基类 里面的 loadClass(String name) 用于加载指定名称的类那就从这里开始分析 在源码注释上的 **第三步：如果类依然没有找到，则使用 findClass 方法来查找类** `findClass` 方法是 `ClassLoader` 类中的抽象方法。**BaseDexClassLoader** 及其子类（如 `DexClassLoader`）提供了具体的实现 这边先去看 **DexClassLoader** 里面的代码

路径为:/external/apache-commons-bcel/src/main/java/org/apache/bcel/util/ClassLoader.java

```
protected Class loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    // 第一步：检查类是否已经被加载
    Class c = findLoadedClass(name);
     
    // 如果类尚未加载
    if (c == null) {
        try {
            // 第二步：委托给父类加载器加载
            // 如果存在父加载器，则尝试使用父加载器加载类
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                // 如果没有父加载器，则尝试从引导类加载器中加载类
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
 
        }
 
        // 第三步：如果类依然没有找到，则使用 findClass 方法来查找类
        if (c == null) {
            c = findClass(name);
        }
    }
     
    // 第四步：如果需要解析类，则进行解析
    if (resolve) {
        resolveClass(c);
    }
     
    return c;
}

```

#### DexClassLoader

这里直接去看 **BaseDexClassLoader**

路径为: libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java

```
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        // 调用BaseDexClassLoader的构造函数。
        // dexPath参数指定了包含 DEX 文件的路径，optimizedDirectory参数已废弃，
        // librarySearchPath用于查找本地库，parent是父类加载器。
        super(dexPath, null, librarySearchPath, parent);
    }
}

```

#### BaseDexClassLoader

在 **BaseDexClassLoader** 使用 **DexPathList** 实例中的 `findClass` 方法查找类 继续跟进

路径为: libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java

```
protected Class findClass(String name) throws ClassNotFoundException {
    // 首先，检查共享库加载器中是否可以找到指定的类。
    // `sharedLibraryLoaders` 是一个 ClassLoader 列表，包含了可以用于加载共享库的加载器。
    if (sharedLibraryLoaders != null) {
        for (ClassLoader loader : sharedLibraryLoaders) {
            try {
                // 尝试通过共享库加载器加载类。如果加载成功，立即返回该类。
                return loader.loadClass(name);
            } catch (ClassNotFoundException ignored) {
                // 如果在共享库加载器中未找到该类，则继续尝试下一个加载器。
                // 这里的异常被忽略，因为我们要继续检查其他加载器。
            }
        }
    }
     
    // 如果在共享库加载器中没有找到该类，接下来在当前类加载器的 dexPath 中查找。
    // `pathList` 是一个 DexPathList 实例，负责从 DEX 文件中加载类。
    List suppressedExceptions = new ArrayList();
    // `findClass` 方法会在 `pathList` 所管理的 DEX 文件中查找并返回类的字节码。
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c != null) {
        // 如果在 DEX 文件中找到该类，则返回它。
        return c;
    }
     
    // 如果类仍然未找到，检查 "after" 共享库中是否可以找到该类。
    // `sharedLibraryLoadersAfter` 是一个 ClassLoader 列表，包含了当前加载器之后的共享库加载器。
    if (sharedLibraryLoadersAfter != null) {
        for (ClassLoader loader : sharedLibraryLoadersAfter) {
            try {
                // 尝试通过 "after" 共享库加载器加载类。如果加载成功，立即返回该类。
                return loader.loadClass(name);
            } catch (ClassNotFoundException ignored) {
                // 如果在 "after" 共享库加载器中未找到该类，则继续尝试下一个加载器。
                // 这里的异常被忽略，因为我们要继续检查其他加载器。
            }
        }
    }
     
    // 如果在所有检查中都未找到该类，则抛出 ClassNotFoundException。
    // 创建一个 ClassNotFoundException 实例，并附加所有在查找过程中收集到的异常。
    if (c == null) {
        ClassNotFoundException cnfe = new ClassNotFoundException(
                "Didn't find class \"" + name + "\" on path: " + pathList);
        // 将收集到的异常附加到主异常中，以便提供详细的错误信息。
        for (Throwable t : suppressedExceptions) {
            cnfe.addSuppressed(t);
        }
        // 抛出 ClassNotFoundException，表示指定的类未找到。
        throw cnfe;
    }
    // 如果在最后的步骤中找到类，返回该类。
    return c;
} 
```

#### DexPathList

在 **DexPathList** 注释写的很清楚 dexElements 里面的 Element 对象封装了 DEX 文件的路径、加载器、以及其他相关信息 对于每一个 **element.findClass** 方法尝试在其对应的 DEX 文件中找到指定的类。如果找到，则返回该类；如果未找到，则返回 `null` 这里继续跟进去看 **element.findClass** 的具体代码

路径为: libcore/dalvik/src/main/java/dalvik/system/DexPathList.java

```
public Class findClass(String name, List suppressed) {
    // 遍历 dexElements 中的每一个 Element
    // `dexElements` 是一个 Element 数组，每个 Element 代表一个 DEX 文件或类路径中的类加载器。
    for (Element element : dexElements) {
        // 在当前 Element 中查找指定的类
        // `findClass` 方法尝试在 Element 所管理的 DEX 文件中查找类。
        // `definingContext` 是类定义的上下文，`suppressed` 是用于存储查找过程中的异常的列表。
        Class clazz = element.findClass(name, definingContext, suppressed);
        if (clazz != null) {
            // 如果找到了类，则返回它
            return clazz;
        }
    }
 
    // 如果在所有 dexElements 中未找到类，检查并添加所有 `dexElementsSuppressedExceptions`
    // `dexElementsSuppressedExceptions` 是在查找过程中可能收集到的异常数组
    if (dexElementsSuppressedExceptions != null) {
        // 将所有收集到的异常添加到 `suppressed` 列表中
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
     
    // 如果仍未找到类，则返回 null
    return null;
} 
```

##### element.findClass

在 **element.findClass** 里 name 传到了 **loadClassBinaryName** 继续跟入

路径为: libcore/dalvik/src/main/java/dalvik/system/DexPathList.java

```
public Class findClass(String name, ClassLoader definingContext,
        List suppressed) {
    // 检查 `dexFile` 是否为 null。
    // `dexFile` 是一个包含 DEX 文件内容的对象，它封装了对 DEX 文件的访问。
    return dexFile != null
        ? // 如果 `dexFile` 不为 null，则调用 `dexFile.loadClassBinaryName` 方法来查找指定的类。
          // `name` 是需要查找的类的全名，例如 "com.example.MyClass"。
          // `definingContext` 是定义类的上下文，通常是当前的 ClassLoader 实例。
          // `suppressed` 是一个列表，用于收集在查找过程中发生的异常。
          dexFile.loadClassBinaryName(name, definingContext, suppressed)
        : // 如果 `dexFile` 为 null，则返回 null。
          // 这表示当前 `Element` 对象没有关联的 DEX 文件，因此无法进行类的查找。
          null;
} 
```

##### loadClassBinaryName

在 `DexFile` 类中，`loadClassBinaryName` 方法用于从 DEX 文件中加载指定的类。它使用 `defineClass` 方法来定义和加载类

继续跟如 **defineClass**

路径为: libcore/dalvik/src/main/java/dalvik/system/DexPathList.java

```
@UnsupportedAppUsage
public Class loadClassBinaryName(String name, ClassLoader loader, List suppressed) {
    // 调用 defineClass 方法来定义并加载类。
    // `name` 是要加载的类的全名，例如 "com.example.MyClass"。
    // `loader` 是当前的 ClassLoader，它提供了加载类的上下文。
    // `mCookie` 是一个用于标识 DEX 文件的标记。
    // `this` 是当前的 DexFile 实例。
    // `suppressed` 是一个列表，用于记录在加载过程中产生的异常。
    return defineClass(name, loader, mCookie, this, suppressed);
} 
```

#### DexFile

`defineClass` 是一个私有静态方法，用于从 DEX 文件中定义并加载一个类。它依赖于 `defineClassNative` 来处理

继续跟入去看 **defineClassNative**

路径为: libcore/dalvik/src/main/java/dalvik/system/DexFile.java

```
private static Class defineClass(String name, ClassLoader loader, Object cookie,
                                 DexFile dexFile, List suppressed) {
    Class result = null; // 定义一个变量来存储最终定义的类
 
    try {
        // 调用本地方法 defineClassNative 来定义类。
        // 该方法使用提供的类名、ClassLoader、cookie 和 DexFile 来定义类。
        result = defineClassNative(name, loader, cookie, dexFile);
    } catch (NoClassDefFoundError e) {
        // 捕获 NoClassDefFoundError 异常。
        // 这个异常表示在定义类时找不到类的定义。
        if (suppressed != null) {
            // 如果 suppressed 列表不为 null，则将异常添加到列表中。
            suppressed.add(e);
        }
    } catch (ClassNotFoundException e) {
        // 捕获 ClassNotFoundException 异常。
        // 这个异常表示在加载过程中找不到指定的类。
        if (suppressed != null) {
            // 如果 suppressed 列表不为 null，则将异常添加到列表中。
            suppressed.add(e);
        }
    }
 
    // 返回定义的类，如果定义过程中发生了异常，则 result 可能为 null。
    return result;
} 
```

##### defineClassNative

**defineClassNative** 是一个 Native 方法 这里去看 **DexFile_defineClassNative** 里面的具体实现

路径为: libcore/dalvik/src/main/java/dalvik/system/DexFile.java

```
private static native Class defineClassNative(String name, ClassLoader loader, Object cookie,
                                              DexFile dexFile)
        throws ClassNotFoundException, NoClassDefFoundError;

```

#### dalvik_system_DexFile

**DexFile_defineClassNative** 主要关注咱们的类加载器流程也就是 **DefineClass** 跟入去看看

路径为: art/runtime/native/dalvik_system_DexFile.cc

```
static jclass DexFile_defineClassNative(JNIEnv* env,
                                        jclass,
                                        jstring javaName,
                                        jobject javaLoader,
                                        jobject cookie,
                                        jobject dexFile) {
  // 创建一个用于存放 Dex 文件指针的向量，以及一个 OatFile 指针。
  // 这些 Dex 文件将用于查找和定义类。
  std::vector dex_files;
  const OatFile* oat_file;
 
  // ConvertJavaArrayToDexFiles：将传入的 cookie 对象转换为 Dex 文件列表，并可能获取 Oat 文件。
  // `cookie` 通常包含有关 Dex 文件的位置和其他信息。
  if (!ConvertJavaArrayToDexFiles(env, cookie, /*out*/ dex_files, /*out*/ oat_file)) {
    VLOG(class_linker) << "Failed to find dex_file"; // 日志记录失败原因。
    DCHECK(env->ExceptionCheck()); // 检查是否有异常发生。
    return nullptr; // 返回空值表示失败。
  }
 
  // 将 Java 字符串转换为 UTF-8 编码的 C 字符串。
  // `ScopedUtfChars` 是一个 RAII 类，用于 Java 字符串转换。
  ScopedUtfChars class_name(env, javaName);
  // 检查转换是否成功。
  if (class_name.c_str() == nullptr) {
    VLOG(class_linker) << "Failed to find class_name"; // 日志记录转换失败。
    return nullptr; // 返回空值表示失败。
  }
 
  // 将类名从点分格式（如 `com.example.MyClass`）转换为 Java 虚拟机使用的描述符格式（如 `Lcom/example/MyClass;`）。
  // 描述符用于标识类，特别是在 Dex 文件中。
  const std::string descriptor(DotToDescriptor(class_name.c_str()));
  // 计算描述符的哈希值，用于加速查找。
  const size_t hash(ComputeModifiedUtf8Hash(descriptor.c_str()));
 
  // 遍历所有的 Dex 文件，尝试在每个 Dex 文件中查找类定义。
  for (auto& dex_file : dex_files) {
    // `OatDexFile::FindClassDef`：在当前 Dex 文件中查找类定义。
    // 根据类描述符和哈希值来查找类定义。
    const dex::ClassDef* dex_class_def =
        OatDexFile::FindClassDef(*dex_file, descriptor.c_str(), hash);
 
    // 如果找到类定义。
    if (dex_class_def != nullptr) {
      // 创建 ScopedObjectAccess 实例以确保 JNI 环境的正确访问。
      ScopedObjectAccess soa(env);
      // 获取当前的 ClassLinker 实例，该实例负责类加载和链接操作。
      ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
 
      // 创建 StackHandleScope 用于管理 JNI 句柄的生命周期。
      StackHandleScope<1> hs(soa.Self());
      // 创建一个 Handle 对象表示 Java 类加载器。
      Handle class_loader(
          hs.NewHandle(soa.Decode(javaLoader)));
 
      // 注册 Dex 文件并获取 Dex 缓存。Dex 缓存用于加速类加载过程。
      ObjPtr dex_cache =
          class_linker->RegisterDexFile(*dex_file, class_loader.Get());
 
      // 如果注册失败，可能是内存不足（OOME）或 Dex 文件已经被不同的类加载器注册。
      if (dex_cache == nullptr) {
        soa.Self()->AssertPendingException(); // 断言异常已经发生。
        return nullptr; // 返回空值表示失败。
      }
 
      // `DefineClass`：在 JVM 中定义类，并将其添加到类加载器中。
      ObjPtr result = class_linker->DefineClass(soa.Self(),
                                                               descriptor.c_str(),
                                                               hash,
                                                               class_loader,
                                                               *dex_file,
                                                               *dex_class_def);
 
      // 将 Dex 文件插入到InsertDexFileInToClassLoader中
      class_linker->InsertDexFileInToClassLoader(soa.Decode(dexFile),
                                                 class_loader.Get());
 
      // 如果成功定义类，返回类的局部引用。
      if (result != nullptr) {
        VLOG(class_linker) << "DexFile_defineClassNative returning " << result
                           << " for " << class_name.c_str();
        return soa.AddLocalReference(result); // 返回类的局部引用。
      }
    }
  }
 
  // 如果没有找到类定义，记录日志并返回空值。
  VLOG(class_linker) << "Failed to find dex_class_def " << class_name.c_str();
  return nullptr; // 返回空值表示失败。
} 
```

##### DefineClass

在 DefineClass 里面 是很核心的一个点了

需要关注的分别是

*   ##### 分配
    
    *   类的分配是为新定义的类对象分配内存空间

*   ##### 初始化
    
    *   通过 `SetupClass` 函数，为新分配的类对象设置 Dex 缓存、类的状态等。这一步确保类的基本信息在内存中正确设置。
    *   对于某些特殊类（如 `java.lang.String`），需要进行额外的初始化，例如标记类的特殊状态。

*   ##### 加载
    
    *   类的加载过程包括从 Dex 文件中提取类的信息，并将其应用到类对象上
    *   `LoadClass` 函数从 Dex 文件中提取类的字段和方法信息，并将其加载到 `klass` 对象中。这一步确保类对象包含正确的字段和方法定义。
*   ##### 链接
    
    *   类的链接包括将类的字段、方法等实际链接到类对象中，并解决类之间的依赖关系

这里去看一下关于

类的初始化 **SetupClass**

```
SetupClass(*new_dex_file, *new_class_def, klass, class_loader.Get());

```

类的加载 **LoadClass** 里面都干了一些啥 先去看 **SetupClass** 里面干了一些什么东西

```
LoadClass(self, *new_dex_file, *new_class_def, klass);

```

路径为: art/runtime/class_linker.cc

```
ObjPtr ClassLinker::DefineClass(Thread* self,
                                               const char* descriptor,
                                               size_t hash,
                                               Handle class_loader,
                                               const DexFile& dex_file,
                                               const dex::ClassDef& dex_class_def) {
  // 创建一个 ScopedDefiningClass 实例，用于管理当前线程在类定义过程中的状态。
  ScopedDefiningClass sdc(self);
  // 创建一个 StackHandleScope 实例，用于管理多个 Handle 对象的生命周期。
  StackHandleScope<3> hs(self);
  // 记录类加载过程的总时间。
  metrics::AutoTimer timer{GetMetrics()->ClassLoadingTotalTime()};
  // 记录类加载的增量时间。
  metrics::AutoTimer timeDelta{GetMetrics()->ClassLoadingTotalTimeDelta()};
  // 创建一个 Handle 对象，用于存储加载的类。
  auto klass = hs.NewHandle(nullptr);
 
  // 如果初始化尚未完成，则处理一些特殊的内置类。
  if (UNLIKELY(!init_done_)) {
    // 根据描述符获取内置类的根对象。
    if (strcmp(descriptor, "Ljava/lang/Object;") == 0) {
      klass.Assign(GetClassRoot(this));
    } else if (strcmp(descriptor, "Ljava/lang/Class;") == 0) {
      klass.Assign(GetClassRoot(this));
    } else if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
      klass.Assign(GetClassRoot(this));
    } else if (strcmp(descriptor, "Ljava/lang/ref/Reference;") == 0) {
      klass.Assign(GetClassRoot(this));
    } else if (strcmp(descriptor, "Ljava/lang/DexCache;") == 0) {
      klass.Assign(GetClassRoot(this));
    } else if (strcmp(descriptor, "Ldalvik/system/ClassExt;") == 0) {
      klass.Assign(GetClassRoot(this));
    }
  }
 
  // 在 AOT 编译模式下，如果 SDK 检查不允许该描述符，则阻止类定义。
  if (class_loader == nullptr &&
      Runtime::Current()->IsAotCompiler() &&
      DenyAccessBasedOnPublicSdk(descriptor)) {
    ObjPtr pre_allocated =
        Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
    self->SetException(pre_allocated);
    return sdc.Finish(nullptr);
  }
 
  // 如果当前线程不能加载类，阻止进一步操作。
  if (!self->CanLoadClasses()) {
    ObjPtr pre_allocated =
        Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
    self->SetException(pre_allocated);
    return sdc.Finish(nullptr);
  }
 
  // 记录类加载的跟踪信息。
  ScopedTrace trace(descriptor);
  if (klass == nullptr) {
    // 如果 `klass` 为空，分配一个新的类对象。
    // 计算类的大小（不包括嵌入表），并分配内存。
    if (CanAllocClass()) {
      klass.Assign(AllocClass(self, SizeOfClassWithoutEmbeddedTables(dex_file, dex_class_def)));
    } else {
      return sdc.Finish(nullptr);
    }
  }
  if (UNLIKELY(klass == nullptr)) {
    self->AssertPendingOOMException(); // 如果分配失败，检查是否发生了内存不足异常。
    return sdc.Finish(nullptr);
  }
   
  // 获取实际的 Dex 文件和类定义（可能会经过回调修改）。
  DexFile const* new_dex_file = nullptr;
  dex::ClassDef const* new_class_def = nullptr;
  // 执行 Runtime 回调函数，可能会调整 dex 文件和类定义。
  Runtime::Current()->GetRuntimeCallbacks()->ClassPreDefine(descriptor,
                                                            klass,
                                                            class_loader,
                                                            dex_file,
                                                            dex_class_def,
                                                            &new_dex_file,
                                                            &new_class_def);
  // 检查是否在回调过程中发生了异常。
  if (self->IsExceptionPending()) {
    return sdc.Finish(nullptr);
  }
   
  // 注册 Dex 文件，并获取 Dex 缓存。
  ObjPtr dex_cache = RegisterDexFile(*new_dex_file, class_loader.Get());
  if (dex_cache == nullptr) {
    self->AssertPendingException();
    return sdc.Finish(nullptr);
  }
  klass->SetDexCache(dex_cache); // 设置类的 Dex 缓存。
   
  // 初始化类对象。
  SetupClass(*new_dex_file, *new_class_def, klass, class_loader.Get());
 
  // 如果初始化尚未完成，设置字符串类标记。
  if (UNLIKELY(!init_done_)) {
    if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
      klass->SetStringClass();
    }
  }
 
  // 为类对象加锁，确保线程安全。
  ObjectLock lock(self, klass);
  klass->SetClinitThreadId(self->GetTid()); // 设置类初始化的线程 ID。
  // 确保即使在错误的情况下也有一个有效的空接口表。
  klass->SetIfTable(GetClassRoot(this)->GetIfTable());
 
  // 将加载的类插入到已加载的类表中。
  ObjPtr existing = InsertClass(descriptor, klass.Get(), hash);
  if (existing != nullptr) {
    // 如果插入失败（可能由于并发），解决冲突。
    return sdc.Finish(EnsureResolved(self, descriptor, existing));
  }
 
  // 加载类的字段等信息。
  LoadClass(self, *new_dex_file, *new_class_def, klass);
  if (self->IsExceptionPending()) {
    VLOG(class_linker) << self->GetException()->Dump(); // 记录异常信息。
    // 如果发生异常，设置类状态为错误。
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }
 
  // 完成类加载，加载父类和接口。
  CHECK(!klass->IsLoaded()); // 确保类尚未加载。
  if (!LoadSuperAndInterfaces(klass, *new_dex_file)) {
    // 如果加载失败，设置类状态为错误。
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }
  CHECK(klass->IsLoaded()); // 确保类已加载。
 
  // 类加载完成，发布类加载事件。
  Runtime::Current()->GetRuntimeCallbacks()->ClassLoad(klass);
 
  // 链接类（如果必要）。
  CHECK(!klass->IsResolved()); // 确保类尚未解析。
  auto interfaces = hs.NewHandle>(nullptr);
 
  MutableHandle h_new_class = hs.NewHandle(nullptr);
  if (!LinkClass(self, descriptor, klass, interfaces, &h_new_class)) {
    // 如果链接失败，设置类状态为错误。
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }
  self->AssertNoPendingException(); // 确保没有挂起的异常。
  CHECK(h_new_class != nullptr) << descriptor; // 确保新类不为空。
  CHECK(h_new_class->IsResolved()) << descriptor << " " << h_new_class->GetStatus(); // 确保新类已解析。
 
  // 安装插桩代码（如果启用）。
  if (Runtime::Current()->GetInstrumentation()->EntryExitStubsInstalled()) {
    // 确保线程处于可运行状态，以避免在安装插桩代码时被挂起。
    DCHECK_EQ(self->GetState(), ThreadState::kRunnable);
    Runtime::Current()->GetInstrumentation()->InstallStubsForClass(h_new_class.Get());
  }
 
  /*
   * 发送 CLASS_PREPARE 事件到调试器。此时，类的静态字段已准备好，但未执行任何代码（初始化）。
   */
  Runtime::Current()->GetRuntimeCallbacks()->ClassPrepare(klass, h_new_class);
 
  // 通知 JIT 编译器新类型已加载（如果启用 JIT）。
  jit::Jit::NewTypeLoadedIfUsingJit(h_new_class.Get());
 
  return sdc.Finish(h_new_class); // 返回加载并解析的类。
} 
```

##### SetupClass

`SetupClass` 方法负责为一个类对象设置和初始化基本属性，包括类的状态、描述符、访问标志、类加载器等。这些设置确保了类能够在运行时正确地被加载和使用，为后续的类加载和链接过程提供了基础

路径为: art/runtime/class_linker.cc

```
void ClassLinker::SetupClass(const DexFile& dex_file,
                             const dex::ClassDef& dex_class_def,
                             Handle klass,
                             ObjPtr class_loader) {
  // 确保类对象 (klass) 不为空
  CHECK(klass != nullptr);
 
  // 确保类的 Dex 缓存 (DexCache) 已被设置
  CHECK(klass->GetDexCache() != nullptr);
 
  // 确保类的状态为 kNotReady，表示类尚未完全初始化
  CHECK_EQ(ClassStatus::kNotReady, klass->GetStatus());
 
  // 从 Dex 文件中获取类的描述符 (descriptor)
  const char* descriptor = dex_file.GetClassDescriptor(dex_class_def);
  CHECK(descriptor != nullptr);
 
  // 设置类的根对象为 Class 类的实例
  // Class 对象是 Java 类的元数据容器
  klass->SetClass(GetClassRoot(this));
 
  // 获取类的访问标志
  uint32_t access_flags = dex_class_def.GetJavaAccessFlags();
 
  // 确保访问标志只包含合法的 Java 标志
  CHECK_EQ(access_flags & ~kAccJavaFlagsMask, 0U);
 
  // 设置类的访问标志，这些标志来自 Dex 文件中的 ClassDef
  klass->SetAccessFlagsDuringLinking(access_flags);
 
  // 设置类的类加载器
  // 类加载器用于加载和验证类的字节码
  klass->SetClassLoader(class_loader);
 
  // 确保该类不是基本数据类型
  // 在设置类属性之前，必须确保它是一个普通类而不是基础数据类型
  DCHECK_EQ(klass->GetPrimitiveType(), Primitive::kPrimNot);
 
  // 将类的状态设置为 kIdx，表示类的索引阶段已完成
  mirror::Class::SetStatus(klass, ClassStatus::kIdx, nullptr);
 
  // 设置类在 Dex 文件中的 ClassDef 索引
  // Dex 文件中的 ClassDef 索引唯一标识了该类
  klass->SetDexClassDefIndex(dex_file.GetIndexForClassDef(dex_class_def));
 
  // 设置类的类型索引，这个索引在 Dex 文件中唯一标识该类
  klass->SetDexTypeIndex(dex_class_def.class_idx_);
} 
```

##### LoadClass

`LoadClass` 方法主要负责从 Dex 文件中加载类的各种数据，确保字段和方法被正确地分配、初始化和链接。它涉及了内存分配、字段和方法的加载、处理 AOT 编译的代码

这里可以深入一点点去了解

1. **创建 `ClassAccessor` 对象**

*   **功能**: `ClassAccessor` 用于访问类的各种数据，如字段、方法等。
    
*   实现
    
    ```
    ClassAccessor accessor(dex_file, dex_class_def, klass->IsBootStrapClassLoaded());
    
    ```
    

1.  **检查是否有类数据**

*   **功能**: 如果类数据不存在，直接返回，避免不必要的处理。
    
*   实现
    
    ```
    if (!accessor.HasClassData()) {
      return;
    }
    
    ```
    

1.  **防止线程挂起**

*   **功能**: 确保在处理字段和方法时线程不会被挂起，以避免数据丢失或不一致。
    
*   实现
    
    ```
    ScopedAssertNoThreadSuspension nts(__FUNCTION__);
    
    ```
    

1.  **分配字段数组的内存**

*   **功能**: 分配静态字段和实例字段的内存空间。
    
*   实现
    
    ```
    LinearAlloc* const allocator = GetAllocatorForClassLoader(klass->GetClassLoader());
    LengthPrefixedArray* sfields = AllocArtFieldArray(self, allocator, accessor.NumStaticFields());
    LengthPrefixedArray* ifields = AllocArtFieldArray(self, allocator, accessor.NumInstanceFields()); 
    ```
    

1.  **初始化字段计数器**

*   **功能**: 初始化静态字段和实例字段的计数器以及索引，用于跟踪和记录字段的状态。
    
*   实现
    
    ```
    size_t num_sfields = 0u;
    size_t num_ifields = 0u;
    uint32_t last_static_field_idx = 0u;
    uint32_t last_instance_field_idx = 0u;
    
    ```
    

1.  **处理 AOT 编译的类**

*   **功能**: 查找和处理 AOT (Ahead-Of-Time) 编译的类及其相关信息。
    
*   实现
    
    ```
    bool has_oat_class = false;
    const OatFile::OatClass oat_class = (runtime->IsStarted() && !runtime->IsAotCompiler())
        ? OatFile::FindOatClass(dex_file, klass->GetDexClassDefIndex(), &has_oat_class)
        : OatFile::OatClass::Invalid();
    OatClassCodeIterator occi(oat_class);
    
    ```
    

1.  **分配并设置方法数组**

*   **功能**: 分配内存来存储方法，并设置 `klass` 对象的指针以指向这些方法。
    
*   实现
    
    ```
    klass->SetMethodsPtr(
        AllocArtMethodArray(self, allocator, accessor.NumMethods()),
        accessor.NumDirectMethods(),
        accessor.NumVirtualMethods());
    
    ```
    

1.  **处理类的字段和方法**

*   **功能**: 遍历类的字段和方法，加载字段和方法数据，并处理可能的重复定义。
    
*   实现
    
    ```
    accessor.VisitFieldsAndMethods([&](const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
      // 处理静态字段
    }, [&](const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
      // 处理实例字段
    }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
      // 处理直接方法
    }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
      // 处理虚方法
    });
    
    ```
    

1.  **调整字段数组的大小**

*   **功能**: 确保字段数组的大小与实际的字段数量一致，并记录任何警告信息。
    
*   实现
    
    ```
    if (UNLIKELY(num_ifields + num_sfields != accessor.NumFields())) {
      LOG(WARNING) << "Duplicate fields in class " << klass->PrettyDescriptor()
          << " (unique static fields: " << num_sfields << "/" << accessor.NumStaticFields()
          << ", unique instance fields: " << num_ifields << "/" << accessor.NumInstanceFields()
          << ")";
      if (sfields != nullptr) {
        sfields->SetSize(num_sfields);
      }
      if (ifields != nullptr) {
        ifields->SetSize(num_ifields);
      }
    }
    
    ```
    

1.  **设置类的字段数组**

*   **功能**: 将分配并填充的字段数组设置到 `klass` 对象中。
    
*   实现
    
    ```
    klass->SetSFieldsPtr(sfields);
    DCHECK_EQ(klass->NumStaticFields(), num_sfields);
    klass->SetIFieldsPtr(ifields);
    DCHECK_EQ(klass->NumInstanceFields(), num_ifields);
    
    ```
    

1.  **标记字段写操作**

*   **功能**: 确保所有字段写操作都被正确标记，以便垃圾回收器能够捕捉到相关的根。
    
*   实现
    
    ```
    WriteBarrier::ForEveryFieldWrite(klass.Get());
    
    ```
    

1.  **恢复线程挂起状态**

*   **功能**: 恢复线程的挂起状态，使其能够正常进行上下文切换。
    
*   实现
    
    ```
    self->AllowThreadSuspension();
    
    ```
    

路径为: art/runtime/class_linker.cc

```
void ClassLinker::LoadClass(Thread* self,
                            const DexFile& dex_file,
                            const dex::ClassDef& dex_class_def,
                            Handle klass) {
  // 创建一个 ClassAccessor 对象来访问指定类的数据。
  // `parse_hiddenapi_class_data` 参数指示是否解析隐藏 API 数据。
  ClassAccessor accessor(dex_file,
                         dex_class_def,
                         /* parse_hiddenapi_class_data= */ klass->IsBootStrapClassLoaded());
 
  // 如果类数据不存在，则无需进一步处理，直接返回。
  if (!accessor.HasClassData()) {
    return;
  }
 
  // 获取当前运行时的实例，以便在方法中使用。
  Runtime* const runtime = Runtime::Current();
 
  {
    // 确保在字段和方法数组设置完成前，线程不会被挂起。
    // 这避免了在字段或方法处理过程中可能出现的数据丢失问题。
    ScopedAssertNoThreadSuspension nts(__FUNCTION__);
 
    // 获取分配器，以便为类的字段分配内存。
    // `GetAllocatorForClassLoader` 根据类加载器获取合适的线性分配器。
    LinearAlloc* const allocator = GetAllocatorForClassLoader(klass->GetClassLoader());
 
    // 为静态字段和实例字段分别分配内存。
    LengthPrefixedArray* sfields = AllocArtFieldArray(self,
                                                                allocator,
                                                                accessor.NumStaticFields());
    LengthPrefixedArray* ifields = AllocArtFieldArray(self,
                                                                allocator,
                                                                accessor.NumInstanceFields());
 
    // 初始化静态字段和实例字段的计数器。
    size_t num_sfields = 0u;
    size_t num_ifields = 0u;
    uint32_t last_static_field_idx = 0u; // 记录上一个处理的静态字段的索引。
    uint32_t last_instance_field_idx = 0u; // 记录上一个处理的实例字段的索引。
 
    // 检查是否存在 AOT 编译的类，如果存在则获取相关信息。
    bool has_oat_class = false;
    const OatFile::OatClass oat_class = (runtime->IsStarted() && !runtime->IsAotCompiler())
        ? OatFile::FindOatClass(dex_file, klass->GetDexClassDefIndex(), &has_oat_class)
        : OatFile::OatClass::Invalid();
 
    // 创建 OatClassCodeIterator 对象，用于遍历 AOT 编译的代码。
    OatClassCodeIterator occi(oat_class);
 
    // 为方法数组分配内存，并设置 `klass` 的方法指针。
    klass->SetMethodsPtr(
        AllocArtMethodArray(self, allocator, accessor.NumMethods()),
        accessor.NumDirectMethods(),
        accessor.NumVirtualMethods());
 
    // 初始化方法索引跟踪变量。
    size_t class_def_method_index = 0; // 当前类定义的方法索引。
    uint32_t last_dex_method_index = dex::kDexNoIndex; // 记录上一个处理的方法的 Dex 索引。
    size_t last_class_def_method_index = 0; // 记录上一个处理的方法的类定义索引。
 
    // 初始化直接方法和虚方法的注释迭代器。
    MethodAnnotationsIterator mai_direct(dex_file, dex_file.GetAnnotationsDirectory(dex_class_def));
    MethodAnnotationsIterator mai_virtual = mai_direct;
 
    // 获取 JIT 编译选项中的热点阈值。
    uint16_t hotness_threshold = runtime->GetJITOptions()->GetWarmupThreshold();
 
    // 使用访问器遍历类的字段和方法。
    // `VisitFieldsAndMethods` 方法允许定义自定义逻辑来处理字段和方法。
    accessor.VisitFieldsAndMethods([&](
        const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
          uint32_t field_idx = field.GetIndex(); // 获取字段的索引。
          DCHECK_GE(field_idx, last_static_field_idx); // 确保字段索引按顺序。
          if (num_sfields == 0 || LIKELY(field_idx > last_static_field_idx)) {
            // 加载静态字段到 `sfields` 数组中，并更新静态字段计数器和索引。
            LoadField(field, klass, &sfields->At(num_sfields));
            ++num_sfields;
            last_static_field_idx = field_idx;
          }
        }, [&](const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
          uint32_t field_idx = field.GetIndex(); // 获取字段的索引。
          DCHECK_GE(field_idx, last_instance_field_idx); // 确保字段索引按顺序。
          if (num_ifields == 0 || LIKELY(field_idx > last_instance_field_idx)) {
            // 加载实例字段到 `ifields` 数组中，并更新实例字段计数器和索引。
            LoadField(field, klass, &ifields->At(num_ifields));
            ++num_ifields;
            last_instance_field_idx = field_idx;
          }
        }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
          // 处理直接方法。
          ArtMethod* art_method = klass->GetDirectMethodUnchecked(class_def_method_index,
              image_pointer_size_);
          LoadMethod(dex_file, method, klass.Get(), &mai_direct, art_method);
          LinkCode(art_method, class_def_method_index, &occi); // 链接方法的代码。
          uint32_t it_method_index = method.GetIndex(); // 获取方法的索引。
          if (last_dex_method_index == it_method_index) {
            // 处理方法重复定义的情况。
            art_method->SetMethodIndex(last_class_def_method_index);
          } else {
            // 设置方法索引，并更新索引跟踪变量。
            art_method->SetMethodIndex(class_def_method_index);
            last_dex_method_index = it_method_index;
            last_class_def_method_index = class_def_method_index;
          }
          art_method->ResetCounter(hotness_threshold); // 重置方法的计数器。
          ++class_def_method_index;
        }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
          // 处理虚方法。
          ArtMethod* art_method = klass->GetVirtualMethodUnchecked(
              class_def_method_index - accessor.NumDirectMethods(),
              image_pointer_size_);
          art_method->ResetCounter(hotness_threshold); // 重置方法的计数器。
          LoadMethod(dex_file, method, klass.Get(), &mai_virtual, art_method);
          LinkCode(art_method, class_def_method_index, &occi); // 链接方法的代码。
          ++class_def_method_index;
        });
 
    // 检查实际字段数量与预期字段数量是否一致。
    if (UNLIKELY(num_ifields + num_sfields != accessor.NumFields())) {
      // 记录警告信息，指出字段重复的情况。
      LOG(WARNING) << "Duplicate fields in class " << klass->PrettyDescriptor()
          << " (unique static fields: " << num_sfields << "/" << accessor.NumStaticFields()
          << ", unique instance fields: " << num_ifields << "/" << accessor.NumInstanceFields()
          << ")";
      // 根据实际字段数量调整数组大小。
      if (sfields != nullptr) {
        sfields->SetSize(num_sfields);
      }
      if (ifields != nullptr) {
        ifields->SetSize(num_ifields);
      }
    }
 
    // 设置类的静态字段和实例字段数组。
    klass->SetSFieldsPtr(sfields);
    DCHECK_EQ(klass->NumStaticFields(), num_sfields); // 确保静态字段数目匹配。
    klass->SetIFieldsPtr(ifields);
    DCHECK_EQ(klass->NumInstanceFields(), num_ifields); // 确保实例字段数目匹配。
  }
 
  // 确保所有字段写操作都正确标记，以便记忆集可以捕捉到本地根。
  WriteBarrier::ForEveryFieldWrite(klass.Get());
 
  // 恢复线程挂起状态，使线程能够正常挂起。
  self->AllowThreadSuspension();
} 
```

至此类的加载流程差不多分析完了 有错误分析的地方请大佬指出来 QWQ

[[培训] 科锐软件逆向 50 期预科班报名即将截止，速来！！！ 50 期正式班报名火爆招生中！！！](https://mp.weixin.qq.com/s/HFghXQRTiTlk6oRKGotpHA)

[#基础理论](forum-161-1-117.htm)