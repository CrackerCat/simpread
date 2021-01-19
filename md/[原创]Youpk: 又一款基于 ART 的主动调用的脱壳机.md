> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-259854-1.htm)

Youpk: 又一款基于 ART 的主动调用的脱壳机
==========================

原理
--

Youpk 是一款针对 Dex 整体加固 + 各式各样的 Dex 抽取的脱壳机

 

基本流程如下:

1.  从内存中 dump DEX
2.  构造完整调用链, 主动调用所有方法并 dump CodeItem
3.  合并 DEX, CodeItem

### 从内存中 dump DEX

DEX 文件在 art 虚拟机中使用 DexFile 对象表示, 而 ClassLinker 中引用了这些对象, 因此可以采用从 ClassLinker 中遍历 DexFile 对象并 dump 的方式来获取.

```
//unpacker.cc
std::list Unpacker::getDexFiles() {
  std::list dex_files;
  Thread* const self = Thread::Current();
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
  ReaderMutexLock mu(self, *class_linker->DexLock());
  const std::list& dex_caches = class_linker->GetDexCachesData();
  for (auto it = dex_caches.begin(); it != dex_caches.end(); ++it) {
    ClassLinker::DexCacheData data = *it;
    const DexFile* dex_file = data.dex_file;
    dex_files.push_back(dex_file);
  }
  return dex_files;
} 
```

另外, 为了避免 dex 做任何形式的优化影响 dump 下来的 dex 文件, 在 dex2oat 中设置 CompilerFilter 为仅验证

```
//dex2oat.cc
compiler_options_->SetCompilerFilter(CompilerFilter::kVerifyAtRuntime);

```

### 构造完整调用链, 主动调用所有方法

1.  创建脱壳线程
    
    ```
    //unpacker.java
    public static void unpack() {
        if (Unpacker.unpackerThread != null) {
            return;
        }
     
        //开启线程调用
        Unpacker.unpackerThread = new Thread() {
            @Override public void run() {
                while (true) {
                    try {
                        Thread.sleep(UNPACK_INTERVAL);
                    }
                    catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    if (shouldUnpack()) {
                        Unpacker.unpackNative();
                    }  
                }
            }
        };
        Unpacker.unpackerThread.start();
    }
    
    ```
    
2.  在脱壳线程中遍历 DexFile 的所有 ClassDef
    
    ```
    //unpacker.cc
    for (; class_idx < dex_file->NumClassDefs(); class_idx++) {
    
    ```
    
3.  解析并初始化 Class
    
    ```
    //unpacker.cc
    mirror::Class* klass = class_linker->ResolveType(*dex_file, dex_file->GetClassDef(class_idx).class_idx_, h_dex_cache, h_class_loader);
    StackHandleScope<1> hs2(self);
    Handle h_class(hs2.NewHandle(klass));
    bool suc = class_linker->EnsureInitialized(self, h_class, true, true); 
    ```
    
4.  主动调用 Class 的所有 Method, 并修改 ArtMethod::Invoke 使其强制走 switch 型解释器
    
    ```
    //unpacker.cc
    uint32_t args_size = (uint32_t)ArtMethod::NumArgRegisters(method->GetShorty());
    if (!method->IsStatic()) {
        args_size += 1;
    }
     
    JValue result;
    std::vector args(args_size, 0);
    if (!method->IsStatic()) {
        mirror::Object* thiz = klass->AllocObject(self);
        args[0] = StackReference::FromMirrorPtr(thiz).AsVRegValue(); 
    }
    method->Invoke(self, args.data(), args_size, &result, method->GetShorty());
     
    //art_method.cc
    if (UNLIKELY(!runtime->IsStarted() || Dbg::IsForcedInterpreterNeededForCalling(self, this)
    || (Unpacker::isFakeInvoke(self, this) && !this->IsNative()))) {
    if (IsStatic()) {
    art::interpreter::EnterInterpreterFromInvoke(
    self, this, nullptr, args, result, /*stay_in_interpreter*/ true);
    } else {
    mirror::Object* receiver =
    reinterpret_cast*>(&args[0])->AsMirrorPtr();
    art::interpreter::EnterInterpreterFromInvoke(
    self, this, receiver, args + 1, result, /*stay_in_interpreter*/ true);
    }
    }
     
    //interpreter.cc
    static constexpr InterpreterImplKind kInterpreterImplKind = kSwitchImplKind; 
    ```
    
5.  在解释器中插桩, 在每条指令执行前设置回调
    
    ```
    //interpreter_switch_impl.cc
    // Code to run before each dex instruction.
      #define PREAMBLE()                                                                 \
      do {                                                                               \
        inst_count++;                                                                    \
        bool dumped = Unpacker::beforeInstructionExecute(self, shadow_frame.GetMethod(), \
                                                         dex_pc, inst_count);            \
        if (dumped) {                                                                    \
          return JValue();                                                               \
        }                                                                                \
        if (UNLIKELY(instrumentation->HasDexPcListeners())) {                            \
          instrumentation->DexPcMovedEvent(self, shadow_frame.GetThisObject(code_item->ins_size_),  shadow_frame.GetMethod(), dex_pc);                                                                                  \
        }                                                                                \
      } while (false)
    
    ```
    
6.  在回调中做针对性的 CodeItem 的 dump, 这里仅仅是简单的示例了直接 dump, 实际上, 针对某些厂商的抽取, 可以真正的执行几条指令等待 CodeItem 解密后再 dump
    
    ```
    //unpacker.cc
    bool Unpacker::beforeInstructionExecute(Thread *self, ArtMethod *method, uint32_t dex_pc, int inst_count) {
      if (Unpacker::isFakeInvoke(self, method)) {
          Unpacker::dumpMethod(method);
        return true;
      }
      return false;
    }
    
    ```
    

### 合并 DEX, CodeItem

将 dump 下来的 CodeItem 填充到 DEX 的相应位置中即可. 主要是基于 google dx 工具修改.

### 参考链接

FUPK3: https://bbs.pediy.com/thread-246117.htm

 

FART: https://bbs.pediy.com/thread-252630.htm

刷机
--

1.  仅支持 pixel 1 代
2.  重启至 bootloader: `adb reboot bootloader`
3.  解压 Youpk_sailfish.zip 并双击 `flash-all.bat`  
    百度云：https://pan.baidu.com/s/1ySSy2vNW5TyFjH1LNAcd5w  
    提取码：vseh

使用方法
----

1.  **该工具仅仅用来学习交流, 请勿用于非法用途, 否则后果自付！**
    
2.  配置待脱壳的 app 包名, 准确来讲是进程名称
    
    ```
    adb shell "echo cn.youlor.mydemo >> /data/local/tmp/unpacker.config"
    
    ```
    
3.  启动 apk 等待脱壳  
    每隔 10 秒将自动重新脱壳 (已完全 dump 的 dex 将被忽略), 当日志打印 unpack end 时脱壳完成
    
4.  pull 出 dump 文件, dump 文件路径为 `/data/data/包名/unpacker`
    
    ```
    adb pull /data/data/cn.youlor.mydemo/unpacker
    
    ```
    
5.  调用修复工具 dexfixer.jar, 两个参数, 第一个为 dump 文件目录 (必须为有效路径), 第二个为重组后的 DEX 目录 (不存在将会创建)
    
    ```
    java -jar dexfixer.jar /path/to/unpacker /path/to/output
    
    ```
    

适用场景
----

1.  整体加固
2.  抽取:
    *   nop 占坑型 (类似某加密)
    *   naitve 化, 在 <clinit> 中解密 (类似早期阿里)
    *   goto 解密型 (类似新版某加密? najia): https://bbs.pediy.com/thread-259448.htm

常见问题
----

1.  dump 中途退出或卡死，重新启动进程，再次等待脱壳即可
2.  当前仅支持被壳保护的 dex, 不支持 App 动态加载的 dex/jar

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)

最后于 2020-5-31 11:41 被 Youlor 编辑 ，原因： 补充