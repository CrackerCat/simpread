> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Q3Hwttdr1-CUgiuMzl7hBA)

**一、前言**

  

      在安卓 **app** 加固、插件化开发过程中,**DexClassLoader** 用的比较多。像现在的各种类 **Xposed** 的框架在加载外部插件的时候，也使用了 **DexClassLoader** 来进行动态加载模块，然后反射调用插件配置的入口方法。

      在 **8.0** 系统以后，新增了一个 **InMemoryDexClassLoader。**和 **DexClassLoader** 都是继承于 **BaseDexClassLoader**。本章节只分析 **DexClassLoader**，下一章分析 **InMemoryDexClassLoader**。

**二、加载流程分析**

 **1.DexClassLoader 类分析**

 **DexClassLoader** 文件路径如下:

```
libcore\dalvik\src\main\java\dalvik\system\DexClassLoader.java

```

      该类的构造函数如下:  

```
public class DexClassLoader extends BaseDexClassLoader {
    //构造函数，第一个参数dex文件路径
    //第二个参数编译优化存放路径
    //第三个参数so库的搜索路径
    //第四个参数为父classLoader
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}

```

     由以上构造函数分析可以知道调用了父类的构造函数。父类为 **BaseDexClassLoader**。          

   **2.**BaseDexClassLoader** 类分析**

 BaseDexClassLoader 类文件路径如下:

```
libcore\dalvik\src\main\java\dalvik\system\BaseDexClassLoader.java

```

      在改该类中通过构造函数追踪，追踪调用了的构造函数链如下:  

```
//构造函数调用1
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        this(dexPath, librarySearchPath, parent, null, false);
    }
 //构造函数调用2
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent, boolean isTrusted) {
        this(dexPath, librarySearchPath, parent, null, isTrusted);
    }
//构造函数调用3
    public BaseDexClassLoader(String dexPath,
            String librarySearchPath, ClassLoader parent, ClassLoader[] libraries) {
        this(dexPath, librarySearchPath, parent, libraries, false);
    }
    //构造函数调用4
    public BaseDexClassLoader(String dexPath,
            String librarySearchPath, ClassLoader parent, ClassLoader[] sharedLibraryLoaders,
            boolean isTrusted) {
        super(parent);
        // Setup shared libraries before creating the path list. ART relies on the class loader
        // hierarchy being finalized before loading dex files.
        this.sharedLibraryLoaders = sharedLibraryLoaders == null
                ? null
                : Arrays.copyOf(sharedLibraryLoaders, sharedLibraryLoaders.length);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
        if (reporter != null) {
            reportClassLoaderChain();
        }
    }

```

 在以上构造函数 4 中，使用了 **DexPathList** 类传入 **dexPath** 构造对象。下面将分析 **DexPathList** 内部逻辑。

 **3.DexPathList 类流程分析**

   **DexPathList** 类文件路径:

```
libcore\dalvik\src\main\java\dalvik\system\DexPathList.java

```

 该类中相关的构造函数如下:

```
public DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory) {
        this(definingContext, dexPath, librarySearchPath, optimizedDirectory, false);
    }
DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }
        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }
        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }
            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }
        this.definingContext = definingContext;
        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // save dexPath for BaseDexClassLoader
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);
        // Native libraries may exist in both the system and
        // application library paths, and we use this search order:
        //
        //   1. This class loader's library path for application libraries (librarySearchPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        //
        // This order was reversed prior to Gingerbread; see http://b/2933456.
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        this.nativeLibraryPathElements = makePathElements(getAllNativeLibraryDirectories());
        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
    }

```

    在以上构造函数中，关键部分为 **makeDexElements** 函数调用。该函数实现如下:

```
 private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
            List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
      Element[] elements = new Element[files.size()];
      int elementsPos = 0;
      /*
       * Open all files and load the (direct or contained) dex files up front.
       */
      for (File file : files) {
          if (file.isDirectory()) {
              // We support directories for looking up resources. Looking up resources in
              // directories is useful for running libcore tests.
              elements[elementsPos++] = new Element(file);
          } else if (file.isFile()) {
              String name = file.getName();
              DexFile dex = null;
              if (name.endsWith(DEX_SUFFIX)) {
                  // Raw dex file (not inside a zip/jar).
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                      if (dex != null) {
                          elements[elementsPos++] = new Element(dex, null);
                      }
                  } catch (IOException suppressed) {
                      System.logE("Unable to load dex file: " + file, suppressed);
                      suppressedExceptions.add(suppressed);
                  }
              } else {
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                  } catch (IOException suppressed) {
                      /*
                       * IOException might get thrown "legitimately" by the DexFile constructor if
                       * the zip file turns out to be resource-only (that is, no classes.dex file
                       * in it).
                       * Let dex == null and hang on to the exception to add to the tea-leaves for
                       * when findClass returns null.
                       */
                      suppressedExceptions.add(suppressed);
                  }
                  if (dex == null) {
                      elements[elementsPos++] = new Element(file);
                  } else {
                      elements[elementsPos++] = new Element(dex, file);
                  }
              }
              if (dex != null && isTrusted) {
                dex.setTrusted();
              }
          } else {
              System.logW("ClassLoader referenced unknown path: " + file);
          }
      }
      if (elementsPos != elements.length) {
          elements = Arrays.copyOf(elements, elementsPos);
      }
      return elements;
    }

```

 以上函数调用了 **loadDexFile**。**loadDexFile** 函数实现如下:

```
  private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader,
                                       Element[] elements)
            throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file, loader, elements);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
        }
    }

```

       该方法会根据传入的 **optimizedDirectory** 是否为 **null**，进行不同的 **DexFile** 构造。在实际开发过程中，一般 **optimizedDirectory** 都不为 **null**，所以我们追踪 **DexFile.loadDex** 调用情况。

   **4.DexFile 内部流程分析**

  **DexFile** 文件路径如下:

```
libcore\dalvik\src\main\java\dalvik\system\DexFile.java

```

        该类中 **loadDex** 方法实现如下:

```
static DexFile loadDex(String sourcePathName, String outputPathName,
        int flags, ClassLoader loader, DexPathList.Element[] elements) throws IOException {
        /*
         * TODO: we may want to cache previously-opened DexFile objects.
         * The cache would be synchronized with close().  This would help
         * us avoid mapping the same DEX more than once when an app
         * decided to open it multiple times.  In practice this may not
         * be a real issue.
         */
        return new DexFile(sourcePathName, outputPathName, flags, loader, elements);
    }

```

 最终还是调用了 **DexFile** 的构造函数，根据参数类型找到 **DexFile** 的构造函数如下:

```
 private DexFile(String sourceName, String outputName, int flags, ClassLoader loader,
            DexPathList.Element[] elements) throws IOException {
        if (outputName != null) {
            try {
                String parent = new File(outputName).getParent();
                if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                    throw new IllegalArgumentException("Optimized data directory " + parent
                            + " is not owned by the current user. Shared storage cannot protect"
                            + " your application from code injection attacks.");
                }
            } catch (ErrnoException ignored) {
                // assume we'll fail with a more contextual error later
            }
        }
        mCookie = openDexFile(sourceName, outputName, flags, loader, elements);
        mInternalCookie = mCookie;
        mFileName = sourceName;
        //System.out.println("DEX FILE cookie is " + mCookie + " source + outputName);
    }

```

 该构造函数里面调用了 **openDexFile** 方法，该方法的实现如下:

```
 private static Object openDexFile(String sourceName, String outputName, int flags,
            ClassLoader loader, DexPathList.Element[] elements) throws IOException {
        // Use absolute paths to enable the use of relative paths when testing on host.
        return openDexFileNative(new File(sourceName).getAbsolutePath(),
                                 (outputName == null)
                                     ? null
                                     : new File(outputName).getAbsolutePath(),
                                 flags,
                                 loader,
                                 elements);
    }

```

 **openDexFile** 方法内部调用了 **openDexFileNative** 方法，**openDexFileNative** 实现如下:  

```
 private static native Object openDexFileNative(String sourceName, String outputName, int flags,
            ClassLoader loader, DexPathList.Element[] elements);

```

 **openDexFileNative** 是一个 **native** 方法，具体实现在 **DexFile** 类的 **jni** 代码中。在源码中搜索定位找到对应的 **jni** 实现文件为 **dalvik_system_DexFile.cc。**

 **5.**dalvik_system_DexFile.cc** 中内部流程分析**

 **dalvik_system_DexFile.cc** 文件路径如下:

```
art\runtime\native\dalvik_system_DexFile.cc

```

      该文件中找到 **java** 层 **native** 方法 **openDexFileNative** 对应的 **jni** 实现方法为 **DexFile_openDexFileNative** 方法。该方法实现如下:

```
static jobject DexFile_openDexFileNative(JNIEnv* env,
                                         jclass,
                                         jstring javaSourceName,
                                         jstring javaOutputName ATTRIBUTE_UNUSED,
                                         jint flags ATTRIBUTE_UNUSED,
                                         jobject class_loader,
                                         jobjectArray dex_elements) {
  ScopedUtfChars sourceName(env, javaSourceName);
  if (sourceName.c_str() == nullptr) {
    return nullptr;
  }
  std::vector<std::string> error_msgs;
  const OatFile* oat_file = nullptr;
  std::vector<std::unique_ptr<const DexFile>> dex_files =
      Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat(sourceName.c_str(),
                                                                  class_loader,
                                                                  dex_elements,
                                                                  /*out*/ &oat_file,
                                                                  /*out*/ &error_msgs);
  return CreateCookieFromOatFileManagerResult(env, dex_files, oat_file, error_msgs);
}

```

        该方法中关键的调用为:

```
Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat

```

     在源码中找到 **OpenDexFilesFromOat** 实现文件为 **oat_file_manager**.cc。

 **6****.****oat_file_manager.cc** **中内部流程分析**

         **oat_file_manager**.cpp 源文件路径如下:

```
art\runtime\oat_file_manager.cpp

```

 **该文件中 OpenDexFilesFromOat 实现如下:**

```
std::vector<std::unique_ptr<const DexFile>> OatFileManager::OpenDexFilesFromOat(
    const char* dex_location,
    jobject class_loader,
    jobjectArray dex_elements,
    const OatFile** out_oat_file,
    std::vector<std::string>* error_msgs) {
  ScopedTrace trace(__FUNCTION__);
  CHECK(dex_location != nullptr);
  CHECK(error_msgs != nullptr);
  // Verify we aren't holding the mutator lock, which could starve GC if we
  // have to generate or relocate an oat file.
  Thread* const self = Thread::Current();
  Locks::mutator_lock_->AssertNotHeld(self);
  Runtime* const runtime = Runtime::Current();
  std::unique_ptr<ClassLoaderContext> context;
  // If the class_loader is null there's not much we can do. This happens if a dex files is loaded
  // directly with DexFile APIs instead of using class loaders.
  if (class_loader == nullptr) {
    LOG(WARNING) << "Opening an oat file without a class loader. "
                 << "Are you using the deprecated DexFile APIs?";
    context = nullptr;
  } else {
    context = ClassLoaderContext::CreateContextForClassLoader(class_loader, dex_elements);
  }
  OatFileAssistant oat_file_assistant(dex_location,
                                      kRuntimeISA,
                                      !runtime->IsAotCompiler(),
                                      only_use_system_oat_files_);
  // Get the oat file on disk.
  std::unique_ptr<const OatFile> oat_file(oat_file_assistant.GetBestOatFile().release());
  VLOG(oat) << "OatFileAssistant(" << dex_location << ").GetBestOatFile()="
            << reinterpret_cast<uintptr_t>(oat_file.get())
            << " (executable=" << (oat_file != nullptr ? oat_file->IsExecutable() : false) << ")";
  const OatFile* source_oat_file = nullptr;
  CheckCollisionResult check_collision_result = CheckCollisionResult::kPerformedHasCollisions;
  std::string error_msg;
  if ((class_loader != nullptr || dex_elements != nullptr) && oat_file != nullptr) {
    // Prevent oat files from being loaded if no class_loader or dex_elements are provided.
    // This can happen when the deprecated DexFile.<init>(String) is called directly, and it
    // could load oat files without checking the classpath, which would be incorrect.
    // Take the file only if it has no collisions, or we must take it because of preopting.
    check_collision_result = CheckCollision(oat_file.get(), context.get(), /*out*/ &error_msg);
    bool accept_oat_file = AcceptOatFile(check_collision_result);
    if (!accept_oat_file) {
      // Failed the collision check. Print warning.
      if (runtime->IsDexFileFallbackEnabled()) {
        if (!oat_file_assistant.HasOriginalDexFiles()) {
          // We need to fallback but don't have original dex files. We have to
          // fallback to opening the existing oat file. This is potentially
          // unsafe so we warn about it.
          accept_oat_file = true;
          LOG(WARNING) << "Dex location " << dex_location << " does not seem to include dex file. "
                       << "Allow oat file use. This is potentially dangerous.";
        } else {
          // We have to fallback and found original dex files - extract them from an APK.
          // Also warn about this operation because it's potentially wasteful.
          LOG(WARNING) << "Found duplicate classes, falling back to extracting from APK : "
                       << dex_location;
          LOG(WARNING) << "NOTE: This wastes RAM and hurts startup performance.";
        }
      } else {
        // TODO: We should remove this. The fact that we're here implies -Xno-dex-file-fallback
        // was set, which means that we should never fallback. If we don't have original dex
        // files, we should just fail resolution as the flag intended.
        if (!oat_file_assistant.HasOriginalDexFiles()) {
          accept_oat_file = true;
        }
        LOG(WARNING) << "Found duplicate classes, dex-file-fallback disabled, will be failing to "
                        " load classes for " << dex_location;
      }
      LOG(WARNING) << error_msg;
    }
    if (accept_oat_file) {
      VLOG(class_linker) << "Registering " << oat_file->GetLocation();
      source_oat_file = RegisterOatFile(std::move(oat_file));
      *out_oat_file = source_oat_file;
    }
  }
  std::vector<std::unique_ptr<const DexFile>> dex_files;
  // Load the dex files from the oat file.
  if (source_oat_file != nullptr) {
    bool added_image_space = false;
    if (source_oat_file->IsExecutable()) {
      ScopedTrace app_image_timing("AppImage:Loading");
      // We need to throw away the image space if we are debuggable but the oat-file source of the
      // image is not otherwise we might get classes with inlined methods or other such things.
      std::unique_ptr<gc::space::ImageSpace> image_space;
      if (ShouldLoadAppImage(check_collision_result,
                             source_oat_file,
                             context.get(),
                             &error_msg)) {
        image_space = oat_file_assistant.OpenImageSpace(source_oat_file);
      }
      if (image_space != nullptr) {
        ScopedObjectAccess soa(self);
        StackHandleScope<1> hs(self);
        Handle<mirror::ClassLoader> h_loader(
            hs.NewHandle(soa.Decode<mirror::ClassLoader>(class_loader)));
        // Can not load app image without class loader.
        if (h_loader != nullptr) {
          std::string temp_error_msg;
          // Add image space has a race condition since other threads could be reading from the
          // spaces array.
          {
            ScopedThreadSuspension sts(self, kSuspended);
            gc::ScopedGCCriticalSection gcs(self,
                                            gc::kGcCauseAddRemoveAppImageSpace,
                                            gc::kCollectorTypeAddRemoveAppImageSpace);
            ScopedSuspendAll ssa("Add image space");
            runtime->GetHeap()->AddSpace(image_space.get());
          }
          {
            ScopedTrace trace2(StringPrintf("Adding image space for location %s", dex_location));
            added_image_space = runtime->GetClassLinker()->AddImageSpace(image_space.get(),
                                                                         h_loader,
                                                                         dex_elements,
                                                                         dex_location,
                                                                         /*out*/&dex_files,
                                                                         /*out*/&temp_error_msg);
          }
          if (added_image_space) {
            // Successfully added image space to heap, release the map so that it does not get
            // freed.
            image_space.release();  // NOLINT b/117926937
            // Register for tracking.
            for (const auto& dex_file : dex_files) {
              dex::tracking::RegisterDexFile(dex_file.get());
            }
          } else {
            LOG(INFO) << "Failed to add image file " << temp_error_msg;
            dex_files.clear();
            {
              ScopedThreadSuspension sts(self, kSuspended);
              gc::ScopedGCCriticalSection gcs(self,
                                              gc::kGcCauseAddRemoveAppImageSpace,
                                              gc::kCollectorTypeAddRemoveAppImageSpace);
              ScopedSuspendAll ssa("Remove image space");
              runtime->GetHeap()->RemoveSpace(image_space.get());
            }
            // Non-fatal, don't update error_msg.
          }
        }
      }
    }
    if (!added_image_space) {
      DCHECK(dex_files.empty());
      dex_files = oat_file_assistant.LoadDexFiles(*source_oat_file, dex_location);
      // Register for tracking.
      for (const auto& dex_file : dex_files) {
        dex::tracking::RegisterDexFile(dex_file.get());
      }
    }
    if (dex_files.empty()) {
      error_msgs->push_back("Failed to open dex files from " + source_oat_file->GetLocation());
    } else {
      // Opened dex files from an oat file, madvise them to their loaded state.
       for (const std::unique_ptr<const DexFile>& dex_file : dex_files) {
         OatDexFile::MadviseDexFile(*dex_file, MadviseState::kMadviseStateAtLoad);
       }
    }
  }
  // Fall back to running out of the original dex file if we couldn't load any
  // dex_files from the oat file.
  if (dex_files.empty()) {
    if (oat_file_assistant.HasOriginalDexFiles()) {
      if (Runtime::Current()->IsDexFileFallbackEnabled()) {
        static constexpr bool kVerifyChecksum = true;
        const ArtDexFileLoader dex_file_loader;
        if (!dex_file_loader.Open(dex_location,
                                  dex_location,
                                  Runtime::Current()->IsVerificationEnabled(),
                                  kVerifyChecksum,
                                  /*out*/ &error_msg,
                                  &dex_files)) {
          LOG(WARNING) << error_msg;
          error_msgs->push_back("Failed to open dex files from " + std::string(dex_location)
                                + " because: " + error_msg);
        }
      } else {
        error_msgs->push_back("Fallback mode disabled, skipping dex files.");
      }
    } else {
      error_msgs->push_back("No original dex files found for dex location "
          + std::string(dex_location));
    }
  }
  if (Runtime::Current()->GetJit() != nullptr) {
    ScopedObjectAccess soa(self);
    Runtime::Current()->GetJit()->RegisterDexFiles(
        dex_files, soa.Decode<mirror::ClassLoader>(class_loader));
  }
  return dex_files;
}

```

     以上代码主要是将 **dex** 文件转 **oat**。如果转 **oat** 失败，那就打开 **dex** 的方式来加载。  

**如果你对安卓相关的开发学习感兴趣:**

       可加作者的 QQ 群（**1017017661**), 本群专注安卓方面的技术，欢迎加群技术交流。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |** 查看更多文章**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)