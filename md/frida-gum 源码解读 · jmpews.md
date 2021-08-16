> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jmpews.github.io](https://jmpews.github.io/2017/06/27/pwn/frida-gum%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/)

> 前言 frida 封装的真的厉害. 这里的分析主要关注 interceptor 拦截器部分. frida 使用的是 inlinehook (废话), 模块化做的非常好, 把小模块都进行了拆分和封装. 这里......

[](#前言 "前言")前言
--------------

frida 封装的真的厉害. 这里的分析主要关注 `interceptor` 拦截器部分.

frida 使用的是 `inlinehook` (废话), 模块化做的非常好, 把小模块都进行了拆分和封装. 这里主要是为了学习 frida 进行 `inlinehook` 的套路.

这里额外提一点, frida-tracer 并不是 hook 了 `objc_msgSend` 而是进行正则匹配查找符号, 进行批量的 `inlinehook`, 所以才会在 `__handlers__` 出现那么多模块.

[](#具体分析 "具体分析")具体分析
--------------------

#### [](#前置知识 "前置知识")前置知识

```
// gobject

// http://blog.csdn.net/yanbixing123/article/details/52970804

G_DEFINE_TYPE (GumInterceptor, gum_interceptor, G_TYPE_OBJECT);
```

#### [](#拦截器初始化部分 "拦截器初始化部分")拦截器初始化部分

这里主要是初始化 `内存分配模块` 和 `调度器模块`.

内存分配模块, 预先分配了很多内存页, 针对 `darwin` 架构的系统采用的内存页分配函数是 `mach_vm_allocate` 具体可以参考 `frida-gum-master/gum/backend-darwin/gummemory-darwin.c`, 并没有使用 `mmap`, 至于为什么, 可以参考附录, 简单提一句, `darwin` 架构上的实现本质是利用 `mach_vm_allocate`.

调度器模块, 可以理解为所有被 hook 的函数都必须经过的函数, 类似于 `objc_msgSend`, 在这里通过栈来函数 (`pre_call`, `replace_call`, `post_call`) 调用顺序.

```
gum_interceptor_obtain {

  // 经过 gobject 处理

  gum_interceptor_init {

    1. 初始化拦截器的函数列表

    2. gum_code_allocator_init, 代码分配器(内存片段)初始化

    3. 拦截器后端创建 _gum_interceptor_backend_create, 并调用 gum_interceptor_backend_create_thunks 初始化入口点和离开点, 包括 enter_thunk 和 leave_thunk. 其中 enter_thunk 主要作用: "构造自己的函数栈(包括返回地址和寄存器状态), 并调用函数 _gum_function_context_begin_invocation. ", 其中 _gum_function_context_begin_invocation 用于引导调用过程, 之后需要具体具体分析, leave_thunk 作用同样类似(_gum_function_context_end_invocation).

    4. gum_interceptor_transaction_init

  }

}
```

#### [](#添加-hook-listenr-构造跳板 "添加 hook-listenr 构造跳板")添加 hook-listenr 构造跳板

这里涉及到如何构造跳板以及指令修复.

跳板函数的构造, 跳板函数主要作用就是进行跳转, 并准备 `跳转目标` 需要的参数. 举个例子, 被 hook 的函数经过入口跳板 (`enter_trampoline`), 跳转到调度函数 (`enter_chunk`), 需要被 hook 的函数相关信息等, 这个就需要在构造跳板是完成

指令修复, 由于涉及到覆盖原函数指令, 这里需要进行备份原指令, 由于备份地址发生改变, 需要修改跟 pc 相关的指令.

```
gum_interceptor_attach_listener {

  gum_interceptor_transaction_begin {

  }

  gum_interceptor_instrument {

    /*

      创建跳板 on_enter_trampoline, on_leave_trampoline, on_invoke_trampoline 分别用于跳到 enter_thunk ,leave_thunk 和 正常函数.

      ATTENTION: 这个过程涉及到对于设计 PC 寄存器的指令进行修复的过程.

     */

    _gum_interceptor_backend_create_trampoline

    /*

      把当前的 transaction 加入任务, 添加 callback 函数 `gum_interceptor_activate`

     */

    gum_interceptor_transaction_schedule_prologue_write

  }

  gum_interceptor_transaction_end {

    /*

      激活拦截器, 激活跳板, 这一部分涉及对当原函数的指令的覆盖写

     */

    gum_interceptor_activate {

      /*

        设置跳转到 on_enter_trampoline

       */

      _gum_interceptor_backend_activate_trampoline

    }

  }

}
```

#### [](#函数调用导向 "函数调用导向")函数调用导向

在 `frida-gum` 中是指 `listener`, 通用说应该是 `pre_call` 和 `post_call`.

如何利用栈内的保存的函数返回地址, 进行函数调用顺序导向?

处理函数在 `frida-gum-master/gum/backend-arm64/guminterceptor-arm64.c:gum_emit_enter_thunk`, 可以具体对照参考.

```
原函数栈(不可动)

_____________

经过跳板(enter_trampoline)

_____________

进入调度中心(enter_chunk)

函数寄存器状态保存(gum_emit_prolog), 需要进行压栈操作保存(x16, x30)

_____________

进入调度函数(enter_thunk), 包含三个关键参数: 函数信息, x16, x30

其中 x16 作为 next_hop, x30 作为返回地址, x30 需要根据不同情况进行设置

调用 pre_call, 由于此时栈地址发生改变, 已经不能通过普通的方式获取到参数, 只能根据之前保存寄存器获取参数(存在 valist 的情况) 

_____________

寄存器状态恢复(gum_emit_epilog), 导致修改(x16, x30), 并通过 br x16, 跳转到下一个跳跃点

_____________

如果存在 on_leave, 则 x30 被改变, 会在整个函数执行完毕后跳转到 leave_trampolien.
```

#### [](#被拦截函数指令执行流程 "被拦截函数指令执行流程")被拦截函数指令执行流程

```
1. 函数的入口指令被修改, 先跳转到入口跳板(`on_enter_trampoline`), 之后跳转到 `enter_thunk`

2. 根据之前构造的 `enter_thunk`, 跳转到 `_gum_function_context_begin_invocation`, 并在这里触发 on_enter 函数的调用. 这里利用栈返回地址进行函数跳转导向. 比如 `*caller_ret_addr = function_ctx->on_leave_trampoline;` 以及 `*next_hop = function_ctx->on_invoke_trampoline;`.

3. 开始执行 `on_invoke_trampoline` 函数执行完毕, 由于 `*caller_ret_addr = function_ctx->on_leave_trampoline`, 所以会自动跳转到 `on_leave_trampoline` 继续执行.

4. 随后跳转到 `_gum_function_context_end_invocation`, 并在这里触发 on_leave 的调用.
```

#### [](#被拦截函数指令执行流程-流程图表示 "被拦截函数指令执行流程(流程图表示)")被拦截函数指令执行流程 (流程图表示)

这里演示了如何利用栈返回地址控制函数指令流程, 感觉和 `ROP` 有点像.

```
1.  on_enter_trampoline

2.  enter_thunk

3.  _gum_function_context_begin_invocation

4.  调用 on_enter

5.  修改返回地址(on_leave_trampoline, on_invoke_trampoline)

6.  利用返回地址进入 on_invoke_trampoline

7.  调用原函数

8.  利用返回地址进入 on_leave_trampoline

9.  leave_thunk

10. _gum_function_context_end_invocation

11. 调用 on_leave

12. 正常执行
```

#### [](#code-patch-内存属性修改 "code patch(内存属性修改)")code patch(内存属性修改)

```
substitute/lib/darwin/execmem.c:execmem_foreign_write_with_pc_patch

frida-gum-master/gum/gummemory.c:gum_memory_patch_code
```

对原函数进行 code patch, 需要修改内存属性 `r-x` 为 `rw-`, 在修改完后重新修改为 `r-x`, 很容易想到的就是 `mach_vm_protect`, 这里存在一个坑, 其实只是需要几十字节的内存属性的变更. 但是 `mach_vm_protect`会做修正, 起始地址必须是页对齐, 以及长度必须是页大小的倍数. 这就导致一个问题, 在修改函数入口指令时, 会导致整个页的内存属性为 `rw-`, 任何执行到此地址范围的之指令都会异常. 但是也可以判断内存也是否支持 `rwx`, 大致的判断判方法就是, 先分配一页内存, 之后尝试设置内存属性 `(PROT_READ | PROT_WRITE | PROT_EXEC)` 看是否确实设置为 `rwx`

这里介绍在 `substrate` 和 `frida-gum` 中使用到的方法, 两个方法稍有不同, 但本质都是利用 `mmap`.

在 `substrate` 中先分配一页内存, 复制目标函数那一页的内容到该页, 在该页做内存属性修改和 code patch, 之后需要修改该页内存属性为 `r-x`, 最后通过 `mach_vm_remap` 函数重新映射到 目标内存页.

在 `frida-gum` 中前面的步骤类似, 最后一步有所不同, `frida-gum` 将需要替换的指令持久化成一个临时文件, 之后通过 `mmap` + 文件描述符, 重新映射到目标内存页.

[](#总结 "总结")总结
--------------

对几个方面做一个总结

#### [](#模块化方面 "模块化方面")模块化方面

frida 把一部分都分省了单个小模块, 以保证自由度. 例如: `on_enter_trampoline` 和 `enter_thunk` 每一部分只负责一小部分功能.

#### [](#指令修复部分 "指令修复部分")指令修复部分

指令修复是保存原始指令很重要的一步. 这里以 `arm64` 的指令修复为例. 主要实现在函数 `gum_arm64_relocator_write_one`.

大致步骤是需要使用 `capstone` 判断指令 `id`, 是否为 `PC` 相关的指令, 这里可以参考 `ARM Architecture Reference Manual ARMv8, for ARMv8-A architecture profile> C6.1.2 Use of the PC`, 这其实也就是 `frida-gum` 如何判断哪些指令进行修复.

#### [](#hook-思路方面 "hook 思路方面")hook 思路方面

仔细研究会发现其实 frida 的 `Interceptor(拦截器)` 和 `objc_msgSend` 有异曲同工之妙, 这也是之前想搞的一个思路, 所有被拦截 (hook) 的函数都会经过 `interceptor_backend`(`enter_thunk` 和 `leave_thunk`) 进行之后的分支跳转, 比如 `on_enter` 或 `on_leave`.

所有被 hook 的函数都被封装为 `_GumFunctionContext` , 之后根据 `target` 函数地址进行 hashmap 快速的查找.

如果再说一点就是利用栈返回地址控制函数指令流程, 有点像 `ROP` 的思路.

[](#附录 "附录")附录
--------------

```
# 内存分配

  substitute/lib/darwin/execmem.c:gum_alloc_n_pages

  frida-gum-master/gum/backend-darwin/gummemory-darwin.c:gum_alloc_n_pages

  mach mmap use __vm_allocate and __vm_map

  https://github.com/bminor/glibc/blob/master/sysdeps/mach/hurd/mmap.c

  https://github.com/bminor/glibc/blob/master/sysdeps/mach/munmap.c

  http://shakthimaan.com/downloads/hurd/A.Programmers.Guide.to.the.Mach.System.Calls.pdf

#
```