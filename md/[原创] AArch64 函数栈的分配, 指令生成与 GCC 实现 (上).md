> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272176.htm)

> [原创] AArch64 函数栈的分配, 指令生成与 GCC 实现 (上)

本篇主要介绍原理，源码分析见  [AArch64 函数栈的分配, 指令生成与 GCC 实现 (下)](https://blog.csdn.net/lidan113lidan/article/details/123961954)  

一、术语定义
======

  为便于理解，本文中后续术语含义解释如下:

1.  首地址: 一个变量 / 内存的首地址指的是此变量 / 内存最低位的地址
2.  尾地址: 一个变量 / 内存的尾地址指的是此变量 / 内存最高地址 (+1), 如变量存储空间为 [0x0, 0x10), 则此变量首地址为 0x0, 尾地址为 0x10
3.  GPR/FPR: AArch64 中的通用寄存器 (R0-R30) 后面称为 GPR(General Purpose Register); 浮点寄存器称为 FPR(Floating Point Register)

二、AArch64 函数栈的组成
================

  aarch64 下函数栈是向低地址方向增长, 按照由高 => 低的顺序其函数栈可以包含如下区域:

```
/* 从高地址=>低地址, 这些区域包括:
  * incoming stack: 传入栈参数区域 (这段区域实际上属于caller函数, 在callee中作为栈参数使用)
  * varargs: 匿名寄存器参数区域
  * local variables: 局部变量区域
  * callee-saved regs: callee-saved 寄存器存储区域
  * dynamic allocation: 动态栈分配区域
  * outgoing stack: 传出栈参数区域
*/
+-------------------------------+ <- highmem
|                               |
|  incoming stack arguments     |
|                               |
+-------------------------------+ <-- incoming stack (aligned)/caller sp
|                               |
|  callee-allocated save area   |
|  for register varargs         |
|                               |
+-------------------------------+
|  local variables              |
|                               |
+-------------------------------+
|  padding                      | \
+-------------------------------+  |
|  callee-saved registers       |  | frame.saved_regs_size
+-------------------------------+  |
|  LR'                          |  |
+-------------------------------+  |
|  FP'                          | /
+-------------------------------+ <-- 硬件寄存器fp(x29)指向这里(若有)
|  dynamic allocation           |
+-------------------------------+
|  padding                      |
+-------------------------------+
|  outgoing stack arguments     |
|                               |
+-------------------------------+ <-- 硬件寄存器sp指向这里/callee sp
|                               |
                    | <- lowmem

```

  虽然函数栈整体是由高地址 => 低地址增长的，但每个区域内部有自己的变量分配顺序。这些区域的主要作用为:

**1. incoming stack arguments:**
--------------------------------

  即**传入栈参数区域, 内部元素分配方向:** **低地址 => 高地址**。

  严格来讲这段区域是 caller 分配的, 并不属于 callee 栈帧的一部分, 当 caller 调用 callee 时, callee 通过 incoming 区域接收 caller 的参数。

  对于 caller 来说这段区域称为 outgoing stack argument(传出栈参数区域), 一个函数的 incoming 区域首地址等于其父函数的 outgoing 区域的首地址。

**2. callee-allocated save area for register varargs:**
-------------------------------------------------------

  即**匿名寄存器参数区域, 内部元素分配方向:** **低地址 => 高地址**。

  对于可变参函数，此区域用来保存函数中可能用到的所有匿名寄存器参数，va_xxx 系列函数的参数解析依赖于此区域。

  从此区域开始 (以及向低地址方向的内存) 均为 callee 分配, 此区域按照高地址 =>低地址实际上分为 GPRs/FPRs 两个子区域; 每个区域内部元素分配方向均是低地址 =>高地址;

**3. local variables:**
-----------------------

  即**局部变量区域, 内部元素分配方向: 高地址 => 低地址**。

  函数中显示定义的变量，编译器内部生成的临时变量均存放在此区域中。

  需要注意的是，编译优化可能导致变量分配顺序与源码中定义顺序不同，在不考虑优化的情况下其内部元素分配方向默认是高地址 => 低地址。

**4. callee-saved registers:**
------------------------------

  即 **callee-saved 寄存器区, 内部元素分配方向:** **低地址 => 高地址**。

  在 AAPCS64 标准中 [3]:

*   [R19, R28] 是 callee-saved registers, 这些寄存器在函数调用结束后要保持原始值，如果 callee 中修改了这些寄存器，则在返回前必须恢复其原始值。
*   LR(R30), FP(R29) 并非 callee-saved 寄存器, 但由于操作流程类似, 后续放在 callee-saved 寄存器中一并讨论。

**5. dynamic allocation:**
--------------------------

  即**动态栈分配区, 内部元素分配方向: 高地址 => 低地址**。

  当函数中调用 alloca 时会在栈中 dynamic allocation 区域分配需内存，这部分区域在编译期间不存在, 在运行期间大小可以随执行变化。

**6. outgoing stack arguments:**
--------------------------------

  即**栈参数传出区域**, **内部元素分配方向:** **低地址 => 高地址**;

  当 caller 需要通过栈向子函数传递参数 (如大于 8 个 GPR 参数) 时, 栈参数保存在 caller 的 outgoing 区域。

  一个函数 (作为 caller) 其 outgoing 区域的大小是固定的，取决于其调用的 callees 中使用了最多栈参数的那个函数。当函数中存在动态分配时，其 outging 区域的首地址是可能动态变化的，但总是等于当前硬件寄存器 sp 的值, caller 调用 callee 时也是通过此 sp 将其 outgoing 区域首地址传递给 callee(作为 callee 的 incoming 区域首地址)。

三、incoming/outgoing 区域与栈参数
==========================

  按照 AAPCS64 标准，以通用寄存器 (GPRs) 传参为例(不考虑浮点参数和结构体参数等) :

*   若一个函数调用的参数 <=8 个则参数会通过 R0-R7 传入
*   对于 > 8 的参数，则会通过栈传递

  如以下函数:

```
#define weak __attribute__((weak))
 
weak int func1(int x0, int x1, int x2, int x3, int x4, int x5, int x6, int x7, int x8, int x9)
{
        printf("func1: incoming:%p ,%p\n",  &x8, &x9);
}
 
weak int func2(int x0, int x1, int x2, int x3, int x4, int x5, int x6, int x7, int x8, int x9, int x10)
{
        printf("func2: incoming:%p ,%p, %p\n",  &x8, &x9, &x10);
}
 
int main(void)
{
   int x8 = 8;
   int x9 = 9;
   int x10 = 10;
   register void * sp asm("sp");
   printf("main1: sp:%p, fp:%p\n", sp, __builtin_frame_address(0));
   func1(0, 1, 2, 3, 4, 5, 6, 7, x8, x9);
   func2(0, 1, 2, 3, 4, 5, 6, 7, x8, x9, x10);
}
 
//aarch64-linux-gnu-gcc main.c -O2 -o main
tangyuan@ubuntu:~/tests/gcc/aarch64_stack/caller_callee_parm_test$ ./main
main1: sp:0x40007ff830, fp:0x40007ff850
func1: incoming:0x40007ff830 ,0x40007ff838
func2: incoming:0x40007ff830 ,0x40007ff838, 0x40007ff840

```

  其中:

1. func1/func2 的栈参数来自其 caller(main1) 的函数栈

2. 两次函数调用的前两个寄存器参数在栈中的地址是相同的, 其中:

*   第 8 个参数 (x8) 总是在地址 0x40007ff830
*   第 9 个参数 (x9) 总是在地址 0x40007ff838

  在 incoming/outgoing 区域, 栈参数是基于 callee 入口时 sp 向高地址方向寻址的, **正是由于参数首地址相同且 ABI 规则相同 (AAPCS64), caller 和 callee 之间才能正确的传递栈参数**.

以上代码的函数栈如图:

![](https://bbs.pediy.com/upload/attach/202204/490870_57B6QBD9MCMYM4Z.jpg)

  这里需要注意的是:

*   caller 函数在编译期间需要一个固定大小的 outgoing 区域 (否则栈生成逻辑、性能上都会有影响), 一个 caller 可能调用多个 callee, 故对于 **caller 来说其 outgoing 区域的大小要以占用 outgoing 区域最大的那个子函数为准** (细节可见源码分析)。 如上面 func1 中需要存储 2 个栈参数，func3 中需要存储 3 个栈参数，则在 caller 中需要分配 (对齐后)0x20 大小的空间用来存储 outgoing 的参数。
*   caller 在编译期间总是可以确定其需要向每一个调用的 callee 传递多少个参数，不论 callee 是否为可变参函数, 因此 caller 在编译期间即可确定 outgoing 区域大小。

  通过汇编代码可以更好的理解 outgoing/incoming 区域:

```
.LC2:
        .string "main1: sp:%p, fp:%p\n"
        .......
main:
        sub     sp, sp, #64                 ## main函数整个栈空间 64 byte，包括局部变量和outgoing区域
        adrp    x0, .LC2                    ## printf字符串参数
        mov     x1, sp
        add     x0, x0, :lo12:.LC2
        stp     x29, x30, [sp, 32]
        add     x29, sp, 32
        mov     x2, x29
        stp     x19, x20, [sp, 48]
        bl      printf
        mov     w19, 9
        mov     w20, 8
        str     w20, [sp]                   ## func1的第一个栈参数x8保存到 *sp中,sp为当前outgoing首地址
        str     w19, [sp, 8]                ## func1的第二个栈参数x9保存到 *[sp+8]中, outgoing中第二个参数地址
        mov     w7, 7                       ## 其他为寄存器传参
        mov     w6, 6
        mov     w5, 5
        mov     w4, 4
        mov     w3, 3
        mov     w2, 2
        mov     w1, 1
        mov     w0, 0
        bl      func1                       ## call func1
        mov     w0, 10
        str     w20, [sp]                   ## func2的第一个栈参数x8保存到 *sp中, sp为当前outgoing首地址
        str     w19, [sp, 8]                ## func2的第二个栈参数x9保存到 *(sp+8)中
        mov     w7, 7
        str     w0, [sp, 16]                ## func2的第三个栈参数x10保存到 *(sp+16)中
        mov     w6, 6                       ## 其余为寄存器参数
        mov     w5, 5
        mov     w4, 4
        mov     w3, 3
        mov     w2, 2
        mov     w1, 1
        mov     w0, 0
        bl      func2                       ## call func2
        mov     w0, 0
        ldp     x29, x30, [sp, 32]
        ldp     x19, x20, [sp, 48]
        add     sp, sp, 64
        ret
 
func1:
        stp     x29, x30, [sp, -16]!        ## func1自身栈大小16 byte,用来保存callee-saved寄存器
        adrp    x0, .LC0
        add     x0, x0, :lo12:.LC0
        mov     x29, sp
        add     x2, sp, 24                  ## callee的incoming = callee入口时的sp, 由于函数入口时已经做了sp-=16,
                                            ## 所以此时callee第二个栈参数来自 *(sp + 24), 也就是*(incoming + 8)
        add     x1, sp, 16                  ## callee的第一个栈参数来自 *(sp + 16), 也就是 *(incoming)
        bl      printf
        ldp     x29, x30, [sp], 16
        ret
 
func2:
        stp     x29, x30, [sp, -16]!
        adrp    x0, .LC1
        add     x0, x0, :lo12:.LC1
        mov     x29, sp
        add     x3, sp, 32                  ## func2同理,第一个栈参数来自 *(sp + 16), 第二个来自 *(sp+24)
                                            ## 第三个来自 *(sp+32)
        add     x2, sp, 24
        add     x1, sp, 16
        bl      printf
        ldp     x29, x30, [sp], 16
        ret

```

四、可变参函数与匿名寄存器参数的存储
==================

1. 可变参函数
--------

  可变参函数指的是一个可以接受可变个参数的函数, 调用此函数时**只有 caller 知道为此函数传入了多少个参数, 可变参函数 callee 只知道 caller 最少传入了多少个参数**:

*   callee 中可以确定的参数称为命名参数 (Named arguments)
*   callee 中不确定的参数称为匿名参数 (Anonymous arguments)

  对于可变参函数 (callee), 其需要:

*   在**其函数栈中预留空间存储所有可能的匿名寄存器参数** (否则由于不确定传入了多少个寄存器参数，AAPCS64 标准中的所有传参寄存器在 callee 中均不可直接使用)
*   在**运行时通常需要通过已知的某个命名参数来确定 caller 到底传入了多少个参数** (caller 在调用可变参数函数时通常需要显式或隐式的告知 callee 自己传入了多少个参数), 如:
    

```
/* 常见的处理可变参的宏定义如下，后续代码中直接使用内联函数
    #define va_start(v,l)   __builtin_va_start(v,l)
    #define va_end(v)   __builtin_va_end(v)
    #define va_arg(v,l) __builtin_va_arg(v,l)
    #define va_copy(d,s)    __builtin_va_copy(d,s)
*/
 
//int printf (const char * format, ... );
#define weak __attribute__((weak))
weak int func1(int x, ...)
{
        int i = x;
     
        __builtin_va_list vl;
        __builtin_va_start (vl, x);
        while(i > 0)
        {
          printf("%d\n", va_arg(vl, int)); // callee根据caller传入的参数个数逐个获取参数
          i --;
        }
        __builtin_va_end (vl);
 
        return 0;
}
 
int main(void)
{
   register void * sp asm("sp");
   printf("main1: sp:%p, fp:%p\n", sp, __builtin_frame_address(0)); //printf通过参数R0隐式确定参数个数
    
   func1(3, 1, 2, 3);   //一般情况下调用可变参数需通过命名参数显式告知参数个数
} 
 
## 输出
tangyuan@ubuntu:~/tests/gcc/aarch64_stack/caller_callee_parm_test$ ./main
main1: sp:0x40007ff860, fp:0x40007ff860
1
2
3

```

2. 可变参函数的参数
-----------

  可变参函数的参数分为四类, 即**寄存器命名参数, 寄存器匿名参数, 栈命名参数, 栈匿名参数**, 举例如下:

```
#define REP7(X) X,X,X,X,X,X,X
#define REP8(X) X,X,X,X,X,X,X,X
#define weak __attribute__((weak))
/*
   func1的:
   * 第1个参数(x0,必须指定)为命名寄存器参数
   * 第2-8个参数(若有)为匿名寄存器参数
   * 第9-n个参数(若有)为匿名栈参数
*/
weak int func1(int x0, ...) { ... }
/*
   func2的:
   * 前8个参数(x0-x7,必须指定)为命名寄存器参数
   * 第9个参数(x8,必须指定)为命名栈参数
   * 第10-n个参数(若有)为匿名栈参数
*/
weak int func2(int x0, int x1, int x2, int x3, int x4, int x5, int x6, int x7, int x8, ...) { ... }

```

  可变参函数的参数传递同样满足 AAPCS64:

* 对 caller 来说, 编译期间即可确定其向可变参 callee 传入了多少个参数，故其直接按照 AAPCS64 标准布局参数 (寄存器参数直接赋值给硬件寄存器，栈参数保存到 outgoing 栈对应位置).

* 对 callee 来说, 编译期间只能确定有多少个命名参数:

*   **命名寄存器参数**直接通过硬件寄存器获取
*   **命名栈参数**通过 incoming 栈获取

* 对 callee 来说, 匿名参数的个数需直接或间接通过某个命名参数得知:

*   **匿名栈参数**同样**继续**通过 incoming 区域获取
*   **匿名寄存器参数**则是从**匿名寄存器存储区域** (callee-allocated save area for register varargs) 获取

  举例如下:

```
#include #include struct __va_list            /* 自定义的__va_list结构体 */
{
        void *__stack;
        void *__gr_top;
        void *__vr_top;
        int   __gr_offs;
        int   __vr_offs;
};
 
int func(int p0, int p1, int p2, ...)
{
        register unsigned long sp asm("sp");
        unsigned long  var0 = (unsigned long)&var0 - sp;    /* 第一个局部变量到callee入口sp的offset */
        unsigned long  var1 = (unsigned long)&var1 - sp;    /* 第二个局部变量到callee入口sp的offset */
        __builtin_va_list vl;
 
        p0 = (unsigned long)&p0 - sp;               /* 第一个命名寄存器参数的存储位置到callee入口sp的offset
                                                                 * (解引用会导致callee在局部变量中为命名寄存器参数分配存储空间,
                                                                 *  这里顺便解释局部变量的内存分配)
                                                                 */
        p1 = (unsigned long)&p1 - sp;               /* 第二个命名寄存器参数的存储位置到callee入口sp的offset */
        printf("[+] sp:%p\n", sp);
        printf("[+] var0 in addr %p (offset %p)\n", &var0, var0);
        printf("[+] var1 in addr %p (offset %p)\n", &var1, var1);
        printf("[+] vl in addr %p, size:%d\n", &vl, sizeof(vl));    /* 局部变量vl的地址及大小 */
 
        printf("[+] p0 in addr %p (offset %p)\n", &p0, p0);
        printf("[+] p1 in addr %p (offset %p)\n", &p1, p1);
 
        var0 = 0;
        __builtin_va_start (vl, p2);    /* p2记录此函数匿名寄存器个数 */
        while(var0 < p2) {
                var0 ++;
                if( var0 < 6) {
                        /* 输出剩余匿名通用寄存器参数的存储首地址和offset */
                        var1 = ((struct __va_list *)&vl)->__gr_top + ((struct __va_list *)&vl)->__gr_offs;
                } else {
                        /* 输出下一个匿名栈参数的存储位置 */
                        var1 = ((struct __va_list *)&vl)->__stack;
                }
                printf("[+] p%d in addr %p (offset %p) with value %d\n", var0 + 2, var1, var1 - sp, va_arg(vl, int));
        }
}
 
int main(void)
{
        func(0, 0, 7, 1, 2, 3, 4, 5, 6, 7);
} 
```

  输出结果:

```
##aarch64-linux-gnu-gcc main.c -O2 -o main## 输出结果tangyuan@ubuntu:~/tests/gcc/aarch64_stack/caller_callee_parm_test$ ./main
[+] sp:0x40007ffdf0
[+] var0 in addr 0x40007ffe38 (offset 0x48)
[+] var1 in addr 0x40007ffe30 (offset 0x40)
[+] vl in addr 0x40007ffe10, size:32
[+] p0 in addr 0x40007ffe0c (offset 0x1c)
[+] p1 in addr 0x40007ffe08 (offset 0x18)
[+] p3 in addr 0x40007ffec8 (offset 0xd8) with value 1
[+] p4 in addr 0x40007ffed0 (offset 0xe0) with value 2
[+] p5 in addr 0x40007ffed8 (offset 0xe8) with value 3
[+] p6 in addr 0x40007ffee0 (offset 0xf0) with value 4
[+] p7 in addr 0x40007ffee8 (offset 0xf8) with value 5
[+] p8 in addr 0x40007ffef0 (offset 0x100) with value 6
[+] p9 in addr 0x40007ffef8 (offset 0x108) with value 7

```

  其函数栈分布如图:

![](https://bbs.pediy.com/upload/attach/202204/490870_ZJ72CK3TQCMTX27.jpg)![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

  由此函数可以看到:

*   匿名寄存器参数 (varargs) 分为两部分，由高地址 =>低地址分别是 GPRs/FPRs.
*   GPRs/FPRs 内部的元素增长方向都是低地址 => 高地址 (如 p3 地址 < p4 地址)
*   以 GPRs 为例, 编译期间会根据当前函数可能出现的匿名寄存器个数确定此区域大小，如此函数中只需在匿名寄存器区域为 [p3-p7] 预留空间即可.
*   匿名栈参数和命名栈参数一样，均按照顺序存储在 incoming 区域 (对于 caller 没有匿名参数一说，所有栈参数按照顺序存储)
*   若 func 有大于 8 个整型, 8 个浮点型参数，则其函数栈中不再需要为匿名寄存器参数预留空间, 如:
    

```
## 此函数虽然是可变参数，但不需要分配匿名寄存器存储区域
int func(int p0, int p1, int p2, int p3, int p4, int p5, int p6, int p7,
        double d0, double d1, double d2, double d3, double d4, double d5, double d6, double d7,
        ...);

```

*   整个 outgoing 区域 + varargs 区域是可变参函数的参数存储区 (这里只介绍栈布局，关于可变参函数 va_xxx 系列函数的分析可参考 [5])

五、局部变量区
=======

  在不开启优化时, 局部变量通常是按照其定义顺序依次分配到局部变量区域的 (高地址 => 低地址), 但开启优化时局部变量的分配可能经过延迟展开, 重排等优化, 此时源码中定义顺序与分配顺序可能不同。

1. 定参函数的局部变量区域
--------------

  局部变量区的尾地址 (即第一个局部变量的尾地址):

*   对于定参函数来说，总是此函数入口时的 sp.
*   对于不定参函数来说，通常有个匿名寄存器区的偏移.

  故对于定参函数，通常会看到如下布局:

```
#include int main(void)
{
        int x;
        int y;
        printf("%p, %p\n", &x, &y);
} 
```

```
## aarch64-linux-gnu-gcc main.c -O0 -S -o main.s
# +--------------+ <------ sp        ## callee入口时sp
# |  x           |
# +--------------+ <------ sp - 4    ## 局部变量 x
# |  y           |
# +--------------+ <------ sp - 8    ## 局部变量 y
# |......(align) |
# +--------------+ <------ sp' + 16
# |  x30         |
# +--------------+ <------ sp' + 8
# |  x29         |
# +--------------+ <------ sp'
# |  ......      |
# +--------------+
 
main:
        stp     x29, x30, [sp, -32]!            ## 函数入口 sp' = sp - 32;                                                                                                                                                                     
        mov     x29, sp
        add     x1, sp, 24
        add     x0, sp, 28
        mov     x2, x1                          ## 变量y的地址为 sp' + 24, 即 sp - 8
        mov     x1, x0                          ## 变量x的 地址为 sp' + 28, 即 sp - 4
        adrp    x0, .LC0
        add     x0, x0, :lo12:.LC0
        bl      printf
        mov     w0, 0
        ldp     x29, x30, [sp], 32
        ret

```

2. 局部变量区的存储内容
-------------

  局部变量区除了用来保存源码中定义的局部变量外，还用来存储 gcc 编译过程中生成的临时变量以及硬件寄存器参数的原始值 (若需要), 以**三. 2** 中代码为例, 由其局部变量区可见:

![](https://bbs.pediy.com/upload/attach/202204/490870_C4SU64FWSCGBERZ.jpg)![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

1. 局部变量区域的起始地址为变量 var0 的尾地址

2. 源码中的局部变量 var0/var1/vl 按照高地址 => 低地址的顺序依次分布 (但并非总是如此，编译优化可能改变分配顺序)

3. 源码中存在对参数 p0/p1/p2 的解引用, 在局部变量区依次为其分配存储空间 (参数的存储位置总是在源码中局部变量之后), 实际上:

*   在开启 - O0 优化时，在局部变量栈中无条件为所有寄存器参数预留空间 (如 p0/p1/p2)
*   在开启 - O2 优化时，则默认不为寄存器参数预留空间 (除非需要).

六、callee-saved registers
========================

  callee-saved regs 区域用来保存 callee-saved register 的, 按照 AAPCS64 标准 [r19, r28] 是 callee-saved register，这些寄存器在子函数返回时要保持不变，也就是说如果子函数中使用了这些寄存器，则子函数需要保存并在返回前恢复其原始值。 函数栈中 callee-saved register 区域就是用来保存这些寄存器原始值的。callee-saved regs 区域内部也是由低地址 => 高地址增长的, 但区别在于**此区域按照寄存器的编号顺序保存，而不是其在源码中被使用的顺序**, 如:

```
#include extern int func1(void);
 
int main(void)
{
    asm("":::"x30");
    asm("":::"x19");
    asm("":::"x20");
    func1();
}
 
#aarch64-linux-gnu-gcc main.c -O0 -S -o main.s  -fomit-frame-pointer
# +--------------+ <------ sp        ## callee入口时sp
# |  ......      |
# +--------------+                 
# |  x30         |
# +--------------+ <------ sp' + 16
# |  x20         |
# +--------------+ <------ sp' + 8
# |  x19         |
# +--------------+ <------ sp' = sp - 32  ## callee-saved reg按照编号x19,x20,x30依次存储
 
main:
        stp     x19, x20, [sp, -32]!
        str     x30, [sp, 16]
        bl      func1
        mov     w0, 0
        ldr     x30, [sp, 16]
        ldp     x19, x20, [sp], 32
        ret 
```

   需要注意的是，当当前函数需要保存 frame chain 时, x29,x30 总是放在 callee-saved regs 最低端，同样是如上代码:

```
##aarch64-linux-gnu-gcc main.c -O0 -S -o main.s  -fno-omit-frame-pointer
# +--------------+ <------ sp        ## callee入口时sp
# |  x20         |
# +--------------+                 
# |  x19         |
# +--------------+ <------ sp' + 16          ## 其余寄存器依旧按照编号x19,x20顺序保存
# |  x30         |
# +--------------+ <------ sp' + 8
# |  x29         |
# +--------------+ <------ sp' = sp - 32 ## 需要frame-chain时, 总是先保存x29, x30
 
main:
        stp     x29, x30, [sp, -32]!
        mov     x29, sp                     ## 若函数需要frame chain, 则其汇编指令中会出现 mov x29,sp 指令
        stp     x19, x20, [sp, 16]
        bl      func1
        mov     w0, 0
        ldp     x19, x20, [sp, 16]
        ldp     x29, x30, [sp], 32
        ret

```

七、动态分配区域
========

  当源码中调用 alloca/__builtin_alloca 时会在当前函数栈中为此变量分配存储空间, alloca 函数转换成伪 c 代码可以表示为:

```
void * alloca(int x)
{
    sp = sp - aligned(x);
    return sp + outgoing_arg_size;      //outgoing_arg_size是编译常数
}

```

*   若当前函数没有 outgoing 区域，那么 alloca(x) 返回的分配地址总是调整后的当前硬件寄存器 sp 的值
*   若当前函数存在 outgoing 区域，那么 alloca(x) 返回的是 sp + outgonig 区域的大小 (编译常数)

动态栈分配和 outgoing 区域并不冲突，当前函数栈中:

*   [sp, sp + outgoing] 总是属于 outgoing 区域 (随 sp 变化而变化)
*   alloca(x) 执行后新分配的空间总是 [sp - x, sp] (需要对齐)

以下面的函数为例:

```
#include #include int func(int x, ...)
{
  return 0;
}
 
int x = 11;
int main(void)
{
        register unsigned long sp asm("sp");
        printf("[+] caller sp:%p, callee sp:%p\n", __builtin_dwarf_cfa(), sp);
        void * ptr = alloca(x);
        printf("[+] new callee sp:%p, alloca ptr:%p\n", sp, ptr);
        func(1,2,3,4,5,6,7,8,9);
}
 
## 运行
tangyuan@ubuntu:~/tests/gcc/aarch64_stack/alloca_test$ ./main
[+] caller sp:0x40007fff20, callee sp:0x40007ffef0
[+] new callee sp:0x40007ffee0, alloca ptr:0x40007ffef0
 
##alloca之前的栈布局
/*
 * +===============================+ <- highmem
 * |                               |
 * |  incoming stack arguments     |
 * |                               |
 * +===============================+ <-- caller sp: 0x40007fff20
 * |  ......                       |
 * |  x30                          |
 * |  x29                          |
 * +===============================+ <-- callee-saved regs: 0x40007fff00
 * |  outgoing stack arguments     | //0x10
 * |  ......                       |
 * +-------------------------------+ <-- callee sp: 0x0x40007ffef0
 */
  
 ##alloca之后的栈布局
/*
 * +===============================+ <- highmem
 * |                               |
 * |  incoming stack arguments     |
 * |                               |
 * +===============================+ <-- caller sp: 0x40007fff20
 * |  ......                       |
 * |  x30                          |
 * |  x29                          |
 * +===============================+ <-- callee-saved regs: 0x40007fff00
 * |  dynamic allocation           | //0x10
 * +-------------------------------+
 * |  padding                      |
 * |  ptr = alloca(11)             |
 * +-------------------------------+ <-- alloca ptr: 0x40007ffef0
 * |  outgoing stack arguments     | //0x10
 * |  ......                       |
 * +-------------------------------+ <--  callee sp: 0x40007ffee0
 */ 
```

八、pro/epilogue 与函数栈帧的构建
=======================

  函数栈帧是在其 prologue 中构建，在其 epilogue 中释放的。prologue 的主要工作包括两点: 1) 通过减小 sp 预留栈空间 2) 保存所有 callee-saved 寄存器; epilogue 则相反。gcc 内部主要依靠 4 个变量决定如何生成 prologue 中的指令，分别是:

*   frame_size: 当前函数编译期间确定的整个栈帧大小，其中包括 匿名寄存器参数，局部变量，callee-saved 寄存器，outgoing 区域.
*   hard_fp_offset: 除了 outgoing 区域外，其他所有区域的大小，即包括 匿名寄存器参数，局部变量，callee-saved 寄存器区域.
*   saved_regs_size: callee-saved regs 区域的大小
*   outgoing_arg_size: outgoing 区域的大小

  如下:

```
+-------------------------------+ <- highmem
|                               |
|  incoming stack arguments     |
|                               |
+-------------------------------+ <-- incoming stack (aligned)/caller sp
|                               |                  \               \
|  callee-allocated save area   |                   |               |
|  for register varargs         |                   |               |
|                               |                   |               |
+-------------------------------+                   |hard_fp_offset |
|  local variables              |                   |               |
|                               |                   |               |
+-------------------------------+                   |               |
|  padding                      | \                 |               |frame_size
+-------------------------------+  |                |               |
|  callee-saved registers       |  |saved_regs_size |               |
+-------------------------------+  |                |               |
|  LR'                          |  |                |               |
+-------------------------------+  |                |               |
|  FP'                          | /                /                |
+-------------------------------+ <-- fp(x29)                       |
|  padding                      | \                                 |
+-------------------------------+  | outgoing_arg_size              |
|  outgoing stack arguments     |  |                                |
|                               | /                                /
+-------------------------------+ <--  callee sp
|                               | <- lowmem

```

  在汇编代码中我们通常可以看到 4 种不同的指令生成方式, 按照 gcc 中的判断顺序:

1. 当函数栈较小, 且没有 outgoing 区域时
---------------------------

  以如下函数为例:

```
/* 当frame_size <= 0/256/512 且 outgoing_arg_size == 0 时, 指令模板如:
   1) stp/str R1, R2, [sp, -frame_size]!    // 优先通过此指令保存栈帧,此指令相当于 sub sp, sp, frame_size; stp r0, r1, [sp]
      add fp, sp, 0             // 此指令是否发射只取决于若当前函数需要保存frame_chain(后续模板中若无特殊用处则忽略),
                                                // 此指令若存在则 fp总是指向callee-saved regs区域的首地址
   2) stp/str R3, R4, [sp, 16]          // 按照顺序继续save后续所有寄存器
*/
int case1_0()               /* 当没有calle-saved reg 时, 不可使用此模板 */
{
}
 
int case1_1()
{      
        unsigned char a[224];       /* 当calle-saved reg = 1时, frame_size > 256 则不可使用此模板 */
        __asm__ ("":::"x19");
}
 
int case1_2()
{
        unsigned char a[480];       /* 当calle-saved reg >=2 时, frame_size > 512 则不可使用此模板 */  
        __asm__ ("":::"x19");
        __asm__ ("":::"x20");
}
 
#aarch64-linux-gnu-gcc main.c -O0 -S -o main.s
case1_0:
        nop                 /* 没有callee-saved reg无需使用此模板 */
        ret
case1_1:
        str     x19, [sp, -240]!    /* str!指令可访问的最大地址偏移 <= 256 */
        nop
        ldr     x19, [sp], 240
        ret
case1_2:
        stp     x19, x20, [sp, -496]!   /* stp!指令可访问的最大地址偏移 <= 512,故frame_size过大不可使用此模板 */
        nop
        ldp     x19, x20, [sp], 496
        ret

```

  prologue 的主要作用是调整 sp 以及保存 callee-saved regs，**优先选用 stp/str Rm, Rn, [sp, -frame_size]! 模板的原因是因为此指令可以在调整栈帧的同时直接在栈顶 save 两个寄存器**，但同时此模板也受到两个限制:

**1. frame_size 不能过大**:

    frame_size 多少算过大取决于当前函数保存的 callee-saved reg 的数量：

*   若为 0 的话，此时不需要 save，整个 prologue 最多只需要调整 sp 即可，此时使用 sub sp, sp, frame_size 指令会更快.
*   若为 1 的话, 此时只能使用 str! 指令保存一个寄存器, 而 str! 指令访问的偏移最大不超过 256.
*   如 >=2 的话, 此时可以使用 stp! 先保存两个寄存器，stp! 指令访问的最大偏移不超过 512; 若还有其他寄存器需要保存则后续使用 **stp Rm, Rn, [sp, offset]** 即可, offset 为 Rm/Rn 在 callee-saved reg 中的偏移.

**2. 当前函数不能存在 outgoing 区域:**

   若当前函数存在 outgoing 区域, 则 **stp/str Rm, Rn, [sp, -frame_size]!** 会直接将 Rm/Rn 保存到 outgoing 栈，故不能实用此模板**。**

2. 当 outgoing+callee_saved reg 区域较小时
------------------------------------

  若 1 不成立则优先考虑此模板，以如下函数为例:

```
/* 当 outgoing_args_size + saved_regs_size < 512时, 指令模板如:
   1) sub sp, sp, frame_size            //此时可以先一步到位调整栈帧
   2) stp reg1, reg2, [sp, outgoing_args_size]  //之后再依次保存所有callee-saved regs, 此时 callee-saved区域首地址为 sp + outgoing_arg_size
   3) stp reg3, reg4, [sp, outgoing_args_size + 16]    
*/
extern int func1(int x, ...);
int case2_0()
{
        func1(1,2,3,4,5,6,7,8,9);           //func1存在栈参数，故case2_0函数需要outging区域
        __asm__ ("":::"x19");
        __asm__ ("":::"x20");
}
 
int case2_1()
{
        unsigned char a[481];               //a[x], x > 480时, frame_size > 512, 此时不满足1,但满足2
        __asm__ ("":::"x19");
        __asm__ ("":::"x20");
}
 
#aarch64-linux-gnu-gcc main.c -O0 -S -o main.s
case2_0:
        sub     sp, sp, #48             ## 先一步到位调整sp
        stp     x29, x30, [sp, 16]          ## 向callee_saved reg中保存 x29, x30(此时outgoing_arg_size = 16)
        add     x29, sp, 16
        stp     x19, x20, [sp, 32]          ## 继续保存 x19, x20
        mov     w0, 9
        str     w0, [sp]
        mov     w7, 8
        ......
        mov     w1, 2
        mov     w0, 1
        bl      func1
        nop
        ldp     x19, x20, [sp, 32]
        ldp     x29, x30, [sp, 16]
        add     sp, sp, 48
        ret
case2_1:
        sub     sp, sp, #512                ## 一步到位调整sp
        stp     x19, x20, [sp]              ## 此函数没有outgoing区域，故直接基于sp保存 x19/x20
        nop
        ldp     x19, x20, [sp]
        add     sp, sp, 512
        ret

```

3. 若 outgoing+callee_saved reg 较大
---------------------------------

  若 1 不成立，且此时 outgoing + callee_saved reg 区域也较大时，则只能使用最麻烦的方法, 即:

1.  先将 sp 移动到 callee-saved register 位置
2.  save 所有 callee-saved register
3.  再将 sp 移动到 outgoing 区域结尾

  但这实际上又分为两种情况:

### 3.1 若 hard_fp_offset 区域较小

  若此时 hard_fp_offset 区域较小，则上面 1)/2) 有部分可以合并处理, 即通过 **stp reg1, reg2, [sp, -hard_fp_offset]!** 一条指令即将 sp 先指向 callee-saved regs 区域，又同时保存了两个寄存器, 如:

```
/* 当不满足1，且当 outgoing_args_size + saved_regs_size > 256/512时, 指令模板如:
   1) stp reg1, reg2, [sp, -hard_fp_offset]!    //这里和模板1的第一条指令是一样的, 区别在于这里先将sp调整到callee-saved reg的位置;
                                                //相同点是二者都save了前两个callee-saved寄存器.
   2) stp reg3, reg4, [sp, 16]          //继续保存其他callee-saved寄存器(若需要)
   3) sub sp, sp, outgoing_args_size        //最后再一次调整栈帧，hard_fp_offset + outgoing_args_size整体就是frame_size
*/
 
#define REP9(X) X,X,X,X,X,X,X,X
#define REP81(X) REP9(REP9(X))
int x;
extern int func1(int x, ...);
int case3_1(int x1, ...)
{
        __asm__ ("":::"x19","x20");
        func1(REP81 (x));           //足够大的outgoing区域，导致1,2均不满足
}
 
 
## aarch64-linux-gnu-gcc main.c -O0 -S -o main.s
case3_1:
        stp     x29, x30, [sp, -432]!           ## 由于hard_fp_offset < 512(stp!), 先调整到callee_saved区域首地址，并saved 前两个寄存器
        mov     x29, sp
        stp     x19, x20, [sp, 16]          ## 保存后续其他寄存器
        stp     x21, x22, [sp, 32]
        stp     x23, x24, [sp, 48]
        stp     x25, x26, [sp, 64]
        stp     x27, x28, [sp, 80]
        sub     sp, sp, #448                ## 最后再次调整栈帧
        ......
        bl      func1
        nop
        add     sp, sp, 448
        ldp     x19, x20, [sp, 16]
        ldp     x21, x22, [sp, 32]
        ldp     x23, x24, [sp, 48]
        ldp     x25, x26, [sp, 64]
        ldp     x27, x28, [sp, 80]
        ldp     x29, x30, [sp], 432
        ret

```

### 3.2 若 hard_fp_offset 区域也较大

那只能使用最慢的方式，如:

```
/* 这里通常包含所有其他case,此模板虽然最慢但总是可以解决问题，指令模板如:
   1) sub sp, sp, hard_fp_offset            //先调整sp指向callee-save regs区域首地址.
   2) stp x29, x30, [sp, 0]                 //save所有callee-saved寄存器
   3) sub sp, sp, outgoing_args_size        //最后再一次调整栈帧，hard_fp_offset + outgoing_args_size整体就是frame_size
*/
 
#define REP9(X) X,X,X,X,X,X,X,X
#define REP81(X) REP9(REP9(X))
int x;
extern int func1(int x, ...);
 
int case3_2(int x1, ...)
{
        unsigned char a[481];               //hard_fp_offset较大
        func1(REP81 (x));                   //outgoing区域也较大
        __asm__ ("":::"x19","x20");
}
 
## aarch64-linux-gnu-gcc main.c -O0 -S -o main.s
case3_2:
        sub     sp, sp, #928                //1)
        stp     x29, x30, [sp]              //2)
        mov     x29, sp
        stp     x19, x20, [sp, 16]
        stp     x21, x22, [sp, 32]
        stp     x23, x24, [sp, 48]
        stp     x25, x26, [sp, 64]
        stp     x27, x28, [sp, 80]
        sub     sp, sp, #448                //3)
        ......
        bl      func1
        nop
        add     sp, sp, 448
        ldp     x19, x20, [sp, 16]
        ldp     x21, x22, [sp, 32]
        ldp     x23, x24, [sp, 48]
        ldp     x25, x26, [sp, 64]
        ldp     x27, x28, [sp, 80]
        ldp     x29, x30, [sp]
        add     sp, sp, 928
        ret

```

参考资料:
=====

[1] [GCC 源码分析 (十五) — gimple 转 RTL(pass_expand)(上)_ashimida@的博客 - CSDN 博客_gcc gimple](https://blog.csdn.net/lidan113lidan/article/details/120027878)

[2] [GCC 源码分析 (八) — 语法 / 语义分析之声明与函数定义的解析_ashimida@的博客 - CSDN 博客_gcc 源码分析](https://blog.csdn.net/lidan113lidan/article/details/119976814)

[3] 《Procedure Call Standard for the ARM 64-bit Architecturn》

[4] [GCC 源码分析—shrink-wrapping_ashimida@的博客 - CSDN 博客](https://blog.csdn.net/lidan113lidan/article/details/122953830)

[5] [https://blog.csdn.net/lidan113lidan/article/details/123962416](https://blog.csdn.net/lidan113lidan/article/details/123962416)

> 版权声明：本文为笔者本人「ashimida@」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。  
> 原文链接：https://blog.csdn.net/lidan113lidan/article/details/123961152
> 
> 更多内容可关注微信公众号 ![](https://bbs.pediy.com/upload/attach/202204/490870_9XUXE8UZ8AZS3GE.jpg)

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 1 小时前 被 ashimida 编辑 ，原因：

[#基础知识](forum-41-1-130.htm)