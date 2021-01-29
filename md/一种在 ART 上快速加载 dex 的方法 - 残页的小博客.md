> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.canyie.top](https://blog.canyie.top/2020/02/15/fast-load-dex-on-art-runtime/)

在国内的大环境下，Android 上插件化 / 热修复等技术百花齐放，而这一切都基于代码的动态加载。Android 提供了一个 DexClassLoader。用这个 API 能成功加载 dex，但有一个比较严重的问题：Android Q 以下，当这个 dex 被加载时，如果没有已经生成的 oat，则会执行一次 dex2oat 把这个 dex 编译为 oat，导致第一次加载 dex 会非常非常慢。个人认为这样的设计是非常不合理的，虽然转换成 oat 之后执行会很快，但完全可以让用户以解释器模式先愉快的用着，dex2oat 放另一个线程执行多好。Android 8.0 上谷歌还提供了一个 InMemoryDexClassLoader，而以前的 Android 版本，就要开发者自己想办法了……

[](#源码分析 "源码分析")源码分析
--------------------

注：为了说明此方法能在较低版本的 ART 上运行，本文分析的源码是 [Android 5.0](http://aospxref.com/android-5.0.0_r7.0.1/) 的源码，之后的 Android 版本里 OpenDexFileFromOat 方法搬到了 OatFileManager 里，而调用 dex2oat 的部分则重构到了 OatFileAssistant 中，大致逻辑相同，感兴趣的可以自己去看看；至于 Android 4.4，简单扫了一下源码似乎是生成 oat 失败就会直接抛一个 IOException 拒绝加载，emmm……

我们在 Java 代码里用`new DexClassLoader()`的方式加载 dex，最后会调用到`DexFile.openDexFileNative`中，这个函数的[实现](http://aospxref.com/android-5.0.0_r7.0.1/xref/art/runtime/native/dalvik_system_DexFile.cc#101)是这样的：

```
static jlong DexFile_openDexFileNative(JNIEnv* env, jclass, jstring javaSourceName, jstring javaOutputName, jint) {
  ScopedUtfChars sourceName(env, javaSourceName);
  if (sourceName.c_str() == NULL) {
    return 0;
  }
  NullableScopedUtfChars outputName(env, javaOutputName);
  if (env->ExceptionCheck()) {
    return 0;
  }

  ClassLinker* linker = Runtime::Current()->GetClassLinker();
  std::unique_ptr<std::vector<const DexFile*>> dex_files(new std::vector<const DexFile*>());
  std::vector<std::string> error_msgs;

  bool success = linker->OpenDexFilesFromOat(sourceName.c_str(), outputName.c_str(), &error_msgs,
                                             dex_files.get());

  if (success || !dex_files->empty()) {
    
    
    return static_cast<jlong>(reinterpret_cast<uintptr_t>(dex_files.release()));
  } else {
      
  }
}
```

这里的注释很有意思，如果返回 false（生成 oat 失败），但是有被成功加载的 dex，那么还是应该当做成功。  
可以看出具体实现在 ClassLinker 中的 OpenDexFilesFromOat 里，我们点进去看看：

```
bool ClassLinker::OpenDexFilesFromOat(const char* dex_location, const char* oat_location,
                                      std::vector<std::string>* error_msgs,
                                      std::vector<const DexFile*>* dex_files) {
  
  
  bool needs_registering = false;

  const OatFile::OatDexFile* oat_dex_file = FindOpenedOatDexFile(oat_location, dex_location,
                                                                 dex_location_checksum_pointer);
  std::unique_ptr<const OatFile> open_oat_file(
      oat_dex_file != nullptr ? oat_dex_file->GetOatFile() : nullptr);

  

  
  
  
  
  
  
  ScopedFlock scoped_flock;

  if (open_oat_file.get() == nullptr) { 
    if (oat_location != nullptr) {
      std::string error_msg;

      
      if (!scoped_flock.Init(oat_location, &error_msg)) {
        error_msgs->push_back(error_msg);
        return false;
      }

      
      open_oat_file.reset(FindOatFileInOatLocationForDexFile(dex_location, dex_location_checksum,
                                                             oat_location, &error_msg));

      if (open_oat_file.get() == nullptr) {
        std::string compound_msg = StringPrintf("Failed to find dex file '%s' in oat location '%s': %s",
                                                dex_location, oat_location, error_msg.c_str());
        VLOG(class_linker) << compound_msg;
        error_msgs->push_back(compound_msg);
      }
    } else {
      
      bool obsolete_file_cleanup_failed;
      open_oat_file.reset(FindOatFileContainingDexFileFromDexLocation(dex_location,
                                                                      dex_location_checksum_pointer,
                                                                      kRuntimeISA, error_msgs,
                                                                      &obsolete_file_cleanup_failed));
      
      
      
      
      if (obsolete_file_cleanup_failed) {
        return false;
      }
    }
    needs_registering = true;
  }

  
  
  bool success = LoadMultiDexFilesFromOatFile(open_oat_file.get(), dex_location,
                                              dex_location_checksum_pointer,
                                              false, error_msgs, dex_files);
  if (success) {
      
  } else {
    if (needs_registering) {
      
      open_oat_file.reset();
    } else {
      open_oat_file.release();  
    }
  }

  

  
  std::string cache_location;
  if (oat_location == nullptr) {
    
    const std::string dalvik_cache(GetDalvikCacheOrDie(GetInstructionSetString(kRuntimeISA)));
    cache_location = GetDalvikCacheFilenameOrDie(dex_location, dalvik_cache.c_str());
    oat_location = cache_location.c_str();
  }

  bool has_flock = true;
  
  if (!scoped_flock.HasFile()) {
    std::string error_msg;
    if (!scoped_flock.Init(oat_location, &error_msg)) {
      error_msgs->push_back(error_msg);
      has_flock = false;
    }
  }

  if (Runtime::Current()->IsDex2OatEnabled() && has_flock && scoped_flock.HasFile()) {
    
    open_oat_file.reset(CreateOatFileForDexLocation(dex_location, scoped_flock.GetFile()->Fd(),
                                                    oat_location, error_msgs));
  }

  
  if (open_oat_file.get() == nullptr) { 
    std::string error_msg;
    
    DexFile::Open(dex_location, dex_location, &error_msg, dex_files);
    error_msgs->push_back(error_msg);
    return false;
  }
  
}
```

这个函数比较长，所以做了一点精简。  
我们在这里看到了一点端倪，这个函数做了这些事情：

1.  检查我们是否已经有一个打开了的 oat
2.  如果没有，那么检查 oat 缓存目录（创建 DexClassLoader 时传入的第二个参数）是否已经有了一个 oat，并且检查这个 oat 的有效性
3.  如果没有或者这个 oat 是无效的，那么生成一个 oat 文件

我们首次加载 dex 时，肯定没有有效的 oat，最后会生成一个新的 oat：

```
if (Runtime::Current()->IsDex2OatEnabled() && has_flock && scoped_flock.HasFile()) {
  
  open_oat_file.reset(CreateOatFileForDexLocation(dex_location, scoped_flock.GetFile()->Fd(),
                                                  oat_location, error_msgs));
}
```

这里有一个 if 判断，直接决定是否进行 dex2oat，我们看看能不能通过各种手段让这个判断不成立。

[](#禁用dex2oat "禁用dex2oat")禁用 dex2oat
------------------------------------

### [](#第一招：修改Runtime中的变量 "第一招：修改Runtime中的变量")第一招：修改 Runtime 中的变量

这个 if 判断里，第一个条件就是`Runtime::Current()->IsDex2OatEnabled()`，如果返回 false，那么就不会生成 oat。这个函数的实现如下：

```
bool IsDex2OatEnabled() const {
  return dex2oat_enabled_ && IsImageDex2OatEnabled();
}

bool IsImageDex2OatEnabled() const {
  return image_dex2oat_enabled_;
}
```

`dex2oat_enabled_`与`image_dex2oat_enabled_`都是 Runtime 对象中的成员变量，而 Runtime 可以通过 JavaVM 获取，**所以我们只需要修改这个值就能禁用 dex2oat**。已经有其他人实现了这一步，具体可以看看[这篇博客](https://fucknmb.com/2018/12/30/art-dex2oat%E5%8A%A0%E8%BD%BD%E5%8A%A0%E9%80%9F%E6%B5%85%E6%9E%90/)。  
然而事情真的会这么简单吗？  
查看源码发现 **Runtime 是一个炒鸡大的结构体**，Android 里有什么东西都往这扔，你几乎可以从 Runtime 对象上直接或间接获取到任何东西，然而也正是因为 Runtime 太大了，使得没有什么好的办法获取里面的值。

让我们看看还有没有其他方法：

### [](#第二招：使用PathClassLoader "第二招：使用PathClassLoader")第二招：使用 PathClassLoader

我们可以看见，在 if 判断里，还有两个条件：`has_flock`和`scoped_flock.HasFile()`，让我们看看是否可以让这两个条件不成立。  
`has_flock`的赋值：

```
bool has_flock = true;

if (!scoped_flock.HasFile()) {
  std::string error_msg;
  if (!scoped_flock.Init(oat_location, &error_msg)) {
    error_msgs->push_back(error_msg);
    has_flock = false;
  }
}
```

又是 scoped_flock，看看在上面 scoped_flock 可能在哪里被初始化：

```
ScopedFlock scoped_flock;
if (open_oat_file.get() == nullptr) { 
  if (oat_location != nullptr) {
    std::string error_msg;

    
    if (!scoped_flock.Init(oat_location, &error_msg)) {
      error_msgs->push_back(error_msg);
      return false;
    }
    
  }
}
```

看看 ScopedFlock 的 Init 方法：

```
bool ScopedFlock::Init(const char* filename, std::string* error_msg) {
  while (true) {
    file_.reset(OS::OpenFileWithFlags(filename, O_CREAT | O_RDWR));
    if (file_.get() == NULL) {
      *error_msg = StringPrintf("Failed to open file '%s': %s", filename, strerror(errno));
      return false;
    }
    
  }
}
```

可以看见，会打开这个文件，flags 为 O_CREAT | O_RDWR，那我们只需要设置 oat_location 为不可写的路径，就能让 ScopedFlock::Init 返回 false。不过我们要注意的是，如果 oat_location 不为 null 并且无法使用，那在上面的一个判断里就会直接返回 false。怎么办？

是时候请出我们的主角`PathClassLoader`了！

PathClassLoader 作为 DexClassLoader 的兄弟~（也可能是姐妹？）~，受到的待遇与 DexClassLoader 截然不同：网上讲解动态加载 dex 的文章几乎都只讲 DexClassLoader，而对于 PathClassLoader 则是一笔带过：“PathClassLoader 只能加载系统中已经安装过的 apk”。  
然而事实真的是这样吗？或许 Android 5.0 以前是，但 Android 5.0 时就已经可以加载外部 dex 了，今天我要为 PathClassLoader 正名！  
让我们来对比一下 DexClassLoader 和 PathClassLoader 的源码。  
DexClassLoader：

```
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

对，你没看错，有效代码就这么点。  
让我们再看看 PathClassLoader 的源码：

```
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

实际上所以实现代码都在 BaseDexClassLoader 中，DexClassLoader 和 PathClassLoader 都调用了同一个构造函数：

```
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
}
```

注意第二个参数，optimizedDirectory，**DexClassLoader 传入的是 new File(optimizedDirectory)，而 PathClassLoader 传入的是 null。**记住这一点。  
这两种情况最后都会调用到 DexFile.openDexFileNative 中

```
private static native long openDexFileNative(String sourceName, String outputName, int flags);
```

如果是 PathClassLoader，outputName 为 null，会进入这个 if 分支中：

```
std::string cache_location;
if (oat_location == nullptr) {
  
  const std::string dalvik_cache(GetDalvikCacheOrDie(GetInstructionSetString(kRuntimeISA)));
  cache_location = GetDalvikCacheFilenameOrDie(dex_location, dalvik_cache.c_str());
  oat_location = cache_location.c_str();
}
```

这里**会把 oat_location 设置成 / data/dalvik-cache / 下的路径**，接下来因为**我们根本没有对 dalvik-cache 的写入权限**，所以无法打开 fd，然后就会走到这里**直接加载原始 dex**：

```
if (open_oat_file.get() == nullptr) {
  std::string error_msg;
  
  DexFile::Open(dex_location, dex_location, &error_msg, dex_files);
  error_msgs->push_back(error_msg);
  return false;
}
```

至此整个逻辑已经明朗，通过 PathClassLoader 加载会把 oat 输出路径设置成 / data/dalvik-cache / 下，然后因为我们没有对 dalvik-cache 的写入权限，所以无法打开 fd，之后会直接加载原始 dex，不会进行 dex2oat。  
（注：本文分析源码是 [Android 5.0](http://aospxref.com/android-5.0.0_r7.0.1/)，**在 [Android 8.1](http://aospxref.com/android-8.1.0_r65/) 时源码有改动**，就算是 DexClassLoader 也会把 optimizedDirectory 设置成 null，输出的 oat 在 dex 的父目录 / oat / 下，所以无法通过 PathClassLoader 快速加载 dex，但在 8.1 时已经有 InMemoryDexClassLoader 了，直接通过 InMemoryDexClassLoader 加载就好了。

简单做了个小测试，在我的 AVD（Android 7.1.1）上，用 DexClassLoader 加载 75M 的 qq apk 用了近 80 秒，并生成了一个 313M 的 oat，而 PathClassLoader 用时稳定在 2 秒左右，emmm……

看起来我们已经有一个比较好的办法禁用 dex2oat 了，不过需要修改源码没法直接全局禁用，修改 Runtime 风险又太大，让我们看看还有没有其他方法。

### [](#第三招：hook-execv "第三招：hook execv")第三招：hook execv

到了这里，上面那个判断肯定会成立了，似乎进行 dex2oat 已成定局？我们继续看 CreateOatFileForDexLocation。

```
const OatFile* ClassLinker::CreateOatFileForDexLocation(const char* dex_location,
                                                        int fd, const char* oat_location,
                                                        std::vector<std::string>* error_msgs) {
  
  VLOG(class_linker) << "Generating oat file " << oat_location << " for " << dex_location;
  std::string error_msg;
  if (!GenerateOatFile(dex_location, fd, oat_location, &error_msg)) {
    CHECK(!error_msg.empty());
    error_msgs->push_back(error_msg);
    return nullptr;
  }
  std::unique_ptr<OatFile> oat_file(OatFile::Open(oat_location, oat_location, nullptr,
                                            !Runtime::Current()->IsCompiler(),
                                            &error_msg));
  if (oat_file.get() == nullptr) {
    std::string compound_msg = StringPrintf("\nFailed to open generated oat file '%s': %s",
                                            oat_location, error_msg.c_str());
    error_msgs->push_back(compound_msg);
    return nullptr;
  }

  return oat_file.release();
}
```

GenerateOatFile 是核心逻辑，这个函数大部分都是我们不关心的配置 dex2oat 参数就不贴出来了，最后会 fork 出一个新进程，然后在子进程里执行 execv() 调用 dex2oat。  
看起来我们必然要执行 dex2oat 了？别慌，还有办法。虽然没有直接的开关去阻止 dex2oat，但**我们还有 hook 大法**！生成 oat 最后是通过 execv 调用 dex2oat 进行的，所以我们可以 **hook 掉 execv 函数，如果是执行 dex2oat 那么直接让这个进程退出即可**！Lody 大神的早期作品 [TurboDex](https://github.com/asLody/TurboDex) 就是这样实现的。不过这个项目其实还可以优化一下：TurboDex 是使用的 Substrate 进行 hook，这是一个 inline hook 库，而 execv 是来自 libc.so 的导出符号，其实直接通过 GOT Hook 就能 hook 到，没有必要去用 inline hook，反而增加 crash 风险。

[](#总结 "总结")总结
--------------

本来我只是为了研究 DexClassLoader 与 PathClassLoader 的区别的，网上的文章和实验的结果完全不一样，结果意外发现一个快速加载 dex 的方法，就写出来了 :)  
这个故事告诉我们，没事多看源码（手动滑稽）  
另外个人建议，快速加载 dex 之后后台可以开一个线程单独进行 dex2oat，具体可以参考 [ArtDexOptimizer](https://github.com/canyie/DreamlandManager/blob/master/app/src/main/java/com/canyie/dreamland/manager/utils/ArtDexOptimizer.java)，下次启动的时候如果完成了可以直接用生成好的 oat 文件，毕竟用 oat 比直接加载 dex 快得多，而且更稳定~