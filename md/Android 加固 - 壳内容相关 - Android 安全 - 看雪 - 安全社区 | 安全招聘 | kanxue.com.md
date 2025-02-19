> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-275826.htm)

> Android 加固 - 壳内容相关

Dex 壳
=====

普通 Dex 壳
--------

### App 启动流程

熟悉 App 启动流程才能知道加壳 App 怎么制作。

 

![](https://bbs.kanxue.com/upload/attach/202301/754019_B629EDAPJPH44AJ.png)

 

通过 Zygote 进程到最终进入到 app 进程世界，我们可以看到 ActivityThread.main() 是进入 App 世界的大门，下面对该函数体进行简要的分析，具体分析请看文末的参考链接。

```
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
 
    // Install selective syscall interception
    AndroidOs.install();
 
    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);
 
    Environment.initForCurrentUser();
 
    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);
 
    Process.setArgV0("");
 
    Looper.prepareMainLooper();
 
    // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
    // It will be in the format "seq=114"
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(
                        args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
 
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
 
    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
 
    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();
 
    throw new RuntimeException("Main thread loop unexpectedly exited");
} 
```

对于 ActivityThread 这个类，其中的 sCurrentActivityThread 静态变量用于全局保存创建的 ActivityThread 实例，同时还提供了 public static ActivityThread currentActivityThread() 静态函数用于获取当前虚拟机创建的 ActivityThread 实例。ActivityThread.main() 函数是 java 中的入口 main 函数, 这里会启动主消息循环，并创建 ActivityThread 实例，之后调用 thread.attach(false) 完成一系列初始化准备工作，并完成全局静态变量 sCurrentActivityThread 的初始化。之后主线程进入消息循环，等待接收来自系统的消息。当收到系统发送来的 bindapplication 的进程间调用时，调用函数 handlebindapplication 来处理该请求。

```
@UnsupportedAppUsage
private void handleBindApplication(AppBindData data) {
    //step 1: 创建LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...
    //step 2: 创建ContextImpl对象;
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
 
    //step 3: 创建Instrumentation
    mInstrumentation = new Instrumentation();
 
    //step 4: 创建Application对象;在makeApplication函数中调用了newApplication，在该函数中又调用了app.attach(context)，在attach函数中调用了Application.attachBaseContext函数
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    mInitialApplication = app;
 
    //step 5: 安装providers
    List providers = data.providers;
    installContentProviders(app, providers);
 
    //step 6: 执行Application.Create回调
    mInstrumentation.callApplicationOnCreate(app); 
```

在 handleBindApplication 函数中第一次进入了 app 的代码世界，该函数功能是启动一个 application，并把系统收集的 apk 组件等相关信息绑定到 application 里，在创建完 application 对象后，接着调用了 application 的 attachBaseContext 方法，之后调用了 application 的 onCreate 函数。由此可以发现，app 的 Application 类中的 attachBaseContext 和 onCreate 这两个函数是最先获取执行权进行代码执行的。这也是为什么各家的加固工具的主要逻辑都是通过替换 app 入口 Application，并自实现这两个函数，在这两个函数中进行代码的脱壳以及执行权交付的原因。

### 加壳原理

在 App 启动流程中我们最终可以得出结论，app 最先获得执行权限的是 app 中声明的 Application 类中的 attachBaseContext 和 onCreate 函数。因此，壳要想完成应用中加固代码的解密以及应用执行权的交付就都是在这两个函数上做文章。下面这张图大致讲了加壳应用的运行流程。

 

![](https://bbs.kanxue.com/upload/attach/202301/754019_UN8EWDAUBPZZ4HJ.jpg)

 

打开加固的 app，整个流程如下：如果有签名校验和二次打包校验的话会直接校验签名和二次打包中一些文件是否被更改，如果被更改则崩溃，如果没有就进行加载处理。会在壳程序的 application 类中的 attachbasecontext 方法中对原 dex 进行解密处理，反射修改 loadapk 中的加载器为自定义加载器，反射设置 dexelements 将解密出来的 dex 文件路径加进去，获取源程序的 application 名称，反射生成 application 对象，反射设置 activitythread 中的 application 信息，activity 加载流程源程序正常运行。

### ClassLoader 在安卓加固中的作用

DexClassLoader 加载的类是没有组件生命周期的，也就是说即使 DexClassLoader 通过对 APK 的动态加载完成了对组件类的加载，当系统启动该组件时，依然会出现加载类失败的异常。为什么组件类被动态加载入虚拟机，但系统却出现加载类失败呢？

 

从 ClassLoader 来看，两种解决方案：

1.  替换系统组建类加载器为我们的 DexClassLoader，同时设置 DexClassLoader 的 parent 为系统组件类加载器;
2.  打破原有的双亲关系，在系统组件类加载器和 BootClassLoader 的中间插入我们自己的 DexClassLoader 即可;
3.  当然也可以对 BaseClassLoader 子类中的 Elements 进行合并。

> **LoadedApk:**
> 
> **private** [ClassLoader](elink@24eK9s2c8@1M7q4)9K6b7g2)9J5c8W2)9J5c8X3q4F1k6s2u0G2K9h3c8^5M7X3g2X3i4K6u0W2j5$3!0E0i4K6u0r3z5q4)9J5k6e0m8Q4x3X3f1H3i4K6g2X3M7U0c8Q4x3V1k6K6i4K6y4r3k6r3g2X3M7#2)9K6c8p5y4D9j5i4y4K6e0r3!0S2k6r3g2J5i4K6t1$3j5h3#2H3i4K6y4n7M7s2u0G2K9X3g2U0N6q4)9K6c8r3k6J5j5h3#2W2N6$3!0J5K9%4x3`.) [mClassLoader](elink@e1dK9s2c8@1M7q4)9K6b7g2)9J5c8W2)9J5c8X3q4F1k6s2u0G2K9h3c8^5M7X3g2X3i4K6u0W2j5$3!0E0i4K6u0r3z5q4)9J5k6e0m8Q4x3X3f1H3i4K6g2X3M7U0c8Q4x3V1k6K6i4K6y4r3M7X3g2X3M7#2)9K6c8r3#2o6L8r3q4K6M7@1I4G2j5h3c8W2M7W2)9J5y4X3q4E0M7q4)9K6b7Y4m8J5L8$3A6W2j5%4c8Q4x3@1c8X3M7X3q4E0k6i4N6G2M7X3E0K6);

```
// ActivityThread.java
public static ActivityThread currentActivityThread() {
   return sCurrentActivityThread;
}
```

**DexPathList.java 中的**

> **private** [Element](elink@b7eK9s2c8@1M7q4)9K6b7g2)9J5c8W2)9J5c8X3q4F1k6s2u0G2K9h3c8^5M7X3g2X3i4K6u0W2j5$3!0E0i4K6u0r3z5q4)9J5k6e0m8Q4x3X3f1H3i4K6g2X3M7U0c8Q4x3V1k6K6i4K6y4r3k6r3g2X3M7#2)9K6c8p5g2D9k6h3#2W2L8Y4c8Q4x3U0k6S2L8i4m8Q4x3@1u0H3M7X3!0B7k6h3y4@1i4K6y4p5L8r3W2T1j5$3!0J5k6b7`.`.)[] [dexElements](elink@bc7K9s2c8@1M7q4)9K6b7g2)9J5c8W2)9J5c8X3q4F1k6s2u0G2K9h3c8^5M7X3g2X3i4K6u0W2j5$3!0E0i4K6u0r3z5q4)9J5k6e0m8Q4x3X3f1H3i4K6g2X3M7U0c8Q4x3V1k6K6i4K6y4r3M7X3g2X3M7#2)9K6c8r3c8W2P5p5g2D9k6h3#2W2L8Y4c8K6i4K6t1$3j5h3#2H3i4K6y4n7M7s2u0G2K9X3g2U0N6q4)9K6c8r3I4A6j5X3y4G2M7X3f1`.);
> 
> 这个 field 是热修复插件经常使用的。通过对 Element 数组的对象进行合并达到效果。

### 参考

*   https://bbs.kanxue.com/thread-252630.htm
*   http://gityuan.com/2016/03/26/app-process-create/

抽取壳
---

上班的时候研究过这个，但是碍于当时水平有限，能够脱取抽取壳都无能为力，更别谈研发出来了。抽取壳最重要的就是抽取 codeitem 并在 app 运行的时候回填 codeitem。相当于是 Dex 壳加上 codeitem 抽取并回填。

 

主要讲解 codeitem 抽取回填

### codeitem 抽取

CodeItem 是 dex 文件中存放函数字节码相关数据的结构。下图显示的就是 CodeItem 大概的样子。  
![](https://bbs.kanxue.com/upload/attach/202301/754019_CTZY2PQZ5ET84FH.png)

 

说是提取 CodeItem，其实我们提取的是 CodeItem 中的 insns，它里面存放的是函数真正的字节码。提取 insns，我们使用的是 Android 源码中的 [dx](elink@024K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0W2k6$3!0G2k6$3I4W2M7$3!0#2M7X3y4W2i4K6u0W2j5$3!0E0i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6V1j5h3I4$3K9h3E0Q4x3V1k6Q4x3V1u0Q4x3V1k6J5k6h3k6K6i4K6u0r3K9r3g2S2k6s2y4Q4x3V1k6E0j5i4y4@1k6i4u0Q4x3V1k6V1P5q4)9J5c8R3`.`.) 工具，使用 dx 工具可以很方便的读取 dex 文件的各个部分。

 

下面的代码遍历所有 ClassDef，并遍历其中的所有函数，再调用 extractMethod 对单个函数进行处理。

```
public static List extractAllMethods(File dexFile, File outDexFile) {
    List instructionList = new ArrayList<>();
    Dex dex = null;
    RandomAccessFile randomAccessFile = null;
    byte[] dexData = IoUtils.readFile(dexFile.getAbsolutePath());
    IoUtils.writeFile(outDexFile.getAbsolutePath(),dexData);
 
    try {
        dex = new Dex(dexFile);
        randomAccessFile = new RandomAccessFile(outDexFile, "rw");
        Iterable classDefs = dex.classDefs();
        for (ClassDef classDef : classDefs) {
 
            ......
 
            if(classDef.getClassDataOffset() == 0){
                String log = String.format("class '%s' data offset is zero",classDef.toString());
                logger.warn(log);
                continue;
            }
 
            ClassData classData = dex.readClassData(classDef);
            ClassData.Method[] directMethods = classData.getDirectMethods();
            ClassData.Method[] virtualMethods = classData.getVirtualMethods();
            for (ClassData.Method method : directMethods) {
                Instruction instruction = extractMethod(dex,randomAccessFile,classDef,method);
                if(instruction != null) {
                    instructionList.add(instruction);
                }
            }
 
            for (ClassData.Method method : virtualMethods) {
                Instruction instruction = extractMethod(dex, randomAccessFile,classDef, method);
                if(instruction != null) {
                    instructionList.add(instruction);
                }
            }
        }
    }
    catch (Exception e){
        e.printStackTrace();
    }
    finally {
        IoUtils.close(randomAccessFile);
    }
 
    return instructionList;
} 
```

处理函数的过程中发现没有代码（通常为 native 函数）或者 insns 的容量不足以填充 return 语句则跳过处理。这里就是对应函数抽取壳的抽取操作

```
private static Instruction extractMethod(Dex dex ,RandomAccessFile outRandomAccessFile,ClassDef classDef,ClassData.Method method)
        throws Exception{
    String returnTypeName = dex.typeNames().get(dex.protoIds().get(dex.methodIds().get(method.getMethodIndex()).getProtoIndex()).getReturnTypeIndex());
    String methodName = dex.strings().get(dex.methodIds().get(method.getMethodIndex()).getNameIndex());
    String className = dex.typeNames().get(classDef.getTypeIndex());
    //native函数
    if(method.getCodeOffset() == 0){
        String log = String.format("method code offset is zero,name =  %s.%s , returnType = %s",
                TypeUtils.getHumanizeTypeName(className),
                methodName,
                TypeUtils.getHumanizeTypeName(returnTypeName));
        logger.warn(log);
        return null;
    }
    Instruction instruction = new Instruction();
    //16 = registers_size + ins_size + outs_size + tries_size + debug_info_off + insns_size
    int insnsOffset = method.getCodeOffset() + 16;
    Code code = dex.readCode(method);
    //容错处理
    if(code.getInstructions().length == 0){
        String log = String.format("method has no code,name =  %s.%s , returnType = %s",
                TypeUtils.getHumanizeTypeName(className),
                methodName,
                TypeUtils.getHumanizeTypeName(returnTypeName));
        logger.warn(log);
        return null;
    }
    int insnsCapacity = code.getInstructions().length;
    //insns容量不足以存放return语句，跳过
    byte[] returnByteCodes = getReturnByteCodes(returnTypeName);
    if(insnsCapacity * 2 < returnByteCodes.length){
        logger.warn("The capacity of insns is not enough to store the return statement. {}.{}() -> {} insnsCapacity = {}byte(s),returnByteCodes = {}byte(s)",
                TypeUtils.getHumanizeTypeName(className),
                methodName,
                TypeUtils.getHumanizeTypeName(returnTypeName),
                insnsCapacity * 2,
                returnByteCodes.length);
 
        return null;
    }
    instruction.setOffsetOfDex(insnsOffset);
    //这里的MethodIndex对应method_ids区的索引
    instruction.setMethodIndex(method.getMethodIndex());
    //注意：这里是数组的大小
    instruction.setInstructionDataSize(insnsCapacity * 2);
    byte[] byteCode = new byte[insnsCapacity * 2];
    //写入nop指令
    for (int i = 0; i < insnsCapacity; i++) {
        outRandomAccessFile.seek(insnsOffset + (i * 2));
        byteCode[i * 2] = outRandomAccessFile.readByte();
        byteCode[i * 2 + 1] = outRandomAccessFile.readByte();
        outRandomAccessFile.seek(insnsOffset + (i * 2));
        outRandomAccessFile.writeShort(0);
    }
    instruction.setInstructionsData(byteCode);
    outRandomAccessFile.seek(insnsOffset);
    //写出return语句
    outRandomAccessFile.write(returnByteCodes);
 
    return instruction;
}
```

### codeitem 回填

当一个类被加载的时候，它的调用链是这样的 (部分流程已省略)：

```
ClassLoader.java::loadClass -> DexPathList.java::findClass -> DexFile.java::defineClass ->
class_linker.cc::LoadClass -> class_linker.cc::LoadClassMembers -> class_linker.cc::LoadMethod
```

也就是说，当一个类被加载，它是会去调用 LoadMethod 函数的，我们看一下它的函数原型：

```
void ClassLinker::LoadMethod(const DexFile& dex_file,
                             const ClassDataItemIterator& it,
                             Handle klass,
                             ArtMethod* dst); 
```

这个函数太爆炸了，它有两个爆炸性的参数，DexFile 和 ClassDataItemIterator，我们可以从这个函数得到当前加载函数所在的 DexFile 结构和当前函数的一些信息，可以看一下 ClassDataItemIterator 结构：

```
  class ClassDataItemIterator{
 
  ......
 
  // A decoded version of the method of a class_data_item
  struct ClassDataMethod {
    uint32_t method_idx_delta_;  // delta of index into the method_ids array for MethodId
    uint32_t access_flags_;
    uint32_t code_off_;
    ClassDataMethod() : method_idx_delta_(0), access_flags_(0), code_off_(0) {}
 
   private:
    DISALLOW_COPY_AND_ASSIGN(ClassDataMethod);
  };
  ClassDataMethod method_;
 
  // Read and decode a method from a class_data_item stream into method
  void ReadClassDataMethod();
 
  const DexFile& dex_file_;
  size_t pos_;  // integral number of items passed
  const uint8_t* ptr_pos_;  // pointer into stream of class_data_item
  uint32_t last_idx_;  // last read field or method index to apply delta to
  DISALLOW_IMPLICIT_CONSTRUCTORS(ClassDataItemIterator);
};
```

其中最重要的字段就是`code_off_`它的值是当前加载的函数的 CodeItem 相对于 DexFile 的偏移，当相应的函数被加载，我们就可以直接访问到它的 CodeItem。其他函数是否也可以？在上面的流程中没有比 LoadMethod 更适合我们 Hook 的函数，所以它是最佳的 Hook 点。

 

Hook LoadMethod 稍微复杂一些，倒不是 Hook 代码复杂，而是 Hook 触发后处理的代码比较复杂，我们要适配多个 Android 版本，每个版本 LoadMethod 函数的参数都可能有改变，幸运的是，LoadMethod 改动也不是很大。那么，我们如何读取 ClassDataItemIterator 类中的`code_off_`呢？比较直接的做法是计算偏移，然后在代码中维护一份偏移。不过这样的做法不易阅读很容易出错。dpt 的做法是把 ClassDataItemIterator 类拷过来，然后将 ClassDataItemIterator 引用直接转换为我们自定义的 ClassDataItemIterator 引用，这样就可以方便的读取字段的值。

 

下面是 LoadMethod 被调用后做的操作，逻辑是读取存在 map 中的 insns，然后将它们填回指定位置。

```
void LoadMethod(void *thiz, void *self, const void *dex_file, const void *it, const void *method,
                void *klass, void *dst) {
 
    if (g_originLoadMethod25 != nullptr
        || g_originLoadMethod28 != nullptr
        || g_originLoadMethod29 != nullptr) {
        uint32_t location_offset = getDexFileLocationOffset();
        uint32_t begin_offset = getDataItemCodeItemOffset();
        callOriginLoadMethod(thiz, self, dex_file, it, method, klass, dst);
 
        ClassDataItemReader *classDataItemReader = getClassDataItemReader(it,method);
 
 
        uint8_t **begin_ptr = (uint8_t **) ((uint8_t *) dex_file + begin_offset);
        uint8_t *begin = *begin_ptr;
        // vtable(4|8) + prev_fields_size
        std::string *location = (reinterpret_cast((uint8_t *) dex_file +
                                                                 location_offset));
        if (location->find("base.apk") != std::string::npos) {
 
            //code_item_offset == 0说明是native方法或者没有代码
            if (classDataItemReader->GetMethodCodeItemOffset() == 0) {
                DLOGW("native method? = %s code_item_offset = 0x%x",
                      classDataItemReader->MemberIsNative() ? "true" : "false",
                      classDataItemReader->GetMethodCodeItemOffset());
                return;
            }
 
            uint16_t firstDvmCode = *((uint16_t*)(begin + classDataItemReader->GetMethodCodeItemOffset() + 16));
            if(firstDvmCode != 0x0012 && firstDvmCode != 0x0016 && firstDvmCode != 0x000e){
                NLOG("this method has code no need to patch");
                return;
            }
 
            uint32_t dexSize = *((uint32_t*)(begin + 0x20));
 
            int dexIndex = dexNumber(location);
            auto dexIt = dexMap.find(dexIndex - 1);
            if (dexIt != dexMap.end()) {
 
                auto dexMemIt = dexMemMap.find(dexIndex);
                if(dexMemIt == dexMemMap.end()){
                    changeDexProtect(begin,location->c_str(),dexSize,dexIndex);
                }
 
 
                auto codeItemMap = dexIt->second;
                int methodIdx = classDataItemReader->GetMemberIndex();
                auto codeItemIt = codeItemMap->find(methodIdx);
 
                if (codeItemIt != codeItemMap->end()) {
                    CodeItem* codeItem = codeItemIt->second;
                    uint8_t  *realCodeItemPtr = (uint8_t*)(begin +
                                                classDataItemReader->GetMethodCodeItemOffset() +
                                                16);
 
                    memcpy(realCodeItemPtr,codeItem->getInsns(),codeItem->getInsnsSize());
                }
            }
        }
    }
} 
```

#### 参考

*   https://www.cnblogs.com/luoyesiqiu/p/dpt.html
*   https://github.com/luoyesiqiu/dpt-shell

Java2C
------

关键 Java 函数 native 化。

### dcc 的使用方式

1.  首先在 app 代码合适的位置, 如 Application 的静态代码块或 onCreate 等, 添加加载 so 库代码, 并重新生成 apk

```
try {
    System.loadLibrary("nc");
} catch (UnsatisfiedLinkError e) {
    e.printStackTrace();
}
```

1.  制定编译方式

#### 使用黑白名单

dcc 支持使用黑白名单来过滤需要编译或禁止编译的函数. 修改 filter.txt, 使用正则表达式配置需要处理的函数. 默认编译 Activity.onCreate, 和测试 demo 中的所有函数.

> vi filter.txt

#### 使用注解

在任意包中新增 Dex2C 注解类

```
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
 
@Retention(RetentionPolicy.RUNTIME)
public @interface Dex2C {}
```

然后使用 Dex2C 标记需要编译的类或者方法

#### 加固 APP

> python3 dcc.py your_app.apk -o out.apk

### 分析

捡重点的说

 

核心代码，Java 2 C 在`dcc.py`中的`compile_dex`函数

```
def compile_dex(apkfile, filtercfg):
    show_logging(level=logging.INFO)
 
    d = auto_vm(apkfile)
    dx = analysis.Analysis(d)
 
    method_filter = MethodFilter(filtercfg, d)
 
    compiler = Dex2C(d, dx)
 
    compiled_method_code = {}
    errors = []
 
    for m in d.get_methods():
        method_triple = get_method_triple(m)
 
        jni_longname = JniLongName(*method_triple)
        full_name = ''.join(method_triple)
 
        if len(jni_longname) > 220:
            logger.debug("name to long %s(> 220) %s" % (jni_longname, full_name))
            continue
 
        if method_filter.should_compile(m):
            logger.debug("compiling %s" % (full_name))
            try:
                code = compiler.get_source_method(m)
            except Exception as e:
                logger.warning("compile method failed:%s (%s)" % (full_name, str(e)), exc_info=True)
                errors.append('%s:%s' % (full_name, str(e)))
                continue
 
            if code:
                compiled_method_code[method_triple] = code
 
    return compiled_method_code, errors
```

其中的 compiled_method_code 产生过程最重要，因为这是 Java 代码转 C 代码的实际部分，最后 dcc 会讲转成的 C 代码写入以目标类名为名字的. cpp 文件中。

 

首先会对 dex 文件中的 classes、methods、strings、fields 做分析；过滤出要转化的函数，再将函数抽离出来转化成 C 的代码，保留成 C 代码的字符串，交给 code。

 

然后再将保存好的 cpp 文件，与前面提到的预设的加载 so nc 的源码一起编译成 so，再重打包出 apk。

### 建议或者优化的点

整个转化的代码大部分都是 python 写的，在回顾上问的抽取壳，完全可以整合在一块，抽取出来后，在对其内容进行转化，dcc 的作者专门做了个 ir 来做转化，可以说很棒了，但是完全可以整合在一起，这样就是一个加固产品了，就是无法保证稳定性之类的问题。

### 参考

*   https://github.com/amimo/dcc

DexVMP
------

抽取 smali 指令变成 VM 的 opcode，在 vm 中运行。

 

可以参考 Java2C 的过程，无非就是转变的对象不一样，Java2C 是转变成 C 代码，DexVMP 就是转变成 vmp 的 opcode，再配合相应位置的调用即可实现 opcode 的 vm 解释器进行解释执行。

 

参考的 nmmp 项目就是一个 DexVMP 加固项目，实现上跟上面说的一致，具体实现细节可以阅读源码了解，vm 太大，不做过多展开。

### 参考

*   https://github.com/maoabc/nmmp.git

SO 壳
====

外部加固 - UPX 壳
------------

upx 壳是一种以压缩为手段的壳，对二进制文件压缩后，执行的时候释放，让静态反编译工具反编译他无从下手，从而实现保护效果。

### 压缩部分

`elf header`：这部分保持跟原 so 文件相同，不过里面部分数据需要修改，比如 section headers 的偏移等。

 

`program headers`：该部分会对第一个 LOAD 段的 size 大小做修改（因为使用了 upx 壳后体积变小了），其他数据保持不变。

 

`sections`：这里存放着原 so 中的节，这些节包括："DT_GNU_HASH","DT_HASH", "DT_STRTAB","DT_SYMTAB","DT_REL","DT_RELA","DT_JMPREL","DT_VERDEF","DT_VERSYM","DT_VERNEEDED"，因为在 so 中跟重定位相关的节不能够被加壳，并且在文件中的偏移位置需要保持不变，所以一般情况下是在. text 节之后开始做压缩处理。

 

`section headers`：upx 压缩壳会将 section headers 表上移，放到上一部分的 sections 后面，并且不做任何压缩处理，原因是在高版本的 android 系统中会对 section headers 做校验，需要读取 section headers 中的数据，因此需要以明文的形式存放在 so 中。

 

`.shstrtab`：这个节用于存放各个 section 的名字。

 

`l_info`：为一个结构体，其格式如下，其中 magic 为’7fUPX’，各个产商会对此做修改。

```
{
LE32 l_checksum;
LE32 l_magic;
LE16 l_size;
unsigned char l_version;
unsigned char l_format;
}
```

`p_info`：为一个结构体，其格式如下，

```
{
unsigned p_progid;
unsigned p_filesize;
unsigned p_blocksize;
}
```

`compressed data`：被压缩的数据，即实现对代码段做加密。

 

`b_info`：为一个结构体，其格式如下，其中 sz_unc 为压缩后数据的大小，sz_cpr 为压缩前数据的大小，b_method 为使用的压缩方法。

```
{
unsigned sz_unc;
unsigned sz_cpr;
unsigned char b_method;
unsigned char b_ftid;
unsigned char b_cto8;
unsigned char b_unused;
}
```

`other info`：保存着可以传递给 stub 的参数，其中包括：dt_init 地址，xct_off（sections 中最后的地址）等

 

`stub`：为整个 upx 的核心部分，该部分用于运行时解压并还原 compressed data。

 

`2th LOAD`：为 so 的第二个 LOAD 段，因为第二个 LOAD 段中的 dynamic 节涉及重定位环节，所以 upx 无法对其做压缩处理。

 

`PackHeader`：为 upx 的尾部，保存一些 upx 相关信息。  
![](https://bbs.kanxue.com/upload/attach/202301/754019_S5AU72PUPV8DUM7.png)

### 解压部分

在 upx 的 stub 中主要有三个部分，分别为：main、supervise、hatch。

1.  在 main 中使用 mmap 开辟新空间 A，将 compressed data、supervise、hatch 拷贝到该空间 A 中。并将 pc 指针指向 A 中的 supervise 开始执行指令。
2.  supervise 中包含这解压过程中所需要的代码包括： 解压算法 decompress、过滤器 unfilter、其他。在该部分中使用 mmap 并使用 MAP_FIXED 参数来开辟内存，该内存地址为原 so 加载时的空间地址，MAP_FIXED 这个参数能够覆盖原来 mmap 的内存空间，确保解压后数据与原 so 保持一致。对 compressed data 做解压处理，并将 hatch 拷贝到第一个 LOAD 末尾（多余空间）。在拷贝以及解压完成后，将 pc 指针跳转到 hatch，执行 hatch。
3.  hatch 用于 unmap 释放在 1 中开辟的空间 A，并跳转到原 so 的 dt_init 中，如果原 so 没有 dt_init 则跳转到 linker 中。

该过程流程图如下：

 

![](https://bbs.kanxue.com/upload/attach/202301/754019_2PN8SZZ5YPFAHF5.png)

### 参考

*   https://bbs.kanxue.com/thread-248779.htm
*   https://github.com/upx/upx

内部加固 - VMP 壳
------------

借助 LLVM 编译工具，对 native 代码关键部分做到 VMP 保护，代码抽取成 vm opcode，在原来调用的地方替换成 vm 接口，实际代码转化成 opcode 在 vm 中运行。

 

没有开源项目先不说了。

### 参考

*   [[译文]VMProtect2 - 虚拟机架构的详细分析](elink@f04K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6@1N6h3N6G2K9r3!0K6N6q4)9J5k6h3N6A6N6r3S2#2j5W2)9J5k6h3W2G2i4K6u0r3x3U0l9J5x3W2)9J5c8U0p5J5i4K6u0r3x3o6k6Q4x3V1k6h3e0g2m8J5L8%4c8W2j5%4b7J5i4K6g2X3k6r3g2@1j5h3W2D9k6h3c8Q4y4h3k6S2L8X3q4D9P5i4y4A6M7#2)9#2k6X3!0X3i4K6g2X3N6r3S2W2i4K6g2X3N6X3W2J5N6s2g2S2L8q4)9#2k6X3#2S2j5$3S2A6L8X3g2Q4y4h3k6S2M7X3y4Z5K9i4c8W2j5%4c8#2M7X3g2Q4x3V1j5`.)

[[招生] 科锐逆向工程师培训 (2025 年 3 月 11 日实地，远程教学同时开班, 第 52 期)！](https://bbs.kanxue.com/thread-51839.htm)

最后于 2023-1-18 11:38 被 TUGOhost 编辑 ，原因：

[#其他](forum-161-1-129.htm)