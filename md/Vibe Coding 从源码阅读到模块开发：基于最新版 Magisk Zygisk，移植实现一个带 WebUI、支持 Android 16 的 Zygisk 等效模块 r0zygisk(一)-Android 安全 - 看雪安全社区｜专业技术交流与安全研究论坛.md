> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291209.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

这篇文章主要记录了基于最新版 `Magisk` 的 `Zygisk`，整理并移植到 `r0zygisk` 的完整过程，同时把 `KernelSU / SukiSU` 环境下的 WebUI 一并补齐。

完整源码和发布包已经开源在 GitHub Releases：

[r0zygisk Releases](https://bbs.kanxue.com/elink@f5dK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6X3P5i4u0D9L8%4k6W2i4K6u0r3M7U0m8*7P5h3N6A6M7$3E0Q4x3V1k6J5k6h3I4W2j5i4y4W2M7H3%60.%60.)

整个过程可以归纳成八条主线：

1.  先解决项目本身无法构建的问题。
2.  再修复生成 zip 后 `SukiSU / KernelSU` 无法安装的问题（缺少正确目录的 `libzygisk.so`）。
3.  然后迁移 `r0zygisk` 的 Zygisk API，以及 Android 16 / Baklava 相关 JNI 签名。
4.  接着梳理 `zygiskd` 的执行链路，为后续排查 `daemon / loader / companion` 问题建立上下文。
5.  再重构 WebUI，并围绕真实刷入反馈连续修复 `exec`、状态显示和 SukiSU 模块描述污染问题。
6.  完成第一版阶段性收敛，固化可复现的构建与刷入结果。
7.  对 Hunter 检测链路做逆向定位，明确命中点在 native 侧 `checkZygisk` 分支。
8.  结合迁移记录完成第二版大修：切换注入模式并统一命名为 `r0z`。

1.  项目背景与环境
2.  先确认最新版 Magisk Zygisk 是否已适配 Android 16
3.  借助 code-panorama + Codex 梳理 Zygisk 源码
4.  开始移植：把最新版 Zygisk 迁到 r0zygisk
5.  WebUI 重构与联调记录
6.  最终状态
7.  第二版大修：通过 Hunter 的 Zygisk 检测
8.  结合 AI：使用新的注入方式完成第二版代码
9.  总结

关于 [Magisk](https://bbs.kanxue.com/elink@ccbK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6@1L8%4m8B7L8$3S2F1N6%4g2Q4x3V1k6y4j5h3N6A6M7$3D9%60.) 和 `ZygiskNext` 老版本源码的阅读，这次我用到了一个开源库：

https://github.com/xuanyuanzhifeng/code-panorama

这个工具对源码阅读和结构定位帮助很大。

文章整体也使用了 AI 辅助完成，具体环境是：`codex cli + gpt5.4` 模型。  
![](https://bbs.kanxue.com/upload/attach/202605/703941_C38QYK7VDTKDV8J.png)

结果测试覆盖了 Android 14 到 Android 16 机型。

项目初始目录结构如下：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_VCBPCKAXJB3QDPX.png)

`r0zygisk` 是参考 `ZygiskNext` 的内容继续做的，使用 Android Studio 编译，开发环境如下。

Android Gradle Plugin 和 Gradle Wrapper 版本：

*   Android Gradle Plugin：`8.9.1`
*   Gradle Wrapper：`8.11.1`

JDK 配置在 `gradle.properties` 中：

*   org.gradle.java.home=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home

项目打包过程中，前期遇到的问题主要有这些：

1.  Gradle 使用 Java 8，改为项目级固定 JDK 17。
2.  Java/Gradle 访问 Google Maven TLS 失败，增加国内 Maven 镜像。
3.  中文路径导致 Gradle/JVM 编码问题，增加 UTF-8 JVM 参数。
4.  当前目录不是 Git 仓库，版本号读取失败，增加 fallback。
5.  本机缺少项目指定 NDK，改为本机已安装的 NDK `26.1.10909125`。
6.  源码包缺少 `lsplt` 子模块，按 `.gitmodules` 克隆补齐。
7.  Rust 缺少 Android 交叉编译 target，用 `rustup target add` 安装四个目标平台。

开发环境准备好之后，就开始进入正式工作：基于最新版 `Magisk` 的 `Zygisk`，做一个支持 Android 16 的 `r0zygisk`。

动手移植之前，先确认当前 `Magisk-master` 这版到底有没有完成 Android 16 适配。

我先让 AI 帮我判断这版 `Magisk-master` 的适配情况，提问如下：

```
目录中的 Magisk-master 是最新版 Magisk，分析这个版本和之前版本的差异，是否适配了 Android 16。


```

![](https://bbs.kanxue.com/upload/attach/202605/703941_HA43BBZKSRNBBCE.png)

得到的分析结果如下：

```
• 结论：Magisk-master 这个版本已经适配 Android 16，并且适配点明确覆盖 Android 16 QPR2。更准确地说，它不是只把 App 的 targetSdk 改到 36，而是同时处理了 MagiskInit、MagiskPolicy、
  Zygisk 三个对 root/注入影响最大的底层点。

  本地版本判断
  本地 Magisk-master/docs/changes.md:3 显示当前是 v30.7 (2026.2.23)，重点变更是：

  - Magisk-master/docs/changes.md:5：MagiskInit 支持 Android 16 QPR2 sepolicy format
  - Magisk-master/docs/changes.md:7：Zygisk 支持 Android 16 QPR2 and higher
  - Magisk-master/docs/changes.md:29：v30.3 已经加入 Android 16 QPR2 新 sepolicy binary format 支持

  App 构建配置也对上了 Android 16/QPR2：

  - Magisk-master/app/build-logic/src/main/java/Setup.kt:46：compileSdk 使用 release(36)
  - Magisk-master/app/build-logic/src/main/java/Setup.kt:48：minorApiLevel = 1，对应 Android 16 QPR2 SDK 36.1
  - Magisk-master/app/build-logic/src/main/java/Setup.kt:229：targetSdk = 36

  这些配置和 Google 官方 Android 16/QPR2 SDK 文档一致：Android 16 是 API 36，QPR2 使用 release(36) { minorApiLevel = 1 }。参考 Android 官方 API level 表、Android 16 SDK 设置、
  Android 16 QPR2 SDK 设置：
  https:
  https:
  https:

  和之前版本的主要差异
  如果按 changelog 看，Android 16 相关差异主要集中在 v30.3 到 v30.7：

  - v30.3：开始支持 Android 16 QPR2 引入的新 sepolicy binary format，并支持安装到 vendor_boot。
  - v30.5：改进命令行参数解析，resetprop 支持带 property overrides 的 Android 版本。
  - v30.6：回滚一个可能导致 bootloop 的改动。
  - v30.7：补齐 Android 16 QPR2 的关键适配，尤其是 MagiskInit 和 Zygisk。

  本地还有一个 文章发布版本/Magisk-master，我对比了文件清单和关键版本信息，它同样是 v30.7 / versionCode 30700，不是一个明显的旧版目录。因此这里的“之前版本”更适合按 changelog 里的
  v30.6/v30.5/v30.3 来分析，而不是把旁边那个目录当作旧版。

  底层适配证据
  Zygisk 适配不是简单判断 SDK_INT，而是维护多组 Zygote JNI 签名。Android 16/QPR2 相关新增参数体现在：

  - Magisk-master/native/src/core/zygisk/gen_jni_hooks.py:282：fas_u
  - Magisk-master/native/src/core/zygisk/gen_jni_hooks.py:303：包含 mount_sysprop_overrides
  - Magisk-master/native/src/core/zygisk/gen_jni_hooks.py:497：spec_u
  - Magisk-master/native/src/core/zygisk/gen_jni_hooks.py:516：同样处理 mount_sysprop_overrides

  sepolicy 编译/加载路径也按 split policy、mapping、compat CIL、vendor/product/system_ext/odm 等路径处理：

  - Magisk-master/native/src/sepolicy/policydb.cpp:153：读取并设置 policy version
  - Magisk-master/native/src/sepolicy/policydb.cpp:163：加载 platform mapping CIL
  - Magisk-master/native/src/sepolicy/policydb.cpp:166：加载 compat CIL
  - Magisk-master/native/src/sepolicy/policydb.cpp:192：继续加载 vendor 侧策略

  官方上游 changelog 也和本地内容一致，v30.7 明确写了 Android 16 QPR2 sepolicy 和 Zygisk 支持：
  https:

  判断
  可以认为这个 Magisk-master 已经适配 Android 16，且适配范围包含 Android 16 QPR2：

  - App 层：compileSdk 36.1、targetSdk 36
  - MagiskInit：支持 Android 16 QPR2 sepolicy format
  - MagiskPolicy：从 v30.3 开始支持 Android 16 QPR2 新 sepolicy binary format
  - Zygisk：明确支持 Android 16 QPR2 and higher，并包含对应 JNI 签名变体

  剩下的风险不在“代码是否声明适配”，而在具体机型：Android 16/QPR2 的不同 ROM、内核、vendor_boot 布局、SELinux policy、OEM 改 Zygote 签名，都可能让某些设备还需要实机验证。代码层面
  看，这版已经做了 Android 16 适配。


```

通过对 `Magisk` 仓库的查看和 AI 分析结果，可以确认这个版本已经适配 Android 16，那么后续就可以直接基于这版 `Zygisk` 来做迁移。

在开始改代码之前，先把最新版 `Magisk Zygisk` 的执行流程和关键入口梳理清楚。

阅读 `Magisk` 源码时，我先把前面提到的开源库本地安装并运行起来：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_AE6ZTBBUC2738JV.png)

点一下生成全景图，就开始分析了：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_V2H2HVVGZBYKJW3.png)

生成结果如下：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_9TCDXH3E9Z32X75.png)  
借助这个视图，可以快速定位主要入口。

由于我主要关注的是 `zygisk`，因此后续单独针对这部分代码继续下钻：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_BMFW5DFW29BM83B.png)

使用这个库后，项目入口和主要结构会清晰很多。具体使用过程这里就不展开了。

在这个基础上，我再结合 `codex` 继续分析代码。

在 `/Magisk-master/native/src/core/zygisk` 目录下，打开 codex，给它下指令：

```
我现在要详细了解Magisk中zygisk的执行流程和加载过程，逐行详细解释目录中的代码，并生成到MAGISK-ZYGISK-SOURCE.md


```

![](https://bbs.kanxue.com/upload/attach/202605/703941_YB2HH7KFFU9U8ZR.png)

等待 AI 执行：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_H3C9ZHA4BSH4WH4.png)

结果：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_U54DBYRNXQUYVPT.png)

#### 3.2.0 分析标题与范围

本文基于当前目录 `/native/src/core/zygisk` 中的源码编写，目标是把 Zygisk 的 “如何被加载、如何接管 Zygote、如何加载模块、如何与 magiskd/zygiskd 协作、最后如何清理卸载” 串成一条完整链路，并按文件解释代码。

目录内文件职责：

<table><thead><tr><th>文件</th><th>作用</th></tr></thead><tbody><tr><td><code>zygisk.hpp</code></td><td>Zygisk 内部公共声明、native bridge 常量、日志宏、NativeBridge ABI 结构</td></tr><tr><td><code>entry.cpp</code></td><td>Zygisk 的两个 C++ 入口：NativeBridge 注入口、zygisk companion 进程入口</td></tr><tr><td><code>hook.cpp</code></td><td>bootstrap 核心：PLT hook、JNI hook、真实 native bridge 续载、Zygisk 自卸载</td></tr><tr><td><code>module.hpp</code></td><td>模块 ABI、API 表、specialize 参数对象、<code>ZygiskContext</code>/<code>ZygiskModule</code> 类型定义</td></tr><tr><td><code>module.cpp</code></td><td>模块加载、API 适配、pre/post specialize 调度、FD 清理、denylist/unmount 处理</td></tr><tr><td><code>api.hpp</code></td><td>面向 Zygisk 模块作者的公开 API 头文件</td></tr><tr><td><code>jni_hooks.hpp</code></td><td>由脚本生成的 Zygote JNI native 方法包装器</td></tr><tr><td><code>gen_jni_hooks.py</code></td><td>生成 <code>jni_hooks.hpp</code> 的脚本，维护不同 Android/OEM JNI 签名</td></tr><tr><td><code>daemon.rs</code></td><td>magiskd 侧 Zygisk 状态、属性注入、模块 fd 下发、companion 连接管理</td></tr><tr><td><code>mod.rs</code></td><td>Rust 模块入口，导出 C++ 调用的 <code>exec_companion_entry</code></td></tr></tbody></table>

#### 3.2.1 总执行链路

##### 3.2.1.1 启用阶段：magiskd 写入 native bridge 属性

Zygisk 不是通过修改 Zygote 可执行文件直接进入的，而是借用了 Android native bridge 加载机制。

Rust 侧 `ZygiskState::set_prop()` 修改属性：

```
ro.dalvik.vm.native.bridge


```

写入策略：

1.  如果原属性为空或 `"0"`，写成 `libzygisk.so`。
2.  如果原本已有 native bridge，则写成 `libzygisk.so` 加上原值。

因此 Zygote 启动 native bridge 时会优先加载 `libzygisk.so`。如果系统原本就配置了 native bridge，Zygisk 后续会从属性字符串里剥出原值并重新加载真实 native bridge，避免破坏原系统行为。

##### 3.2.1.2 注入阶段：Zygote 加载 `libzygisk.so`

Android 的 `LoadNativeBridge` 会查找动态库中的导出符号 `NativeBridgeItf`。Zygisk 在 `entry.cpp` 中导出同名符号：

```
extern "C" NativeBridgeCallbacks NativeBridgeItf { ... };


```

当 `isCompatibleWith()` 被调用时，Zygisk 已经进入 Zygote 进程。这里执行：

1.  `zygisk_logging()` 初始化日志。
2.  `hook_entry()` 开始 bootstrap。
3.  返回 `false`，表示自己不是真正要使用的 native bridge。

##### 3.2.1.3 Bootstrap 阶段：安装早期 PLT hook

`hook_entry()` 创建全局 `HookContext`，调用 `HookContext::hook_plt()`。

第一批 PLT hook 挂在两个库上：

<table><thead><tr><th>目标库</th><th>被 hook 的符号</th><th>目的</th></tr></thead><tbody><tr><td><code>libnativebridge.so</code></td><td><code>dlclose</code></td><td>捕获 native bridge 加载流程结束点，拿到 runtime callbacks，并补载真实 native bridge</td></tr><tr><td><code>libandroid_runtime.so</code></td><td><code>strdup</code></td><td>捕获 <code>com.android.internal.os.ZygoteInit</code> 字符串，触发 JNI 方法替换</td></tr><tr><td><code>libandroid_runtime.so</code></td><td><code>fork</code></td><td>Zygisk 提前 fork 后，让原始逻辑看到缓存 pid，避免重复 fork</td></tr><tr><td><code>libandroid_runtime.so</code></td><td><code>unshare</code></td><td>进入新 mount namespace 后执行 denylist unmount</td></tr><tr><td><code>libandroid_runtime.so</code></td><td><code>selinux_android_setcontext</code></td><td>secontext 切换前预取 logd</td></tr><tr><td><code>libandroid_runtime.so</code></td><td><code>__android_log_close</code></td><td>fork/specialize 过程中控制 log fd 关闭</td></tr></tbody></table>

##### 3.2.1.4 接管 Zygote JNI：替换 specialize native 方法

AndroidRuntime 启动 Java 层 ZygoteInit 前会处理类名字符串。`strdup("com.android.internal.os.ZygoteInit")` 触发 Zygisk 的 `new_strdup()`，调用 `hook_zygote_jni()`。

`hook_zygote_jni()` 做三件事：

1.  找到当前 `JavaVM` 和 `JNIEnv`。
2.  用 NativeBridgeRuntimeCallbacks 枚举 `com/android/internal/os/Zygote` 已注册 native 方法。
3.  用 `RegisterNatives` 替换这些目标方法：
    *   `nativeForkAndSpecialize`
    *   `nativeSpecializeAppProcess`
    *   `nativeForkSystemServer`

替换后的函数来自 `jni_hooks.hpp`。它们都是包装器：先构造 `ZygiskContext`，执行 pre 回调，再调用原始 JNI 函数，最后执行 post 回调。

##### 3.2.1.5 App/system_server fork 与模块加载

当 Zygote 要创建 app 或 system_server 时，流程进入 Zygisk 包装器：

1.  构造 `AppSpecializeArgs_v5` 或 `ServerSpecializeArgs_v1`。
2.  构造栈上 `ZygiskContext`，全局 `g_ctx` 指向它。
3.  pre 阶段：
    *   对 fork 型方法，Zygisk 先调用真实 `fork()`。
    *   子进程向 magiskd 查询当前 uid/process 的 flags。
    *   如果进程不在 enforced denylist 且不是 Magisk app，则接收模块 so 的 fd。
    *   使用 `android_dlopen_ext(..., ANDROID_DLEXT_USE_LIBRARY_FD, ...)` 从 fd 加载模块。
    *   调用模块 `zygisk_module_entry`、`onLoad`、`preAppSpecialize` 或 `preServerSpecialize`。
4.  调用原始 Zygote native specialize 函数。
5.  post 阶段：
    *   调用模块 `postAppSpecialize` 或 `postServerSpecialize`。
    *   根据模块选项可能 `dlclose` 模块。
    *   恢复 Zygote JNI hook。
    *   安装自卸载 hook。

##### 3.2.1.6 companion 进程

模块如果需要 root 权限，可以调用 `Api::connectCompanion()`。这不会让 app 进程变成 root，而是：

1.  模块进程向 magiskd 发 `ZygiskRequest::ConnectCompanion`。
2.  magiskd 按 32/64 位选择或启动一个 `zygiskd`。
3.  `zygiskd` 预先加载所有模块 companion entry。
4.  每次模块请求 companion 时，magiskd 把 client fd 转交给对应 `zygiskd`。
5.  `zygiskd` 调用目标模块的 `zygisk_companion_entry(client)`。

##### 3.2.1.7 清理与自卸载

子进程 specialize 完成后，Zygisk 不希望一直留在 app/system_server 进程中。析构 `ZygiskContext` 时：

1.  清掉模块 API 表。
2.  标记 `g_hook->should_unmap = true`。
3.  恢复 Zygote JNI 方法。
4.  在 `libart.so` 上 hook `pthread_attr_destroy`。

后续 JVM 创建线程时会调用 `pthread_attr_destroy`。Zygisk 在这个安全时机：

1.  恢复早期 PLT hook。
2.  用 `musttail` 跳转到 `dlclose(self_handle)`。
3.  避免 `dlclose` 返回到已经被 unmap 的 Zygisk 代码。

#### 3.2.2 `zygisk.hpp` 逐行 / 逐段解释

##### 行 1-4：基础包含

`#pragma once` 防止头文件重复包含。`jni.h` 提供 JNI 类型，`core.hpp` 提供 Magisk core 内部工具、日志、IPC 等声明。

##### 行 6-8：关键常量

`ZYGISKLDR` 是 Zygisk loader 库名：`libzygisk.so`。

`NBPROP` 是 native bridge 属性名：`ro.dalvik.vm.native.bridge`。Rust 与 C++ 两边都使用同一语义。

##### 行 9-19：按 ABI 区分日志前缀

64 位编译时日志前缀为 `zygisk64`，32 位为 `zygisk32`。这样同一设备有双 Zygote 时，logcat 中可以区分是哪一套代码在运行。

##### 行 21-23：极详细日志开关

`ZLOGV` 默认被定义成空操作。取消注释可把 verbose 日志转成 debug 日志，但正常构建不启用，避免刷屏和性能影响。

##### 行 25-26：跨文件函数声明

`hook_entry()` 是 native bridge 回调进入 bootstrap 的入口。

`hookJniNativeMethods()` 是 Zygisk API 暴露给模块的 JNI hook 实现入口，内部会转给当前全局 `HookContext`。

##### 行 28-42：NativeBridge ABI 结构

这两个结构是 Android native bridge 头文件中的 ABI 子集：

`NativeBridgeRuntimeCallbacks` 提供三类能力：

1.  通过 method id 获取 shorty。
2.  查询某个 class 的 native 方法数量。
3.  导出某个 class 的 native 方法表。

Zygisk 依赖它枚举并替换 Zygote native 方法。

`NativeBridgeCallbacks` 是 native bridge 动态库导出的 `NativeBridgeItf` 结构。Zygisk 只填了版本、padding、`isCompatibleWith`。

#### 3.2.3 `entry.cpp` 逐行 / 逐段解释

##### 行 1-9：系统与内部头文件

`android/dlext.h` 用于从 fd 加载 so。`dlfcn.h` 用于 `dlsym`。`poll.h` 用于 companion daemon 等待请求。

##### 行 13-14：companion entry 类型

`comp_entry` 是模块 companion 函数类型：`void (*)(int)`。

`exec_companion_entry` 是 Rust 侧 `mod.rs` 导出的 C ABI 函数。C++ 不直接创建线程，而是把 companion 请求交给 Rust 线程池执行。

##### 行 16-19：`zygiskd` 参数校验

`zygiskd(int socket)` 是 root companion daemon 主循环。开头要求：

1.  当前 uid 必须是 root。
2.  传入 socket fd 必须有效。

不满足直接退出。

##### 行 20-26：设置进程名与日志

64 位进程命名为 `zygiskd64`，32 位为 `zygiskd32`。这和 magiskd 中按 ABI 维护两个 socket 对应。

##### 行 28-49：加载模块 companion entry

从 magiskd 传来的 socket 中接收一组模块 fd。每个 fd：

1.  `fstat` 确认是普通文件。
2.  使用 `android_dlopen_ext("/jit-cache", RTLD_LAZY, &info)` 从 fd 加载。
3.  查找符号 `zygisk_companion_entry`。
4.  不管成功与否，都往 `modules` vector 放入 entry 指针，保证模块 id 与数组下标一致。
5.  关闭原 fd。

`"/jit-cache"` 只是提供给 linker 的伪路径，真实库来自 fd。

##### 行 51-53：启动确认

`write_int(socket, 0)` 告诉 magiskd：zygiskd 已经加载模块 companion 表，可以开始转发请求。

##### 行 54-73：zygiskd 主循环

`poll` 阻塞等待 magiskd 发来 client fd。每次请求：

1.  `recv_fd(socket)` 收到一个 client socket。
2.  从 client 读 `module_id`。
3.  如果 id 合法且该模块有 companion entry，调用 `exec_companion_entry(client, entry)`。
4.  否则关闭 client。

这里的 companion entry 不是直接同步执行，而是进入 Rust 线程池，允许并发处理。

##### 行 76-84：`zygisk_main`

这是 magisk applet 入口，只有内部执行 `magisk zygisk companion <fd>` 时使用。

参数匹配 `argc == 3 && argv[1] == "companion"` 时进入 `zygiskd(parse_int(argv[2]))`。

##### 行 86-96：NativeBridge 注入口

导出符号 `NativeBridgeItf`，让 Android native bridge loader 能找到。

`isCompatibleWith` lambda 是 Zygisk 第一次执行自己代码的关键点：

1.  初始化 Zygisk 日志。
2.  调用 `hook_entry()`。
3.  打印加载成功。
4.  返回 `false`。

返回 `false` 是有意的：Zygisk 要借 native bridge 入口执行注入，而不是最终承担 native bridge 功能。

#### 3.2.4 `hook.cpp` 逐行 / 逐段解释

##### 行 1-15：依赖

`sys/mman.h`、`sys/resource.h`、`dlfcn.h`、`unwind.h` 支撑内存映射扫描、fd 上限、动态符号、栈回溯。

`lsplt.hpp` 是 PLT hook 库。Zygisk bootstrap 和模块 API 的 PLT hook 都依赖它。

`module.hpp` 提供 `ZygiskContext`，`jni_hooks.hpp` 提供生成好的 JNI 包装函数列表。

##### 行 18-90：源码内 bootstrap 流程图

这段注释是官方写在源码里的关键说明。它表达的核心是：

1.  `NativeBridgeItf` 被 native bridge loader 调用。
2.  Zygisk 安装 PLT hook。
3.  `dlclose` hook 捕获 native bridge 加载结束。
4.  `strdup("ZygoteInit")` hook 捕获 JVM 即将进入 ZygoteInit 的时机。
5.  Zygisk 替换 Zygote JNI native 方法。
6.  specialize 后通过 `pthread_attr_destroy` hook 自卸载。

##### 行 91-99：常量与类型别名

`kZygoteInit` 是 Java 层入口类全名。

`kZygote` 是 JNI 查找类名格式：`com/android/internal/os/Zygote`。

`kForkApp`、`kSpecializeApp`、`kForkServer` 是要替换的 native 方法名。

`JNIMethods` 是 `std::span<JNINativeMethod>`，方便对生成数组做统一处理。

`JNIMethodsDyn` 是动态分配的 JNI 方法表和数量。

##### 行 100-119：`HookContext`

`HookContext` 继承 `JniHookDefinitions`，因此直接持有三组生成的 JNI hook 数组：

1.  app fork 方法数组。
2.  app specialize 方法数组。
3.  system_server fork 方法数组。

字段含义：

<table><thead><tr><th>字段</th><th>含义</th></tr></thead><tbody><tr><td><code>plt_backup</code></td><td>记录已经成功注册的 PLT hook，用于后续恢复</td></tr><tr><td><code>runtime_callbacks</code></td><td>NativeBridgeRuntimeCallbacks，枚举 JNI native 方法时使用</td></tr><tr><td><code>self_handle</code></td><td>Zygisk 自身的 dlopen handle，用于最后 dlclose</td></tr><tr><td><code>should_unmap</code></td><td>是否允许自卸载</td></tr></tbody></table>

方法分两类：

1.  bootstrap / 卸载：`hook_plt`、`hook_unloader`、`restore_plt_hook`、`post_native_bridge_load`。
2.  JNI 替换：`hook_zygote_jni`、`restore_zygote_hook`、`hook_jni_methods`。

##### 行 123-136：全局上下文

`g_ctx` 指向当前 specialize 流程中的 `ZygiskContext`。它通常指向栈对象，只在包装 JNI 函数执行期间有效。

`g_hook` 指向长期存在的 `HookContext`，从 native bridge 加载后一直存在到自卸载前。

`get_defs()` 返回 `g_hook`，供 `jni_hooks.hpp` 生成的包装函数获取原始 native 函数指针。

##### 行 140-149：`strdup` hook

`new_strdup(const char *str)` 检查字符串是否等于 `com.android.internal.os.ZygoteInit`。

如果匹配，调用 `g_hook->hook_zygote_jni()`。

随后调用原始 `strdup`。这个 hook 的意义是抓住 AndroidRuntime 即将启动 ZygoteInit 的窗口，此时 JVM 已创建，Zygote native 方法也已经可枚举和替换。

##### 行 151-154：`fork` hook

`new_fork()` 的逻辑：

1.  如果 `g_ctx` 存在并且 `g_ctx->pid >= 0`，返回缓存 pid。
2.  否则调用原始 `fork()`。

Zygisk 在 `fork_pre()` 中会提前执行真正 fork。随后原始 Zygote native 函数内部再调用 `fork()` 时，会被这个 hook 拦截并返回同一个 pid，避免 fork 两次。

##### 行 156-167：`unshare` hook

Zygote specialize 过程中可能调用 `unshare(CLONE_NEWNS)` 创建私有 mount namespace。

如果当前在 Zygisk specialize 流程中，且创建 mount namespace 成功，并且 flags 中有 `DO_REVERT_UNMOUNT`，则调用 `revert_unmount()`。这用于 denylist 或模块强制隐藏时，从目标进程 mount namespace 中卸载 Magisk / 模块痕迹。

##### 行 169-175：`selinux_android_setcontext` hook

secontext 改变后，进程权限和 SELinux 限制会发生变化。这里在切换前调用 `zygisk_get_logd()`，预先拿到 logd fd，避免后续上下文下无法访问或日志管道异常。

##### 行 177-185：`__android_log_close` hook

Zygote fork/specialize 时会关闭日志 fd。Zygisk 根据 `SKIP_CLOSE_LOG_PIPE` 控制是否关闭自己的 log 管道，然后调用原函数。

##### 行 187-194：`dlclose` hook

这是 native bridge bootstrap 的关键：

1.  native bridge loader 在加载结束附近会 `dlclose` loader handle。
2.  Zygisk 把 `libnativebridge.so` 的 `dlclose` PLT 替换成 `new_dlclose`。
3.  第一次触发时 `self_handle` 还为空，调用 `post_native_bridge_load(handle)`。
4.  返回 0，不真正关闭。

`post_native_bridge_load()` 会用这个 handle 记录 Zygisk 自身，并通过栈回溯找到真实 `LoadNativeBridge` 和 runtime callbacks。

##### 行 196-223：`pthread_attr_destroy` hook

Zygisk 不能直接在自己的函数栈上调用 `dlclose(self_handle)`，否则代码被 unmap 后返回地址可能落在已卸载内存里。

解决方式：

1.  后期 hook `libart.so` 的 `pthread_attr_destroy`。
2.  JVM 创建 daemon 线程时会调用它。
3.  Zygisk 在主线程触发时恢复 PLT hook。
4.  如果仍允许卸载，取出 `self_handle`，删除 `g_hook`。
5.  使用 `[[clang::musttail]] return dlclose(self_handle);`，让 `dlclose` 直接返回给 `pthread_attr_destroy` 的调用者。

这样不会返回到 Zygisk 的已卸载代码。

##### 行 229-237：fd 上限与 `ZygiskContext` 构造

`get_fd_max()` 读取 `RLIMIT_NOFILE`，默认准备 32768。

`ZygiskContext` 构造函数保存 `JNIEnv`、specialize 参数、初始化 pid/flags/fd 位图和 mutex，并设置全局 `g_ctx = this`。

##### 行 239-260：`ZygiskContext` 析构

析构时先把 `g_ctx` 清空，因为它指向栈对象。

如果不是子进程，直接返回。父 zygote 不能卸载 Zygisk，否则后续 fork 无法继续被 hook。

子进程中：

1.  关闭 Zygisk logd。
2.  重新初始化 Android logging。
3.  清空每个模块的 API 表，防止 post 后继续使用。
4.  设置 `should_unmap = true`。
5.  恢复 Zygote JNI hook。
6.  安装 `pthread_attr_destroy` 卸载 hook。

##### 行 264-276：`unwind_get_region_start`

封装 `_Unwind_GetRegionStart`。ARM32 下如果 PC 处于 Thumb 模式，要把最低位置 1，保证函数地址匹配真实调用地址。

##### 行 278-344：`find_runtime_callbacks`

目标是从栈回溯上下文中找 `NativeBridgeRuntimeCallbacks *`。

步骤：

1.  扫描 `/proc/self/maps`，找 `libart.so` 的可读写段。
2.  这个 callbacks 对象位于 libart 的 writable 区域。
3.  不同 ABI 参数传递方式不同：
    *   arm64：查 r19-r28。
    *   arm32：查 r4-r10。
    *   x86：从 ebp 相对位置读第二参数。
    *   x86_64：查 rbx/r12-r15 等 callee-saved 寄存器。
    *   riscv：查 callee-saved x8/x9/x18-x27。
4.  找到落在 libart writable 范围内的值，即认为是 callbacks 指针。

这是 Zygisk 能枚举已注册 JNI 方法的基础。

##### 行 346-381：`post_native_bridge_load`

被 `dlclose` hook 触发。

主要流程：

1.  保存 `self_handle`。
2.  使用 `_Unwind_Backtrace` 回溯调用栈。
3.  找到帧所属库是 `libnativebridge.so` 的函数地址，认为它是 `android::LoadNativeBridge`。
4.  在同一帧上下文中调用 `find_runtime_callbacks()`。
5.  若属性 `ro.dalvik.vm.native.bridge` 的值长于 `libzygisk.so`，说明后面拼接了真实 native bridge 名。
6.  调用真实 `LoadNativeBridge(原 native bridge, callbacks)` 补载它。
7.  保存 callbacks 到 `runtime_callbacks`。

##### 行 385-429：注册第一批 PLT hook

`register_hook()` 包装 `lsplt::RegisterHook`，成功后把 `(dev, inode, symbol, old_func)` 存入 `plt_backup`。

`hook_plt()` 扫描 maps 找：

1.  `libandroid_runtime.so` 的 dev/inode。
2.  `libnativebridge.so` 的 dev/inode。

然后注册：

```
libnativebridge.so: dlclose
libandroid_runtime.so: fork
libandroid_runtime.so: unshare
libandroid_runtime.so: selinux_android_setcontext
libandroid_runtime.so: strdup
libandroid_runtime.so: __android_log_close


```

最后 `lsplt::CommitHook()` 执行实际 patch，并删除没有成功备份 old_func 的记录。

##### 行 431-446：注册自卸载 hook

`hook_unloader()` 扫描 maps 找 `libart.so`，注册 `pthread_attr_destroy` hook。这一步发生在子进程 specialize 完成后，而不是最初 bootstrap 时。

##### 行 448-460：恢复 PLT hook

遍历 `plt_backup`，把每个 symbol 替回 old_func。若任何一步失败，设置 `should_unmap = false`，避免在 hook 状态不确定时卸载自己。

##### 行 464-482：读取并注册 JNI native 方法

`get_jni_methods()` 通过 runtime callbacks：

1.  查询 class 的 native 方法数量。
2.  分配数组。
3.  填充方法表。

`register_jni_methods()` 逐个调用 `RegisterNatives`，允许失败。失败时清 exception，并把该方法的 `fnPtr` 置空。

##### 行 484-522：`hook_jni_methods`

这是替换 JNI native 方法的核心。

流程：

1.  读取旧方法表。
2.  对传入的候选 methods 逐个尝试 `RegisterNatives`。
3.  再读取新方法表。
4.  对每个成功替换的方法，根据 name/signature 找到旧方法指针。
5.  把旧方法指针写回 `method.fnPtr`。

注意源码中特别强调：runtime callbacks 返回的 signature 不是标准格式，所以不能只靠字符串提前判断，而是要直接 `RegisterNatives` 试。

##### 行 525-532：按类名 hook JNI 方法

对模块 API 提供的入口。若 callbacks/env/class 无效，则把所有 `fnPtr` 清空。否则 `FindClass` 后调用上面的核心替换函数。

##### 行 534-609：`hook_zygote_jni`

这是 “接管 Zygote specialize” 的关键函数。

步骤：

1.  找 `JNI_GetCreatedJavaVMs`。
    *   先 `dlsym(RTLD_DEFAULT, ...)`。
    *   找不到就扫描并 `dlopen` `libnativehelper.so`。
2.  获取 `JavaVM`。
3.  从 `JavaVM` 获取当前 `JNIEnv`。
4.  `FindClass("com/android/internal/os/Zygote")`。
5.  通过 callbacks 获取 Zygote 已注册 native 方法表。
6.  遍历方法名：
    *   遇到 `nativeForkAndSpecialize`，用 `fork_app_methods` 替换。
    *   遇到 `nativeSpecializeAppProcess`，用 `specialize_app_methods` 替换。
    *   遇到 `nativeForkSystemServer`，用 `fork_server_methods` 替换。
7.  如果任何已存在目标方法替换失败，恢复已替换的方法，并清空对应数组，避免半 hook 状态。

##### 行 611-616：恢复 Zygote JNI 方法

把 `fork_app_methods`、`specialize_app_methods`、`fork_server_methods` 中保存的旧函数指针重新注册回 Zygote 类。

##### 行 620-627：导出给 entry/API 的入口

`hook_entry()` 创建 `g_hook` 并安装早期 PLT hook。

`hookJniNativeMethods()` 是公开 API 的 C++ 实现，转给 `g_hook->hook_jni_methods`。

#### 3.2.5 `module.hpp` 逐行 / 逐段解释

##### 行 1-7：基础声明

包含 regex、list 和公开 API 头 `api.hpp`。模块 API 兼容层需要正则保存 PLT hook 规则。

##### 行 8-29：前置声明与 ABI 版本别名

Zygisk 支持多个 API/ABI 版本。这里用别名表达版本兼容关系：

1.  `AppSpecializeArgs_v2 = AppSpecializeArgs_v1`。
2.  `AppSpecializeArgs_v4 = AppSpecializeArgs_v3`。
3.  `module_abi_v2/v3/v4/v5` 都等同于 v1 布局。
4.  `api_abi_v3 = api_abi_v2`，`api_abi_v5 = api_abi_v4`。

这表示某些版本只扩展语义或参数对象，不改变模块 ABI 函数表布局。

##### 行 31-69：App specialize 参数 v3/v5

`AppSpecializeArgs_v3` 保存 app specialize 的参数引用。必须用引用，因为模块可以修改这些参数，让原始 Zygote specialize 使用修改后的值。

必选字段包括 uid/gid/gids/runtime_flags/rlimits/mount_external/se_info/nice_name/instruction_set/app_data_dir。

可选字段用指针表示，因为不同 Android 版本和 OEM 签名不同：

```
fds_to_ignore
is_child_zygote
is_top_app
pkg_data_info_list
whitelisted_data_info_list
mount_data_dirs
mount_storage_dirs


```

`AppSpecializeArgs_v5` 在 v3 基础上增加 `mount_sysprop_overrides`。

##### 行 71-97：App specialize 参数 v1 兼容层

旧 API v1/v2 没有 `rlimits`，构造函数从 v5 转换到 v1 视图。模块如果声明旧 API，Zygisk 会创建这个兼容对象再调用模块回调。

##### 行 99-113：system_server specialize 参数

`ServerSpecializeArgs_v1` 保存 system_server 参数引用：

```
uid, gid, gids, runtime_flags, permitted_capabilities, effective_capabilities


```

模块可在 pre 阶段修改这些值。

##### 行 115-122：模块 ABI 函数表

`module_abi_v1` 是模块注册给 Zygisk 的 ABI：

<table><thead><tr><th>字段</th><th>含义</th></tr></thead><tbody><tr><td><code>api_version</code></td><td>模块使用的 Zygisk API 版本</td></tr><tr><td><code>impl</code></td><td>模块对象指针</td></tr><tr><td><code>preAppSpecialize</code></td><td>app specialize 前回调</td></tr><tr><td><code>postAppSpecialize</code></td><td>app specialize 后回调</td></tr><tr><td><code>preServerSpecialize</code></td><td>system_server specialize 前回调</td></tr><tr><td><code>postServerSpecialize</code></td><td>system_server specialize 后回调</td></tr></tbody></table>

##### 行 124-131：flags 掩码

`static_assert` 保证内部 flag 与公开 API flag 值一致。

`UNMOUNT_MASK` 表示 “进程在 denylist 且 denylist enforced”。

`PRIVATE_MASK` 是不应该暴露给模块的内部 flag，包括 enforced 状态和 Magisk app 标记。

##### 行 133-168：Zygisk API ABI 表

`api_abi_base` 每个版本都有：

1.  `impl` 指向 `ZygiskModule`。
2.  `registerModule` 用于模块注册自身 ABI。

v1 提供：

```
hookJniNativeMethods
pltHookRegister(regex 版本)
pltHookExclude
pltHookCommit
connectCompanion
setOption


```

v2 增加：

```
getModuleDir
getFlags


```

v4 改造 PLT hook API，从 regex 匹配改为 dev/inode 精确定位，并增加：

```
exemptFd


```

`ApiTable` union 让同一块内存按不同 ABI 版本解释。

##### 行 170-209：`ZygiskModule`

`ZygiskModule` 表示一个已加载到目标进程的 Zygisk 模块。

重要字段：

<table><thead><tr><th>字段</th><th>含义</th></tr></thead><tbody><tr><td><code>id</code></td><td>模块在 magiskd 模块列表中的下标，用于 companion/getModuleDir</td></tr><tr><td><code>unload</code></td><td>模块是否要求 post 后 dlclose</td></tr><tr><td><code>handle</code></td><td>dlopen handle</td></tr><tr><td><code>entry</code></td><td><code>zygisk_module_entry</code> 函数</td></tr><tr><td><code>api</code></td><td>Zygisk 注入给模块的 API 表</td></tr><tr><td><code>mod</code></td><td>模块回填的 ABI 表</td></tr></tbody></table>

重要方法：

1.  `onLoad()` 调用模块入口。
2.  `pre/post App/ServerSpecialize()` 调模块回调。
3.  `valid()` 验证模块 ABI 是否完整。
4.  `connectCompanion()` 请求 root companion。
5.  `getModuleDir()` 从 magiskd 获取模块目录 fd。
6.  `setOption()` 处理模块选项。
7.  `tryUnload()` 根据选项 dlclose 模块。

##### 行 211-221：全局符号与内部 flags

`g_ctx` 在 hook.cpp 定义，供模块 API 访问当前 specialize 上下文。

`old_fork` 是 PLT hook 保存的原始 fork。

内部 flags：

<table><thead><tr><th>flag</th><th>含义</th></tr></thead><tbody><tr><td><code>POST_SPECIALIZE</code></td><td>已进入 post 阶段</td></tr><tr><td><code>APP_FORK_AND_SPECIALIZE</code></td><td>当前是 fork app 流程</td></tr><tr><td><code>APP_SPECIALIZE</code></td><td>当前是 specialize app 流程</td></tr><tr><td><code>SERVER_FORK_AND_SPECIALIZE</code></td><td>当前是 system_server fork 流程</td></tr><tr><td><code>DO_REVERT_UNMOUNT</code></td><td>需要执行 denylist unmount</td></tr><tr><td><code>SKIP_CLOSE_LOG_PIPE</code></td><td>不关闭 Zygisk log pipe</td></tr></tbody></table>

##### 行 227-284：`ZygiskContext`

`ZygiskContext` 是一次 specialize 流程的上下文，生命周期通常是 JNI 包装函数的栈帧。

字段：

<table><thead><tr><th>字段</th><th>含义</th></tr></thead><tbody><tr><td><code>env</code></td><td>当前 JNIEnv</td></tr><tr><td><code>args</code></td><td>app 或 server specialize 参数</td></tr><tr><td><code>process</code></td><td>进程名</td></tr><tr><td><code>modules</code></td><td>当前子进程加载的 Zygisk 模块列表</td></tr><tr><td><code>pid</code></td><td>Zygisk 预 fork 的结果，子进程为 0，父进程为 child pid</td></tr><tr><td><code>flags</code></td><td>当前流程内部状态</td></tr><tr><td><code>info_flags</code></td><td>magiskd 返回的进程状态</td></tr><tr><td><code>allowed_fds</code></td><td>fork 后允许保留的 fd 位图</td></tr><tr><td><code>exempted_fds</code></td><td>模块通过 API 申请豁免关闭的 fd</td></tr><tr><td><code>register_info</code></td><td>旧版 regex PLT hook 注册项</td></tr><tr><td><code>ignore_info</code></td><td>旧版 regex PLT hook 排除项</td></tr></tbody></table>

方法按功能分组：

1.  `run_modules_pre/post()`：加载和执行模块。
2.  `fork/app/server/nativeXXX_pre/post()`：各类 JNI 包装流程。
3.  `get_module_info()`：向 magiskd 查询 flags 和模块 fd。
4.  `sanitize_fds()`：关闭不允许保留的 fd。
5.  `exempt_fd()`：实现模块 fd 豁免 API。
6.  `plt_hook_*()`：兼容旧版 regex PLT hook API。

#### 3.2.6 `module.cpp` 逐行 / 逐段解释

##### 行 14-19：`zygisk_request`

连接 magiskd 的 `RequestCode::ZYGISK`，写入具体 `ZygiskRequest` 子命令，然后返回 socket fd。

后续 `GetInfo`、`ConnectCompanion`、`GetModDir` 都从这里发起。

##### 行 21-27：`ZygiskModule` 构造

保存 id、handle、entry。清零 API 表，设置：

```
api.base.impl = this
api.base.registerModule = RegisterModuleImpl


```

模块入口拿到 API 表后，会调用 `registerModule` 把模块 ABI 回填给 Zygisk。

##### 行 29-69：`RegisterModuleImpl`

模块注册入口。

流程：

1.  空指针检查。
2.  读取模块声明的 `api_version`。
3.  如果版本大于当前 `ZYGISK_API_VERSION`，拒绝。
4.  保存模块 ABI 指针到 `mod`。
5.  按版本填充 API 函数指针。

v1 API：

1.  `hookJniNativeMethods` 指向全局实现。
2.  旧版 `pltHookRegister/Exclude/Commit` 走 `g_ctx` 的 regex 兼容层。
3.  `connectCompanion`、`setOption` 转到当前 `ZygiskModule`。

v2 API：

1.  `getModuleDir`。
2.  `getFlags`。

v4 API：

1.  `pltHookRegister(dev, inode, ...)` 直接走 lsplt。
2.  `exemptFd` 转给 `g_ctx->exempt_fd`。

##### 行 71-85：`valid`

模块必须满足：

1.  已注册 ABI。
2.  API 版本在 1-5。
3.  `impl` 与四个 pre/post 回调指针都不为空。

否则模块会从 `modules` 列表删除。

##### 行 87-98：`connectCompanion`

向 magiskd 发 `ConnectCompanion` 请求。写入当前进程 ABI：

```
64 位写 true
32 位写 false


```

再写模块 id。返回的 fd 是连接到 companion handler 的 socket。

##### 行 100-106：`getModuleDir`

向 magiskd 发 `GetModDir`，写模块 id，接收一个目录 fd。这个 fd 指向模块根目录。

##### 行 108-119：`setOption`

支持两个模块选项：

1.  `FORCE_DENYLIST_UNMOUNT`：设置 `DO_REVERT_UNMOUNT`。
2.  `DLCLOSE_MODULE_LIBRARY`：post 后允许 `dlclose` 模块库。

##### 行 121-127：flags 与模块卸载

`getFlags()` 返回 `info_flags` 去掉内部私有位后的结果。

`tryUnload()` 如果模块设置 unload，就 `dlclose(handle)`。

##### 行 131-160：调用不同 ABI 版本的模块回调

`call_app` 宏负责兼容旧 API：

1.  API v1/v2：构造 `AppSpecializeArgs_v1` 兼容对象。
2.  API v3/v4/v5：直接传 `AppSpecializeArgs_v5` 指针。

server 参数没有这种版本分流，直接调用。

##### 行 164-181：旧版 regex PLT hook 注册

v1/v2 模块用 regex 匹配库路径。Zygisk 保存：

1.  regex。
2.  symbol。
3.  callback。
4.  backup 指针。

exclude 规则也保存 regex 与 symbol。symbol 为空表示排除整个库。

##### 行 183-221：旧版 regex PLT hook 提交

`plt_hook_process_regex()` 扫描 maps：

1.  只处理 offset 为 0、private、可读的映射。
2.  对每条注册 regex 匹配路径。
3.  再检查 exclude 规则。
4.  未被排除就 `lsplt::RegisterHook(map.dev, map.inode, symbol, callback, backup)`。

`plt_hook_commit()` 加锁、处理 regex、释放 regex 对象、清空列表，然后 `lsplt::CommitHook()`。

##### 行 225-241：`get_module_info`

向 magiskd 发 `GetInfo`：

1.  写 uid。
2.  写进程名。
3.  写 ABI 位数。
4.  读取 `info_flags`。
5.  如果 `zygisk_should_load_module(info_flags)` 为真，接收模块 fd 列表。

返回 fd 是因为 system_server 流程后续还要把加载失败的模块 id 写回 magiskd。

##### 行 243-294：`sanitize_fds`

先关闭 Zygisk logd fd。

父进程直接返回。子进程中：

1.  如果支持 `fds_to_ignore`，把模块豁免的 fd 合并到 Zygote 的忽略列表。
2.  把已有 `fds_to_ignore` 中的 fd 标记为 allowed。
3.  创建新 jintArray，把原 fd 与 exempted fd 合并。
4.  扫描 `/proc/self/fd`。
5.  不在 `allowed_fds` 中且不是扫描目录 fd 的，全部关闭。

目的是避免从 Zygote 继承不该进入 app 的 fd，防止崩溃和泄漏。

##### 行 296-307：fd 豁免 API

`exempt_fd(fd)`：

1.  如果已经 post specialize，或者当前跳过关闭 log pipe，返回 true。
2.  如果当前流程不能豁免 fd，返回 false。
3.  否则加入 `exempted_fds`。

`can_exempt_fd()` 要求当前是 `APP_FORK_AND_SPECIALIZE`，且 app 参数有 `fds_to_ignore`。

##### 行 309-346：`fork_pre`/`fork_post`

`fork_pre()` 是 Zygisk 避免第三方模块代码污染父 Zygote 的关键。

流程：

1.  block `SIGCHLD`。
2.  调用原始 `old_fork()`。
3.  父进程直接返回。
4.  子进程扫描 `/proc/self/fd`，记录当前允许保留的 fd。
5.  目录 fd 标记为不允许。
6.  Zygisk logd fd 单独标记为不允许。

之后原始 Zygote 方法再调用 `fork()` 时，会被 `new_fork()` 返回缓存 pid，不会真的再 fork。

`fork_post()` unblock `SIGCHLD`。

##### 行 348-386：`run_modules_pre`

加载并执行模块 pre 阶段。

第一轮遍历 fd：

1.  fd 必须是普通文件。
2.  使用 `android_dlopen_ext` 从 fd 加载模块。
3.  查找 `zygisk_module_entry`。
4.  成功则构造 `ZygiskModule(i, h, e)`。
5.  system_server 中 dlopen 失败会记录 warning，并把 fd 标成 -1，稍后上报失败模块。

第二轮：

1.  调用 `onLoad(env)`。
2.  模块通过 API 表注册自身 ABI。
3.  `valid()` 通过则保留，否则删除。

第三轮：

1.  app 流程调用 `preAppSpecialize(args.app)`。
2.  system_server 流程调用 `preServerSpecialize(args.server)`。

##### 行 388-398：`run_modules_post`

设置 `POST_SPECIALIZE`，再对每个模块：

1.  app 调 `postAppSpecialize`。
2.  system_server 调 `postServerSpecialize`。
3.  根据模块选项 `tryUnload()`。

##### 行 400-421：app specialize 共享 pre/post

`app_specialize_pre()`：

1.  设置 `APP_SPECIALIZE`。
2.  向 magiskd 查询进程 flags 和模块 fd。
3.  如果进程在 enforced denylist，设置 `DO_REVERT_UNMOUNT`，不加载模块。
4.  否则加载并运行模块 pre。

`app_specialize_post()`：

1.  调模块 post。
2.  如果当前进程是 Magisk app，设置环境变量 `ZYGISK_ENABLED=1`。
3.  释放 `nice_name` 的 UTF chars。

##### 行 423-445：system_server pre/post

`server_specialize_pre()`：

1.  用 uid 1000 和进程名 `system_server` 查询 magiskd。
2.  如果没有模块，写回 0。
3.  如果有模块，加载并运行 pre。
4.  收集加载失败的模块 id，写回 magiskd。magiskd 会在对应模块目录下创建 `zygisk/unloaded` 标记。

`server_specialize_post()` 调 `run_modules_post()`。

##### 行 449-460：`nativeSpecializeAppProcess` 包装流程

这是 Android Q 以后部分流程使用的 “不 fork，只 specialize 当前进程” 的路径。

pre：

1.  从 `nice_name` 获取进程名。
2.  设置 `SKIP_CLOSE_LOG_PIPE`。
3.  调 app pre。

post：

1.  调 app post。

##### 行 462-480：`nativeForkSystemServer` 包装流程

pre：

1.  设置 `SERVER_FORK_AND_SPECIALIZE`。
2.  进程名固定为 `system_server`。
3.  调 `fork_pre()`。
4.  子进程执行 server pre。
5.  清理 fd。

post：

1.  子进程执行 server post。
2.  父子都执行 `fork_post()` 解锁 SIGCHLD。

##### 行 482-500：`nativeForkAndSpecialize` 包装流程

pre：

1.  从 `nice_name` 获取 app 进程名。
2.  设置 `APP_FORK_AND_SPECIALIZE`。
3.  先真实 fork。
4.  子进程执行 app pre。
5.  清理 fd。

post：

1.  子进程执行 app post。
2.  父子都执行 `fork_post()`。

#### 3.2.7 `api.hpp` 逐行 / 逐段解释

##### 行 1-21：版权、用途和警告

这是面向模块开发者的公开 API 头。源码提示不要直接修改，并建议模块开发使用发布版 sample 仓库中的头文件。

##### 行 24-27：JNI 与 API 版本

包含 `jni.h`，定义当前 API 版本 `ZYGISK_API_VERSION 5`。

##### 行 28-100：公开说明

注释解释了 Zygisk 的模型：

1.  Android app 进程从 Zygote fork。
2.  fork 后执行 specialize，把进程放入应用沙箱。
3.  Zygisk 允许模块在 app/system_server specialize 前后运行代码。
4.  模块代码运行在目标 app/system_server 进程，不运行在长期的 Zygote daemon 中。
5.  需要 root 能力时，使用 companion handler。

##### 行 102-141：`zygisk::ModuleBase`

模块作者继承此类，实现：

<table><thead><tr><th>方法</th><th>调用时机</th></tr></thead><tbody><tr><td><code>onLoad(Api*, JNIEnv*)</code></td><td>模块 so 被加载后立即调用</td></tr><tr><td><code>preAppSpecialize(AppSpecializeArgs*)</code></td><td>app specialize 前，仍接近 Zygote 权限</td></tr><tr><td><code>postAppSpecialize(const AppSpecializeArgs*)</code></td><td>app specialize 后，处于 app 沙箱</td></tr><tr><td><code>preServerSpecialize(ServerSpecializeArgs*)</code></td><td>system_server specialize 前</td></tr><tr><td><code>postServerSpecialize(const ServerSpecializeArgs*)</code></td><td>system_server specialize 后</td></tr></tbody></table>

##### 行 143-178：公开参数对象

`AppSpecializeArgs` 暴露 app specialize 参数。必选字段保证所有支持 Android 版本存在；可选字段要先判空。

`ServerSpecializeArgs` 暴露 system_server 参数。

字段都是引用，模块可在 pre 阶段修改它们，影响后续原始 Zygote specialize 调用。

##### 行 185-200：模块选项

`FORCE_DENYLIST_UNMOUNT`：强制对当前进程执行 Magisk / 模块文件 unmount。

`DLCLOSE_MODULE_LIBRARY`：post 后卸载模块库。注释明确警告：如果模块 hook 了进程函数，不应启用该选项，否则回调地址会变成悬空指针。

##### 行 202-209：公开状态 flags

`PROCESS_GRANTED_ROOT` 表示当前进程 uid 已被授权 root。

`PROCESS_ON_DENYLIST` 表示当前进程在 denylist 中。

内部 flag 如 denylist enforced、Magisk app 不直接暴露给模块。

##### 行 211-290：`zygisk::Api`

模块可用 API：

1.  `connectCompanion()`：连接 root companion 进程。
2.  `getModuleDir()`：获取模块根目录 fd。
3.  `setOption()`：设置模块选项。
4.  `getFlags()`：获取当前进程状态。
5.  `exemptFd()`：让指定 fd 不被 Zygote 自动关闭。
6.  `hookJniNativeMethods()`：替换某个 Java class 已注册 native 方法。
7.  `pltHookRegister()`：按 dev/inode/symbol 注册 PLT hook。
8.  `pltHookCommit()`：提交 PLT hook。

这些 API 在 post 后会失效，因为 Zygisk 会卸载并清空 API 表。

##### 行 292-310：注册宏

`REGISTER_ZYGISK_MODULE(clazz)` 导出 `zygisk_module_entry`，Zygisk 加载模块后查找这个符号。

`REGISTER_ZYGISK_COMPANION(func)` 导出 `zygisk_companion_entry`，zygiskd 加载模块 companion 后查找这个符号。

##### 行 317-360：内部 ABI 实现

`internal::module_abi` 是模块侧构造的 ABI 表，保存 API 版本、模块对象和四个回调 trampoline。

`internal::api_table` 是 Zygisk 传入模块的函数表，布局对应 `module.hpp` 中的 `api_abi_v4`。

`entry_impl<T>()`：

1.  创建静态 `Api`。
2.  绑定传入 table。
3.  创建静态模块对象 `T module`。
4.  创建静态 ABI。
5.  调 `table->registerModule` 注册。
6.  成功后调用模块 `onLoad(&api, env)`。

##### 行 364-387：Api inline 方法

这些方法只是薄封装：检查函数指针是否存在，再通过 `tbl` 调用 Zygisk 内部实现。

##### 行 391-398：默认导出声明

声明两个 C 符号：

```
zygisk_module_entry
zygisk_companion_entry


```

模块可以只注册其中一个，也可以两个都注册。

#### 3.2.8 `jni_hooks.hpp` 逐行 / 逐段解释

这是 `gen_jni_hooks.py` 生成的文件，不建议手写修改。它的职责是覆盖不同 Android 版本和厂商修改后的 Zygote native 方法签名。

##### 行 1-7：生成声明

声明 `JniHookDefinitions` 和 `get_defs()`。`get_defs()` 在 `hook.cpp` 中返回 `g_hook`，因此生成代码可以访问保存旧函数指针的数组。

##### 行 9-230：`fork_app_methods`

数组包含 12 个 `nativeForkAndSpecialize` 签名：

1.  AOSP Android L/O/P/R/U/B 等版本变化。
2.  Samsung M/N/O/P 变体。
3.  Nubia U 变体。

每个元素结构相同：

1.  `name` 固定为 `nativeForkAndSpecialize`。
2.  `signature` 是对应 Android/OEM 的 JNI 签名。
3.  `fnPtr` 是一个静态 lambda。

lambda 统一流程：

1.  根据参数构造 `AppSpecializeArgs_v5 args(...)`。
2.  对当前签名存在的可选参数，把 `args.xxx = &xxx`。
3.  构造 `ZygiskContext ctx(env, &args)`。
4.  调 `ctx.nativeForkAndSpecialize_pre()`。
5.  通过 `get_defs()->fork_app_methods[index].fnPtr` 调回原始 JNI native 函数。
6.  调 `ctx.nativeForkAndSpecialize_post()`。
7.  返回 `ctx.pid`。

注意第 5 步中 `fnPtr` 在 hook 完成后已经被替换成原始函数指针，这是 `HookContext::hook_jni_methods()` 做的。

##### 行 232-361：`specialize_app_methods`

数组包含 7 个 `nativeSpecializeAppProcess` 签名。

这个路径返回 `void`，因为它不 fork，只 specialize 当前进程。

lambda 流程与 fork app 类似，但调用：

```
ctx.nativeSpecializeAppProcess_pre()
原始 nativeSpecializeAppProcess
ctx.nativeSpecializeAppProcess_post()


```

##### 行 363-394：`fork_server_methods`

数组包含 2 个 `nativeForkSystemServer` 签名：

1.  AOSP 标准签名。
2.  Samsung Q 变体。

lambda 流程：

1.  构造 `ServerSpecializeArgs_v1`。
2.  构造 `ZygiskContext`。
3.  `ctx.nativeForkSystemServer_pre()`。
4.  调原始 `nativeForkSystemServer`。
5.  `ctx.nativeForkSystemServer_post()`。
6.  返回 `ctx.pid`。

#### 3.2.9 `gen_jni_hooks.py` 逐行 / 逐段解释

##### 行 3-19：JNI 类型建模

`JType` 保存 C++ 类型名和 JNI 签名字符。

`JArray` 根据元素类型生成数组类型：

1.  primitive 数组如 `jintArray`。
2.  非 primitive 数组统一为 `jobjectArray`。

##### 行 21-38：参数建模

`Argument` 保存参数名、类型、是否要写入 `AppSpecializeArgs` 的可选字段。

`Anon` 表示 Zygisk 不关心但签名中必须保留的 OEM 参数，自动命名为 `_0`、`_1` 等。

##### 行 40-69：JNI 方法建模

`Return` 保存返回值表达式和返回类型。

`JNIMethod` 能生成：

1.  C++ 参数列表。
2.  C++ 函数指针类型。
3.  lambda 签名。
4.  JNI signature 字符串。

##### 行 71-139：三类 hook 生成器

`JNIHook` 是抽象基类。

`ForkApp` 生成 `nativeForkAndSpecialize` 包装体：

1.  初始化 args。
2.  设置可选参数指针。
3.  构造 `ZygiskContext`。
4.  调 pre。
5.  调原始函数。
6.  调 post。
7.  返回 `ctx.pid`。

`SpecializeApp` 改目标名为 `nativeSpecializeAppProcess`，返回 void。

`ForkServer` 改目标名为 `nativeForkSystemServer`，初始化 server args。

##### 行 140-181：公共参数定义

定义 AOSP 与 OEM 签名中会出现的参数对象，例如 uid/gid/gids/runtime_flags、`fds_to_ignore`、`is_child_zygote`、`mount_sysprop_overrides`、server capabilities 等。

带 `set_arg=True` 的参数会被写入 `AppSpecializeArgs_v5` 可选字段。

##### 行 183-612：具体签名列表

逐个定义不同 Android 版本 / OEM 的方法签名：

1.  `fas_*`：fork app。
2.  `spec_*`：specialize app。
3.  `server_*`：fork system_server。

Samsung、Nubia、XR 等变体通过 `Anon` 或额外参数保持 ABI 匹配，但 Zygisk 只提取自己关心的参数。

##### 行 615-631：生成数组代码

`gen_jni_def(field, methods)` 生成一个 `std::array<JNINativeMethod, N>`。

每个元素包含：

1.  方法名。
2.  JNI 签名。
3.  包装 lambda。

##### 行 634-669：写出 `jni_hooks.hpp`

脚本打开 `jni_hooks.hpp` 并写入：

1.  文件头。
2.  前置声明。
3.  `JniHookDefinitions` struct。
4.  三组方法数组。

#### 3.2.10 `daemon.rs` 逐行 / 逐段解释

##### 行 1-17：依赖

引入模块根目录常量、MagiskD、FFI 枚举、resetprop、Unix socket 扩展、fd 工具、fork 工具、日志工具等。

##### 行 18-25：常量与加载判断

`NBPROP` 与 C++ 一致，是 native bridge 属性。

`ZYGISKLDR` 与 C++ 一致，是 `libzygisk.so`。

`UNMOUNT_MASK` 表示 denylist enforced 且当前进程在 denylist。

`zygisk_should_load_module(flags)` 返回 true 的条件：

1.  不是 enforced denylist 命中。
2.  不是 Magisk app。

##### 行 27-59：`exec_zygiskd`

启动 companion daemon。

关键点：

1.  清除 remote socket 的 close-on-exec，让 fd 在 exec 后仍然有效。
2.  64 位 magiskd 中，如果目标是 32 位 companion，执行 `magisk32`；否则执行 `magisk`。
3.  构造执行路径：`get_magisk_tmp()/magisk` 或 `magisk32`。
4.  执行参数：

```
magisk zygisk companion <fd>


```

这个参数最终进入 C++ `zygisk_main()`。

##### 行 61-67：`ZygiskState`

字段：

<table><thead><tr><th>字段</th><th>含义</th></tr></thead><tbody><tr><td><code>lib_name</code></td><td>当前写入 native bridge 属性的值，用于恢复原属性</td></tr><tr><td><code>sockets</code></td><td>32 位和 64 位 zygiskd 的连接 socket</td></tr><tr><td><code>start_count</code></td><td>zygote crash 计数，用于回滚</td></tr></tbody></table>

##### 行 69-108：`connect_zygiskd`

处理模块 `connectCompanion()` 请求。

流程：

1.  从 client 读 ABI 位数。
2.  选择 32 位或 64 位 zygiskd socket。
3.  如果已有 socket，用 `poll` 检查是否仍有效。
4.  有效则直接把 client fd 发给 zygiskd。
5.  无效或不存在，则创建 socket pair。
6.  fork 子进程执行 `exec_zygiskd`。
7.  把当前 ABI 的所有模块 fd 发送给新 zygiskd。
8.  等 zygiskd ack。
9.  把当前 client fd 发给 zygiskd。
10.  保存 local socket 供复用。

##### 行 110-127：`reset`

Zygisk 状态重置：

1.  restore 为 true 时重置 crash 计数并恢复属性。
2.  restore 为 false 时清空 companion sockets，增加 crash 计数。
3.  crash 超过 3 次，认为 zygote 崩溃过多，回滚 native bridge 属性。

##### 行 129-146：`set_prop`

如果 `lib_name` 不为空，说明已设置过，直接返回。

否则读取原 native bridge 属性：

1.  空或 `"0"`：`lib_name = "libzygisk.so"`。
2.  非空：`lib_name = "libzygisk.so" + 原值`。

写回 `ro.dalvik.vm.native.bridge`。

如果 Huawei Maple 开启，则设置 `ro.maple.enable=0`，因为 Maple 特殊 Zygote 可能绕开 native bridge 创建 system_server。

##### 行 148-155：`restore_prop`

如果 `lib_name` 比 `libzygisk.so` 长，后半段就是原 native bridge；否则恢复为 `"0"`。

写回属性并清空 `lib_name`。

##### 行 158-176：`MagiskD::zygisk_handler`

magiskd 收到 `RequestCode::ZYGISK` 后进入这里。

读取 `ZygiskRequest`，分发：

1.  `GetInfo`：查询进程 flags 和模块 fd。
2.  `ConnectCompanion`：启动 / 复用 zygiskd，转发 client fd。
3.  `GetModDir`：发送模块目录 fd。

##### 行 178-189：`get_module_fds`

从模块列表取每个模块对应 ABI 的 zygisk so fd：

1.  64 位取 `m.z64`。
2.  32 位取 `m.z32`。
3.  无效 fd 用 `STDOUT_FILENO` 占位，因为通过 socket 发送 fd 时必须是有效 fd；magiskd 中 stdout 始终是 `/dev/null`，zygiskd / 目标进程会把它视为无效模块。

##### 行 191-240：`get_process_info`

处理目标进程查询。

读取：

1.  uid。
2.  process 名。
3.  ABI 位数。

计算 flags：

1.  `update_deny_flags` 写入 denylist/enforced 信息。
2.  如果 uid 是当前用户的 Magisk manager uid，设置 `ProcessIsMagiskApp`。
3.  如果 uid 已授权 root，设置 `ProcessGrantedRoot`。

然后：

1.  先写 flags 给客户端。
2.  如果允许加载模块，发送模块 fd。
3.  如果不是 system_server，结束。
4.  如果是 system_server，继续读取 C++ 写回的 failed module ids。
5.  对失败模块创建 `zygisk/unloaded` 标记文件。

##### 行 242-257：`get_mod_dir`

读取模块 id，查模块列表，打开模块根目录并把 fd 发送给客户端。

##### 行 260-265：`zygisk_enabled`

FFI 辅助方法，返回 atomic `zygisk_enabled` 状态。

#### 3.2.11 `mod.rs` 逐行 / 逐段解释

##### 行 1-6：模块组织与导出

引入 `daemon` 子模块，并重新导出：

```
ZygiskState
zygisk_should_load_module


```

##### 行 8-28：`exec_companion_entry`

这是给 C++ `entry.cpp` 调用的 C ABI 函数。

参数：

1.  `client`：目标进程与 companion handler 通信的 socket fd。
2.  `companion_handler`：模块导出的 `zygisk_companion_entry`。

执行流程：

1.  把任务提交到 `ThreadPool::exec_task`。
2.  调用前用 `fd_get_attr(client)` 记录 fd 对应文件的 dev/inode。
3.  执行模块 companion handler。
4.  handler 返回后再次检查 fd。
5.  如果 fd 仍是同一个文件，关闭它。

这样做是为了避免模块 handler 自己关闭 fd 后，系统复用了相同 fd 数字，Zygisk 又误关了别的文件。

#### 3.2.12 关键时序图

##### 3.2.12.1 Zygote 启动注入

```
magiskd set_prop(ro.dalvik.vm.native.bridge)
    ↓
zygote LoadNativeBridge("libzygisk.so...")
    ↓
NativeBridgeItf.isCompatibleWith()
    ↓
hook_entry()
    ↓
HookContext::hook_plt()
    ↓
dlclose hook -> post_native_bridge_load()
    ↓
保存 callbacks，必要时补载真实 native bridge
    ↓
strdup("com.android.internal.os.ZygoteInit")
    ↓
hook_zygote_jni()
    ↓
替换 Zygote native specialize 方法


```

##### 3.2.12.2 app fork 流程

```
Zygote.nativeForkAndSpecialize(...)
    ↓
Zygisk generated lambda
    ↓
ZygiskContext ctx
    ↓
ctx.nativeForkAndSpecialize_pre()
    ↓
old_fork() 先 fork
    ↓
子进程 get_module_info(uid, process)
    ↓
magiskd 返回 flags + module fds
    ↓
子进程 dlopen 模块
    ↓
模块 onLoad + preAppSpecialize
    ↓
调用原始 nativeForkAndSpecialize
    ↓
fork hook 返回缓存 pid，避免二次 fork
    ↓
原始 specialize 完成
    ↓
ctx.nativeForkAndSpecialize_post()
    ↓
模块 postAppSpecialize
    ↓
ctx 析构，恢复 JNI hook，准备自卸载


```

##### 3.2.12.3 system_server fork 流程

```
Zygote.nativeForkSystemServer(...)
    ↓
Zygisk generated lambda
    ↓
ctx.nativeForkSystemServer_pre()
    ↓
old_fork()
    ↓
子进程 get_module_info(1000, "system_server")
    ↓
加载模块并执行 preServerSpecialize
    ↓
把加载失败模块 id 写回 magiskd
    ↓
调用原始 nativeForkSystemServer
    ↓
ctx.nativeForkSystemServer_post()
    ↓
模块 postServerSpecialize


```

##### 3.2.12.4 companion 流程

```
模块 Api::connectCompanion()
    ↓
ZygiskModule::connectCompanion()
    ↓
magiskd zygisk_handler ConnectCompanion
    ↓
connect_zygiskd()
    ↓
启动或复用 zygiskd32/zygiskd64
    ↓
zygiskd 收 module fds 并查 zygisk_companion_entry
    ↓
magiskd 把 client fd 发送给 zygiskd
    ↓
zygiskd 读 module_id
    ↓
exec_companion_entry(client, module_entry)
    ↓
Rust ThreadPool 调模块 companion_handler(client)


```

#### 3.2.13 几个容易误解的点

##### 3.2.13.1 模块不是加载在长期 Zygote daemon 中

公开 API 注释也强调：模块只在 fork 后的 app/system_server 进程中加载。Zygisk 先 fork，再在子进程加载第三方模块，避免模块代码污染父 Zygote。

##### 3.2.13.2 `isCompatibleWith()` 返回 false 不代表注入失败

Zygisk 不是要成为真正 native bridge，它只是借这个入口完成 bootstrap。真正 native bridge 如存在，会在 `post_native_bridge_load()` 中补载。

##### 3.2.13.3 为什么要 hook `fork`

Zygisk 需要在第三方模块加载前先真实 fork，确保模块只进入子进程。但原始 Zygote native 函数内部仍会调用 fork。`new_fork()` 返回缓存 pid，使原始逻辑继续按正常路径执行但不再创建新进程。

##### 3.2.13.4 为什么要枚举 JNI 方法再 RegisterNatives

不同 Android 版本和厂商修改了 `Zygote` native 方法签名。Zygisk 不能假设固定签名，所以生成大量候选签名，逐个 `RegisterNatives` 尝试，成功后再反查旧函数指针。

##### 3.2.13.5 为什么要自卸载

Zygisk 的职责是在 specialize 前后提供模块回调。完成后继续常驻 app 进程会增加检测面、内存占用和 hook 风险。因此子进程完成后恢复 hook 并 `dlclose` 自己。

#### 3.2.14 关键函数索引

<table><thead><tr><th>函数 / 符号</th><th>文件</th><th>作用</th></tr></thead><tbody><tr><td><code>NativeBridgeItf</code></td><td><code>entry.cpp</code></td><td>native bridge 注入入口</td></tr><tr><td><code>zygisk_main</code></td><td><code>entry.cpp</code></td><td>companion applet 入口</td></tr><tr><td><code>zygiskd</code></td><td><code>entry.cpp</code></td><td>root companion daemon 主循环</td></tr><tr><td><code>hook_entry</code></td><td><code>hook.cpp</code></td><td>初始化 <code>HookContext</code> 并安装 bootstrap PLT hook</td></tr><tr><td><code>HookContext::hook_plt</code></td><td><code>hook.cpp</code></td><td>hook <code>dlclose/fork/unshare/strdup/...</code></td></tr><tr><td><code>HookContext::post_native_bridge_load</code></td><td><code>hook.cpp</code></td><td>找 callbacks 并补载真实 native bridge</td></tr><tr><td><code>HookContext::hook_zygote_jni</code></td><td><code>hook.cpp</code></td><td>替换 Zygote native specialize 方法</td></tr><tr><td><code>ZygiskContext::fork_pre</code></td><td><code>module.cpp</code></td><td>提前 fork 并记录 fd</td></tr><tr><td><code>ZygiskContext::run_modules_pre</code></td><td><code>module.cpp</code></td><td>dlopen 模块并调用 onLoad/pre</td></tr><tr><td><code>ZygiskContext::run_modules_post</code></td><td><code>module.cpp</code></td><td>调用 post 并按需卸载模块</td></tr><tr><td><code>ZygiskModule::RegisterModuleImpl</code></td><td><code>module.cpp</code></td><td>模块 ABI 注册和 API 表填充</td></tr><tr><td><code>ZygiskState::set_prop</code></td><td><code>daemon.rs</code></td><td>写入 native bridge 属性</td></tr><tr><td><code>MagiskD::get_process_info</code></td><td><code>daemon.rs</code></td><td>返回进程 flags 和模块 fd</td></tr><tr><td><code>ZygiskState::connect_zygiskd</code></td><td><code>daemon.rs</code></td><td>启动 / 复用 companion daemon</td></tr><tr><td><code>exec_companion_entry</code></td><td><code>mod.rs</code></td><td>在线程池中执行 companion handler</td></tr></tbody></table>

#### 3.2.15 用一句话概括

Zygisk 的核心是：magiskd 通过 native bridge 属性让 Zygote 加载 `libzygisk.so`，Zygisk 在 native bridge 初始化窗口安装 PLT hook，再在 ZygoteInit 前替换 Zygote 的 JNI specialize 方法；真正创建 app/system_server 时，它先 fork 到子进程，向 magiskd 获取模块 fd，加载模块执行 pre/post 回调，完成后恢复 hook 并从目标进程自卸载。

源码流程梳理清楚之后，就可以开始真正移植了。

指令：

```
当前目录中有Magisk-master和r0zygisk两个项目。Magisk-master中已存在适配 Android 16的zygisk，我现在需要移植这个版本到r0zygisk，用作 KernelSU / Magisk 环境下的 Zygisk 替代实现。最终要形成打包文件：r0zygisk-v1.0.0-1-release.zip


```

AI 执行过程：

1.  环境检测

```
    1. 先检查 `Magisk-master` 是否体现 Android 16 支持。
       - `app/build-logic/src/main/java/Setup.kt` 中 `compileSdk` / `targetSdk` 使用 API 36。
       - `scripts/avd.sh` 中 `Baklava) api=36 ;;`，说明脚本知道 Android 16 代号和 API 级别。
       - `native/src/core/zygisk/gen_jni_hooks.py` 和生成的 `jni_hooks.hpp` 中有新的 Zygote JNI 签名候选，例如 `nativeForkAndSpecialize_b`。
    
    2. 再检查 `r0zygisk` 的结构。
       - `r0zygisk` 不是普通 Zygisk 模块，而是独立 Zygisk 实现，包含自己的 `loader`、`zygiskd`、ptrace 注入逻辑和模块分发逻辑。
       - 因此不能直接把 Magisk 的 `hook.cpp/module.cpp` 整套替换进去，否则会破坏 r0zygisk 自己的架构。
    
    3. 采用保守迁移策略。
       - 同步公开 Zygisk API 到 Magisk 最新 API v5。
       - 保留 r0zygisk 自己的 daemon / ptrace / 模块加载流程。
       - 只把必要的 Zygote JNI 签名和参数结构迁移进 r0zygisk 的现有 hook 生成器。
    
    4. 修改版本命名。
       - 按要求改为固定输出 `r0zygisk-v1.0.0-1-release.zip`。


```

#### 1. 同步公开 API 到 v5

文件：

```
loader/src/include/api.hpp


```

改动：

*   使用 `Magisk-master/native/src/core/zygisk/api.hpp` 覆盖 r0zygisk 旧版公开 API 头文件。
*   `ZYGISK_API_VERSION` 从旧版本升级为：

```
#define ZYGISK_API_VERSION 5


```

意义：

*   新编译的 Zygisk 模块会按 API v5 注册。
*   r0zygisk loader 需要能识别并加载 API v5 模块。

#### 2. 内部参数结构升级为 `AppSpecializeArgs_v5`

文件：

```
loader/src/injector/module.hpp


```

改动：

*   增加 `AppSpecializeArgs_v5`。
*   `AppSpecializeArgs_v5` 继承 `AppSpecializeArgs_v3`，并单独持有：

```
jboolean *mount_sysprop_overrides = nullptr;


```

*   增加 API v5 兼容别名：

```
using module_abi_v5 = module_abi_v1;
using api_abi_v5 = api_abi_v4;


```

*   `call_app` 增加 `case 5`，让 API v5 模块按新版参数传入。

意义：

*   兼容 Magisk 最新 Zygisk API v5。
*   保留对旧 API v1-v4 模块的兼容。

#### 3. Hook 上下文改用 `AppSpecializeArgs_v5`

文件：

```
loader/src/injector/hook.cpp


```

改动：

*   `ZygiskContext` 中 app 参数指针从：

```
AppSpecializeArgs_v3 *app;


```

改为：

```
AppSpecializeArgs_v5 *app;


```

*   `ZygiskModule::valid()` 接受 API v5：

```
case 5:


```

意义：

*   r0zygisk 的模块加载逻辑可以接受 v5 模块。
*   Zygote JNI wrapper 传入的是新版 specialize 参数对象。

#### 4. 更新 JNI hook 生成器

文件：

```
loader/src/injector/gen_jni_hooks.py


```

改动：

*   wrapper 初始化参数从 `AppSpecializeArgs_v3` 改为 `AppSpecializeArgs_v5`。
*   增加 Magisk 最新 Zygisk 中已有的新参数：

```
is_perception_app
use_fifo_ui


```

*   增加新的 Zygote JNI 签名候选：

```
fas_b
fas_nubia_u
spec_xr_u
spec_nubia_u


```

*   重新生成：

```
loader/src/injector/jni_hooks.hpp


```

新增生成结果中包含：

```
nativeForkAndSpecialize_b
nativeForkAndSpecialize_nubia_u
nativeSpecializeAppProcess_xr_u
nativeSpecializeAppProcess_nubia_u


```

意义：

*   `nativeForkAndSpecialize_b` 是 Android 16 / Baklava 相关适配的重要线索。
*   新增的 OEM / XR 签名提高了对新系统和厂商 ROM 的兼容范围。

#### 1. 固定版本名和版本号

文件：

```
build.gradle.kts


```

改动前：

```
val verName by extra("v4-0.9.1.1")
val verCode by extra(gitCommitCount)


```

改动后：

```
val verName by extra("v1.0.0")
val verCode by extra(1)


```

意义：

*   项目从自己的第一版开始。
*   `versionCode` 固定为 `1`。

#### 2. 去掉 zip 文件名中的 commit hash

文件：

```
module/build.gradle.kts


```

改动前：

```
val zipFileName = "$moduleName-$verName-$verCode-$commitHash-$buildTypeLowered.zip".replace(' ', '-')


```

改动后：

```
val zipFileName = "$moduleName-$verName-$verCode-$buildTypeLowered.zip".replace(' ', '-')


```

意义：

*   不再因为目录不是 git 仓库而输出 `unknown`。
*   最终文件名符合要求。

#### 3. 去掉 `module.prop` 版本显示中的 commit hash

文件：

```
module/build.gradle.kts


```

改动前：

```
"versionName" to "$verName ($verCode-$commitHash-$variantLowered)",


```

改动后：

```
"versionName" to "$verName ($verCode-$variantLowered)",


```

生成结果：

```
version=v1.0.0 (1-release)
versionCode=1


```

执行过的验证命令：

```
./gradlew :loader:assembleRelease --no-daemon --console=plain


```

结果：

```
BUILD SUCCESSFUL


```

执行完整打包：

```
./gradlew zipRelease --no-daemon --console=plain


```

结果：

```
BUILD SUCCESSFUL


```

最终新产物：

```
module/build/outputs/release/r0zygisk-v1.0.0-1-release.zip


```

项目已经开源，发布页在 GitHub Releases：

[r0zygisk Releases](https://bbs.kanxue.com/elink@a4eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6X3P5i4u0D9L8%4k6W2i4K6u0r3M7U0m8*7P5h3N6A6M7$3E0Q4x3V1k6J5k6h3I4W2j5i4y4W2M7H3%60.%60.)

```
build.gradle.kts
module/build.gradle.kts
loader/src/include/api.hpp
loader/src/injector/module.hpp
loader/src/injector/hook.cpp
loader/src/injector/gen_jni_hooks.py
loader/src/injector/jni_hooks.hpp


```

当前源码已经完成：

*   Zygisk API v5 迁移。
*   Android 16 / Baklava 相关 JNI 签名候选加入。
*   r0zygisk 自有 loader / zygiskd / ptrace 架构保留。
*   输出文件名改为 `r0zygisk-v1.0.0-1-release.zip`。
*   完整打包通过。

在 Android 14 的 SukiSU 环境刷入后，我又额外测试了 LSPosed 模块，确认加载正常。

到这里，移植主体已经基本完成，下一步开始补 WebUI，这部分同样结合 AI 一起推进。

指令：

```
r0zygisk-v1.0.0-1-release.zip 刷入并测试后，功能正常。但是当前版本中，我没有看到 WebUI，我希望增加 WebUI 的功能。


```

AI 给出的结论如下：

```
    项目中已经存在一个最小 WebUI，位于：
    
    - `module/src/webroot/index.html`
    - `module/src/webroot/styles.css`
    - `module/src/webroot/app.js`
    
    初始页面很简陋，只显示：
    
    - 当前状态原始输出
    - `start`
    - `stop`
    - `exit`


```

实机刷入后，确认 WebUI 虽然已经生成，但界面和信息组织都比较粗糙，这也是后面重做界面的直接原因。这里当时没有保留截图。  
指令：

```
`r0zygisk-v1.0.0-1-release.zip` 刷入并测试后，功能本身是正常的，也已经生成了 WebUI；但页面既不够直观，也缺少功能说明。因此我进一步参考了 GitHub 上 `KernelSU` 和 `ZygiskNext` 的 WebUI，想先弄清它们分别提供了什么功能，再决定当前项目里应该接哪些能力。


```

#### 目标

弄清楚官方 ZygiskNext WebUI 提供哪些功能，再决定哪些功能能在当前 `r0zygisk` fork 中真实接入。

#### 排查

本地先检查项目结构，确认 WebUI 在：

```
module/src/webroot/index.html
module/src/webroot/styles.css
module/src/webroot/app.js


```

再查看控制脚本：

```
module/src/zygisk-ctl.sh


```

实际内容是通过：

```
exec $MODDIR/bin/zygisk-ptrace64 ctl $*


```

调用 native 里的 `ctl start|stop|exit`。

随后查看 `loader/src/ptracer/main.cpp`，确认当前后端只支持：

```
ctl start
ctl stop
ctl exit
version


```

换句话说，当前 fork 并没有暴露官方 ZygiskNext 中那些设置项对应的配置接口。

#### 官方 WebUI 功能参考

通过 GitHub 官方仓库的 `i18n` 分支文案确认，ZygiskNext WebUI 功能大致包括：

*   状态页
*   基本信息
*   设置页
*   Root 实现显示
*   Zygote monitor 状态
*   Zygisk module 状态
*   ZN Module 状态
*   KernelSU / Magisk / APatch 排除列表策略说明
*   日志写入 dmesg
*   非 root 应用作为排除列表
*   强制排除列表
*   匿名内存加载
*   Zygisk Next linker
*   Zygote 注入状态说明
*   模块问题提示

但是当前 `r0zygisk` 后端没有这些设置项的持久化配置或控制命令。因此不能做 “看起来能点，实际没有接线” 的假开关。

#### 第一版方案

把页面做成：

*   状态仪表盘
*   追踪器控制按钮
*   已发现 Zygisk 模块列表
*   每个按钮的说明
*   原始诊断输出
*   ZygiskNext 参考功能地图，并标注当前 fork 是否接入

#### 修改文件

重写：

*   `module/src/webroot/index.html`
*   `module/src/webroot/styles.css`
*   `module/src/webroot/app.js`

#### 页面结构改造

新增的页面结构包括：

*   顶部标题：`Zygisk 控制台`
*   总状态评分：`0/3` 到 `3/3`
*   三个状态卡片：
    *   Daemon
    *   Zygote
    *   Root
*   三个操作按钮：
    *   启动追踪
    *   停止追踪
    *   退出监控
*   已发现的 Zygisk 模块列表
*   操作说明
*   原始诊断信息

#### 前端状态读取逻辑

初版 WebUI 通过管理器 WebUI bridge 执行 shell：

```
cat /data/adb/modules/r0zygisk/module.prop
ps -A | grep -E "zygiskd|zygisk-ptrace|app_process"
for d in /data/adb/modules/*; do ...


```

然后从 `module.prop` 的 `description=` 中解析：

```
[monitor:tracing, zygote64:injected, daemon64:running(...)]


```

#### 验证

执行：

```
node --check module/src/webroot/app.js
./gradlew :module:prepareModuleFilesRelease --no-daemon --console=plain
./gradlew :module:zipRelease --no-daemon --console=plain


```

构建通过，产物为：

```
module/build/outputs/release/r0zygisk-v1.0.0-1-release.zip


```

#### 用户现象

刷入后界面看起来不错，但出现：

*   当前状态显示追踪为运行
*   Daemon 为运行
*   进程表未发现 `zygisk`
*   Zygote 未确认
*   Root 未识别
*   无法读取 `module.prop`
*   三个按钮点击没有任何反应

#### 初步判断

当时判断存在两个可能问题：

1.  模块路径被硬编码为：

```
/data/adb/modules/r0zygisk


```

但实际刷入后模块目录可能在其他路径或更新目录中。

1.  WebUI 的 `exec` 调用方式可能不兼容当前管理器。

#### 模块路径修复

加入自动定位模块目录逻辑：

```
for base in /data/adb/modules /data/adb/modules_update /data/adb/ksu/modules /data/adb/ap/modules; do
  ...
  if grep -q '^id=r0zygisk$' "$prop" || grep -q '^; then
    MODDIR=${prop%/module.prop}
  fi
done


```

前端不再固定依赖 `/data/adb/modules/r0zygisk`，而是在刷新和点击按钮时动态查找真实模块目录。

#### 第一版 exec 兼容修复

一开始用过：

```
bridge.exec(command, callback)


```

并尝试根据返回值判断 Promise、同步结果或 callback。

重新打包后，问题还没有彻底解决。

#### 用户现象

实机反馈如下：

*   不再显示 `ZygiskNext 参考` 这个块
*   诊断信息显示：

```
无法直接执行 WebUI 命令
exec 超时，管理器没有返回命令结果


```

*   页面上显示：
    *   无法连接执行接口
    *   Daemon、Zygote、Root 都等待刷新
    *   按钮仍然没有结果

#### 删除参考块

从 `index.html` 删除：

```
<section class="panel">
  ...
  <h2>官方 WebUI 功能地图</h2>
  ...
</section>


```

同时从 `app.js` 删除 `officialFeatures` 和 `renderFeatures()`，从 `styles.css` 删除对应的 `.feature-grid`、`.feature-item` 样式。

#### exec 超时根因

继续查 KernelSU WebUI API 后确认：

KernelSU 的底层 bridge 不是：

```
exec(command, callbackFunction)


```

而是：

```
exec(command, JSON.stringify(options), callbackFunctionName)


```

关键差异是第三个参数传的是 “全局回调函数名”，不是函数对象。

#### 修复方案

在 `module/src/webroot/app.js` 中实现：

```
const callbackName = `r0zygisk_exec_${Date.now()}_${callbackCounter++}`;

window[callbackName] = (errno, stdout, stderr) => {
  finish({ errno, stdout, stderr });
};

bridge.exec(command, "{}", callbackName);


```

同时保留降级：

```
bridge.exec(command, callbackName)
bridge.exec(command)


```

最终执行顺序：

1.  `bridge.exec(command, "{}", callbackName)`
2.  `bridge.exec(command, callbackName)`
3.  `bridge.exec(command)`

#### 验证

执行：

```
node --check module/src/webroot/app.js
./gradlew :module:zipRelease --no-daemon --console=plain


```

并检查 zip：

```
unzip -p module/build/outputs/release/r0zygisk-v1.0.0-1-release.zip webroot/app.js | rg "callbackName|exec\\(command"


```

确认 zip 中包含：

```
bridge.exec(command, "{}", callbackName)


```

#### 用户现象

实机反馈中，SukiSU 模块列表里出现了：

```
tracing, zygote64: injected, daemon64: running(Root: KernelSU, module(3): ...)


```

并询问这是什么意思。

#### 解释

这是 `r0zygisk` 原本把运行状态写进 `module.prop` 的 `description=` 中导致的。

在 `loader/src/ptracer/monitor.cpp` 原始逻辑中：

```
fprintf(prop.get(), "%s[%s] %s", pre_section.c_str(), status_text.c_str(), post_section.c_str());


```

它会生成类似：

```
description=[monitor:tracing, zygote64:injected, daemon64:running(...)] Standalone implementation of Zygisk.


```

SukiSU 的模块列表直接显示 `description`，所以用户会看到这串运行状态。

#### 含义说明

这些字段含义：

*   `monitor: tracing`：追踪器正在运行
*   `zygote64: injected`：64 位 zygote 已注入
*   `daemon64: running`：64 位 daemon 正在运行
*   `Root: KernelSU`：当前 root 实现识别为 KernelSU 系
*   `module(3)`：发现了 3 个 Zygisk 模块

这些状态本身说明模块运行正常，但并不适合直接显示在模块列表里。

#### 调整目标

这里的目标已经明确：

即：

*   SukiSU 模块列表保持干净描述
*   运行状态不要再塞进 `module.prop description`
*   状态写入独立文件，如：

```
/data/adb/modules/r0zygisk/status.json


```

#### native 修改

修改文件：

```
loader/src/ptracer/monitor.cpp


```

主要变化：

1.  删除 `pre_section`、`post_section` 的描述拼接逻辑。
2.  不再生成临时 `module.prop` 并 bind mount 回模块目录。
3.  新增：

```
static std::string status_path;


```

1.  `prepare_environment()` 改为：

```
status_path = "./status.json";
close(open(status_path.c_str(), O_WRONLY | O_CREAT | O_TRUNC, 0644));
updateStatus();


```

1.  `updateStatus()` 写 JSON：

```
{
  "monitor": "tracing",
  "stop_reason": "",
  "zygote64": "injected",
  "daemon64": "running",
  "daemon64_info": "Root: KernelSU,module(3): ...",
  "zygote32": "unsupported",
  "daemon32": "unsupported",
  "daemon32_info": "",
  "raw": "monitor:tracing, zygote64:injected, daemon64:running(...)"
}


```

1.  状态文本里的表情也移除，改成纯文本：

```
tracing
stopped
exited
injected
not injected
running
crashed
unknown


```

#### WebUI 修改

修改：

```
module/src/webroot/app.js


```

刷新命令新增读取：

```
cat "$MODDIR/status.json" 2>/dev/null || true


```

诊断信息增加：

```
--- status.json ---
...


```

前端逻辑改为：

*   优先解析 `status.json`
*   如果旧版本还在运行，没有 `status.json`，则回退解析 `module.prop description`

#### 验证

执行：

```
node --check module/src/webroot/app.js
./gradlew :module:zipRelease --no-daemon --console=plain


```

并检查 native so：

```
strings module/build/outputs/module/release/lib/arm64-v8a/libzygisk_ptrace.so | rg "status\\.json|description=|monitor:"


```

确认包含：

```
./status.json


```

并确认不再依赖 `description=` 状态拼接。

#### 用户刷入后反馈

再次刷入后，SukiSU 模块列表只显示：

```
Standalone implementation of Zygisk.


```

解释为：

```
Zygisk 的独立实现。


```

也就是 `r0zygisk` 不依赖 Magisk 自带 Zygisk，而是自己实现一套 Zygisk 注入和守护进程机制，让 KernelSU / SukiSU 环境也能加载 Zygisk 模块。

这说明把运行状态迁移到 `status.json` 的方案已经生效。

#### 用户现象

后续实机反馈如下：

*   SukiSU 模块描述已经干净
*   但 WebUI 中 `ZYGOTE` 红点
*   显示 “未确认”
*   上方状态只有 `2/3`
*   正常应该是 `3/3`

#### 根因

native 已经写出结构化 JSON：

```
"zygote64": "injected"


```

但前端判断还主要依赖 `raw` 文本：

```
const status = parseStatus(readStatusRaw(statusText, prop));
const zygoteOk = status.zygotes.some((item) => /:injected\b/.test(item));


```

如果 `raw` 文本解析失败，或者和旧格式有细微差异，前端就会漏判 Zygote。

#### 修复

在 `module/src/webroot/app.js` 中把状态读取改成：

```
function readStatus(statusText, prop) {
  const text = statusText.trim();
  let json = null;
  let raw = prop.description || "";

  if (text) {
    try {
      json = JSON.parse(text);
      if (json && json.raw) {
        raw = String(json.raw);
      }
    } catch (_) {
      raw = text;
    }
  }

  const parsed = parseStatus(raw);
  parsed.json = json;
  return parsed;
}


```

判断逻辑改成优先用 JSON 字段：

```
const json = status.json || {};

const monitorOk =
  json.monitor === "tracing" ||
  status.monitor.includes("tracing") ||
  /zygisk-ptrace/.test(processes);

const daemonOk =
  json.daemon64 === "running" ||
  json.daemon32 === "running" ||
  status.daemons.some((item) => item.includes("running")) ||
  /zygiskd/.test(processes);

const zygoteOk =
  json.zygote64 === "injected" ||
  json.zygote32 === "injected" ||
  status.zygotes.some((item) => /:injected\b/.test(item));


```

显示详情也改为优先显示 JSON：

```
zygote64:injected
daemon64:running


```

#### 验证

执行：

```
node --check module/src/webroot/app.js
./gradlew :module:zipRelease --no-daemon --console=plain


```

并确认 zip 中包含：

```
unzip -p module/build/outputs/release/r0zygisk-v1.0.0-1-release.zip webroot/app.js | rg "json\\.zygote64"


```

输出包含：

```
json.zygote64 === "injected"


```

最终 zip 时间：

```
2026-04-17 14:54


```

截至本记录生成时，主要文件状态如下。

文件：

```
module/src/webroot/index.html
module/src/webroot/styles.css
module/src/webroot/app.js


```

当前能力：

*   显示总评分 `0/3` 到 `3/3`
*   显示 monitor / daemon / zygote / root
*   从 `status.json` 读取结构化状态
*   回退兼容旧 `description` 状态
*   自动定位真实模块目录
*   显示已安装 Zygisk 模块列表
*   支持追踪器操作：
    *   启动追踪
    *   停止追踪
    *   退出监控
*   诊断区显示：
    *   `module.dir`
    *   `module.prop`
    *   `status.json`
    *   `processes`
    *   `modules`

文件：

```
loader/src/ptracer/monitor.cpp


```

当前行为：

*   不再把状态写入 `module.prop description`
*   不再污染 SukiSU 模块列表
*   状态写入：

```
/data/adb/modules/r0zygisk/status.json


```

文件：

```
module/src/module.prop


```

当前描述：

```
description=Standalone implementation of Zygisk.


```

SukiSU 中显示这句话是正常现象，含义是：

```
Zygisk 的独立实现。


```

补充过程中的一些截图和当前最终版本截图：

Android 15 测试：

![](https://bbs.kanxue.com/upload/attach/202605/703941_KSUHGJMPJ23SB9B.png)

Android 16 测试:

![](https://bbs.kanxue.com/upload/attach/202605/703941_M27WGXD5ANJSE9V.png)  
![](https://bbs.kanxue.com/upload/attach/202605/703941_BDJTGB85GXRKXEZ.png)  
![](https://bbs.kanxue.com/upload/attach/202605/703941_V2EEBM54FD4Y28P.png)  
![](https://bbs.kanxue.com/upload/attach/202605/703941_22CUMJ9RB8828KC.png)

Android 14 SukiSU：

最后把模块加载修改为列表显示，不挤成一坨，当前最终版的 WebUI 样式：

![](https://bbs.kanxue.com/upload/attach/202605/703941_FGHA9A5MRHC4S3K.png)

测试后功能没有问题，但使用 Hunter 最新版 6.58 发现仍会被检测：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_P2KX9KHB4CXKNTP.png)

以上为使用 ptrace 的方式注入。

这里尝试了多种修改方式，最终仍被检测到；期间还出现过一次卡开机，随后重刷并重建环境后继续分析。

第一版基于 `ptrace` 的注入方案在功能上已经稳定，但用 Hunter `6.58` 实测时仍会被检测，日志里出现：

```
Ptrace Check Zygisk
find zygisk detected, zygote root pid ...


```

这说明问题不在 “是否能注入”，而在 “注入行为特征是否被对方检测到”。所以第二版的核心工作不是继续堆规避技巧，而是先把 Hunter 的检测链路完整拆出来，再按检测点反推改造方向。

使用 `apktool` 解包 `com.zhenxi.hunter.apk`，全局搜索 `Ptrace Check`、`find zygisk detected`、`zygote root pid` 等关键字。

结果很快明确：

1.  smali 层能看到 `NativeEngine.checkZygisk()` 的声明与调用。
2.  关键日志模板不在业务 smali 里，而在 `lib/arm64-v8a/libhunter.so`。

这一步很关键。因为如果误判成 “Java 层静态字符串检测”，后续优化方向会完全跑偏。

接下来把谁在触发 `checkZygisk()` 这件事补齐。沿着 smali 回溯后，能定位到两条主要路径：

1.  `ZhenxiServerIpc` 路径：最终进入 `P1(6)`，再走到 `NativeEngine.checkZygisk()`。
2.  `ZhenxiServerTwinIpc` 路径：最终进入 `P1(8)`，同样会调用 `NativeEngine.checkZygisk()`。

同时在 `AndroidManifest.xml` 中可以看到对应服务进程（包括 `hunter_server_iso` 与 twin 服务），和运行时日志中的进程标签可以对上。这意味着检测不是偶发调用，而是有明确的服务化触发链路。

因为目标 so 是 `ELF aarch64` 且有 strip，不能依赖常规符号名直接定位，所以采用了 “字符串 + JNI 注册表 + 反汇编交叉” 的方式：

1.  在 so 里抽到关键字符串：
    *   `Ptrace Check Zygisk`
    *   `find zygisk detected ,zygote root pid`
    *   `/system/lib64/libzygisk_loader.so`
    *   `/system/lib64/libzygisk.so`
    *   `/sbin/.magisk/modules/zygisk_lsposed`
2.  定位 `JNI_OnLoad` 的动态注册逻辑，确认 `checkZygisk` 来自 `RegisterNatives`，不是静态导出。
3.  还原 `JNINativeMethod` 表后，锁定 `checkZygisk` 对应 native 函数入口：`libhunter.so@0x2a93f8`。

到这一步，链路已经闭合：Java 触发点、JNI 绑定点、native 实现点三者一致。

`checkZygisk` 不是单一判断，而是 “多分支命中后统一组装结果对象” 的结构。结合反汇编可还原为：

1.  先构造标题 `Ptrace Check Zygisk`。
2.  进行 pid/zygote 关系相关检查；命中后拼接 `find zygisk detected ,zygote root pid ...`。
3.  进行路径 / 特征串检查，涉及 `libzygisk_loader.so`、`libzygisk.so`、`zygisk_lsposed` 等标识。
4.  把命中结果封装成返回对象（`ListItemBean`）上报到上层。

也就是说，Hunter 在这一块并非只看某个固定文件是否存在，而是把 “进程关系 + 特征路径 + 运行态信息” 做了组合判定。

这一阶段的目标很明确：不再继续打补丁式修修补补，而是参考 `ZygiskNext 1.3.4` 的成熟路线，重做第二版运行链路。整体工作分成两部分：先改注入方式，再完成工程与模块命名收敛。

第一版是典型的 `ptrace -> 远程 dlopen` 路线：

*   通过 `r0z-trace32/64` 常驻进程监控 `zygote/app_process`
*   命中后注入 `libr0z.so`
*   运行过程中会留下较明显的 ptrace 与进程关系特征

在第七部分中确认 Hunter 检测是 “多维组合判定” 后，这条路线风险较高，所以第二版切换为 `native bridge loader -> payload -> zygisk core -> daemon`。

关键改动如下：

1.  新增并接入：`libzn_loader.so`、`libpayload.so`、`libzygisk.so`。  
    这一步不是简单多加几个库，而是把 “入口加载”“功能实现”“Zygisk 主体” 拆成三层。好处是桥接库只负责承接系统加载，payload 只负责承载运行逻辑，真正的 Zygisk 核心则保留自己的职责边界，后续替换和排查都更清楚。
2.  `entry.cpp` 增加 `zygisk_entry`，构建产物从 `libr0z.so + libr0z_ptrace.so` 切换为三段式库输出。  
    原来的 ptrace 路线依赖主动盯进程，特征非常明显；新的 native bridge 路线则是让系统按属性把桥接库拉起来，再由入口去串起后续流程。`zygisk_entry` 就是为了把这个入口补齐。
3.  模块脚本不再依赖 `r0z-trace`，改为 `system.prop` 设置 `ro.dalvik.vm.native.bridge=libzn_loader.so`，并在 `post-fs-data.sh/service.sh` 启动 daemon。  
    这里的变化核心是把 “前台监控” 改成“系统加载”。`r0z-trace` 这类常驻监控进程最容易留下 ptrace 痕迹，所以第二版直接放弃它，把启动触发点收敛到属性注入和系统阶段脚本上。
4.  库安装路径统一为 `$MODPATH/system/lib` 与 `$MODPATH/system/lib64`。  
    这样做是为了按 32/64 位 ABI 分目录落盘，确保 native bridge 和后续 payload 在系统预期的库路径里可直接被加载，避免额外的路径适配和查找失败。
5.  daemon 状态链路改造为本地 `status.json`，移除对旧 `init_monitor` socket 的硬依赖。  
    旧的 socket 更像临时联调通道，不适合作为 WebUI 的稳定数据源。改成 `status.json` 后，状态就变成了可直接读取的结构化文件，WebUI、模块列表和调试输出可以各自独立展示，不再互相绑死。

这次改造是架构级切换，不是启动命令替换，实际同时调整了注入入口、动态库产物、安装路径、启动时序与状态通道。

完成注入路线切换后，第二版继续做命名统一，避免构建、日志、脚本、WebUI 之间出现旧名残留。

指令：

```
修改目录中 zygisk 字符串为 r0z，不要影响核心功能。


```

执行过程：

```
统一规则如下：

- 模块标识：`r0zygisk` -> `r0z`
- daemon：`zygiskd` -> `r0zd`
- 控制脚本：`zygisk-ctl.sh` -> `r0z-ctl.sh`
- 注入核心库发布命名：`libzygisk.so` -> `libr0zgk.so`
- 版本：`v1.0.1`，`versionCode=2`

同步范围覆盖构建、打包、脚本、WebUI 与文档：

1. Gradle/Cargo 任务与逻辑名映射。
2. `module.prop`、安装脚本、控制脚本文案统一 `r0z`。
3. 打包签名与文件清单映射更新。
4. WebUI 标题与状态描述统一。
5. 发布包命名统一为 `r0z-v1.0.1-2-<buildType>.zip`。

这一部分的目的不是“改展示文字”，而是把第二版运行链路与工程标识统一成同一套语义，保证后续排障和迭代基线稳定。


```

执行结果：项目完成改名后刷入测试，仍被检测；原因是检测侧仍能发现 `libzygisk.so`。

接下来继续让 AI 进行修改：

指令：

```
为什么生成产物中仍存在 `libzygisk.so`？它的功能是什么，是否可以改名为 `libr0zgk`？


```

执行结果：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_6CPK4KE6N5247DF.png)

指令:

```
改名并保持可运行


```

执行结果：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_JRSHNNRMD3AGCQ2.png)  
![](https://bbs.kanxue.com/upload/attach/202605/703941_84AU2W5G5PECGU6.png)  
![](https://bbs.kanxue.com/upload/attach/202605/703941_D9RRQWJQT6FX25W.png)

刷入手机结果：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_BCNB84JMK7NPJBV.png)

WebUI：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_6UV42T5TBERSSC4.png)

安装和打开 lsposed 模块也正常：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_VSRA2KXN43QAA8D.png)

最新版 Hunter 检测结果全绿：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_4CZV823HYEAAUUT.png)

ruru 结果：  
![](https://bbs.kanxue.com/upload/attach/202605/703941_Y2EJE7XB9QH9ZH8.png)

至此，第二版大修结束，已通过新版 Hunter 的 Zygisk 检测。

这次做下来，最大的收获其实不只是把模块跑起来了，而是把一条完整的开发链路真正走通了：先确认最新版 `Magisk` 的 `Zygisk` 已经适配 `Android 16`，再借助 `code-panorama` 和 AI 快速梳理源码结构，后面一步步补齐 API、JNI 签名、打包链路、`WebUI`、守护进程状态和注入方式，最后再去处理真实设备上的兼容问题和检测问题。一路做下来，才把现在这个带 `WebUI`、功能上等效于 `Zygisk` 的 `r0z` 模块完整落地。

完整源码和发布包已经开源在 GitHub Releases：

[r0zygisk Releases](https://bbs.kanxue.com/elink@8b7K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6X3P5i4u0D9L8%4k6W2i4K6u0r3M7U0m8*7P5h3N6A6M7$3E0Q4x3V1k6J5k6h3I4W2j5i4y4W2M7H3%60.%60.)

这篇文章记录的是一个完整模块从源码阅读、迁移实现、联调修复，到最后对抗检测的过程。源码规模够大、调用链够长的时候，AI 能帮人省掉很多纯体力活，但真正决定结果的，还是自己去拆执行流程、判断注入链路、定位检测点、解决刷机和兼容性问题。至少对这次来说，目标已经完成了：模块做出来了，`WebUI` 跑起来了，`Android 16` 适配了，第二版也通过了新版 `Hunter` 的检测。

不过目前已经有朋友反馈，还是存在被检测到的情况。后续还会继续修改和完善，项目也会持续更新，新的进展我会继续补到后面的文章里。谢谢大家关注。

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 11 小时前 被 fyrlove 编辑 ，原因： 补漏