> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286703.htm)

> [推荐][原创][分享] 记录关于 ClassLoader 一代整体壳的脱壳点

/art/libdexfile/dex/dex_file_loader.cc 下的 OpenCommon 方法（android-14.0.0_r21）
---------------------------------------------------------------------------

DexClassLoader

```
public class DexClassLoader extends BaseDexClassLoader {
36      /**
37       * Creates a {@code DexClassLoader} that finds interpreted and native
38       * code.  Interpreted classes are found in a set of DEX files contained
39       * in Jar or APK files.
40       *
41       * <p>The path lists are separated using the character specified by the
42       * {@code path.separator} system property, which defaults to {@code :}.
43       *
44       * @param dexPath the list of jar/apk files containing classes and
45       *     resources, delimited by {@code File.pathSeparator}, which
46       *     defaults to {@code ":"} on Android
47       * @param optimizedDirectory this parameter is deprecated and has no effect since API level 26.
48       * @param librarySearchPath the list of directories containing native
49       *     libraries, delimited by {@code File.pathSeparator}; may be
50       *     {@code null}
51       * @param parent the parent class loader
52       */
53      public DexClassLoader(String dexPath, String optimizedDirectory,
54              String librarySearchPath, ClassLoader parent) {
55          super(dexPath, null, librarySearchPath, parent);
56      }
57  }
```

BaseDexClassLoader

```
  public BaseDexClassLoader(String dexPath,
151              String librarySearchPath, ClassLoader parent, ClassLoader[] sharedLibraryLoaders,
152              ClassLoader[] sharedLibraryLoadersAfter,
153              boolean isTrusted) {
   ......
160          this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
161  
162          this.sharedLibraryLoadersAfter = sharedLibraryLoadersAfter == null
163                  ? null
164                  : Arrays.copyOf(sharedLibraryLoadersAfter, sharedLibraryLoadersAfter.length);
165          // Run background verification after having set 'pathList'.
166          this.pathList.maybeRunBackgroundVerification(this);
167  
168          reportClassLoaderChain();
169      }
```

进入 DexPathList

```
 DexPathList(ClassLoader definingContext, String dexPath,
138              String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
139          if (definingContext == null) {
140              throw new NullPointerException("definingContext == null");
141          }
142  
143          if (dexPath == null) {
144              throw new NullPointerException("dexPath == null");
145          }
146  
147          if (optimizedDirectory != null) {
148              if (!optimizedDirectory.exists())  {
149                  throw new IllegalArgumentException(
150                          "optimizedDirectory doesn't exist: "
151                          + optimizedDirectory);
152              }
153  
154              if (!(optimizedDirectory.canRead()
155                              && optimizedDirectory.canWrite())) {
156                  throw new IllegalArgumentException(
157                          "optimizedDirectory not readable/writable: "
158                          + optimizedDirectory);
159              }
160          }
161  
162          this.definingContext = definingContext;
163  
164          ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
165          // save dexPath for BaseDexClassLoader
166          this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
167                                             suppressedExceptions, definingContext, isTrusted);
168  
   ......
182          this.nativeLibraryPathElements = makePathElements(getAllNativeLibraryDirectories());
183  
184          if (suppressedExceptions.size() > 0) {
185              this.dexElementsSuppressedExceptions =
186                  suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
187          } else {
188              dexElementsSuppressedExceptions = null;
189          }
190      }
```

dexElements 是 App 所有的 dex 集合，nativeLibraryPathElements 是 App 所有的 so 本地共享库（lib 目录下的 so）

splitDexPath(dexPath) 分割路径成多个 dex，封装成 File

进入 makeDexElements(splitDexPath(dexPath), optimizedDirectory,suppressedExceptions, definingContext, isTrusted);

```
private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
369              List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
370        Element[] elements = new Element[files.size()];
371        int elementsPos = 0;
372        /*
373         * Open all files and load the (direct or contained) dex files up front.
374         */
375        for (File file : files) {
376            if (file.isDirectory()) {
377                // We support directories for looking up resources. Looking up resources in
378                // directories is useful for running libcore tests.
379                elements[elementsPos++] = new Element(file);
380            } else if (file.isFile()) {
381                String name = file.getName();
382  
383                DexFile dex = null;
384                if (name.endsWith(DEX_SUFFIX)) {
385                    // Raw dex file (not inside a zip/jar).
386                    try {
387                        dex = loadDexFile(file, optimizedDirectory, loader, elements);
388                        if (dex != null) {
389                            elements[elementsPos++] = new Element(dex, null);
390                        }
391                    } catch (IOException suppressed) {
392                        System.logE("Unable to load dex file: " + file, suppressed);
393                        suppressedExceptions.add(suppressed);
394                    }
395                } else {
396                    try {
397                        dex = loadDexFile(file, optimizedDirectory, loader, elements);
398                    } catch (IOException suppressed) {
399                        /*
400                         * IOException might get thrown "legitimately" by the DexFile constructor if
401                         * the zip file turns out to be resource-only (that is, no classes.dex file
402                         * in it).
403                         * Let dex == null and hang on to the exception to add to the tea-leaves for
404                         * when findClass returns null.
405                         */
406                        suppressedExceptions.add(suppressed);
407                    }
408  
409                    if (dex == null) {
410                        elements[elementsPos++] = new Element(file);
411                    } else {
412                        elements[elementsPos++] = new Element(dex, file);
413                    }
414                }
415                if (dex != null && isTrusted) {
416                  dex.setTrusted();
417                }
418            } else {
419                System.logW("ClassLoader referenced unknown path: " + file);
420            }
421        }
422        if (elementsPos != elements.length) {
423            elements = Arrays.copyOf(elements, elementsPos);
424        }
425        return elements;
426      }
```

dex = loadDexFile(file, optimizedDirectory, loader, elements);

进入 loadDexFile

```
   private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader,
435                                         Element[] elements)
436              throws IOException {
437          if (optimizedDirectory == null) {
438              return new DexFile(file, loader, elements);
439          } else {
440              String optimizedPath = optimizedPathFor(file, optimizedDirectory);
441              return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
442          }
443      }
444  
```

optimizedDirectory 前面传进来的是 null

进入 new DexFile(file, loader, elements);

```
    DexFile(String fileName, ClassLoader loader, DexPathList.Element[] elements)
127              throws IOException {
128          mCookie = openDexFile(fileName, null, 0, loader, elements);
129          mInternalCookie = mCookie;
130          mFileName = fileName;
131          //System.out.println("DEX FILE cookie is " + mCookie + " fileName=" + fileName);
132      }
```

进入 openDexFile(fileName, null, 0, loader, elements);

```
  private static Object openDexFile(String sourceName, String outputName, int flags,
404              ClassLoader loader, DexPathList.Element[] elements) throws IOException {
405          // Use absolute paths to enable the use of relative paths when testing on host.
406          return openDexFileNative(new File(sourceName).getAbsolutePath(),
407                                   (outputName == null)
408                                       ? null
409                                       : new File(outputName).getAbsolutePath(),
410                                   flags,
411                                   loader,
412                                   elements);
413      }

7      private static native Object openDexFileNative(String sourceName, String outputName, int flags,
478              ClassLoader loader, DexPathList.Element[] elements);
```

可以看到 openDexFileNative(new File(sourceName).getAbsolutePath(), (outputName == null) ? null: new File(outputName).getAbsolutePath(),flags,loader,elements); 是一个 native 方法，搜 DexFile_openDexFileNative, 由 java 层的类名拼接上方法名字

代码位置：/art/runtime/native/dalvik_system_DexFile.cc

```
static jobject DexFile_openDexFileNative(JNIEnv* env,
369                                           jclass,
370                                           jstring javaSourceName,
371                                           jstring javaOutputName ATTRIBUTE_UNUSED,
372                                           jint flags ATTRIBUTE_UNUSED,
373                                           jobject class_loader,
374                                           jobjectArray dex_elements) {
  ......
393    std::vector<std::unique_ptr<const DexFile>> dex_files =
394        Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat(sourceName.c_str(),
395                                                                    class_loader,
396                                                                    dex_elements,
397                                                                    /*out*/ &oat_file,
398                                                                    /*out*/ &error_msgs);
399    return CreateCookieFromOatFileManagerResult(env, dex_files, oat_file, error_msgs);
400  }
```

进入 OpenDexFilesFromOat(sourceName.c_str(),class_loader,dex_elements, &oat_file, &error_msgs)

代码位置：/art/runtime/oat_file_manager.cc

```
 std::vector<std::unique_ptr<const DexFile>> OatFileManager::OpenDexFilesFromOat(
178      const char* dex_location,
179      jobject class_loader,
180      jobjectArray dex_elements,
181      const OatFile** out_oat_file,
182      std::vector<std::string>* error_msgs) {
183    ScopedTrace trace(StringPrintf("%s(%s)", __FUNCTION__, dex_location));
184    CHECK(dex_location != nullptr);
185    CHECK(error_msgs != nullptr);
186  
187    // Verify we aren't holding the mutator lock, which could starve GC when
188    // hitting the disk.
189    Thread* const self = Thread::Current();
190    Locks::mutator_lock_->AssertNotHeld(self);
191    Runtime* const runtime = Runtime::Current();
192  
193    std::vector<std::unique_ptr<const DexFile>> dex_files;
194    std::unique_ptr<ClassLoaderContext> context(
195        ClassLoaderContext::CreateContextForClassLoader(class_loader, dex_elements));
196  
197    // If the class_loader is null there's not much we can do. This happens if a dex files is loaded
198    // directly with DexFile APIs instead of using class loaders.
199    if (class_loader == nullptr) {
200      LOG(WARNING) << "Opening an oat file without a class loader. "
201                   << "Are you using the deprecated DexFile APIs?";
202    } else if (context != nullptr) {
203      auto oat_file_assistant = std::make_unique<OatFileAssistant>(dex_location,
204                                                                   kRuntimeISA,
205                                                                   context.get(),
206                                                                   runtime->GetOatFilesExecutable(),
207                                                                   only_use_system_oat_files_);
208  
209      
  ......
448  
449    // If we arrive here with an empty dex files list, it means we fail to load
450    // it/them through an .oat file.
451    if (dex_files.empty()) {
452      std::string error_msg;
453      static constexpr bool kVerifyChecksum = true;
454      ArtDexFileLoader dex_file_loader(dex_location);
455      if (!dex_file_loader.Open(Runtime::Current()->IsVerificationEnabled(),
456                                kVerifyChecksum,
457                                /*out*/ &error_msg,
458                                &dex_files)) {
459        ScopedTrace fail_to_open_dex_from_apk("FailedToOpenDexFilesFromApk");
460        LOG(WARNING) << error_msg;
461        error_msgs->push_back("Failed to open dex files from " + std::string(dex_location)
462                              + " because: " + error_msg);
463      }
464    }
465  
//将加载完的DexFile注册到Jit中，Java虚拟机的即时编译器（JIT）
466    if (Runtime::Current()->GetJit() != nullptr) {
467      Runtime::Current()->GetJit()->RegisterDexFiles(dex_files, class_loader);
468    }
469  
//唤醒虚拟机可以工作了
470    // Now that we loaded the dex/odex files, notify the runtime.
471    // Note that we do this everytime we load dex files.
472    Runtime::Current()->NotifyDexFileLoaded();
473  
474    return dex_files;
475  }
```

```
ArtDexFileLoader dex_file_loader(dex_location);
 dex_file_loader.Open(Runtime::Current()->IsVerificationEnabled(),kVerifyChecksum,&error_msg,&dex_files)
```

ArtDexFileLoader 继承自 DexFileLoader

进入 dex_file_loader.Open

代码位置：/art/libdexfile/dex/art_dex_file_loader.h

```
184    bool Open(bool verify,
185              bool verify_checksum,
186              std::string* error_msg,
187              std::vector<std::unique_ptr<const DexFile>>* dex_files) {
188      DexFileLoaderErrorCode error_code;
189      return Open(verify,
190                  verify_checksum,
191                  /*allow_no_dex_files=*/false,
192                  &error_code,
193                  error_msg,
194                  dex_files);
195    }
```

进入 Open(verify,verify_checksum,false,&error_code,error_msg,dex_files);

代码位置：/art/libdexfile/dex/dex_file_loader.cc

```
bool DexFileLoader::InitAndReadMagic(uint32_t* magic, std::string* error_msg) {
217    if (root_container_ != nullptr) {
218      if (root_container_->Size() < sizeof(uint32_t)) {
219        *error_msg = StringPrintf("Unable to open '%s' : Size is too small", location_.c_str());
220        return false;
221      }
222      *magic = *reinterpret_cast<const uint32_t*>(root_container_->Begin());
223    } else {
224      // Open the file if we have not been given the file-descriptor directly before.
225      if (!file_.has_value()) {
226        CHECK(!filename_.empty());
//设置file_
227        file_.emplace(filename_, O_RDONLY, /* check_usage= */ false);
228        if (file_->Fd() == -1) {
229          *error_msg = StringPrintf("Unable to open '%s' : %s", filename_.c_str(), strerror(errno));
230          return false;
231        }
232      }
233      if (!ReadMagicAndReset(file_->Fd(), magic, error_msg)) {
234        return false;
235      }
236    }
237    return true;
238  }
```

// 设置 file_，和读取 Magic，代码位置：/art/libdexfile/dex/dex_file_loader.cc

```
 bool DexFileLoader::MapRootContainer(std::string* error_msg) {
241    if (root_container_ != nullptr) {
242      return true;
243    }
244  
245    CHECK(MemMap::IsInitialized());
246    CHECK(file_.has_value());
247    struct stat sbuf;
248    memset(&sbuf, 0, sizeof(sbuf));
249    if (fstat(file_->Fd(), &sbuf) == -1) {
250      *error_msg = StringPrintf("DexFile: fstat '%s' failed: %s", filename_.c_str(), strerror(errno));
251      return false;
252    }
253    if (S_ISDIR(sbuf.st_mode)) {
254      *error_msg = StringPrintf("Attempt to mmap directory '%s'", filename_.c_str());
255      return false;
256    }
257    MemMap map = MemMap::MapFile(sbuf.st_size,
258                                 PROT_READ,
259                                 MAP_PRIVATE,
260                                 file_->Fd(),
261                                 0,
262                                 /*low_4gb=*/false,
263                                 filename_.c_str(),
264                                 error_msg);
265    if (!map.IsValid()) {
266      DCHECK(!error_msg->empty());
267      return false;
268    }
269    root_container_ = std::make_shared<MemMapContainer>(std::move(map));
270    return true;
271  }
```

可以看到

MemMap map = MemMap::MapFile()

将文件进行内存映射，在 linux 中对设备和文件读写都是通过系统调用 `mmap()` 将文件映射到进程的地址空间中，通过指针进行读写。（万物皆文件）

赋值 root_container_ = std::make_shared<MemMapContainer>(std::move(map));

代码位置：/art/libdexfile/dex/dex_file_loader.cc

```
273  bool DexFileLoader::Open(bool verify,
274                           bool verify_checksum,
275                           bool allow_no_dex_files,
276                           DexFileLoaderErrorCode* error_code,
277                           std::string* error_msg,
278                           std::vector<std::unique_ptr<const DexFile>>* dex_files) {
279    DEXFILE_SCOPED_TRACE(std::string("Open dex file ") + location_);
280  
281    DCHECK(dex_files != nullptr) << "DexFile::Open: out-param is nullptr";
282  
283    uint32_t magic;
//
284    if (!InitAndReadMagic(&magic, error_msg)) {
285      return false;
286    }
287  
  ...
  
324    if (IsMagicValid(magic)) {
//这里初始化root_container_
325      if (!MapRootContainer(error_msg)) {
326        return false;
327      }
328      DCHECK(root_container_ != nullptr);
329      std::unique_ptr<const DexFile> dex_file =
330          OpenCommon(root_container_,
331                     root_container_->Begin(),
332                     root_container_->Size(),
333                     location_,
334                     /*location_checksum*/ {},  // Use default checksum from dex header.
335                     /*oat_dex_file=*/nullptr,
336                     verify,
337                     verify_checksum,
338                     error_msg,
339                     nullptr);
340      if (dex_file.get() != nullptr) {
341        dex_files->push_back(std::move(dex_file));
342        return true;
343      } else {
344        return false;
345      }
346    }
347    *error_msg = StringPrintf("Expected valid zip or dex file");
348    return false;
349  }
```

一代整体壳的脱壳点 / art/libdexfile/dex/dex_file_loader.cc 的 OpenCommon（android-14.0.0_r21）
----------------------------------------------------------------------------------

到这里，这里的 OpenCommon，有 root_container_->Begin(), root_container_->Size(), 可以 hook 该函数进行脱壳，对于一代整体壳可以脱壳。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#脱壳反混淆](forum-161-1-122.htm) [#系统相关](forum-161-1-126.htm)