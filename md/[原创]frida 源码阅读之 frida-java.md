> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-229215.htm)

frida 源码阅读之 frida-java
======================

frida-java 简介
-------------

frida 的 JavaScript API 按功能划分了许多模块，frida-java 具体实现了其中的 Java 模块，提供了 Java Runtime 相关的 API。我们知道 JNI 连接了 native 世界和 java 世界，而 frida-java 相当于实现了一个 js 世界到 java 世界的单向通道。利用 frida-java，我们可以使用 js 代码实现：调用 java 方法、创建 java 对象、对 java 函数进行 hook 等操作。

 

frida-java 源码结构如下:

*   index.js: 封装了 JavaScript API 中 Java 模块的 api
*   lib
    *   android.js: 封装了一些 Android 虚拟机的 api
    *   api.js
    *   class-factory.js: java 类的一些处理函数
    *   env.js: js 中实现了 JNIEnv 的一个代理
    *   mkdex.js: dex 文件的一些处理函数
    *   result.js: 检测 jni 调用是否产生异常
    *   vm.js: js 中实现了 JavaVM 的一个代理

接下来通过下面两个问题来分析 frida-java 源码：

1.  frida-java 是如何连通到 java 世界的？
2.  frida-java 如何实现 java 方法 hook？

[](#frida-java是如何连通到java世界？)frida-java 是如何连通到 java 世界？
------------------------------------------------------

总的来说 frida-java 通过两步实现 js 世界到 java 世界的单向通道，首先利用 frida-gum 提供的 js 接口操作 native 世界，然后再基于 jni 连通到 java 世界。

 

![](https://bbs.pediy.com/upload/attach/201806/657395_KUK5RJUFFQG7YWN.jpg)

 

主要步骤如下：

1.  连通 native 世界，获得一些虚拟机的接口，如 JNI_GetCreatedJavaVMs
2.  获得 JavaVM
3.  获得 JNIEnv
4.  获得 java 类的 class 引用
5.  操作该 java 类，如创建对象，调用方法等。

### 连通 native 世界

利用 frida-gum 中提供的 Module、Memory、NativeFunction 等模块，可以实现查找、调用、hook 导出函数；读写、分配内存等操作。如下面例子所示, 可以在 js 代码中调用 native 函数。

```
var friendlyFunctionName = new NativeFunction(friendlyFunctionPtr, 'void', ['pointer', 'pointer']);
var returnValue = Memory.alloc(sizeOfLargeObject);
friendlyFunctionName(returnValue, thisPtr);

```

### 获得 JavaVM

想要使用 JNI 连通 java 世界，首先需要获取的就是 JavaVM，frida-java 通过调用 JNI_GetCreatedJavaVMs 来获取 JavaVM，Dalvik 虚拟机和 ART 虚拟机均实现了该函数。核心代码如下:

```
//lib/android.js
const vms = Memory.alloc(pointerSize);
const vmCount = Memory.alloc(jsizeSize);
checkJniResult('JNI_GetCreatedJavaVMs', temporaryApi.JNI_GetCreatedJavaVMs(vms, 1, vmCount));
if (Memory.readInt(vmCount) === 0) {
  return null;
}
temporaryApi.vm = Memory.readPointer(vms);

```

此时获取的 vm 只是 JavaVM 的一个指针，在此基础上 frida-java 还会构造一个 VM 对象，该对象相当于在 js 层实现了一个 JavaVM 的代理对象，封装了一些 JavaVM 的方法，如 getEnv，其中 vm.handle 中保存的是原始的 JavaVM 对象。

```
//lib/vm.js
function VM (api) {
  let handle = null;  //保存JavaVM指针
  let attachCurrentThread = null;  //封装了JavaVM.AttachCurrentThread
  let detachCurrentThread = null;  //封装了JavaVM.DetachCurrentThread;
  let getEnv = null;  //封装了JavaVM.getEnv
  const attachedThreads = {};
  function initialize () {
    handle = api.vm;
    //获取JavaVM的虚函数表
    const vtable = Memory.readPointer(handle);
    attachCurrentThread = new NativeFunction(Memory.readPointer(vtable.add(4 * pointerSize)), 'int32', ['pointer', 'pointer', 'pointer']);
    detachCurrentThread = new NativeFunction(Memory.readPointer(vtable.add(5 * pointerSize)), 'int32', ['pointer']);
    getEnv = new NativeFunction(Memory.readPointer(vtable.add(6 * pointerSize)), 'int32', ['pointer', 'pointer', 'int32']);
  }
}

```

VM 的初始化过程为首先获取 JavaVM 的指针（通过 JNI_GetCreatedJavaVMs 调用），然后读取 JavaVM 的虚函数表，获得 JavaVM 的一些重要方法，并在 js 层包装一层，这样就在 js 层实现了一个 JavaVM 的代理，可以通过调用 VM.getEnv 来实现 native 层的 JavaVM.getEnV 调用。

### 获得 JNIEnv

获取到 JavaVM 后，就可以通过将当前线程与 JavaVM 相关联，然后得到 JNIEnv 对象，进行后续操作。上述工作由 VM.perform 完成，看下 VM.perform 源码：

```
//vm.js
this.tryGetEnv = function () {
    const envBuf = Memory.alloc(pointerSize);
    const result = getEnv(handle, envBuf, JNI_VERSION_1_6);
    if (result !== JNI_OK) {
      return null;
    }
    return new Env(Memory.readPointer(envBuf), this);
  };
this.perform = function (fn) {
    let threadId = null;
    //将当前线程附加到JavaVM，获取JNIEnv对象
    let env = this.tryGetEnv();
    const alreadyAttached = env !== null;
    if (!alreadyAttached) {
      env = this.attachCurrentThread();
 
      threadId = Process.getCurrentThreadId();
      attachedThreads[threadId] = true;
    }
    try {
      fn();  //执行fn
    } finally {
      if (!alreadyAttached) {
        const allowedToDetach = attachedThreads[threadId];
        delete attachedThreads[threadId];
 
        if (allowedToDetach) {
          this.detachCurrentThread();
        }
      }
    }
  };

```

和 JavaVM 一样，frida-java 也会在 js 层为 JNIEnv 建立一个代理，具体在 env.js 实现

### 获得 Java 类的 class 引用

和 JNI 操作方式一样，我们在 native 层获得了 JNIEnv 后，要想操作 java 类，可以通过调用 env->findClass 来获得 java 类的 class 引用。但是这里有个问题，因为 frida-java 所在的线程是通过 pthread_create 创造的，然后通过 AttachCurrentThread 获取的 JNIEnv，此时 FindClass 只会从系统的 classloader 开始查找，所以 app 自身的类是无法通过 env->findClass 来获取。因此需要手工的获取到加载该 app 的 classloader。Java.perform 在调用 VM.perform 之前会先获取加载该 app 的 classloader，并保存到 classFactory.loader。

```
this.perform = function (fn) {
    assertJavaApiIsAvailable();
      //目标进程不是app，并且classloader已经初始化
    if (!isAppProcess() || classFactory.loader !== null) {
      threadsInPerform++;
      try {
        vm.perform(fn);
      } catch (e) {
        setTimeout(() => { throw e; }, 0);
      } finally {
        threadsInPerform--;
      }
    } else {
      //第一次调用java.perform时，会先获取加载该app的classloader
      pending.push(fn);
      if (pending.length === 1) {
        threadsInPerform++;
        try {
          vm.perform(() => {
            const ActivityThread = classFactory.use('android.app.ActivityThread');
            const app = ActivityThread.currentApplication();
            if (app !== null) {
                    //获取到加载该app的classloader
              classFactory.loader = app.getClassLoader();
              performPending(); // already initialized, continue
            } else {
              const m = ActivityThread.getPackageInfoNoCheck;
              let initialized = false;
              m.implementation = function () {
                const apk = m.apply(this, arguments);
                if (!initialized) {
                  initialized = true;
                  classFactory.loader = apk.getClassLoader();
                  performPending();
                }
                return apk;
              };
            }
          });
        } finally {
          threadsInPerform--;
        }
      }
    }
  };

```

frida-java 使用 Java.use 来获得 java 类的 class 引用，Java.use(className), 返回 java 类的一个 wrapper, 在 js 世界里，用该 wrapper 来操作对应的 java 类。Java.use 直接调用了 classFactory.use, 代码如下：

```
//lib/class-factory.js
this.use = function (className) {
    let C = classes[className]; //先从缓存中查找
    if (!C) {
      const env = vm.getEnv();  //获取jni_env, 调用native层的JavaVm.GetEnv  
      if (loader !== null) {  //loader已经在Java.perform中初始化了
        const usedLoader = loader;
 
        if (cachedLoaderMethod === null) {
          cachedLoaderInvoke = env.vaMethod('pointer', ['pointer']);
          cachedLoaderMethod = loader.loadClass.overload('java.lang.String').handle;
        }
 
        const getClassHandle = function (env) {
          const classNameValue = env.newStringUtf(className);
          const tid = Process.getCurrentThreadId();
          ignore(tid);
          try {       //env.handle 指向jni层的JNIEnv, 利用jni调用classloader.loadClass(className)
            return cachedLoaderInvoke(env.handle, usedLoader.$handle, cachedLoaderMethod, classNameValue);
          } finally {
            unignore(tid);
            env.deleteLocalRef(classNameValue);
          }
        };
        //构建对应java类的wrapper
        C = ensureClass(getClassHandle, className);
      }
    }

```

借助于该 wrapper，可以对 java 类进行操作，如调用构造函数创建对象。该 wrapper 的初始化过程如下：

```
//lib/class-factory.js
function initializeClass () {
    klass.__name__ = name;
 
    let ctor = null;
    let getCtor = function (type) {};
    //定义了一些每个类都公有的函数和属性
    Object.defineProperty(klass.prototype, '$new', {});
    Object.defineProperty(klass.prototype, '$alloc', {});
    Object.defineProperty(klass.prototype, '$init', {});
    klass.prototype.$dispose = dispose;
    klass.prototype.$isSameObject = function (obj) {});
    Object.defineProperty(klass.prototype, 'class', {});
    Object.defineProperty(klass.prototype, '$className', {});
    //添加该类特有的函数和属性
    addMethodsAndFields();
}

```

借助于该 wrapper 的 $init 方法，就可以创建 java 对象。

[](#frida-java如何实现java方法调用与java方法hook？)frida-java 如何实现 java 方法调用与 java 方法 hook？
-------------------------------------------------------------------------------

### frida-java Dalvik Hook 原理

frida-java 采用常见的 Dalvik Hook 方案，将待 hook 的 java 函数修改为 native 函数，当调用该函数时，会执行自定义的 native 函数。但是和其他 hook 框架不同的是，使用 frida 时，我们 hook 的代码是 js 实现的，所以有一个基于 js 代码生成 native 函数过程。具体 hook 实现代码如下：

```
function replaceDalvikImplementation (fn) {
  if (fn === null && dalvikOriginalMethod === null) {
    return;
  }
  //保存原Method结构
  if (dalvikOriginalMethod === null) {
    dalvikOriginalMethod = Memory.dup(methodId, DVM_METHOD_SIZE);
    dalvikTargetMethodId = Memory.dup(methodId, DVM_METHOD_SIZE);
  }
  if (fn !== null) {
    implementation = implement(f, fn);  //由js代码生成对应的native函数
 
    let argsSize = argTypes.reduce((acc, t) => (acc + t.size), 0);
    if (type === INSTANCE_METHOD) {
      argsSize++;
    }
    /*
     * make method native (with kAccNative)
     * insSize and registersSize are set to arguments size
     */
    const accessFlags = (Memory.readU32(methodId.add(DVM_METHOD_OFFSET_ACCESS_FLAGS)) | kAccNative) >>> 0;
    const registersSize = argsSize;
    const outsSize = 0;
    const insSize = argsSize;
 
    Memory.writeU32(methodId.add(DVM_METHOD_OFFSET_ACCESS_FLAGS), accessFlags);
    Memory.writeU16(methodId.add(DVM_METHOD_OFFSET_REGISTERS_SIZE), registersSize);
    Memory.writeU16(methodId.add(DVM_METHOD_OFFSET_OUTS_SIZE), outsSize);
    Memory.writeU16(methodId.add(DVM_METHOD_OFFSET_INS_SIZE), insSize);
    Memory.writeU32(methodId.add(DVM_METHOD_OFFSET_JNI_ARG_INFO), computeDalvikJniArgInfo(methodId));
    //利用 dvmUSeJNIBridge完成替换Method结构的insns和NativeFunc
    api.dvmUseJNIBridge(methodId, implementation);
    patchedMethods.add(f);
  } else {
    patchedMethods.delete(f);
 
    Memory.copy(methodId, dalvikOriginalMethod, DVM_METHOD_SIZE);
    implementation = null;
  }
}

```

hook 后执行的 native 函数就是用 implement 函数实现的，该函数时最终调用 frida-node 的 new NativeCallback 接口实现将 js 函数转换为 native 函数。

### frida-java ART Hook 原理

常见的 ART Hook 方法为：替换方法的入口点，即 ArtMethod 的 entry_point_from_quick_compiled_code_，并将原方法的信息备份到 entry_point_from_jni_。替换后的入口点，会重新准备栈和寄存器，执行 hook 的方法。

 

frida-java 采用的 hook 方法，我在其他地方并未遇见过，其原理为：首先将方法 native 化，然后将 ArtMethod 的 entry_point_from_jni_ 替换为 hook 的方法，并将 entry_point_from_quick_compiled_code_ 替换为 art_quick_generic_jni_trampoline。当调用被 hook 的方法时，首先会跳转到 art_quick_generic_jni_trampoline，该函数会做一些 jni 调用的准备，然后跳转到 ArtMethod 结构的 entry_point_from_jni_ 所指向的 hook 方法，这样就完成了一次 hook。完成 art hook 的源码如下：

```
function replaceArtImplementation (fn) {
      if (fn === null && artOriginalMethodInfo === null) {
        return;
      }
      const artMethodSpec = getArtMethodSpec(vm);
      const artMethodOffset = artMethodSpec.offset; //获取ArtMethod各个字段的偏移
      if (artOriginalMethodInfo === null) {
        artOriginalMethodInfo = fetchMethod(methodId);    //保存原方法信息
      }
      if (fn !== null) {
        implementation = implement(f, fn);
 
        // kAccFastNative so that the VM doesn't get suspended while executing JNI
        // (so that we can modify the ArtMethod on the fly)
        patchMethod(methodId, {
           //替换entry_point_from_jni_为hook的方法;
          'jniCode': implementation,
           //native化
          'accessFlags': (Memory.readU32(methodId.add(artMethodOffset.accessFlags)) | kAccNative | kAccFastNative) >>> 0,
           //替换entry_point_from_quick_compiled_code_为art_quick_generic_jni_trampoline;
          'quickCode': api.artQuickGenericJniTrampoline,
           //entry_point_from_interpreter_;
          'interpreterCode': api.artInterpreterToCompiledCodeBridge
        });
 
        patchedMethods.add(f);
      } else {
        patchedMethods.delete(f);
 
        patchMethod(methodId, artOriginalMethodInfo);
        implementation = null;
      }
    }

```

然后看下 art_quick_generic_jni_trampoline 源码，art_quick_generic_jni_trampoline 主要负责 jni 调用的准备，包括堆栈的设置，参数的设置等。个人能力有限只能大概看看流程。

```
ENTRY art_quick_generic_jni_trampoline
    SETUP_REFS_AND_ARGS_CALLEE_SAVE_FRAME_WITH_METHOD_IN_R0
 
    // Save rSELF
    mov r11, rSELF
    // Save SP , so we can have static CFI info. r10 is saved in ref_and_args.
    mov r10, sp
    .cfi_def_cfa_register r10
 
    sub sp, sp, #5120
 
    // prepare for artQuickGenericJniTrampoline call, 转为为c调用约定
    // (Thread*,  SP)
    //    r0      r1   <= C calling convention
    //  rSELF     r10  <= where they are
 
    mov r0, rSELF   // Thread*
    mov r1, r10     // ArtMethod**
    //调用c的 artQuickGenericJniTrampoline
    blx artQuickGenericJniTrampoline  // (Thread*, sp)
 
    // The C call will have registered the complete save-frame on success.
    // The result of the call is:
    // r0: pointer to native code, 0 on error.
    // r1: pointer to the bottom of the used area of the alloca, can restore stack till there.
 
    // Check for error = 0.
    cbz r0, .Lexception_in_native
 
    // Release part of the alloca.
    mov sp, r1
 
    // Save the code pointer
    mov r12, r0
 
    // Load parameters from frame into registers.
    pop {r0-r3}

```

主要是将调用约定转换为 c 的调用约定，将参数准备好，然后调用 artQuickGenericJniTrampoline, 该函数源码如下：

```
extern "C" TwoWordReturn artQuickGenericJniTrampoline(Thread* self, ArtMethod** sp)
    SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
  ArtMethod* called = *sp;
  DCHECK(called->IsNative()) << PrettyMethod(called, true);
  uint32_t shorty_len = 0;
  const char* shorty = called->GetShorty(&shorty_len);
 
  // Run the visitor and update sp.
  BuildGenericJniFrameVisitor visitor(self, called->IsStatic(), shorty, shorty_len, &sp);
  visitor.VisitArguments();
  visitor.FinalizeHandleScope(self);
 
  // Fix up managed-stack things in Thread.
  self->SetTopOfStack(sp);
 
  self->VerifyStack();
 
  // Start JNI, save the cookie.
  uint32_t cookie;
  if (called->IsSynchronized()) {
    cookie = JniMethodStartSynchronized(visitor.GetFirstHandleScopeJObject(), self);
    if (self->IsExceptionPending()) {
      self->PopHandleScope();
      // A negative value denotes an error.
      return GetTwoWordFailureValue();
    }
  } else {
    cookie = JniMethodStart(self);
  }
  uint32_t* sp32 = reinterpret_cast(sp);
  *(sp32 - 1) = cookie;
 
  // Retrieve the stored native code.
  //获取到ArtMethod结构的entry_point_from_jni_
  void* nativeCode = called->GetEntryPointFromJni();
 
  // There are two cases for the content of nativeCode:
  // 1) Pointer to the native function.
  // 2) Pointer to the trampoline for native code binding.
  // In the second case, we need to execute the binding and continue with the actual native function
  // pointer.
  DCHECK(nativeCode != nullptr);
  if (nativeCode == GetJniDlsymLookupStub()) {
#if defined(__arm__) || defined(__aarch64__)
    nativeCode = artFindNativeMethod();
#else
    nativeCode = artFindNativeMethod(self);
#endif
    if (nativeCode == nullptr) {
      DCHECK(self->IsExceptionPending());    // There should be an exception pending now.
      DCHECK(self->IsExceptionPending());    // There should be an exception pending now.
 
      // End JNI, as the assembly will move to deliver the exception.
      jobject lock = called->IsSynchronized() ? visitor.GetFirstHandleScopeJObject() : nullptr;
      if (shorty[0] == 'L') {
        artQuickGenericJniEndJNIRef(self, cookie, nullptr, lock);
      } else {
        artQuickGenericJniEndJNINonRef(self, cookie, lock);
      }
 
      return GetTwoWordFailureValue();
    }
    // Note that the native code pointer will be automatically set by artFindNativeMethod().
  }
 
  // Return native code addr(lo) and bottom of alloca address(hi).
  return GetTwoWordSuccessValue(reinterpret_cast(visitor.GetBottomOfUsedArea()),
                                reinterpret_cast(nativeCode));
} 
```

总结
--

笔记分析的还是很粗糙的，发出来是希望可以抛砖引玉，可以在论坛看到更多 frida 的文章，如有错误，恳请大佬指出。

[安卓应用层抓包通杀脚本发布！《高研班》2021 年 3 月班开始招生！](https://bbs.pediy.com/thread-264283.htm)