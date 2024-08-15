> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [evilpan.com](https://evilpan.com/2022/04/17/frida-java/)

> 前面的文章中介绍了 frida 的基础组件 frida-core，用于实现进程注入、通信和管理等功能。

前面的文章中介绍了 frida 的基础组件 `frida-core`，用于实现进程注入、通信和管理等功能。加上 frida-gum 和 gum-js 的核心能力，我们已经可以很方便地使用 JavaScript 脚本来进行代码劫持、动态跟踪等进程分析操作。

不过 frida 并不满足于此，而是又实现了针对高级语言的支持，比如 `Java`、`Objective-C`、`Swift` 等。这些额外支持实际上是在 `gum-js` 的基础上针对对应高级语言的 Runtime 进行 hack 而实现的，统一称为对应语言的 bridge。例如，针对 Java 语言的封装称为 `frida-java-bridge`，不仅实现了 Oracle JVM 的封装、还支持 Android 中的 Dalvik 虚拟机和 ART 虚拟机。本文就以 ART 为例来看看 frida 中的具体实现。

传送门:

*   [Frida Internal - Part 1: 架构、Gum 与 V8](https://evilpan.com/2022/04/05/frida-internal/)
*   [Frida Internal - Part 2: frida-core](https://evilpan.com/2022/04/09/frida-core/)
*   [Frida Internal - Part 3: frida-java-bridge 与 ART hook (本文)](https://evilpan.com/2022/04/17/frida-java/)

[frida-java-bridge](https://github.com/frida/frida-java-bridge) 就是我们在编写 frida js 脚本时使用的 `Java.use` 等接口实现。Java 接口是 Android Runtime 的封装，实现在 `index.js` 的 `class Runtime` 中。比如 `Java.perform` 就对应了 `Runtime.perform` 函数。 在 Runtime 的构造函数中分别初始化了 api、vm 和 classFactory 属性等属性:

```
_tryInitialize () {
    // ...
    this.api = getApi();
    this.vm = new VM(api);

    ClassFactory._initialize(vm, api);
    this.classFactory = new ClassFactory();
}
```

下面分别对其进行介绍。

API 是从对应 Java 虚拟机的动态库中所抽象出来的一套统一接口，用以实现对运行时、垃圾回收、堆栈管理等底层操作，是实现上层方法劫持和替换的基础设施。

在 `lib/api.js` 中指定了 API 的类型，如下所示:

```
let { getApi, getAndroidVersion } = require('./android');
try {
  getAndroidVersion();
} catch (e) {
  getApi = require('./jvm').getApi;
}
module.exports = getApi;
```

通过尝试调用 libc 的 `__system_property_get` 函数获取属性，如果成功表示当前进程为 Android 环境，即 Java 实现为 Dalvik 或者 ART 虚拟机；如果失败则认为是 Oracle 的 JVM 虚拟机实现。java-bridge 的实现是基于 inline-hook (gum-js) 对目标 Java 虚拟机中的符号进行解析、调用、替换从而实现 Java Hook 的功能，其支持的三种虚拟机及其对应动态库分别是:

*   Dalvik: libdvm.so
*   ART: libart.so
*   JVM: jvm.(dll|dylib|so)

由于三种实现大同小异，因此我们只需要关注其中一种。因为笔者对于 Android 较为感兴趣，故而重点关注 ART 下的实现。在 `lib/android.js` 中 `_getApi` 同时处理了 Dalvik 和 ART 虚拟机的情况，区分这二者的方法是判断当前进程中加载的模块是 libdvm.so 还是 libart.so。

在两种情况下都需要先获取对应动态库中的一些符号地址，对于 ART 而言，主要需要下面函数:

*   JNI_GetCreatedJavaVMs
*   art::JavaVMExt::AddGlobalRef
*   art::IndirectReferenceTable::Add
*   art::JavaVMExt::DecodeGlobal
*   art::ThreadList::SuspendAll
*   art::ThreadList::ResumeAll
*   art::ClassLinker::VisitClasses
*   art::ClassLinker::VisitClassLoaders
*   art::gc::Heap::VisitObjects
*   art::gc::Heap::GetInstances
*   art::StackVisitor::StackVisitor
*   art::StackVisitor::WalkStack
*   art::StackVisitor::GetMethod
*   art::StackVisitor::DescribeLocation
*   art::StackVisitor::GetCurrentQuickFrameInfo
*   art::Thread::GetLongJumpContext
*   art::mirror::Class::GetDescriptor
*   art::ArtMethod::PrettyMethod
*   art::ArtMethod::PrettyMethodNullSafe
*   art::Thread::CurrentFromGdb
*   art::mirror::Object::Clone
*   art::Dbg::SetJdwpAllowed
*   art::Dbg::ConfigureJdwp
*   art::InternalDebuggerControlCallback::StartDebugger
*   art::Dbg::StartJdwp
*   art::Dbg::GoActive
*   art::Dbg::RequestDeoptimization
*   art::Dbg::ManageDeoptimization
*   art::Instrumentation::EnableDeoptimization
*   art::Instrumentation::DeoptimizeEverything
*   art::Runtime::DeoptimizeBootImage
*   art::Instrumentation::Deoptimize
*   art::jni::JniIdManager::DecodeMethodId
*   art::interpreter::GetNterpEntryPoint
*   art::Monitor::TranslateLocation

以及两个用于判断调试器状态的变量:

*   art::Dbg::gRegistry - 判断 JDWP 是否启动
*   art::Dbg::gDebuggerActive - 判断是否正在调试

值得一提的是，这两个变量也可以用在安全 SDK 中作为一种隐秘的反调试手法。

上面的函数通过在模块中的符号中查找到对应地址后 (如 `Module.enumerateExports("libart.so")`)，便将其转换为 `NativeFunction` 类然后保存到 temporaryApi 字典中留作备用。

在找到上述的函数和变量地址后，就可以通过这些 Native 函数获取需要的信息，首先第一步是获取当前进程中所有创建的 Java 虚拟机。这里使用的是 **JavaScript** 代码去调用，乍看起来有点别扭，但这是做 Native 调用的常规操作:

```
const vms = Memory.alloc(pointerSize);
const vmCount = Memory.alloc(jsizeSize);
temporaryApi.JNI_GetCreatedJavaVMs(vms, 1, vmCount);
if (vmCount.readInt() === 0)
  return null;
temporaryApi.vm = vms.readPointer();
```

该函数原型为:

```
jint JNI_GetCreatedJavaVMs(JavaVM **vmBuf, jsize bufLen, jsize *nVMs);
```

虽然参数说可以返回多个 VM，但在单个进程中是不支持创建多个 Java 虚拟机的，因此这里也只用获取一个 VM 即可。获取到的 Java 虚拟机，(即 `JavaVM *` 地址) 保存在 temporaryApi.vm 变量中，这是我们一系列后续操作的基础。

通过获得的 JavaVM 指针，我们可以获取一系列重要数据结构的地址，比如:

*   art::Runtime
*   art::Instrumentation
*   art::Heap
*   art::ThreadList
*   art::ClassLinker
*   各个 trampoline 的地址
*   …

其中的一些是通过结构体的固定偏移来获得，比如 art::Runtime 就在 JavaVM 结构的第二个位置:

```
struct _JavaVM {
    const struct JNIInvokeInterface* functions;
}

class JavaVMExt : public JavaVM {
public:
// 一些函数 ...
Runtime* const runtime_;
}
```

用 gdb 也可以进行验证:

```
(gdb) ptype /o JavaVM
type = struct _JavaVM {
/*    0      |     8 */    const struct JNIInvokeInterface *functions;

                           /* total size (bytes):    8 */
                         }

(gdb) ptype /o art::JavaVMExt
/* offset    |  size */  type = class art::JavaVMExt : public JavaVM {
                         private:
/*    8      |     8 */    class art::Runtime * const runtime_;
...
```

但是对于一些复杂的数据结构就没有那么简单了，比如在 `art::Runtime` 结构体中查找 heap、threadList、classLinker 等属性的时候，就需要先搜索到特定的属性位置，然后通过不同版本的相对偏移去定位其他属性:

```
(gdb) ptype /o JavaVM
type = struct _JavaVM {
/*    0      |     8 */    const struct JNIInvokeInterface *functions;

                           /* total size (bytes):    8 */
                         }

(gdb) ptype /o art::JavaVMExt
/* offset    |  size */  type = class art::JavaVMExt : public JavaVM {
                         private:
/*    8      |     8 */    class art::Runtime * const runtime_;
...
```

由于 JavaVM 的地址我们已知, 故而可以在 Runtime 的起始地址处开始一直往前搜索，发现匹配后就可以确定 `java_vm_` 属性的位置，进而可以根据相对偏移往前找到 **InternTable**、**ClassLinker**、**ThreadList**、**gc::Heap** 等属性地址。之所以这样查找是因为 art::Runtime 这个数据结构相当复杂，总大小有 2000 字节以上，搜索 `java_vm_` 的时候也做了一些优化，从第 50 个指针大小位置处开始搜索，以加快搜索速度。

上面这些偏移都还算好找，但是在寻找 **Instrumentation** 偏移的时候就需要一些额外的技巧，当前 frida 使用的方法是先找到 **libart.so** 的 `MterpHandleException` 函数地址，由于该函数中会使用到 `Runtime->instrumentation` 的数据，因此逐条解析汇编指令即可定位到对应偏移。

该函数定义在 mterp.cc 中，代码如下:

```
class Runtime {
// ...
gc::Heap* heap_;                // <-- we need to find this
std::unique_ptr<ArenaPool> jit_arena_pool_;    //  <----- API level >= 24
std::unique_ptr<ArenaPool> arena_pool_;        //      __
std::unique_ptr<ArenaPool> low_4gb_arena_pool_;//  <--|__ API level >= 23
std::unique_ptr<LinearAlloc> linear_alloc_;    //      \_
size_t max_spins_before_thin_lock_inflation_;
MonitorList* monitor_list_;
MonitorPool* monitor_pool_;
ThreadList* thread_list_;        // <--- and these
InternTable* intern_table_;      // <--/
ClassLinker* class_linker_;      // <-/
SignalCatcher* signal_catcher_;
std::unique_ptr<jni::JniIdManager> jni_id_manager_; // <- API level >= 30 or Android R Developer Preview
bool use_tombstoned_traces_;     // <-------------------- API level 27/28
std::string stack_trace_file_;   // <-------------------- API level <= 28
JavaVMExt* java_vm_;             // <-- so we find this then calculate our way backwards
// ...
}
```

虽然道理很简单，但实际操作时候需要区分不同架构的指令解析方法，frida 中支持以下几种架构:

```
extern "C" size_t MterpHandleException(Thread* self, ShadowFrame* shadow_frame)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK(self->IsExceptionPending());
  const instrumentation::Instrumentation* const instrumentation =
      Runtime::Current()->GetInstrumentation();
  return MoveToExceptionHandler(self, *shadow_frame, instrumentation);
}
```

以 **arm64** 为例，观察该函数，可以通过寻找 `add` 指令来定位偏移，因为 `GetInstrumentation` 等函数实际上都被 inline 优化成了直接指令，如下所示，实际偏移即为 `0x2b8`:

```
const instrumentationOffsetParsers = {
  ia32: parsex86InstrumentationOffset,
  x64: parsex86InstrumentationOffset,
  arm: parseArmInstrumentationOffset,
  arm64: parseArm64InstrumentationOffset
};
```

frida 的代码实现为:

```
(gdb) disassemble MterpHandleException
Dump of assembler code for function MterpHandleException(art::Thread*, art::ShadowFrame*):
   0x00000000007551ac <+0>:     stp     x29, x30, [sp, #-16]!
   0x00000000007551b0 <+4>:     mov     x29, sp
   0x00000000007551b4 <+8>:     adrp    x8, 0xa17000 <_ZN3art2gc9allocator8RosAlloc27dedicated_full_run_storage_E+3464>
   0x00000000007551b8 <+12>:    ldr     x8, [x8, #3280]
   0x00000000007551bc <+16>:    add     x2, x8, #0x2b8 // <-- 这里即为所求偏移！
   0x00000000007551c0 <+20>:    bl      0x3e1170 <art::interpreter::MoveToExceptionHandler(art::Thread*, art::ShadowFrame&, art::instrumentation::Instrumentation const*)>
   0x00000000007551c4 <+24>:    and     x0, x0, #0x1
   0x00000000007551c8 <+28>:    ldp     x29, x30, [sp], #16
   0x00000000007551cc <+32>:    ret
End of assembler dump.
```

其中 `insn` 是 `Instruction` 类型，是 frida 中用以解析汇编指令的关键类，这里不再赘述。另外一个属性 `jni_ids_indirection_` 的查找也是类似套路，有点类似漏洞利用过程中寻找 gadget 的过程。

根据上面获取到的各个重要字段在 **art::Runtime** 中的偏移后，就可以计算出这些属性在目标内存中的实际地址了。另外一个重要的数据结构是 **art::ClassLinker**，其中包含了 ART hook 所需要的几个重要的共享跳板代码地址，即 trampoline，如下所示:

```
(gdb) disassemble MterpHandleException
Dump of assembler code for function MterpHandleException(art::Thread*, art::ShadowFrame*):
   0x00000000007551ac <+0>:     stp     x29, x30, [sp, #-16]!
   0x00000000007551b0 <+4>:     mov     x29, sp
   0x00000000007551b4 <+8>:     adrp    x8, 0xa17000 <_ZN3art2gc9allocator8RosAlloc27dedicated_full_run_storage_E+3464>
   0x00000000007551b8 <+12>:    ldr     x8, [x8, #3280]
   0x00000000007551bc <+16>:    add     x2, x8, #0x2b8 // <-- 这里即为所求偏移！
   0x00000000007551c0 <+20>:    bl      0x3e1170 <art::interpreter::MoveToExceptionHandler(art::Thread*, art::ShadowFrame&, art::instrumentation::Instrumentation const*)>
   0x00000000007551c4 <+24>:    and     x0, x0, #0x1
   0x00000000007551c8 <+28>:    ldp     x29, x30, [sp], #16
   0x00000000007551cc <+32>:    ret
End of assembler dump.
```

`intern_table_` 的地址我们已经在 art::Runtime 中获得了，因此可以用前面类似的搜索方法依次找出这几个 trampoline 的地址。需要注意的是在 Android 5.x 和 6.x 之后 ClassLinker 的数据结构有所不同，frida 针对这两种情况都分别进行了处理。

quick trampoline 一般是指被编译成 native 代码后的字节码在运行过程中所使用到的跳转地址表，比如 `quick_resolution_trampoline_` 所指向的 stub 作用就是给 (native 代码) 第一次调用某个方法时候解析指定方法，同理 `generic_jni_trampoline` 就是用来跳转到 JNI 方法 (代码) 的 stub，`quick_to_interpreter_bridge_trampoline_` 则是用来从 native 代码跳转到解释器执行的 stub。

在获取完各个地址后，`android.js` 中初始化了一个全局的变量 artController，其中主要包含一个 CModule，用以保存、搜索、当前进程的 hook 方法信息，方法的值是 ArtMethod 指针，数据保存在全局的线程安全哈希表中。 C 代码的部分关键实现如下:

```
function parseArm64InstrumentationOffset (insn) {
  if (insn.mnemonic !== 'add') {
    return null;
  }

  const ops = insn.operands;
  if (ops.length !== 3) {
    return null;
  }

  const op2 = ops[2];
  if (op2.type !== 'imm') {
    return null;
  }

  return op2.value;
}
```

同时在 CModule 中还定义若干个 native 方法用以实现一些高频调用的函数 hook，比如:

*   on_interpreter_do_call
*   on_art_method_get_oat_quick_method_header
*   on_art_method_pretty_method
*   on_leave_gc_concurrent_copying_copying_phase

最后获取了 C++ 的 new 和 delete 函数地址，以及根据运行时确定 `MethodMangler` 所属类，完成整个 API 的初始化。MethodMangler 类是对一个指定 Java 方法的封装，实现了调用、劫持、还原等操作，在后文还会继续进行介绍。

VM 顾名思义，是对 Java 虚拟机的封装，更准确地说是实现了对 `JavaVM *` 的封装，包括下面的接口:

*   attachCurrentThread
*   detachCurrentThread
*   getEnv
*   …

值得一提的是初始化的地方:

```
class ClassLinker {
...
InternTable* intern_table_;                        // <-- We find this then calculate our way forwards
const void* quick_resolution_trampoline_;
const void* quick_imt_conflict_trampoline_;
const void* quick_generic_jni_trampoline_;         // <-- ...to this
const void* quick_to_interpreter_bridge_trampoline_;
...
}
```

`handle` 是构造函数传入的 `api.vm`，即前文说到的 `JNI_GetCreatedJavaVMs` 调用的结果，其指针类型是 `JavaVM *`，内存布局如下所示:

```
void init (void) {
  g_mutex_init (&lock);
  methods = g_hash_table_new_full (NULL, NULL, NULL, NULL);
  replacements = g_hash_table_new_full (NULL, NULL, NULL, NULL);
}

void set_replacement_method (gpointer original_method, gpointer replacement_method) {
  g_mutex_lock (&lock);
  g_hash_table_insert (methods, original_method, replacement_method);
  g_hash_table_insert (replacements, replacement_method, original_method);
  g_mutex_unlock (&lock);
}
```

这是兼容 C 和 C++ 的写法，由于 ART 虚拟机是用 C++ 写的，因此要想找到这几个函数的地址，首先需要获得 `JNIInvokeInterface_` 的地址，其正好是 `JavaVM` 的第一个元素。然后再根据下面的偏移即可获取到 `AttachCurrentThread` 等函数的实际地址。

```
function initialize () {
  const vtable = handle.readPointer();
  const options = {
    exceptions: 'propagate'
  };
  attachCurrentThread = new NativeFunction(vtable.add(4 * pointerSize).readPointer(), 'int32', ['pointer', 'pointer', 'pointer'], options);
  detachCurrentThread = new NativeFunction(vtable.add(5 * pointerSize).readPointer(), 'int32', ['pointer'], options);
  getEnv = new NativeFunction(vtable.add(6 * pointerSize).readPointer(), 'int32', ['pointer', 'pointer', 'int32'], options);
}
```

在 Runtime 中最后一个初始化的属性是 `ClassFactory`，虽然它其貌不扬，但却是很多核心操作的实施者，比如通过类名获取目标类的接口 `Java.use`，又或者是获取某个类在内存中实例的接口 `Java.choose`，等等。

在平时对一些 APP 进行测试的时候，我们会发现 frida 对于一些加壳的 APP 也同样能够进行注入和劫持，即便是类似 libjiagu 这种动态修改 ClassLoader 的壳，那么 frida 是如何查找类的呢？或者说，是如何找到被加固的类所属的 ClassLoader 的呢？还有 `Java.choose` 为什么可以找到类的示例，原理是什么？……

了解 frida 的人可能都知道，我们一般使用 `Java.use` 来获取目标类的 Wrapper，其实现为:

```
#ifdef __cplusplus
typedef JavaVM_ JavaVM;
#else
typedef const struct JNIInvokeInterface_ *JavaVM;
#endif
struct JavaVM_ {
    const struct JNIInvokeInterface_ *functions;
#ifdef __cplusplus

    jint DestroyJavaVM() {
        return functions->DestroyJavaVM(this);
    }
    jint AttachCurrentThread(void **penv, void *args) {
        return functions->AttachCurrentThread(this, penv, args);
    }
    jint DetachCurrentThread() {
        return functions->DetachCurrentThread(this);
    }

    jint GetEnv(void **penv, jint version) {
        return functions->GetEnv(this, penv, version);
    }
    jint AttachCurrentThreadAsDaemon(void **penv, void *args) {
        return functions->AttachCurrentThreadAsDaemon(this, penv, args);
    }
#endif
};
```

可能很多人都不知道 use 还可以携带第二个 options 参数，可用于指定是否缓存查找的结果。除去缓存的逻辑，如果没有特殊指定 loader，那么将会使用 `makeBasicClassHandleGetter` 来获取对应类的查找方法:

```
struct JNIInvokeInterface_ {
    void *reserved0;
    void *reserved1;
    void *reserved2;

    jint (JNICALL *DestroyJavaVM)(JavaVM *vm);
    jint (JNICALL *AttachCurrentThread)(JavaVM *vm, void **penv, void *args);
    jint (JNICALL *DetachCurrentThread)(JavaVM *vm);
    jint (JNICALL *GetEnv)(JavaVM *vm, void **penv, jint version);
    jint (JNICALL *AttachCurrentThreadAsDaemon)(JavaVM *vm, void **penv, void *args);
};
```

这里需要注意的是使用了 `env.findClass` 来获取类 handle，从名称看也能猜出这是 `env->FindClass` 方法，所以 handle 本身是一个 `jclass` 封装后的对象。`_make` 方法简化如下:

```
use (className, options = {}) {
  const allowCached = options.cache !== 'skip';

  let C = allowCached ? this._getUsedClass(className) : undefined;
  if (C === undefined) {
    try {
      const env = vm.getEnv();

      const { _loader: loader } = this;
      const getClassHandle = (loader !== null)
        ? makeLoaderClassHandleGetter(className, loader, env)
        : makeBasicClassHandleGetter(className);

      C = this._make(className, getClassHandle, env);
    } finally {
      if (allowCached) {
        this._setUsedClass(className, C);
      }
    }
  }

  return C;
}
```

上面省略了一些初始化和异常处理的代码，重要的是最后两个语句，ensureClassInitialized 确保对应类被 Java 虚拟机成功加载:

```
function makeBasicClassHandleGetter (className) {
  const canonicalClassName = className.replace(/\./g, '/');

  return function (env) {
    const tid = getCurrentThreadId();
    ignore(tid);
    try {
      return env.findClass(canonicalClassName);
    } finally {
      unignore(tid);
    }
  };
}
```

实现上就是通过 JNI 函数 `GetFieldID` 来试着取一下目标类的属性，即便该属性不存在，也会触发该类的初始化，当然还需要清理一下 `NoSuchFieldException` 异常。

`env->FindClass` 仅能找到一个 jclass，frida 在此基础上封装了一个类模型，使用 `ClassModel.build` 进行创建，其定义在 `lib/class-model.js` 中。在 Model 里定义了一个 CModule，与 Native 交互比较多或者调用频繁的代码直接使用 C 语言实现 (这里主要原因是前者)，包含的函数有:

*   model_new/has/find/list - 用于创建、查找、列举自定义的 `Model *` 信息；
*   enumerate_methods_art/jvm - 用于获取指定类的所有方法；
*   dalloc - 使用 `g_free` 释放堆内存；

看过 ART 源码的同学应该比较熟悉，`jclass` 本质上是对应 `art::mirror::Class` 在内存中的地址，在 AOSP 中一般通过下述方式进行转换和使用:

```
_make (name, getClassHandle, env) {
  const C = makeClassWrapperConstructor();
  const proto = Object.create(Wrapper.prototype, { ... });
  C.prototype = proto;

  const classWrapper = new C(null);
  proto.$w = classWrapper;

  const h = classWrapper.$borrowClassHandle(env);
  const classHandle = h.value;

  ensureClassInitialized(env, classHandle);
  proto.$l = ClassModel.build(classHandle, env);

  return classWrapper;
}
```

有了 `mirror::Class` 之后，就可以通过以下的字段去进一步获取类相关信息:

```
function ensureClassInitialized (env, classRef) {
  const api = getApi();
  if (api.flavor !== 'art') {
    return;
  }

  env.getFieldId(classRef, 'x', 'Z');
  env.exceptionClear();
}
```

frida 中根据不同版本的 ART 写死了 `ifields_` 的偏移，并以此往后计算其他字段，但是 `copied_methods_offset_` 除外，因为其位置并不固定在上述字段具体位置，因此也需要通过不同的 ART 版本进行确定。

*   **ifields_** 表示该类的实例属性 (instance fields)，不包含父类所定义的属性。该值为一个指针，指向一个带有长度前缀的 `ArtFields` 数组；
*   **methods_** 指针同样是带长度前缀的数组，元素类型为 `ArtMethod`，其中包括所有该类定义的方法。
    *   `virtual_methods_offset_` 之前的是直接方法；
    *   `[virtual_methods_offset_, copied_methods_offset_)` 之间的是抽象方法；
    *   `copied_methods_offset_` 之后的是从接口复制过来的默认方法；
*   **sfields_** 和 ifields 类似，区别是表示静态属性；

frida 中对类模型的封装是如下:

```
static void SomeFunc(JNIEnv *env, jclass javaClass) {
    ScopedObjectAccess soa(env);
    ObjPtr<mirror::Class> c = soa.Decode<mirror::Class>(javaClass);
}
```

`cm.new` 对应 CModule 的 C 函数:

```
// C++ mirror of java.lang.Class
class MANAGED Class final : public Object {
  // ...
  uint64_t ifields_;
  uint64_t methods_;
  uint64_t sfields_;
  uint32_t access_flags_;
  // ...
  uint16_t copied_methods_offset_;
}
```

可见，对应 Class 的方法和属性都会保存在 Model 中的哈希表中，在我们使用对应 ClassWrapper 访问属性时候就可以通过接口进行快速地查找。注意这里哈希表的 key 是方法名称，value 是方法类型、`jmethodId` 以及重载信息，后续需要调用或者 hook 对应方法的时候才进行实际的方法模型构建操作。

前面介绍完了 `Java.use` 的具体实现，即类的查找和封装操作，下一步自然是针对方法的实现分析。前面分析 API 初始化的末尾处我们看到指定了 MethodMangler 为 `ArtMethodMangler` 作为方法的封装，本节介绍其具体实现。

MethodMangler 同样是 API 定义的一个工具类，主要功能是用于实现 Java 方法封装和 Hook 操作。

在 Android 中分为 ART 和 Dalvik 两种实现，二者都提供了相同的接口:

*   constructor (methodId)
*   replace (impl, isInstanceMethod, argTypes, vm, api)
*   revert (vm)
*   resolveTarget (wrapper, isInstanceMethod, env, api)

每次 hook 指定方法的时候都会使用对应方法的 methodId 构造一个 MethodMangler，然后使用 replace 函数来替换目标方法实现；revert 则用来恢复原来的方法。

回忆使用 frida 进行 Android hook 的典型操作:

```
class Model {
  static build (handle, env) {
    ensureInitialized(env);

    return unwrap(handle, env, object => {
      return new Model(cm.new(handle, object, env));
    });
  }
}
```

类初始化的部分前面已经介绍过了，第二行通过指定方法的 implementation 属性赋值来实现函数替换的操作本质上就是通过 MethodMangler 完成的，代码在 `lib/class-factory.js`，关键部分如下:

```
Model * model_new (jclass class_handle, gpointer class_object, JNIEnv * env) {
  model = g_new (Model, 1);

  members = g_hash_table_new_full (g_str_hash, g_str_equal, g_free, g_free);
  model->members = members;

  elements = read_art_array (class_object, art_api.class_offset_methods, sizeof (gsize), NULL);
  n = *(guint16 *) (class_object + art_api.class_offset_copied_methods_offset);
  // 读取方法
  for (i = 0; i != n; i++) {
    jmethodID id;
    guint32 access_flags;
    jboolean is_static;
    jobject method, name;
    const char * name_str;
    jint modifiers;

    id = elements + (i * art_api.method_size);

    access_flags = *(guint32 *) (id + art_api.method_offset_access_flags);
    if ((access_flags & kAccConstructor) != 0)
      continue;
    is_static = (access_flags & kAccStatic) != 0;
    method = to_reflected_method (env, class_handle, id, is_static);
    name = call_object_method (env, method, java_api.method.get_name);
    name_str = get_string_utf_chars (env, name, NULL);
    modifiers = access_flags & 0xffff;

    model_add_method (model, name_str, id, modifiers);

  }
  // 读取属性
  for (field_array_cursor = 0; field_array_cursor != G_N_ELEMENTS (field_arrays); field_array_cursor++) {
    // ...
    model_add_field (model, name_str, id, modifiers);
  }
  return model;
}
```

当对目标方法赋值时，调用了 set 方法，首先会判断是否之前有替换过，如果有的话直接将原 hook 函数替换，从这里可以看出 frida 在 针对 Java 目前还没有实现多重 hook；对于未 hook 情况，会创建对应的 MethodMangler 并保存到 `Method._r._m` 中。

值得注意的是 `Method._p` 属性，这是一个数组，依次包含对应方法的信息:

<table><thead><tr><th>idx</th><th>meaning</th></tr></thead><tbody><tr><td>0</td><td>methodName</td></tr><tr><td>1</td><td>classWrapper</td></tr><tr><td>2</td><td>type</td></tr><tr><td>3</td><td>methodId</td></tr><tr><td>4</td><td>retType</td></tr><tr><td>5</td><td>argTypes</td></tr><tr><td>6</td><td>jniCall</td></tr><tr><td>7</td><td>…</td></tr></tbody></table>

`replacement` 将用户提供的 hook JavaScript 方法封装后转换成一个 `NativeCallbacks` 进行后续处理，而实际替换则使用 `mangler.replace` 进行实现。

上面传给 MehtodMangler 的 methodId 对应 JNI 的 jmethodId，那么问题来了，jmethodId 如何转换成 `ArtMethod` 指针呢？直觉认为应该和 `jclass` 差不多，直接强制转换就行了，很可惜这个直觉只对了一半，对于旧版本的 ART 这个猜测是正确的，但是在新版本中进行了部分重构。frida 中使用的是之前保存的一个 ART 函数 `art::jni::JniIdManager::DecodeMethodId` 来实现。

通过查看 Android 源码可以发现，该方法在一部分情况下是可以直接转换的，但是当使用摇色子模式且 jmethodId 是奇数时，会使用一个静态的哈希表来获取对应的原始指针，如下所示:

```
const Activity = Java.use("com.evilpan.MainActivity");
Activity.onCreate.implementation = function(bundle) {
    console.log("We're in!");
    return this.onCreate(bundle);
}
```

由于转换方式相对繁琐，因此 frida 没有自己计算，而是通过 JniIdManager 的成员方法来辅助获取。

回顾上文介绍的 `implementaion.set`，其中 Java 方法替换主要就是通过 `MethodMangler` 的 `replace` 的方法实现的，这其中是 frida 进行 ART hook 的主要逻辑，需要我们重点关注。

其实在之前的文章 “[ART 在 Android 安全攻防中的应用](https://evilpan.com/2021/12/26/art-internal/)” 中已经简要介绍过了 ART Hook 的大致流程。这里再深入介绍一次，当然主要原因是我自己忘得差不多了，学而时习之，不亦悦乎？

在 ART 虚拟机中，对于方法的调用，大部分会调用到 `ArtMethod::Invoke`，因此我们可以以此为起点理解实际调用的过程。注意是大部分而不是所有，后面会解释为什么。

Invoke 的核心逻辑如下:

```
methodPrototype = Object.create(Function.prototype, {
    methodName: {
        enumerable: true,
        get () {
        return this._p[0];
        }
    },
    // holder: {},
    // type: {},
    // handle: {},
    implementation: {
        enumerable: true,
        get () {
            const replacement = this._r;
            return (replacement !== undefined) ? replacement : null;
        },
        set (fn) {
            const params = this._p;
            const holder = params[1];
            const type = params[2];

            if (type === CONSTRUCTOR_METHOD) {
                throw new Error('Reimplementing $new is not possible; replace implementation of $init instead');
            }

            const existingReplacement = this._r;
            if (existingReplacement !== undefined) {
                holder.$f._patchedMethods.delete(this);

                const mangler = existingReplacement._m;
                mangler.revert(vm);

                this._r = undefined;
            }

            if (fn !== null) {
                const [methodName, classWrapper, type, methodId, retType, argTypes] = params;

                const replacement = implement(methodName, classWrapper, type, retType, argTypes, fn, this);
                const mangler = makeMethodMangler(methodId);
                replacement._m = mangler;
                this._r = replacement;

                mangler.replace(replacement, type === INSTANCE_METHOD, argTypes, vm, api);

                holder.$f._patchedMethods.add(this);
            }
        }
    },
    // returnType: {},
    // argumentTypes: {},
    // canInvokeWith: {},
    // clone: {},
    // invoke: {}
});
```

主要分为两种情况，一种是 ART 未初始化完成或者系统配置强制以解释模式运行，此时则进入解释器；另一种情况是有 native 代码时，比如 JNI 代码、OAT 提前编译过的代码或者 JIT 运行时编译过的代码以及代理方法等，此时则直接跳转到 invoke_stub 去执行。

对于解释执行的情况，也细分为两种情况，一种是真正的解释执行，不断循环解析 CodeItem 中的每条指令并进行解析；另外一种是在当前解释执行遇到 native 方法时，这种情况一般是遇到了 JNI 函数，这时则通过 `method->GetEntryPointFromJni()` 获取对应地址进行跳转，所跳转的地址为:

```
template <typename ArtType> ArtType* JniIdManager::DecodeGenericId(uintptr_t t) {
  if (Runtime::Current()->GetJniIdType() == JniIdType::kIndices && (t % 2) == 1) {
    ReaderMutexLock mu(Thread::Current(), *Locks::jni_id_lock_);
    size_t index = IdToIndex(t);
    DCHECK_GT(GetGenericMap<ArtType>().size(), index);
    return GetGenericMap<ArtType>().at(index);
  } else {
    DCHECK_EQ((t % 2), 0u) << "id: " << t;
    return reinterpret_cast<ArtType*>(t);
  }
}

ArtMethod* JniIdManager::DecodeMethodId(jmethodID method) {
  return DecodeGenericId<ArtMethod>(reinterpret_cast<uintptr_t>(method));
}
```

而对于快速执行的模式是跳转到 stub 代码，以非静态方法为例，该 stub 定义在 **art/runtime/arch/arm64/quick_entrypoints_arm64.S** 文件中，大致作用是将参数保存在对应寄存器中，然后跳转到实际的地址执行:

```
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result, const char* shorty) {
    if (UNLIKELY(!runtime->IsStarted() || (self->IsForceInterpreter() && !IsNative() && !IsProxyMethod() && IsInvokable()))) {
        if (IsStatic()) {
            art::interpreter::EnterInterpreterFromInvoke(
                self, this, nullptr, args, result, /*stay_in_interpreter=*/ true);
        } else {
            mirror::Object* receiver = reinterpret_cast<StackReference<mirror::Object>*>(&args[0])->AsMirrorPtr();
            art::interpreter::EnterInterpreterFromInvoke(self, this, receiver, args + 1, result, /*stay_in_interpreter=*/ true);
        }
  } else {
    if (!IsStatic()) {
        (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
    } else {
        (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
    }
  }
}
```

`x0` 的值保存着 ArtMethod 的地址，实际跳转的是对于 ArtMethod 的某个偏移，这个偏移定义在 **art/tools/cpp-define-generator/art_method.def**:

```
class ArtMethod final {
// ...
struct PtrSizedFields {
    // Depending on the method type, the data is
    //   - native method: pointer to the JNI function registered to this method
    //                    or a function to resolve the JNI function,
    //   - resolution method: pointer to a function to resolve the method and
    //                        the JNI function for @CriticalNative.
    //   - conflict method: ImtConflictTable,
    //   - abstract/interface method: the single-implementation if any,
    //   - proxy method: the original interface method or constructor,
    //   - other methods: during AOT the code item offset, at runtime a pointer
    //                    to the code item.
    void* data_;

    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into
    // the interpreter.
    void* entry_point_from_quick_compiled_code_;
} ptr_sized_fields_;
// ...
};
```

对应的正好是 `entry_point_from_quick_compiled_code_` 属性的偏移:

```
.macro INVOKE_STUB_CALL_AND_RETURN

    REFRESH_MARKING_REGISTER
    REFRESH_SUSPEND_CHECK_REGISTER

    // load method-> METHOD_QUICK_CODE_OFFSET
    ldr x9, [x0, #ART_METHOD_QUICK_CODE_OFFSET_64]
    // Branch to method.
    blr x9

    // Pop the ArtMethod* (null), arguments and alignment padding from the stack.
    mov sp, xFP
    // ...
.endm
```

因此，不管是解释模式还是其他模式，只要目标方法有 native 代码，那么该方法的代码地址都是会保存在 `entry_point_from_quick_compiled_code_` 字段，只不过这个字段的含义在不同的场景中略有不同。

所以我们若想要实现 ART Hook，理论上只要找到对应方法在内存中的 ArtMethod 地址，然后替换其 entrypoint 的值即可。但是前面说过，并不是所有方法都会走到 `ArtMethod::Invoke`。比如对于系统函数的调用，OAT 优化时会直接将对应系统函数方法的调用替换为汇编跳转，跳转的目的就是就是对应方法的 entrypoint，因为 `boot.oat` 由 zygote 加载，对于所有应用而言内存地址都是固定的，因此 ART 可以在优化过程中省略方法的查找过程从而直接跳转。

在这样的前提下，再让我们回到 frida 的代码，看它如何能实现针对包括系统函数在内所有方法的 hook。

首先 MethodMangler 接受的是 JNI 层的 `jmethodId`，通过前面介绍过的转换方法获取到了原始的 `ArtMethod` 指针，并保存到 `methodId` 属性中:

```
ASM_DEFINE(ART_METHOD_QUICK_CODE_OFFSET_64,
           art::ArtMethod::EntryPointFromQuickCompiledCodeOffset(art::PointerSize::k64).Int32Value())
```

当然 ArtMethod 结构体在不同版本中也是不同的，因此 frida 使用了一种特别的搜索方法来确定 jniCode(即`data_`)、quickCode、interpreterCode、accessFlag 的偏移，以及 ArtMethod 结构的大小。这些信息保存在 `originalMethod` 字段。

在执行方法替换的时候，会将原方法拷贝一份，地址保存在 `replacementMethodId` 中，然后对拷贝的方法进行 patch，主要修改以下几个字段:

```
ASM_DEFINE(ART_METHOD_QUICK_CODE_OFFSET_64,
           art::ArtMethod::EntryPointFromQuickCompiledCodeOffset(art::PointerSize::k64).Int32Value())
```

jniCode 替换为用户指定的 js 函数封装而成的 NativeFunction，并将 accessFlags 设置为 `kAccNative`，即 JNI 方法。quickCode 和 interpreterCode 分别是 Quick 模式和解释器模式的入口，替换为了上文中查找保存的 trampoline，令 Quick 模式跳转到 JNI 入口，解释器模式跳转到 Quick 代码，这样就实现了该方法的拦截，每次执行都会当做 JNI 函数执行到 jniCode 即我们替换的代码中。

这样就完成了吗？基本功能已经实现了，但是还要处理一些特殊的情况。比如防止解释器模式进入 `fast_path`:

```
static constexpr MemberOffset EntryPointFromQuickCompiledCodeOffset(PointerSize pointer_size) {
  return MemberOffset(PtrSizedFieldsOffset(pointer_size) + OFFSETOF_MEMBER(
      PtrSizedFields, entry_point_from_quick_compiled_code_) / sizeof(void*)
          * static_cast<size_t>(pointer_size));
}
```

另外，在 Android 8.0 后的 ART 中，使用了新一代的解释器 [Nterp(Next-Generation Interpreter)](https://source.android.com/devices/tech/dalvik/improvements#interpreter-performance)，直接使用汇编实现以提高执行性能和内存使用。

如果待 hook 方法是解释执行，且使用了 nterp，那么 frida 会将其入口替换为 `art_quick_to_interpreter_bridge` 来跳出解释器，并进入到实际的 quickCode 去执行，从而让我们的方法替换功能保持完整。

```
class ArtMethodMangler {
  constructor (opaqueMethodId) {
    const methodId = unwrapMethodId(opaqueMethodId);

    this.methodId = methodId;
    this.originalMethod = null;
    this.hookedMethodId = methodId;
    this.replacementMethodId = null;

    this.interceptor = null;
  }
```

虽然此时我们已经将目标 ArtMethod 改成了 Native 方法，且 JNI 的入口指向我们的 hook 函数，但如果该方法已经被 OAT 或者 JIT 优化成了二进制代码，此时在字节码层调用 `invoke-xxx` 时会通过方法的 `entry_point_from_quick_compiled_code_` 直接跳转到 native 代码执行，而不是 `quick_xxx_trampoline`。

因此对于这种情况，我们可以将 entrypoint 的地址重新指向 trampoline，但如前文所说，对于系统函数而言，其地址已知，因此调用方被优化后很可能直接就调转到了对应的 native 地址，而不会通过 entrypoint 去查找。因此 frida 采用的方法是直接修改目标方法的 quickCode 内容，将其替换为一段跳板代码，然后再间接跳转到我们的劫持实现中。

以 ARM64 为例，trampoline 的部分代码如下所示:

```
patchArtMethod(replacementMethodId, {
    jniCode: impl,
    accessFlags: ((originalFlags & ~(kAccCriticalNative | kAccFastNative | kAccNterpEntryPointFastPathFlag)) | kAccNative) >>> 0,
    quickCode: api.artClassLinker.quickGenericJniTrampoline,
    interpreterCode: api.artInterpreterToCompiledCodeBridge
}, vm);
```

quickCode 的替换使用使用 `ArtQuickCodeInterceptor` 完成，姑且理解为跳板管理器，保存在 MethodMangler 的 interceptor 属性中，这样在程序退出或者用户调用 `method.implementation = null` 时还会调用 `interceptor.deactivate` 恢复方法，防止残留 hook 代码导致的运行时崩溃。

本文介绍了 frida 针对 ART 运行时的实现，其基本思路是通过运行时中的私有函数来与对应的 Java 虚拟机进行交互，其中大部分操作都基于 gum-js 接口进行实现，包括 Java 类和方法的封装、ART hook 以及 trampoline 汇编的编写等。除了 ART 还实现了 Dalvik 和 JVM 这两个虚拟机的接口，流程大同小异。甚至对于其他语言来说也是类似的，这也得益于 frida 开源社区的活跃性，相信未来会出现对更多高级语言的支持。

*   [frida-java-bridge](https://github.com/frida/frida-java-bridge)
*   [frida JavaScript API](https://frida.re/docs/javascript-api/)
*   [SandHook - Docs](https://github.com/asLody/SandHook/blob/master/doc/doc.md)
*   [在 Android N 上对 Java 方法做 hook 遇到的坑](http://rk700.github.io/2017/06/30/hook-on-android-n/)
*   [我为 Dexposed 续一秒——论 ART 上运行时 Method AOP 实现](https://weishu.me/2017/11/23/dexposed-on-art/)
*   [ART 在 Android 安全攻防中的应用](https://evilpan.com/2021/12/26/art-internal/)

> **版权声明**: 自由转载 - 非商用 - 非衍生 - 保持署名 ([CC 4.0 BY-SA](https://creativecommons.org/licenses/by-nc/4.0/))  
> **原文地址**: [https://evilpan.com/2022/04/17/frida-java/](https://evilpan.com/2022/04/17/frida-java/)  
> **微信订阅**: **『[有价值炮灰](http://t.evilpan.com/qrcode.jpg)』**  
> – _TO BE CONTINUED_.