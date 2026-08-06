> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-292312.htm#msg_header_h3_7)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

**ART 运行时深层结构完全解析**  
**——从 ArtMethod、ClassLinker 到 Invoke 全链路，兼谈 FART 在 Android 12–16 的失效与修复**

如果你已经拿 FART 脱过壳、或者自己写过主动调用，那这篇应该能直接对上。  
前面不废话， 背景尽量压短，后面把 `Invoke` 分流、Nterp、dump 点选择、高版本失效和现在怎么处理写细一点，代码也会多放一些。

先把认为最重要的东西放在这里 以下  
`ArtMethod` 只是入口。真正决定主动调用能不能脱到东西的，是这四个问题同时成立：

1.  当前方法的 Entry Point 是什么
2.  `ArtMethod::Invoke` 会不会把你带进能看到 CodeItem 的路径
3.  壳的恢复窗口和你的观察窗口有没有重合
4.  dump 到的东西最后能不能修回去

FART 早期能打，是因为这四个问题当时相对好对齐。Android 12 之后开始大面积失效，不是单纯工具老了，而是执行路径、结构布局、存储限制、Profile/AOT、壳对抗一起变了。

`ArtMethod` 里真正关键的是三块：

身份：class、method index、access flag  
代码位置：早期靠 `dex_code_item_offset_`，后面更多走 `DexFile` / `CodeItemDataAccessor`  
执行入口：`entry_point_from_quick_compiled_code_`

调用时真正跳出去的，通常是最后一个。

所以会有这种误判：

ArtMethod 找到了，方法也调用了，CodeItem 却还是空的。  
不一定是 dump 代码写错，更可能是你看的入口和壳恢复时用的不是同一条路。

`ClassLinker` 负责 load/link/init。很多抽取壳不是 loadClass 后立刻恢复，而是拖到第一次真正执行前。  
`DexCache` 则让 “文件镜像里的 Dex” 和“运行时真正用到的解析结果”可能不一致。所以只 dump 整体 Dex 不够，还得单独拿 CodeItem。

执行后端现在也不单一：

<table><thead><tr><th>后端</th><th>含义</th><th>直接后果</th></tr></thead><tbody><tr><td>AOT</td><td>提前编译</td><td>可能根本不进解释器</td></tr><tr><td>JIT</td><td>运行时编译</td><td>入口会中途切换</td></tr><tr><td>Nterp</td><td>快速解释</td><td>仍解释，但不保证经旧 <code>Execute</code></td></tr><tr><td>Switch Interpreter</td><td>传统解释</td><td>更接近 FART 老点</td></tr><tr><td>JNI trampoline</td><td>native</td><td>另一套逻辑</td></tr></tbody></table>

还按 “禁用 dex2oat 后全部长期解释” 来设计方案，在高版本里前提已经不稳。

主动调用前，先看：

```
ArtMethod::GetEntryPointFromQuickCompiledCode()


```

对应到代码里，大致就是：

```
const void* ArtMethod::GetEntryPointFromQuickCompiledCode() const {
  return GetEntryPointFromQuickCompiledCodePtrSize(kRuntimePointerSize);
}





```

这个指针可能是：

AOT/JIT 机器码  
`art_quick_to_interpreter_bridge`  
Nterp 入口  
Generic JNI stub  
Resolution trampoline  
Instrumentation stub

它不是静态常量。`ClassLinker::LinkCode`、`Instrumentation::InitializeMethodsCode`、JIT 完成、deoptimize，都可能改它。

再叠加这些 flag：

```
bool ArtMethod::HasNterpEntryPointFastPathFlag() const {
  constexpr uint32_t mask = kAccNative | kAccNterpEntryPointFastPathFlag;
  return (GetAccessFlags() & mask) == kAccNterpEntryPointFastPathFlag;
}

void ArtMethod::SetNterpInvokeFastPathFlag() {
  AddAccessFlags(kAccNterpInvokeFastPathFlag);
}


```

只打印 `ArtMethod*`、不看 entry 和 flag，后面很容易盲排。

一个很具体的例子：

在 Android 14 Pixel 7 上，有个被加固的 `onCreate`，当时打印出来的 entry 是 `0x70b1c8a000`，反解后落到 `art_quick_to_interpreter_bridge`。按 FART 老路子主动调用，CodeItem 始终没恢复。  
第一反应是 dump 时机不对，或者 `self == nullptr` 那个标记没进分支。后来开了 `--force-interpreter` 再调，entry 切到了 Nterp，以为这次稳了，结果 `Execute` 还是没进——Nterp 不走老 `Execute`。  
最后把 dump 点从 `Execute` 往上提到 `EnterInterpreterFromInvoke`，才拿到完整 CodeItem。

排查时建议直接把关键信息打全：

```
void DumpMethodState(ArtMethod* m) {
  LOG(INFO) << "method = " << m->PrettyMethod();
  LOG(INFO) << "ArtMethod* = " << m;
  LOG(INFO) << "access_flags = 0x" << std::hex << m->GetAccessFlags();
  LOG(INFO) << "entry = " << m->GetEntryPointFromQuickCompiledCode();
  LOG(INFO) << "is_native = " << m->IsNative();
  LOG(INFO) << "is_static = " << m->IsStatic();
}


```

实用判断：

entry 已是编译代码，主动调用大概率碰不到解释器恢复逻辑  
entry 是 bridge，还有空间，但别默认后面一定进老 `Execute`  
entry 变成 Nterp 后，老 `Execute` hook 可能直接失效  
调用前后 CodeItem 完全不变，优先怀疑路径，不先改文件名和输出目录

#### 主干分流

结合 AOSP 常见实现，主干可以写成：

```
void ArtMethod::Invoke(Thread* self,
                       uint32_t* args,
                       uint32_t args_size,
                       JValue* result,
                       const char* shorty) {
  if (UNLIKELY(__builtin_frame_address(0) < self->GetStackEnd())) {
    ThrowStackOverflowError(self);
    return;
  }

  ManagedStack fragment;
  self->PushManagedStackFragment(&fragment);
  Runtime* runtime = Runtime::Current();

  
  
  
  
  

  if (UNLIKELY(!runtime->IsStarted() ||
               (self->IsForceInterpreter() &&
                !IsNative() &&
                !IsProxyMethod() &&
                IsInvokable()) ||
               )) {
    if (IsStatic()) {
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, nullptr, args, result, true);
    } else {
      mirror::Object* receiver =
          reinterpret_cast<StackReference<mirror::Object>*>(&args[0])->AsMirrorPtr();
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, receiver, args + 1, result, true);
    }
  } else {
    bool have_quick_code = GetEntryPointFromQuickCompiledCode() != nullptr;
    if (LIKELY(have_quick_code)) {
      if (!IsStatic()) {
        (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
      } else {
        (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
      }
    }
  }
}


```

主要来说有三

1.  会不会强制解释
2.  有没有 quick entry
3.  quick entry 是真机器码，还是 bridge / nterp / stub

FART 早期靠这个标记：

```
extern "C" void myfartInvoke(ArtMethod* artmethod)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  JValue* result = nullptr;
  Thread* self = nullptr;   
  uint32_t temp = 6;
  uint32_t* args = &temp;
  uint32_t args_size = 6;
  artmethod->Invoke(self, args, args_size, result, "fart");
}


```

早期好用，是因为大多数被抽空方法最终还会掉进可观察的解释路径，Invoke 又足够靠前。  
高版本继续只靠这个标记会越来越飘：调用表面上成功，实际执行已经从 quick entry 出去了。

#### EnterInterpreterFromInvoke

进解释器后，更关键的一层其实是这里，不是老的 `Execute` 单点：

```
void EnterInterpreterFromInvoke(Thread* self,
                                ArtMethod* method,
                                ObjPtr<mirror::Object> receiver,
                                uint32_t* args,
                                JValue* result,
                                bool stay_in_interpreter) {
  
  
  
  
  
}


```

也就是说我们已经知道了

当前 `ArtMethod*`  
方法是否静态  
参数区  
后续到底会进传统解释，还是被更快的解释后端接走

前面 Pixel 7 那个案子，真正转机不是又找了一个更花哨的 hook，而是 dump 点从 `Execute` 上移到了这一层。

一个可落的插法：

```
void EnterInterpreterFromInvoke(...) {
  
  if () {
    dumpArtMethod(method);
  }

  
}


```

这比死钉 `Execute` 更能兜住 “已经进解释体系，但不走老 Execute” 的情况。

#### Execute

传统 FART 会盯 `Execute`，原因很多时候：

干扰 dex2oat 后，不少方法仍解释执行  
`<clinit>` 相对稳定走解释  
这里能同时碰到 `ArtMethod` 和当时的 `CodeItem`

老插法通常是：

```
static inline JValue Execute(Thread* self,
                             const DexFile::CodeItem* code_item,
                             ShadowFrame& shadow_frame,
                             JValue result_register,
                             bool stay_in_interpreter = false) {
  ArtMethod* method = shadow_frame.GetMethod();

  if (strstr(method->PrettyMethod().c_str(), "<clinit>") != nullptr) {
    dumpDexFileByExecute(method);
  }

  
  
}


```

整体 Dex dump 也可以放在这附近：

```
void dumpDexFileByExecute(ArtMethod* artmethod) {
  const DexFile* dex_file = artmethod->GetDexFile();
  const uint8_t* begin = dex_file->Begin();
  size_t size = dex_file->Size();

  
  
}


```

但 Nterp 把 “解释执行 = 进旧 Execute” 这个等式打断了。

#### Nterp

Nterp 仍然解释字节码，入口却可以直接挂在 quick entry 上，调用约定更接近编译代码。  
结果就是：

```
force interpreter
  → entry 变成 Nterp
  → 老 Execute hook 不触发
  → CodeItem 看起来像没恢复


```

其实不一定是没恢复，更可能是你观察点没盖住这条路。

所以现在更稳的判断不是 “有没有进 Execute”，而是：

1.  entry 现在是什么
2.  有没有进入 `EnterInterpreterFromInvoke`
3.  进入之后 CodeItem 有没有变

#### quick stub

Invoke 一旦认为有 quick code，就进：

```
art_quick_invoke_stub / art_quick_invoke_static_stub
  → entry_point_from_quick_compiled_code_


```

对应关系可以粗写成：

```
const void* code = method->GetEntryPointFromQuickCompiledCode();



```

但：

`have_quick_code == true` 不等于 “已经是最终机器码”。  
后面可能是 bridge，也可能是 nterp。看到 quick entry 就当 AOT，会直接排错方向。

主动调用如果只是 “调了一下”，却没有把 CodeItem 正确摘出来，后面修复照样废。

简化版 dump 逻辑：

```
void dumpArtMethod(ArtMethod* artmethod) {
  const DexFile* dex_file = artmethod->GetDexFile();

  
  const uint8_t* begin = dex_file->Begin();
  size_t dex_size = dex_file->Size();
  

  
  const dex::CodeItem* code_item = artmethod->GetCodeItem();
  if (code_item == nullptr) {
    LOG(INFO) << "CodeItem is null: " << artmethod->PrettyMethod();
    return;
  }

  
  
  uint32_t code_item_len = ComputeCodeItemSize(code_item);

  uint32_t method_idx = artmethod->GetDexMethodIndex();
  
}


```

长度计算是脏活，也是修失败的高发区。思路大致是：

```
CodeItem
  ├─ registers_size / ins_size / outs_size / tries_size
  ├─ insns[]
  ├─ try_item[]          // 如果 tries_size > 0
  └─ encoded_catch_handler_list


```

伪代码：

```
uint32_t ComputeCodeItemSize(const dex::CodeItem* code_item) {
  const uint8_t* base = reinterpret_cast<const uint8_t*>(code_item);
  const uint8_t* p = base;

  
  
  

  if (code_item->tries_size_ > 0) {
    
    
    
  }

  return static_cast<uint32_t>(p - base);
}


```

很多 “bin 有了但修回去反编译仍坏” 的问题，不是主动调用没触发，而是这里少算了 handler 段。

FART 强在三步闭环，不在单点 hook：

1.  整体 Dex dump
2.  主动调用后 dump CodeItem
3.  用 bin 回补 Dex 并验证可解析

Java 侧主动调用链，常见是从 ClassLoader 枚举下去：

```
Object pathList = getField(classLoader, "pathList");
Object[] dexElements = (Object[]) getField(pathList, "dexElements");

for (Object element : dexElements) {
  Object dexFile = getField(element, "dexFile");
  Object cookie = getField(dexFile, "mCookie");
  String[] classNames = getClassNameList(cookie);

  for (String name : classNames) {
    Class<?> clazz = classLoader.loadClass(name);
    for (Method m : clazz.getDeclaredMethods()) {
      dumpMethodCode(m); 
    }
  }
}


```

native 再转：

```
static void DexFile_dumpMethodCode(JNIEnv* env, jclass, jobject method) {
  if (method == nullptr) return;
  ArtMethod* artmethod = ArtMethod::FromReflectedMethod(...);
  myfartInvoke(artmethod);
}


```

到了高版本，这套闭环还在，但 “调用一定能把真实 CodeItem 暴露出来” 不再自动成立。路径不对，后面全白做。

#### 1. 路径变了

旧路径：

```
主动调用 → Invoke → 解释器 → Execute → 恢复/dump


```

现在常见：

```
主动调用 → Invoke → quick entry → Nterp / AOT / JIT / bridge


```

桥接和 Nterp 特别容易制造假象：

看到 `art_quick_to_interpreter_bridge`，以为稳了  
force interpreter 后看到 Nterp，又以为稳了  
两边都可能不进老 `Execute`

Pixel 7 上那个 `onCreate` 就是标准复现：

```
entry = 0x70b1c8a000
  -> art_quick_to_interpreter_bridge
  -> 老 FART 调，CodeItem 不变

--force-interpreter
  -> entry 切到 Nterp
  -> Execute 不进

dump 点上移到 EnterInterpreterFromInvoke
  -> CodeItem 完整


```

#### 2. 布局变了

`ArtMethod` 大小、字段布局、pointer-sized fields 一直在变。  
写死 Android 8/10 offset 的脚本，到 12 后读错是常态。

与其写死，不如运行时探：

```
const artMethod = ptr(methodAddr);
const accessFlags = artMethod.add(accessFlagsOff).readU32();
const quickCode = artMethod.add(quickCodeOff).readPointer();
console.log('flags=', accessFlags.toString(16), 'entry=', quickCode);


```

CodeItem 获取也建议做成多版本后端，不要假设某一个固定位移永远能取到指令体。

#### 3. 存储变了

`/sdcard/fart/...` 在 Scoped Storage 后经常直接写失败。  
表现很误导：日志像跑完了，目录却是空的。

更稳的路径一般是：

```
/sdcard/Android/data/<pkg>/files/fart/


```

写文件前先把返回值打出来：

```
int fd = open(path.c_str(), O_WRONLY | O_CREAT | O_TRUNC, 0644);
if (fd < 0) {
  LOG(ERROR) << "open failed: " << path << " errno=" << errno;
  return;
}


```

#### 4. 壳对抗变了

现在更常见：

垃圾类一初始化就退  
检测异常 ClassLoader 遍历 / 反射调用  
破坏内存 Dex 头  
恢复时机压到真实执行点  
识别固定线程名、固定路径、固定 so

所以全量主动调用本身就可能成为触发器。

配置化会务实很多：

```
dump=true
sleep=60000
force=com.target.*
ignore=androidx.*,com.google.*,kotlin.,kotlinx.


```

#### 5. 编译策略变了

同一 App，刚装完和跑过一段时间后，entry 状态可以不同。  
有没有 profile、ART 有没有 Mainline 更新，都会改结果。  
“我在 Android 14 上试过” 信息量不够，得补当前 entry 类型和编译状态。

版本自适应先做：

探测 `access_flags` / `quickCode` / `jniCode`  
识别 entry 类型  
CodeItem 读取做成多版本后端

主动调用改成可控配置，不先全量扫。

脱壳点不要只留一个：

<table><thead><tr><th>点位</th><th>作用</th></tr></thead><tbody><tr><td><code>EnterInterpreterFromInvoke</code></td><td>比老 <code>Execute</code> 更靠前，能兜住 Nterp 前后的解释入口</td></tr><tr><td>解释器 / Nterp 附近</td><td>看解释态 CodeItem</td></tr><tr><td><code>Invoke</code> 特殊分支</td><td>主动调用触发</td></tr><tr><td>ClassLinker / DexFile 加载</td><td>整体 Dex</td></tr><tr><td>Frida 枚举 ClassLoader</td><td>补动态加载和遗漏 loader</td></tr></tbody></table>

一个更稳的顺序是：

```
1. 打印 entry / flags
2. 必要时 force interpreter
3. 再看 entry 变成了什么
4. 决定 dump 点放哪
5. 小范围 force 调用
6. 对比调用前后 CodeItem
7. 修复并反编译验收


```

修复得要验收：

Dex 头有没有坏  
method_idx` 对不对  
tries/catch 有没有算对  
修完能否被 jadx/baksmali 正常解析

我现在一般不急着上全量主动调用。

先看整体 Dex 在不在内存里。主体都还没解密，方法体先别谈，否则后面全是空转。确认主体在了，再抽几个方法看 Entry Point，不用多，三五个就行：业务入口一个，壳相关工具类一个，再挑一两个明显被抽空的。把这些记下来就够了：

entry 值  
是不是 bridge  
是不是 nterp  
是不是已经编译  
调用前后 CodeItem 有没有变化

这里有几个坑是反复踩过的。

entry 如果是 `art_quick_to_interpreter_bridge`，别默认后面一定会进老 `Execute`。Android 14 Pixel 7 上那个加固 `onCreate` 就是这样，entry 打出来是 `0x70b1c8a000`，反解到 bridge，按 FART 老路子调，CodeItem 一直不变。后来开 `--force-interpreter`，entry 切到 Nterp，以为稳了，结果 `Execute` 还是没进。Nterp 根本不走那条老路径。最后把 dump 点从 `Execute` 提到 `EnterInterpreterFromInvoke`，才拿到完整 CodeItem。

所以后面我基本按这个习惯处理：

调用前后 CodeItem 完全不变，先查路径，别先去改输出目录。  
force interpreter 之后如果变成 Nterp，而 hook 还钉在 `Execute`，优先上移观察点。  
进程一全量调用就没，先把 force 列表收窄，排查垃圾类。  
日志有、文件没有，先看存储权限和目录是不是根本没写成。

小范围 force 能稳定出 bin 了，再扩。修完一定拿 jadx 或 baksmali 验一下，目录里有文件不算成功。

源码的话，优先翻这些就行：

`art/runtime/art_method.cc`  
`art/runtime/art_method.h`  
`art/runtime/class_linker.cc`  
`art/runtime/instrumentation.cc`  
`art/runtime/interpreter/interpreter.cc`  
`art/runtime/interpreter/interpreter_common.h`  
`art/runtime/entrypoints/...`

读的时候别铺太开，就盯四个问题：entry 是谁设置的，`Invoke` 怎么分流，解释器和 Nterp 怎么接上，哪个位置能稳定看到恢复后的 CodeItem。

高版本继续做抽取壳，已经不是再找一个更靠前的 hook 点就完事了。先看当前方法走哪条执行后端，再决定主动调用要把它往哪条路上逼，最后才是 dump 和修复。路径没对上，后面写再多 dump 代码也没用。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)