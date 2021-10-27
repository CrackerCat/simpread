> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269964.htm)

> Unidbg 的底层支持 - Unicorn

Unidbg 的底层支持 - Unicorn
======================

*   [Unidbg 的底层支持 - Unicorn](#unidbg的底层支持-unicorn)
    *   [Unicorn 简介](#unicorn简介)
    *   [Unicorn 使用步骤](#unicorn使用步骤)
        *   [引入 Unicorn](#引入unicorn)
        *   [使用 Unicorn](#使用unicorn)
        *   [Unicorn 原生 Hook](#unicorn原生hook)
            *   [CodeHook](#codehook)
            *   [BlockHook](#blockhook)
            *   [ReadHook](#readhook)
            *   [WriteHook](#writehook)
            *   [MemHook](#memhook)
            *   [InterruptHook](#interrupthook)
            *   [EventMemHook](#eventmemhook)
    *   [Keystone](#keystone)
        *   [引入](#引入)
        *   [使用](#使用)
    *   [Capstone](#capstone)
        *   [引入](#引入-1)
        *   [使用](#使用-1)
    *   [总结](#总结)

Unicorn 简介
----------

> https://github.com/unicorn-engine/unicorn

 

Unicorn 的官网简介很简单明了：Unicorn 是一个基于 Qemu 的轻量级的多平台、多架构的 CPU 模拟器框架。一句话让我们明白了它是做什么的。在 Unidbg 对一个 Elf 文件进行模拟执行的时候，我们一般是在跨平台运行的，所以就底层就需要一个模拟器 Backend。其中一种就是 Unicorn，下面我们就来看看 Unicorn 的使用

Unicorn 使用步骤
------------

*   语言: Java
*   架构: Arm

我们使用一个例子，就可以讲述 Unicorn 的使用。

### 引入 Unicorn

Unicorn 是开源的，可自行编译。凯神已经编译好了，可以直接使用

```
 com.github.zhkl0228
    unicorn
    1.0.12 

```

### 使用 Unicorn

我们先来写一段简单的 Thumb 汇编代码

```
movs r0, #3
movs r1, #2
add  r0, r1

```

这三行汇编如果被 CPU 执行，结束后 R0 寄存器的值应该为 5 对吧。那么我们就看一下用 Unicorn 如何来模拟执行这三条指令

 

Unicorn 是不认识汇编代码的，它跟 CPU 一样，只认识机器码，所以我们可以去下面的网站在线转换成机器码:

> https://armconverter.com/

 

转换成机器码后

```
0x0320
0x0221
0x0844

```

```
public class UnicornTest {
 
    long BASE = 0x1000;
 
    @Test
    public void test(){
        // 先来定义我们的机器码
        byte[] code = new byte[]{
            0x03,0x20,
            0x02,0x21,
            0x08,0x44};
 
        // 创建 Unicorn 对象, 它就像一个CPU, 我们可以使用它的接口来操作这个CPU
        // 参数一：架构
        // 参数二：运行模式(在ARM、Thumb中无关紧要，最终还是靠运行时判断)
        Unicorn unicorn = new Unicorn(UnicornConst.UC_ARCH_ARM,UnicornConst.UC_MODE_THUMB);
 
        // mem_map 进行内存映射，在使用Unicorn提供的内存时，必须先进行映射
        // 参数一：起始地址
        // 参数二：映射内存区域的大小
        // 参数三：内存操作权限
        unicorn.mem_map(BASE,0x1000,UnicornConst.UC_PROT_WRITE | UnicornConst.UC_PROT_READ | UnicornConst.UC_PROT_EXEC);
 
        // mem_write 可以进行对映射出的内存进行写入操作
        // 参数一：写入的地址
        // 参数二：写入的内容
        unicorn.mem_write(BASE,code);
 
        // emu_start 开始执行CPU
        // 参数一：开始执行的地址(Thumb指令地址+1)
        // 参数二：结束地址(当结束地址命中时，结束执行)
        // 参数三：超时时间(ms)，当该值为0时，Unicorn将在无限时间内模拟代码，直到模拟完成
        unicorn.emu_start(BASE+1,BASE+code.length,0,0);
 
        // 此时三条Thumb指令已经模拟完成
        // reg_read 可以读取寄存器，相应的reg_write可以写入寄存器的值
        // 参数：寄存器常量
        Long o = (Long) unicorn.reg_read(ArmConst.UC_ARM_REG_R0);
 
        System.out.println("the emulate finished result is ==> "+o.intValue());
 
    }
}

```

我们来看下执行结果, 符合我们的预期

```
the emulate finished result is ==> 5

```

对于上面代码的补充：在 mem_map 的时候，参数一起始地址必须是 1K(32 位下)/4K(64 位下) 对齐的，否则会抛出 Invalid argument (UC_ERR_ARG) 异常，参数三的操作权限可参考下表:

```
public static final int UC_PROT_NONE = 0;
public static final int UC_PROT_READ = 1;
public static final int UC_PROT_WRITE = 2;
public static final int UC_PROT_EXEC = 4;
public static final int UC_PROT_ALL = 7;

```

### Unicorn 原生 Hook

#### CodeHook

```
// hook_add 添加一个Hook(后面的Hook参数通用，只有第一个不同)
// 参数一：Hook回调
// 参数二：Hook起始地址
// 参数三：Hook结束地址
// 参数四：自定义参数，可以在Hook的回调中拿到，也就是 Object user
unicorn.hook_add(new CodeHook() {
        /* CodeHook将注册UC_HOOK_CODE类型的Hook，它将在Unicorn对起始地址跟结束地址中间每一条指令的执行时进行调用，当结束地址 < 起始地址时，则将对每一条指令执行时进行调用，相当于Trace*/
        /* 回调函数有当前Unicorn对象，所以我们可以基于此对象做关于CPU的任何事情*/
        @Override
        public void hook(Unicorn u, long address, int size, Object user) {
            System.out.print(String.format(">>> Tracing instruction at 0x%x, instruction size = 0x%x\n", address, size));
        }
    },0,-1,null);

```

```
>>> Tracing instruction at 0x1000, instruction size = 0x2
>>> Tracing instruction at 0x1002, instruction size = 0x2
>>> Tracing instruction at 0x1004, instruction size = 0x2
the emulate finished result is ==> 5

```

#### BlockHook

```
/* BlockHook将注册UC_HOOK_BLOCK类型的Hook，当输入的基本块且基本块的地址(BB)在起始地址<=BB<=结束地址范围内时，将调用已注册的回调函数。当结束地址 < 起始地址时，输入任何基本块时，将调用回调*/
/* 基本块：在Unicorn中，基本块就相当于未发生跳转的所有指令为一基本块，发生跳转就是另一块*/
unicorn.hook_add(new BlockHook() {
            @Override
            public void hook(Unicorn u, long address, int size, Object user) {
                System.out.print(String.format(">>> Tracing basic block at 0x%x, block size = 0x%x\n", address, size));
            }
        }, 0, -1, null);

```

```
>>> Tracing basic block at 0x1000, block size = 0x6
the emulate finished result is ==> 5

```

如果我们执行一段有跳转的汇编 (后面我们测试也使用这段汇编代码)

```
0x1000: movs r0, #3
0x1002: movs r1, #2
0x1004: add r0, r1
0x1006: bl #0x1100
0x100a: add r0, r1
...
0x1100: add r0, r1
0x1102: movs.w r2, #0x1200
0x1106: str.w lr, [r2]
0x110a: ldr.w pc, [r2]
...

```

那么我们再来看 BlockHook 的结果

```
>>> Tracing basic block at 0x1000, block size = 0xa
>>> Tracing basic block at 0x1100, block size = 0xe
>>> Tracing basic block at 0x100a, block size = 0x2
the emulate finished result is ==> 9

```

#### ReadHook

```
/* ReadHook注册类型为UC_HOOK_MEM_READ的Hook，只要在地址范围起始地址<=read_addr<=结束地址内执行内存读取，就会调用已注册的回调函数。当结束地址 < 起始地址时，将为所有内存读取调用回调*/
/* 注意：这个内存读取必须由指令进行，如果使用Unicorn的api mem_read进行读取内存，是不会进行回调的*/
unicorn.hook_add(new ReadHook() {
    @Override
    public void hook(Unicorn u, long address, int size, Object user) {
        byte[] bytes = u.mem_read(address, size);
        System.out.print(String.format(">>> Memory read at 0x%x, block size = 0x%x, value is = 0x%s\n", address, size, Integer.toHexString(bytes[0] & 0xff)));
    }
}, 0, -1, null);

```

我们执行上面的汇编指令，输出结果。就说明在 0x1200 地址上，有对内存的读取操作

```
>>> Memory read at 0x1200, block size = 0x4
the emulate finished result is ==> 9

```

#### WriteHook

```
/* WriteHook注册类型为UC_HOOK_MEM_WRITE的Hook，每当在地址范围起始地址<=write_addr<=结束地址内执行内存写入时，将调用已注册的回调函数。当结束地址 < 起始地址时，将为所有内存写入调用回调。*/
/* 注意：这个内存写入操作必须由指令进行，同ReadHook*/
unicorn.hook_add(new WriteHook() {
    @Override
    public void hook(Unicorn u, long address, int size, long value, Object user) {
        System.out.print(String.format(">>> Memory write at 0x%x, block size = 0x%x, value is = 0x%x\n", address, size, value));
    }
}, 0, -1, null);

```

```
>>> Memory write at 0x1200, block size = 0x4, value is = 0x100b
the emulate finished result is ==> 9

```

#### MemHook

MemHook 就是 ReadHook 跟 WriteHook 的结合体，就不单独介绍了

#### InterruptHook

```
/* InterruptHook注册类型为UC_HOOK_INTR的Hook，每当执行中断指令时，将调用已注册的回调函数。*/
/* 在Linux的Arm架构中，所有的系统调用都是通过中断进入，且只有这一条入口，所以我们可以通过InterruptHook来处理系统调用*/
unicorn.hook_add(new InterruptHook() {
    @Override
    public void hook(Unicorn u, int intno, Object user) {
        System.out.print(String.format(">>> Interrup occur, intno = %d\n",intno));
    }
}, null);

```

加一条 svc #0 指令，产生异常

```
>>> Interrup occur, intno = 2
the emulate finished result is ==> 9

```

#### EventMemHook

```
/* 注册类型为UC_HOOK_MEM_XXX_UNMAPPED 或者 UC_HOOK_MEM_XXX_PROT的Hook，每当尝试从无效或受保护的内存地址进行读取或写入时，将调用注册的回调函数。*/
unicorn.hook_add(new EventMemHook() {
    @Override
    public boolean hook(Unicorn u, long address, int size, long value, Object user) {
         System.out.println(String.format("UC_HOOK_MEM_READ_UNMAPPED at 0x%x, value = 0x%x",address,value));
        return true;
    }
}, UC_HOOK_MEM_READ_UNMAPPED, null);

```

我们加段指令，来读取 0x2000(此处未 map) 的内容

```
movs r2, 0x2000
ldr r3, [r2]

```

```
UC_HOOK_MEM_READ_UNMAPPED at 0x2000, value = 0x0
unicorn.UnicornException: Invalid memory read (UC_ERR_READ_UNMAPPED)

```

Keystone
--------

接下来我们介绍下 Keystone 跟 Capstone 这对兄弟，先来看 Keystone。

 

上面我们使用在线网站进行汇编代码与机器码的相互转换，使用起来非常的麻烦，我们就可以借助 Keystone 这个项目来帮助我们对汇编代码，直接转换成机器码

### 引入

```
 com.github.zhkl0228
    keystone
    0.9.5 

```

### 使用

```
Keystone keystone = new Keystone(KeystoneArchitecture.Arm, KeystoneMode.ArmThumb);
String assembly =
        "movs r0, #3\n" +
        "movs r1, #2\n" +
        "add r0,r1";
byte[] code = keystone.assemble(assembly).getMachineCode();
// code = [0x03, 0x20, 0x02, 0x21, 0x08, 0x44]

```

Capstone
--------

Capstone 跟 Keystone 是相反的

### 引入

```
 com.github.zhkl0228
    capstone
    3.0.11 

```

### 使用

我们跟 CodeHook 结合来看下 Capstone 如何使用，也顺便看下它们的结合使用

```
unicorn.hook_add(new CodeHook() {
    @Override
    public void hook(Unicorn u, long address, int size, Object user) {
        Capstone capstone = new Capstone(Capstone.CS_ARCH_ARM, Capstone.CS_MODE_THUMB);
        Capstone.CsInsn[] disasm = capstone.disasm(u.mem_read(address, size), address);
        for (Capstone.CsInsn ins : disasm) {
            System.out.println("0x"+Long.toHexString(ins.address) + ": " + ins.mnemonic + " " + ins.opStr );
        }
    }
}, 0, -1, null);

```

```
0x1000: movs r0, #3
0x1002: movs r1, #2
0x1004: add r0, r1
0x1006: bl #0x1100
0x1100: add r0, r1
0x1102: movs.w r2, #0x1200
0x1106: str.w lr, [r2]
0x110a: ldr.w pc, [r2]
0x100a: svc #0
0x100c: add r0, r1
the emulate finished result is ==> 9

```

总结
--

上面我们介绍了 Unicorn 的详细使用，有了 Unicorn 的加持，我们就可以模拟 Arm 架构的指令集，从而执行 So 中的各种指令，当然多条指令组合成的函数都是可以的。我们可以看到 Unicorn 本身提供了丰富的 API 供我们使用，它本身提供的 Hook，覆盖了我们需要实现的所有场景。最后介绍了 Keystone 跟 Capstone 这对兄弟，那么本次的分享就结束啦，对本块内容有兴趣的朋友可以加个 VX 一起学习呀: roy5ue

[2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

最后于 2 天前 被 r0ysue 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm) [#工具脚本](forum-161-1-128.htm)