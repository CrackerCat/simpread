> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284744.htm)

> [原创]dpt-shell 源码详细解析（v1.11.3）

dpt-shell 自 22 年以来进行了不少的调整，本文试对 v1.11.3 版的 dpt-shell 进行源码分析，补充作者在 HowItWorks 中未撰写出来的部分并作积累。

简而言之，dpt-shell 可以分为两个模块，一个是 Processeor 模块，用于对原 app 进行指令抽空并构建新 app；另一个是 shell 模块，用于在 app 运行时回填指令，顺利执行 app 的代码。以下是对这两个模块的详细分析

入口点在 src\main\java\com\luoye\dpt\Dpt.java，解析用户的运行参数后进入 apk.protect() 进行抽取

extractDexCode：调用 DexUtils.extractAllMethods 获取 List<Instruction> ret，最后将每个方法的字节码信息写入到`assets/OoooooOooo`文件

extractAllMethods：解析 Dex 文件的结构体，获取 directMethods，virtualMethods，调用 extractMethod 来进行 patch

extractMethod：一边用 byteCode 保存原字节码，一边用 outRandomAccessFile.writeShort(0) 写入 nop

下面主要贴一下 extractDexCode 的源码好了

主要是保存了一下源 ApplicationName 和 AppComponentFactory，后续壳加载的时候用

此外修改了程序的 AMF.xml，替换为代理 ApplicationName 和代理 AppComponentFactory

源 dex 压缩存放到 assets/i11111i111.zip

combineDexZipWithShellDex：将壳文件添加到 dex，并修复 size、sha1、checksum

copyNativeLibs 这里就没啥好说的，直接添加（shell-files/libs → assets/vwwwwwvwww）

encryptSoFiles 这块对比于初始版本是新添加的功能：

zipalign，对 zip 进行对齐

对 APK 进行签名，assets/dpt.jks

signApkDebug

signApk，调用 command 来实现

完成！

这里的逻辑也会初始版本发生了一些变化，直接分析当前版本的

JniBridge.loadShellLibs(applicationInfo.dataDir,applicationInfo.sourceDir)

JniBridge.ia()：init_app

JniBridge.cbde：combineDexElements，动态合并新的 dex

先调用父类的 onCreate，后面主要看 replaceApplication，同样发生在 native 层

JniBridge.ra：replaceApplication，主要实例化了一个 application，执行 replaceApplicationOnLoadedApk 和 replaceApplicationOnActivityThread 来替换这个实例

replaceApplicationOnLoadedApk

replaceApplicationOnActivityThread

这里做个区分：

JniBridge.craa：callRealApplicationAttach

JniBridge.craoc：callRealApplicationOnCreate

壳的 so 文件在 init_array 中优先调用

decrypt_bitcode

dpt_hook

函数路径位于 src\main\cpp\dpt_hook.cpp

hook libc.so 的 execve

字符串子串匹配 dex2oat，禁用 dex2oat

hook libc.so 的 mmap

添加写权限，以便后续修改 dex

hook DefineClass

这里通过 DobbyHook 库来对 java 层的代码进行 hook

classloader 在加载类的时候会调用 defineClass，根据 sdk 版本选择合适的 hook 函数，如

patchClass**（指令回填的核心函数）**

patchMethod

createAntiRiskProcess

detectFrida

doPtrace

在系统需要创建组件实例时按需调用，通过代理的方式控制组件的实例化，优先使用目标 `AppComponentFactory` 创建组件，创建失败再用默认的

回看 22 年的帖子，当时作者使用 LoadMethod 作为 hook 和指令回填的目标，现在已经换成 DefineClass，原因在 HowItWorks.md 也有解释

ClassDef 这个结构还有一个特点，它是 dex 文件的结构，也就是说 dex 文件格式不变，它一般就不会变。还有，DefineClass 函数的参数会改变吗？目前来看从 Android M 到现在没有变过。所以使用它不用太担心随着 Android 版本的升级而导致字段偏移的变化，也就是兼容性较强。这就是为什么用 DefineClass 作为 Hook 点。

dpt 之前就是使用的 LoadMethod 函数作为 Hook 点，在 LoadMethod 函数里面做 CodeItem 填充操作。但是后来发现，LoadMethod 函数参数不太固定，随着 Android 版本的升级可能要不断适配，而且每个函数都要填充，会影响一定的性能。

[分享一个自己做的函数抽取壳 - 吾爱破解 - 52pojie.cn](https://www.52pojie.cn/thread-1576245-1-1.html)

[https://github.com/luoyesiqiu/dpt-shell](https://github.com/luoyesiqiu/dpt-shell)

`private` `static` `void` `process(Apk apk){` `if``(!``new` `File(``"shell-files"``).exists()) {` `LogUtils.error(``"Cannot find shell files!"``);` `return``;` `}` `File apkFile =` `new` `File(apk.getFilePath());`   `if``(!apkFile.exists()){` `LogUtils.error(``"Apk not exists!"``);` `return``;` `}`   `//apk extract path` `String apkMainProcessPath = apk.getWorkspaceDir().getAbsolutePath();`   `LogUtils.info(``"Apk main process path: "` `+ apkMainProcessPath);`   `ZipUtils.unZip(apk.getFilePath(),apkMainProcessPath);` `String packageName = ManifestUtils.getPackageName(apkMainProcessPath + File.separator +` `"AndroidManifest.xml"``);` `apk.setPackageName(packageName);` `// 1. 指令抽空` `apk.extractDexCode(apkMainProcessPath);` `//  2. AMF的处理` `apk.saveApplicationName(apkMainProcessPath);` `// 保存原始ApplicationName到assets/app_name` `apk.writeProxyAppName(apkMainProcessPath);` `// 写入代理ApplicationName` `if``(apk.isAppComponentFactory()){` `apk.saveAppComponentFactory(apkMainProcessPath);` `// 保存原始AppComponentFactory到assets/app_acf` `apk.writeProxyComponentFactoryName(apkMainProcessPath);` `// 写入代理AppComponentFactory` `}` `if``(apk.isDebuggable()) {` `LogUtils.info(``"Make apk debuggable."``);` `apk.setDebuggable(apkMainProcessPath,` `true``);` `}`   `apk.setExtractNativeLibs(apkMainProcessPath);` `apk.addJunkCodeDex(apkMainProcessPath);` `// 3. 压缩源dex到新的路径并删除旧的路径` `apk.compressDexFiles(apkMainProcessPath);` `// 源dex压缩存放到assets/i11111i111.zip` `apk.deleteAllDexFiles(apkMainProcessPath);` `// 4. 合并壳dex和原dex` `apk.combineDexZipWithShellDex(apkMainProcessPath);` `// 5. 复制壳的so文件，加密so文件` `apk.copyNativeLibs(apkMainProcessPath);` `// 复制壳的so文件` `apk.encryptSoFiles(apkMainProcessPath);`   `// 6. 构建apk` `apk.buildApk(apkFile.getAbsolutePath(),apkMainProcessPath, FileUtils.getExecutablePath());`   `File apkMainProcessFile =` `new` `File(apkMainProcessPath);` `if` `(apkMainProcessFile.exists()) {` `FileUtils.deleteRecurse(apkMainProcessFile);` `}` `LogUtils.info(``"All done."``);` `}` `public` `void` `protect() {` `process(``this``);` `}` `private` `static` `void` `process(Apk apk){`

 `if``(!``new` `File(``"shell-files"``).exists()) {`

 `LogUtils.error(``"Cannot find shell files!"``);`

 `return``;`

 `}`

 `File apkFile =` `new` `File(apk.getFilePath());`

 

 `if``(!apkFile.exists()){`

 `LogUtils.error(``"Apk not exists!"``);`

 `return``;`

 `}`

 

 `//apk extract path`

 `String apkMainProcessPath = apk.getWorkspaceDir().getAbsolutePath();`

 

 `LogUtils.info(``"Apk main process path: "` `+ apkMainProcessPath);`

 

 `ZipUtils.unZip(apk.getFilePath(),apkMainProcessPath);`

 `String packageName = ManifestUtils.getPackageName(apkMainProcessPath + File.separator +` `"AndroidManifest.xml"``);`

 `apk.setPackageName(packageName);`

 `// 1. 指令抽空`

 `apk.extractDexCode(apkMainProcessPath);`

 `//  2. AMF的处理`

 `apk.saveApplicationName(apkMainProcessPath);` `// 保存原始ApplicationName到assets/app_name`

 `apk.writeProxyAppName(apkMainProcessPath);` `// 写入代理ApplicationName`

 `if``(apk.isAppComponentFactory()){`

 `apk.saveAppComponentFactory(apkMainProcessPath);` `// 保存原始AppComponentFactory到assets/app_acf`

 `apk.writeProxyComponentFactoryName(apkMainProcessPath);` `// 写入代理AppComponentFactory`

 `}`

 `if``(apk.isDebuggable()) {`

 `LogUtils.info(``"Make apk debuggable."``);`

 `apk.setDebuggable(apkMainProcessPath,` `true``);`

 `}`

 

 `apk.setExtractNativeLibs(apkMainProcessPath);`

 `apk.addJunkCodeDex(apkMainProcessPath);`

 `// 3. 压缩源dex到新的路径并删除旧的路径`

 `apk.compressDexFiles(apkMainProcessPath);` `// 源dex压缩存放到assets/i11111i111.zip`

 `apk.deleteAllDexFiles(apkMainProcessPath);`

 `// 4. 合并壳dex和原dex`

 `apk.combineDexZipWithShellDex(apkMainProcessPath);`

 `// 5. 复制壳的so文件，加密so文件`

 `apk.copyNativeLibs(apkMainProcessPath);` `// 复制壳的so文件`

 `apk.encryptSoFiles(apkMainProcessPath);`

 

 `// 6. 构建apk`

 `apk.buildApk(apkFile.getAbsolutePath(),apkMainProcessPath, FileUtils.getExecutablePath());`

 

 `File apkMainProcessFile =` `new` `File(apkMainProcessPath);`

 `if` `(apkMainProcessFile.exists()) {`

 `FileUtils.deleteRecurse(apkMainProcessFile);`

 `}`

 `LogUtils.info(``"All done."``);`

`}`

`public` `void` `protect() {`

 `process(``this``);`

`}` `private` `void`  `extractDexCode(String apkOutDir){` `List<File> dexFiles = getDexFiles(apkOutDir);` `Map<Integer,List<Instruction>> instructionMap =` `new` `HashMap<>();` `String appNameNew =` `"OoooooOooo"``;` `String dataOutputPath = getOutAssetsDir(apkOutDir).getAbsolutePath() + File.separator + appNameNew;`   `CountDownLatch countDownLatch =` `new` `CountDownLatch(dexFiles.size());` `for``(File dexFile : dexFiles) {` `ThreadPool.getInstance().execute(() -> {` `final` `int` `dexNo = getDexNumber(dexFile.getName());` `if``(dexNo < 0){` `return``;` `}` `String extractedDexName = dexFile.getName().endsWith(``".dex"``) ? dexFile.getName().replaceAll(``"\\.dex$"``,` `"_extracted.dat"``) :` `"_extracted.dat"``;` `File extractedDexFile =` `new` `File(dexFile.getParent(), extractedDexName);`   `List<Instruction> ret = DexUtils.extractAllMethods(dexFile, extractedDexFile, getPackageName(), isDumpCode());` `instructionMap.put(dexNo,ret);`   `File dexFileRightHashes =` `new` `File(dexFile.getParent(),FileUtils.getNewFileSuffix(dexFile.getName(),``"dat"``));` `DexUtils.writeHashes(extractedDexFile,dexFileRightHashes);` `dexFile.``delete``();` `extractedDexFile.``delete``();` `dexFileRightHashes.renameTo(dexFile);` `countDownLatch.countDown();` `});`   `}`   `ThreadPool.getInstance().shutdown();`   `try` `{` `countDownLatch.await();` `}` `catch` `(Exception ignored){` `}`   `MultiDexCode multiDexCode = MultiDexCodeUtils.makeMultiDexCode(instructionMap);`   `MultiDexCodeUtils.writeMultiDexCode(dataOutputPath,multiDexCode);`   `}` `private` `void`  `extractDexCode(String apkOutDir){`

 `List<File> dexFiles = getDexFiles(apkOutDir);`

 `Map<Integer,List<Instruction>> instructionMap =` `new` `HashMap<>();`

 `String appNameNew =` `"OoooooOooo"``;`

 `String dataOutputPath = getOutAssetsDir(apkOutDir).getAbsolutePath() + File.separator + appNameNew;`

 

 `CountDownLatch countDownLatch =` `new` `CountDownLatch(dexFiles.size());`

 `for``(File dexFile : dexFiles) {`

 `ThreadPool.getInstance().execute(() -> {`

 `final` `int` `dexNo = getDexNumber(dexFile.getName());`

 `if``(dexNo < 0){`

 `return``;`

 `}`

 `String extractedDexName = dexFile.getName().endsWith(``".dex"``) ? dexFile.getName().replaceAll(``"\\.dex$"``,` `"_extracted.dat"``) :` `"_extracted.dat"``;`

 `File extractedDexFile =` `new` `File(dexFile.getParent(), extractedDexName);`

 

 `List<Instruction> ret = DexUtils.extractAllMethods(dexFile, extractedDexFile, getPackageName(), isDumpCode());`

 `instructionMap.put(dexNo,ret);`

 

 `File dexFileRightHashes =` `new` `File(dexFile.getParent(),FileUtils.getNewFileSuffix(dexFile.getName(),``"dat"``));`

 `DexUtils.writeHashes(extractedDexFile,dexFileRightHashes);`

 `dexFile.``delete``();`

 `extractedDexFile.``delete``();`

 `dexFileRightHashes.renameTo(dexFile);`

 `countDownLatch.countDown();`

 `});`

 

 `}`

 

 `ThreadPool.getInstance().shutdown();`

 

 `try` `{`

 `countDownLatch.await();`

 `}`

 `catch` `(Exception ignored){`

 `}`

 

 `MultiDexCode multiDexCode = MultiDexCodeUtils.makeMultiDexCode(instructionMap);`

 

 `MultiDexCodeUtils.writeMultiDexCode(dataOutputPath,multiDexCode);`

 

`}` `private` `static` `boolean` `signApk(String apkPath, String keyStorePath, String signedApkPath,` `String keyAlias,` `String storePassword,` `String KeyPassword) {` `ArrayList<String> commandList =` `new` `ArrayList<>();`   `commandList.add(``"sign"``);` `commandList.add(``"--ks"``);` `commandList.add(keyStorePath);` `commandList.add(``"--ks-key-alias"``);` `commandList.add(keyAlias);` `commandList.add(``"--ks-pass"``);` `commandList.add(``"pass:"` `+ storePassword);` `commandList.add(``"--key-pass"``);` `commandList.add(``"pass:"` `+ KeyPassword);` `commandList.add(``"--out"``);` `commandList.add(signedApkPath);` `commandList.add(``"--v1-signing-enabled"``);` `commandList.add(``"true"``);` `commandList.add(``"--v2-signing-enabled"``);` `commandList.add(``"true"``);` `commandList.add(``"--v3-signing-enabled"``);` `commandList.add(``"true"``);` `commandList.add(apkPath);`   `int` `size = commandList.size();` `String[] commandArray =` `new` `String[size];` `commandArray = commandList.toArray(commandArray);`   `try` `{` `ApkSignerTool.main(commandArray);` `}` `catch` `(Exception e) {` `e.printStackTrace();` `return` `false``;` `}` `return` `true``;` `}` `private` `static` `boolean` `signApk(String apkPath, String keyStorePath, String signedApkPath,`

 `String keyAlias,`

 `String storePassword,`

 `String KeyPassword) {`

 `ArrayList<String> commandList =` `new` `ArrayList<>();`

 

 `commandList.add(``"sign"``);`

 `commandList.add(``"--ks"``);`

 `commandList.add(keyStorePath);`

 `commandList.add(``"--ks-key-alias"``);`

 `commandList.add(keyAlias);`

 `commandList.add(``"--ks-pass"``);`

 `commandList.add(``"pass:"` `+ storePassword);`

 `commandList.add(``"--key-pass"``);`

 `commandList.add(``"pass:"` `+ KeyPassword);`

 `commandList.add(``"--out"``);`

 `commandList.add(signedApkPath);`

 `commandList.add(``"--v1-signing-enabled"``);`

 `commandList.add(``"true"``);`

 `commandList.add(``"--v2-signing-enabled"``);`

 `commandList.add(``"true"``);`

 `commandList.add(``"--v3-signing-enabled"``);`

 `commandList.add(``"true"``);`

 `commandList.add(apkPath);`

 

 `int` `size = commandList.size();`

 `String[] commandArray =` `new` `String[size];`

 `commandArray = commandList.toArray(commandArray);`

 

 `try` `{`

 `ApkSignerTool.main(commandArray);`

 `}` `catch` `(Exception e) {`

 `e.printStackTrace();`

 `return` `false``;`

 `}`

 `return` `true``;`

`}` `@Override` `protected` `void` `attachBaseContext(Context base) {` `super``.attachBaseContext(base);` `// 先调用父类的attachBaseContext` `Log.d(TAG,``"dpt attachBaseContext classloader = "` `+ base.getClassLoader());` `realApplicationName = FileUtils.readAppName(``this``);` `if``(!Global.sIsReplacedClassLoader) {` `ApplicationInfo applicationInfo = base.getApplicationInfo();` `if``(applicationInfo ==` `null``) {` `throw` `new` `NullPointerException(``"application info is null"``);` `}` `FileUtils.unzipLibs(applicationInfo.sourceDir,applicationInfo.dataDir);` `JniBridge.loadShellLibs(applicationInfo.dataDir,applicationInfo.sourceDir);` `Log.d(TAG,``"ProxyApplication init"``);` `JniBridge.ia();` `ClassLoader targetClassLoader = base.getClassLoader();` `JniBridge.cbde(targetClassLoader);` `Global.sIsReplacedClassLoader =` `true``;` `}` `}` `@Override`

`protected` `void` `attachBaseContext(Context base) {`

 `super``.attachBaseContext(base);` `// 先调用父类的attachBaseContext`

 `Log.d(TAG,``"dpt attachBaseContext classloader = "` `+ base.getClassLoader());`

 `realApplicationName = FileUtils.readAppName(``this``);`

 `if``(!Global.sIsReplacedClassLoader) {`

 `ApplicationInfo applicationInfo = base.getApplicationInfo();`

 `if``(applicationInfo ==` `null``) {`

 `throw` `new` `NullPointerException(``"application info is null"``);`

 `}`

 `FileUtils.unzipLibs(applicationInfo.sourceDir,applicationInfo.dataDir);`

 `JniBridge.loadShellLibs(applicationInfo.dataDir,applicationInfo.sourceDir);`

 `Log.d(TAG,``"ProxyApplication init"``);`

 `JniBridge.ia();`

 `ClassLoader targetClassLoader = base.getClassLoader();`

 `JniBridge.cbde(targetClassLoader);`

 `Global.sIsReplacedClassLoader =` `true``;`

 `}`

`}` `DPT_ENCRYPT` `void` `init_app(JNIEnv *env, jclass __unused) {` `DLOGD(``"init_app!"``);` `clock_t start = clock();`   `void` `*apk_addr = nullptr;` `size_t apk_size =` `0``;` `load_apk(env,&apk_addr,&apk_size);`   `uint64_t entry_size =` `0``;` `if``(codeItemFilePtr == nullptr) {` `read_zip_file_entry(apk_addr,apk_size,CODE_ITEM_NAME_IN_ZIP,&codeItemFilePtr,&entry_size);` `}` `else` `{` `DLOGD(``"no need read codeitem from zip"``);` `}` `readCodeItem((uint8_t *)codeItemFilePtr,entry_size);`   `pthread_mutex_lock(&g_write_dexes_mutex);` `extractDexesInNeeded(env,apk_addr,apk_size);` `pthread_mutex_unlock(&g_write_dexes_mutex);`   `unload_apk(apk_addr,apk_size);` `printTime(``"read apk data took ="` `, start);` `}` `DPT_ENCRYPT` `void` `init_app(JNIEnv *env, jclass __unused) {`

 `DLOGD(``"init_app!"``);`

 `clock_t start = clock();`

 

 `void` `*apk_addr = nullptr;`

 `size_t apk_size =` `0``;`

 `load_apk(env,&apk_addr,&apk_size);`

 

 `uint64_t entry_size =` `0``;`

 `if``(codeItemFilePtr == nullptr) {`

 `read_zip_file_entry(apk_addr,apk_size,CODE_ITEM_NAME_IN_ZIP,&codeItemFilePtr,&entry_size);`

 `}`

 `else` `{`

 `DLOGD(``"no need read codeitem from zip"``);`

 `}`

 `readCodeItem((uint8_t *)codeItemFilePtr,entry_size);`

 

 `pthread_mutex_lock(&g_write_dexes_mutex);`

 `extractDexesInNeeded(env,apk_addr,apk_size);`

 `pthread_mutex_unlock(&g_write_dexes_mutex);`

 

 `unload_apk(apk_addr,apk_size);`

 `printTime(``"read apk data took ="` `, start);`

`}` `DPT_ENCRYPT` `void` `combineDexElement(JNIEnv* env, jclass __unused, jobject targetClassLoader,` `const` `char``* pathChs) {` `jobjectArray extraDexElements = makePathElements(env,pathChs);`   `dalvik_system_BaseDexClassLoader targetBaseDexClassLoader(env,targetClassLoader);`   `jobject originDexPathListObj = targetBaseDexClassLoader.getPathList();`   `dalvik_system_DexPathList targetDexPathList(env,originDexPathListObj);`   `jobjectArray originDexElements = targetDexPathList.getDexElements();`   `jsize extraSize = env->GetArrayLength(extraDexElements);` `jsize originSize = env->GetArrayLength(originDexElements);`   `dalvik_system_DexPathList::Element element(env, nullptr);` `jclass ElementClass = element.getClass();` `jobjectArray  newDexElements = env->NewObjectArray(originSize + extraSize,ElementClass, nullptr);`   `for``(``int` `i = 0;i < originSize;i++) {` `jobject elementObj = env->GetObjectArrayElement(originDexElements, i);` `env->SetObjectArrayElement(newDexElements,i,elementObj);` `}`   `for``(``int` `i = originSize;i < originSize + extraSize;i++) {` `jobject elementObj = env->GetObjectArrayElement(extraDexElements, i - originSize);` `env->SetObjectArrayElement(newDexElements,i,elementObj);` `}`   `targetDexPathList.setDexElements(newDexElements);`   `DLOGD(``"combineDexElement success"``);` `}` `DPT_ENCRYPT` `void` `combineDexElement(JNIEnv* env, jclass __unused, jobject targetClassLoader,` `const` `char``* pathChs) {`

 `jobjectArray extraDexElements = makePathElements(env,pathChs);`

 

 `dalvik_system_BaseDexClassLoader targetBaseDexClassLoader(env,targetClassLoader);`

 

 `jobject originDexPathListObj = targetBaseDexClassLoader.getPathList();`

 

 `dalvik_system_DexPathList targetDexPathList(env,originDexPathListObj);`

 

 `jobjectArray originDexElements = targetDexPathList.getDexElements();`

 

 `jsize extraSize = env->GetArrayLength(extraDexElements);`

 `jsize originSize = env->GetArrayLength(originDexElements);`

 

 `dalvik_system_DexPathList::Element element(env, nullptr);`

 `jclass ElementClass = element.getClass();`

 `jobjectArray  newDexElements = env->NewObjectArray(originSize + extraSize,ElementClass, nullptr);`

 

 `for``(``int` `i = 0;i < originSize;i++) {`

 `jobject elementObj = env->GetObjectArrayElement(originDexElements, i);`

 `env->SetObjectArrayElement(newDexElements,i,elementObj);`

 `}`

 

 `for``(``int` `i = originSize;i < originSize + extraSize;i++) {`

 `jobject elementObj = env->GetObjectArrayElement(extraDexElements, i - originSize);`

 `env->SetObjectArrayElement(newDexElements,i,elementObj);`

 `}`

 

 `targetDexPathList.setDexElements(newDexElements);`

 

 `DLOGD(``"combineDexElement success"``);`

`}` `private` `void` `replaceApplication() {` `if` `(Global.sNeedCalledApplication && !TextUtils.isEmpty(realApplicationName)) {` `realApplication = (Application) JniBridge.ra(realApplicationName);` `Log.d(TAG,` `"applicationExchange: "` `+ realApplicationName+``"  realApplication="``+realApplication.getClass().getName());`   `JniBridge.craa(getApplicationContext(), realApplicationName);` `JniBridge.craoc(realApplicationName);` `Global.sNeedCalledApplication =` `false``;` `}` `}` `private` `void` `replaceApplication() {`

 `if` `(Global.sNeedCalledApplication && !TextUtils.isEmpty(realApplicationName)) {`

 `realApplication = (Application) JniBridge.ra(realApplicationName);`

 `Log.d(TAG,` `"applicationExchange: "` `+ realApplicationName+``"  realApplication="``+realApplication.getClass().getName());`

 

 `JniBridge.craa(getApplicationContext(), realApplicationName);`

 `JniBridge.craoc(realApplicationName);`

 `Global.sNeedCalledApplication =` `false``;`

 `}`

`}` `DPT_ENCRYPT` `void` `replaceApplicationOnLoadedApk(JNIEnv *env, jclass __unused,jobject realApplication) {` `android_app_ActivityThread activityThread(env);`   `jobject mBoundApplicationObj = activityThread.getBoundApplication();` `// 获取 BoundApplication 对象`   `android_app_ActivityThread::AppBindData appBindData(env,mBoundApplicationObj);` `jobject loadedApkObj = appBindData.getInfo();`   `android_app_LoadedApk loadedApk(env,loadedApkObj);` `// LoadedApk对象是APK文件在内存中的表示`   `//make it null` `loadedApk.setApplication(nullptr);` `// 以便可以替换为新的 Application 对象。`   `jobject mAllApplicationsObj = activityThread.getAllApplication();`   `java_util_ArrayList arrayList(env,mAllApplicationsObj);`   `jobject removed = (jobject)arrayList.``remove``(0);` `// 移除原来的 Application 对象。` `if``(removed != nullptr){` `DLOGD(``"replaceApplicationOnLoadedApk proxy application removed"``);` `}`   `jobject ApplicationInfoObj = loadedApk.getApplicationInfo();` `// 获取 ApplicationInfo 对象。`   `android_content_pm_ApplicationInfo applicationInfo(env,ApplicationInfoObj);`   `char` `applicationName[128] = {0};` `getClassName(env,realApplication,applicationName, ARRAY_LENGTH(applicationName));` `// 获取真实 Application 对象的类名`   `DLOGD(``"applicationName = %s"``,applicationName);` `char` `realApplicationNameChs[128] = {0};` `parseClassName(applicationName,realApplicationNameChs);` `// 前面获取了类名了，现在解析类` `jstring realApplicationName = env->NewStringUTF(realApplicationNameChs);` `auto` `realApplicationNameGlobal = (jstring)env->NewGlobalRef(realApplicationName);`   `android_content_pm_ApplicationInfo appInfo(env,appBindData.getAppInfo());`   `//replace class name    替换类名` `applicationInfo.setClassName(realApplicationNameGlobal);` `appInfo.setClassName(realApplicationNameGlobal);`   `DLOGD(``"replaceApplicationOnLoadedApk begin makeApplication!"``);`   `// call make application` `loadedApk.makeApplication(JNI_FALSE,nullptr);`   `DLOGD(``"replaceApplicationOnLoadedApk success!"``);` `}` `DPT_ENCRYPT` `void` `replaceApplicationOnLoadedApk(JNIEnv *env, jclass __unused,jobject realApplication) {`

 `android_app_ActivityThread activityThread(env);`

 

 `jobject mBoundApplicationObj = activityThread.getBoundApplication();` `// 获取 BoundApplication 对象`

 

 `android_app_ActivityThread::AppBindData appBindData(env,mBoundApplicationObj);`

 `jobject loadedApkObj = appBindData.getInfo();`

 

 `android_app_LoadedApk loadedApk(env,loadedApkObj);` `// LoadedApk对象是APK文件在内存中的表示`

 

 `//make it null`

 `loadedApk.setApplication(nullptr);` `// 以便可以替换为新的 Application 对象。`

 

 `jobject mAllApplicationsObj = activityThread.getAllApplication();`

 

 `java_util_ArrayList arrayList(env,mAllApplicationsObj);`

 

 `jobject removed = (jobject)arrayList.``remove``(0);` `// 移除原来的 Application 对象。`

 `if``(removed != nullptr){`

 `DLOGD(``"replaceApplicationOnLoadedApk proxy application removed"``);`

 `}`

 

 `jobject ApplicationInfoObj = loadedApk.getApplicationInfo();` `// 获取 ApplicationInfo 对象。`

 

 `android_content_pm_ApplicationInfo applicationInfo(env,ApplicationInfoObj);`

 

[登录后可查看完整内容](javascript:;)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#混淆加固](forum-161-1-121.htm) [#源码框架](forum-161-1-127.htm)