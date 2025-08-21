> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2054765-1-1.html)

> [md]# JSC 字节码反编译初探——以 Typora 1.10.8 为例在之前的 [文章](https://www.52pojie.cn/thread-2040749-1-1.html) 中，我们已经尝试了通......

![](https://avatar.52pojie.cn/data/avatar/001/64/88/59_avatar_middle.jpg)xqyqx _ 本帖最后由 xqyqx 于 2025-8-21 09:38 编辑_  

JSC 字节码反编译初探——以 Typora 1.10.8 为例
--------------------------------

在之前的[文章](https://www.52pojie.cn/thread-2040749-1-1.html)中，我们已经尝试了通过 hook node api 的方式替换公钥，这次我们来尝试一下从那个已经被编译为字节码的 jsc 入手

在网上搜索 jsc 反编译相关的内容，找到一篇相关[教程](https://guage.cool/wiz-license.html)，作者通过修改 d8，添加`Disassemble`、`LoadJSC`函数，从而实现解析 jsc，我们也依照此思路进行分析

### 制作反编译器

在安装目录下的 version 文件中可以看到 electron 版本为`32.1.2`，在网上搜索可知对应的 v8 版本为`12.8.374.33`，我们先在本地搭建 v8 编译环境，并`git checkout 12.8.374.33`

由于自 12 版本开始，v8 引擎做了比较大的 api 变动，作者原先教程中的修改代码不再适用，下面是我在此基础上做的修正：

在`src/d8/d8.cpp`中添加下面两个方法：

```
static void Disassemble(v8::internal::Isolate* isolate, 
                        v8::internal::Tagged<v8::internal::BytecodeArray> bytecode, 
                        std::unordered_set<uintptr_t>& visited,
                        int depth) {
  if (depth > 100) { 
    v8::internal::PrintF("Recursion depth limit reached, aborting disassembly for this path.\n");
    fflush(stdout);
    return;
  }

  uintptr_t key = reinterpret_cast<uintptr_t>(bytecode.ptr());
  if (visited.count(key)) {
    return;
  }
  visited.insert(key);
  for (int i = 0; i < depth; ++i) v8::internal::PrintF("  ");
  v8::internal::PrintF("Disassembling BytecodeArray at: %p\n", reinterpret_cast<void*>(bytecode.ptr()));
  fflush(stdout);
  v8::internal::OFStream os(stdout);
  bytecode->Disassemble(os);

  auto consts = bytecode->constant_pool();

  for (int i = 0; i < depth; ++i) v8::internal::PrintF("  ");
  v8::internal::PrintF("Constant pool size: %d\n", consts->length());
  fflush(stdout);

  for (int i = 0; i < consts->length(); i++) {
    auto obj = consts->get(i);
    if (v8::internal::IsSharedFunctionInfo(obj)) {
      auto shared = v8::internal::Cast<v8::internal::SharedFunctionInfo>(obj);

      for (int i = 0; i < depth; ++i) v8::internal::PrintF("  ");
      v8::internal::PrintF("--> Found SFI in constant pool at index %d: ", i);

      auto function_name = shared->Name();
      if (function_name->length() > 0) {
          v8::internal::PrintF("%s\n", function_name->ToCString().get());
      } else {
          v8::internal::PrintF("(anonymous)\n");
      }
      fflush(stdout);

      if (shared->HasBytecodeArray()) {
          Disassemble(isolate, shared->GetBytecodeArray(isolate), visited, depth + 1);
      } else {
          for (int i = 0; i < depth; ++i) v8::internal::PrintF("  ");
          v8::internal::PrintF("    (SFI has no bytecode array, skipping)\n");
          fflush(stdout);
      }
    }
  }
}

void v8::Shell::LoadJSC(const v8::FunctionCallbackInfo<v8::Value>& args) {
  auto isolate = reinterpret_cast<v8::internal::Isolate*>(args.GetIsolate());
  for (int i = 0; i < args.Length(); i++) {
    v8::String::Utf8Value filename(args.GetIsolate(), args[i]);
    if (*filename == NULL) {
      args.GetIsolate()->ThrowException(v8::Exception::Error(
          v8::String::NewFromUtf8(args.GetIsolate(), "Error loading file").ToLocalChecked()));
      return;
    }
    int length = 0;
    auto filedata = reinterpret_cast<uint8_t*>(ReadChars(*filename, &length));
    if (filedata == NULL) {
      args.GetIsolate()->ThrowException(v8::Exception::Error(
          v8::String::NewFromUtf8(args.GetIsolate(), "Error reading file").ToLocalChecked()));
      return;
    }
    v8::internal::AlignedCachedData cached_data(filedata, length);
    auto source = isolate->factory()
                      ->NewStringFromUtf8(base::CStrVector("source"))
                      .ToHandleChecked();
    v8::internal::ScriptDetails script_details;
    v8::internal::MaybeHandle<v8::internal::SharedFunctionInfo> maybe_fun =
        v8::internal::CodeSerializer::Deserialize(isolate, &cached_data, source, script_details);

    v8::internal::Handle<v8::internal::SharedFunctionInfo> fun;
    if (!maybe_fun.ToHandle(&fun)) {
      args.GetIsolate()->ThrowException(v8::Exception::Error(
          v8::String::NewFromUtf8(args.GetIsolate(), "Deserialize failed, possibly version mismatch or invalid .jsc file").ToLocalChecked()));
      delete[] filedata;
      return;
    }

    v8::internal::PrintF("---- Starting disassembly of %s ----\n", *filename);
    fflush(stdout);

    std::unordered_set<uintptr_t> visited;
    Disassemble(isolate, fun->GetBytecodeArray(isolate), visited, 0); 

    v8::internal::PrintF("---- Finished disassembly of %s ----\n", *filename);
    fflush(stdout);

    delete[] filedata;
  }
}

```

并在`Shell::CreateGlobalTemplate`中添加代码：

```
global_template->Set(
    v8::String::NewFromUtf8(isolate, "loadjsc", v8::NewStringType::kNormal)
        .ToLocalChecked(),
    v8::FunctionTemplate::New(isolate, v8::Shell::LoadJSC));

```

在`src/d8/d8.h`的`class Shell`中添加`LoadJSC`声明：

```
static void LoadJSC(const v8::FunctionCallbackInfo<v8::Value>& args);

```

`src/diagnostics/objects-printer.cc`:

注释掉：`PrintSourceCode(os);`

在

```
  os << "\n - age: " << age();
  os << "\n";

```

后添加

```
  os << "\nStart BytecodeArray\n";
  this->GetActiveBytecodeArray(isolate)->Disassemble(os);
  os << "\nEnd BytecodeArray\n";

```

在

```
void HeapObject::HeapObjectShortPrint(std::ostream& os) {
  PtrComprCageBase cage_base = GetPtrComprCageBase();

```

后添加

```
  Isolate* isolate = nullptr;
  if (!GetIsolateFromHeapObject(*this, &isolate) || isolate == nullptr) {
    os << "[!!! Corrupted HeapObject (cannot get Isolate) at "
       << reinterpret_cast<void*>(this->ptr()) << " !!!]";
    return;
  }
  ReadOnlyRoots roots(isolate);
  Tagged<Map> map_of_this_object = this->map(cage_base);
  if (map_of_this_object.ptr() == kNullAddress) {
    os << "[!!! Corrupted HeapObject (null map pointer) at "
       << reinterpret_cast<void*>(this->ptr()) << " !!!]";
    return;
  }
  if (map_of_this_object->map(cage_base) != roots.meta_map()) {
    os << "[!!! Corrupted HeapObject (invalid map) at "
       << reinterpret_cast<void*>(this->ptr()) << " !!!]";
    return;
  }


```

在

```
    os << accumulator.ToCString().get();
    return;
  }


```

后添加

```
  if (map(cage_base)->instance_type() == ASM_WASM_DATA_TYPE) {
    os << "<ArrayBoilerplateDescription> ";
    Cast<ArrayBoilerplateDescription>(*this)
        ->constant_elements()
        ->HeapObjectShortPrint(os);
    return;
  }

```

在

```
    case FIXED_ARRAY_TYPE:
      os << "<FixedArray[" << Cast<FixedArray>(*this)->length() << "]>";

```

后添加

```
      os << "\nStart FixedArray\n";
      Cast<FixedArray>(*this)->FixedArrayPrint(os);
      os << "\nEnd FixedArray\n";

```

在

```
    case OBJECT_BOILERPLATE_DESCRIPTION_TYPE:
      os << "<ObjectBoilerplateDescription["
         << Cast<ObjectBoilerplateDescription>(*this)->capacity() << "]>";

```

后添加

```
      os << "\nStart ObjectBoilerplateDescription\n";
      Cast<ObjectBoilerplateDescription>(*this)
          ->ObjectBoilerplateDescriptionPrint(os);
      os << "\nEnd ObjectBoilerplateDescription\n";

```

在

```
    case FIXED_DOUBLE_ARRAY_TYPE:
      os << "<FixedDoubleArray[" << Cast<FixedDoubleArray>(*this)->length()
         << "]>";

```

后添加

```
      os << "\nStart FixedDoubleArray\n";
      Cast<FixedDoubleArray>(*this)->FixedDoubleArrayPrint(os);
      os << "\nEnd FixedDoubleArray\n";

```

在

```
      } else {
        os << "<SharedFunctionInfo>";
      }

```

后添加

```
      os << "\nStart SharedFunctionInfo\n";
      shared->SharedFunctionInfoPrint(os);
      os << "\nEnd SharedFunctionInfo\n";

```

`src/snapshot/code-serializer.cc`：

替换`SanityCheck`、`SanityCheckWithoutSource`函数：

```
SerializedCodeSanityCheckResult SerializedCodeData::SanityCheck(
    uint32_t expected_ro_snapshot_checksum,
    uint32_t expected_source_hash) const {
  return SerializedCodeSanityCheckResult::kSuccess;
}

SerializedCodeSanityCheckResult SerializedCodeData::SanityCheckWithoutSource(
    uint32_t expected_ro_snapshot_checksum) const {
  // Always return kSuccess to bypass all checks.
  return SerializedCodeSanityCheckResult::kSuccess;
}

```

`src/snapshot/deserializer.cc`：

替换`ReadReadOnlyHeapRef`函数：

```
int Deserializer<IsolateT>::ReadReadOnlyHeapRef(uint8_t data,
                                                SlotAccessor slot_accessor) {
  uint32_t chunk_index = source_.GetUint30();
  uint32_t chunk_offset = source_.GetUint30();

  ReadOnlySpace* read_only_space = isolate()->heap()->read_only_space();

  if (chunk_index >= read_only_space->pages().size()) {
    Tagged<Hole> the_hole = *isolate()->factory()->the_hole_value();

    return WriteHeapPointer(slot_accessor, the_hole,
                            GetAndResetNextReferenceDescriptor());
  }

  ReadOnlyPageMetadata* page = read_only_space->pages()[chunk_index];
  Address address = page->OffsetToAddress(chunk_offset);
  Tagged<HeapObject> heap_object = HeapObject::FromAddress(address);

  return WriteHeapPointer(slot_accessor, heap_object,
                          GetAndResetNextReferenceDescriptor());
}

```

以上为全部修改，之后使用`ninja -C out.gn/x64.release d8`编译

编译好后运行`./out.gn/x64.release/d8 -e "loadjsc('atom.compiled.dist.jsc')" > atom.txt`即可得到反编译后的结果：

[https://wwri.lanzouo.com/iB9Ea34181jc](https://wwri.lanzouo.com/iB9Ea34181jc)

### 分析 atom.txt

面对海量的字节码，我们直奔主题，寻找 rsa 公钥，在之前版本的 atom.js 中，我们可以得知公钥是 base64 解析出来的：

```
T = JSON.parse(Buffer.from("WyItLS0tLUJFR0lOIFBVQkxJQyBLRVktLS0tLSIsIk1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBN25Wb0dDSHFJTUp5cWdBTEVVcmMiLCI1SkpoYXAwK0h0SnF6UEUwNHB6NHkrbnJPbVk3LzEyZjNIdlp5eW9Sc3hLZFhUWmJPMHdFSEZJaDBjUnFzdWFKIiwiUHlhT09QYkEwQnNhbG9mSUFZM21SaFFRM3ZTZitybjNnK3cwUyt1ZFdtS1Y5RG5tSmxwV3FpekZhalU0VC9FNCIsIjVaZ01OY1h0M0UxaXBzMzJyZGJUUjBObmVuOVBWSVR2cmJKM2w2Q0kyQkZCSW1aUVoyUDhOK0xzcWZKc3F5VlYiLCJ3RGt0M21IQVZ4VjdGWmJmWVdHKzhGRFN1S1FIYUNtdmdBdENoeDlod2wzSjZSZWtrcURWYTZHSVYxM0QyM0xTIiwicWRrMEpiNTIxd0ZKaS9WNlFBSzZTTEJpYnk1Z1lONnpRUTVSUXBqWHRSNTNNd3pUZGlBekdFdUtkT3RyWTJNZSIsIkR3SURBUUFCIiwiLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tIiwiIiwiIl0=","base64").toString("utf8")).join("\n"),
I = 864e5;
var W = "https://store.typora.io";

```

我们直接在反编译结果中搜索这段 base64 没有搜索到，怀疑是 v8 对其进行了优化，我们可以通过后面的字符串`https://store.typora.io`来尝试对这段代码进行定位，在反编译结果中搜索，向上可以找到一个长度为 476 的数组：

```
0x3340001a57f9: [FixedArray] in OldSpace
 - map: [!!! Corrupted HeapObject (cannot get Isolate) at 0x33400000065d !!!]
 - length: 476
           0: 91
           1: 34
         2-6: 45
           7: 66
           8: 69
           9: 71
          10: 73
          11: 78
          12: 32
          13: 80
          14: 85
          15: 66
          16: 76
          17: 73
          18: 67

```

将数字转换为 ascii，发现是 PEM 公钥的数组形式

那我们如何对其进行修改呢？

注意到这个数组在 Constant pool 中，也就是储存在堆中的，而不是动态生成，因此我们可以在 jsc 文件中找到并对其进行修改

在 jsc 中我们同样以`https://store.typora.io`作为锚点进行寻找，发现往上有一个很长的数据块：

```
03 00 00 B6 00 00 00 44 00 00 00 5A 00 00 00 5A
00 00 00 5A 00 00 00 5A 00 00 00 5A 00 00 00 84
00 00 00 8A 00 00 00 8E 00 00 00 92 00 00 00 9C
00 00 00 40 00 00 00 A0 00 00 00 AA 00 00 00 84
00 00 00 98 00 00 00 92 00 00 00 86 00 00 00 40
00 00 00 96 00 00 00 8A 00 00 00 B2 00 00 00 5A
00 00 00 5A 00 00 00 5A 00 00 00 5A 00 00 00 5A
00 00 00 44 00 00 00 58 00 00 00 44 00 00 00 9A
00 00 00 92 00 00 00 92 00 00 00 84 00 00 00 92
00 00 00 D4 00 00 00 82 00 00 00 9C 00 00 00 84
......

```

这里我们发现了两组 5 个重复的 0x5A，中间夹着 16 个字节，这恰好与我们先前在 Constant pool 中找到的数组相对应（两组 5 个重复的 45 中间夹着 16 个数），可以确定这段数据就是公钥

尝试异或发现不对，于是猜测应该是类似 cython 一样，每个字符都有对应的标识符（例如这里 0x5A 对应 45），由于原始公钥内有足够多的字符，应该可以生成一个对应表，这样就可以替换公钥了

关于网验：  
可以找到下面一段字节码：

```
         0xba50004e020 @   76 : c2                Star8
         0xba50004e021 @   77 : 0d 2f             LdaSmi [47]
         0xba50004e023 @   79 : c0                Star10
         0xba50004e024 @   80 : 0d 61             LdaSmi [97]
         0xba50004e026 @   82 : bf                Star11
         0xba50004e027 @   83 : 0d 70             LdaSmi [112]
         0xba50004e029 @   85 : be                Star12
         0xba50004e02a @   86 : 0d 69             LdaSmi [105]
         0xba50004e02c @   88 : bd                Star13
         0xba50004e02d @   89 : 0d 2f             LdaSmi [47]
         0xba50004e02f @   91 : bc                Star14
         0xba50004e030 @   92 : 0d 63             LdaSmi [99]
         0xba50004e032 @   94 : bb                Star15
         0xba50004e033 @   95 : 0d 6c             LdaSmi [108]
         0xba50004e035 @   97 : 18 e9             Star r16
         0xba50004e037 @   99 : 0d 69             LdaSmi [105]
         0xba50004e039 @  101 : 18 e8             Star r17
         0xba50004e03b @  103 : 0d 65             LdaSmi [101]
         0xba50004e03d @  105 : 18 e7             Star r18
         0xba50004e03f @  107 : 0d 6e             LdaSmi [110]
         0xba50004e041 @  109 : 18 e6             Star r19
         0xba50004e043 @  111 : 0d 74             LdaSmi [116]
         0xba50004e045 @  113 : 18 e5             Star r20
         0xba50004e047 @  115 : 0d 2f             LdaSmi [47]
         0xba50004e049 @  117 : 18 e4             Star r21
         0xba50004e04b @  119 : 0d 72             LdaSmi [114]
         0xba50004e04d @  121 : 18 e3             Star r22
         0xba50004e04f @  123 : 0d 65             LdaSmi [101]
         0xba50004e051 @  125 : 18 e2             Star r23
         0xba50004e053 @  127 : 0d 6e             LdaSmi [110]
         0xba50004e055 @  129 : 18 e1             Star r24
         0xba50004e057 @  131 : 0d 65             LdaSmi [101]
         0xba50004e059 @  133 : 18 e0             Star r25
         0xba50004e05b @  135 : 0d 77             LdaSmi [119]

```

对应的字符串为`/api/client/renew`，用 16 进制搜索替换掉这些立即数即可![](https://avatar.52pojie.cn/images/noavatar_middle.gif)pangpang12138 感谢楼主的精彩分享，受益匪浅！![](https://avatar.52pojie.cn/data/avatar/001/37/21/93_avatar_middle.jpg)我是不会改名的 应该就是右移 1 位  
0xb6>>1     91  
0x44>>1     34 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 感谢分享，新的思路![](https://avatar.52pojie.cn/images/noavatar_middle.gif)PoJieDaWang123 来了来了，看到 Typora 必进。感谢楼主，也感谢 Typora 对技术普及作出的贡献