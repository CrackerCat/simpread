> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293385.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

前排提示，纯 ai 文，直接 ai 总结即可

> 开源仓库：[](https://bbs.kanxue.com/elink@3a8K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8R3%60.%60.)AndroidReverser-Test/KCLF
> 
> 分析版本：KCLF 1.5.4，源码基线为 [](https://bbs.kanxue.com/elink@53aK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8Y4c8J5k6h3g2Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2)`6d53c294`，核对日期：2026-10-31。
> 
> 本文依据当前源码和上游文档进行静态分析，面向自有设备、测试 App 和授权研究；不把设计意图等同于真机验证结果，也不承诺任何检测工具的绕过率。

谈到 Android SO 注入，最容易想到的流程是：附加目标进程，修改寄存器，让目标调用 `dlopen()`，然后恢复执行。KCLF 解决的也是 “让一个 SO 在目标进程里运行” 这个问题，但它改变的不只是调用入口。

理解这个框架，需要先拆开两个经常被混为一谈的问题：**谁负责取得目标线程的执行机会，以及谁负责把 ELF 变成可运行的内存镜像。**

KCLF 的第一层答案是：由 KernelPatch 内核模块通过 `task_work` 安排工作，在目标线程返回用户态时重定向执行，不建立 ptrace 跟踪关系。

第二层答案则有两个：默认 `manual` 后端自行完成 ELF 映射、符号解析和重定位；显式 `linker` 后端仍调用目标进程中的 bionic loader。

因此，KCLF 最准确的定位是：**一个内核控制的线程借用执行框架，配有独立手工装载器和系统 linker 两种后端。** 它不是把 SO 放到内核里运行，也不是一个能让用户态代码从内存中消失的 “隐身容器”。

“注入技术” 通常同时涉及三件事。

1.  **执行入口**：怎样让目标开始运行外来的代码？例如 ptrace 调整线程上下文、启动阶段预加载、Zygote 生命周期回调，或者 KCLF 的内核返回边界重定向。
2.  **装载方式**：由系统动态链接器装载 SO，还是自行解析 ELF、分配内存和修复地址？
3.  **驻留能力**：进入进程之后，只执行一个构造函数，还是继续运行脚本引擎、通信通道、Java Hook 或 Native Hook 系统？

这三件事可以组合。ptrace 不等于 `dlopen`，手工映射也不是内核框架专有的能力；Frida 则是一整套动态插桩系统，不能被简化成某一个注入系统调用。

放到这个坐标系中，KCLF 有两条清晰的组合路径：

```
KCLF manual = 内核 task_work + 线程返回现场重定向 + 自行装载 ELF
KCLF linker = 内核 task_work + 线程返回现场重定向 + bionic 动态装载
```

后面讨论 “加载器看不见”“没有 SO 路径” 等现象时，必须说明是哪条路径。manual 的性质不能自动套用到 linker 后端。

KCLF 构建为 KernelPatch 加载的可重定位 KPM，最终使用 `ld -r` 生成模块，而不是常规 Linux LKM。控制命令通过受 superkey 保护的 KPM 接口进入框架。设备必须已经具备可用的 KernelPatch 环境；KCLF 本身不是提权漏洞，也不是普通 App 可以直接调用的无权限注入接口。

下文用 EL0 表示用户态、EL1 表示内核态，payload 则指待注入的 SO 载荷。

核心流程可以概括为：

```
控制端提交目标 TID 和 SO 路径
    ↓
为目标线程排队 task_work
    ↓
目标线程内核上下文：准备临时映射、装载元数据与执行跳板
    ↓
修改该线程返回 EL0 的 PC / SP
    ↓
EL0：执行 IFUNC / 构造函数，或调用 bionic loader
    ↓
EL0：清理 ABI 参数页、恢复现场、写完成 token、返回原 PC
    ↓
内核在后续观察点收集结果，再回收临时映射
```

`kclf_work_prepare_body()` 在目标线程自己的上下文中执行。这一点很重要：准备阶段面对的是目标的地址空间、凭据和命名空间，而不是控制端的环境，也不是另起一个普通内核 worker 去冒充目标。

默认通知方式为 `TWA_RESUME`，等待目标返回用户态；可选的 `wake` 使用经过兼容性检查的 `TWA_SIGNAL`，帮助唤醒可中断等待。这里的 `SIGNAL` 是 task_work 通知机制的名称，不代表框架给 App 人为投递了一个供信号处理函数消费的用户信号，更不是一次 `SIGSTOP` 附加。

不过，“没有 ptrace-stop” 不等于 “没有暂停感”。文件读取、准备映射和执行构造函数仍然占用被借用线程的时间；在主线程上运行缓慢的构造函数，同样可能造成界面卡顿。

准备阶段创建临时匿名区域，容纳生成代码、ABI 参数、私有现场，以及必要时使用的临时栈。随后再排队 hijack 工作，在经过验证的 `pt_regs` 帧中保存原始现场，将返回 PC 指向跳板入口，将 SP 指向选定栈。

`kclf_exec_hijack()` 还会检查目标是否为 64 位 EL0、用户地址是否合理、是否存在不支持的向量状态，以及相应的跟踪 / 模拟标志。默认还检查系统调用边界和 linker 内部执行位置；这些检查属于降低风险的启发式，不是 “这个时刻一定没有锁” 的证明。

实现还把 `syscallno` 设为 `-1`，避免把新跳板入口当成原系统调用返回进行重启处理。用户函数则始终在 EL0 执行：内核准备地址和元数据，不在 EL1 直接调用 SO 构造函数。

对应实现见 [](https://bbs.kanxue.com/elink@6e3K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9J5k6h3y4Q4x3U0y4x3y4o6l9K6i4K6u0V1e0o6b7^5y4b7%60.%60.)`kclf_work_prepare_body()` / `kclf_work_hijack_body()` 和 [](https://bbs.kanxue.com/elink@ee0K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6p5J5y4K6m8Q4x3X3c8x3x3e0x3@1x3b7%60.%60.)`kclf_exec_hijack()`。

EL0 跳板完成工作后，先把结果复制到私有页，清零 ABI 参数页，再恢复基础浮点 / SIMD（FPSIMD）状态、栈指针 SP、条件标志 NZCV 和通用寄存器，写入每请求不同的完成 token，最后返回原 PC。

这里有一个重要例外：最终通过 `RET x16` 返回，x16 被用于保存原 PC，因此原 x16 的值没有恢复。不能把这个过程描述为 “所有寄存器完全无损恢复”；后文会讨论其影响。

完成 token 只是一笔内存写入，不会主动产生专用系统调用、故障或信号。内核主要通过两条路径发现它：

*   被借用线程进入被观察的普通系统调用时，在 before hook 中读取 token 和结果。
*   控制端执行 `status`、`reap` 或 `quiesce` 时，为目标排队 poll task_work，由它检查完成情况。

源码的观察池包含 70 个系统调用，实际安装数取决于内核与 KernelPatch 环境。这些 hook 的职责是读取请求状态，不是改写、吞掉 App 的系统调用，也不是过滤其读取到的 maps 内容。

由于 token 在最终返回前就被写入，poll 路径还会检查目标 PC 是否已经离开整个临时区域，避免过早解除仍在执行的代码映射。完整结果被收集后，cleanup 才应用延迟的 RELRO 保护、提交或释放镜像，并回收临时区。见[](https://bbs.kanxue.com/elink@5a6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9J5k6h3y4Q4x3U0y4x3y4o6R3%4i4K6u0V1e0o6M7%4y4l9%60.%60.)完成观察与清理实现及[](https://bbs.kanxue.com/elink@2b0K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6p5J5x3o6S2Q4x3X3c8x3x3e0t1I4y4b7%60.%60.)退出临时区检查。

KCLF 分开管理请求槽与 manual 镜像表。请求槽描述一次执行事务，镜像表记录已经装载的 SO 及引用计数。回收请求槽，不会顺便删除驻留 SO。

同一进程、同一地址空间、完全相同路径字符串的重复 manual 加载，可以复用镜像并增加引用，不重复执行构造函数。最后一个引用被释放时才执行析构并解除镜像映射；该引用计数由 KCLF 自己管理，不是 bionic 的引用计数。

因此，观察到 `state=done` 也不能立刻断言资源全部清理：还要区分残留的 `pending`、临时 `map` 和有意保持驻留的镜像。相关实现见[](https://bbs.kanxue.com/elink@f6cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3y4U0l9K6i4K6u0V1e0o6j5#2x3R3%60.%60.)镜像事务登记与 [](https://bbs.kanxue.com/elink@7adK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3y4K6b7#2i4K6u0V1e0o6M7^5y4l9%60.%60.)`kclf_manual_finish()`。

默认 manual 后端不调用 bionic 的 `dlopen`、`dlsym`、`dlclose`，也不编辑 `soinfo`。它自行承担了系统 loader 的一部分职责。

新镜像主要经过以下步骤：

1.  **读取文件快照。** 在目标上下文打开 SO，把文件内容读入有大小限制的内核缓冲区。它仍然需要一个目标可读的源文件，并非凭空生成的 “纯内存文件入口”。
2.  **校验并映射 ELF。** 要求 ELF64、小端、AArch64、`ET_DYN`，根据 `PT_LOAD` 预留匿名区域，复制各段内容，利用匿名页的零初始化形成 BSS。
3.  **保留解析结果，处理内存头部。** 默认清除内存 ELF 头；符合范围条件时连同 program headers 清除。后续装载使用已保存的解析元数据。
4.  **寻找已加载依赖。** 扫描目标已有映射，解析可识别的 ELF/SONAME，根据 `DT_NEEDED` 构建直接和传递依赖的有序闭包。
5.  **解析符号并修复地址。** 支持 SysV/GNU hash、版本符号，以及常见 AArch64 RELA、RELR 和 Android APS2 packed RELA。普通重定位由内核准备阶段完成；来自依赖库的 IFUNC resolver（运行时选择具体函数实现的解析函数）延迟到 EL0 执行。
6.  **执行初始化并建立生命周期。** EL0 按 `DT_INIT`、正序 `INIT_ARRAY` 调用；需要卸载时，按逆序 `FINI_ARRAY`、`DT_FINI` 调用。驻留记录保存在 KCLF 的镜像表中。

这里的重定位不是简单 “把地址整体加上基址”：例如 `RELATIVE` 使用装载偏移加 addend，`GLOB_DAT` / `JUMP_SLOT` 等还需要找到目标符号及其版本。KCLF 会检查表和写入地址的边界，并拒绝未实现的语义。具体入口见 [](https://bbs.kanxue.com/elink@afdK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3y4U0f1@1i4K6u0V1e0o6M7J5z5l9%60.%60.)`kclf_manual_prepare()`及 [](https://bbs.kanxue.com/elink@e81K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2D9k6W2)9J5k6h3x3%60.)ELF 解析与重定位代码。

为什么这会影响模块枚举？以 AOSP bionic 为例，`do_dl_iterate_phdr()` 遍历的是 linker 管理的 `soinfo` 列表，不是遍历进程中每一块可执行内存。manual 镜像没有登记进去，自然不会仅因为 “已经可以执行” 就出现在这份清单中。这是从装载方式和枚举实现共同得出的结论，而不是 KCLF 伪造了 API 返回值。[](https://bbs.kanxue.com/elink@f9aK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0W2k6$3!0G2k6$3I4W2M7$3!0#2M7X3y4W2i4K6u0W2j5$3!0E0i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6T1K9h3!0F1K9h3y4Q4x3V1k6Q4x3V1u0Q4x3V1k6J5k6h3k6K6i4K6u0r3K9r3g2S2k6s2y4Q4x3V1k6E0j5h3W2F1i4K6u0r3L8r3W2F1K9$3g2J5i4K6u0r3L8r3W2F1K9$3g2J5i4K6u0W2j5%4m8H3)AOSP linker 实现

不过，系统 loader 没有管理它，也意味着系统 loader 不会自动提供完整的运行时服务：`dladdr` 地址归属查询、常规 `dlsym` 查找、异常展开、TLS 和调试器集成都不能照搬普通 SO 的预期。KCLF 返回的 manual `image-id` 是框架 ID，不能当作标准 `dlopen` handle 使用。

同名 SONAME 在 Android 进程里不一定只有一份。KCLF 根据导入对象和候选提供库的来源路径打分，综合同目录、同 APEX、同分区等线索；无法消除歧义时失败，而不是任意取第一份。

这套机制是在已加载对象中寻找合适依赖的启发式，不是对 bionic namespace、全局符号抢占、可见性及引用管理的完整重现。Android linker namespace 本身负责库之间的隔离和允许的连接关系，因此 “没有调用 linker” 与“正确复刻全部 linker 语义”是两回事。[](https://bbs.kanxue.com/elink@4faK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6K6L8%4g2J5j5$3g2Q4x3X3g2S2L8X3c8J5L8$3W2V1i4K6u0W2j5$3!0E0i4K6u0r3k6r3!0U0M7#2)9J5c8X3y4G2M7X3g2Q4x3V1k6S2M7X3y4Z5K9i4c8W2j5%4c8#2M7X3g2Q4x3V1k6$3L8X3c8C8i4K6u0r3L8r3W2F1K9$3g2J5i4K6u0V1L8X3q4E0k6i4y4H3j5h3y4W2)Android linker namespace 文档

更重要的是，manual **不会自动加载缺失依赖，也不会为提供库增加 bionic 引用计数**。调用方必须保证依赖已加载，并在整个 payload 生命周期内保持有效；另一个线程并发 `dlclose` 或解除相关映射，仍可能破坏执行条件。见[](https://bbs.kanxue.com/elink@14cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3x3e0p5&6i4K6u0V1e0o6x3@1z5l9%60.%60.)依赖收集、来源选择与符号解析。

显式选择 `linker` 时，框架扫描目标已有映射，解析 loader 入口，优先使用 `__loader_dlopen`，必要时回退到 `libdl.so` 的 `dlopen`；随后由 EL0 跳板调用。加载失败可以获取 `dlerror`，一次性模式可以调用 `dlclose`。

这一后端的差异在 “如何得到执行机会”，而不在 “是否经过系统动态链接器”。SO 及其依赖通常仍按 bionic 的装载规则进入对应管理结构，文件映射和模块枚举也仍具有普通动态库的特征。

`__loader_dlopen` 的 caller address 参与 namespace 选择。当前 KCLF 优先取已发现的 libc 地址，再考虑 libdl 或 linker 地址；这不等于自动拥有任意 ClassLoader 对应 namespace 的访问权，也不保证任意 SO 路径都能打开。[](https://bbs.kanxue.com/elink@a76K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6f1#2x3g2)9J5k6p5H3$3y4K6p5%60.)KCLF loader 解析、[](https://bbs.kanxue.com/elink@290K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0W2k6$3!0G2k6$3I4W2M7$3!0#2M7X3y4W2i4K6u0W2j5$3!0E0i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6T1K9h3!0F1K9h3y4Q4x3V1k6Q4x3V1u0Q4x3V1k6J5k6h3k6K6i4K6u0r3K9r3g2S2k6s2y4Q4x3V1k6E0j5h3W2F1i4K6u0r3L8r3W2F1K9$3g2J5i4K6u0r3k6r3I4X3j5$3&6Q4x3X3g2U0M7s2l9%60.)AOSP loader 入口

两个后端是显式选择关系：manual 失败不会自动退回 linker。另一个容易忽略的事实是，直接调用 native loader 不等于经过 Java VM 的 Native Library 装载流程；不能把 “切换 linker 后端” 理解为框架会自动调用 `JNI_OnLoad` 或补齐全部 ART/JNI 集成。

先给出整体对照。表中的可见性描述针对典型路径，具体工具可以额外组合手工映射、清理或其他机制。

<table><thead><tr><th>方案</th><th>取得执行机会的方式</th><th>SO / 模块管理方式</th><th>与 KCLF 的关键区别</th></tr></thead><tbody><tr><td>ptrace + <code>dlopen</code></td><td>调试接口控制目标线程现场</td><td>bionic linker</td><td>存在 tracer/tracee 关系；装载语义更接近 KCLF linker</td></tr><tr><td>ptrace + manual map</td><td>仍由 ptrace 取得执行控制</td><td>自定义 ELF loader</td><td>可以同样不登记 <code>soinfo</code>；区别主要在执行入口，不在 “手工装载” 本身</td></tr><tr><td>启动预加载 / App 主动装载</td><td>在受控启动流程或应用代码中执行</td><td>通常是系统 loader</td><td>需要控制启动或应用集成点，而非在已有线程返回边界插入工作</td></tr><tr><td>Zygisk</td><td>App / system_server specialization 前后回调</td><td>由模块与框架决定</td><td>依托进程创建生命周期，KCLF 则面向已存在的目标线程</td></tr><tr><td>Frida</td><td>注入、嵌入、预加载等多种模式</td><td>agent / Gadget 加动态插桩运行时</td><td>不只是 SO 装载器，还提供插桩、脚本和通信能力</td></tr><tr><td>KCLF linker</td><td>内核 task_work + 返回现场重定向</td><td>bionic linker</td><td>无 ptrace 附加，但仍保留常规 loader 装载痕迹</td></tr><tr><td>KCLF manual</td><td>同上</td><td>内核准备的匿名 ELF 镜像 + 独立登记表</td><td>同时改变执行入口和装载管理方式，兼容性责任也转移给框架</td></tr></tbody></table>

ptrace 提供线程级观察与控制能力，包括读写内存、寄存器，以及围绕停止 / 继续组织执行。典型远程函数调用式注入是在这个机制上构造一次目标内的调用；ptrace 自身并不要求一定使用 `dlopen`。[](https://bbs.kanxue.com/elink@aadK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0j5h3%5E5%4i4K6u0W2L8%4u0Y4i4K6u0r3L8r3W2F1N6i4S2Q4x3V1k6E0j5h3&6Q4x3X3c8H3j5h3N6W2M7#2)9J5c8X3#2S2L8U0u0Q4x3V1k6H3N6s2u0S2j5$3g2Q4x3X3f1J5i4K6u0W2K9s2c8E0L8l9%60.%60.)Linux `ptrace(2)`

KCLF 不通过外部 tracer 来完成这件事，而是让内核在目标返回用户态之前安排执行现场。由此形成四个关键差异。

**第一，权限基础不同。** ptrace 路径依赖相应的附加权限和安全策略；KCLF 路径依赖已经部署的 KernelPatch 及内核控制能力。这不是在相同权限下换一个更隐蔽的用户态 API，而是把控制点放到了不同层级。

**第二，跟踪关系不同。** KCLF 不因本次注入建立 ptrace 关系，因此不会因这次附加产生对应的 tracer 身份。`TracerPid` 表示当前的跟踪者，而不是 “历史上是否曾发生注入”；ptrace 方案解除附加后，也不能只靠该字段判断之前做过什么。多线程场景还需要区分主线程与各 TID 的状态。[](https://bbs.kanxue.com/elink@58cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0j5h3%5E5%4i4K6u0W2L8%4u0Y4i4K6u0r3L8r3W2F1N6i4S2Q4x3V1k6E0j5h3&6Q4x3X3c8H3j5h3N6W2M7#2)9J5c8X3#2S2L8U0g2Q4x3V1k6H3M7X3!0U0i4K6g2X3M7r3W2V1i4K6g2X3M7%4c8S2N6s2g2K6i4K6u0W2y4g2)9J5k6h3S2@1L8h3H3%60.)Linux `/proc/pid/status` 文档

**第三，停止和恢复协议不同。** 传统 `PTRACE_ATTACH` 会发送 `SIGSTOP`，但不能因此说所有 ptrace 都必须这样：`PTRACE_SEIZE` 附加时不直接停止目标，后续可使用 `PTRACE_INTERRUPT`。真正的区别是 KCLF 没有依赖 ptrace 的跟踪与停止协议，而不是简单比较 “有没有 SIGSTOP”。[](https://bbs.kanxue.com/elink@450K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0j5h3%5E5%4i4K6u0W2L8%4u0Y4i4K6u0r3L8r3W2F1N6i4S2Q4x3V1k6E0j5h3&6Q4x3X3c8H3j5h3N6W2M7#2)9J5c8X3#2S2L8U0u0Q4x3V1k6H3N6s2u0S2j5$3g2Q4x3X3f1J5i4K6u0W2K9s2c8E0L8l9%60.%60.)Linux `ptrace(2)` 的 attach/seize 语义

**第四，工程风险的落点不同。** KCLF 可以把准备映射和完成观察放在内核侧，但必须适配目标内核 ABI、管理 task_work 生命周期、选择安全返回点，并处理当前 x16 恢复限制。内核入口并没有消除重入、锁和异步状态问题，也不能据此直接得出性能必然优于 ptrace 的结论。

尤其要强调：匿名映射、擦除 ELF 头、缩短临时映射寿命、随机化跳板，都不是 “只要不用 ptrace 就自动获得” 的性质；用户态手工装载器也可以采用其中一些策略。KCLF 的特点是这些策略与内核控制入口、独立装载器、EL0 自恢复路径的组合。

启动预加载或在 App 中显式装载库，利用的是受控的进程启动环境或既有应用执行流。Android 的 `wrap.sh` 是官方支持的启动包装机制之一，但它有 API 级别、打包和 debuggable 条件，并不是对所有线上 App 都可直接使用的入口。[](https://bbs.kanxue.com/elink@139K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6V1k6i4k6W2L8r3!0H3k6i4u0Q4x3X3g2S2L8X3c8J5L8$3W2V1i4K6u0W2j5$3!0E0i4K6u0r3L8X3c8C8i4K6u0r3k6%4g2A6k6r3g2K6i4K6u0r3N6%4u0S2M7q4)9J5k6s2y4U0M7X3W2H3N6l9%60.%60.)Android NDK `wrap.sh` 文档

把 Gadget 或其他库集成进 APK，则意味着控制了应用构建 / 集成流程；如果采用重打包，还改变了 APK 制品本身。KCLF 的核心路径不要求修改目标 APK，也不要求目标预先调用某个加载函数，但代价是设备侧必须先具备内核模块运行条件。

所以这里不能笼统说谁 “更方便”：前者要求控制启动或应用，后者要求控制内核环境。它们对研究者和部署环境的前置条件不同。

Zygisk 提供 App 和 system_server specialization 前后的模块回调，天然靠近进程创建阶段。KCLF 则把某个已存在线程作为操作对象，在合适的返回边界借用它执行工作。前者便于在应用生命周期早期介入，后者不以重新走一次 Zygote 创建流程为前提。[](https://bbs.kanxue.com/elink@ba5K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6@1L8%4m8B7L8$3S2F1N6%4g2Q4x3V1k6*7P5h3N6A6M7$3E0Q4x3X3c8E0L8$3c8#2L8r3g2Q4x3X3c8K6j5h3#2H3L8r3g2Q4x3V1k6T1L8r3!0T1i4K6u0r3L8h3q4K6N6r3g2J5i4K6u0r3L8h3!0V1N6h3I4W2i4K6u0r3K9X3&6A6i4K6u0r3P5Y4W2Y4K9i4y4C8i4K6u0W2K9s2m8H3)Zygisk 官方模块 API

Xposed / LSPosed 还涉及 ART 方法 Hook、模块 API 和运行时管理。以 LSPosed 项目自己的定位为例，它是建立在 Riru / Zygisk 模块入口上的 ART Hook 框架。KCLF 当前不提供这些上层能力；成功把一个 SO 的构造函数跑起来，并不等于已经拥有 Java 方法替换、ClassLoader 管理和完整 JNI 生命周期。[](https://bbs.kanxue.com/elink@b39K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6x3f1#2m8G2M7$3g2V1i4K6u0r3e0q4y4b7L8%4y4W2k6l9%60.%60.)LSPosed 项目说明

Frida 官方区分 Injected、Embedded 和 Preloaded 等运行模式。其价值不只在进入进程，还包括 Gum/GumJS、函数拦截、内存操作、脚本执行，以及需要时的双向通信。因此，“Frida 就是 ptrace 注入” 不是准确描述。[](https://bbs.kanxue.com/elink@ba4K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6X3M7X3W2V1j5g2)9J5k6i4u0W2i4K6u0r3k6r3!0U0M7#2)9J5c8X3#2G2k6r3g2K6i4K6u0r3)Frida 运行模式文档

KCLF 当前核心不要求在 App 内常驻一个脚本引擎、RPC 服务或管理线程；它主要负责装载、执行和生命周期协调。这减少了框架必须引入的用户态常驻组件，但也意味着高级插桩能力需要 payload 自行提供。

如果 payload 自己创建线程、通信服务或 Hook 框架，这些组件仍会形成它们自己的可观察行为。不能因为入口换成了 KCLF，就把载荷的特征也视为被消除；更不能假定任意大型 agent 都符合 manual 后端的 ELF 与运行时限制。

`memfd_create()` 创建的是匿名文件，解决的是内容如何以文件对象存在的问题，而不是自动解决线程控制或模块登记问题。如果之后仍交给系统 linker 装载，是否登记模块仍由 loader 路径决定。[](https://bbs.kanxue.com/elink@f90K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0j5h3%5E5%4i4K6u0W2L8%4u0Y4i4K6u0r3L8r3W2F1N6i4S2Q4x3V1k6E0j5h3&6Q4x3X3c8H3j5h3N6W2M7#2)9J5c8X3#2S2L8U0u0Q4x3V1k6E0k6h3#2X3k6q4)9#2k6X3y4J5k6h3q4@1k6g2)9J5k6e0u0Q4x3X3g2Z5N6r3#2D9)Linux `memfd_create(2)`

KCLF manual 当前从指定路径读取 SO，再复制到匿名内存；它不是一个 memfd 加载器，也不因此自动删除源文件。“磁盘上是否有文件”“maps 是否显示源路径”“linker 是否登记镜像” 是三个独立问题。

这里的 “普通 App” 指没有额外内核权限，但可以通过自身 Native 代码检查自己的地址空间、线程和运行时状态的应用。讨论重点是它在进程内部能观察到什么，而不是假设检测者已经可以任意检查内核。

KCLF 的策略并非统一 “屏蔽检测 API”，而是减少一部分痕迹的产生，弱化另一部分痕迹的固定形状，并缩短临时资源的存在时间。

框架使用 task_work 和返回现场重定向，没有通过 ptrace 附加目标，也没有实现把 `/proc` 中 `TracerPid` 改成零的逻辑。

区别在于：它减少的是这次加载产生的 “被调试器附加” 状态，而不是事后遮盖这个状态。因此，单纯检查是否存在 tracer，不足以覆盖这类执行入口；同样，`TracerPid=0` 也不能证明一个进程没有额外代码。

这项性质适用于 manual 和 linker 两个后端，但与 SO 在内存中是否可被识别没有直接等价关系。

manual 的映射、符号解析和重定位发生在内核准备路径中，不需要通过目标的 `dlopen` / `dlsym` 完成这次装载。因此，仅监控这些用户态装载入口，或只枚举 linker 模块表，会缺失这一类镜像。

这不是 “先用 dlopen 正常加载，再偷偷摘链”，而是该镜像从一开始就由另一套装载器管理。它也没有为隐藏自身而修改目标现有函数入口或已有库的 GOT/PLT；payload 自己安装的 Hook 不在这一结论内。

边界同样清晰：linker 后端依旧调用真实 loader；manual payload 的构造函数或其他代码也仍可以自行调用库函数、分配内存或触发新的动态装载。框架没有屏蔽这些后续行为。[](https://bbs.kanxue.com/elink@ce6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3x3%60.)manual 后端定义及实现

这是 manual 后端针对 “按路径找库” 和“按头部认库”的主要处理。

首先，镜像使用匿名私有映射承载，而不是把源 SO 文件直接映射为对应的文件 VMA。因此，maps 中不会因为这块镜像的文件后备关系而自动出现源 SO 路径。注意，这不表示源文件不存在，也不表示 SO 名称不会以字符串形式出现在其他内存中。

其次，镜像前预留 1–8 页随机大小的 pad，并增加对齐余量。保护设置时，前导区域采用首个装载段的权限，使其具备与相邻区域合并的条件，降低 “VMA 起点就是 ELF 起点” 这一固定关系。实际 VMA 合并仍受内核条件影响，不能把它描述成任意设备上的绝对保证。

最后，默认擦除内存中的 ELF 头；只有 program header 表满足首个页与受限前缀范围条件时，才一并清除。`keepelf` 可以保留头部，适用于需要检查自身 ELF 信息的载荷。[](https://bbs.kanxue.com/elink@622K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3y4o6x3#2i4K6u0V1e0o6b7%4y4b7%60.%60.)`map_image()`、[](https://bbs.kanxue.com/elink@bbbK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3y4e0l9^5i4K6u0V1e0o6f1%4x3b7%60.%60.)`protect_image()`、[](https://bbs.kanxue.com/elink@ea2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3y4e0M7K6i4K6u0V1e0o6j5H3x3b7%60.%60.)`scrub_headers()`

这三项处理降低了简单路径匹配和 ELF magic 扫描的直接命中机会，但没有加密或重写整个 payload。代码段、动态字符串、常量、重定位后的指针关系，以及其他未清除内容仍可能提供识别线索。

KCLF 不是每次复制完全相同的静态 trampoline blob，而是通过 `kclf_prog_build()` 生成 AArch64 指令。当前实现会变化的内容包括：

*   代码入口偏移和代码页未使用区域的内容。
*   x19–x26 之间的八个工作寄存器角色。
*   FPSIMD 保存配对、存储槽位及通用寄存器恢复顺序。
*   只影响失活临时值的填充指令。
*   临时区域内代码页、ABI 页、私有页的排列、间隔及总长度。
*   私有页中现场、结果和 token 等字段的位置，以及每请求非零随机完成 token。

这降低了固定字节串、固定偏移和固定邻接关系的可靠性。准确说法是 “ABI 页和私有页没有固定相对位置”，而不是 “两页永远不会相邻”；随机布局中仍可能出现相邻情况。私有布局生成失败时，代码也准备了紧凑回退布局。

这种随机化针对的是模板特征，不是加密隔离。页内容仍在目标地址空间中，语义、控制流和访问关系仍然存在；高熵填充也不保证更像某个具体 App 的正常内存。[](https://bbs.kanxue.com/elink@80cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6M7%4y4q4)9J5k6p5H3I4x3e0b7$3)临时映射与私有布局、[](https://bbs.kanxue.com/elink@ac7K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3y4G2k6r3g2Y4k6h3&6Q4x3X3g2U0i4K6t1K6e0o6x3^5x3q4)9J5k6p5H3$3y4o6x3%60.)机器码生成器

临时区先以 RW 分配，写完代码后将代码页改为 RX，参数页和栈保持 RW；需要独立栈时，在栈下安排 `PROT_NONE` guard。manual 镜像则按 `PT_LOAD` 设置 R、RW、RX，拒绝 W+X 装载段，可访问段之外的空洞保持不可访问。

这避免了 “长期存在一大块 RWX 注入页” 这种明显模式，同时也是内存权限方面的工程约束。不过，RX 只表示不可写，不表示不存在：匿名可执行映射仍可出现在目标的内存布局里。

还有一处值得单独说明：GNU_RELRO 用于把重定位后不应再修改的相关区域设为只读。manual 在 EL0 初始化工作完成后的内核 cleanup 中应用这项保护，构造函数执行期间相关页仍然可写。这不能描述为与常规 linker 完全相同的保护时序，也不是额外的隐藏保证。[](https://bbs.kanxue.com/elink@5eaK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3y4e0l9^5i4K6u0V1e0o6f1%4x3b7%60.%60.)映射保护实现、[](https://bbs.kanxue.com/elink@ccaK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6p5K6y4o6y4Q4x3X3c8x3x3e0x3$3x3R3%60.%60.)延迟 RELRO 应用

框架不要求为这次装载创建一个新的用户线程。栈条件允许时，它优先使用目标 SP 下方预留的空间，避免让 SP 长时间落在一段额外的临时匿名栈里。

当前检查包括栈 VMA 大小、SP 下方可用空间和 hijack 前后的 SP 变化；不满足时才使用带 guard 的临时栈。这样可以减少额外线程和异常栈位置这两类简单特征，但不能消除执行时序、实际 PC 或调用链的变化。[](https://bbs.kanxue.com/elink@6aeK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6V1J5x3g2)9J5k6p5H3&6y4U0l9%60.)`kclf_choose_stack()`

特别是，跳板仍会以真实调用方式执行构造函数、IFUNC 或 loader。不能把 “完成尾部没有额外库调用” 扩展成“整个加载过程没有调用栈痕迹”。载荷创建的线程、修改的线程局部状态或留下的栈内容，也不会自动被框架全部复原。

EL0 跳板的受控指令路径不发出 `SVC`；完成尾部也不靠一个专用 libc 调用、故意异常或合成信号通知内核。它写 token 后自行返回，内核等后续观察点来确认完成。

这种设计减少了 “从匿名跳板发起某个固定完成调用” 的观察点，也不要求为完成协议向 App 的 seccomp 策略争取一个特殊系统调用。

但是，不能由此推出 “整个加载没有内核活动” 或“seccomp 已失效”：映射和保护调整由内核准备 / 清理路径完成；payload、IFUNC 和 linker 执行时仍可能产生普通系统调用和信号，目标原有的用户态系统调用约束没有因此被删除。

框架本身还确实安装了用于观察完成的内核 syscall hook。所以它不是 “无 hook 系统”，只是没有把这些 hook 用来伪造 App 的检测结果。[](https://bbs.kanxue.com/elink@352K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3y4G2k6r3g2Y4k6h3&6Q4x3X3g2U0i4K6t1K6e0o6f1%4x3q4)9J5k6p5H3$3y4o6x3%60.)完成尾部、[](https://bbs.kanxue.com/elink@87bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9J5k6h3y4Q4x3U0y4x3y4U0b7%4i4K6u0V1e0o6M7%4y4l9%60.%60.)观察 hook

ABI 页包含路径、调用地址和初始化表等信息。跳板把必要结果转移到私有页后，会清零整个 ABI 页；确认完成后，内核再解除临时代码、参数和专用栈所在的映射。

这使加载脚手架不必与 SO 一直共同驻留。但这里至少有三个边界：

1.  参数页在运行期间真实存在，清零不是从未出现。
2.  结果、原始现场和 token 所在的私有页，要到临时映射回收时才从地址空间中移除；它不是运行期间对 App 不可访问的 “内核私有内存”。
3.  驻留模式只清理脚手架，不清理 SO 本体；失败或尚未被观察到完成时，临时区也可能继续保留。

一次性 `unload` 可以进一步缩短 SO 的驻留时间，但这要求所有工作真正结束。它不是安全清除后台线程、异步回调和 Hook 的通用按钮。相关处理见 [](https://bbs.kanxue.com/elink@12fK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3y4G2k6r3g2Y4k6h3&6Q4x3X3g2U0i4K6t1K6e0o6f1^5x3q4)9J5k6p5H3#2z5o6V1%60.)ABI 页清理与 [](https://bbs.kanxue.com/elink@802K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6p5K6y4U0S2Q4x3X3c8x3x3e0b7H3x3b7%60.%60.)`kclf_exec_finish()`。

最重要的结论是：**linker 没有登记的镜像，不等于内核没有映射的内存。**

`/proc/pid/maps` 描述的是已映射区域及权限，并不只列出动态链接器认识的 SO。当前 KCLF 没有实现 maps/smaps 行过滤、VMA 隐藏或这些文件内容的伪造；源码中读取 `/proc/thread-self/maps` 的逻辑，是为了定位现有映射，不是为了替换 App 看到的结果。[](https://bbs.kanxue.com/elink@7f4K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0j5h3%5E5%4i4K6u0W2L8%4u0Y4i4K6u0r3L8r3W2F1N6i4S2Q4x3V1k6E0j5h3&6Q4x3X3c8H3j5h3N6W2M7#2)9J5c8X3#2S2L8U0g2Q4x3V1k6H3M7X3!0U0i4K6g2X3M7r3W2V1i4K6g2X3L8h3q4H3M7#2)9J5k6e0g2Q4x3X3g2Z5N6r3#2D9)Linux maps 文档、[](https://bbs.kanxue.com/elink@e03K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6b7J5z5g2)9J5k6p5H3#2y4o6V1%60.)KCLF VMA 遍历实现

从当前实现可以推导出下面这些可见性边界；它们是分析结论，不是本文已经跑过的检测实验。

<table><thead><tr><th>观察面</th><th>KCLF 减少了什么</th><th>仍然留下什么</th></tr></thead><tbody><tr><td>跟踪状态</td><td>不为本次注入建立 ptrace 关系</td><td>跟踪字段只能描述跟踪状态，无法证明有无外来代码</td></tr><tr><td>模块枚举</td><td>manual 不进入正常 linker 登记</td><td>linker 后端仍有正常登记；自定义内存检查不依赖该清单</td></tr><tr><td>maps / smaps</td><td>manual 镜像不以源 SO 文件后备映射的形式出现</td><td>匿名区域、权限、大小及内存用量变化仍在</td></tr><tr><td>内存内容</td><td>默认头部清理、跳板与布局随机化</td><td>payload 指令、常量、动态字符串、指针及部分 ELF 元数据仍可能存在</td></tr><tr><td>线程与栈</td><td>不要求新增装载线程，优先复用原栈</td><td>短时 PC / 调用链变化、回退临时栈和 payload 自建线程</td></tr><tr><td>完成通知</td><td>没有专用完成 syscall 或故障事件</td><td>普通后续 syscall、内核观察 hook 和异步清理窗口</td></tr><tr><td>业务行为</td><td>框架不要求常驻通信运行时</td><td>载荷的文件、网络、回调、Hook 和状态修改不会自动消失</td></tr><tr><td>设备环境</td><td>本文未发现相应隐藏逻辑</td><td>KernelPatch、root / 系统改动的环境线索属于另一类检测问题</td></tr></tbody></table>

一个具有代表性的现象是：某块 RX 区域在内存映射中存在，却不属于普通 linker 模块清单。对于 manual 镜像，这个差异来自其管理方式；但差异本身还不是注入的充分证据，合法 JIT 和其他运行时代码也可能不按普通 SO 方式登记。

同样，RWX、高熵页、无文件名的 RX 映射都不能单独充当确定结论。评价框架时，应区分 “一个简单规则不再命中” 和“无法被更完整的进程内观察识别”。KCLF 的代码支持前一种可能性，并没有证明后一种绝对结论。

当前恢复尾部需要 x16 保存恢复地址，因此不能同时恢复 x16 原值。虽然 AArch64 调用约定把 x16/IP0 视为可被调用过程使用的临时寄存器，但 “函数调用边界允许被改写” 不等于“任意异步中断点都没有活跃值”。

因此，可靠使用需要一个 x16 已无活跃值、相关锁和协议状态也允许执行载荷的协作边界。系统调用白名单不能证明这些条件；`force` 仅跳过部分启发式，不会修复现场恢复能力。

基础 q0–q31、FPCR/FPSR 有保存恢复路径，但 SVE/SME 可扩展状态不在支持范围内。对异步信号观察者而言，整个恢复过程也不是原子的，可能出现中间现场。[](https://bbs.kanxue.com/elink@003K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0i4K6t1K6e0o6p5J5y4K6m8Q4x3X3c8x3x3e0x3@1x3b7%60.%60.)现场检查、[](https://bbs.kanxue.com/elink@c8cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3y4G2k6r3g2Y4k6h3&6Q4x3X3g2U0i4K6t1K6e0o6f1&6z5q4)9J5k6p5H3$3y4o6x3%60.)实际恢复尾部

manual 当前拒绝或不提供完整支持的内容包括 payload TLS、emutls、线程析构注册、需要解析的 payload 自定义 IFUNC、IRELATIVE、跨 DSO CFI、可执行栈和 W+X 段等。带 `PT_GNU_PROPERTY` 的 payload 也会被拒绝，不能仅凭跳板自身使用了 BTI 入口指令，就宣称完整支持载荷的 BTI/PAC 属性契约。它也不自动建立异常展开、调试器及完整 C++/JNI loader 集成。

这不是说所有用 C++ 编译的简单 SO 都必然不能运行，而是不能把 “某个构造函数测试通过” 推广成“任意 C++ / JNI 模块都兼容”。是否可用取决于该文件实际使用的 ELF 特性、导入符号和运行时行为。[](https://bbs.kanxue.com/elink@2daK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2D9k6W2)9J5k6h3y4Q4x3U0y4x3y4K6g2Q4x3X3c8x3x3e0p5@1)ELF 布局检查、[](https://bbs.kanxue.com/elink@836K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3y4Q4x3U0y4x3x3U0R3@1i4K6u0V1e0o6x3@1z5l9%60.%60.)导入符号限制

当前准备阶段使用目标凭据打开 SO，没有为此自动替换目标凭据、修改 non-dumpable 状态或关闭 SELinux。控制端具备内核控制能力，不等于传入的每个路径都对目标 App 可读。

同理，payload 在 EL0 调用 API 时仍处于目标原有环境中；不能把 “不依赖 ptrace 附加检查” 说成“所有访问控制都不再生效”。linker 后端还要满足相应 namespace 与文件执行映射条件。

SO 构造函数可以启动后台线程、注册回调或留下函数指针。构造函数返回后立即解除映射，并不能保证这些使用者已经结束；之后再执行该地址会导致崩溃或卡死。

框架的 `release` 管理镜像引用，不负责替载荷注销所有外部使用者。对于异常完成，代码有保留所有权、标记 abandoned 并等待目标退出的处理，说明设计也没有把所有失败都假定为可以立即回滚。

卸载 KPM 还涉及另一层生命期：不能在仍有 task_work 指向模块代码时强行释放模块。仓库明确要求先 quiesce、排空请求和镜像再卸载，并提示 stock KernelPatch 下不能单靠 exit 返回忙来保证模块文本继续保留。[](https://bbs.kanxue.com/elink@c37K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3d9f1#2b7e0p5g2y4c8f1&6f1b7g2c8u0e0@1&6Q4x3X3g2E0k6l9%60.%60.)项目生命周期与卸载限制

当前实现面向 Android ARM64、64 位目标，接受 Linux 5.x/6.x ABI 家族。它通过 BTF 内核类型信息验证部分结构布局和函数 ABI；缺少某些信息时，使用内核访问函数、procfs 遍历或已观察到的 syscall wrapper 帧进行回退。

这些是兼容性机制，不是 “所有 5.x/6.x 内核和所有页大小都已通过测试” 的声明。厂商回移、内核符号、线程标志、KernelPatch 版本及 16K/64K 页设备仍需要单独验证。[](https://bbs.kanxue.com/elink@510K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3y4G2L8i4m8S2N6q4)9J5k6h3x3%60.)`kclf_compat.c`

当前仓库提供了测试 App、普通 payload、FP 状态破坏 payload、延时构造 payload，以及设备回归脚本。代码包含 marker 检查、FPSIMD 哨兵、临时映射回收、引用计数、取消、多请求槽、目标退出与排空卸载等测试逻辑。[](https://bbs.kanxue.com/elink@4b8K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8Y4c8J5k6h3g2Q4x3V1j5$3k6o6f1K6j5K6t1&6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3N6r3g2K6N6q4)9J5c8X3q4H3M7l9%60.%60.)测试目录

这些测试首先回答的是 “代码是否按预期执行和清理”，并不能仅凭它们的存在证明 “普通 App 检测不到”。测试 APK 本身还采用了 debuggable 配置，不能直接代表所有生产应用的权限和运行时环境。

本文没有执行设备注入、构建或检测回归。因此，以下三个层次必须分开：

*   **源码事实**：例如 manual 不调用 bionic 装载 API、默认清除头部、完成尾部不发专用 syscall。
*   **机制推导**：例如仅依赖 linker 模块清单的观察会遗漏未登记镜像，匿名 RX 区域仍有独立观察价值。
*   **设备实验结论**：例如某个 Android 版本、某个具体 App 或检测器是否发现异常，需要单独实验，本文不代为宣称。

如果要继续阅读源码，建议按下面的入口顺序，而不是先从全部 ELF 常量开始：

<table><thead><tr><th>想回答的问题</th><th>建议阅读位置</th></tr></thead><tbody><tr><td>请求怎样进入目标、何时确认完成？</td><td><a href="https://bbs.kanxue.com/elink@3c8K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&amp;6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9J5k6h3x3%60."></a><code>kclf.c</code>：<code>kclf_cmd_inject</code>、<code>kclf_work_*</code>、<code>kclf_token_before</code></td></tr><tr><td>内存、栈和返回现场怎样准备？</td><td><a href="https://bbs.kanxue.com/elink@011K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&amp;6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2^5k6h3y4Q4x3X3g2U0"></a><code>kclf_exec.c</code>：<code>kclf_exec_prepare</code>、<code>kclf_exec_hijack</code>、<code>kclf_exec_finish</code></td></tr><tr><td>manual 如何管理镜像、依赖和引用？</td><td><a href="https://bbs.kanxue.com/elink@0c4K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&amp;6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3#2S2L8Y4g2S2L8q4)9J5k6h3x3%60."></a><code>kclf_manual.c</code>：<code>map_image</code>、<code>scrub_headers</code>、<code>kclf_manual_prepare</code>、<code>kclf_manual_finish</code></td></tr><tr><td>ELF 支持范围究竟是什么？</td><td><a href="https://bbs.kanxue.com/elink@f0eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&amp;6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3g2D9k6W2)9J5k6h3x3%60."></a><code>kclf_elf.c</code>：<code>ke_file_layout</code>、<code>ke_open</code>、<code>ke_lookup</code>、<code>ke_relocate</code></td></tr><tr><td>跳板怎样执行和恢复？</td><td><a href="https://bbs.kanxue.com/elink@4dbK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&amp;6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3y4G2k6r3g2Y4k6h3&amp;6Q4x3X3g2U0"></a><code>kclf_codegen.c</code>：<code>kclf_prog_build</code></td></tr><tr><td>为什么某个内核不能运行？</td><td><a href="https://bbs.kanxue.com/elink@535K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&amp;6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3y4G2L8i4m8S2N6q4)9J5k6h3x3%60."></a><code>kclf_compat.c</code>、<a href="https://bbs.kanxue.com/elink@66eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8W2)9J5c8X3u0D9L8$3u0Q4x3V1j5$3k6o6f1K6j5K6t1&amp;6y4r3t1#2y4e0u0X3y4X3q4U0z5r3q4T1y4h3x3#2y4K6x3H3j5e0y4W2z5o6p5@1z5h3j5#2y4o6g2X3y4e0x3#2i4K6u0r3K9$3y4D9k6W2)9#2k6X3u0@1k6W2)9J5k6h3R3%60."></a><code>kclf_btf.h</code></td></tr></tbody></table>

KCLF 与 “ptrace + dlopen” 的区别，可以归结为两次职责转移：把线程控制从外部调试接口转到内核返回边界；在 manual 后端中，再把 SO 装载从 bionic 转到框架自己的 ELF 装载器。

围绕这两个变化，当前实现组合了不建立 ptrace 关系、不登记 manual 镜像、匿名映射与头部清理、按请求生成跳板、随机化临时布局、复用线程 / 栈、非专用事件完成协议，以及加载结束后的参数清理和临时映射回收。

这些措施确实改变了普通 App 能直接利用的观察面，但并没有让内存、执行现场和载荷行为消失。尤其是匿名映射可见、x16 恢复不完整、loader 集成有限和依赖生命周期需要外部保证，都是理解这个版本时不能省略的部分。

评价它时，比 “绝对隐身” 更准确的表述是：**KCLF 以更高的部署权限和更复杂的兼容性责任，换取对执行入口、装载过程与临时痕迹的更细粒度控制。**

项目源码与后续更新见：[](https://bbs.kanxue.com/elink@93eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6m8L8X3c8J5L8$3W2V1f1X3g2$3k6i4u0K6k6i4u0Q4x3X3c8f1k6i4y4@1i4K6u0r3d9@1y4x3c8R3%60.%60.)AndroidReverser-Test/KCLF。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)