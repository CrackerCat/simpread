> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xtuly.cn](https://xtuly.cn/article/arm64-kernel-research-1)

> 实现一个劫持内核内 BL 指令跳转到自身模块函数执行的简单 inline hook

实现一个劫持内核内 BL 指令跳转到自身模块函数执行的简单 inline hook

本篇将实现一个劫持内核内 BL 指令跳转到自身模块函数执行的简单 inline hook。

[](#cb56e850a2b24d1696db2856ec091128 "ARM64 指令基础")ARM64 指令基础
------------------------------------------------------------

### [](#71b54522e90f494ab6d45872906dbdde "BL 指令")BL 指令

跳转到相对于 PC 的指定地址，并将下一条指令地址存入 LR 寄存器。

跳转范围：±128MB

#### [](#7f9374f8416f4969b5f2387fd0877020 "指令格式")指令格式

```
31 | 30 29 28 27 26 | 25 ... 0
1  | 0  0  1  0  1  |  imm2631 | 30 29 28 27 26 | 25 ... 0
1  | 0  0  1  0  1  |  imm26

```

#### [](#a02f3c12b30045ca879deca8b2226aea "地址编码")地址编码

imm26 负数使用补码表示

> 正数和 0 的补码就是该数字本身再补上符号位 0。负数的补码则是将其对应正数按位取反再加 1。

可能有人会有疑惑，明明立即数只有 26 位，跳转范围应该是 ±32MB 才对，为什么会是 ±128MB 呢？

因为 arm64 指令长度都是 4 字节，所以编码地址的时候除了 4，比如跳转 0x4，imm26 是 1

#### [](#ed403596076e4d0989a6f1f117cdb3ea "简单例子")简单例子

```
BL 0x4
0b100101 00000000000000000000000001

BL -0x4
0b100101 11111111111111111111111111BL 0x4
0b100101 00000000000000000000000001

BL -0x4
0b100101 11111111111111111111111111

```

#### [](#237ae8511fdc4c268f9ceeac76856e9a "使用keystone生成机器码")使用 keystone 生成机器码

```
from keystone import *
import struct
ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
code, count = ks.asm("BL 0x4")
code = struct.unpack("<I", bytes(code))[0]
print(bin(code))

```

> 以上代码输出： 0b10010100000000000000000000000001

#### [](#559445ad37714bfebd333188b307276d "模拟运行一下试试吧")模拟运行一下试试吧

```
from capstone import *
from keystone import *
from unicorn import *

test_code = """NOP
mov x0, #1
b 0x14
add x0, x0, #1
add x0, x0, #1
add x0, x0, #1
add x0, x0, #1
"""


def hook_code(_mu, address, size, user_data):
    instruction = _mu.mem_read(address, size)
    
    
    for i in cs.disasm(instruction, 0x0):
        print('[0x%08X] %s\t%s' % (address, i.mnemonic, i.op_str))


ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
code, code_count = ks.asm(test_code)

mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
mu.mem_map(0x0, 0x1000, UC_PROT_ALL)
mu.mem_write(0x0, bytes(code))
mu.hook_add(UC_HOOK_CODE, hook_code)
mu.emu_start(0x0, code_count * 4)

```

> 输出如下
> 
> [0x00000000] nop [0x00000004] movz x0, #0x1 [0x00000008] b #0xc [0x00000014] add x0, x0, #1 [0x00000018] add x0, x0, #1

[](#aff8016c84ee44d38dd9b6998ac8d301 "相关内核基础")相关内核基础
----------------------------------------------------

### [](#5fb1d7c9805a403f864ad13c08223bbb "内核分段及权限设定")内核分段及权限设定

#### [](#295d5d2cd9e940109a845b1fd39ad7b2 "一、分了哪些段")一、分了哪些段

见 arch/arm64/kernel/vmlinux.lds.S

*   _text → _etext 之间是代码段

*   __start_rodata → __end_rodata 之间是只读数据段

*   __init_begin → __init_end 之间是内核初始化相关的段，包括代码和数据

*   _data → _end 之间是可读可写的数据段

#### [](#1ee6e88e20154896ad49691e89b49812 "二、各个段的权限设定")二、各个段的权限设定

见 arch/arm64/mm/mmu.c 的 map_kernel 函数

```
pgprot_t text_prot = rodata_enabled ? PAGE_KERNEL_ROX : PAGE_KERNEL_EXEC;
map_kernel_segment(pgdp, _text, _etext, text_prot, &vmlinux_text, 0,
			   VM_NO_GUARD);
map_kernel_segment(pgdp, __start_rodata, __inittext_begin, PAGE_KERNEL,
			   &vmlinux_rodata, NO_CONT_MAPPINGS, VM_NO_GUARD);
map_kernel_segment(pgdp, __inittext_begin, __inittext_end, text_prot,
			   &vmlinux_inittext, 0, VM_NO_GUARD);
map_kernel_segment(pgdp, __initdata_begin, __initdata_end, PAGE_KERNEL,
			   &vmlinux_initdata, 0, VM_NO_GUARD);
map_kernel_segment(pgdp, _data, _end, PAGE_KERNEL, &vmlinux_data, 0, 0);

```

```
#define _PROT_DEFAULT		(PTE_TYPE_PAGE | PTE_AF | PTE_SHARED)
#define PROT_DEFAULT		(_PROT_DEFAULT | PTE_MAYBE_NG)
#define PROT_NORMAL		(PROT_DEFAULT | PTE_PXN | PTE_UXN | PTE_WRITE | PTE_ATTRINDX(MT_NORMAL))
#define PAGE_KERNEL		__pgprot(PROT_NORMAL)
#define PAGE_KERNEL_ROX		__pgprot((PROT_NORMAL & ~(PTE_WRITE | PTE_PXN)) | PTE_RDONLY)
#define PAGE_KERNEL_EXEC	__pgprot(PROT_NORMAL & ~PTE_PXN)

```

#### [](#7bd1525908514d5c9655020bd976bab1 "三、结论")三、结论

*   代码段：_text → _etext

rodata_enabled ? PAGE_KERNEL_ROX : PAGE_KERNEL_EXEC;

可读特权模式可执行，rodata_enabled 为假时可写

*   只读数据段

PAGE_KERNEL 可读可写不可执行

*   初始化代码段：

rodata_enabled ? PAGE_KERNEL_ROX : PAGE_KERNEL_EXEC; 可读特权模式可执行，rodata_enabled 为假时可写

*   初始化数据段：

PAGE_KERNEL 可读可写不可执行

*   数据段：

PAGE_KERNEL 可读可写不可执行

### [](#686fe89e41b24b1f9e2b42d56fa24770 "寻找对应符号地址")寻找对应符号地址

开了 KASLR 怎么办？摆！

#### [](#91f0babf44fc4244a178ff314302ab9b "方法一：kallsyms文件")方法一：kallsyms 文件

解除内核符号限制

```
echo 0 > /proc/sys/kernel/kptr_restrict

```

> 部分设备需要 echo 1 才行

获取符号地址

```
cat /proc/kallsyms | grep xxxxx

```

#### [](#a9406fdd1d0944dd9e143310997b4c54 "方法二：kprobe大法 (*推荐)")方法二：kprobe 大法 (* 推荐)

上代码：

```
uintptr_t kprobe_get_addr(const char *symbol_name) {
    int ret;
    struct kprobe kp;
    uintptr_t tmp = 0;
    kp.addr = 0;
    kp.symbol_name = symbol_name;
    ret = register_kprobe(&kp);
    tmp = kp.addr;
    if (ret < 0) {
        goto out; 
    }
    unregister_kprobe(&kp);
out:
    return tmp;
}

```

底层原理：

使用 kallsyms_lookup_name 解析符号地址

> 需高版本内核！

#### [](#94c0cd63421b4bdaae08c736e5318459 "方法三：kallsyms_lookup_name函数")方法三：kallsyms_lookup_name 函数

如果该函数导出，可直接使用该函数定位，但是大部分内核中该函数并未导出。

#### [](#dccb49efd59b4566ba74c3520b499e63 "方法四：奇门遁甲")方法四：奇门遁甲

1.  将内核丢进 IDA 分析，依靠字符串和源码慢慢寻找位置

2.  特征码定位

通过一些特征汇编代码定位

3.  根据导出函数，结合偏移辅助定位

比如 &printk - offsetof(printk) + offsetof(foo)

[](#566488262f4843dfa2774aa4a19112a8 "让我们开始吧")让我们开始吧
----------------------------------------------------

### [](#133e215f96c1432181f6c093de1b8702 "绕过内核只读限制")绕过内核只读限制

#### [](#1215246fdb064197a1345adaae2cecd7 "方法一")方法一

修改内核，将 rodata_enabled 改为 0

优点：简单方便，快捷高效

缺点：安全性降低

评价：开发机要什么安全，方便就完了！

#### [](#12543f4e38c54427acf9a07fb9c25cb4 "方法二")方法二

修改 pte，给权限加上可写

详见下面代码

### [](#bd6d8570b39745e182e16c811c67d07c "开始写hook咯")开始写 hook 咯

目标：__arm64_sys_faccessat 的 BL do_faccessat

目的：`/memfd:` 今天必须给我存在

```
__arm64_sys_faccessat

var_s0          =  0

    HINT            #0x19
    STR             X30, [X18],#8
    STP             X29, X30, [SP,#-0x10+var_s0]!
    MOV             X29, SP
    LDR             W8, [X0]
    LDR             X1, [X0,#8]
    LDR             W2, [X0,#0x10]
    MOV             W3, WZR
    MOV             W0, W8
    BL              do_faccessat
    LDP             X29, X30, [SP+var_s0],#0x10
    LDR             X30, [X18,#-8]!
    HINT            #0x1D
    RET
; End of function __arm64_sys_faccessat__arm64_sys_faccessat

var_s0          =  0

    HINT            #0x19
    STR             X30, [X18],#8
    STP             X29, X30, [SP,#-0x10+var_s0]!
    MOV             X29, SP
    LDR             W8, [X0]
    LDR             X1, [X0,#8]
    LDR             W2, [X0,#0x10]
    MOV             W3, WZR
    MOV             W0, W8
    BL              do_faccessat
    LDP             X29, X30, [SP+var_s0],#0x10
    LDR             X30, [X18,#-8]!
    HINT            #0x1D
    RET
; End of function __arm64_sys_faccessat

```

代码：

```
#include <linux/cpu.h>
#include <linux/memory.h>
#include <linux/uaccess.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/printk.h>
#include <linux/string.h>
#include <asm-generic/errno-base.h>

#ifdef pr_fmt
#undef pr_fmt
#define pr_fmt(fmt) "InlineHookDemo: " fmt
#endif

static const uint32_t mbits = 6u;
static const uint32_t mask  = 0xfc000000u; 
static const uint32_t rmask = 0x03ffffffu; 
static const uint32_t op_bl = 0x94000000u; 

typedef long (*do_faccessat_t)(int, const char __user *, int, int) ;
static do_faccessat_t my_do_faccessat;

unsigned int orig_insn, hijack_insn;
unsigned long func_addr, insn_addr = 0;

uintptr_t kprobe_get_addr(const char *symbol_name) {
    int ret;
    struct kprobe kp;
    uintptr_t tmp = 0;
    kp.addr = 0;
    kp.symbol_name = symbol_name;
    ret = register_kprobe(&kp);
    tmp = (uintptr_t)kp.addr;
    if (ret < 0) {
        goto out; 
    }
    unregister_kprobe(&kp);
out:
    return tmp;
}

bool is_bl_insn(unsigned long addr){
	uint32_t insn = *(uint32_t*)addr;
	const uint32_t opc = insn & mask;
	if (opc == op_bl) {
		return true;
	}
	return false;
}

uint64_t get_bl_target(unsigned long addr){
	uint32_t insn = *(uint32_t*)addr;
	int64_t absolute_addr = (int64_t)(addr) + ((int32_t)(insn << mbits) >> (mbits - 2u)); 
	return (uint64_t)absolute_addr;
}

uint32_t build_bl_insn(unsigned long addr, unsigned long target){
	uint32_t insn = *(uint32_t*)addr;
	const uint32_t opc = insn & mask;
	int64_t new_pc_offset = ((int64_t)target - (int64_t)(addr)) >> 2; 
	uint32_t new_insn = opc | (new_pc_offset & ~mask);
	return new_insn;
}

uint32_t get_insn(unsigned long addr){
	return *(unsigned int*)addr;
}

void set_insn(unsigned long addr, unsigned int insn){
	cpus_read_lock();
	*(unsigned int*)addr = insn;
	cpus_read_unlock();
}

long hijack_do_faccessat(int dfd, const char __user *filename, int mode, int flags){
	char prefix[8];
	pr_emerg("hijack success!");
	copy_from_user(prefix, filename, 8);
	prefix[7] = 0;
	pr_emerg("access: %s", prefix);
	if (strcmp(prefix, "/memfd:") == 0) {
		pr_emerg("magic!");
		return 0;
	}
	return my_do_faccessat(dfd, filename, mode, flags);
}

int ihd_init(void){
	int i;
	
	
	func_addr = kprobe_get_addr("__arm64_sys_faccessat");
	pr_emerg("func_addr:%lX, ", func_addr);
	
	for(i = 0; i < 0x100; i++){
		if (is_bl_insn(func_addr + i * 0x4)) {
			insn_addr = func_addr + i * 0x4;
			break;
		}
	}
	if (insn_addr == 0) { 
		return -ENOENT;
	}
	orig_insn = get_insn(insn_addr);
	my_do_faccessat = (do_faccessat_t)insn_addr;
	pr_emerg("insn_addr:%lX, ", insn_addr);
	pr_emerg("orig_insn:%X orig_target_addr:%lX", orig_insn, get_bl_target(insn_addr));
	hijack_insn = build_bl_insn(insn_addr, (unsigned long)&hijack_do_faccessat);
	set_insn(insn_addr, hijack_insn);
	pr_emerg("new_insn:%X new_target_addr:%lX", hijack_insn, get_bl_target(insn_addr));
	return 0;
}

void ihd_exit(void){
	
	set_insn(insn_addr, orig_insn);
}

module_init(ihd_init);
module_exit(ihd_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ylarod");
MODULE_DESCRIPTION("A simple inline hook demo");

```

问题：

加载成功后没多久就寄掉了，日志中没有原因，就戛然而止，但是还是成功打印了`hijack success!`

### [](#f49558f0c5af49979c3782c3826b26ea "更好的方法")更好的方法

使用 kprobe 对内核进行 hook，更方便，而且问题也更少

```
static struct kprobe kp = {
    .symbol_name = "__arm64_sys_faccessat",
    .pre_handler = handler_pre,
    .post_handler = handler_post,
    .fault_handler = handler_fault
};

static void handler_post(struct kprobe *p, struct pt_regs *regs, unsigned long flags) {
}

static int handler_fault(struct kprobe *p, struct pt_regs *regs, int trapnr) {
    return 0;
}

struct Param {
    long dfd;
    const char __user *filename;
    long mode;
    long flags;
};

static int handler_pre(struct kprobe *p, struct pt_regs *regs) {
    struct Param param = *(struct Param *)regs->regs[0];
    
    pr_emerg("hijack success!");
    return 0;
}



```

特别注意：

kprobes 回调函数的运行期间关闭了抢占，同时也可能关闭中断。因此不论在何种情况下，在回调函数中不能调用会放弃 CPU 的函数（如信号量、mutex 锁等），否则会领取死机重启大礼包。

下期预告：

基于 **DirtyPipe 漏洞** 实现一个简单的 kernel root