> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291616.htm#msg_header_h2_9)

> [原创]ARM64 页表断点——基于 PTE 缺页异常的透明拦截、上下文捕获与寄存器实时修改

ARM64 页表断点——基于 PTE 缺页异常的透明拦截、上下文捕获与寄存器实时修改
==========================================

摘要
--

本文介绍一个基于 ARM64 PTE（Page Table Entry）权限位操控的 Linux 内核模块，通过拦截 `do_page_fault` 缺页异常实现对用户态进程的**完全透明**的指令级断点。系统支持执行、读、写三种断点类型，捕获函数调用时的全部参数（GPR x0-x7、SIMD q0-q31、栈参数），并允许在断点命中时实时修改寄存器值。整个过程对目标进程不可感知——不会产生 SIGSEGV、不会修改 VMA、不会触发任何用户态可检测的副作用。

一、项目背景
------

在 Android 游戏逆向工程中，需要在特定函数入口处拦截调用、获取参数、甚至修改参数值以影响游戏行为。常见的方案存在各种局限：

<table><thead><tr><th>方案</th><th>问题</th></tr></thead><tbody><tr><td><code>ptrace</code></td><td>性能差，易被检测（<code>TracerPid</code>），需要停止进程</td></tr><tr><td>硬件断点</td><td>ARM64 只有 4-6 个硬件断点寄存器，数量极其有限</td></tr><tr><td>Inline Hook</td><td>需要写代码段（触发 <code>mprotect</code> 或写时复制），XOM 保护下有风险</td></tr><tr><td>PLT/GOT Hook</td><td>仅限动态链接函数，无法 hook 内部函数</td></tr></tbody></table>

**本方案**利用 MMU 缺页异常机制：通过修改目标地址的 PTE 权限位制造 "违规访问"，在 `do_page_fault` 中拦截并记录上下文后恢复 PTE，CPU 重试指令成功。整个过程在 CPU 异常级别 EL1 完成，EL0 完全无感知。

二、技术架构
------

### 2.1 运行环境

<table><thead><tr><th>组件</th><th>详情</th></tr></thead><tbody><tr><td><strong>目标设备</strong></td><td>ARM64 Android 手机，已 root</td></tr><tr><td><strong>注入框架</strong></td><td>APatch (基于 KernelPatch)，加载 KPM 内核模块</td></tr><tr><td><strong>内核模块</strong></td><td><code>Kernel_prctl.kpm</code>，<code>-mgeneral-regs-only -fno-builtin</code> 编译</td></tr><tr><td><strong>用户态程序</strong></td><td><code>bp_test</code>，NDK clang++ 交叉编译，通过 <code>prctl</code> 与内核通信</td></tr><tr><td><strong>编译器</strong></td><td><code>aarch64-none-elf-gcc</code> 11.2 (KPM) / <code>aarch64-linux-android23-clang++</code> (user)</td></tr></tbody></table>

### 2.2 核心原理

ARM64 页表条目（PTE）包含多个权限控制位。系统通过修改这些位制造违规访问：

```
断点类型       PTE 操作                    CPU 异常                    ESR_EL1.EC
──────────────────────────────────────────────────────────────────────────────
执行断点       PTE_UXN = 1 (bit 54)      Instruction Abort (EL0)    0x20
读断点         PTE_USER = 0 (bit 6)      Data Abort (EL0)           0x24, WnR=0
写断点         PTE_RDONLY = 1 (bit 7)    Permission Fault (EL0)     0x24, WnR=1


```

操作流程：

```
[1] 设置断点
    bp_set() → walk_and_modify_pte() → 修改 PTE 权限位 → flush TLB
    g_bp_table[slot].active = 1

[2] 进程触发
    CPU 取指/读写 → MMU 检查 PTE → 权限违规
    → 硬件保存 EL0 上下文到 pt_regs
    → 跳转 do_page_fault(addr, esr, regs)

[3] Hook 拦截 (before_do_page_fault)
    → 匹配 g_bp_table (PID + 页地址 + 访问类型)
    → 读取 x0-x7 (pt_regs) + 栈参数 (copy_from_user) + SIMD q0-q31 (stp 汇编)
    → 应用寄存器修改 (ins/ldr 汇编 + pt_regs 写回)
    → 恢复原始 PTE (写预映射地址)
    → g_bp_table[slot].active = 2

[4] 原始 do_page_fault 执行
    → PTE 已正常 → 返回 0

[5] CPU 重试指令
    → 使用修改后的寄存器 → 函数正常执行

[6] 用户态轮询
    PRCTL_BP_QUERY → 获取命中信息 → 重新 PRCTL_BP_SET (re-arm)


```

三、开发过程
------

### 3.1 版本迭代

整个项目经历了三个主要阶段：

**V1：阻塞模式 + 寄存器修改**

*   断点命中后 `before_do_page_fault` 进入 `schedule_timeout` 忙等
*   用户态在此期间读取寄存器并决定修改值
*   问题：阻塞导致游戏卡死、性能差、逻辑复杂

**V2：纯 One-Shot 非阻塞模式**

*   命中后只记录寄存器快照，PTE 立即恢复
*   用户态轮询查询结果
*   引入 Sticky Page 机制防止无限缺页循环
*   分支：`brk-failed` (V1 备份) / `stable` (V2+)

**V3：非阻塞 + 寄存器修改 (当前版本)**

*   在 V2 基础上新增 `PRCTL_BP_MODIFY`
*   用户态预先设置修改数据 → 断点命中时内核直接应用
*   支持实时更新：用户态循环调用 `PRCTL_BP_MODIFY`，每次命中使用最新值
*   支持同时修改多个 GPR (x0-x7)、SIMD (v0-v7)、SP

### 3.2 分支结构

```
* b0d7c59 master, stable — V3: one-shot + 寄存器修改 (PRCTL_BP_MODIFY)
* 6283514              — V2: 纯 one-shot，移除阻塞模式
* c1e0d1b brk-failed   — V1: 阻塞模式 + 寄存器修改 (备份)


```

四、关键数据结构
--------

### 4.1 用户 - 内核通信协议

所有通信通过 `prctl()` 系统调用完成（内核模块 hook 了 `prctl` 的 `before` 回调）：

```
// 设置断点
struct bp_request {
    pid_t    target_pid;
    uint64_t vaddr;
    uint32_t bp_type;   // BP_TYPE_EXEC(4) / READ(1) / WRITE(2)
    int32_t  bp_id;     // 输出: 分配的断点 ID (0-31)
};

// 查询命中
struct bp_hit_info {
    int32_t  bp_id;
    uint64_t vaddr, hit_pc, hit_esr;
    uint64_t x_regs[8];       // x0-x7 整数参数
    uint64_t v_regs[64];      // v0-v31: [N*2]=lo64, [N*2+1]=hi64
    uint64_t sp;              // 栈指针
    uint64_t stack_args[6];   // 栈参数 a13-a18
    int32_t  active;
};

// 寄存器修改
struct bp_modify {
    int32_t  bp_id;
    uint16_t x_mask;          // bit[r]=1 → 修改 x_r
    uint16_t v_mask;          // bit[r]=1 → 修改 v_r
    uint8_t  v_mode;          // LO32(0) / LO64(1) / Q128(2)
    uint8_t  mod_sp;          // 0=不改, 1=修改
    uint64_t x_regs[8];       // x0-x7 新值
    uint64_t v_lo[8];         // v0-v7 低 64 位新值
    uint64_t v_hi[8];         // v0-v7 高 64 位新值 (Q128 用)
    uint64_t new_sp;
};


```

### 4.2 内核断点表

```
#define BP_MAX_COUNT 32

struct mmu_breakpoint {
    pid_t    target_pid;
    uint64_t vaddr;
    uint32_t bp_type;
    uint64_t orig_pte;        // 保存原始 PTE
    uint64_t pte_table_kva;   // PTE 表预映射 KVA (避免 hook 内 memremap)
    int      pte_index;       // 表内索引

    // 命中快照
    uint64_t hit_pc, hit_esr, hit_sp;
    uint64_t hit_x_regs[8];
    uint64_t hit_v_regs[64];
    uint64_t hit_stack_args[6];
    int      active;          // 0=空闲, 1=armed, 2=triggered

    // 修改缓存 (PRCTL_BP_MODIFY 实时更新)
    uint16_t mod_x_mask, mod_v_mask;
    uint8_t  mod_v_mode, mod_sp, mod_ready;
    uint64_t mod_x_regs[8], mod_v_lo[8], mod_v_hi[8], mod_new_sp;
};


```

五、核心技术实现
--------

### 5.1 页表遍历与 PTE 修改

手动遍历目标进程的页表（PGD → PUD → PMD → PTE），直接操作物理内存：

```
Level 0 (PGD):  idx = (vaddr >> 30) & 0x1FF   // 39-bit VA, 3 级页表
Level 2 (PMD):  idx = (vaddr >> 21) & 0x1FF   // PUD 折叠
                block mapping? → 直接修改 block descriptor
Level 3 (PTE):  idx = (vaddr >> 12) & 0x1FF
                new_pte = (orig_pte & ~clear_mask) | set_mask


```

**关键设计**：

*   PGD 使用直接内核虚拟地址（线性映射，无需 `memremap`）
*   子表通过 `memremap(WB)` 映射物理页
*   支持 2MB block mapping（PMD 级别），不支持 1GB block（PUD 级别，太罕见）
*   运行态探测页表层级（3 级 vs 4 级），通过检查 init 进程 PGD[1] 条目

**权限掩码计算**：

```
执行断点: set_mask |= PTE_UXN    // bit 54, 禁止 EL0 执行
读断点:   clear_mask |= PTE_USER // bit 6,  EL0 无访问权限
写断点:   set_mask |= PTE_RDONLY // bit 7,  标记只读


```

### 5.2 do_page_fault Inline Hook

使用 KernelPatch 的 `hook_wrap3` 框架在 `do_page_fault` 前插入回调：

```
static void before_do_page_fault(hook_fargs3_t *args, void *udata) {
    unsigned long fault_addr = (unsigned long)args->arg0;
    unsigned int  esr        = (unsigned int)args->arg1;
    struct pt_regs *regs     = (struct pt_regs *)args->arg2;

    unsigned int ec = (esr >> 26) & 0x3F;  // Exception Class

    // 仅处理 EL0 异常
    if (ec != 0x20 && ec != 0x24) return;  // IABT_EL0 / DABT_EL0

    // 确定访问类型
    int hit_type = (ec == 0x20) ? BP_TYPE_EXEC
                 : (esr & (1<<6)) ? BP_TYPE_WRITE : BP_TYPE_READ;
    ...
}


```

**`do_page_fault` 签名**：

```
void do_page_fault(unsigned long addr, unsigned int esr, struct pt_regs *regs)


```

安装和卸载：

```
// init
g_bp_fault_func = kallsyms_lookup_name("do_page_fault");
hook_wrap3((void *)g_bp_fault_func, before_do_page_fault, NULL, 0);

// exit
hook_unwrap(g_bp_fault_func, before_do_page_fault, NULL);


```

### 5.3 SIMD 寄存器读取

ARM64 内核模块使用 `-mgeneral-regs-only` 编译，内核自身绝不使用 NEON/SIMD 寄存器。因此在 `do_page_fault` 处理期间（EL1），硬件 FPSIMD 寄存器仍保持 EL0 用户态进程的值。

通过内联汇编 `stp`（Store Pair）一次性 dump 全部 32 个 Q 寄存器：

```
uint64_t buf[64] __attribute__((aligned(16)));
asm volatile(
    "stp q0,  q1,  [%0, #0]\n\t"
    "stp q2,  q3,  [%0, #32]\n\t"
    "stp q4,  q5,  [%0, #64]\n\t"
    "stp q6,  q7,  [%0, #96]\n\t"
    "stp q8,  q9,  [%0, #128]\n\t"
    // ... 至 q31
    "stp q30, q31, [%0, #480]\n\t"
    :: "r"(buf) : "memory"
);
// 布局: buf[0]=q0.lo, buf[1]=q0.hi, buf[64]=q31.lo, buf[63]=q31.hi


```

每条 `stp` 存储 2 个 128-bit Q 寄存器 (32 字节)，16 条指令覆盖 q0-q31。

### 5.4 寄存器修改

修改原理：`pt_regs` 中的 GPR 写回即生效；SIMD 硬件寄存器直接通过内联 asm 写入。

**GPR 修改**：

```
regs->regs[r] = g_bp_table[i].mod_x_regs[r];


```

**SIMD 修改**（三种模式）：

```
// LO32 — 仅低 32 位 (float)
asm volatile("ins vN.s[0], %w0" :: "r"(val_lo32) : "vN");

// LO64 — 仅低 64 位 (double)
asm volatile("ins vN.d[0], %0"  :: "r"(val_lo64) : "vN");

// Q128 — 全部 128 位
asm volatile("ldr qN, [%0]"     :: "r"(&buf)    : "qN", "memory");


```

`ins` 指令只覆盖目标位域，其余位保持不变。`ldr` 从对齐缓冲区加载完整 128 位。

### 5.5 PTE 预映射优化

`do_page_fault` hook 上下文不可调用 `memremap()`/`memunmap()`（可能触发睡眠）。解决方案：

*   **设置时**（prctl 上下文，安全）：`memremap` PTE 表页，存储 KVA + 索引到 `pte_table_kva` / `pte_index`
*   **命中时**（fault 上下文，不安全）：直接 `pte_table_kva[pte_index] = orig_pte`（纯内存写）

```
// bp_set() — prctl 上下文
g_bp_table[slot].pte_table_kva = (uint64_t)memremap(phys_addr, PAGE_SIZE, MEMREMAP_WB);
g_bp_table[slot].pte_index = pte_idx;

// before_do_page_fault — 缺页上下文
uint64_t *pte = (uint64_t *)g_bp_table[i].pte_table_kva;
pte[g_bp_table[i].pte_index] = g_bp_table[i].orig_pte;


```

### 5.6 Sticky Page 机制

**问题**：One-shot 模式下，PTE 恢复后 CPU 重试指令。若此时立即 `bp_set` re-arm，PTE 又被修改，CPU 还在同一页执行 → 再次触发缺页 → 无限循环。

**方案**：引入 Sticky Page 状态机：

```
断点命中 → 设置 sticky(page, pid, slot)
          → 恢复 PTE
          → active = 2 (triggered)

同页再次缺页 → 检测 sticky 匹配
             → 静默恢复 PTE (不记录命中, 不改 active)
             → return  (让游戏继续跑)

游戏离开该页 → 新页缺页
             → 检测 sticky 不匹配
             → 重新修改旧页 PTE (re-arm)
             → 清除 sticky
             → 处理新缺页正常流程


```

### 5.7 线程安全

**mod_ready 标志**：用户态可能随时调用 `PRCTL_BP_MODIFY` 更新修改数据，与 `do_page_fault` 形成竞态。

```
Writer (PRCTL_BP_MODIFY):           Reader (before_do_page_fault):
  mod_ready = 0;                      if (!mod_ready) skip;
  compiler_barrier();                 compiler_barrier();
  写入所有 mod_* 字段                  使用 mod_* 数据
  compiler_barrier();
  mod_ready = 1;


```

ARM64 对齐字段的单拷贝原子性保证不会读到撕裂值。Reader 看到 `mod_ready=0` 则跳过本次修改（安全默认），下次命中生效。

六、难点与解决方案
---------

### 6.1 sp=0 bug — 数据拷贝遗漏

**现象**：`bp_hit_info.sp` 始终为 0。

**根因**：`PRCTL_BP_QUERY` 的 `copy_to_user` 逻辑只复制了 header（到 `x_regs[7]`）+ `v_regs[64]`。而 `sp`、`stack_args[6]`、`active` 这三个字段在 `v_regs` **之后**，从未被拷贝到用户态。

```
// 修复前 (只拷贝 header + v_regs)
int hdr_sz = offsetof(bp_hit_info, v_regs);
copy_to_user(buf, &info, hdr_sz);
for (c = 0; c < 64; c += 16) copy_to_user(buf+hdr_sz+c*8, &info.v_regs[c], 128);
// sp, stack_args, active — 丢失!

// 修复后 (追加尾部拷贝)
int trail_off = offsetof(bp_hit_info, sp);
int trail_sz  = sizeof(info) - trail_off;
copy_to_user(buf + trail_off, &info.sp, trail_sz);


```

### 6.2 TLS 段对齐错误

**现象**：编译 `bp_test` 时如果使用 `-static`，运行时报错：

```
executable's TLS segment is underaligned: alignment is 8,
needs to be at least 64 for ARM64 Bionic


```

**原因**：ARM64 Android Bionic libc 要求 TLS（Thread Local Storage）段 64 字节对齐，静态链接下 LLD 无法保证。

**解决**：移除 `-static`，使用动态链接即可。

### 6.3 大页不支持

Block mapping（2MB）在 PMD 级别可以处理（直接修改 block descriptor），但 1GB PUD block 罕见且不支持，`walk_and_modify_pte` 返回 `-1`。

### 6.4 每页一个断点

同一 4KB 页只允许一个活跃断点。原因是 PTE 修改是页级别的——同一页上的两个地址共享同一个 PTE，无法独立控制。

### 6.5 mprotect 检测规避

`/proc/pid/maps` 显示的权限来自 VMA（`vm_area_struct`），而非 PTE。本系统只修改 PTE，VMA 不变，所有用户态页权限检测（`mprotect`、`/proc/maps`、`mincore`）均不可见。

七、性能与限制
-------

<table><thead><tr><th>指标</th><th>数值</th></tr></thead><tbody><tr><td>断点槽位上限</td><td>32 个全局</td></tr><tr><td>每页断点限制</td><td>1 个</td></tr><tr><td>PTE 修改开销</td><td>~1μs（一次 <code>memremap</code> + 写 PTE）</td></tr><tr><td>缺页处理延迟</td><td>~5μs（hook 内读取寄存器 + 恢复 PTE + TLB flush）</td></tr><tr><td>TLB flush 范围</td><td>本地（<code>tlbi vmalle1</code>），不影响其他 CPU</td></tr><tr><td>SIMD 修改范围</td><td>v0-v7（函数参数寄存器）</td></tr><tr><td>支持页表深度</td><td>3 级 (39-bit VA) / 4 级 (48-bit VA)</td></tr></tbody></table>

### 限制

*   大页（2MB PTE block）+ EXEC 断点：修改 block descriptor 会影响整个 2MB 范围的所有代码
*   同一页上的不同指令需共享断点（同一 PTE）
*   `do_page_fault` 签名依赖，不同内核版本可能参数顺序不同
*   仅支持 EL0 异常（用户态），不支持 EL1 内核态断点

八、使用示例
------

### 8.1 设置执行断点并读取参数

```
#include "kernel.h"

c_driver *drv = new c_driver();
uint64_t addr = drv->getModuleBase(pid, "libUE4.so") + 0x5C7C834;

// 设置断点
bp_request req = {pid, addr, BP_TYPE_EXEC, -1};
int bp_id = drv->setBreakpoint(&req);

// 轮询命中
bp_hit_info info;
if (drv->queryBreakpoint(bp_id, &info) == 0) {
    // 查看参数
    printf("x0=%llx, v0.s[0]=%f\n",
           info.x_regs[0],
           *(float*)&info.v_regs[0]);

    // 重新设置
    drv->setBreakpoint(&req);
}


```

### 8.2 修改浮点参数

```
// 修改 v3.s[0]=1.0f, v4.s[0]=1.0f
bp_modify mod = {};
mod.bp_id  = bp_id;
mod.v_mask = (1u << 3) | (1u << 4);
mod.v_mode = BP_MOD_SIMD_LO32;
mod.v_lo[3] = 0x3F800000;  // 1.0f
mod.v_lo[4] = 0x3F800000;  // 1.0f
drv->modifyRegs(&mod);


```

### 8.3 实时更新修改值

```
while (running) {
    mod.v_lo[3] = get_dynamic_value();  // 实时变化
    drv->modifyRegs(&mod);

    bp_hit_info info;
    if (drv->queryBreakpoint(bp_id, &info) == 0) {
        // 命中：原值在 info.v_regs，修改值生效于游戏
        drv->setBreakpoint(&req);
        mod.bp_id = bp_id;
    }
    usleep(10000);
}


```

九、总结
----

本文介绍的 MMU 断点系统通过 ARM64 PTE 权限位操控实现了对用户态进程的完全透明拦截。核心优势在于：

1.  **完全透明**：不修改 VMA、不触发信号、不使用 `ptrace`，对进程不可见
2.  **数量不限**：理论上支持任意多断点（受 PTE 表大小和每页限制约束）
3.  **完整上下文**：捕获全部 GPR、SIMD、栈参数
4.  **寄存器修改**：支持实时、多寄存器的参数篡改
5.  **低延迟**：单次命中 ~5μs kernel overhead

该项目从初始的阻塞模式，经过简化到纯 one-shot，最终加入寄存器修改功能，形成了一套完整的调试 / 修改框架。代码以 KernelPatch KPM 模块形式运行于 APatch 框架，编译于 `aarch64-none-elf-gcc` + `-mgeneral-regs-only` 工具链。

完整的代码和使用指南见 [user-test/](../user-test/) 和 [kpms/prctlhookRWMemoryNew/](../kpms/prctlhookRWMemoryNew/)。

附录 A：PTE 断点检测方式分析
-----------------

### A.1 时序检测

通过测量代码执行延迟来判断是否有断点。

**正常 vs 断点的延迟对比：**

<table><thead><tr><th>场景</th><th>延迟</th><th>倍数</th></tr></thead><tbody><tr><td>TLB 热命中（正常）</td><td>1-4 cycles</td><td>基准</td></tr><tr><td>TLB 冷命中（正常缺 TLB）</td><td>100-200 cycles</td><td>~50×</td></tr><tr><td>断点触发（缺页异常路径）</td><td>5000-20000+ cycles</td><td>50-200× vs 冷 TLB</td></tr></tbody></table>

断点触发路径的额外开销来自：CPU 保存 EL0 上下文 → 向量表跳转 → 栈切换 → 内核 `do_page_fault` → hook 处理（匹配断点表、dump SIMD、恢复 PTE、TLB flush） → ERET 回 EL0 → CPU 重试 → TLB miss（刚被 flush）。

**检测方法**（需 Android 内核开启 PMU 用户态访问 `PMUSERENR_EL0.EN`）：

```
// ARM64 周期计数器 (需要内核启用 PMU 用户态访问)
static inline uint64_t read_pmccntr(void) {
    uint64_t val;
    asm volatile("mrs %0, PMCCNTR_EL0" : "=r"(val));
    return val;
}

// 对照相邻页测量
uint64 t1 = read_pmccntr();
target_code(args);             // 被测代码（可能在断点监控下）
uint64 t2 = read_pmccntr();

neighbor_code(args);           // 相邻页代码（不受断点影响）
uint64 t3 = read_pmccntr();

if ((t2 - t1) > (t3 - t2) * 10)  // 目标页延迟异常
    → 疑似有断点


```

**实际困难**：One-shot 断点仅第一次触发慢，之后即正常；移动设备上的系统噪声（调度抖动、中断、DVFS 频率调节、thermal throttling）远大于 5μs 的 hook 延迟；冷 TLB 误报难以区分。在 Android 上 timing 检测的可靠性大幅降低。

### A.2 Minor Page Fault 计数

每次断点命中都是一次 minor page fault。`/proc/pid/status` 中的 `min_flt` 计数器可以通过对比正常与异常时期的增量变化来检测异常频率。但噪声极大（正常缺页也计数），仅作为弱信号。

### A.3 PMU 异常事件计数

ARM64 PMU 可统计异常事件，但 Android 内核默认不将指令异常事件暴露给 EL0（`PMUSERENR_EL0` 控制），需内核模块配合才能从用户态读取。

### A.4 检测矩阵

<table><thead><tr><th>检测方式</th><th>可行性</th><th>难度</th><th>对 one-shot PTE 的影响</th></tr></thead><tbody><tr><td>Timing (cycle counter)</td><td>理论可行</td><td>高（单次噪声大，需统计）</td><td>低（窗口极小，一次慢）</td></tr><tr><td>Timing (对照相邻页)</td><td>较可靠</td><td>中（需注入代码）</td><td>低</td></tr><tr><td><code>/proc/pid/status</code> minor fault</td><td>弱信号</td><td>低（噪音多）</td><td>极低</td></tr><tr><td>PMU exception count</td><td>较强</td><td>高（需内核配合）</td><td>中</td></tr><tr><td>PTE 异常扫描</td><td><strong>Android 上不可行</strong></td><td>—（GKI + SELinux 限制）</td><td>无</td></tr><tr><td><strong><code>/proc/pid/maps</code></strong></td><td><strong>不可行</strong></td><td>—</td><td><strong>无</strong>（VMA 未修改）</td></tr><tr><td><strong>SIGSEGV handler</strong></td><td><strong>不可行</strong></td><td>—</td><td><strong>无</strong>（信号不送达）</td></tr><tr><td><strong>mprotect 检查</strong></td><td><strong>不可行</strong></td><td>—</td><td><strong>无</strong>（PTE ≠ VMA）</td></tr></tbody></table>

附录 B：反作弊检测寄存器修改的手段
------------------

以下聚焦**纯本地、纯客户端**层面的检测。假设只改本地数据（渲染、动画、客户端预测），服务端不参与校验。

### B.1 直接数值校验

**值域 / 边界检查**：修改的参数若超出物理 / 逻辑合法范围直接暴露。

```
正常移动速度: 0 ~ 600
改成 999999 → 立即触发


```

**双份存储比对 (Shadow Copy)**：引擎对关键数据结构维护两份拷贝。

```
static float speed;         // 运行时使用
static float speed_shadow;  // 影子备份，不参与实际逻辑

if (fabs(speed - speed_shadow) > EPSILON)
    → 检测到篡改，上报反作弊


```

只改了寄存器，内存中的 shadow 副本未同步 → 下一轮检查暴露。

**CRC / Checksum**：某游戏安全 SDK 在 UE4 中插入大量 `CheckIntegrity()` 调用，对 `ACharacter`、`AController`、`UWorld` 等关键结构体做周期性 hash。任何外部内存写入都会被检测。

### B.2 一致性校验

同一帧内多个相互关联的参数必须自洽。

```
位置变化 vs 速度:
  位移 / deltaTime 必须 ≤ 速度
  把速度改成 0，但位移正常 → 矛盾

血量变化:
  200 → 198 → 200  (未经治疗的自动恢复) → 矛盾

状态机:
  IDLE → JUMPING  (跳过了 MOVING) → 非法状态转换


```

### B.3 浮点指纹检测

**NaN / Inf 注入**：错误的 SIMD 写入可能产生非数值。

```
v_mode 选错 → 高位残留垃圾
→ reinterpret_cast<double> 产生 NaN
→ 任何后续运算传播 NaN
→ 反作弊检查: isnan(any_float_result) → 标记异常


```

**精度模式**：`ins vN.s[0]` 仅写低 32 位 (float)，高 96 位保持不变。如果原本的 qN 高位不是干净的（例如之前被用作 double 或 int64），残留值可被检测。

**FPSR 异常标志**：ARM64 FPSR 寄存器记录浮点运算异常（IDC/IXC/UFC/OFC/DZC/IOC）。强反作弊周期性读取并清零 FPSR，若在不应该有浮点异常的代码段中出现异常标志 → 有外部干扰。

### B.4 ARM64 寄存器状态检测

**SP 对齐**：ARM64 ABI 要求 SP 必须 16 字节对齐。通过 `mod_sp` 修改 SP 若未保证对齐 → `ldp`/`stp` 触发 SP alignment fault → 被信号处理器捕获。

**NZCV 标志位**：修改 GPR 不直接影响条件码。但若修改了会触发条件跳转的关键值（如修改 x0 后紧接着 `cbz x0`），执行流可能偏离预期路径，间接导致崩溃。

### B.5 帧时间分析

单个断点 5μs 可忽略。但如果同时有多个断点，或 hook 了 per-frame 高频函数（每帧调用数百次），累积延迟可超过 1ms。Android 上可通过 `Choreographer` / `FrameMetrics` API 或 UE4 内置的 `Stats` 系统检测帧时间异常。

### B.6 UE4 内置检测

*   **`check()/ensure()` 断言**：遍布引擎代码，参数不合理直接触发（如 `check(DeltaSeconds > 0 && DeltaSeconds < 1.0)`）
*   **`FPlatformMisc::IsDebuggerPresent()`**：检查 `/proc/self/status` 的 TracerPid
*   **全局指针合法性**：`GWorld`、`GEngine` 等指针若指向非内核态合法地址 → 异常

### B.7 环境完整性检测

Android 上的反作弊会在**用户态**扫描运行环境：

*   Root / Magisk / APatch 框架特征（文件系统扫描、挂载点检查）
*   SELinux 状态（Permissive = 高风险环境）
*   可疑进程、可疑文件句柄
*   Bootloader unlock 状态
*   `/proc/self/maps` 中的可疑模块

这些检测的是**注入框架本身**而非寄存器修改行为，但环境异常会触发整体的安全标记。注意：Android GKI 限制下反作弊内核驱动的能力有限，上述检测主要在用户态执行。

### B.8 检测手段风险矩阵

<table><thead><tr><th>检测手段</th><th>风险</th><th>本项目暴露面</th></tr></thead><tbody><tr><td>值域校验 (边界 / NaN)</td><td><strong>中</strong></td><td>v_mode 选错致高位垃圾，或值超出合法范围</td></tr><tr><td>双份存储比对 (shadow)</td><td><strong>高</strong></td><td>只改了寄存器，内存 shadow 未同步</td></tr><tr><td>CRC / Checksum</td><td><strong>高</strong></td><td>同上，引擎周期性 hash 检测</td></tr><tr><td>一致性校验</td><td><strong>中</strong></td><td>修改单一参数导致关联参数不自洽</td></tr><tr><td>帧间连续性</td><td><strong>中</strong></td><td>硬赋值导致跳变而非渐变</td></tr><tr><td>FPSR 异常标志</td><td>低</td><td><code>ins</code>/<code>ldr</code> 不产生浮点运算，FPSR 不受影响</td></tr><tr><td>NZCV 污染</td><td>无</td><td>不改条件码</td></tr><tr><td>SP 对齐</td><td>低</td><td>不改 SP 或确保 16 对齐即安全</td></tr><tr><td>帧 timing</td><td>低</td><td>one-shot + 低频 hook，5μs 可忽略</td></tr><tr><td>环境扫描</td><td><strong>高</strong></td><td>框架本身 (APatch/KPM) 需加固隐藏</td></tr></tbody></table>

### B.9 安全修改的原则

1.  **选择纯计算中间结果**：改不会被写回内存的临时值（寄存器中的计算中间值），避免 shadow copy 不一致
2.  **值在合法边界内**：改 599 而不是 999999；用渐变函数而不是硬赋值
3.  **同步关联参数**：改速度也改位移；改伤害也改护甲扣减和血量变化
4.  **保持帧间连续性**：用 lerp/smooth 逐步逼近目标值
5.  **避免服务端权威数据**：只改纯客户端侧（渲染矩阵、FOV、本地动画）
6.  **v_mode 选对**：LO32 写 float 时确保高位干净；Q128 时用完整 16 字节
7.  **若需要改内存 shadow**：配合 `writeMem()` 同步修改内存中的双份拷贝
8.  **条件性下断点，缩小窗口**：不要一直保持断点 armed。用 `readMem()` 持续读取游戏状态（如血量、位置、目标距离），只在满足前置条件时才 `bp_set`，命中后立即处理 + re-arm 或清除。PTE 异常仅存在于 "前置条件满足 → 函数被调用" 的极短间隙中，将暴露窗口从 "永久" 压缩到毫秒级。

```
// 不好的做法: 一直 armed
bp_set(&req);  // 游戏一启动就下断点，永久等待

// 好的做法: 条件触发
while (running) {
    float distance = drv->read<float>(enemy_distance_addr);
    if (正在开枪 < 100.0f) {  // 当前人物状态/joystick按键等判断开火
        bp_set(&req);          // 下断点
        // 修改其中某个函数，修改弹道朝向指定方向
        drv->clearBreakpoint(bp_id);  // 立即清除，不保持 armed
    }
    usleep(10000);
}


```

[[招生] 科锐逆向工程师培训 (2026 年 7 月 3 日实地，远程教学同时开班, 第 56 期)！](https://bbs.kanxue.com/thread-51839.htm)

[#基础理论](forum-161-1-117.htm) [#源码框架](forum-161-1-127.htm) [#系统相关](forum-161-1-126.htm) [#HOOK 注入](forum-161-1-125.htm)