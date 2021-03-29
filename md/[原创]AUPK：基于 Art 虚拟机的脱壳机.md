> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266716.htm)

### 一. 背景

如今, 随着开发者安全意识的提高, 加壳的 app 比比皆是, 反而不加壳的并不多见. 对于一款加壳的 app, 脱壳是逆向工程中非常重要的一个步骤. 有一个好的脱壳机, 可以大大节省逆向人员的时间与精力.

 

android 系统的换代升级非常块, 在 4.4 版本后舍弃了以往的 dalvik 虚拟机, 取而代之的是更高效的 art 虚拟机. 在 android 脱壳领域, 有许多前辈共享了其思路, 代码, 使本人得已站在巨人的肩膀上进行学习, 研究. 如 [FART：ART 环境下基于主动调用的自动化脱壳方案](https://bbs.pediy.com/thread-252630.htm)中,[hanbingle](https://bbs.pediy.com/user-home-632473.htm) 大佬开源的基于 art 的主动调用方案:[FART](https://github.com/hanbinglengyue/FART).

###### 那么, 既然有了 FART, 为什么我还要写 AUPK 呢?

1.  FART 是通过 ClassLoader 里的 path_list 来获取 DexFile 对象，再通过以下方法来获取到所有类,
    
    ```
    package dalvik.system.DexFile;
    String[] getClassNameList(DexFile dexFile,Object mCookie);
    
    ```
    
    进而获取到所有 Method 对象并进行主动调用. 这种方案在大多数情况下非常有效, 但对某些壳, 如某南极动物壳, 这里无法获取到关键的 DexFile 对象, 导致无法进行后面的主动调用. AUPK 提供了其他思路来应对这种情况.
    
2.  FART 的调用链只到了 Invoke 函数即返回, 对于许多函数恢复时机更晚的抽取壳, 如某加密, 某叠字壳, 如果函数是第一次执行, 此时 dump 下来的 codeItem, 显然是未解密的, AUPK 构造了更深的调用链, 以及更晚的 dump 时机来解决这一问题.
    

论坛也有许多其他的大佬，提供了 ART 下的主动调用脱壳思路，但遗憾的是，均未开源。所以我决定自己实现一个脱壳机。

### 二. 原理

整体流程图：

 

![](https://bbs.pediy.com/upload/attach/202103/887837_QMYM2K2NWHJ3RCM.png)

##### 脱壳

关于如何整体 dump Dex 文件，网上有诸多的解决方案, 各家都有不同的脱壳点，如`DexFile:init`,`DexFile::OpenCommon`等，其原理是因为 dex 文件在进行加载的时候，必定会调用这些函数，且此时能方便的拿到`DexFile`对象, 从而进行 dump。AUPK 的脱壳点在`ExecuteSwitchImpl`，当满足特定条件时，类中的某些函数必定会走到这里，此时可以通过栈帧拿到 ArtMethod 对象，进而获取到 DexFile 对象。

 

在[拨云见日：安卓 APP 脱壳的本质以及如何快速发现 ART 下的脱壳点](https://bbs.pediy.com/thread-254555.htm)一文中, 作者阐明了, 不管壳是否禁用了 dex2oat, 类的初始化函数始终运行在 ART 下的 inpterpreter 模式, 即 class.<clinit> 函数是必定要运行到 Execute 函数中的, 当 app 运行时, 不可避免的要加载 dex, 加载、链接、解析、初始化类，而当有类进行初始化的时候，就一定会走到 Execute

 

Execute 部分代码如下所示:

```
// interpreter.cc
static inline JValue Execute(
        Thread *self,
        const DexFile::CodeItem *code_item,
        ShadowFrame &shadow_frame,
        JValue result_register,
        bool stay_in_interpreter = false) SHARED_REQUIRES(Locks::mutator_lock_){
    if (kInterpreterImplKind == kMterpImplKind)
    {
        if (transaction_active)
        {
            // No Mterp variant - just use the switch interpreter.
            return ExecuteSwitchImpl(self, code_item, shadow_frame, result_register,
                                                  false);
        }
        else if (UNLIKELY(!Runtime::Current()->IsStarted()))
        {
            return ExecuteSwitchImpl(self, code_item, shadow_frame, result_register,
                                                   false);
        }
        else
        {
            while (true)
            {
                // Mterp does not support all instrumentation/debugging.
                if (MterpShouldSwitchInterpreters())
                {
                    return ExecuteSwitchImpl(self, code_item, shadow_frame, result_register,false);
                }
                bool returned = ExecuteMterpImpl(self, code_item, &shadow_frame, &result_register);
                if (returned)
                {
                    return result_register;
                }
                else
                {
                    // Mterp didn't like that instruction.  Single-step it with the reference interpreter.
                    result_register = ExecuteSwitchImpl(self, code_item, shadow_frame,result_register, true);
                    if (shadow_frame.GetDexPC() == DexFile::kDexNoIndex)
                    {
                        // Single-stepped a return or an exception not handled locally.  Return to caller.
                        return result_register;
                    }
                }
            }
        }
    }
    else if (kInterpreterImplKind == kSwitchImplKind)
    {
        if (transaction_active)
        {
            return ExecuteSwitchImpl(self, code_item, shadow_frame, result_register,
                                                  false);
        }
        else
        {
            return ExecuteSwitchImpl(self, code_item, shadow_frame, result_register,
                                                   false);
        }
    }
    else
    {
        DCHECK_EQ(kInterpreterImplKind, kComputedGotoImplKind);
        if (transaction_active)
        {
            return ExecuteGotoImpl(self, code_item, shadow_frame, result_register);
        }
        else
        {
            return ExecuteGotoImpl(self, code_item, shadow_frame, result_register);
        }
    }
    ...
 } 
```

可以看到，Execute 会根据当前选择的解释器类型，进入不同的解释器去执行，当我们手动把默认的解释器改为`kSwitchImplKind`，则可以让 Execute 默认走 swich 型解释器，从而进入 ExecuteSwitchImpl

```
// interpreter.cc
//static constexpr InterpreterImplKind kInterpreterImplKind = kMterpImplKind;
static constexpr InterpreterImplKind kInterpreterImplKind = kSwitchImplKind;

```

在 ExecuteSwitchImpl 函数中，通过 shadow_frame.GetMethod() 可以拿到当前 ArtMethod 对象，再通过 artMethod->GetDexFile() 可以获取 DexFile 对象，

 

最后通过 DexFile 对象的 Begin(),Size() 获取到起始位置和大小，来 dump Dex 文件.

```
template JValue ExecuteSwitchImpl(Thread *self, const DexFile::CodeItem *code_item,
                             ShadowFrame &shadow_frame, JValue result_register,
                             bool interpret_one_instruction) SHARED_REQUIRES(Locks::mutator_lock_){
    ArtMethod *artMethod = shadow_frame.GetMethod();
    bool isFakeInvokeMethod = Aupk::isFakeInvoke(self, artMethod);
 
    if (!isFakeInvokeMethod && strstr(PrettyMethod(shadow_frame.GetMethod()).c_str(), "") != nullptr)
    {      
        const DexFile *dexFile = artMethod->GetDexFile();
        char feature[] = "ExecuteSwitchImpl";
        Aupk::dumpDexFile(dexFile, feature);
        Aupk::dumpClassName(dexFile, feature);
    }
} 
```

与此同时，在 dumpClassName 函数中，获取 Dex 文件的所有类名, 并写入到文件 (关于这一步的作用, 后面会讲到)：

```
json root;
for (size_t i = 0; i < dexFile->NumClassDefs(); i++)
{
    const DexFile::ClassDef &classDef = dexFile->GetClassDef(i);
    const char *descriptor = dexFile->GetClassDescriptor(classDef);
    root["count"] = i + 1;
    root["data"][i] = descriptor;
}
ofstream oFile;
oFile.open(fileName, ios::out);
if (oFile.is_open())
{
    oFile << root;
    oFile.close();
    LOG(INFO) << "AUPK->dump class name:success:" << fileName;
}

```

到这里, 我们已经拿到了 Dex 文件，以及 Dex 文件包含的所有类的类名。

 

若想对函数进行主动调用，则必须得到函数对应的 ArtMethod 对象，以下方法将 jMethod 转为 ArtMethod

```
// aupk_method.cc
ArtMethod *jMethodToArtMethod(JNIEnv *env, jobject jMethod)
{
    ScopedFastNativeObjectAccess soa(env);
    ArtMethod *method = ArtMethod::FromReflectedMethod(soa, jMethod);
    return method;
}

```

在 Java 中，Method 对象可以通过 Class 对象的成员方法 getDeclaredMethods() 来获得

```
// 获取目标类的所有构造函数
Constructor constructors[] = klass.getDeclaredConstructors();
for (Object constructor : constructors)
{
    String methodName=klass.getName()+constructor.toString();
    classInfo.methodMap.put(methodName,constructor);
    count++;
}
 
// 获取目标类的所有成员函数
Method[] methods = klass.getDeclaredMethods();
for (Method method : methods)
{
    String methodName=klass.getName()+method.toString();
    classInfo.methodMap.put(methodName,method);
    count++;
}

```

所以，只要获取到了所有的 Class 对象，那么就可以得到所有的 ArtMethod 对象。在前面，我们获取到了 Dex 里包含的所有类的名字，此时便可对这些类进行加载，并通过上述方法获取到所有 Method 对象。

```
// 通过类名加载类，得到Class对象
Class klass=getClassLoader().loadClass(className);

```

> 小结：首先通过类名加载并获取类对象，再通过类对象获取类中所有的方法，进而将方法传入 native 层，转为 artMethod，并进行主动调用。

 

至此，已经可以拿到所有的 ArtMethod 对象了。

 

接下来就可以构造参数，并对其进行主动调用：

```
void Aupk::aupkFakeInvoke(ArtMethod *artMethod) SHARED_REQUIRES(Locks::mutator_lock_)
{
    if (artMethod->IsAbstract() || artMethod->IsNative() || (!artMethod->IsInvokable()) || artMethod->IsProxyMethod())
    {
        return;
    }
    JValue result;
    Thread *self = Thread::Current();
    uint32_t args_size = (uint32_t)ArtMethod::NumArgRegisters(artMethod->GetShorty());
 
    if (!artMethod->IsStatic())
    {
        args_size += 1;
    }
    std::vector args(args_size, 0);
    if (!artMethod->IsStatic())
    {
        args[0] = 0x12345678;
    }
    artMethod->Invoke(self, args.data(), args_size, &result, artMethod->GetShorty());
} 
```

在解释执行模式下, Native 方法和动态 Proxy 方法没有意义, 所以直接跳过, 抽象方法不能执行, 需要其子类提供实现, 所以也直接跳过

> 需要注意的是，函数的第一个参数要赋值为除 0 以外的任何值, 因为后面会判断 CHECK(arg[0]!=nullptr)

 

在 art 中，系统会在 app 安装或运行过程中对 dex 文件进行 dex2oat，将其中的函数编译成目标代码，而不再是虚拟机执行的字节码，同时也保留了原来由虚拟机执行的字节码，所以 art 代码既可以运行在解释器模式下，也可以作为 quick_code 来执行。为了使主动调用的函数能够到达`ExecuteSwitchImpl`函数，这里使主动调用的函数强制走解释器：

```
//art_method.cc
void ArtMethod::Invoke(Thread *self, uint32_t *args, uint32_t args_size, JValue *result,const char *shorty)
{
    ...
    bool isFakeInvokeMethod = Aupk::isFakeInvoke(self, this);
    if (UNLIKELY(!runtime->IsStarted() || Dbg::IsForcedInterpreterNeededForCalling(self, this) || isFakeInvokeMethod))
    {
      if (IsStatic())
      {      
        art::interpreter::EnterInterpreterFromInvoke(
            self, this, nullptr, args, result, /*stay_in_interpreter*/ true);
      }
      else
      {
        mirror::Object *receiver =
            reinterpret_cast *>(&args[0])->AsMirrorPtr();      
        art::interpreter::EnterInterpreterFromInvoke(
            self, this, receiver, args + 1, result, /*stay_in_interpreter*/ true);
      }
    }
    else
    {
        ...
        if (!IsStatic())
        {
          (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
        }
        else
        {
          (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
        }
        ...
    }
} 
```

这样一来，主动调用时，函数不管是静态还是非静态，都会走 art::interpreter::EnterInterpreterFromInvoke，分析 EnterInterpreterFromInvoke 函数可知，如果调用的方法为非 native 方法，则会进入 Execute 函数继续执行, 而 native 方法，并非我们所关注的对象，所以在主动调用前，会过滤掉 native 函数

```
void EnterInterpreterFromInvoke(Thread *self, ArtMethod *method, Object *receiver,
                                    uint32_t *args, JValue *result,
                                    bool stay_in_interpreter){
    ...
    if (LIKELY(!method->IsNative()))
    {
        JValue r = Execute(self, code_item, *shadow_frame, JValue(), stay_in_interpreter);
        if (result != nullptr)
        {
            *result = r;
        }
     }
    else
    {
        ...
    }
    ...
}

```

如上文所述，进入 Execute 后, 函数最终会进入 ExecuteSwitchImpl 执行。在 ExecuteSwitchImpl 函数中，准备工作都已做好，函数指令即将开始执行，对于大多数壳来说，此时函数的 code_item 都已修复，否则函数无法正常执行，这也是为什么会将主动调用链构造到这里的原因。事实上，对于大多数壳，在这里都可以获取到已被恢复的 code_item，个别情况，如在函数体内部 goto 跳转进行自解密的情况，则需要单独处理：

```
bool firstInsIsGoto = false;
if (isFakeInvokeMethod)
{
    inst_data = inst->Fetch16(0);
    // 当执行的为方法的第一条指令时
    if (dex_pc == 0)
    {
        if (inst->Opcode(inst_data) == Instruction::GOTO ||
            inst->Opcode(inst_data) == Instruction::GOTO_16 ||
            inst->Opcode(inst_data) == Instruction::GOTO_32)
        {
            // 如果第一条指令为goto,则继续执行
            firstInsIsGoto = true;
        }
        else
        {
            // 如果第一条指令不是goto,则直接dumpMethod
            char feature[] = "ExecuteSwitchImpl";
            Aupk::dumpMethod(method, feature);
            return JValue();
        }
    }
 
    if(inst->Opcode(inst_data) == Instruction::INVOKE_STATIC)
    {
        if (firstInsIsGoto)
        {
            // 如果指令为invoke-static,且第一条指令为goto,则等invoke-static执行完毕后再dumpMethod
            DoInvoke(self, shadow_frame, inst, inst_data, &result_register);
            char feature[] = "ExecuteSwitchImpl";
            LOG(INFO)<<"AUPK->ExecuteSwitchImpl goto:"<
```

上述处理方式并不通用，需要根据具体的情况来作处理，总的来说，dump 时机一定要在函数指令被恢复之后，但又要在函数真正的指令被执行之前。

 

除了我们主动调用的函数，依然有大量的其他函数会执行到`ExecuteSwitchImpl`，所以我们需要在此处进行判断：如果是我们主动调用的函数，则不让其执行真正的指令，否则正常执行。那么，如何判断函数调用是否为我们主动调用的呢？

 

这里可采用的方法多种多样，AUPK 中采取了如下方式：在最开始，我们创建了一个主动调用线程，那么在进行 fakeInvoke 的时候，可以保存当前的`Thread`对象，以及即将进行主动调用的`ArtMethod`对象，在主动调用过程中，比对这两个值，即可判断是否为主动调用的函数。不过，此方式需要过滤掉我们主动调用过程中，所用到的 java 层函数，否则会引发各种错误。

```
// aupk.cc
static void Aupk_native_fakeInvoke(JNIEnv *env, jclass, jobject jMethod) SHARED_REQUIRES(Locks::mutator_lock_)
{
    if (jMethod != nullptr)
    {
        Thread *self = Thread::Current();
        ArtMethod *artMethod = jMethodToArtMethod(env, jMethod);
        // 保存Thread对象
        Aupk::setThread(self);
        // 保存ArtMethod对象
        Aupk::setMethod(artMethod);
        // 发起主动调用
        Aupk::aupkFakeInvoke(artMethod);
    }
    return;
}
 
Thread *Aupk::aupkThread = nullptr;
ArtMethod *Aupk::aupkArtMethod = nullptr;
 
 
void Aupk::setMethod(ArtMethod *method)
{
    aupkArtMethod = method;
}
 
void Aupk::setThread(Thread *thread)
{
    aupkThread = thread;
}
 
bool Aupk::isFakeInvoke(Thread *thread, ArtMethod *method) SHARED_REQUIRES(Locks::mutator_lock_)
{
    if (aupkThread == nullptr || aupkArtMethod == nullptr || thread == nullptr || method == nullptr)
    {
        return false;
    }
    if ((thread->GetTid() == aupkThread->GetTid()) &&
        strcmp(PrettyMethod(method).c_str(), PrettyMethod(aupkArtMethod).c_str()) == 0)        
    {                                        
        return true;
    }
    return false;
}

```

我画了一张图，来对上述流程做一个总结

 

![](https://bbs.pediy.com/upload/attach/202103/887837_VE87PXMEZQEEUB2.png)

 

其中`fakeInvoke()`为主动调用的入口，省略了非主动调用的函数的执行流程。

##### 修复

###### [](#修复的整体思路如下：)修复的整体思路如下：

*   解析 dump 下来的 Dex 文件
*   遍历 Dex 文件所有 direct_method 和 virtual_method, 并获取其 index
*   根据 index 去找到 dump 下来的函数信息的 code_item
*   将 code_item 回填
*   部分壳需要还原 dex_header 中的 Magic 字段

其所需的知识主要是对 dex 文件结构的理解，网上已有许多相关资料，此处不多赘述。

### 三. 实验

首先安装目标 app

```
$ adb install your_app

```

运行 app，可在 logcat 看到如下信息：  
![](https://bbs.pediy.com/upload/attach/202103/887837_PD2HHSKQZ8Y2H2Y.png)

 

此时`data/data/your_app/aupk`下多出许多文件

```
$ cd data/data/your_app/aupk
$ ls -l

```

![](https://bbs.pediy.com/upload/attach/202103/887837_JPGH7VVX3FQ2EWD.png)

 

其中，`.dex`为整体 dump 下来的 Dex 文件，`class.json`记录了 Dex 文件里所有的类名，前缀数字代表 Dex 文件的大小。

 

可以用`findstr`命令来查找某一个类的类名在哪个文件中, 如：

```
$ findstr /m "SplashActivity" *class.json

```

![](https://bbs.pediy.com/upload/attach/202103/887837_UNN6RZB83WR9MQ2.png)

 

可以看到，"SplashActivity" 类在 8273364 大小的 Dex 文件中，那么，我们可以通过以下命令，来写入配置文件

 

以开始对`8273364_ExecuteSwitchImpl_class.json`里所有的类的所有方法进行主动调用

```
$ echo "com.klcxkj.zqxy 8273364_ExecuteSwitchImpl_class.json">data/local/tmp/aupk.config

```

或者对所有的 Dex 里的类的所有方法进行主动调用：

```
$ echo "com.klcxkj.zqxy">data/local/tmp/aupk.config

```

主动调用过程中打印的 log:

 

![](https://bbs.pediy.com/upload/attach/202103/887837_2DQAFDRR4MC6RP7.png)

 

有的壳 hook 了 Log 函数，导致 Log.i() 打印不出来消息，但 jni 层的 LOG 和 Log.e() 依然有效，当打印出`Aupk run over`时，代表整个主动调用过程结束，可以在`data/data/you_app/aupk`下看到一些以`method.json`结尾的文件，这些文件包含了 Dex 文件的函数 CodeItem 信息，用于后面对 Dex 文件的修复。

 

并非等整个主动调用过程结束才会生成`method.json`文件，而是每完成对一个`class.json`文件的解析和调用，就会立即生成对应的`method.json`，所以，利用主动调用的这段时间，你可以先修复已经完成了主动调用的 Dex 文件，或者去泡杯咖啡。

 

将脱下来的文件拷贝到电脑:

```
$ adb pull data/data/your_app/aupk

```

开始修复 Dex，回填 CodeItem：

```
$ dp fix -d 8273364_ExecuteSwitchImpl.dex -j 8273364_ExecuteSwitchImpl_method.json [--nolog]

```

等待片刻，即可完成修复：

 

![](https://bbs.pediy.com/upload/attach/202103/887837_394BKD4PXFK3CW6.png)

 

带 patched 后缀的就是修复后的 Dex 文件

 

![](https://bbs.pediy.com/upload/attach/202103/887837_JJ3XAF8VX4B84QD.png)

 

反编译看看效果：

 

![](https://bbs.pediy.com/upload/attach/202103/887837_SCWER6B4C6EGNNA.png)

### 四. 对抗手段

以下为某些场景下的对抗手段（纯 YY, 并未遇到过），以及个人的一些应对思路

1.  文件检测

![](https://bbs.pediy.com/upload/attach/202103/887837_DGE2HYMHDDKRVHV.png)

 

应对方法：修改配置文件和中间文件的名字，路径。

1.  方法，类检测
    
    ![](https://bbs.pediy.com/upload/tmp/887837_4M8CQWNDRB8PAVJ.png)
    
    应对方法：更改类名，方法名
    
2.  插入垃圾类
    
    ![](https://bbs.pediy.com/upload/attach/202103/887837_HQJMQ2KNYJKM9QH.png)
    
    AUPK 使用`loadClass`加载类，并不会触发类的 <clinit> 函数, 也就不会触发静态代码块里的检测。但在调用静态函数时，如果类未初始化，art 会调用`EnsureInitialized`去初始化这个类，此时会调用类的初始化函数，触发静态代码块，其过程如下所示：
    
    ```
    void EnterInterpreterFromInvoke(Thread* self, ArtMethod* method, Object* receiver,
                                    uint32_t* args, JValue* result)
    {
        ...
        // Do this after populating the shadow frame in case EnsureInitialized causes a GC.
        if (method->IsStatic() && UNLIKELY(!method->GetDeclaringClass()->IsInitialized()))
        {
            ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
            StackHandleScope<1> hs(self);
            Handle h_class(hs.NewHandle(method->GetDeclaringClass()));
            if (UNLIKELY(!class_linker->EnsureInitialized(self, h_class, true, true)))
            {
                CHECK(self->IsExceptionPending());
                self->PopShadowFrame();
                return;
            }
        }
        ...  
    } 
    ```
    
    应对方法：
    
    1.  壳为了保证效率，通常在恢复函数指令后，就不会再对其进行抽取了。所以，可以等 app 运行到一定阶段，确保感兴趣的地方已经被执行过，就可以采用 dex 整体 dump 的方式，来 dump Dex 文件，此时虽然部分函数依然是抽取状态，但关注的部分函数已经解密。
        
        在 AUPK 中，你可以删除掉所有的 *.dex 文件，然后重新操作一下 app, 即可完成对 Dex 文件的重新 dump。
        
        ```
        $ cd data/data/your_app/aupk
        $ rm *.dex
        
        ```
        
    2.  frida 提供了一个 api:`enumerateLoadedClasses`，该方法可以获取到所有已加载的类，然后通过反射去获取类中的所有函数，并调用 Aupk 中的`fakeInvoke` ，进行主动调用。这样获得的类虽然不全，但包含了 app 正常使用中所用到的类，且避开了插入的垃圾类。该方法是在类的粒度进行处理，理论上讲，比方法 1 修复得更完整。伪代码如下：
        
        ```
        function test() {
            Java.perform(function () {
                Java.enumerateLoadedClasses({
                    onMatch: function (className) {
                        var klass = loadClass(className);
                        var methods = klass.getMethods();
                        fakeInvoke(methods);
                    }, onComplete: function () {
         
                    }
                })
            });
        }
        
        ```
        
    3.  在`fakeInvoke`前，将当前方法所属的类名写入文件`temp.txt`，当最后一次执行`fakeInvoke`时，清空`temp.txt`. 在发起主动调用前，先检查`temp.txt`，将其中`temp.txt`中的项追加到`detectClass.txt`，并跳过所有属于`detectClass.txt`中的类的函数。当壳检测到脱壳机，终止 app 时，`temp.txt`中的内容为非空。如此反复，到最后主动调用成功时，`detectClass.txt`中则记录着所有检测点所属的类。
        

### 五. 写在最后

从学习脱壳到现在，不知不觉，竟已两月有余，期间遇到过许许多多，一言难尽的问题。对于这类事物，我认为仅仅靠一两篇文章，读者依然很难去复现它。为了能让像我一样的新手去更好的学习和参考，我将同 hanbingle 大佬一样，将其开源。

 

AUPK:https://github.com/FeJQ/AUPK

 

DexPatcher:https://github.com/FeJQ/DexPatcher

 

参考文献：

 

[1] hanbingle.[FART：ART 环境下基于主动调用的自动化脱壳方案](https://bbs.pediy.com/thread-252630.htm)

 

[2] Youlor.[Youpk: 又一款基于 ART 的主动调用的脱壳机](https://bbs.pediy.com/thread-259854.htm)

 

[3] 邓凡平. 深入理解 Android：Java 虚拟机 ART[M]. 机械工业出版社. 2019.04

 

[4] [FART](https://github.com/hanbinglengyue/FART)

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 1 小时前 被 FeJQ 编辑 ，原因： 标题多了个前缀