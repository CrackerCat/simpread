> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291155.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

根据[官方文档](https://bbs.kanxue.com/elink@b69K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6X3M7X3W2V1j5g2)9J5k6i4u0W2i4K6u0r3k6r3!0U0M7#2)9J5c8X3u0J5K9h3c8Y4k6i4y4Q4x3V1k6Q4x3@1k6#2N6r3#2Q4y4h3k6K6L8%4g2J5j5$3g2Q4x3@1c8U0K9r3q4@1k6%4m8@1i4K6u0W2j5$3!0E0)，Frida 17 之后的版本中， GumJs 运行时不再捆绑 bridges（例如 frida-java-bridge、frida-objc-bridge, frida-swift-bridge）。因此这篇 Java Hook 原理分析参考的项目源码在 [frida-java-bridge](https://bbs.kanxue.com/elink@7a2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6X3M7X3W2V1j5g2)9J5c8X3k6J5K9h3c8S2i4K6u0V1K9X3q4$3j5g2)9J5k6r3u0J5K9h3c8Y4k6b7%60.%60.) 中。

以如下例子进行原理分析

```
var Adapter = Java.use(targetClass);
Adapter["doAdapter"].implementation = function (i) {
    console.log(">>> doAdapter is called: i=" + i);
    var result = this.doAdapter(i);
    console.log("<<< doAdapter result=" + result);
    return result;
}


```

Adapter["doAdapter"] 获取到的是 methodPrototype 对象，然后将自定义的 hook 函数赋值给 implementation 字段，使用到的是 implementation 的`set()`方法来安装 Hook（对应 android.js 1697 行处）。

```
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
}


```

`set()` 首先校验目标方法是否是构造方法，此方法不通过当前路径实现。然后检查目标方法是否已经被 hook，如果被 hook 过了，则撤销之前的 hook 逻辑。最后，如果存在新的 hook 逻辑，则调用`implement()`方法将用户定义的 js hook 逻辑封装成 native 函数，并调用`mangler.replace()`方法对目标方法进行 hook。

```
function implement (methodName, classWrapper, type, retType, argTypes, handler, fallback = null) {
  const pendingCalls = new Set();
  
  const f = makeMethodImplementation([methodName, classWrapper, type, retType, argTypes, handler, fallback, pendingCalls]);
  
  const impl = new NativeCallback(f, retType.type, ['pointer', 'pointer'].concat(argTypes.map(t => t.type)));
  impl._c = pendingCalls;

  return impl;
}

function makeMethodImplementation (params) {
  return function () {
    return handleMethodInvocation(arguments, params);
  };
}


```

该函数主要是对用户定义的 hook 逻辑封装成 native 函数。其中`makeMethodImplementation()` 的作用是把一堆上下文参数封进闭包，生成一个符合 `NativeCallback` 签名的入口函数，其内部调用了`handleMethodInvocation()`方法。

```
function handleMethodInvocation (jniArgs, params) {
  
  const env = new Env(jniArgs[0], vm);

  const [methodName, classWrapper, type, retType, argTypes, handler, fallback, pendingCalls] = params;

  const ownedObjects = [];
  
  let self;
  if (type === INSTANCE_METHOD) {
    const C = classWrapper.$C;
    self = new C(jniArgs[1], STRATEGY_VIRTUAL, env, false);
  } else {
    self = classWrapper;
  }

  const tid = getCurrentThreadId();
  
  env.pushLocalFrame(3);
  let haveFrame = true;
  
  vm.link(tid, env);

  try {
    pendingCalls.add(tid);
    
    let fn;
    if (fallback === null || !ignoredThreads.has(tid)) {
      fn = handler;
    } else {
      fn = fallback;
    }
    
    const args = [];
    const numArgs = jniArgs.length - 2; 
    for (let i = 0; i !== numArgs; i++) {
      const t = argTypes[i];

      const value = t.fromJni(jniArgs[2 + i], env, false);
      args.push(value);
      
      ownedObjects.push(value);
    }
    
    const retval = fn.apply(self, args);

    if (!retType.isCompatible(retval)) {
      throw new Error(`Implementation for ${methodName} expected return value compatible with ${retType.className}`);
    }
    
    let jniRetval = retType.toJni(retval, env);
    
    
    if (retType.type === 'pointer') {
      jniRetval = env.popLocalFrame(jniRetval);
      haveFrame = false;

      ownedObjects.push(retval);
    }

    return jniRetval;
  }
  ...
}


```

当 Java 调用某个被 Hook 的方法时，负责将 JNI 的数据结构转换成 js 对象，执行用户定义的 js hook 代码，然后再将结果转换成回 JNI 的变量类型。

这里分析 andriod 平台下的 java hook，因此我们分析`lib\android.js`下的`ArtMethodMangler`，该类的初始化函数如下：

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
  ...
}


```

对应`ArtMethodMangler.replace()`方法，位置在`lib\android.js`3707 行处。接下来进行拆解分析。

```
replace (impl, isInstanceMethod, argTypes, vm, api) {
    const { kAccCompileDontBother, artNterpEntryPoint } = api;
    
    this.originalMethod = fetchArtMethod(this.methodId, vm);


```

这里主要是获取原函数的 ArtMethod 结构体中部分字段的值，具体为`jniCode`（）、`accessFlags`（）、`quickCode`（）、`interpreterCode`（方法解释执行入口），对于 f`etchArtMethod`方法的详细分析见 [fetchArtMethod](#msg_header_h3_1)。

```
const originalFlags = this.originalMethod.accessFlags;

if ((originalFlags & kAccXposedHookedMethod) !== 0 && xposedIsSupported()) {
    
    
    const hookInfo = this.originalMethod.jniCode;
    
    this.hookedMethodId = hookInfo.add(2 * pointerSize).readPointer();
    
    this.originalMethod = fetchArtMethod(this.hookedMethodId, vm);
}


```

这部分主要检测是否进行了 xposed hook，如果进行了，则借助 xposed hook 的信息获取原方法的 ArtMethod 指针，并重新调用`fetchArtMethod`获取原函数的 ArtMethod 结构体中部分字段的值。

```
const { hookedMethodId } = this; 

const replacementMethodId = cloneArtMethod(hookedMethodId, vm);
this.replacementMethodId = replacementMethodId;

patchArtMethod(replacementMethodId, {
    jniCode: impl, 
    accessFlags: ((originalFlags & ~(kAccCriticalNative | kAccFastNative | kAccNterpEntryPointFastPathFlag)) | kAccNative | kAccCompileDontBother) >>> 0,
    quickCode: api.artClassLinker.quickGenericJniTrampoline,
    interpreterCode: api.artInterpreterToCompiledCodeBridge
}, vm);


```

调用`cloneArtMethod()`复制原函数的 ArtMethod，后续的修改都在这个副本上进行的。之后调用`patchArtMethod()`方法对复制出来的 ArtMethod 中的`jniCode`、`accessFlags`、`quickCode`、`interpreterCode`进行修复，具体见 [patchArtMethod](#msg_header_h3_2)。这里分析一下入参：

*   accessFlags：
    
    *   清掉 `kAccCriticalNative`，避免走 critical native 调用路径。
    *   清掉 `kAccFastNative`，避免走 fast native 调用路径。
    *   清掉 `kAccNterpEntryPointFastPathFlag`，避免 nterp（ART 使用的新一代解释器，逐步取代了早期的 Mterp）快路径绕过 Frida 预期的调用路径。
    *   加上 `kAccNative`，告诉 ART 这个 method 是 native 方法。
    *   加上 `kAccCompileDontBother`，告诉 ART 编译器不要尝试编译这个方法，因为 native 方法已经是机器码，不需要 ART JIT/AOT 编译。
    
    **总结一下就是 把方法标记为普通的 native 方法，同时禁用 ART 的特殊快速路径优化。**
    
*   quickCode
    
    原本是 quick 模式入口，现修改成`api.artClassLinker.quickGenericJniTrampoline`，代表的是`ClassLinker`的`quick_generic_jni_trampoline_`字段。也就是说，从 quick 路径进入这个方法时，不直接执行原来的 quick compiled code，而是进入 ART 的通用 JNI 调用桥，再由 JNI 桥读取 `jniCode` 并调用 native 化的 hook 代码。
    
*   interpreterCode
    
    原本是解释器模式入口，现修改成`api.artInterpreterToCompiledCodeBridge`，是从解释器模式转成机器码模式的入口。这样解释器执行该方法时，会桥接到 compiled/JNI 路径，最终进入 generic JNI trampoline 和 native 化的 hook 代码。
    

```
let hookedMethodRemovedFlags = kAccFastInterpreterToInterpreterInvoke | kAccSingleImplementation | kAccNterpEntryPointFastPathFlag;
if ((originalFlags & kAccNative) === 0) {
    hookedMethodRemovedFlags |= kAccSkipAccessChecks;
}

patchArtMethod(hookedMethodId, {
    accessFlags: ((originalFlags & ~(hookedMethodRemovedFlags)) | kAccCompileDontBother) >>> 0
}, vm);


```

这里对原方法的`accessFlags`进行了处理，禁用了快速调用路径标志，如果原方法不是 native，还需要移除访问检查跳过标志，最后加上`kAccCompileDontBother`，告诉 ART 不要尝试 JIT/AOT 编译这个方法。

```
const quickCode = this.originalMethod.quickCode;



if (artNterpEntryPoint !== null && quickCode.equals(artNterpEntryPoint)) {
    patchArtMethod(hookedMethodId, {
        quickCode: api.artQuickToInterpreterBridge
    }, vm);
}


```

如果原方法是解释执行，且使用了 Nterp，那么`quickCode`就会替换成`art_quick_to_interpreter_bridge`，强制跳出解释器模式，并改用机器码模式。

```
if (!isArtQuickEntrypoint(quickCode)) {
    const interceptor = new ArtQuickCodeInterceptor(quickCode);
    
    interceptor.activate(vm);

    this.interceptor = interceptor;
}


```

如果原方法的 quickCode 是 ART Quick 编译路径的入口，也就是说原方法已经是机器码模式了，那么就需要通过`ArtQuickCodeInterceptor`对机器码进行 patch，这部分见 [ArtQuickCodeInterceptor.activate](#msg_header_h2_4)。

最后是收尾工作

```
artController.replacedMethods.set(hookedMethodId, replacementMethodId);
notifyArtMethodHooked(hookedMethodId, vm);


```

```
function fetchArtMethod (methodId, vm) {
  const artMethodSpec = getArtMethodSpec(vm);
  const artMethodOffset = artMethodSpec.offset;
  return (['jniCode', 'accessFlags', 'quickCode', 'interpreterCode']
    .reduce((original, name) => {
      const offset = artMethodOffset[name];
      if (offset === undefined) {
        return original;
      }
      const address = methodId.add(offset);
      const read = (name === 'accessFlags') ? readU32 : readPointer;
      original[name] = read.call(address);
      return original;
    }, {}));
}


```

`getArtMethodSpec`返回的是`ArtMethod`结构体布局描述，返回的结构是

```
{
  size: <ArtMethod结构体大小>,
  offset: {
    jniCode: <偏移，对应字段为JNI函数指针(native方法入口)>,
    quickCode: <偏移，对应字段为指向JIT/AOT编译后的机器码的入口>,
    accessFlags: <偏移，对应字段为方法的修饰信息>,
    interpreterCode: <偏移，对应字段为方法解释执行的入口>
  }
}


```

由于 Android 版本的差异，有些字段就不存在（例如 interpreterCode），于是借助`reduce`过滤出存在的字段并获取对应字段的值。

```
function patchArtMethod (methodId, patches, vm) {
  const artMethodSpec = getArtMethodSpec(vm);
  const artMethodOffset = artMethodSpec.offset;
  Object.keys(patches).forEach(name => {
    const offset = artMethodOffset[name];
    if (offset === undefined) {
      return;
    }
    const address = methodId.add(offset);
    const write = (name === 'accessFlags') ? writeU32 : writePointer;
    write.call(address, patches[name]);
  });
}


```

获取`jniCode`、`accessFlags`、`quickCode`、`interpreterCode`字段，并进行修正，设置为相应的入参值。

```
activate (vm) {
    const constraints = this._allocateTrampoline();

    const { trampoline, quickCode, redirectSize } = this;
    
    const writeTrampoline = artQuickCodeReplacementTrampolineWriters[Process.arch];
    const prologueLength = writeTrampoline(trampoline, quickCode, redirectSize, constraints, vm);
    this.overwrittenPrologueLength = prologueLength;

    this.overwrittenPrologue = Memory.dup(this.quickCodeAddress, prologueLength);
    
    const writePrologue = artQuickCodePrologueWriters[Process.arch];
    writePrologue(quickCode, trampoline, redirectSize);
}


```

该函数首先调用`_allocateTrampoline()`方法分片 trampoline 的内存空间、确定用于重定向的字节大小，以及获取空闲寄存器。以 Arm64 为例，接下来调用`writeArtQuickCodeReplacementTrampolineArm64()`函数编写 trampoline，返回用于重定向的字节大小，然后保存原函数 quickCode 入口处相应字节大小的指令，用于后续恢复。最后调用`writeArtQuickCodePrologueArm64()`函数修改 quickCode 入口使其跳转到编写好的 trampoline 处。

```
function writeArtQuickCodeReplacementTrampolineArm64 (trampoline, target, redirectSize, { availableScratchRegs }, vm) {
  const artMethodOffsets = getArtMethodSpec(vm).offset;

  let offset;
  Memory.patchCode(trampoline, 256, code => {
    const writer = new Arm64Writer(code, { pc: trampoline });
    const relocator = new Arm64Relocator(target, writer);
    
    
    writer.putPushRegReg('d0', 'd1');
    writer.putPushRegReg('d2', 'd3');
    writer.putPushRegReg('d4', 'd5');
    writer.putPushRegReg('d6', 'd7');

    
    writer.putPushRegReg('x1', 'x2');
    writer.putPushRegReg('x3', 'x4');
    writer.putPushRegReg('x5', 'x6');
    writer.putPushRegReg('x7', 'x20');
    writer.putPushRegReg('x21', 'x22');
    writer.putPushRegReg('x23', 'x24');
    writer.putPushRegReg('x25', 'x26');
    writer.putPushRegReg('x27', 'x28');
    writer.putPushRegReg('x29', 'lr');

    
    writer.putSubRegRegImm('sp', 'sp', 16); 
    writer.putStrRegRegOffset('x0', 'sp', 0);
    
    writer.putCallAddressWithArguments(artController.replacedMethods.findReplacementFromQuickCode, ['x0', 'x19']);

    writer.putCmpRegReg('x0', 'xzr');
    writer.putBCondLabel('eq', 'restore_registers');
    
    
    writer.putStrRegRegOffset('x0', 'sp', 0);
    
    writer.putLabel('restore_registers');
    
    
    writer.putLdrRegRegOffset('x0', 'sp', 0);
    writer.putAddRegRegImm('sp', 'sp', 16);

    
    writer.putPopRegReg('x29', 'lr');
    writer.putPopRegReg('x27', 'x28');
    writer.putPopRegReg('x25', 'x26');
    writer.putPopRegReg('x23', 'x24');
    writer.putPopRegReg('x21', 'x22');
    writer.putPopRegReg('x7', 'x20');
    writer.putPopRegReg('x5', 'x6');
    writer.putPopRegReg('x3', 'x4');
    writer.putPopRegReg('x1', 'x2');

    
    writer.putPopRegReg('d6', 'd7');
    writer.putPopRegReg('d4', 'd5');
    writer.putPopRegReg('d2', 'd3');
    writer.putPopRegReg('d0', 'd1');

    writer.putBCondLabel('ne', 'invoke_replacement');
    
    do {
      offset = relocator.readOne();
    } while (offset < redirectSize && !relocator.eoi);
    
    relocator.writeAll();
    
    if (!relocator.eoi) {
      const scratchReg = Array.from(availableScratchRegs)[0];
      writer.putLdrRegAddress(scratchReg, target.add(offset));
      writer.putBrReg(scratchReg);
    }
    
    writer.putLabel('invoke_replacement');

    writer.putLdrRegRegOffset('x16', 'x0', artMethodOffsets.quickCode);
    writer.putBrReg('x16');

    writer.flush();
  });

  return offset;
}


```

首先保存部分寄存器，然后调用`findReplacementFromQuickCode`方法查找是否存在 replacement 实现，如果有，则返回相应地址，否则为 NULL（0），这里分情况进行讨论分析：

*   没有 replacement 实现，即 `x0 == 0`，跳到 `restore_registers`标签，从栈中恢复部分寄存器，此时`x0`还是原函数的 methodId，接着将原始指令重写到 trampoline 中，执行完这些指令后，最后跳转到原 quick code 剩下的部分继续执行。
*   有 replacement 实现，即 `x0 != 0`，把返回的 replacement methodId 写回栈顶，然后恢复部分寄存器，此时`x0`就是 replacement methodId。接着跳转到`invoke_replacement`标签处，跳转到 replacement method 的 quickCode 入口进行执行。

#### findReplacementFromQuickCode

位置在`lib\android.js` 1925 行处。

```
gpointer
find_replacement_method_from_quick_code (gpointer method, gpointer thread)
{
  gpointer replacement_method;
  gpointer managed_stack;
  gpointer top_quick_frame;
  gpointer link_managed_stack;
  gpointer * link_top_quick_frame;

  replacement_method = get_replacement_method (method);
  if (replacement_method == NULL)
    return NULL;

  
  managed_stack = thread + ${threadOffsets.managedStack};
  top_quick_frame = *((gpointer *) (managed_stack + ${managedStackOffsets.topQuickFrame}));
  if (top_quick_frame != NULL)
    return replacement_method;

  link_managed_stack = *((gpointer *) (managed_stack + ${managedStackOffsets.link}));
  if (link_managed_stack == NULL)
    return replacement_method;

  link_top_quick_frame = GSIZE_TO_POINTER (*((gsize *) (link_managed_stack + ${managedStackOffsets.topQuickFrame})) & ~((gsize) 1));
  if (link_top_quick_frame == NULL || *link_top_quick_frame != replacement_method)
    return replacement_method;

  return NULL;
}

gpointer
get_replacement_method (gpointer original_method)
{
  gpointer replacement_method;

  g_mutex_lock (&lock);

  replacement_method = g_hash_table_lookup (methods, original_method);

  g_mutex_unlock (&lock);

  return replacement_method;
}


```

主要是根据传入的原函数 methodId 查找对应的 replacement methodId，返回 NULL 表示应调用原始方法，否则返回指向 replacement ArtMethod 的指针。

这里做了栈检查，这样来自 hook 逻辑中的原函数调用就不会是无限递归，而是返回 NULL，执行原函数逻辑。这就是我们在 hook A 函数，并能够在 hook 逻辑内部调用原始 A 函数的原因。

```
function writeArtQuickCodePrologueArm64 (target, trampoline, redirectSize) {
  Memory.patchCode(target, 16, code => {
    const writer = new Arm64Writer(code, { pc: target });

    if (redirectSize === 16) {
      writer.putLdrRegAddress('x16', trampoline);
    } else {
      writer.putAdrpRegAddress('x16', trampoline);
    }

    writer.putBrReg('x16');

    writer.flush();
  });
}


```

这里则是根据可用重定位空间为 8 字节或 16 字节，设置不同的跳转指令。

Frida 在进行 Java Hook 时，首先会将用户编写的 JS 函数包装成 `NativeCallback`，使其具备 JNI native 函数的调用形式。之后的 hook 工作都围绕 `ArtMethod` 做修改。它会读取目标方法的 `jniCode`、`accessFlags`、`quickCode`、`interpreterCode` 字段，复制出一个 `ArtMethod`副本，用于 replacement method，并**把这个副本改造成 native 方法**：`jniCode` 指向前面生成的 `NativeCallback`，`quickCode` 指向 ART 的通用 JNI trampoline，`interpreterCode` 指向解释器到编译代码的桥接入口。这样无论方法从解释器路径还是 quick compiled code 路径进入，最终都能转到 Frida 的 replacement 实现。需要额外注意的是，**对于已经编译成机器码的方法，Frida 还会 patch 原始机器码入口**，即修改原始机器码前 8/16 字节为跳转到 trampoline 的指令，trampoline 中再通过 `findReplacementFromQuickCode()` 查询当前方法是否存在 replacement。如果存在，就跳转到 replacement method 入口；如果不存在，或者当前调用来自 Hook 逻辑内部对原函数的调用，则恢复执行原方法。

参考：

https://deepwiki.com/frida/frida-java-bridge/4.1-method-hooking-basics

[[原创] 源码简析之 ArtMethod 结构与涉及技术介绍 - Android 安全 - 看雪安全社区｜专业技术交流与安全研究论坛](https://bbs.kanxue.com/thread-248898-1.htm)

[Frida Internal - Part 3: Java Bridge 与 ART hook - 有价值炮灰](https://bbs.kanxue.com/elink@1bfK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6W2N6X3W2D9M7r3q4F1i4K6u0W2j5$3!0E0i4K6u0r3x3U0l9J5x3W2)9J5c8U0l9@1i4K6u0r3x3e0N6Q4x3V1k6X3M7X3W2V1j5g2)9J5k6r3A6S2N6X3q4Q4x3V1j5%60.)

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 28 分钟前 被 gal2xy 编辑 ，原因：