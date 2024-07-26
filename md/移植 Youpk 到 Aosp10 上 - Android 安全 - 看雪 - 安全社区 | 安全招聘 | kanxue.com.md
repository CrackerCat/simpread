> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282634.htm)

> 移植 Youpk 到 Aosp10 上

移植 Youpk 到 Aosp10 上
===================

参考文章：

[https://github.com/jitcor/Youpk8](https://github.com/jitcor/Youpk8)

[https://github.com/Youlor/Youpk](https://github.com/Youlor/Youpk)

[https://bbs.kanxue.com/thread-271358-1.htm#msg_header_h3_0](https://bbs.kanxue.com/thread-271358-1.htm#msg_header_h3_0)

[https://www.jianshu.com/p/18ff0b8e0b01](https://www.jianshu.com/p/18ff0b8e0b01)

Fart 脱壳王课程

更新说明：
=====

本文章会定时更新，有问题直接向我提问，我会**补充到文章**里

同时我会在完善编译的所有流程（为小白铺路）

并完善移植原理的解读（为想要了解原理的大佬解答）

同时我会完善我如何从 aosp8 移植到 10 的过程，如何去寻找变更 api（授人以渔，**大家学会可以自行把我目前的移植到 aosp12**）

前言：
===

Youpk 是一个很强大的框架，他的模块化组织形式非常新颖，但是随着安卓系统的不断更新，移植难度也非常大，由于使用了大量的 api，导致移植有一定的难度，与 fart 相比，模块化的插桩更加优雅。

已经有大佬做了 fart10 的移植 (见参考文章 3），我这里就不和他重复了，来尝试下 youpk 的移植，并**去除特征指纹**。

测试设备：pixel1

aosp 移植版本：**aosp10.0.0_r2（本来想移植 fartext，失败了）**

（aosp11 与 10 的 api 相关类似，可以自己尝试）

移植前需要注意的
========

1. 需要你有基础的 aosp 编译修改经验，简单修改能编译成功

1.  需要有趁手的 ide 修改经验 我使用的是 android studio（调试 java 层） clion（调试 native 层）
2.  可能你移植全部完成以后，还是过不去企业壳，但是你能了解基本的 art 修改思想
3.  如果你准备好开始，检测你的内存和硬盘 **内存推荐大于 32G**（ide 要占用 10+10 左右）

**硬盘需要大于 1TB 我是 32G+2TB 编译体验非常差 有条件推荐上 4TB+64G**

[如何开启 clion 和 android studio 导入项目源码：](%E7%A7%BB%E6%A4%8DYoupk%E5%88%B0Aosp10%E4%B8%8A%20ead82bb5990c4574a9fc0e5d899beaa1/%E5%A6%82%E4%BD%95%E5%BC%80%E5%90%AFclion%E5%92%8Candroid%20studio%E5%AF%BC%E5%85%A5%E9%A1%B9%E7%9B%AE%E6%BA%90%E7%A0%81%EF%BC%9A%20e235d254b6f4418a92dda80d5477d732.md)

编译的时候要注意 repo 的 python 版本，最好大于 3.7 如果在低版本 ubuntu 系统，**需要自己编译 python**

repo fatal: error unknown url type: https

原因：python 没有设置 ssl

./configure --prefix=/usr/local/python3 --with-ssl

解决文档：

[https://stackoverflow.com/questions/18317682/android-aosp-repo-init-fatal-error-unknown-url-type-https](https://stackoverflow.com/questions/18317682/android-aosp-repo-init-fatal-error-unknown-url-type-https)

解决：在编译的时候配置 ssl

**找到一个趁手的编译环境 + IDE 环境是成功编译的第一步骤**

开始初步移植 JAVA 层
=============

由于 7.1→8.0→10.0 有多版本跨度，我们**选择 youpk8 的源码**进行移植，来减少 api 差异

第一步，导入 unpacker 类到

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432996.png)

这里我们可以第一步去除指纹，修改包名

我这里的包名是 com.jiqiu

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432005.png)

```
package com.jiqiu;
import android.app.ActivityThread;
import android.os.Looper;
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.File;
 
public class Unpacker {
//    public static String UNPACK_CONFIG = "/data/local/tmp/unpacker.config";
    //  去指纹位置2，修改配置名文件，不一定需要config尾缀
    public static String UNPACK_CONFIG = "/data/local/tmp/gagaga";
    public static int UNPACK_INTERVAL = 10 * 1000;
    public static Thread unpackerThread = null;
 
    public static boolean shouldUnpack() {
        boolean should_unpack = false;
        String processName = ActivityThread.currentProcessName();
        BufferedReader br = null;
        try {
            br = new BufferedReader(new FileReader(UNPACK_CONFIG));
            String line;
            while ((line = br.readLine()) != null) {
                if (line.equals(processName)) {
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
 
    public static void unpack() {
        if (Unpacker.unpackerThread != null) {
            return;
        }
 
        if (!shouldUnpack()) {
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
                    Unpacker.unpackNative();
                }
            }
        };
        Unpacker.unpackerThread.start();
    }
 
    public static native void unpackNative();
}

```

**类名也完全可以修改**，在修改后要在

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432014.png)

这个文件里加上自己的包名，否则编译不过

比如我打的包名是 com.jiqiu

在里面就是

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432024.png)

之后进入

core/java/android/app/ActivityThread.java

导入自己的包名

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432034.png)

在 app 启动后，注入自己的脱壳线程

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432044.png)

注意：如果你修改了类名，这里的导入和调用也需要修改

NATIVE 层移植
==========

想比于 fart，youpk 的主动调用部分在 native 层实现，java 层仅仅是启动一个线程 启动 native 函数

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432057.png)

[第一步修改 dexopt.cc](http://%E7%AC%AC%E4%B8%80%E6%AD%A5%E4%BF%AE%E6%94%B9dexopt.cc) 路径如上

```
--- a/dex2oat/dex2oat.cc
+++ b/dex2oat/dex2oat.cc
@@ -1036,6 +1036,8 @@ class Dex2Oat final {
         CompilerFilter::NameOfFilter(compiler_options_->GetCompilerFilter()));
     key_value_store_->Put(OatHeader::kConcurrentCopying,
                           kUseReadBarrier ? OatHeader::kTrueValue : OatHeader::kFalseValue);
+
+
     if (invocation_file_.get() != -1) {
       std::ostringstream oss;
       for (int i = 0; i < argc; ++i) {
@@ -1089,7 +1091,23 @@ class Dex2Oat final {
       *out = true;
     }
   }
-
+    //patch by Youlor
+    //++++++++++++++++++++++++++++
+    const char* UNPACK_CONFIG = "/data/local/tmp/gagaga";
+    bool ShouldUnpack() {
+        std::ifstream config(UNPACK_CONFIG);
+        std::string line;
+        if(config) {
+            while (std::getline(config, line)) {
+                std::string package_name = line.substr(0, line.find(':'));
+                if (oat_location_.find(package_name) != std::string::npos) {
+                    return true;
+                }
+            }
+        }
+        return false;
+    }
+    //++++++++++++++++++++++++++++
   // Parse the arguments from the command line. In case of an unrecognized option or impossible
   // values/combinations, a usage error will be displayed and exit() is called. Thus, if the method
   // returns, arguments have been successfully parsed.
@@ -1240,7 +1258,14 @@ class Dex2Oat final {
     ProcessOptions(parser_options.get());
  
     // Insert some compiler things.
+
     InsertCompileOptions(argc, argv);
+    //patch by Youlor
+    //++++++++++++++++++++++++++++
+      if (ShouldUnpack()) {
+          compiler_options_->SetCompilerFilter(CompilerFilter::kVerify);
+      }
+  //++++++++++++++++++++++++++++
   }

```

注意，这里的 config 路径一定要与 java 层设置的一致，见去指纹 2

const char* UNPACK_CONFIG = "/data/local/tmp/gagaga";

拷贝 youpk 项目到 art/runtime 目录下
----------------------------

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432067.png)

并且修改 Android.bp

添加目标编译文件

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432077.png)

添加编译文件后我们就可以初步处理 youpk 的所有不兼容 api，附件提供了修改前后的 youpk 文件夹，在数十次的编译中，已经修改成安卓系统最新支持 api

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432088.png)

在新版系统编译，这个宏定义视为不安全，直接使用 math 库的同名函数即可过编译，记得注释原来的

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432098.png)

在新版系统中，dex 相关库文件移动到了 libdexfile 文件夹下，我们只需改动 libdexfile/Android.bp

导出其依赖的库，并按新版文件调用，即可解决依赖问题

```
   // Check whether the oat output files are writable, and open them for later. Also open a swap
diff --git a/libdexfile/Android.bp b/libdexfile/Android.bp
index 30d1bcd..2ff2f10 100644
--- a/libdexfile/Android.bp
+++ b/libdexfile/Android.bp
@@ -95,7 +95,7 @@ cc_defaults {
         },
     },
     generated_sources: ["dexfile_operator_srcs"],
-    export_include_dirs: ["."],
+    export_include_dirs: [".","dex"],
 }

```

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432109.png)

globals 位置发生改变

#include "base/globals.h”

mirror::Class * 指针修改为 ObjPtrmirror::Class

setstatus 的状态码发生改变 mirror::Class::kStatusInitialized 变为 ClassStatus::kInitialized

删除 size_t Unpacker::getCodeItemSize(ArtMethod* method) 方法 新版有 api 可以直接实现

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432119.png)

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432134.png)

uint32_t code_item_size = method->GetDexFile()->GetCodeItemSize(*code_item);

可以直接获取到 codeitem 的 size 省去了上面的函数

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432144.png)

method->GetCodeItem()->insns_;

新版本的 codeitem 没有 insns_属性，需要迭代器访问

见参考文章 4

修改为 const uint16_t* const insns = CodeItemInstructionAccessor(*method->GetDexFile(),method->GetCodeItem()).Insns(); 即可编译通过

最后修改注册函数

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432155.png)

REGISTER_NATIVE_METHODS("com/jiqiu/Unpacker");

包名和类名修改过的可以去修改下

art 各生命周期函数插桩
=============

artmethod.cc 注册函数
-----------------

```
--- a/runtime/runtime.cc
+++ b/runtime/runtime.cc
@@ -15,7 +15,9 @@
  */
  
 #include "runtime.h"
-
+//add
+#include "unpacker/unpacker.h"
+//addend
 // sys/mount.h has to come before linux/fs.h due to redefinition of MS_RDONLY, MS_BIND, etc
 #include #ifdef __linux__
@@ -1907,6 +1909,10 @@ void Runtime::RegisterRuntimeNativeMethods(JNIEnv* env) {
   register_org_apache_harmony_dalvik_ddmc_DdmServer(env);
   register_org_apache_harmony_dalvik_ddmc_DdmVmInternal(env);
   register_sun_misc_Unsafe(env);
+
+  //add
+  Unpacker::register_cn_youlor_Unpacker(env);
+  //addend
 }
  
 std::ostream& operator<<(std::ostream& os, const DeoptimizationKind& kind) { 
```

修改 art/runtime 下的 Android.bp 使其走向选择分支解释模式
-----------------------------------------

```
--- a/runtime/Android.bp
+++ b/runtime/Android.bp
@@ -350,6 +352,9 @@ libart_cc_defaults {
                 // ART is allowed to link to libicuuc directly
                 // since they are in the same module
                 "-DANDROID_LINK_SHARED_ICU4C",
+                 "-Wno-error",
+                "-DART_USE_CXX_INTERPRETER=1",
             ],
         },

```

class_linker.h 增加友元函数，使其可以访问内部的 dex 缓存字段
----------------------------------------

```
--- a/runtime/class_linker.h
+++ b/runtime/class_linker.h
@@ -1385,6 +1385,9 @@ class ClassLinker {
   class FindVirtualMethodHolderVisitor;
  
   friend class AppImageLoadingHelper;
+  //add
+  friend class Unpacker;
+  //addend
   friend class ImageDumper;  // for DexLock
   friend struct linker::CompilationHelper;  // For Compile in ImageTest.
   friend class linker::ImageWriter;  // for GetClassRoots
diff --git a/runtime/interpreter/interpreter_switch_impl-inl.h b/runtime/interpreter/interpreter_switch_impl-inl.h
index 36cfee4..b6e5ff6 100644

```

修改 artmethod 增加判断分支（这里和 youpk 原版移植方式一样）
---------------------------------------

```
diff --git a/runtime/art_method.cc b/runtime/art_method.cc
index 0890da8..2cd96d2 100644
--- a/runtime/art_method.cc
+++ b/runtime/art_method.cc
@@ -50,7 +50,9 @@
 #include "runtime_callbacks.h"
 #include "scoped_thread_state_change-inl.h"
 #include "vdex_file.h"
-
+//add
+#include "unpacker/unpacker.h"
+//addend
 namespace art {
  
 using android::base::StringPrintf;
@@ -322,13 +324,28 @@ void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue*
   // If the runtime is not yet started or it is required by the debugger, then perform the
   // Invocation by the interpreter, explicitly forcing interpretation over JIT to prevent
   // cycling around the various JIT/Interpreter methods that handle method invocation.
-  if (UNLIKELY(!runtime->IsStarted() ||
-               (self->IsForceInterpreter() && !IsNative() && !IsProxyMethod() && IsInvokable()) ||
-               Dbg::IsForcedInterpreterNeededForCalling(self, this))) {
+
+//  if (UNLIKELY(!runtime->IsStarted() ||
+//               (self->IsForceInterpreter() && !IsNative() && !IsProxyMethod() && IsInvokable()) ||
+//               Dbg::IsForcedInterpreterNeededForCalling(self, this))) {
+//add
+    if (UNLIKELY(!runtime->IsStarted() || Dbg::IsForcedInterpreterNeededForCalling(self, this)
+                 || (Unpacker::isFakeInvoke(self, this) && !this->IsNative()))) {
+
+        //addend
     if (IsStatic()) {
       art::interpreter::EnterInterpreterFromInvoke(
           self, this, nullptr, args, result, /*stay_in_interpreter=*/ true);
     } else {
+        //patch by Youlor
+        //++++++++++++++++++++++++++++
+        //如果是主动调用fake invoke并且是native方法则不执行
+        if (Unpacker::isFakeInvoke(self, this) && this->IsNative()) {
+            // Pop transition.
+            self->PopManagedStackFragment(fragment);
+            return;
+        }
+        //++++++++++++++++++++++++++++
       mirror::Object* receiver =
           reinterpret_cast*>(&args[0])->AsMirrorPtr();
       art::interpreter::EnterInterpreterFromInvoke(
diff --git a/runtime/class_linker.h b/runtime/class_linker.h 
```

解释器分支移植
=======

a/runtime/interpreter/interpreter_switch_impl-inl.h

注意这个不是在 cpp 中实现了，**在 interpreter_switch_impl-inl.h 头中实现**

而且 youpk 插桩的宏定义在 aosp10 中改为了函数判断，无法在函数中插桩

只需要在相应位置插桩即可解决（已给出 patch）

```
--- a/runtime/interpreter/interpreter_switch_impl-inl.h
+++ b/runtime/interpreter/interpreter_switch_impl-inl.h
@@ -18,7 +18,9 @@
 #define ART_RUNTIME_INTERPRETER_INTERPRETER_SWITCH_IMPL_INL_H_
  
 #include "interpreter_switch_impl.h"
-
+//add
+#include "unpacker/unpacker.h"
+//addend
 #include "base/enums.h"
 #include "base/globals.h"
 #include "base/memory_tool.h"
@@ -225,6 +227,7 @@ class InstructionHandler {
     if (!CheckForceReturn()) {
       return false;
     }
+
     if (UNLIKELY(instrumentation->HasDexPcListeners())) {
       uint8_t opcode = inst->Opcode(inst_data);
       bool is_move_result_object = (opcode == Instruction::MOVE_RESULT_OBJECT);
@@ -243,6 +246,8 @@ class InstructionHandler {
         return false;
       }
     }
+
+      //addend
     return true;
   }
  
@@ -2643,12 +2648,25 @@ ATTRIBUTE_NO_SANITIZE_ADDRESS void ExecuteSwitchImplCpp(SwitchImplContext* ctx)
       << "Entered interpreter from invoke without retry instruction being handled!";
  
   bool const interpret_one_instruction = ctx->interpret_one_instruction;
+
+  //add
+  int inst_count=-1;
+  //addend
   while (true) {
     dex_pc = inst->GetDexPc(insns);
     shadow_frame.SetDexPC(dex_pc);
     TraceExecution(shadow_frame, inst, dex_pc);
     inst_data = inst->Fetch16(0);
     {
+    //add
+    inst_count++;                                                                               \
+    bool dumped = Unpacker::beforeInstructionExecute(self, shadow_frame.GetMethod(),            \
+                                                     dex_pc, inst_count);                       \
+
+      if(dumped) {
+          return;
+      }
+      //addend
       bool exit_loop = false;
       InstructionHandler handler(
           ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, exit_loop);
@@ -2662,6 +2680,7 @@ ATTRIBUTE_NO_SANITIZE_ADDRESS void ExecuteSwitchImplCpp(SwitchImplContext* ctx)
         continue;
       }
     }
+
     switch (inst->Opcode(inst_data)) {
 #define OPCODE_CASE(OPCODE, OPCODE_NAME, pname, f, i, a, e, v)                                    \
       case OPCODE: {                                                                              \
@@ -2681,6 +2700,13 @@ DEX_INSTRUCTION_LIST(OPCODE_CASE)
     if (UNLIKELY(interpret_one_instruction)) {
       break;
     }
+      //patch by Youlor
+      //++++++++++++++++++++++++++++
+      bool dumped = Unpacker::afterInstructionExecute(self, shadow_frame.GetMethod(), dex_pc, inst_count);
+      if (dumped) {
+          return ;
+      }
+      //++++++++++++++++++++++++++++
   }
   // Record where we stopped.
   shadow_frame.SetDexPC(inst->GetDexPc(insns));
diff --git a/runtime/runtime.cc b/runtime/runtime.cc
index 51a40e7..275324c 100644 
```

成品测试
====

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432166.png)

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432178.png)

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432188.png)

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432199.png)

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432210.png)

![](https://qiude1tuchuang.oss-cn-beijing.aliyuncs.com/blog/202407261432220.png)

关于 Youpk 与 FART 检测的思路
=====================

youpk 独特检测思路
------------

1.  由于作者指定了特定名称的包名，所以可以通过反射的方法来检测是否存在这个类，如果存在，则判断为脱壳机环境（在系统的类里面，可以使用 hideapi 来绕过反射限制）

[https://github.com/LSPosed/AndroidHiddenApiBypass](https://github.com/LSPosed/AndroidHiddenApiBypass)

1.  由于作者独特的包名过滤机制，需要配置特定名字的 config 在 data/local/tmp 下 其他 app 可扫描这个目录下的 config，来判断脱壳环境

解决方法：

换一个注册的名字，可以选定一些厂商的特殊包名注册，如 xiaomi meizu huawei 等特殊包名

修改 config 名字或落地地方挑选 app 不可达路径（之后会集中讨论）

fart8 独特检测思路
------------

DexFIile 静态注册了一个函数，作为与 native 桥接的函数，由于比较独特，直接可以通过反射调用，甚至可以做进一步下沉，在 libart.so 的导出表中发现这个函数

在 ActivityThred 中出现了很多工具类函数，可以反射调用检测，以及在 art_method 中有额外的导出函数，可以通过扫描 libart.so 的导出表来扫描制定名字

解决方法：

1.  学习 youpk，将逻辑打包到一个包里，进行插桩复用
2.  进行 api 复用（fart10 已经做到），将必要的逻辑实现在系统 api 里，通过参数判断是否返回

共同监测点
-----

### 文件落地的检测，比如 fart 选择落地在 sdcard 下的文件夹，厂商可以选择对 sdcard 做扫描，来判断该机器是否是脱壳机，以及自己私有目录下异常文件的扫描（以及 user 落地后无法提取）

**难点：**

权限的申请（脱壳机一般都有）

用户隐私的保护 （厂商一般不管）

**解决办法**：

修改系统 selinux，注册全新的 selinux 标签，编写系统应用，进行文件的存储和获取（小肩膀沙盒定制思路）

注册系统服务，app 通过调用，存储到 / system 目录下

**以上两种方法均可进行 dex 和 config 文件的落地**

### 导出函数的名字，以及导出函数数量的 api 的检测

**难点：**

各大厂商对于 rom 均有定制，libart.so 数量均不一致

无法太大的做到导出函数数量的特征（这个是真的致命打击）

**解决办法：**

建立机型库，对机型的各个文件进行模型建立，检测是否为异常 libart，判断是否为异常机型

### aosp 的检测 以及机型的检测

脱壳机一般使用 pixel 以及 nexus 等机型做定制 rom，可以针对这些机型做风控（误杀率高）

所以可以对 aosp 的定制进行检测，对 aosp 的指纹进行检测（目前大部分脱壳机过不去企业壳都折在了这里）

解决办法：

使用开源的 lineageos 以及 pixel experience 系统进行定制，这些系统都已经去除了很多 aosp 的特征

（除非日后国内哪家厂商开源了他们的操作系统）

使用支持这些 rom 的手机进行定制，这里我强烈推荐一加手机，简直无敌 随便刷随便解锁 还能 9008 救砖（比某 xel6 代好太多了）

附件： 提供下载
========

[https://fortunate-decimal-730.notion.site/Youpk-Aosp10-ead82bb5990c4574a9fc0e5d899beaa1?pvs=4](https://fortunate-decimal-730.notion.site/Youpk-Aosp10-ead82bb5990c4574a9fc0e5d899beaa1?pvs=4)

移植后的 unpcker 模块

所有 patch：

[Untitled](%E7%A7%BB%E6%A4%8DYoupk%E5%88%B0Aosp10%E4%B8%8A%20ead82bb5990c4574a9fc0e5d899beaa1/Untitled.txt)

[[竞赛]2024 KCTF 大赛征题截止日期 08 月 10 日！](https://bbs.kanxue.com/thread-281194.htm)

最后于 2 小时前 被 mb_qzwrkwda 编辑 ，原因：

[#源码框架](forum-161-1-127.htm)