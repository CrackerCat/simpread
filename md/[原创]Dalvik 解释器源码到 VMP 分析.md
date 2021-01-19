> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-226214.htm)

Dalvik 解释器源码到 VMP 分析
====================

前言
--

学习这块的主要目的还是想知道 vmp 是如何实现的，如何与系统本身的虚拟机配合工作，所以简单的学习了 Dalvik 的源码并对比分析了数字公司的解释器。笔记结构如下：

1.  dalvik 解释器分析
    1.  dalvik 解释器解释指令前的准备工作
    2.  dalvik 解释器的模型
    3.  invoke-super 指令实例分析
2.  数字壳解释器分析
    1.  解释器解释指令前的准备工作
    2.  解释器的模型
    3.  invoke-super 指令实例分析

dalvik 解释器分析
------------

### dalvik 解释器解释指令前的准备工作

从外部进入解释器的调用链如下：  
dvmCallMethod -> dvmCallMethodV -> dvmInterpret

 

这三个函数是在解释器取指令，选分支之前被调用，主要负责一些准备工作，包括分配虚拟寄存器，放入参数，初始化解释器参数等。其中 dvmCallMethod，直接调用了 dvmCallMethodV. 下面分析下后两个函数。

#### dvmCallMethodV

dalvik 虚拟机是基于寄存器架构的，可想而知，在具体执行函数之前，首先要做的就是分配好虚拟寄存器空间，并且将函数所需的参数，放入虚拟寄存器中。主要流程：

1.  取出函数的简单声明，如 onCreate 函数的简单声明为：VL
2.  分配虚拟寄存器栈
3.  放入 this 参数，根据参数类型放入申明中的参数
4.  如果方法是 native 方法，直接跳转到 method->nativeFunc 执行
5.  如果方法是 java 方法，进入 dvmInterpret 解释执行

```
void dvmCallMethodV(Thread* self, const Method* method, Object* obj, bool fromJni, JValue* pResult, va_list args)
{
    //取出方法的简要声明
    const char* desc = &(method->shorty[1]); // [0] is the return type.
    int verifyCount = 0;
    ClassObject* clazz;
    u4* ins;
    //访问权限检查，以及分配函数调用栈，在栈中维护了一份虚拟寄存器列表。
    clazz = callPrep(self, method, obj, false);
    if (clazz == NULL)
        return;
 
    /* "ins" for new frame start at frame pointer plus locals */
    //指向第一个参数
    ins = ((u4*)self->interpSave.curFrame) + (method->registersSize - method->insSize);
 
    //放入this指针，到第一个参数。
    /* put "this" pointer into in0 if appropriate */
    if (!dvmIsStaticMethod(method)) {
        *ins++ = (u4) obj;
        verifyCount++;
    }
    //根据后续参数的类型，放入后续参数
    while (*desc != '\0') {
        switch (*(desc++)) {
            case 'D': case 'J': {
                u8 val = va_arg(args, u8);
                memcpy(ins, &val, 8);       // EABI prevents direct store
                ins += 2;
                verifyCount += 2;
                break;
            }
            case 'F': {
                /* floats were normalized to doubles; convert back */
                float f = (float) va_arg(args, double);
                *ins++ = dvmFloatToU4(f);
                verifyCount++;
                break;
            }
            case 'L': {     /* 'shorty' descr uses L for all refs, incl array */
                void* arg = va_arg(args, void*);
                assert(obj == NULL || dvmIsHeapAddress(obj));
                jobject argObj = reinterpret_cast(arg);
                if (fromJni)
                    *ins++ = (u4) dvmDecodeIndirectRef(self, argObj);
                else
                    *ins++ = (u4) argObj;
                verifyCount++;
                break;
            }
            default: {
                /* Z B C S I -- all passed as 32-bit integers */
                *ins++ = va_arg(args, u4);
                verifyCount++;
                break;
            }
        }
    }
    //如果是本地方法，就直接跳转到本地方法，若是java方法，进入解释器，解释执行。
    if (dvmIsNativeMethod(method)) {
        TRACE_METHOD_ENTER(self, method);
        /*
         * Because we leave no space for local variables, "curFrame" points
         * directly at the method arguments.
         */
        (*method->nativeFunc)((u4*)self->interpSave.curFrame, pResult,
                              method, self);
        TRACE_METHOD_EXIT(self, method);
    } else {
        dvmInterpret(self, method, pResult);
    }
    dvmPopFrame(self);
} 
```

#### dvmInterpret

dvmInterpret 作为虚拟机的入口，主要做了如下工作：

1.  初始化解释器的执行环境。主要是对解释器的变量进行初始化，如将要执行方法的指针，当前函数栈的指针，程序计数器等。
2.  判断将要执行的方法是否合法
3.  JIT 环境的设置
4.  根据系统参数选择解释器（Fast 解释器或者 Portable 解释器）

```
void dvmInterpret(Thread* self, const Method* method, JValue* pResult)
{
    //解释器的状态
    InterpSaveState interpSaveState;
    ExecutionSubModes savedSubModes;
 
#if defined(WITH_JIT)
    double calleeSave[JIT_CALLEE_SAVE_DOUBLE_COUNT];
#endif
 
    //保存之前的解释器状态，并将新的状态和之前的状态连接起来(链表)
    interpSaveState = self->interpSave;
    self->interpSave.prev = &interpSaveState;
    /*
     * Strip out and save any flags that should not be inherited by
     * nested interpreter activation.
     */
    savedSubModes = (ExecutionSubModes)(
              self->interpBreak.ctl.subMode & LOCAL_SUBMODE);
    if (savedSubModes != kSubModeNormal) {
        dvmDisableSubMode(self, savedSubModes);
    }
#if defined(WITH_JIT)
    dvmJitCalleeSave(calleeSave);
#endif
 
#if defined(WITH_TRACKREF_CHECKS)
    self->interpSave.debugTrackedRefStart =
        dvmReferenceTableEntries(&self->internalLocalRefTable);
#endif
    self->debugIsMethodEntry = true;
#if defined(WITH_JIT)
    /* Initialize the state to kJitNot */
    self->jitState = kJitNot;
#endif
 
    /初始化解释器的执行环境
 
    self->interpSave.method = method;  //初始化执行的方法
    self->interpSave.curFrame = (u4*) self->interpSave.curFrame; //初始化函数调用栈
    self->interpSave.pc = method->insns;  //初始化程序计数器
    //检查方法是否为本地方法
    assert(!dvmIsNativeMethod(method));
    //方法的类是否初始化
    if (method->clazz->status < CLASS_INITIALIZING || method->clazz->status == CLASS_ERROR)
    {
        ALOGE("ERROR: tried to execute code in unprepared class '%s' (%d)",
            method->clazz->descriptor, method->clazz->status);
        dvmDumpThread(self, false);
        dvmAbort();
    }
    // 选择解释器
    typedef void (*Interpreter)(Thread*);
    Interpreter stdInterp;
    if (gDvm.executionMode == kExecutionModeInterpFast)
        stdInterp = dvmMterpStd;
#if defined(WITH_JIT)
    else if (gDvm.executionMode == kExecutionModeJit ||
             gDvm.executionMode == kExecutionModeNcgO0 ||
             gDvm.executionMode == kExecutionModeNcgO1)
        stdInterp = dvmMterpStd;
#endif
    else
        stdInterp = dvmInterpretPortable;
 
    // Call the interpreter
    (*stdInterp)(self);
 
    *pResult = self->interpSave.retval;
 
    /* Restore interpreter state from previous activation */
    self->interpSave = interpSaveState;
#if defined(WITH_JIT)
    dvmJitCalleeRestore(calleeSave);
#endif
    if (savedSubModes != kSubModeNormal) {
        dvmEnableSubMode(self, savedSubModes);
    }
}

```

### dalvik 解释器流程分析

dalvik 解释器有两种：Fast 解释器，Portable 解释器。选择分析 Portable 解释器，因为 Portable 解释器的可读性更好。在分析前，先看下 Portable 解释器的模型。

#### Thread Code 技术

实现解释器的一个常见思路如下代码，循环取指令，然后判断指令类型，去相应分支执行，执行完成后，再返回到 switch 执行下条指令。

```
while (*ins) {
    switch (*ins) {
        case NOP:
            break;
        case MOV:
            break;
        ......
    }
}

```

但是当每次执行一条指令，都需要重新判断下条指令类型，然后选择 switch 分支，这是个昂贵的开销。Dalvik 为了解决这个问题，引入了 Thread Code 技术。简单的说就是在执行函数之前，建立一个分发表 GOTO_TABLE，每条指令在表中有一个对应条目，条目里存放的就是处理该条指令的 handler 地址。比如 invoke-super 指令, 它的 opcode 为 6f，那么处理该条指令的 handler 地址就是：GOTO_TABLE[6f]. 那么在每条指令的解释程序末尾，都可以加上取指动作，然后 goto 到下条指令的 handler。

#### dvmInterpretPortable 源码分析

dvmInterpretPortable 是 Portable 型虚拟机的具体实现，流程如下

1.  初始化一些关于虚拟机执行环境的变量
2.  初始化分发表
3.  FINISH(0) 开始执行指令

```
void dvmInterpretPortable(Thread* self)
{
 
    DvmDex* methodClassDex;     // curMethod->clazz->pDvmDex
    JValue retval;
 
    //一些核心的状态
    const Method* curMethod;    // 要执行的方法
    const u2* pc;               // 指令计数器
    u4* fp;                     // 函数栈指针
    u2 inst;                    // 当前指令
    /* instruction decoding */
    u4 ref;                     // 用来表示类的引用
    u2 vsrc1, vsrc2, vdst;      // 寄存器索引
    /* method call setup */
    const Method* methodToCall;
    bool methodCallRange;
 
    //建立分发表
    DEFINE_GOTO_TABLE(handlerTable);
 
    //初始化上面定义的变量
    curMethod = self->interpSave.method;
    pc = self->interpSave.pc;
    fp = self->interpSave.curFrame;
    retval = self->interpSave.retval;   /* only need for kInterpEntryReturn? */
    methodClassDex = curMethod->clazz->pDvmDex;
 
    if (self->interpBreak.ctl.subMode != 0) {
        TRACE_METHOD_ENTER(self, curMethod);
        self->debugIsMethodEntry = true;   // Always true on startup
    }
 
    methodToCall = (const Method*) -1;
 
    //取出第一条指令，并且执行
    FINISH(0);                  /* fetch and execute first instruction */
 
//下面就是定义了每条指令的处理分支。
//NOP指令的处理程序：什么都不做，然后处理下条指令
HANDLE_OPCODE(OP_NOP)
    FINISH(1);
OP_END
.....

```

### invoke-super 指令实例分析

invoke-super 这条指令的 handler 如下：

```
#define GOTO_invoke(_target, _methodCallRange)                              \
    do {                                                                    \
        methodCallRange = _methodCallRange;                                 \
        goto _target;                                                       \
    } while(false)
 
HANDLE_OPCODE(OP_INVOKE_SUPER /*vB, {vD, vE, vF, vG, vA}, meth@CCCC*/)
    GOTO_invoke(invokeSuper, false);
OP_END

```

invokeSuper 这个标签定义如下：

```
//invoke-super位描述符如下：A|G|op BBBB F|E|D|C
//methodCallRange depending on whether this is a "/range" instruction.
GOTO_TARGET(invokeSuper, bool methodCallRange)
    {
        Method* baseMethod;
        u2 thisReg;
 
        EXPORT_PC();
        //取出AG的值
        vsrc1 = INST_AA(inst); 
        //要调用的method索引
        ref = FETCH(1);
        //要作为参数的寄存器的索引
        vdst = FETCH(2);        
 
        //取出this寄存器的索引，比如thisReg为3的话，表示第三个寄存器，放的是this参数。
        if (methodCallRange) {
            ILOGV("|invoke-super-range args=%d @0x%04x {regs=v%d-v%d}",
                vsrc1, ref, vdst, vdst+vsrc1-1);
            thisReg = vdst;
        } else {
            ILOGV("|invoke-super args=%d @0x%04x {regs=0x%04x %x}",
                vsrc1 >> 4, ref, vdst, vsrc1 & 0x0f);
            thisReg = vdst & 0x0f;
        }
 
        //检查this 是否为空
        if (!checkForNull((Object*) GET_REGISTER(thisReg)))
            GOTO_exceptionThrown();
 
        //解析要调用的方法
        baseMethod = dvmDexGetResolvedMethod(methodClassDex, ref);
        if (baseMethod == NULL) {
            baseMethod = dvmResolveMethod(curMethod->clazz, ref,METHOD_VIRTUAL);
            if (baseMethod == NULL) {
                ILOGV("+ unknown method or access denied");
                GOTO_exceptionThrown();
            }
        }
 
        if (baseMethod->methodIndex >= curMethod->clazz->super->vtableCount) {
            /*
             * Method does not exist in the superclass.  Could happen if
             * superclass gets updated.
             */
            dvmThrowNoSuchMethodError(baseMethod->name);
            GOTO_exceptionThrown();
        }
        methodToCall = curMethod->clazz->super->vtable[baseMethod->methodIndex];
 
#if 0
        if (dvmIsAbstractMethod(methodToCall)) {
            dvmThrowAbstractMethodError("abstract method not implemented");
            GOTO_exceptionThrown();
        }
#else
        assert(!dvmIsAbstractMethod(methodToCall) ||
            methodToCall->nativeFunc != NULL);
#endif
        LOGVV("+++ base=%s.%s super-virtual=%s.%s",
            baseMethod->clazz->descriptor, baseMethod->name,
            methodToCall->clazz->descriptor, methodToCall->name);
        assert(methodToCall != NULL);
        //调用方法
        GOTO_invokeMethod(methodCallRange, methodToCall, vsrc1, vdst);
    }
GOTO_TARGET_END

```

解析完要调用的方法后，跳转到 invokeMethod 结构来执行函数调用，invokeMethod 为要调用的函数创建虚拟寄存器栈，新的寄存器栈和之前的栈是由重叠的。然后重新设置解释器执行环境的参数，调用 FINISH(0) 执行函数

```
GOTO_TARGET(invokeMethod, bool methodCallRange, const Method* _methodToCall, u2 count, u2 regs)
{      
        //节选
        if (!dvmIsNativeMethod(methodToCall)) {
            /*
             * "Call" interpreted code.  Reposition the PC, update the
             * frame pointer and other local state, and continue.
             */
            curMethod = methodToCall;     //设置要调用的方法
            self->interpSave.method = curMethod; 
            methodClassDex = curMethod->clazz->pDvmDex;  
            pc = methodToCall->insns;     //重置pc到要调用的方法
            fp = newFp;
            self->interpSave.curFrame = fp;
#ifdef EASY_GDB
            debugSaveArea = SAVEAREA_FROM_FP(newFp);
#endif
            self->debugIsMethodEntry = true;        // profiling, debugging
            ILOGD("> pc <-- %s.%s %s", curMethod->clazz->descriptor,
                curMethod->name, curMethod->shorty);
            DUMP_REGS(curMethod, fp, true);         // show input args
            FINISH(0);                              // jump to method start
        }

```

数字壳解释器分析
--------

### 数字壳解释执行前的准备工作

进入解释器的流程为 onCreate->sub_D930->sub_3FE5C->sub_3FF5C。sub_3FF5C 真正的解释器入口，sub_D930 和 sub_3FE5C 负责执行前的准备工作。这部分准备工作和 dalvik 解释器的准备工作类似。

#### sub_D930

sub_D930 分为两部分，调用 sub_66BD4 之前为第一部分，之后为第二部分。这两部分主要做的事情如下：

*   第一部分
    1.  jni 的一些初始化工作，FindClass，GetMethodID 之类的工作
    2.  利用 java.lang.Thread.getStackTrace 获取到调用当前方法的类的类名以及函数名
*   第二部分
    1.  调用 sub_66BD4 获取一些全局信息，以及待解释函数的信息
    2.  构建解释器的虚拟寄存器栈
    3.  解析待解释函数的简单声明，将函数参数放入虚拟寄存器

主要分析第二部分，首先引入一些数据结构，这类数据结构是动态分析出来的，有些字段的含义还不清楚标记为 unkonw。

```
struct GlobalInfo {
    DexInfo *dex_info;
    vector method_list;
};
 
struct DexInfo {
    void *dexBase; //指向内存中dex的基址
    int unknow[12];
    void *dexHeader; //指向dex header，用来解析dex的。
    .....
    optable[];
};
 
struct MethodInfo {
    int dex_type_id;     //该方法在dex文件中DexTypeId列表的索引
    int dex_class_def_id;  // 该方法所在类在dex文件中DexClassDef列表索引
    int unknow;
    int codeoff;  //该方法的DexCode结构距离dex头的偏移。
};
 
struct StackInfo {
    JNIEnv * jni_env;
    int registerSize; //函数所需寄存器数量
    DexCode *dexCode;    //指向DexCode结构
    int unkonw;      
    void *registerSpace;   // 放入原始参数, malloc(4 * registerSize)
    void *registerSpace2;  // 新建一个object引用上面的参数, malloc(4 * registerSize)
    int unkonw2;       
    char key;             //解密指令的key 
    char origin_key;      //计算解密指令key所需的一个数据
}; 
```

sub_66BD4 返回的是指向 GlobalInfo 结构的指针。这个全局信息里面包含了 dex 有关的信息和待解释函数的信息。有了这个信息就可以构建解释器所需的虚拟寄存器栈，完成准备工作。

```
int __fastcall sub_D930(unsigned int a1, JNIEnv *a2, _DWORD *a3)
{
  //节选
  //获取全局信息
  global_info = sub_66BD4(v109, &v114);
  if ( v114 & 1 )
    j_j_j__ZdlPv(*(v47 + 5));
  if ( global_info && (v52 = *(global_info + 4), (*(global_info + 8) - v52) >> 2 > v4) )
  {
    v53 = v52 + 4 * v4;
    a2c = v3;
    method_info = *v53;
    dexInfo = *global_info;
    DexCode = (**global_info + *(*v53 + 12));   // DexCode
    ((*v3)->PushLocalFrame)();
    //创建并初始化StackInfo结构
    stackinfo = malloc_1(0x20u);                // 创建StackInfo结构
    v56 = *DexCode;
    *stackinfo = a2c;                           // stackinfo->jni_env
    *(stackinfo + 4) = v56;                     // stackinfo->registerSize
    *(stackinfo + 8) = DexCode;                 // stackinfo->dexCode
    *(stackinfo + 12) = 0;
    v57 = 4 * v56;
    v58 = malloc_0(4 * v56);                    // 创建虚拟寄存器栈
    *(stackinfo + 16) = v58;
    memset(v58, v57, 0);
    v59 = malloc_0(v57);                        // 创建虚拟寄存器栈
    *(stackinfo + 20) = v59;
    memset(v59, v57, 0);
    *(stackinfo + 29) = 0;
    *(stackinfo + 28) = 0;
    v60 = sub_3E268();
    v61 = DexCode;
    *(stackinfo + 28) = *DexCode ^ *(v60 + 24) ^ *(method_info + 4) ^ *method_info;  // stackinfo->key
    *(stackinfo + 29) = *(v60 + 24);
}

```

通过创建并初始化 StackInfo 结构，就完成了虚拟寄存器栈的创建，可以看到这里分配了两个虚拟寄存器栈。后面调试发现主要使用的是第二个虚拟寄存器栈。猜测这两个虚拟寄存器栈和 dalvik 拥有两个虚拟寄存器栈一样的原因一样，是一个用来执行 native 方法，一个执行 java 方法。

 

创建完虚拟寄存器栈的下一步工作就是放入函数参数。

```
int __fastcall sub_D930(unsigned int a1, JNIEnv *a2, _DWORD *a3)
{
    //节选
    v107 = stackinfo;
    proto = (*dexInfo
           + *(*dexInfo
             + *(dexInfo[4] + 60)
             + 4 * *(*dexInfo + *(dexInfo[4] + 76) + 12 * *(*dexInfo + *(dexInfo[4] + 92) + 8 * *(method_info + 4) + 2))));
    do
      v63 = proto++;                            // 方法的简单声明: VL
    while ( *v63 < 0 );
    v64 = j_j_strlen(proto);
    v99 = v64;
    v65 = *v61 - v61[1];                        // 函数内部使用寄存器个数：DexCode.registerSize - DexCode.insSize
    v105 = v65 - 1;
    v66 = v95 + 1;
    v67 = v107;
    if ( !*(method_info + 8) )
    {
      v105 = v65;
      v68 = 4 * v65;
      v69 = *(*(v107 + 20) + v68);              // 函数的第一个参数 thisReg
      v96 = *v95;
      //节选
      v111 = v68;
      v73 = (*(v67 + 20) + v68);
      v74 = *v73;
      if ( *v73 && !*(v74 + 4) )
      {
        *(v74 + 4) = 1;
        v78 = v111;
        v77 = v96;
        **(*(v67 + 20) + v111) = v96;
      }
      else
      {
        v75 = v64;
        v76 = malloc_1(8u);                     // 重新创建了一个MainActivity object，并放入第二个register space位置。
        *v76 = 0;
        *(v76 + 4) = 0;
        v77 = v96;
        *v76 = v96;
        *(v76 + 4) = 1;
        *v73 = v76;
        v67 = v107;
        v64 = v75;
        v78 = v111;
      }
      *(*(v67 + 16) + v78) = v77;               // 直接将Activity实例，放入第一个register space的相应位置、
    }
    v108 = v67;
    if ( v64 >= 2 )                             // 处理除了this之外的其他参数。
    {
      v79 = 1;
      v112 = 1;
      do
      {
        v80 = v63[v79++ + 1];
        if ( v80 > 89 )
        {
          if ( v80 == 90 )
          {
            *(*(v67 + 16) + 4 * (v112++ + v105)) = *v66;
            ++v66;
          }
        }
        else
        {
          v81 = v80 - 66;
          if ( v81 <= 0x11 )
            JUMPOUT(__CS__, *(&off_E308 + v81) + 58120);// 调用jni->newGlobalRef，构造将传递给onCreate的参数
        }
      }
      while ( v79 < v64 );
    }
 
}

```

这里是从 dex 文件中提取出函数的简要声明，onCreate 的简单声明为 VL，然后根据声明，放入参数。首先放入 this 参数，然后根据后续参数的类型，将参数放入相应位置。和 dalvik 虚拟机流程类似。

#### sub_3FE5C

sub_D930 完成了虚拟寄存器栈的构建并放入参数后，调用了 sub_3FE5C。sub_3FE5C 主要负责初始化解释器的一些状态，主要是 InterpState 结构。

```
struct InterpState {
    void *pc;
    char key;
    DexInfo *dex_info;
};
 
LOAD:0003FF18                 BL              malloc_1                  ; 创建InterpState结构
LOAD:0003FF1C                 PUSH            {R4}
LOAD:0003FF1E                 POP             {R1}
LOAD:0003FF20                 PUSH            {R0}
LOAD:0003FF22                 POP             {R4}
LOAD:0003FF24                 STR             R4, [SP,#0x74+var_24]
LOAD:0003FF26                 LDRB            R0, [R1,#0x1C]
LOAD:0003FF28                 ADDS            R6, #0x10
LOAD:0003FF2A                 STR             R6, [SP,#0x74+var_2C]
LOAD:0003FF2C                 STR             R6, [R4]                  ; InterpState->pc
LOAD:0003FF2E                 STRB            R0, [R4,#4]               ; InterpState->key
LOAD:0003FF30                 STR             R5, [R4,#8]               ; InterpState->dex_info

```

### 数字壳解释器模型

sub_D930 和 sub_3FE5C 完成了解释器的准备工作。sub_3FF5C 负责解释执行。数字壳的解释器模型就是上面提到的那种最直观的模型，循环取指令，然后判断指令类型，去相应分支执行，执行完成后，再返回到 switch 执行下条指令。

```
//R4寄存器存放的是InterpState结构
LOAD:0003FF5C                 LDRB            R1, [R4,#4]   ; 取出key，InterpState->key
LOAD:0003FF5E                 MOVS            R6, #0x31 ; '1'
LOAD:0003FF60                 PUSH            {R1}
LOAD:0003FF62                 POP             {R2}
LOAD:0003FF64                 ANDS            R2, R6
LOAD:0003FF66                 LDR             R0, loc_402C8
LOAD:0003FF68                 PUSH            {R0}
LOAD:0003FF6A                 POP             {R3}
LOAD:0003FF6C                 BICS            R3, R1
LOAD:0003FF6E                 ORRS            R3, R2
LOAD:0003FF70                 MOVS            R2, #0xEF00
LOAD:0003FF74                 LSLS            R1, R1, #8
LOAD:0003FF76                 ANDS            R2, R1
LOAD:0003FF78                 BICS            R0, R1
LOAD:0003FF7A                 ORRS            R0, R2
LOAD:0003FF7C                 EORS            R0, R3
LOAD:0003FF7E                 LDR             R1, [R4]    ; 取出pc， Interpstate->pc
LOAD:0003FF80                 LDRH            R4, [R1]    ; 取出指令
 
//执行完invoke-super后
LOAD:000463EC loc_463EC                             
LOAD:000463EC                                    
LOAD:000463EC                 LDR             R0, [R4]
LOAD:000463EE                 ADDS            R0, #6 
LOAD:000463F0                 STR             R0, [R4]  ; InterpState->pc = InterpState->pc + 6
LOAD:000463F2                 BL              loc_3FF5C ; 跳转到解释器开头，执行下一条指令。

```

### 数字壳 invoke-super 指令分析

invoke-super 指令包含了函数调用的过程，可以看到 dalvik 解释器虚拟寄存器栈是比较复杂的，设计很多数据结构。目前为止数字壳的相关结构中，并未发现类似的结构。所以想分析下数字壳的解释器是如何处理函数调用的。调试分析后发现，数字壳的解释器其实并未实现真正的函数调用，它是通过调用 jni 中的 CallVoidMethod 方法来实现函数调用。

 

处理 invoke-super 的 handler 为 sub_4878C, 流程如下：

1.  提取 invoke-super 指令中包含的 MethodID，从 dex 中获取到 DexMethodId 结构，获取函数名。
2.  从 DexMethodId 中获取 ClassId
3.  利用获取的 ClassId 可以获取到类名，利用 jni->FindClass 获取类
4.  利用 jni->GetMethod 获取函数
5.  准备函数参数
6.  利用 jni->CallVoidMehtod 调用函数，实现 invoke-method

```
int  sub_4878C(int a1, bool methodCallRange, JNIEnv *a3, DexInfo *a4, StackInfo *a5, InterpState *a6, int opNumber, void *a8)
{
  //节选
  a1a = v8;
  dex_base = *v8;
  dex_header = *(a1a + 16);
  key = *(a6 + 4);                              // 取出key
  v11 = (key << 8) | key;
  DexMethodId = dex_base + *(dex_header + 92) + 8 * (*(*a6 + 2) ^ v11);
  v13 = dex_base + *(dex_header + 60);
  methodName = (dex_base + *(v13 + 4 * *(DexMethodId + 4)));   //获取函数名
  do
    v15 = *methodName++ < 0;
  while ( v15 );
  DexMethodId2 = (dex_base + *(dex_header + 92) + 8 * (*(*a6 + 2) ^ ((key << 8) | key)));
  proto = (dex_base
         + *(v13
           + 4 * *(dex_base + *(dex_header + 68) + 4 * *(dex_base + *(dex_header + 76) + 12 * *(DexMethodId + 2) + 4))));
  do
    v17 = *proto++;                        //获取函数简单声明
  while ( v17 < 0 );
  v18 = *(*a6 + 4);
  if ( v52 == 1 )
    thisReg = v18 ^ ((key << 8) | key);     //this参数
  else
    thisReg = (v18 ^ key) & 0xF;
  v20 = 0;
  if ( v51 )
  {
    v21 = *(*(a5 + 20) + 4 * thisReg);
    if ( !v21 || (v20 = *v21) == 0 )
    {
      v30 = ((*v53)->FindClass)(v53, "java/lang/NullPointerException");
      .....
    }
  }
  arg0 = v20;                                   // this object
  v62 = v53;
  v63 = 0;
  if ( ((*v53)->ExceptionCheck)(v53) )
    v63 = 0;
  if ( v51 != 2 && v51 != 4 )
  {
    sub_610BC(a1a, v53, *DexMethodId2);         // 获取到classid所指定的类
    return sub_48AE2(v32, v33, v34, v35, a5);   // 利用jni->CallVoidMethod调用父类方法
  }
 
 
//48EAA: 构建invoke-super除this外的参数
//49770：调用jni->CallVoidMethod方法

```

数字壳比较复杂，通过分析可以学到很多东西，比如各种反调试，linker，适配很多版本的动态加载，解释器等，感谢数字公司提供的免费加固。笔记如果有错误，欢迎指正，也希望大佬们可以交流下其他 vmp 的实现思路。

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)