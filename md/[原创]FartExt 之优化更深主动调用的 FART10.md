> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268760.htm)

> [原创]FartExt 之优化更深主动调用的 FART10

### 参考

> [将 FART 和 Youpk 结合来做一次针对函数抽取壳的全面提升](https://bbs.pediy.com/thread-260052.htm)
> 
> 看雪高研班课程

 

寒冰大佬的 FART 带动了不少新的主动调用思想的抽取壳方案。看了上面这篇文章，感觉意犹未尽，当然是要动手实践一翻来优化一波。于是我重新翻阅 FART 和 Youpk 的源码，我准备和那位大佬一样，参考 Youpk 在 FART 的基础上进行升级改造。开工前先明确出我的需求。

### 需求

> *   对指定进程脱壳，非目标进程不要执行脱壳线程。
> *   对指定类列表进行脱壳
> *   将 FART 升级到 aosp10 实现。
> *   去 FART 指纹
> *   FART 保存出的函数修复合并为 dex。
> *   实现 FART 更深的主动调用
> *   更快的主动调用（暂未优化）

[](#一、什么是fart)一、什么是 FART
------------------------

还未了解过的请看原作者对于 fart 的介绍

> [FART：ART 环境下基于主动调用的自动化脱壳方案](https://bbs.pediy.com/thread-252630.htm)
> 
> [FART 正餐前甜点：ART 下几个通用简单高效的 dump 内存中 dex 方法](https://bbs.pediy.com/thread-254028.htm)
> 
> [拨云见日：安卓 APP 脱壳的本质以及如何快速发现 ART 下的脱壳点](https://bbs.pediy.com/thread-254555.htm)

 

关于 fart 源码的调用流程可以看看我以前整理的一篇文章

> [fart 的理解和分析过程](https://bbs.pediy.com/thread-263401.htm)

 

**简单总结：**

 

这是一个基于主动调用来脱抽取壳的方案。

 

**简述实现原理：**

 

在进程启动的时候通过双亲委派机制遍历所有 classloader，然后遍历里面的所有 class，取出所有函数，直接调用。然后在 ArtMethod 的 Invoke 函数这里根据参数判断出这是主动调用触发的，然后就取消函数的正常执行，并执行脱壳操作。

[](#二、对指定进程脱壳)二、对指定进程脱壳
-----------------------

首先看看 FART 源码中，脱壳线程的入口点 ActivityThread.java 的 performLaunchActivity 这个函数中开始的 FART 处理。

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        fartthread();
              ...
    }

```

也就是说，所有的进程都会执行脱壳的流程。所以这个地方我觉得还是有必要优化的。应该是对指定的进程脱壳会更加好一些。Youpk 中已经有了这个优化。所以我直接拿 Youpk 的处理来使用。下面是修改后的入口启动部分

```
//判断这个进程是否应该脱壳
    public static boolean shouldUnpack() {
    boolean should_unpack = false;
    String processName = ActivityThread.currentProcessName();
    BufferedReader br = null;
    String configPath="/data/local/tmp/fext.config";
    try {
        br = new BufferedReader(new FileReader(configPath));
        String line;
        while ((line = br.readLine()) != null) {
            if (processName.equals(line))) {
                should_unpack = true;
                break;
            }
        }
        br.close();
    }
    catch (Exception ignored) {
 
    }
    return should_unpack;
}
    //启动FART脱壳线程
public static void fartthread() {
 
    if (!shouldUnpack()) {
        return;
    }
 
    new Thread(new Runnable() {
        @Override
        public void run() {
            // TODO Auto-generated method stub
            try {
                Log.e("ActivityThread", "start sleep......");
                Thread.sleep(1 * 60 * 1000);
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
            Log.e("ActivityThread", "sleep over and start fart");
            fart();
            Log.e("ActivityThread", "fart run over");
 
        }
    }).start();
}
private void handleBindApplication(AppBindData data) {
    ...
    app = data.info.makeApplication(data.restrictedBackupMode, null);
      app.setAutofillOptions(data.autofillOptions);
      app.setContentCaptureOptions(data.contentCaptureOptions);
      mInitialApplication = app;
      fartthread();
            ...
}

```

**这里主要是做了两点修改：**

 

1、把启动 FART 线程的处理放到了 handleBindApplication 函数，这是因为 performLaunchActivity 这个函数调用有的时候可能会触发两次，而 handleBindApplication 确定只会触发一次。

 

2、FART 线程启动前加了个判断，在配置文件中的进程才需要脱壳。这样基本第一个优化就完成了。

[](#三、对指定类列表进行脱壳)三、对指定类列表进行脱壳
-----------------------------

由于应用的有些类可能是被特殊处理了，主动调用的情况会导致程序崩溃或者退出。所以最好是可以单独对某些类进行主动调用。我调整了 FART 线程启动的逻辑，先是上面的判断是不是要脱壳的目标进程，然后判断有没有设定类列表，如果有类列表就只脱壳类列表，否则就完整主动调用

```
//读取类列表
public static String getClassList() {
        String processName = ActivityThread.currentProcessName();
        BufferedReader br = null;
        String configPath="/data/local/tmp/"+processName;
        Log.e("ActivityThread", "getClassList processName:"+processName);
        StringBuilder sb=new StringBuilder();
        try {
            br = new BufferedReader(new FileReader(configPath));
            String line;
            while ((line = br.readLine()) != null) {
 
                if(line.length()>=2){
                    sb.append(line+"\n");
                }
            }
            br.close();
        }
        catch (Exception ex) {
            Log.e("ActivityThread", "getClassList err:"+ex.getMessage());
            return "";
        }
        return sb.toString();
    }
        //对指定类进行主动调用
    public static void fartWithClassList(String classlist){
        ClassLoader appClassloader = getClassloader();
        if(appClassloader==null){
            Log.e("ActivityThread", "appClassloader is null");
            return;
        }
        Class DexFileClazz = null;
        try {
            DexFileClazz = appClassloader.loadClass("dalvik.system.DexFile");
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
        Method dumpMethodCode_method = null;
        for (Method field : DexFileClazz.getDeclaredMethods()) {
            if (field.getName().equals("fartextMethodCode")) {
                dumpMethodCode_method = field;
                dumpMethodCode_method.setAccessible(true);
            }
        }
        String[] classes=classlist.split("\n");
        for(String clsname : classes){
            String line=clsname;
            if(line.startsWith("L")&&line.endsWith(";")&&line.contains("/")){
                line=line.substring(1,line.length()-1);
                line=line.replace("/",".");
            }
            loadClassAndInvoke(appClassloader, line, dumpMethodCode_method);
        }
    }
 
    public static void fartthread() {
        if (!shouldUnpack()) {
            return;
        }
          //获取类列表。如果有的话就不要完整主动调用了
        String classlist=getClassList();
        if(!classlist.equals("")){
            fartWithClassList(classlist);
            return;
        }
                ...
    }

```

[](#四、fart升级aosp10)四、FART 升级 AOSP10
-----------------------------------

直接将 FART 修改的代码部分直接替换到 AOSP10 中。毫不意外的出现了一堆错误。不过问题比较集中。主要是对于 CodeItem 的成员访问方式发生了变化。这里可以参考下面的文章

> [Android ART 虚拟机 - dex 文件格式要旨](https://www.jianshu.com/p/18ff0b8e0b01)

 

根据这篇文章中对 CodeItem 对象新的访问方式。对 FART 的源码部分做出修改

 

修改文件是 art_method.cc。我这里只贴上部分关键修改的代码

```
extern "C" void dumpArtMethod(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
                ...
        const dex::CodeItem* code_item = artmethod->GetCodeItem();
              const DexFile* dex_=artmethod->GetDexFile();
              CodeItemDataAccessor accessor(*dex_, dex_->GetCodeItem(artmethod->GetCodeItemOffset()));
              if (LIKELY(code_item != nullptr))
        {
              int code_item_len = 0;
              uint8_t *item=(uint8_t *) code_item;
              if (accessor.TriesSize()>0) {
                  const uint8_t *handler_data = accessor.GetCatchHandlerData();
                  uint8_t * tail = codeitem_end(&handler_data);
                  code_item_len = (int)(tail - item);
            }else{
                  code_item_len = 16+accessor.InsnsSizeInCodeUnits()*2;
            }
                        ...
            }
            ...
}

```

需要修改的不止上面这几个地方。但是主要都是针对 CodeItem 的使用以及命名空间的修改。这里我就不全部贴出了。最后我会贴出改完后的版本。

[](#五、去fart指纹)五、去 FART 指纹
-------------------------

由于 FART 的知名度还是挺高的，所以最好还是把 FART 中特有的一些函数名和文件保存路径给修改一下。下面整理下我参考 Youpk 做的一些修改。

 

1、将 ActivityThread 中的 FART 相关函数全部单独放到一个类 cn.mik.Fartext 中。这样如果别人对 ActivityThread 的函数检测就找不到 FART 相关的了。

 

2、将 DexFile 中的 dumpMethodCode 函数名修改为 fartextMethodCode

 

3、将 myfartInvoke 函数名改成 fartextInvoke

 

4、将所有使用 / sdcard/fart 的这个路径全部修改成 / sdcard/fext

 

把这些常见的可能识别的方式都修改之后。一般就识别不出来了。我这种完全没知名度的，想必不会被人检测到了。暗自欣喜。

[](#六、fart的函数修复)六、FART 的函数修复
----------------------------

抽取壳的应用在脱壳后，有两种文件，一个是在当前时机 dump 出来的 dex 文件。另一个是保存 codeitem 出来的 bin 文件。

 

FART 的修复组件是使用开源项目 [FART](https://github.com/hanbinglengyue/FART) 中那个 py 的脚本来解析 dex 文件，将 bin 的 codeitem 修复打印。对于里面的代码解析部分我之前也写了文章。感兴趣可以看看

> [dex 起步探索](https://bbs.pediy.com/thread-268465.htm)

 

但是我仔细研究后，发现修复组件只进行了打印。并没有修复成 dex，而是直接解析打印。最理想的还是修复到 dex，方便使用静态分析工具查看。有大佬也已经写了这个工具。那就是前面参考的 Youpk。他的内部有个 dexfixer 的目录，就是实现了对导出的 codeitem 数据修复到 dex 中。不过他的 codeitem 的保存结构和 FART 的并不大一样。不过没关系，修改一下 codeitem 文件的解析部分就好了。下面贴上 Youpk 和我修改后专门处理 fart 结果的 dexfixer。

> [Youpk](https://github.com/Youlor/Youpk)
> 
> [dexfixer](https://github.com/dqzg12300/dexfixer)

[](#七、更深的主动调用)七、更深的主动调用
-----------------------

### FART 的主动调用深度

首先当然是看看 FART 的主动调用的深度是在哪里，这里的深度其实就是在函数的主动执行过程中，FART 是在执行到哪个流程时，进行的脱壳处理。下面贴上 FART 主动调用的脱壳位置

```
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
    if (self== nullptr) {
        dumpArtMethod(this);
        return;
    }
      ...
 }

```

上面可以看到。当函数执行流程到达 ArtMethod::Invoke 时，就根据参数判断是主动调用的情况，就脱壳并结束了。

### 为什么需要更深的主动调用

一般的函数抽取壳，在执行到 ArtMethod::Invoke 前就已经对抽取函数还原了。但是也有一些抽取壳，执行到 Invoke 时依然还没有还原函数。譬如下面这种抽取壳。

```
.method public constructor ()V
    .registers 2
 
    goto :goto_c
 
    :goto_1
    nop
 
    nop
 
    nop
 
    nop
 
    nop
 
    nop
 
    nop
 
    nop
 
    nop
 
    nop
 
    return-void
 
    :goto_c
    const v0, 0x1669
 
    invoke-static {v0}, Ls/h/e/l/l/H;->i(I)V
 
    goto :goto_1
.end method 
```

可以看到这个抽取的函数，进来之后就 goto，然后执行 invoke-static。接着在 goto 到函数的开始位置。

 

也就是说。这个抽取壳，必须在函数执行了之后，才会还原出真实的函数。回想一下前面说的 FART 的主动调用深度。发现函数真正执行前就已经被我们直接结束掉了。所以我们需要更深的主动调用才能够解决这个抽取壳。

### Youpk 更深的主动调用

我们回头看看上面的抽取壳，我们的目标是要判断如果这个函数的第一个指令是 goto，就正常执行，然后执行到 invoke-static 的指令。这个指令完成之后就直接结束掉函数调用。避免真实函数调用会出现异常。

 

先参考 Youpk 的看看他是如何实现更深的主动调用来解决这个问题的。下面是第一步，先修改默认的解释器为 Switch 的解释器。这是因为 Switch 解释器的可读性更加高，方便我们直接修改源码来达到目的。

```
static constexpr InterpreterImplKind kInterpreterImplKind = kSwitchImplKind;

```

然后我们看看主动调用时 Youpk 是怎么模拟参数的。

```
void Unpacker::invokeAllMethods() {
          ...
      auto methods = klass->GetDeclaredMethods(pointer_size);
      Unpacker::enableFakeInvoke();
      for (auto& m : methods) {
        ArtMethod* method = &m;
        if (!method->IsProxyMethod() && method->IsInvokable()) {
          //获取参数个数
          uint32_t args_size = (uint32_t)ArtMethod::NumArgRegisters(method->GetShorty());
          if (!method->IsStatic()) {
            args_size += 1;
          }
          //模拟参数
          JValue result;
          std::vector args(args_size, 0);
          if (!method->IsStatic()) {
            mirror::Object* thiz = klass->AllocObject(self);
            args[0] = StackReference::FromMirrorPtr(thiz).AsVRegValue(); 
          }
          method->Invoke(self, args.data(), args_size, &result, method->GetShorty());
        }
      }
      Unpacker::disableFakeInvoke();
      cJSON_ReplaceItemInObject(current, "status", cJSON_CreateString("Dumped"));
      writeJson();
    }
  }
} 
```

这里可以看到 Youpk 的参数是模拟赋值进去的。而寒冰大佬的做法不大一样。看看 FART 的函数调用模拟。

```
extern "C" void myfartInvoke(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
    JValue *result=nullptr;
    Thread *self=nullptr;
    uint32_t temp=6;
    uint32_t* args=&temp;
    uint32_t args_size=6;
    artmethod->Invoke(self, args, args_size, result, "fart");
}

```

这样肯定没法顺利往后执行。我们先继续参考 Youpk 的后续。

 

然后看看 Youpk 的 ArtMethod::Invoke 的处理，如果是主动调用并且非 Native 函数就正常执行。

```
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
  ...
  //patch by Youlor
  //++++++++++++++++++++++++++++
  //如果是主动调用fake invoke并且不是native方法则强制走解释器
  if (UNLIKELY(!runtime->IsStarted() || Dbg::IsForcedInterpreterNeededForCalling(self, this)
      || (Unpacker::isFakeInvoke(self, this) && !this->IsNative()))) {
  //++++++++++++++++++++++++++++
    if (IsStatic()) {
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, nullptr, args, result, /*stay_in_interpreter*/ true);
    } else {
      mirror::Object* receiver =
          reinterpret_cast*>(&args[0])->AsMirrorPtr();
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, receiver, args + 1, result, /*stay_in_interpreter*/ true);
    }
  } else {
    //patch by Youlor
    //++++++++++++++++++++++++++++
    //如果是主动调用fake invoke并且是native方法则不执行
    if (Unpacker::isFakeInvoke(self, this) && this->IsNative()) {
      // Pop transition.
      self->PopManagedStackFragment(fragment);
      return;
    }
    //++++++++++++++++++++++++++++
    ...
  }
    ...
} 
```

接下来看解释器的 EnterInterpreterFromInvoke 函数处理。这里 Youpk 没有什么处理。

```
void EnterInterpreterFromInvoke(Thread* self, ArtMethod* method, Object* receiver,
                                uint32_t* args, JValue* result,
                                bool stay_in_interpreter) {
         ...
    JValue r = Execute(self, code_item, *shadow_frame, JValue(), stay_in_interpreter);
    ...
}

```

继续看看函数 Execute。

```
static inline JValue Execute(
    Thread* self,
    const DexFile::CodeItem* code_item,
    ShadowFrame& shadow_frame,
    JValue result_register,
    bool stay_in_interpreter = false) SHARED_REQUIRES(Locks::mutator_lock_) {
      ...
    } else if (kInterpreterImplKind == kSwitchImplKind) {
      if (transaction_active) {
        return ExecuteSwitchImpl(self, code_item, shadow_frame, result_register,
                                              false);
      } else {
        return ExecuteSwitchImpl(self, code_item, shadow_frame, result_register,
                                               false);
      }
    }
        ...
} 
```

然后这 ExecuteSwitchImpl 就是关键的解释指令的函数了。到这里终于有 Youpk 修改的部分了。先看看修改的代码

```
//patch by Youlor
//++++++++++++++++++++++++++++
#define PREAMBLE()                                                                              \
  do {                                                                                          \
    inst_count++;                                                                               \
    bool dumped = Unpacker::beforeInstructionExecute(self, shadow_frame.GetMethod(),            \
                                                     dex_pc, inst_count);                       \
    if (dumped) {                                                                               \
      return JValue();                                                                          \
    }                                                                                           \
    if (UNLIKELY(instrumentation->HasDexPcListeners())) {                                       \
      instrumentation->DexPcMovedEvent(self, shadow_frame.GetThisObject(code_item->ins_size_),  \
                                       shadow_frame.GetMethod(), dex_pc);                       \
    }                                                                                           \
  } while (false)
//++++++++++++++++++++++++++++

```

PREAMBLE 这个函数基本每个指令执行前都会调用 beforeInstructionExecute 来判断下。如果这里 dump 脱壳了，就直接结束掉，这个函数不再往下执行了。如果是上面那种特殊壳，这里就可以暂时先不要 dump。让他正常执行先。下面看看里面的逻辑处理

```
//继续解释执行返回false, dump完成返回true
bool Unpacker::beforeInstructionExecute(Thread *self, ArtMethod *method, uint32_t dex_pc, int inst_count) {
  if (Unpacker::isFakeInvoke(self, method)) {
    const uint16_t* const insns = method->GetCodeItem()->insns_;
    const Instruction* inst = Instruction::At(insns + dex_pc);
    uint16_t inst_data = inst->Fetch16(0);
    Instruction::Code opcode = inst->Opcode(inst_data);
 
    //对于一般的方法抽取(非ijiami, najia), 直接在第一条指令处dump即可
    if (inst_count == 0 && opcode != Instruction::GOTO && opcode != Instruction::GOTO_16 && opcode != Instruction::GOTO_32) {
      Unpacker::dumpMethod(method);
      return true;
    }
    //ijiami, najia的特征为: goto: goto_decrypt; nop; ... ; return; const vx, n; invoke-static xxx; goto: goto_origin;
    else if (inst_count == 0 && opcode >= Instruction::GOTO && opcode <= Instruction::GOTO_32) {
      return false;
    } else if (inst_count == 1 && opcode >= Instruction::CONST_4 && opcode <= Instruction::CONST_WIDE_HIGH16) {
      return false;
    } else if (inst_count == 2 && (opcode == Instruction::INVOKE_STATIC || opcode == Instruction::INVOKE_STATIC_RANGE)) {
      //让这条指令真正的执行
      Unpacker::disableFakeInvoke();
      Unpacker::enableRealInvoke();
      return false;
    } else if (inst_count == 3) {
      if (opcode >= Instruction::GOTO && opcode <= Instruction::GOTO_32) {
        //写入时将第一条GOTO用nop填充
        const Instruction* inst_first = Instruction::At(insns);
        Instruction::Code first_opcode = inst_first->Opcode(inst->Fetch16(0));
        CHECK(first_opcode >= Instruction::GOTO && first_opcode <= Instruction::GOTO_32);
        ULOGD("found najia/ijiami %s", PrettyMethod(method).c_str());
        switch (first_opcode)
        {
        case Instruction::GOTO:
          Unpacker::dumpMethod(method, 2);
          break;
        case Instruction::GOTO_16:
          Unpacker::dumpMethod(method, 4);
          break;
        case Instruction::GOTO_32:
          Unpacker::dumpMethod(method, 8);
          break;
        default:
          break;
        }
      } else {
        Unpacker::dumpMethod(method);
      }
      return true;
    }
    Unpacker::dumpMethod(method);
    return true;
  }
  return false;
}

```

这里可以看到。如果是 INVOKE_STATIC 就让指令正常执行。其他正常的抽取壳的深度就是在这里。这相当于就是指令执行前进行 dump 了。但是这里依然没解决特殊壳的深度问题。必须执行完 INVOKE_STATIC 之后。再进行脱壳并结束掉函数。继续看 Youpk 下面的处理

```
template JValue ExecuteSwitchImpl(Thread* self, const DexFile::CodeItem* code_item,
                         ShadowFrame& shadow_frame, JValue result_register,
                         bool interpret_one_instruction) {
  ...
  //patch by Youlor
  //++++++++++++++++++++++++++++
  int inst_count = -1;
  //++++++++++++++++++++++++++++
  do {
    dex_pc = inst->GetDexPc(insns);
    shadow_frame.SetDexPC(dex_pc);
    TraceExecution(shadow_frame, inst, dex_pc);
    inst_data = inst->Fetch16(0);
    switch (inst->Opcode(inst_data)) {
      ...
      case Instruction::GOTO: {
        PREAMBLE();
        int8_t offset = inst->VRegA_10t(inst_data);
        BRANCH_INSTRUMENTATION(offset);
        if (IsBackwardBranch(offset)) {
          HOTNESS_UPDATE();
          self->AllowThreadSuspension();
        }
        inst = inst->RelativeAt(offset);
        break;
      }
      ...
      case Instruction::INVOKE_STATIC: {
        PREAMBLE();
        bool success = DoInvoke(
            self, shadow_frame, inst, inst_data, &result_register);
        POSSIBLY_HANDLE_PENDING_EXCEPTION(!success, Next_3xx);
        break;
      }
      case Instruction::INVOKE_STATIC_RANGE: {
        PREAMBLE();
        bool success = DoInvoke(
            self, shadow_frame, inst, inst_data, &result_register);
        POSSIBLY_HANDLE_PENDING_EXCEPTION(!success, Next_3xx);
        break;
      }
      ...
    }
    //patch by Youlor
    //++++++++++++++++++++++++++++
    bool dumped = Unpacker::afterInstructionExecute(self, shadow_frame.GetMethod(), dex_pc, inst_count);
    if (dumped) {
      return JValue();
    }
    //++++++++++++++++++++++++++++
  } while (!interpret_one_instruction);
  // Record where we stopped.
  shadow_frame.SetDexPC(inst->GetDexPc(insns));
  return result_register;
}  // NOLINT(readability/fn_size) 
```

这里就看到每个指令都执行了 PREAMBLE 函数。然后每个指令执行完都执行了 afterInstructionExecute 这个函数。在这里就可以判断，如果执行完的指令是 INVOKE_STATIC。就可以直接 return 结束掉函数执行了。看看 Youpk 的处理

```
bool Unpacker::afterInstructionExecute(Thread *self, ArtMethod *method, uint32_t dex_pc, int inst_count) {
  const uint16_t* const insns = method->GetCodeItem()->insns_;
  const Instruction* inst = Instruction::At(insns + dex_pc);
  uint16_t inst_data = inst->Fetch16(0);
  Instruction::Code opcode = inst->Opcode(inst_data);
  if (inst_count == 2 && (opcode == Instruction::INVOKE_STATIC || opcode == Instruction::INVOKE_STATIC_RANGE)
      && Unpacker::isRealInvoke(self, method)) {
    Unpacker::enableFakeInvoke();
    Unpacker::disableRealInvoke();
  }
  return false;
}

```

这里留意了一下。这个函数固定返回的 false。但是通过设置 enableFakeInvoke 和 disableRealInvoke 来控制下一个指令执行的时候来进行退出函数。我感觉这里退出应该也没啥问题。

 

到这里基本就走完大致的流程了。那么欣赏完别人的代码。可以开始我们的改造工作了。

### FartExt 更深的主动调用

和 Youpk 一样。第一步就是先把解释器给改成使用 Switch 解释器。但是由于我使用的是 AOSP10。所以发现修改部分果然不大一样了。

```
#if ART_USE_CXX_INTERPRETER
static constexpr InterpreterImplKind kInterpreterImplKind = kSwitchImplKind;
#else
static constexpr InterpreterImplKind kInterpreterImplKind = kMterpImplKind;
#endif

```

发现这里变成可以通过编译参数来控制的了。搜索一下 ART_USE_CXX_INTERPRETER 的使用

```
if envTrue(ctx, "ART_USE_CXX_INTERPRETER") {
        cflags = append(cflags, "-DART_USE_CXX_INTERPRETER=1")
    }

```

发现这个好像可以通过 cflags 来配置了。所以我修改了下 runtime 下的 Android.pb。如果不想改全局的。也可以在源码里面直接判断是主动调用就强制走 switch 解释器。

```
cflags: [
    // ART is allowed to link to libicuuc directly
    // since they are in the same module
    "-DANDROID_LINK_SHARED_ICU4C",
    "-Wno-error",
    "-DART_USE_CXX_INTERPRETER=1"
],

```

接着就是 ArtMethod::Invoke 的时候不要直接结束了。但是这里我们需要留意的是。第一个参数的 Thread 是 fart 用来判断是否为主动调用的。为了让后面能正常执行，我就直接把第一个参数给赋值了。而后面的调用流程也是需要判断当前执行函数是否为主动调用。Youpk 是用线程和一个变量来控制判断是否为主动调用的。这里使用 result=111111 在后续判断是否为主动调用

```
extern "C" void fartextInvoke(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
    if(artmethod->IsNative()||artmethod->IsAbstract()){
        return;
    }
    JValue result;
  //模拟参数
    Thread *self=Thread::Current();
    uint32_t temp[100]={0};
    uint32_t* args=temp;
    uint32_t args_size = (uint32_t)ArtMethod::NumArgRegisters(artmethod->GetShorty());
    if (!artmethod->IsStatic()) {
      args_size += 1;
    }
  //靠这个值，在后续来判断当前函数是否为主动调用。
    result.SetI(111111);
    LOG(ERROR) << "fartext fartextInvoke";
    Unpacker_self_=self;
    artmethod->Invoke(self, args, args_size, &result,artmethod->GetShorty());
}
 
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
  ...
  //add
  if (result!=nullptr && result->GetI()==111111){
      LOG(ERROR) << "fartext artMethod::Invoke Method "<PrettyMethod().c_str();
      if (IsStatic()) {
        art::interpreter::EnterInterpreterFromInvoke(
                        self, this, nullptr, args, result, /*stay_in_interpreter=*/ true);
      }else{
        //注意这里是把非静态的也当静态的方式处理的。避免使用引用类型参数。
        art::interpreter::EnterInterpreterFromInvoke(
                  self, this, nullptr, args + 1, result, /*stay_in_interpreter=*/ true);
      }
        LOG(ERROR) << "fartext artMethod::Invoke Method Over "<PrettyMethod().c_str();
      self->PopManagedStackFragment(fragment);
      return;
  }
  //add end
  ...
} 
```

这里有个问题是上面这种模拟参数的方式，碰到引用类型的参数会报错。所以在处理参数入栈的时候，也要进行判断处理一下。

```
void EnterInterpreterFromInvoke(Thread* self,
                                ArtMethod* method,
                                ObjPtr receiver,
                                uint32_t* args,
                                JValue* result,
                                bool stay_in_interpreter) {
  ...
  if (!method->IsStatic()) {
    //add  避免使用引用类型的参数
    if(result!=nullptr&&result->GetI()==111111){
        shadow_frame->SetVReg(cur_reg, args[0]);
    }else{
        CHECK(receiver != nullptr);
        shadow_frame->SetVRegReference(cur_reg, receiver);
    }
    //add end
    //shadow_frame->SetVRegReference(cur_reg, receiver);
    ++cur_reg;
  }
  uint32_t shorty_len = 0;
  const char* shorty = method->GetShorty(&shorty_len);
  for (size_t shorty_pos = 0, arg_pos = 0; cur_reg < num_regs; ++shorty_pos, ++arg_pos, cur_reg++) {
    DCHECK_LT(shorty_pos + 1, shorty_len);
    switch (shorty[shorty_pos + 1]) {
      case 'L': {
        //add  避免使用引用类型的参数
        if(result!=nullptr&&result->GetI()==111111){
            shadow_frame->SetVReg(cur_reg, args[0]);
            break;
        }
        //add end
        ObjPtr o =
            reinterpret_cast*>(&args[arg_pos])->AsMirrorPtr();
        shadow_frame->SetVRegReference(cur_reg, o);
        break;
      }
      case 'J': case 'D': {
        uint64_t wide_value = (static_cast(args[arg_pos + 1]) << 32) | args[arg_pos];
        shadow_frame->SetVRegLong(cur_reg, wide_value);
        cur_reg++;
        arg_pos++;
        break;
      }
      default:
        shadow_frame->SetVReg(cur_reg, args[arg_pos]);
        break;
    }
  }
  ...
  if (LIKELY(!method->IsNative())) {
    //这里把我们主动调用函数的标志继续往后面传递
    if(result!=nullptr&&result->GetI()==111111){
        JValue r = Execute(self, accessor, *shadow_frame, *result, stay_in_interpreter);
        if (result != nullptr) {
          *result = r;
        }
          return;
    }else{
        JValue r = Execute(self, accessor, *shadow_frame, JValue(), stay_in_interpreter);
        if (result != nullptr) {
          *result = r;
        }
    }
  }
  ...
} 
```

接下来就开始修改解释器部分的逻辑了。我们只要做到几点处理。就可以搞定这种壳了。

 

1、如果是主动调用并且第一个指令如果不是 GOTO 的。就直接脱壳并结束

 

2、如果是主动调用并且第一个指令是 GOTO 的。让他继续执行

 

3、如果第三个指令是 INVOKE-STATIC 的执行完后直接结束掉

 

接下来准备改代码。然后碰到一个问题。同样也是 AOSP10 的版本导致的。Switch 解释器的逻辑发生了较大的变动。先看看变成了啥样子

```
template ATTRIBUTE_NO_SANITIZE_ADDRESS void ExecuteSwitchImplCpp(SwitchImplContext* ctx) {
  ...
  bool const interpret_one_instruction = ctx->interpret_one_instruction;
  while (true) {
    dex_pc = inst->GetDexPc(insns);
    shadow_frame.SetDexPC(dex_pc);
    TraceExecution(shadow_frame, inst, dex_pc);
    inst_data = inst->Fetch16(0);
    {
      bool exit_loop = false;
      InstructionHandler handler(
          ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, exit_loop);
      //PREAMBLE变成这种方式调用了
      if (!handler.Preamble()) {
        if (UNLIKELY(exit_loop)) {
          return;
        }
        if (UNLIKELY(interpret_one_instruction)) {
          break;
        }
        continue;
      }
    }
    switch (inst->Opcode(inst_data)) {
#define OPCODE_CASE(OPCODE, OPCODE_NAME, pname, f, i, a, e, v)                                    \
      case OPCODE: {                                                                              \
        bool exit_loop = false;                                                                   \
        InstructionHandler handler(                          \
            ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, exit_loop);        \
        handler.OPCODE_NAME();                                                                    \
        /* TODO: Advance 'inst' here, instead of explicitly in each handler */                    \
        if (UNLIKELY(exit_loop)) {                                                                \
          return;                                                                                 \
        }                                                                                         \
        break;                                                                                    \
      }
DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE
    }
    if (UNLIKELY(interpret_one_instruction)) {
      break;
    }
  }
  // Record where we stopped.
  shadow_frame.SetDexPC(inst->GetDexPc(insns));
  ctx->result = ctx->result_register;
  return;
}  // NOLINT(readability/fn_size) 
```

看到了这两个部分都发生了较大的变化。那个超大的 case 都不见了。不过也只是处理的方式发生变化。我们跟着调整下就行了。

```
template ATTRIBUTE_NO_SANITIZE_ADDRESS void ExecuteSwitchImplCpp(SwitchImplContext* ctx) {
  ...
  //add
  int32_t regvalue=ctx->result_register.GetI();
  //这里很重要。需要把我们用来作为主动调用的值给改了。不然调用另外一个函数也会当成fart的主动调用的。
  ctx->result_register=JValue();
  int inst_count = -1;            //当前第几个指令
  bool flag=false;                    //第一个指令是否为goto
  //add end
  bool const interpret_one_instruction = ctx->interpret_one_instruction;
  while (true) {
    ...
    //add
    inst_count++;
    uint8_t opcode = inst->Opcode(inst_data)
    //如果是主动调用
    if(regvalue==111111){
          //第一个指令是goto的处理
        if(inst_count == 0 ){
            if(opcode == Instruction::GOTO || opcode == Instruction::GOTO_16 || opcode == Instruction::GOTO_32){
                LOG(ERROR) << "fartext ExecuteSwitchImplCpp Switch inst_count=0 opcode==GOTO "<PrettyMethod().c_str();
                flag=true;
            }else{
                LOG(ERROR) << "fartext ExecuteSwitchImplCpp Switch inst_count=0 opcode!=GOTO "<PrettyMethod().c_str();
                dumpArtMethod(shadow_frame.GetMethod());
                break;
            }
        }
          //第二个指令是const的处理
        if(inst_count == 1){
            if(opcode >= Instruction::CONST_4 && opcode <= Instruction::CONST_WIDE_HIGH16){
                LOG(ERROR) << "fartext ExecuteSwitchImplCpp Switch inst_count=1 opcode==CONST "<PrettyMethod().c_str();
                flag=true;
            }else{
                LOG(ERROR) << "fartext ExecuteSwitchImplCpp Switch inst_count=1 opcode!=CONST "<PrettyMethod().c_str();
                dumpArtMethod(shadow_frame.GetMethod());
                break;
            }
        }
    }
    //add end
    switch (opcode) {
#define OPCODE_CASE(OPCODE, OPCODE_NAME, pname, f, i, a, e, v)                                    \
      case OPCODE: {                                                                              \
        bool exit_loop = false;                                                                   \
        InstructionHandler handler(                          \
            ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, exit_loop);        \
        handler.OPCODE_NAME();                                                                    \
        /* TODO: Advance 'inst' here, instead of explicitly in each handler */                    \
        if (UNLIKELY(exit_loop)) {                                                                \
          return;                                                                                 \
        }                                                                                         \
        break;                                                                                    \
      }
DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE
    }
    //add
      //指令执行结束后，再判断一下是不是主动调用的
    if(regvalue==111111){
          //如果这是第3个指令
        if(inst_count==2&&flag){
              //如果是下面两种操作码，就可以脱壳并结束了。
            if(opcode == Instruction::INVOKE_STATIC || opcode == Instruction::INVOKE_STATIC_RANGE){
                dumpArtMethod(shadow_frame.GetMethod());
                  ArtMethod::disableFartextInvoke();
                break;
            }
        }
          //如果主动调用的情况还能执行到第4个指令。那就直接脱壳并结束掉。
          if(inst_count>2){
            dumpArtMethod(shadow_frame.GetMethod());
              ArtMethod::disableFartextInvoke();
            break;
        }
    }
    //add end
    if (UNLIKELY(interpret_one_instruction)) {
      break;
    }
  }
  // Record where we stopped.
  shadow_frame.SetDexPC(inst->GetDexPc(insns));
  ctx->result = ctx->result_register;
  return;
}  // NOLINT(readability/fn_size) 
```

[](#八、流程图)八、流程图
---------------

![](https://bbs.pediy.com/upload/attach/202108/659397_VSXS7G8WNMTJTVH.png)

九、更快的主动调用 (暂未优化)
----------------

在测试 FART 的主动调用中发现，主动调用的耗时较长，根据上面的流程图。我们可以看到调用最耗时最核心的函数 dumpArtMethod。就是在这里进行脱壳的。先看看 FART 里面做了什么

```
extern "C" void dumpArtMethod(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
          //存放保存的dex路径
            char *dexfilepath=(char*)malloc(sizeof(char)*1000);   
            if(dexfilepath==nullptr)
            {
                LOG(ERROR) << "ArtMethod::dumpArtMethodinvoked,methodname:"<PrettyMethod().c_str()<<"malloc 1000 byte failed";
                return;
            }
            int result=0;
            int fcmdline =-1;
            char szCmdline[64]= {0};
            char szProcName[256] = {0};
            int procid = getpid();
          //获取进程包名
            sprintf(szCmdline,"/proc/%d/cmdline", procid);
            fcmdline = open(szCmdline, O_RDONLY,0644);
            if(fcmdline >0)
            {
                result=read(fcmdline, szProcName,256);
                if(result<0)
                {
                    LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open cmdline file file error";                                   
                }
                close(fcmdline);
            }
 
            if(szProcName[0])
            {
 
                      const DexFile* dex_file = artmethod->GetDexFile();
                      const uint8_t* begin_=dex_file->Begin();  // Start of data.
                      size_t size_=dex_file->Size();  // Length of data.
 
                      memset(dexfilepath,0,1000);
                      int size_int_=(int)size_;
                      //创建目录
                      memset(dexfilepath,0,1000);
                      sprintf(dexfilepath,"%s","/sdcard/fart");
                      mkdir(dexfilepath,0777);
                        //创建目录     
                      memset(dexfilepath,0,1000);
                      sprintf(dexfilepath,"/sdcard/fart/%s",szProcName);
                      mkdir(dexfilepath,0777);
                        //文件大小_dexfile.dex           
                      memset(dexfilepath,0,1000);
                      sprintf(dexfilepath,"/sdcard/fart/%s/%d_dexfile.dex",szProcName,size_int_);
                      int dexfilefp=open(dexfilepath,O_RDONLY,0666);
                //存在则略过
                      if(dexfilefp>0){
                          close(dexfilefp);
                          dexfilefp=0;
 
                          }else{
                          //dex的数据保存
                                      int fp=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
                                      if(fp>0)
                                      {
                                          result=write(fp,(void*)begin_,size_);
                                          if(result<0)
                                            {
                                                LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open dexfilepath file error";
 
                                            }
                                          fsync(fp);
                                          close(fp); 
                                          memset(dexfilepath,0,1000);
                      //保存对应的classlist
                                          sprintf(dexfilepath,"/sdcard/fart/%s/%d_classlist.txt",szProcName,size_int_);
                                          int classlistfile=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
                                            if(classlistfile>0)
                                            {
                                                for (size_t ii= 0; ii< dex_file->NumClassDefs(); ++ii)
                                                {
                                                    const DexFile::ClassDef& class_def = dex_file->GetClassDef(ii);
                                                    const char* descriptor = dex_file->GetClassDescriptor(class_def);
                                                    result=write(classlistfile,(void*)descriptor,strlen(descriptor));
                                                    if(result<0)
                                                    {
                                                        LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";
 
                                                        }
                                                    const char* temp="\n";
                                                    result=write(classlistfile,(void*)temp,1);
                                                    if(result<0)
                                                    {
                                                        LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";
 
                                                        }
                                                    }
                                                  fsync(classlistfile);
                                                  close(classlistfile);
 
                                                }
                                          }
                            }
                    //获取codeItem
                          const DexFile::CodeItem* code_item = artmethod->GetCodeItem();
                          if (LIKELY(code_item != nullptr))
                          {
                                  int code_item_len = 0;
                                  uint8_t *item=(uint8_t *) code_item;
                    //计算codeitem的大小
                                  if (code_item->tries_size_>0) {
                                      const uint8_t *handler_data = (const uint8_t *)(DexFile::GetTryItems(*code_item, code_item->tries_size_));
                                      uint8_t * tail = codeitem_end(&handler_data);
                                      code_item_len = (int)(tail - item);
                                  }else{
                                      code_item_len = 16+code_item->insns_size_in_code_units_*2;
                                  } 
                    //下面就是获取codeitem的idx和偏移，大小之类的。然后写入数据保存了
                                      memset(dexfilepath,0,1000);
                                      int size_int=(int)dex_file->Size(); 
                                      uint32_t method_idx=artmethod->GetDexMethodIndexUnchecked();
                                      sprintf(dexfilepath,"/sdcard/fart/%s/%d_ins_%d.bin",szProcName,size_int,(int)gettidv1());
                                      int fp2=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
                                      if(fp2>0){
                      //跳到文件末尾写入
                                          lseek(fp2,0,SEEK_END);
                                          memset(dexfilepath,0,1000);
                                          int offset=(int)(item - begin_);
                                          sprintf(dexfilepath,"{name:%s,method_idx:%d,offset:%d,code_item_len:%d,ins:",artmethod->PrettyMethod().c_str(),method_idx,offset,code_item_len);
                                          int contentlength=0;
                                          while(dexfilepath[contentlength]!=0) contentlength++;
                                          result=write(fp2,(void*)dexfilepath,contentlength);
                                          if(result<0)
                                                    {
                                                        LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";
 
                                                        }
                                          long outlen=0;
                                          char* base64result=base64_encode((char*)item,(long)code_item_len,&outlen);
                                          result=write(fp2,base64result,outlen);
                                          if(result<0)
                                                    {
                                                        LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";
 
                                                        }
                                          result=write(fp2,"};",2);
                                          if(result<0)
                                                    {
                                                        LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";
 
                                                        }
                                          fsync(fp2);
                                          close(fp2);
                                          if(base64result!=nullptr){
                                              free(base64result);
                                              base64result=nullptr;
                                              }
                                           }
                            }
 
            }
            if(dexfilepath!=nullptr)
            {
                free(dexfilepath);
                dexfilepath=nullptr;
            }
 
} 
```

这里可以看到 dump 的规则是相同大小的 dex 就跳过。非相同的就写入文件保存。相当于算是一个整体脱壳非常晚的时机。不过这个时机的调用频率较多。相对会影响性能，好处是这个整体脱壳点的时机够晚，绝对能脱掉除抽取函数外的整体壳。而 Youpk 中的优化是不使用这个整体脱壳点，只单纯的把 codeitem 写入 bin 文件，这样就能提高一定的效率。这里我就暂时不修改了。毕竟在这里脱整体壳也有一定的优势。如果想要更快的速度。也可以选择过滤主动调用的范围，来降低调用频率。

[](#十、实战测试)十、实战测试
-----------------

编译完成之后，我们就可以来试试深度主动调用 + dex 修复的效果。为了防止风险，就不放测试的 apk 样本了。

 

安装好 apk 后，先去 / data/local/tmp/fext.config 中填入我们的目标进程。如果主动调用出现崩溃的情况。可以将 class.txt 的文件复制到`/data/local/tmp/进程名称` 来对指定的类进行主动调用。然后打开应用静静的等待脱壳结果。

 

或者是使用整合怪来对指定类进行处理

 

![](https://bbs.pediy.com/upload/attach/202108/659397_TUBGDJMJU7E4B6K.png)

> Ps1: 如果第二次打开应用发现没有触发主动调用，请清理应用：`adb shell pm clear packageName`
> 
> Ps2: 如果不想等待 60 秒，想自己触发 fart 的主动调用。可以使用 frida 扩展
> 
> Ps3: 如果想看 logcat 日志。搜索 fartext 即可，日志统一都添加了这个头部。方便查日志。

 

修复前的数据

 

![](https://bbs.pediy.com/upload/attach/202108/659397_RKZEAWXMZMXCQRP.png)

 

使用前面我修改的修复工具，用下面的命令来修复

 

`java -jar dexfixer.jar dexpath binpath outpath`

 

或者是使用我整合怪的工具来修复

 

![](https://bbs.pediy.com/upload/attach/202108/659397_PGN7S5J9R3QU52D.png)

 

修复后的函数结果如下

 

![](https://bbs.pediy.com/upload/attach/202108/659397_25Q4QU6E6U7GXSR.png)

[](#十一、联合frida扩展)十一、联合 frida 扩展
-------------------------------

可以结合 frida 来直接调用 FART 中准备的函数来对单个类或者类列表进行脱壳。

```
function romClassesInvoke(classes){
    Java.perform(function(){
        klog("romClassesInvoke start load");
        var fartExt=Java.use("cn.mik.Fartext");
        if(!fartExt.fartWithClassList){
            klog("fartExt中未找到fartWithClassList函数，可能是未使用Fartext的rom")
            return ;
        }
        fartExt.fartWithClassList(classes);
    })
}
 
function romFartAllClassLoader(){
    Java.perform(function(){
       var fartExt=Java.use("cn.mik.Fartext");
       if(!fartExt.fartWithClassLoader){
           klog("fartExt中未找到fartWithClassLoader函数，可能是未使用Fartext的rom");
           return;
       }
       Java.enumerateClassLoadersSync().forEach(function(loader){
           klog("romFartAllClassLoader to loader:"+loader);
           if(loader.toString().indexOf("BootClassLoader")==-1){
               klog("fart start loader:"+loader);
               fartExt.fartWithClassLoader(loader);
           }
       })
    });
}

```

同时我的整合怪里面也添加了对我这个 rom 的主动调用和类列表主动调用支持

 

![](https://bbs.pediy.com/upload/attach/202108/659397_PWZE67SU88RCMXW.png)

[](#十二、思考)十二、思考
---------------

整个流程梳理完成后，我们可以由此来借鉴来思考延伸一下。

 

比如，包装一些属于自己的系统层 api 调用。便于我们使用 xposed 或者是 frida 来调用一些功能。

 

再比如，加载应用时，读取配置文件作为开关，我们来对网络流量进行拦截写入保存，或者对所有的 jni 函数调用，或者是 java 函数调用进行 trace。这种就属于是 rom 级别的打桩。

 

再比如，可以做一个应用来读写作为开关的配置文件，而 rom 读取配置文件后，对一些流程进行调整。例如控制 FART 是否使用更深调用。控制是否开启 rom 级别的打桩。

 

以上纯属个人瞎想。刚刚入门，想的有点多，以后了解更深了，我再看看如何定制一个专属的 rom 逆向集合

[](#十三、补充)十三、补充
---------------

整理一下前面所有的资料。

### 文章

> [FART：ART 环境下基于主动调用的自动化脱壳方案](https://bbs.pediy.com/thread-252630.htm)
> 
> [FART 正餐前甜点：ART 下几个通用简单高效的 dump 内存中 dex 方法](https://bbs.pediy.com/thread-254028.htm)
> 
> [拨云见日：安卓 APP 脱壳的本质以及如何快速发现 ART 下的脱壳点](https://bbs.pediy.com/thread-254555.htm)
> 
> [将 FART 和 Youpk 结合来做一次针对函数抽取壳的全面提升](https://bbs.pediy.com/thread-260052.htm)
> 
> [fart 的理解和分析过程](https://bbs.pediy.com/thread-263401.htm)
> 
> [Android ART 虚拟机 - dex 文件格式要旨](https://www.jianshu.com/p/18ff0b8e0b01)
> 
> [dex 起步探索](https://bbs.pediy.com/thread-268465.htm)

### github

> [FART](https://github.com/hanbinglengyue/FART)
> 
> [Youpk](https://github.com/Youlor/Youpk)
> 
> [dexfixer](https://github.com/dqzg12300/dexfixer)
> 
> [fridaUiTools](https://github.com/dqzg12300/fridaUiTools)

### rom 下载

> 链接: https://pan.baidu.com/s/1lgG8P3H2Q5B6e7rZr58cXw 密码: 033p

### FartExt 开源

> [FartExt](https://github.com/dqzg12300/FartExt.git)

 

刚梳理完，难免有些纰漏。我会慢慢修补的。

这里改下：

[[培训] 优秀毕业生寄语： 优秀毕业生寄语：恭喜 id 白小菜拿到双倍于 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#系统相关](forum-161-1-126.htm)