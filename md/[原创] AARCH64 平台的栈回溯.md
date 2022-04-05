> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270936.htm)

> [原创] AARCH64 平台的栈回溯

> > 版权声明：本文为 CSDN 博主「ashimida@」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。  
> > 原文链接：https://blog.csdn.net/lidan113lidan/article/details/121801335
> > 
> > 更多内容可关注微信公众号 ![](https://bbs.pediy.com/upload/attach/202112/A4ZRSPJEVKWXH3D.jpg)    

一、术语解释:
=======

**1. 栈顶 / 栈底 [1]: ** 栈中最后一个 push, 第一个被 pop 的位置是栈顶; 栈中最后一个被 pop, 且 pop 后当前栈为空的位置是栈底;

**2. Current Function Frame Address[2]:** 当前函数栈帧, 在 aarch64 中是当前函数执行完 prologue 后的栈顶地址, 其可以通过__builtin_frame_address(0) 函数获取.

**3. Canonical Frame Address(CFA)[3]:** 标准 / 规范栈帧地址, 在 aarch64 中是当前函数父函数的栈顶地址, 在 DWARF 中对其定义如下:

> "Call Frame" 是在栈中分配的一段内存 (栈帧指的是一段内存), 其可以由栈中的一个地址来表示, 这个地址被称为标准栈帧地址 (CFA), 通常来说标准栈帧地址 CFA=caller 调用 callee 之前的 sp(stack pointer), 这个地址也就是 caller 函数的栈顶地址, 其和当前函数的 Function Frame Address 可能是不同的, 在 gcc 中 CFA 可以通过__builtin_dwarf_cfa 函数获取 (而不是__builtin_frame_address 函数, 后者获取的是 Function Frame Address, 也就是 Callee 的栈顶地址).

**4. prologue/epilogue:** 编译器为每个函数生成的默认的函数入口 / 出口指令序列.

**5. 最后一级栈帧 / 上一级栈帧:** 为便于表述, 这里定义对栈帧级别的描述, 如 A => B => C (A 中调用 B,B 中调用 C), 在函数 C 中发生了栈回溯, 那么后续称 C 函数的栈帧为最后一级栈帧, B 为 C 的上一级栈帧, A 的栈帧为函数 B 的上一级栈帧.

二、基于 fp 的栈回溯
============

  这里仅以 aarch64 为例, 基于 fp 的栈回溯会在函数的每一个栈帧中都将其父函数的栈顶保存到当前函数栈顶位置的内存中, 并同时设置栈帧寄存器 fp, 整个过程可描述如下:  
  1. 函数入口时其 fp 指向 CFA, 函数的 prologue 负责:  
     1) 为当前函数预留栈空间, 同时将父函数的 fp 保存到当前函数栈顶  
     2) 设置 fp 指向当前函数栈顶  
  2. 在当前函数执行过程中, fp 始终指向当前函数栈顶 (也就是函数栈帧 Function Frame Address)  
  3. 函数返回前的 epilogue 负责:  
     1) 从函数栈顶获取父函数的栈顶地址 (即当前函数的 CFA), 并将其写入 fp  
     2) 销毁当前函数栈空间  
     3) 返回父函数  
  此三步对应的源码如下:

```
test1:
        stp     x29, x30, [sp, -32]!        ## step 1.1), x29是栈帧寄存器(fp), x30 是函数返回地址, test1函数的函数栈大小编译期间已经确定, sp = sp-32 总大小为32byte
        add     x29, sp, 0                  ## step 1.2), x29=sp; 当前sp已经指向了test1的栈顶, test1栈帧总大小为32byte. 这里设置fp(x29)指向当前函数栈顶
        .......                             ## step 2. 函数执行过程中, fp时钟指向当前函数栈顶,即 Function Frame Address
        ldp     x29, x30, [sp], 32          ## step 3.1)/3.2), 函数返回时,将栈顶的函数返回地址(x30)和父函数的栈帧地址(x29)均出栈，出栈后 sp=x29=当前函数的CFA
        ret                                 ## step 3.3), 根据x30返回父函数

```

   还是以函数 A=>B=>C 为例, 若编译时未开启 - fomit-frame-pointer, 则函数调用过程中其寄存器变化如下图所示:

![](https://bbs.pediy.com/upload/attach/202112/490870_RX7MZZ6GVYV9SNN.jpg)

![](https://bbs.pediy.com/upload/attach/202112/490870_HFJ8PGD69WRQ82T.jpg)

这里需要注意的是:  
**1. gcc 中编译选项 - fomit-frame-pointer**  
  -fomit-frame-pointer 可以用来忽略栈帧的保存, 若开启此选项则函数 prologue 中不会在函数栈中保存 x29(fp), 所以若开启此编译选项则无法进行基于 fp 的栈回溯.   
**2. 关于 Current Function Frame Address 和 CFA:**  
  在 aarch64 中 prologue 和 epilogue 生成的汇编指令通常如下:

```
test1:
    stp     x29, x30, [sp, -32]!        ##1) sp = sp - 32; sp[0] = x29; sp[1] = x30;
    add     x29, sp, 0                  ##2) x29 = sp;
    .......                     
    ldp     x29, x30, [sp], 32  
    ret

```

  不论是否指定 - fomit-frame-pointer, 在 aarch64 的函数入口都需要先为当前 callee 函数预留整个栈空间, 如这里的 sp=sp-32; 在不考虑动态栈分配的情况下 callee 中的所有局部变量都位于指令 1) 处预留的空间内, **第一条指令执行完毕后, sp 由指向父函数栈顶变为指向子函数栈顶**. 在不忽略栈帧的情况下 (未开启 - fomit-frame-pointer):

*   每个函数入口处第二条指令通常都会将当前函数 (callee) 的栈顶地址 =>x29(fp), 作为新的函数栈帧(Current Function Frame Address); 
*   父函数的栈帧地址 (CFA, 也就是第二条指令直向前的 x29) 被保存在子函数的栈顶中

  所以在刚进入一个函数时, 其:  
    - sp 指向父函数 (caller) 的栈顶  
    - fp 同样指向父函数 (caller) 的栈顶  
 **- LR 指向父函数 (caller) 的下一条指令**  
  进入函数后就立即为当前函数 (callee) 预留栈帧, 完成栈帧预留后:  
    - sp 指向子函数 (callee) 的栈顶  
    - fp 同样指向子函数 (callee) 的栈顶  
 **- LR 还是指向父函数 (caller) 的下一条指令**

根据 GCC 手册和 DWARF 标准:

*   一个函数执行完 prologue 之后的的栈顶地址通常称为当前函数栈帧地址 (Current Function Frame Address), 即(从源码角度来看) 函数栈帧是当前函数的栈顶地址.
*   一个函数执行 prologue 之前的栈顶地址通常称为函数的标准 / 规范栈帧地址 (Canonical Frame Address,CFA), 即(从汇编代码角度看) 函数的 CFA 是当前函数 caller 的栈顶地址.

   gcc 源码中获取 CFA 和当前函数栈帧的函数如下:

```
// __builtin_frame_address(0)    ## 获取当前函数的栈帧地址,即prologue执行后的函数栈顶
// __builtin_frame_address(1)    ## 获取当前函数第1级父函数的栈帧地址, 是父函数执行完其prologue后的栈顶地址,也就是当前函数的CFA(注意当参数x > 1时
//                               ## 如果开启了-fomit-frame-pointer此函数可能会crash, 因为此函数的原理就是根据对fp的解引用逐级获取上一个栈帧地址.
// __builtin_dwarf_cfa()         ## 获取当前函数的CFA,其直接来自于prlogue执行前的sp
  
#include  #include  int main(void)
{
        register volatile unsigned long sp asm("sp");
        //在源码中获取的sp是执行完prologue之后的sp
        printf("sp:%p, fp:%p, caller fp:%p\n", sp, __builtin_frame_address(0), __builtin_dwarf_cfa());
        return 0;
}
  
Disassembly of section .text:
0000000000000000 

:
   4:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
   8:   910003fd        mov     x29, sp
   c:   910003e0        mov     x0, sp                    //参数1: x0 = sp;
  10:   aa1d03e1        mov     x1, x29                   //参数2: x1 = x29; 为callee函数栈帧
  14:   910043a2        add     x2, x29, #0x10            //参数3: x2 = x29 - 0x10; 为callee的CFA
  18:   aa0203e3        mov     x3, x2                    //参数3: x3 = CFA
  1c:   aa0103e2        mov     x2, x1                    //参数2: x2 为callee函数栈帧
  20:   aa0003e1        mov     x1, x0                    //参数1: x1 = sp
  24:   90000000        adrp    x0, 0 

  28:   91000000        add     x0, x0, #0x0
  2c:   94000000        bl      0  30:   52800000        mov     w0, #0x0                  
  34:   a8c17bfd        ldp     x29, x30, [sp], #16
  38:   d65f03c0        ret
  
tangyuan@ubuntu:~/compiler_test/gcc_test/aarch64/test1$ ./main
sp:0x40007ffe90, fp:0x40007ffe90, caller fp:0x40007ffea0 
```

**3. 基于 fp 栈回溯的本质:**  
  **运行时基于 fp 的栈回溯本质上并不是需要一个单独的 fp 寄存器来保存栈帧, 而是需要每个函数在入口将其父函数的栈帧保存到子函数的栈中**, **从而在栈回溯的过程中可以依次追溯找到每一级的栈**，aarch64 的特点是函数入口处先为子函数创建整个栈帧空间, 之后 sp 一直保持不变 (不考虑动态栈分配), 所以理论上用 sp 来做栈回溯也是可以的，只要每个函数都将当前 sp 的值入栈即可。  
  但在实现上 (gcc) 如果不关闭 - fomit-frame-pointer，那么子函数入口不会将其 caller 的栈帧地址保存到栈中，此时**虽然每个函数自身的汇编代码知道如何在返回前将 sp 指向父函数的栈顶**:

```
test1:
        stp     x29, x30, [sp, -32]!     
        add     x29, sp, 0           
        .......
        ldp     x29, x30, [sp], 32                 ## 函数自身知道如何将sp恢复到父函数的栈顶，如这里的sp = sp +32; 这是在编译期间确定的
        ret

```

  **但动态 unwind 过程并不知道如何找到父函数的 sp, 因为栈回溯时并不知道当前函数的栈帧的大小**.  
  在 aarch64 中基于 fp 栈回溯的实现是在每个子函数的入口将父函数的栈顶地址保存到了子函数的栈顶中，子函数通过简单的基于当前 sp/fp 的解引用即可获取到父函数的栈顶地址。  
  简单说就是每个函数入口必须保存其父函数的栈顶才能实现栈回溯，而 aarch64 中使用基于 sp 还是 fp 来做栈回溯理论上通常没有区别（只是多一级少一级的问题)。

三、aarch64 linux 内核中的基于 fp 的栈回溯
==============================

  在 aarch64 linux 中只实现了基于 fp 的栈回溯, 栈回溯的配置选项为 CONFIG_FRAME_POINTER，在 aarch64 中是默认开启的:

```
./arch/arm64/Kconfig
config ARM64
...
select FRAME_POINTER
 
./Makefile
ifdef CONFIG_FRAME_POINTER
KBUILD_CFLAGS   += -fno-omit-frame-pointer -fno-optimize-sibling-calls
...

```

  而**内核在开启基于 fp 的 backtrace 的同时禁止了 sibling call**, 因为一个函数 (caller) 若在某个路径中调用了 sibling call, 那么从函数 (caller) 入口到此 sibling call 的路径中都是不会保存 fp/sp 的, fp/sp 保存工作会交由子函数 (callee) 完成，如:

```
int __attribute__((noinline)) test2(void * addr)
{
   printf("test2 %p\n", addr);
   return 0;
}
  
int x;
int test1(void * addr)
{
   x = 90;
   return test2(addr);
}
  
#aarch64-linux-gnu-gcc main.c -c -O2 -o main.o
0000000000000000 :
   0:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
   4:   aa0003e2        mov     x2, x0
   8:   90000001        adrp    x1, 0  c:   52800020        mov     w0, #0x1                        // #1
  10:   910003fd        mov     x29, sp
  14:   91000021        add     x1, x1, #0x0
  18:   94000000        bl      0 <__printf_chk>
  1c:   52800000        mov     w0, #0x0                        // #0
  20:   a8c17bfd        ldp     x29, x30, [sp], #16
  24:   d65f03c0        ret
  
0000000000000028 :
  28:   90000001        adrp    x1, 4  2c:   52800b42        mov     w2, #0x5a                       // #90
  30:   f9400021        ldr     x1, [x1]
  34:   b9000022        str     w2, [x1]
  38:   14000000        b       0 
```

  **所以 aarch64 kenrel 中为了更好的 backtrace, 在开启了 CONFIG_FRAME_POINTER(默认) 的过程中直接关闭了 sibling call,** 最终完成栈回溯的函数是 dump_backtrace, 其基本原理就是基于 fp 的栈回溯, 代码如下:

```
void dump_backtrace(struct pt_regs *regs, struct task_struct *tsk, const char *loglvl)
{
    struct stackframe frame;
    .......
         
    /* 初始化栈回溯的 stackframe 结构体 frame */
    if (tsk == current) {
        /* 当前进程主动调用的backtrace, 则fp和pd可以直接设定 */
        start_backtrace(&frame, (unsigned long)__builtin_frame_address(0), (unsigned long)dump_backtrace);
    } else {
        /* 若backtrace其他进程,则需要从此进程保存的寄存器组中获取其pc/fp */
        start_backtrace(&frame, thread_saved_fp(tsk),  thread_saved_pc(tsk));
    }
  
    printk("%sCall trace:\n", loglvl);
    do {
        /* 打印当前函数的信息，实际上就是printk("...", pc); */
        if (!skip) {
            dump_backtrace_entry(frame.pc, loglvl);
        } else if (frame.fp == regs->regs[29]) {
            skip = 0;
            dump_backtrace_entry(regs->pc, loglvl);
        }
        
    } while (!unwind_frame(tsk, &frame));     /* 从函数栈中unwind到其父函数的栈帧(fp/pc)，并记录到frame结构体中 */
}
  
int notrace unwind_frame(struct task_struct *tsk, struct stackframe *frame)
{
    unsigned long fp = frame->fp;
    struct stack_info info;
    ......
        
    frame->fp = READ_ONCE_NOCHECK(*(unsigned long *)(fp));            /* 获取caller函数中的栈帧 */
    frame->pc = READ_ONCE_NOCHECK(*(unsigned long *)(fp + 8));        /* lr => pc */
     
    frame->prev_fp = fp;                                              /* 保存之前已经回溯的栈帧分配 */
    ......
  
    frame->pc = ptrauth_strip_insn_pac(frame->pc);                    /* 如果函数返回地址是pac加密过的,则解密 */
    return 0;
}

```

四、基于 DWARF 的栈回溯 (1)—基本概念
========================

  DWARF(Debugging With Attributed Record Formats)[4] 是许多编译器和调试器用来支持源码级调试的**调试文件格式**，其解决了许多过程语言 (如 C,C++,Fortran) 的需求并可以扩展到其他语言。DWARF 独立于体系结构，适用于任何处理器或操作系统，广泛应用于 Unix,Linux 和其他操作系统以及独立环境中，目前最新可用的 DWARF 已经到了第 5 个版本[4]。  
  DWARF 是用于调试的一个标准, 栈回溯信息 (Call Frame Infomation,CFI) 只是此标准中的一部分, 除此之外 DWARF 中包含的调试信息还包括:  
  * 数据和类型的描述  
  * 可执行代码描述  
  * 行号信息  
  * 宏信息  
  * 调用栈信息 (CFI)  
  * ...  
  正常来说编译时开启 -g 选项即可以生成 DWARF 的这些调试信息，这些信息都存储在. debug_* 开头节区中：

```
tangyuan@ubuntu:~/compiler_test/gcc_test/aarch64/test1$ aarch64-linux-gnu-gcc main.c -o main -g
tangyuan@ubuntu:~/compiler_test/gcc_test/aarch64/test1$ readelf -S main|grep -E "debug|eh"
  [16] .eh_frame         PROGBITS         00000000000007f0  000007f0
  [24] .debug_aranges    PROGBITS         0000000000000000  00001040
  [25] .debug_info       PROGBITS         0000000000000000  00001070
  [26] .debug_abbrev     PROGBITS         0000000000000000  00001454
  [27] .debug_line       PROGBITS         0000000000000000  000015a8
  [28] .debug_frame      PROGBITS         0000000000000000  000016a0
  [29] .debug_str        PROGBITS         0000000000000000  00001720

```

其中各个节区的作用定义如下:

<table><tbody><tr><td><p>.debug_abbrev</p></td><td><p>记录. debug_info 段中用到的缩写 (Abbreviation)</p></td></tr><tr><td><p>.debug_aranges</p></td><td><p>记录内存地址到编译单元的映射关系</p></td></tr><tr><td><p>.debug_frame</p></td><td><p>记录 Call&nbsp;Frame Infomation</p></td></tr><tr><td><p>.debug_info</p></td><td><p>包含 DWARF 核心 DIEs 的数据</p></td></tr><tr><td><p>.debug_line</p></td><td><p>包含行号信息</p></td></tr><tr><td><p>.debug_loc</p></td><td><p>位置描述信息</p></td></tr><tr><td><p>.debug_macinfo</p></td><td><p>宏信息</p></td></tr><tr><td><p>.debug_pubnames</p></td><td><p>全局函数和对象的索引表</p></td></tr><tr><td><p>.debug_pubtypes</p></td><td><p>全局类型索引表</p></td></tr><tr><td><p>.debug_ranges</p></td><td><p>记录 DIEs 中涉及到的地址范围</p></td></tr><tr><td><p>.debug_str</p></td><td><p>.debug_info 中的字符串表</p></td></tr><tr><td><p>.debug_types</p></td><td><p>类型描述信息</p></td></tr></tbody></table>

  需要注意的是**这些所有. debug_* 开头的节区都是非 load 的节区，也就是运行时并不会被加载到内存, 可以被直接 strip 掉，这也符合 DWARF 的本意即调试**.  
  因为不能加载到内存，故这些 debug_* 开头的节区无法用来做运行时 unwind 的，**真正支持运行时 unwind 功能的是 .eh_frame 节区**:  
  1) **.eh_frame(包括. eh_frame_hdr, .gcc_except_table) 是 LSB 标准中定义的一个节区 [5]，而并非 DWARF 标准中定义的内容, 但其中使用的数据格式是符合 DWARF 标准的 (并在此基础上有一些扩展)。**  
  2) **编译时. eh_frame 与调试用的 debug_* 节区的生成是相互独立的, 通过编译选项可以单独控制二者是否生成。**  
  3) **.debug_* 段是非 load 段, 将其移除掉不影响程序正常运行, .eh_frame 是 load 段，将其移除则可能导致运行时错误 (发生异常处理, unwind 时)。**  
  4) **增加. eh_frame 段的目的就是为了让其加载到运行时内存中 [6], 这样就可以实现一系列的功能，包括:**  
      * C++ 的异常处理  
      * backtrace()  
      * __attribute__((__cleanup__(f)))  
      * __builtin_return_address(n) for n >0  
      * __attribute__((__cleanup__(f))) 下的 pthread_cleanup_push  
      * ...  
   所以运行时基于 DWARF 的栈回溯实际上是基于. eh_frame 的栈回溯. **和基于 fp 的栈回溯不同的是，基于 DWARF 的栈回溯 (理论上) 还可以回溯各个寄存器在每一个栈帧中的值**。

**五、基于 DWARF 的栈回溯 (2)—数据结构**
============================

  .eh_frame 中的数据结构包括 CIE 和 FDE 两种, 其中 CIE 是多个函数共用的, 而 FDE 则是每个函数单独有一个。  
**1. CIE(Common Information Entry):**  
    CIE 中记录的是一些公用的入口信息, 包括异常的 handler, 数据增强信息等。本文中主要需要关注的是其中记录了一些共用的初始化代码, 每个函数都会对应一个 CIE, 一个 CIE 可以供多个函数共用, 如果两个函数拥有相同的初始化指令序列那么他们通常指向同一个 CIE，其结构体简单记录如下 (详细可参考 [5])：

```
  * Length: 4byte unsigned值, 代表此CIE结构体的大小(不包括Lenth字段自身), 如果Length值为0xffffff,则CIE结构体大小记录在Extended length字段
  * Extened length: 8byte unsigned 值记录CIE字段的大小，不包括 length和extend length字段的长度
  * CIE ID: 一个4byte的值用来区分CIE和FDE,对于CIE此值永远是0
  * Version： 代表Frame information的版本,其应该为1
  * Augmentation: 记录此CIE和CIE相关的FDE的增强特性，其是一个大小写敏感的字符串:
    - 'z': 如果Augment存在,则z必须是第一个字符，这就是个魔数
    - 'L': 代表针对数据的增强
    - 'P':
    - 'R': ...
  * Call Frame Instrctions(CFI): 这里是此CIE的初始化指令

```

**2. FDE(Frame Description Entry):**  
    FDE 中记录了此函数栈回溯相关的信息, 最主要的是记录了一系列指令序列 (CFI,Call Frame Infomation), 此指令序列可用来确定此函数执行到其每个地址时其各个寄存器的值应该如何获取 / 计算, 每个函数都有且仅有一个 FDE 结构体, 其结构体简单记录如下 (详见 [5]):

```
  * lenght/Extended lenght: 和CIE相同，二者可以同时存在且正确
  * CIE pointer: 当前位置到其CIE起始位置的偏移
  * pc begin/pc range:  此FDE负责的地址范围
  * ......
  * Call Frame Instrctions(CFI): 这里是一条条的CFI指令

```

  二进制中. eh_frame 中的信息可以通过 readelf -wf/wF 来查看:

```
## readelf -wF 显示的是一个友好的格式, 直接告知在某个地址各个寄存器的值应该如何计算, 但这并非CFI指令的真实形式
tangyuan@ubuntu:~/compiler_test/gcc_test/aarch64/test1$ readelf -wF main
Contents of the .eh_frame section:
  
00000000 0000000000000010 00000000 CIE "zR" cf=4 df=-8 ra=30
   LOC           CFA
0000000000000000 sp+0
.......
000000a0 0000000000000030 000000a4 FDE cie=00000000 pc=0000000000400690..000000000040070c
   LOC           CFA      x19   x20   x21   x22   x23   x24   x29   ra
##根据CFI指令，友好的显示出各个地址时各个寄存器应该如何栈回溯
0000000000400690 sp+0     u     u     u     u     u     u     u     u
0000000000400694 sp+64    u     u     u     u     u     u     c-64  c-56
000000000040069c sp+64    c-48  c-40  u     u     u     u     c-64  c-56
00000000004006a8 sp+64    c-48  c-40  c-32  c-24  u     u     c-64  c-56
00000000004006b0 sp+64    c-48  c-40  c-32  c-24  c-16  c-8   c-64  c-56
0000000000400708 sp+0     u     u     u     u     u     u     u     u
  
## readelf -wf显示的是 eh_frame中一条条CFI指令真实的形式
tangyuan@ubuntu:~/compiler_test/gcc_test/aarch64/test1$ readelf -wf main                 
Contents of the .eh_frame section:                                    
00000000 0000000000000010 00000000 CIE "zR" cf=4 df=-8 ra=30                             
   LOC           CFA
0000000000000000 sp+0
  
000000a0 0000000000000030 000000a4 FDE cie=00000000 pc=0000000000400690..000000000040070c
  ##真正的栈回溯信息是一条条的CFI指令
  DW_CFA_advance_loc: 4 to 0000000000400694                                              
  DW_CFA_def_cfa_offset: 64  
  DW_CFA_offset: r29 (x29) at cfa-64                                                     
  DW_CFA_offset: r30 (x30) at cfa-56
  DW_CFA_advance_loc: 8 to 000000000040069c
  DW_CFA_offset: r19 (x19) at cfa-48                                                     
  DW_CFA_offset: r20 (x20) at cfa-40                                                     
  DW_CFA_advance_loc: 12 to 00000000004006a8
  DW_CFA_offset: r21 (x21) at cfa-32
  DW_CFA_offset: r22 (x22) at cfa-24
  DW_CFA_advance_loc: 8 to 00000000004006b0
  DW_CFA_offset: r23 (x23) at cfa-16
  DW_CFA_offset: r24 (x24) at cfa-8
  DW_CFA_advance_loc: 88 to 0000000000400708
  DW_CFA_restore: r30 (x30)
  DW_CFA_restore: r29 (x29)
  DW_CFA_restore: r23 (x23)
  DW_CFA_restore: r24 (x24)
  DW_CFA_restore: r21 (x21)
  DW_CFA_restore: r22 (x22)
  DW_CFA_restore: r19 (x19)
  DW_CFA_restore: r20 (x20)
  
##对应源码:
0000000000400690 <__libc_csu_init>:
  400690:       a9bc7bfd        stp     x29, x30, [sp, #-64]!
  400694:       910003fd        mov     x29, sp
  400698:       a90153f3        stp     x19, x20, [sp, #16]
  ......

```

六、基于 DWARF 的栈回溯 (3)—栈回溯原理
=========================

  栈回溯函数通常都存在于运行时库中, 如 gcc 的 libgcc 或 clang 的 libunwind, 在每一级栈回溯的过程中 libgcc/libunwind 中都会通过上下文 (_Unwind_Context) 维护当前栈帧时的寄存器值, 为了便于区分后续用:

*   小写字母代表真实硬件寄存器的值, 如 sp/x30
*   大写字母代表运行时回溯到此栈帧时上下文中计算出的当前栈帧中的各个寄存器值, SPx/RAx/X10_x(x = 0,1,..., 代表每一级栈帧, 0 是最后一级栈帧).

  前面提到过基于 fp 的 unwind 并非需要一个 fp 寄存器, 而是要将父函数的栈顶地址保存在子函数栈中, 因为子函数虽然不需要栈帧即可找到父函数的栈顶, 但运行时的栈回溯函数并不知道此信息。**基于 DWARF 的栈回溯的原理是, 每个函数的 FDE 指令序列实际已经记录了此函数每条指令执行后 SPx 到 CFAx 的偏移 (SPx 和 CFAx 的偏移在编译时即可确定), 在运行时通过查表获取 CFA 的计算规则, 其本质上和函数自身可以找到父函数栈帧的原理是相同的**。 基于 DWARF 的栈回溯在运行时总是满足如下公式:  
 **1) CFAx=SPx+offsetx;**

*   CFAx: 栈回溯时第 x 级函数的 CFA
*   SPx: 此函数运行到其指令位置 pc 时硬件寄存器 sp 的值 (除最后一级函数外此值只能通过计算得出, 故这里为 SPx)
*   offsetx: SPx 和 CFAx 之间的偏移, 其随着函数 A 中指令的执行而变化, 但对于函数中每一条指令, 此偏移都会在编译期间确定, 运行时 offsetx 取决于当前函数 FDE 表中当前 PC 位置处记录的偏移。

 **2) SPx+1 = CFAx;**  
      在第 x 级函数的入口 (汇编角度), 函数的 CFA 总是等于当前硬件寄存器 sp 的值, 因此栈回溯到父函数时父函数当前的 SPx+1 总是可以用 CFAx 来替换。  
  所以如 A(2)=>B(1)=>C(0) 的整个栈回溯可以表示为:

```
//其中函数C计算CFA的指令地址即为pc(C), 函数C的返回地址记为pc(B),函数B的返回地址记为pc(A)
  CFA0 = sp + offset0;        //最后一级栈回溯的sp直接取自硬件寄存器, offset0是查询函数C FDE获取的在地址pc(C)处函数C的CFA和其当前硬件寄存器sp的偏移.
  SP1 = CFA0;                 //C的父函数B在执行到指令位置pc(B)时硬件寄存器sp值就是CFA1, 这里是通过计算得出,故记录为SP1
  CFA1 = SP1 + offset1;       //函数B的CFA(CFA1)和当前sp(SP1)的关系要根据当前函数B的执行位置pc(B),查找其FDE确定(即offset1)
  SP2 = CFA1;                 //B的父函数A在执行到指令位置pc(A)时(也就是调用子函数B时候)硬件寄存器sp的值就是CFA2,这里通过计算得出，记录为SP2
  ......

```

基于 DWARF 的栈回溯特点如下:

*   每个函数的 CFA 都可以通过其当前 sp + 当前 PC 在 FDE 中记录的 offset 计算得出 (PC 就是子函数的返回地址, 通常是子函数中通过 *(CFA +8) 解引用得来)。
*   函数中随着指令的执行其寄存器的值是会动态变化的, CFA(包括每一个寄存器) 的计算与当前函数执行到哪一条指令有关, 故某个函数栈回溯的过程中必须要指定此函数内的一个地址作为参数,  offset 由地址和 FDE 决定
*   由于寄存器是排他性资源 (一个寄存器只能记录其当前的值, 无法记录历史修改), 所以在栈回溯时只有在最后一级函数中可以获取到硬件寄存器 sp 的值
*   但一个函数的父函数在其调用点的当前 sp 一定等于此函数的 CFA, 故如此循环往复即可通过最后一级硬件寄存器 sp+FDE 中的偏移计算各级栈帧完成栈回溯。
*   这也就同时要求基于 DWARF 的栈回溯必须逐级的解析每一级函数的 FDE, 并逐级恢复每一级栈帧时 CFA(和各个寄存器的值), 每一级的计算都直接依赖于其下一级 (callee) 的计算结果.

   除了 CFA 外, 使用 DWARF **格式理论上还能获取到任何指令位置处任何寄存器的值 (**只是理论如此, 实际上不破坏已有标准的情况下只会恢复每个函数入口时各个 callee-saved 寄存器的值)**。**但由于寄存器是排他性资源, 故此前提是目标寄存器的值通常必须在要回溯的指令前保存到内存且执行到此指令时其值未发生变化，同时汇编代码还要插入对应的 DWARF 指令以在 FDE 中标记如何获取目标寄存器的值, 如： 

```
addr1:       stp     x29, x30, [sp, -32]!        ## stp指令负责将x29, x30寄存器入栈
             .cfi_def_cfa_offset 32              ## 定义当前CFA = SP +32; 即在addr2处CFA可以通过当前硬件寄存器SP+32得出.
             .cfi_offset 29, -32                 ## .cfi_offset指令用于标注在addr2处应该通过对CFA -32内存解引用获得此时X29寄存器的值,即在addr2: X29 = *(CFA -32); 
             .cfi_offset 30, -24                 ## X30的获取同理, 需要注意的是 .cfi_xxx系列指令只是用来记录硬件寄存器的值应该如何获取, 其自身并不保证能获取到正确寄存器值, .cfi_xxx指令实际上是对源码寄存器保存的描述.
addr2:       xxxxxxx

```

 **也有一些情况下目标寄存器的值不必保存到内存中, 如:**

*   寄存器的值可以通过 CFA+offset 计算得出, 如某位置处 X19 = CFA +0x10; 那么此时 X19 可以不入栈, 直接通过. cfi_val_offset x19, 0x10 指令标注即可.
*   寄存器的值来自某其他寄存器, 如 X19 = X18; 此时可直接通过 .cfi_register x18, x19 指令标注即可.
*   寄存器的值和在其下一级 (callee) 栈帧中的值相同, 如 callee_saved 寄存器在函数调用过程中保持不变, 此时可以直接通过. cfi_same_value x19 指令标注即可(通常. cfi_undefined 和. cfi_same_value 拥有相同实现)
*   CFAx 自身在栈回溯时通常总是通过偏移计算的, 如 CFA2 = SP2+ offset2 = CFA1 + offset2 = (SP1 + offset1) + offset2 = .....;  每个函数的栈帧大小通常在编译时就确定了, 故每一级的 CFAx 和 SPx 的偏移只需要查表即可, CFA 的值无需保存到内存 (这里和基于 fp 的栈回溯不同, 基于 fp 的栈回溯上一级 CFA 总是保存在当前函数栈顶的).

 七、基于 DWARF 的栈回溯 (4)—DWARF 指令解析
===============================

  通过查看. eh_frame 段的输出可以大体了解运行时 DWARF 指令的解析 (源码分析见后), 以如下函数为例:

```
//源码
int main(void)
{
        asm("":::"x19");       //标记x19为clobber,在函数返回前需要被恢复
        return 0;    
}
  
//汇编代码
main:
.LFB10:
        .cfi_startproc
        str     x19, [sp, -16]!
        .cfi_def_cfa_offset 16        //1)
        .cfi_offset 19, -16           //2)
        mov     w0, 0
        ldr     x19, [sp], 16
        .cfi_restore 19               //3)
        .cfi_def_cfa_offset 0
        ret
        .cfi_endproc
.LFE10:
        .size   main, .-main
  
//反汇编代码
0000000000400a90 

:                                                                                                                                                               
  400a90:       f81f0ff3        str     x19, [sp, #-16]!
  400a94:       52800000        mov     w0, #0x0                        // #0
  400a98:       f84107f3        ldr     x19, [sp], #16
  400a9c:       d65f03c0        ret



```

**1. readelf -wf:**

   readelf -wf 显示的内容和真正函数中的 CFI 指令是基本一致的, 看起来虽然不方便但便于理解原理:

```
## readelf -wf显示的是 eh_frame中一条条CFI指令的真实形式,这里省略无关输出
tangyuan@ubuntu:~/compiler_test/gcc_test/aarch64/test1$ readelf -wf main                 
Contents of the .eh_frame section:                                    
00000000 0000000000000010 00000000 CIE     //没有特别指定的情况下,所有函数函数都会使用此CIE中的指令做初始化
  Version:               1
  Augmentation:          "zR"
  Code alignment factor: 4
  Data alignment factor: -8
  Return address column: 30
  Augmentation data:     1b
  DW_CFA_def_cfa: r31 (sp) ofs 0           //此指令设置在当前函数函数入口时CFAx=SPx+0 (此时当前函数尚未分配栈,此时也同时满足SPx+1=CFAx), 此设置会随栈回溯在函数内发生的位置而调整.
  
                                 //FDE对应CIE的地址  当前FDE对应函数的地址范围
00000104 0000000000000018 00000108 FDE cie=00000000 pc=0000000000400a90..0000000000400aa0
  DW_CFA_advance_loc: 4 to 0000000000400a94     //loc1
                                                //DW_CFA_advance_loc 4的含义是将当前已经分析到的地址(LOC)+4, 当LOC>=PC(栈回溯发生在当前函数的那条指令)时, 此函数的FDE解析结束
                                                //也就是说如果当前函数在返回地址0x4000694发生了栈回溯,那么解析FDE到这一条指令后就结束解析了.
                                                //loc1 之前的指令(这里只有初始化指令)是为地址0x400690 设置栈回溯的寄存器规则, [loc1,loc2]的指令是为地址0x0400694设置寄存器规则
                                                //由于初始化只有CFAx=SPx+0, 而函数中第一条指令就跳到了0x040694,故-wF输出内容如下
                                                // LOC               CFA      x19   
                                                // 0000000000400a90  sp+0     u                //代表PC=0x400a90时(PC所在指令未执行前)的栈回溯规则为 CFAx=SPx+0; x19未定义
                                                // 0000000000400a94 ....
  DW_CFA_def_cfa_offset: 16                     //DW_CFA_def_cfa_offset: 16代表当前CFA计算规则变为CFAx=SPx+16; 这是因为在 400a90: str x19, [sp, #-16]! 执行完毕后会导致硬件寄存器
                                                //sp=sp-16,此时对于此函数来说sp!=CFA了. 若栈回溯发生在PC=0x400a94,则函数的CFAx=SPx+16; 注意不论在哪个地址发生栈回溯,函数的一次执行
                                                //过程中其CFA总是不变的,只是用于计算CFA的当前SP发生了变化, 故CFA的计算规则也要做出相应的调整. 此指令对应汇编代码的1)
  DW_CFA_offset: r19 (x19) at cfa-16            //由于汇编代码2)处中存在 ".cfi_offset 19, -16" 指令, 故当前X19的计算规则也发生了变化, 此时X19=*(CFAx-16),需要注意此指令有效的
                                                //前提是.400a90: str x19, [sp, #-16]!同时将x19保存到栈中,如果没有这条指令,单独的一条.cfi_offset在栈回溯时会改变程序员语义.
  DW_CFA_advance_loc: 8 to 0000000000400a9c     //loc2
                                                //设置 LOC +=8,到此说明地址0x400694/0x0400698的栈回溯指令已经结束(并不需要为每个地址都设置指令),后续的栈回溯指令针对地址0x040069C
                                                //执行到这里对应到 readelf -wF的输出为:
                                                //   LOC            CFA      x19   
                                                // ......
                                                // 0000000000400a94 sp+16    c-16              //代表PC=0x400a94/0x400a98时的栈回溯规则为 CFAx=SPx+16; X19=*(CFAx-16);
                                                // 0000000000400a9c ....
  DW_CFA_restore: r19 (x19)                     //loc3                 
                                                //这里开始的栈回溯规则属于地址0x400a9c,包括restore x19为undefine, 恢复cfa的计算规则为CFAx=SPx=0; 对应汇编代码中的3)
  DW_CFA_def_cfa_offset: 0                      //此两条规则是对地址0x400a98:  ldr x19, [sp], #16 指令的描述, 此指令中设置了sp=sp+16, 故地址0x400a9c处的CFA和当前sp的值又相同,其对应的输出如下:
                                                //   LOC            CFA      x19
                                                // 0000000000400a9c sp+0     u            //代表PC=x400a9c时的栈回溯规则为 CFAx=SPx; x19寄存器的值再次处于未定义状态
  DW_CFA_nop
  DW_CFA_nop

```

**2. readelf -wF**

   -wf 输出不便于阅读, 故通常情况下是通过 - wF 来查看每个地址的栈回溯指令信息:

```
## readelf -wF 显示的是一个友好的格式, 直接告知在此函数某个地址各个寄存器的值应该如何计算
tangyuan@ubuntu:~/compiler_test/gcc_test/aarch64/test1$ readelf -wF main
Contents of the .eh_frame section:
  
00000000 0000000000000010 00000000 CIE "zR" cf=4 df=-8 ra=30
   LOC           CFA
0000000000000000 sp+0
.......
  
00000104 0000000000000018 00000108 FDE cie=00000000 pc=0000000000400a90..0000000000400aa0                                                                                              
   LOC           CFA      x19   
0000000000400a90 sp+0     u     
0000000000400a94 sp+16    c-16          //在PC=0x400a94/0x400a98(PC所在指令未执行前),栈回溯规则为CFAx=SPx+16; X19=*(CFAx-16);
                                        //其中SP为运行到此PC位置时硬件寄存器sp的值, 对于最后一级栈帧其就是硬件寄存器sp的值,对于其他栈帧其等于自下一级栈帧的CFA(SPx=CFAx-1)
0000000000400a9c sp+0     u

```

  需要注意的是, 通常函数中只会在 prologue 中保存 callee-saved 寄存器的值并为其生成. cfi 指令, 整个函数执行期间计算这些寄存器的. cfi 指令可能发生变化, 但这些寄存器最终的值通常不会变化, 即不论在函数的任何位置 (除了 pro/epilogue 外), 根据栈回溯获取的 callee-saved 寄存器的值都应该是相同的 (否则异常处理时会有问题)。所以 FDE 中描述的某寄存器的值通常也可以看做是其对应函数入口时此寄存器的值。

八、基于 DWARF 的栈回溯 (5)—libgcc 中_Unwind_Backtrace 的实现
=================================================

   gcc 中基于 DWARF 的栈回溯是在运行时库 libgcc 中实现的, 其异常处理函数 (如_Unwind_Exception) 和栈回溯函数_Unwind_Backtrace 都使用了基于 DWARF 的栈回溯, 但需要注意的是_Unwind_Backtrace 是 LSB 标准中定义的函数[8], 而_Unwind_Exception 等异常处理函数是 IA-64 C++ ABI 标准中定义的函数[9].

```
//./gcc/config/aarch64.h
/* 增加此属性以确保函数中x29寄存器总是保存到栈中 */
#define LIBGCC2_UNWIND_ATTRIBUTE __attribute__((optimize ("no-omit-frame-pointer")))
  
//./libgcc/unwind-dw2.c
/* 这里需要一个宏定义,以确保__builtin_dwarf_cfa/__builtin_return_address获取到的是展开此宏函数的CFA/LR */
#define uw_init_context(CONTEXT)                               \
  do                                                           \
    {                                                          \
      /* 若一个函数调用了__builtin_unwind_init,则此函数的pro/epilogue中会push/pop所有calee-saved registers */    \
      __builtin_unwind_init ();                                \
      /* 根据uw_init_context_1的caller的CIE/FDE信息初始化一个_Unwind_Context上下文, 以记录每一级栈帧回溯过程中各个寄存器的信息 */ \
      uw_init_context_1 (CONTEXT, __builtin_dwarf_cfa (),      \
             __builtin_return_address (0));                    \
    }                                                          \
  while (0)
  
/* 在IA-64 C++ ABI中定义_Unwind_Context是一个不透明类型(opaque type), 其指向一个系统指定的用于传递unwind的数据结构
  此结构体由系统负责创建销毁,故在不同平台中定义可能不同,这里是libgcc中的定义 */
struct _Unwind_Context
{
  _Unwind_Context_Reg_Val reg[__LIBGCC_DWARF_FRAME_REGISTERS__+1];        /* 记录unwind到当前栈帧时计算出的各个寄存器的值或一个记录此时寄存器值的地址(大多数情况下是地址) */
  void *cfa;                                                              /* 当前函数的CFA,上面reg的计算都要依赖于此CFA */
  void *ra;                        /* 通常在aarch64中ra来自x30, 但.cfi_xxx可以修改返回地址规则, ra是真正语义上当前函数的返回地址, 其值会被当做下一次递归栈回溯查找FDE的PC. */                                                    
  void *lsda;                      /* 异常处理时的lsda信息,栈回溯过程中无用 */
  struct dwarf_eh_bases bases;     /* bases.func记录当前函数的起始地址,这是通过ra+FDE表查询得到的 */
  _Unwind_Word flags;              /* 特殊的flag,为简化流程可以先忽略 */
  _Unwind_Word version;            /* 默认为0 */
  _Unwind_Word args_size;
  char by_value[__LIBGCC_DWARF_FRAME_REGISTERS__+1];        /* by_value[i]记录reg[i]中记录的是寄存器i的值还是指向寄存器值的指针(by_value[i]=0;代表记录的是指针) */
}
  
//./libgcc/unwind.inc
/*
    此函数在c代码中可以直接调用, 其负责回溯当前的整个调用栈,在回溯到每个栈帧前可以通过回调函数trace输出信息, trace_argument可以传入一个全局参数
*/
_Unwind_Reason_Code LIBGCC2_UNWIND_ATTRIBUTE _Unwind_Backtrace(_Unwind_Trace_Fn trace, void * trace_argument)
{
  struct _Unwind_Context context;
  _Unwind_Reason_Code code;
  
  /* 初始化context, context中记录了当前_Unwind_Backtrace函数执行时的CFA,以及各个callee-saved寄存器的初始值
     实际上获取_Unwind_Backtrace函数自身CFA/各个寄存器的值可以直接通过内联汇编代码直接读取各个硬件寄存器即可(这也是libunwind的做法),
     而在libgcc中的uw_init_context函数则是利用_Unwind_Backtrace自身的DWARF信息来计算的初值(也就是所有寄存器的初值都来自_Unwind_Backtrace prologue中此寄存器写入的内存(见其汇编代码)
     此函数返回后context->ra指向_Unwind_Backtrace的返回地址.
   */
  uw_init_context (&context);
  
  while (1)        /* 循环遍历所有栈帧 */
    {
      _Unwind_FrameState fs;
  
      /* 根据_Unwind_Backtrace的返回地址(context->ra), 分析其caller的CIE/FDE信息, 将caller执行到_Unwind_Backtrace时各寄存器的回溯规则记录到fs中 */
      code = uw_frame_state_for (&context, &fs);
  
      if (code != _URC_NO_REASON && code != _URC_END_OF_STACK)
          return _URC_FATAL_PHASE1_ERROR;
  
      /* 在回溯上一帧时可以通过回调函数输出信息 */
      if ((*trace) (&context, trace_argument) != _URC_NO_REASON)
        return _URC_FATAL_PHASE1_ERROR;
  
      if (code == _URC_END_OF_STACK) break;    /* 栈回溯结束直接返回 */
  
      /* 根据fs中记录的caller寄存机计算规则更新context, 更新后的context记录caller调用_Unwind_Backtrace时的CFA和各个寄存器信息 */
      uw_update_context (&context, &fs);       
    }
  
  return code;
}
  
//_Unwind_Bactrace的prologue中保存了所有callee-saved寄存器
0000000000403210 <_Unwind_Backtrace>:
  403210:       d12b83ff        sub     sp, sp, #0xae0
  403214:       a9007bfd        stp     x29, x30, [sp]
  403218:       910003fd        mov     x29, sp
  40321c:       d50320ff        xpaclri
  403220:       aa1e03e2        mov     x2, x30
  403224:       a90153f3        stp     x19, x20, [sp, #16]
  403228:       910283f4        add     x20, sp, #0xa0
  40322c:       a9025bf5        stp     x21, x22, [sp, #32]
  403230:       aa0103f6        mov     x22, x1
  403234:       911183f5        add     x21, sp, #0x460
  403238:       912b83e1        add     x1, sp, #0xae0
  40323c:       a90363f7        stp     x23, x24, [sp, #48]
  403240:       aa0003f7        mov     x23, x0
  403244:       aa1403e0        mov     x0, x20
  403248:       a9046bf9        stp     x25, x26, [sp, #64]
  40324c:       a90573fb        stp     x27, x28, [sp, #80]
  403250:       6d0627e8        stp     d8, d9, [sp, #96]
  
//_Unwind_Backtrace的FDE中记录了各个callee-saved寄存器的计算规则
000007a8 000000000000007c 000007ac FDE cie=00000000 pc=0000000000403210..00000000004032f4
   LOC           CFA      x19   x20   x21   x22   x23   x24   x25   x26   x27   x28   x29   ra    v8    v9    v10   v11   v12   v13   v14   v15   
0000000000403210 sp+0     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u          
0000000000403214 sp+2784  u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u          
0000000000403218 sp+2784  u     u     u     u     u     u     u     u     u     u     c-2784 c-2776 u     u     u     u     u     u     u     u          
0000000000403228 sp+2784  c-2768 c-2760 u     u     u     u     u     u     u     u     c-2784 c-2776 u     u     u     u     u     u     u     u          
0000000000403230 sp+2784  c-2768 c-2760 c-2752 c-2744 u     u     u     u     u     u     c-2784 c-2776 u     u     u     u     u     u     u     u          
0000000000403240 sp+2784  c-2768 c-2760 c-2752 c-2744 c-2736 c-2728 u     u     u     u     c-2784 c-2776 u     u     u     u     u     u     u     u          
0000000000403260 sp+2784  c-2768 c-2760 c-2752 c-2744 c-2736 c-2728 c-2720 c-2712 c-2704 c-2696 c-2784 c-2776 c-2688 c-2680 c-2672 c-2664 c-2656 c-2648 c-2640 c-2632
00000000004032e8 sp+0     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u     u          
00000000004032ec sp+2784  c-2768 c-2760 c-2752 c-2744 c-2736 c-2728 c-2720 c-2712 c-2704 c-2696 c-2784 c-2776 c-2688 c-2680 c-2672 c-2664 c-2656 c-2648 c-2640 c-2632
  
//根据此二者信息uw_init_context_1才可以将所有callee-saved寄存器的初值初始化到_Unwind_Backtrace函数的context中

```

  uw_init_context 函数定义如下：

```
//libgcc/unwind-dw2.c
/* 
   此函数需要noinline属性以确保其不会在其caller函数中展开,否则无法确定__builtin_return_address(0)获取的是哪一级caller的返回地址.
   context: 一个要初始化的_Unwind_Context上下文的指针
   outer_cfa: 是当前函数caller的CFA, 在caller函数可通过调用__builtin_dwarf_cfa()函数传入
   outer_ra: 是当前函数caller的返回地址, 在caller函数中可通过调用__builtin_return_address (0)函数传入
   此函数的作用是通过基于DWARF的栈回溯为其父函数初始化栈回溯的上下文context,返回的context中记录着父函数调用 uw_init_context_1之前CFA和各个寄存器的值/指针.
   需要注意的是,此函数的实现只依赖.eh_frame中的CIE/FDE信息和三个硬件寄存器的值:
   * uw_init_context_1返回到其caller(这里也就是_Unwind_Backtrace函数)的地址,此地址用来确定其caller的CIE/FDE信息在.eh_frame中的位置
   * caller的CFA, 这个值是从caller中传来的, 其是计算caller其他各个寄存器值的基础, 此CFA实际就是caller函数入口时的SP
   * caller的返回地址, 这个值也是从caller中传来的,用于继续回溯上一级栈帧(实际上这个值不传也可以从栈上获取)
   需要注意的是,基于DWARF的栈回溯并不能保证所有寄存器都被初始化,只有caller中入栈的寄存器可以被初始化,这也是为什么在其caller中都要调用 __builtin_unwind_init
  函数的原因, __builtin_unwind_init可以确保_Unwind_Backtrace函数的pro/epilogue保存了所有callee-saved register, 但其他寄存器均没有保存.
   所以此函数返回的context中存在很多未初始化的寄存器(如_Unwind_Backtrace的context中x0-x18均为初始化), 后续继续栈回溯时除非某一级栈帧主动
  保存并定义了这些寄存器,否则基于x0-x18的.cfi指令可能直接导致系统crash(可参考后面SCS的例子).
 */
static void __attribute__((noinline)) uw_init_context_1 (struct _Unwind_Context *context, void *outer_cfa, void *outer_ra)
{
  _Unwind_FrameState fs;
  _Unwind_SpTmp sp_slot;
  _Unwind_Reason_Code code;
  
  /* 获取当前函数uw_init_context_1的返回地址, 如当前函数caller为_Unwind_Backtrace,那么此ra就是_Unwind_Backtrace中的一个地址 */
  void *ra = __builtin_extract_return_addr (__builtin_return_address (0));
    ........
  
  /* 利用DWARF信息, 为uw_init_context_1的父函数初始化一个上下文 context, context中记录父函数调用uw_init_context_1时各个callee-saved寄存器的值/指针 */
  memset (context, 0, sizeof (struct _Unwind_Context));    /* 初始化_Unwind_Context结构体 */
  
  context->ra = ra;    /* context->ra是要回溯的父函数_Unwind_Backtrace 调用uw_init_context_1处的地址 */
  
  /* 根据context->ra确定其所在函数(caller)的CIE/FDE(这里是_Unwind_Backtrace函数), 并解析其中的.cfi指令序列,确定caller执行到地址context->ra时各个寄存器的回溯规则
     此函数不修改context结构体,只是将指令解析结果记录到fs中, fs结构体中记录的是caller CFA/各个寄存器的计算规则,不涉及具体值/指针. */
  code = uw_frame_state_for (context, &fs);
  .......
  
  /*
    这里还是以caller为_Unwind_Backtrace为例, 正常_Unwind_Backtrace函数的 CFA1 = SP1 + offset; 其中SP1是_Unwind_Backtrace执行到uw_init_context_1时SP寄存器的值,
   这种计算并没有错误,但由于传入的outer_cfa本身就是CFA1了,所以这里将规则修改为 SP1 = CFA1; CFA1 = SP1 + 0; 再调用uw_update_context_1结果也是一样的.
    这里实际上有多种写法, 直接设置CFA1=SP0也是可以的(SP0可以通过在uw_init_context_1函数中调用__builtin_dwarf_cfa获取)
  */
  _Unwind_SetSpColumn (context, outer_cfa, &sp_slot);
  fs.regs.cfa_how = CFA_REG_OFFSET;
  fs.regs.cfa_reg = __builtin_dwarf_sp_column ();
  fs.regs.cfa_offset = 0;
  
  /
  /* 此函数执行前 context记录的是callee的运行时信息, 此函数根据caller CFA/各寄存器的计算方法(fs),和callee的context计算出
    caller(如这里_Unwind_Backtrace函数)的context',最终结果写回到context中, 此函数并未更新caller的返回地址 */
  uw_update_context_1 (context, &fs);
  
  /* 下一次要回溯的函数是_Unwind_Backtrace的父函数,故这里将context->ra设置为_Unwind_Backtrace函数的返回地址 */
  context->ra = __builtin_extract_return_addr (outer_ra);
  ......
}

```

  uw_frame_state_for 函数负责解析 caller 的 CIE/FDE, 代码如下;

```
//libgcc/unwind-dw2.c
/*
   context记录的是当前已经分析的函数(callee)中各个寄存器/CFA的值或指针.
   uw_frame_state_for函数负责解析context->ra所在函数(caller)的CIE/FDE指令,以确定在caller执行到context->ra指令时其CFA/各个寄存器的计算规则.
   最终解析结果记录在fs结构体中,callee的上下文(context)在此过程中不做修改. fs中只记录caller中CFA/各个寄存器的计算规则,并不涉及具体值/指针.
*/
static _Unwind_Reason_Code
uw_frame_state_for (struct _Unwind_Context *context, _Unwind_FrameState *fs)
{
  const struct dwarf_fde *fde;
  const struct dwarf_cie *cie;
  const unsigned char *aug, *insn, *end;
  
  memset (fs, 0, sizeof (*fs));
  
  if (context->ra == 0)    return _URC_END_OF_STACK;      /* 如果当前函数的返回地址为0,则代表栈回溯结束直接返回 END_OF_STACK */
  
  /* 根据函数的返回地址(caller中的某指令地址),在.eh_frame查找函数caller所在的FDE, 同时FDE中记录了其caller的首地址, 将其记录到 context->base->func中 */
  fde = _Unwind_Find_FDE (context->ra + _Unwind_IsSignalFrame (context) - 1, &context->bases);
  
  if (fde == NULL)  return _URC_END_OF_STACK;             /* 若fde为空则返回栈回溯结束 */
  
  fs->pc = context->bases.func;                           /* 根据FDE解析结果,设置fs->pc为caller的起始地址, 这也是后续解析caller中.cfi指令序列的起始LOC */
  
  cie = get_cie (fde);                                    /* 根据FDE获取caller的CIE地址 */
   
  insn = extract_cie_info (cie, context, fs);             /* 获取CIE中初始化指令中的第一条指令的地址 */
  
  if (insn == NULL)    return _URC_FATAL_PHASE1_ERROR;    /* 若CIE中没有初始化指令则直接返回error */
  
  end = (const unsigned char *) next_fde ((const struct dwarf_fde *) cie);    /* 根据CIE大小,计算出CIE中最后一条初始化指令的地址 */
  
  /* 逐条解析caller(context->ra所在函数)CIE中的初始化指令([insn,end]), 并将解析结果记录到fs中,fs中不记录具体指针,只记录.cfi指令中规定个CFA/各个寄存器的计算方法 */
  execute_cfa_program (insn, end, context, fs);
  
  /* 解析完caller的初始化指令后,还需要解析caller自身FDE中 LOC < context->ra之前的指令 */
  aug = (const unsigned char *) fde + sizeof (*fde);      /* 获取caller的FDE中第一条指令地址 */
    .......
  insn = aug;
  
  end = (const unsigned char *) next_fde (fde);           /* 获取caller的FDE中最后一条指令地址 */
  
  /* 解析caller(context->ra所在函数)FDE中的所有指令序列([insn,end]),直到解析到fs->pc >= context->ra位置,
     其含义是确定caller函数执行到地址context->ra时,各个寄存器/CFA的值应该如何获取,最终结果同样保存在fs中, 
     此过程并不修改当前函数的上下文(context)
  */
  execute_cfa_program (insn, end, context, fs);
  
  return _URC_NO_REASON;
}
  
/* 此结构体记录一个栈帧对应函数的首地址, 此栈帧中各个寄存器和CFA等的计算方式等信息 */
typedef struct
{
  struct frame_state_reg_info                         /* 此结构体记录CFA/各个寄存器的计算方式,不涉及具体地址 */
  {
    struct {
      enum how;                                       /* 记录计算方式 */
      union {
        _Unwind_Word reg;
        _Unwind_Sword offset;
        const unsigned char *exp;
      } loc;                                          /* 记录计算方式相关的一个数据,可能是一个寄存器编号, 一个偏移或一个表达式 */
    } reg[__LIBGCC_DWARF_FRAME_REGISTERS__+1];        /* 记录CIE/FDE指令中每一个寄存器的计算方式 */
  
    enum cfa_how;                                     /* 记录CFA的计算方式, 可以是reg + offset或一个expr */
    _Unwind_Sword cfa_offset;                         /* 若计算方式为reg+offset,则这两个字段有用 */
    _Unwind_Word cfa_reg;
    const unsigned char *cfa_exp;                     /* 若计算方式是表达式，则此字段指向FDE中具体表达式指令 */
  } regs;
  
  void *pc;      /* 初始化为当前函数的首地址,在execute_cfa_program分析过程中根据.cfi指令调整为已分析到的指令地址(对应DW_CFA_advance_loc: 中的LOC) */
  .......
  _Unwind_Word retaddr_column;                        /* 函数的返回地址来自那一列(index),默认是R30_REGNUM */
  .......
} _Unwind_FrameState;
  
  
/* 
   insn_ptr: 指向FDE/CIE中的一条指令(作为遍历的开始)
   insn_end: 指向FDE/CIE中的一条指令(作为遍历的结束)
   context: 通常是某个函数栈帧已经计算好的上下文, 实际上这里只用到context->ra, 代表此函数的callee的返回地址.
   fs: 记录对context->ra所在函数栈回溯的结果, 传入时要确保fs->pc指向context->ra所在函数的首地址
   此函数负责逐条解析[insn_ptr,insn_end]之间的指令序列(其应该是context->ra所在函数的FDE/CIE中的整个指令序列),在此过程中根据每一条指令修改fs中的数据:
   * 对于如DW_CFA_advance_loc指令,则修改fs->pc到新的位置(对应DW_CFA_advance_loc: 中的LOC)
   * 对于如DW_CFA_def_cfa_register指令,修改CFA的计算规则,结果保存到fs->cfa_how/cfa_offset/cfa_reg/cfa_exp;
   * 对于如DW_CFA_register指令,修改对应寄存器的计算规则,结果保存到fs->regs->reg[i].loc/how;
   此函数的作用是解析context对应函数的caller的指令序列,并将分析结果保存到fs中, 并不直接修改当前的context.
   fs中记录的都是其caller的寄存器/CFA的计算规则,并未计算具体值/指针, 值和指针的计算会在uw_update_context函数中完成,此时fs的结果会同步到新的context中.
*/
static void execute_cfa_program (const unsigned char *insn_ptr, const unsigned char *insn_end,
             struct _Unwind_Context *context, _Unwind_FrameState *fs)
{
  /* fs->pc 是在当前函数中已经解析到的指令位置, context->ra是当前函数的返回地址, 这里负责循环解析FDE/CIE中的指令序列, 
    直到fs->pc >= context->ra(即当前函数已经执行过的指令)为止, 并根据每条解析到的指令序列,设置 fs */
  while (insn_ptr < insn_end
     && fs->pc < context->ra + _Unwind_IsSignalFrame (context))
    {
      unsigned char insn = *insn_ptr++;
      _uleb128_t reg, utmp;
      _sleb128_t offset, stmp;
  
      if ((insn & 0xc0) == DW_CFA_advance_loc)
        fs->pc += (insn & 0x3f) * fs->code_align;     /* 调整当前已解析的指令地址 */
      else if ((insn & 0xc0) == DW_CFA_offset)        /* 对应.cfi_offset register, offset指令, 其作用是设置当前寄存器reg值的计算方式为 *(CFA + offset) */
      {
          reg = insn & 0x3f;                          /* 解析指令中的寄存器 */
          insn_ptr = read_uleb128 (insn_ptr, &utmp);  /* 读取指令中的offset偏移码并计算偏移 */
          offset = (_Unwind_Sword) utmp * fs->data_align;
          .......
          fs->regs.reg[reg].how = REG_SAVED_OFFSET;   /* 记录当前寄存器修改方式 */
          fs->regs.reg[reg].loc.offset = offset;      /* 记录偏移 */
          }
      }
      .......
      else switch (insn)
      {
        case DW_CFA_set_loc:
        {
            _Unwind_Ptr pc;
            insn_ptr = read_encoded_value (context, fs->fde_encoding,insn_ptr, &pc);
            fs->pc = (void *) pc;                     /* 重置pc */
        }
        break;
        case DW_CFA_register:
         .......
        case DW_CFA_def_cfa_register:
         .......
        .......
        default:
          gcc_unreachable ();
      }
    }
}

```

  uw_update_context 负责将 context 从 callee 状态更新为 caller 状态, 其代码如下:

```
//./libgcc/unwind-dw2.c
/*
    此函数调用前, context代表栈回溯过程中已经计算出的某个栈帧对应函数的各个寄存器/CFA的值(指针), 其中context->ra记录的是其返回到caller函数的地址.
    此函数负责根据caller执行到context->ra时候的各个寄存器计算方法(在参数fs中,来自CIE/FDE的解析结果) 更新context,
    更新后的context' 记录的则是caller在执行到context->ra时CFA/各个寄存器的值(指针), context'->ra指向caller的父函数.
*/
static void uw_update_context (struct _Unwind_Context *context, _Unwind_FrameState *fs)
{
  uw_update_context_1 (context, fs);
  
  /* context->ra指向 caller的返回地址(其父函数中某地址), 若返回地址未定义,则代表栈回溯结束 */
  if (fs->regs.reg[DWARF_REG_TO_UNWIND_COLUMN (fs->retaddr_column)].how == REG_UNDEFINED)
    context->ra = 0;
  else
  {
      /* 设置context->ra指向caller的返回地址 */
    context->ra = __builtin_extract_return_addr(_Unwind_GetPtr (context, fs->retaddr_column));
    .......
  }
}
  
/* 根据fs中的计算规则,更新context中CFA/各个寄存器的值(指针),更新后context中则记录caller调用callee时各个寄存器的状态 */
static void uw_update_context_1 (struct _Unwind_Context *context, _Unwind_FrameState *fs)
{
  struct _Unwind_Context orig_context = *context;        /* 先复制一份context, 以确保修改context时可以引用callee栈帧的计算结果 */
  void *cfa;
  long i;
  ......
  _Unwind_SpTmp tmp_sp;
  
  /* 在如aarch64平台中, CIE/FDE中没有sp的规则,在栈回溯过程中context->reg[sp]总是未定义的, 对于这些平台需要手动设置 SP = context->cfa */
  if (!_Unwind_GetGRPtr (&orig_context, __builtin_dwarf_sp_column ()))
    _Unwind_SetSpColumn (&orig_context, context->cfa, &tmp_sp);
  
  /* SP的值不应该继承自上一个栈帧,这没有意义，故这里主动清空 */
  _Unwind_SetGRPtr (context, __builtin_dwarf_sp_column (), NULL);
  
  switch (fs->regs.cfa_how)        /* 先更新CFA */
    {
    case CFA_REG_OFFSET:           /* CFA = reg + offset, reg默认来自sp(=callee的CFA) */
      cfa = _Unwind_GetPtr (&orig_context, fs->regs.cfa_reg);
      cfa += fs->regs.cfa_offset;
      break;
    case CFA_EXP:
    {
      .......
      execute_stack_op (exp, exp + len, &orig_context, 0);    /* 若CFA是表达式,则通过此函数解析表达式 */
      break;
    }
    default:
      gcc_unreachable ();
    }
  
  context->cfa = cfa;               /* caller的CFA已经计算出来,保存到context->cfa中; caller的其他寄存器的计算依赖于此CFA */
  
  /* 遍历所有寄存器的计算方法,最终确定caller各个寄存器的值/指针 */
  for (i = 0; i < __LIBGCC_DWARF_FRAME_REGISTERS__ + 1; ++i)
    switch (fs->regs.reg[i].how)
  {
      case REG_UNSAVED:            /* same/未定义寄存器保持和callee中结果一致 */
      case REG_UNDEFINED:
        break;
  
      case REG_SAVED_OFFSET:       //reg[i] = *(cfa + reg.offset)
        _Unwind_SetGRPtr (context, i,(void *) (cfa + fs->regs.reg[i].loc.offset));
        break;
  
      case REG_SAVED_REG:
        ........
  }
  ......
}

```

九、基于 DWARF 的栈回溯 (6)—libunwind 与 libgcc 在保存上下文时的区别
=================================================

   虽然都满足 IA-64 C++ABI 标准，但不同库对基于 DWARF 的 unwind 的实现是略有不同, 如 libunwind/libgcc 在 context 上下文初始化时的实现就有所区别, 同样是_Unwind_Backtrace 函数:

*   在 libunwind 中_Unwind_Backtrace 函数入口最开始就会通过汇编代码将所有硬件寄存器的当前值保存到 context 中, 所以使用 libunwind 栈回溯时其回调函数中总是可以通过调用_Unwind_GetGR 获取任何一个寄存器的值, 测试输出如下 (见附录中的代码 std_callback):

```
Unwind Frame(0): CFA:00000040007ffe10, SP:00000040007ffe10, RA:000000000023a7a8                                                                                    
R00:00000040007ff950,R01:00000000002ae8e8,R02:00000040007fffc8,R03:000000000023a770,
R04:00000040007ffe30,R05:b7591a76500cc58e,R06:00000000002afcf8,R07:0000400000000000,
R08:00000000002ae000,R09:000000000023a624,R10:0000000800000020,R11:0000000000000000,
R12:0000000000000001,R13:0000000000000000,R14:00000000002b1750,R15:0000000000000000,
R16:0000000000000000,R17:0000000000000000,R18:0000000000000000,R19:0000000000241860,
R20:0000000000241924,R21:0000000000000000,R22:0000000000200338,R23:00000000002ae8a0,
R24:0000000000000018,R25:0000000000200338,R26:00000000002b0000,R27:00000000002b0000,
R28:0000000000000000,R29:00000040007ffe20,R30:0000000000240b44,R31:00000040007ffe10,
 
Unwind Frame(1): CFA:00000040007ffe30, SP:00000040007ffe30, RA:000000000024128c
R00:00000040007ff950,R01:00000000002ae8e8,R02:00000040007fffc8,R03:000000000023a770,
R04:00000040007ffe30,R05:b7591a76500cc58e,R06:00000000002afcf8,R07:0000400000000000,
R08:00000000002ae000,R09:000000000023a624,R10:0000000800000020,R11:0000000000000000,
R12:0000000000000001,R13:0000000000000000,R14:00000000002b1750,R15:0000000000000000,
R16:0000000000000000,R17:0000000000000000,R18:0000000000000000,R19:0000000000241860,
R20:0000000000241924,R21:0000000000000000,R22:0000000000200338,R23:00000000002ae8a0,
R24:0000000000000018,R25:0000000000200338,R26:00000000002b0000,R27:00000000002b0000,
R28:0000000000000000,R29:00000040007ffe30,R30:0000000000240b44,R31:00000040007ffe30,

```

*   在 libgcc 中_Unwind_Backtrace 通过调用一个子函数 uw_init_context 配合_Unwind_Backtrace 自身的 DWARF 信息完成 context 初始化, 此时只有 callee-saved/x29,x30 寄存器被初始化了, 调用_Unwind_GetGR 获取非 callee-saved 寄存器可能会导致 crash(见附录测试函数中 gcc_callback).

  **实际上 libgcc 的实现并没有问题, 只是在 IA-64 ABI 下的接口不太友好** (_Unwind_GetGR 的 crash 无法预测, 为确保不 crash 只能根据 AAPCS64 标准选择只打印 callee-saved 寄存器)。

  因为打印非 callee-saved 寄存器通常没有意义, 这些寄存器在函数调用过程中随时都可能被修改, 若某函数没有将其保存到栈中那么即使 context 中有值通常也是错误的。如上面 libunwind 测试结果中 Unwind Frame 0/1 中的通用寄存器的值完全相同，但并不复合实际情况 (其每次输出的都只是_Unwind_Backtrace 时这些寄存器的值).

   **但 libgcc 的实现对一些指令的解析存在影响,** 如 Shadow Call Stack 需要在在异常处理之前插入 ".cfi_escape 0x16, 0x12, 0x02, 0x82, 0x78" 指令 (即 x18=x18-8), 此指令在 libunwind 中可以正常运行, 但在 libgcc 中就会由于 x18 寄存器未初始化而 crash[10]

参考资料:
=====

[1] [Stack Computers: 1.2 WHAT IS A STACK?](https://users.ece.cmu.edu/~koopman/stack_computers/sec1_2.html)

[2] <Using The GNU Compiler Collection>, __builtin_frame_address 的定义

[3] [Download DWARF Standards](https://dwarfstd.org/Download.php "Download DWARF Standards")

[4] [Dwarf Home](https://dwarfstd.org/ "Dwarf Home")

[5] [Exception Frames](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html "Exception Frames")

[6] [assembly - Why GCC compiled C program needs .eh_frame section? - Stack Overflow](https://stackoverflow.com/questions/26300819/why-gcc-compiled-c-program-needs-eh-frame-section)

[7] [C++ ABI for Itanium: Exception Handling](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)

[8] [_Unwind_Backtrace](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/baselib--unwind-backtrace.html "_Unwind_Backtrace")

[9] [Download DWARF Standards](https://dwarfstd.org/Download.php "Download DWARF Standards")

[10] [[PATCH] [RFC][PR102768] aarch64: Add compiler support for Shadow Call Stack](https://gcc.gnu.org/pipermail/gcc-patches/2021-November/585199.html)

[11] [Stack unwinding | MaskRay](https://maskray.me/blog/2020-11-08-stack-unwinding)

[12] [探索 Android 平台 ARM unwind 技术 - 知乎](https://zhuanlan.zhihu.com/p/336916116?ivk_sa=1024320u "探索Android平台ARM unwind技术 - 知乎")

[13] [linux 栈回溯 (x86_64) - 知乎](https://zhuanlan.zhihu.com/p/302726082)

[14] [Unwind 栈回溯详解：libunwind_RToax-CSDN 博客_libunwind](https://blog.csdn.net/Rong_Toa/article/details/110846509 "Unwind 栈回溯详解：libunwind_RToax-CSDN博客_libunwind")

附:
==

  1. 关于基于 DWARF 的栈回溯和 IA-64 C++ABI 的其他分析可参考 [11-14]

  2. 以下代码用于测试 libgcc/libunwind 中的_Unwind_Backtrace 实现:

```
#include  #include  #include  unsigned long level = 0;
  
#ifndef CC_IS_CLANG
#define AARCH64_DWARF_V0       64
#define AARCH64_DWARF_NUMBER_V 32
#define DWARF_ALT_FRAME_RETURN_COLUMN   (AARCH64_DWARF_V0 + AARCH64_DWARF_NUMBER_V)
#define DWARF_FRAME_REGISTERS           (DWARF_ALT_FRAME_RETURN_COLUMN + 1)
  
struct dwarf_eh_bases
{
  void *tbase;
  void *dbase;
  void *func;
};
  
struct _Unwind_Context
{
  _Unwind_Word reg[DWARF_FRAME_REGISTERS+1];
  void *cfa;
  void *ra;
  void *lsda;
  struct dwarf_eh_bases bases;
  _Unwind_Word flags;
  _Unwind_Word version;
  _Unwind_Word args_size;
  char by_value[DWARF_FRAME_REGISTERS+1];
};
  
//基于libgcc的栈回溯的回调函数需要解除非IA-64 ABI判断寄存器状态才能决定是否可输出
_Unwind_Reason_Code gcc_callback(struct _Unwind_Context * context, void * args)
{
    unsigned long * plevel = args;
    //在sp(x31)未初始化时直接调用_Unwind_GetGR(context, __builtin_dwarf_sp_column()) 会导致crash
    printf("Unwind Frame(%ld): CFA:%016lx, SP:%016lx, RA:%016lx\n", *plevel, \
        _Unwind_GetCFA(context),
        context->reg[__builtin_dwarf_sp_column()]?_Unwind_GetGR(context, __builtin_dwarf_sp_column()):0,    
        _Unwind_GetIP(context));
    int i;
    for(i = 0; i < 32; i++) {
        if(i%4 == 0) {
            if(i != 0) printf("\n");
        }
            //printf("R%02d(%d):%016lx,", i, context->by_value[i], context->reg[i]);
        if(context->reg[i] && !context->by_value[i])
            printf("R%02d:%016lx,", i, _Unwind_GetGR(context, i));
        else
            printf("R%02d:%016lx,", i, context->reg[i]);
    }
    printf("\n\n");
    (*plevel)++;
  
  
    return _URC_NO_REASON;
}
#endif
  
//此函数完全使用IA-64 ABI中的接口函数,在libunwind中运行正常,但基于libgcc则会crash
_Unwind_Reason_Code std_callback(struct _Unwind_Context * context, void * args)
{
    unsigned long * plevel = args;
    printf("Unwind Frame(%ld): CFA:%016lx, SP:%016lx, RA:%016lx\n", *plevel, \
        _Unwind_GetCFA(context), _Unwind_GetGR(context, __builtin_dwarf_sp_column()), _Unwind_GetIP(context));
    int i;
    for(i = 0; i < 32; i++) {
        if(i%4 == 0) {
            if(i != 0) printf("\n");
        }
        printf("R%02d:%016lx,", i, _Unwind_GetGR(context, i));
    }
    printf("\n\n");
    (*plevel)++;
  
  
    return _URC_NO_REASON;
}
  
_Unwind_Reason_Code (*pcallback) (struct _Unwind_Context * context, void * args);
  
int main(void)
{
#ifndef CC_IS_CLANG
    pcallback = gcc_callback;
#else
    pcallback = std_callback;
#endif
    _Unwind_Backtrace(pcallback, &level);
    return 0;
} 
```

  编译命令:

```
.PHONY: all
CC = /mnt/disk0/disk0/toolchains/llvm/clang+llvm-12.0.1-x86_64-linux-gnu-ubuntu-16.04/bin/clang
AARCH64_GCC_TOOLCHAIN = /mnt/disk0/disk0/gcc_source_code/mk_cross_compiler/cross-gcc/
TARGET = aarch64-linux-gnu
INCLUDE := -L/mnt/disk0/disk0/toolchains/llvm/clang+llvm-12.0.1-aarch64-linux-gnu/lib/ -L/mnt/disk0/disk0/toolchains/llvm/clang+llvm-12.0.1-aarch64-linux-gnu/lib/clang/12.0.1/lib/linux
  
CLANG_CC = $(CC) --target=$(TARGET) --gcc-toolchain=$(AARCH64_GCC_TOOLCHAIN) $(INCLUDE) -DCC_IS_CLANG
  
all:
        ## gcc 基于 libgcc的编译
        aarch64-linux-gnu-gcc -static main.c -O0 -o main
        ## clang基于libunwind的编译
        $(CLANG_CC) -static main.c -o main_clang  --rtlib=compiler-rt -lunwind -stdlib=libc++ -stdlib=libstdc++  -fuse-ld=lld -lsupc++

```

[【看雪培训】目录重大更新！《安卓高级研修班》2022 年春季班开始招生！](https://bbs.pediy.com/thread-271992.htm)

最后于 2022-2-19 15:33 被 ashimida 编辑 ，原因：

[#基础知识](forum-41-1-130.htm) [#系统内核](forum-41-1-131.htm) [#开源分享](forum-41-1-134.htm)