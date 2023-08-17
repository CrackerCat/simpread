> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278423.htm)

> [原创]Frida-gum 源代码速通笔记

[原创]Frida-gum 源代码速通笔记

1 天前 327

### [原创]Frida-gum 源代码速通笔记

 [![](http://passport.kanxue.com/upload/avatar/548/924548.png?1681269251)](user-home-924548.htm) [Tokameine](user-home-924548.htm) ![](https://bbs.kanxue.com/view/img/rank/9.png) 2  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 1 天前  327

前言
==

最近做一些逆向的时候卡住了，感觉自己对 frida 的了解过于浅薄了，由于自己对安全研究的一些坏习惯，因此不读一下 frida 的源码就理解不了它的实现原理，于是直接就拿起来开始看了。（尽管现在做的不是安全研究，但希望这种习惯能延续下去吧。）

不得不说，frida 的代码写的是真的赏心悦目，像我这样阅读代码的苦手都能大致通过语义理解原理，我只能说，非常有感觉！

源代码目录
=====

Frida 的源代码目录结构按照模块进行区分，本文只选择其中几个笔者认为比较重要的部分进行分析：

*   **frida-core** / frida 的主要功能实现模块
*   **frida-gum** / 提供 inline hook 、代码跟踪、内存监控、符号查找以及其他多个上层基础设施实现
    *   **gum-js** / 为 frida-gum 提供 JavaScript 语言的接口

出于完整性考虑，笔者也把其他比较重要的模块的介绍贴在这里。如有需要，读者可以自行去深入了解：

> **frida-python**: Frida Python bindings  
> **frida-node**: Frida Node.js bindings  
> **frida-qml**: Frida Qml plugin  
> **frida-swift**: Frida Swift bindings  
> **frida-tools**: Frida CLI tools

本文中，笔者将按照自顶向下的方法去分析对应模块的功能实现。但本篇仅涉及到 Frida-gum 部分，Frida-core 将在另外一篇文章中介绍。

frida-gum
=========

frida-gum 的实现结果是跨架构跨平台的，为此它抽象出了架构无关 / 平台无关 / 系统无关的 api 供用户使用。

该模块中有几个较为关心的子模块：

*   Interceptor: 提供 inline hook 的封装
*   Stalker: 用于跟踪指令
*   MemoryAccessMonitor: 内存监控

Interceptor
-----------

我们从测试样例开始：

```
TESTLIST_BEGIN (interceptor_arm64)
    TESTENTRY (attach_to_thunk_reading_lr)
    TESTENTRY (attach_to_function_reading_lr)
TESTLIST_END ()

```

样例分为了对部分代码或整体函数进行钩取两种，似乎没什么区别，不妨先从函数开始。

```
TESTCASE (attach_to_function_reading_lr)
{
  const gsize code_size_in_pages = 1;
  gsize code_size;
  GumEmitLrFuncContext ctx;
   
  code_size = code_size_in_pages * gum_query_page_size ();
  ctx.code = gum_alloc_n_pages (code_size_in_pages, GUM_PAGE_RW);
  ctx.run = NULL;
  ctx.func = NULL;
  ctx.caller_lr = 0;
   
  gum_memory_patch_code (ctx.code, code_size, gum_emit_lr_func, &ctx);
   
  g_assert_cmphex (ctx.run (), ==, ctx.caller_lr);
   
  interceptor_fixture_attach (fixture, 0, ctx.func, '>', '<');
  g_assert_cmphex (ctx.run (), !=, ctx.caller_lr);
  g_assert_cmpstr (fixture->result->str, ==, "><");
   
  interceptor_fixture_detach (fixture, 0);
  gum_free_pages (ctx.code);
}

```

frida 从 `interceptor_fixture_attach` 函数开始去 hook 对应函数，向下跟进可以找到实现函数：

```
static GumAttachReturn
interceptor_fixture_try_attach (InterceptorFixture * h,
                                guint listener_index,
                                gpointer test_func,
                                gchar enter_char,
                                gchar leave_char)
{
  GumAttachReturn result;
  Arm64ListenerContext * ctx;
 
  ctx = h->listener_context[listener_index];
  if (ctx != NULL)
  {
    arm64_listener_context_free (ctx);
    h->listener_context[listener_index] = NULL;
  }
 
  ctx = g_slice_new0 (Arm64ListenerContext);
 
  ctx->listener = test_callback_listener_new ();
  ctx->listener->on_enter =
      (TestCallbackListenerFunc) arm64_listener_context_on_enter;
  ctx->listener->on_leave =
      (TestCallbackListenerFunc) arm64_listener_context_on_leave;
  ctx->listener->user_data = ctx;
 
  ctx->fixture = h;
  ctx->enter_char = enter_char;
  ctx->leave_char = leave_char;
 
  result = gum_interceptor_attach (h->interceptor, test_func,
      GUM_INVOCATION_LISTENER (ctx->listener), NULL);
  if (result == GUM_ATTACH_OK)
  {
    h->listener_context[listener_index] = ctx;
  }
  else
  {
    arm64_listener_context_free (ctx);
  }
 
  return result;
}

```

可以注意到，其中 `on_enter` 和 `on_leave` 是可以由用户自行重载的。然后再从 `gum_interceptor_attach` 进入，该函数包括了布置 hook 并启动 hook 的任务：

```
GumAttachReturn
gum_interceptor_attach (GumInterceptor * self,
                        gpointer function_address,
                        GumInvocationListener * listener,
                        gpointer listener_function_data)
{
  GumAttachReturn result = GUM_ATTACH_OK;
  GumFunctionContext * function_ctx;
  GumInstrumentationError error;
 
  gum_interceptor_ignore_current_thread (self);
  GUM_INTERCEPTOR_LOCK (self);
  gum_interceptor_transaction_begin (&self->current_transaction);
  self->current_transaction.is_dirty = TRUE;
//1. 获得需要钩取的函数地址
  function_address = gum_interceptor_resolve (self, function_address);
 
//2. 此处用于构造跳板、布置 hook，并将该函数抽象为一个 ctx 结构体
//后续对该函数的引用都将使用 ctx 指代该函数
  function_ctx = gum_interceptor_instrument (self, GUM_INTERCEPTOR_TYPE_DEFAULT,
      function_address, &error);
 
  if (function_ctx == NULL)
    goto instrumentation_error;
//3. 添加监听器
  if (gum_function_context_has_listener (function_ctx, listener))
    goto already_attached;
 
  gum_function_context_add_listener (function_ctx, listener,
      listener_function_data);
 
  goto beach;
 
instrumentation_error:
  {
    switch (error)
    {
      case GUM_INSTRUMENTATION_ERROR_WRONG_SIGNATURE:
        result = GUM_ATTACH_WRONG_SIGNATURE;
        break;
      case GUM_INSTRUMENTATION_ERROR_POLICY_VIOLATION:
        result = GUM_ATTACH_POLICY_VIOLATION;
        break;
      case GUM_INSTRUMENTATION_ERROR_WRONG_TYPE:
        result = GUM_ATTACH_WRONG_TYPE;
        break;
      default:
        g_assert_not_reached ();
    }
    goto beach;
  }
already_attached:
  {
    result = GUM_ATTACH_ALREADY_ATTACHED;
    goto beach;
  }
beach:
  {
//4. 注入跳板
    gum_interceptor_transaction_end (&self->current_transaction);
    GUM_INTERCEPTOR_UNLOCK (self);
    gum_interceptor_unignore_current_thread (self);
 
    return result;
  }
}

```

笔者已经在上述代码的诸事中大致描述了关键部分的代码功能，在这里需要为此做一些额外说明。

所谓 **inline hook** 的工作原理是：将函数开头的指令替换为跳转指令，使得函数在执行时先跳转到 hook 到 `on_entry` 函数中，然后再从中返回执行原函数。

其中，用于从函数开头跳转到 `on_entry` 中的指令被称之为 `跳板`，而构造跳板首先需要获得 hook 函数在内存中的地址。这个操作在本文中不会详细介绍，若读者有兴趣了解原理可以自行阅读代码。

而监听器 (Listener) 的作用则是一个用于记录相关监视数据的结构体，对于已经被 hook 过的函数是不需要添加两个监听器的。

接下来我们跟入 `gum_interceptor_instrument` ：

```
static GumFunctionContext *
gum_interceptor_instrument (GumInterceptor * self,
                            GumInterceptorType type,
                            gpointer function_address,
                            GumInstrumentationError * error)
{
  GumFunctionContext * ctx;
 
  *error = GUM_INSTRUMENTATION_ERROR_NONE;
//1. 获得需要 hook 的函数对象
//该对象在第一次调用 gum_interceptor_instrument 进行初始化入表
//此处由于第一次调用的缘故，在表里查询不到 ctx
//因此会继续往下器创建该 ctx
  ctx = (GumFunctionContext *) g_hash_table_lookup (self->function_by_address,
      function_address);
 
  if (ctx != NULL)
  {
    if (ctx->type != type)
    {
      *error = GUM_INSTRUMENTATION_ERROR_WRONG_TYPE;
      return NULL;
    }
    return ctx;
  }
 
  if (self->backend == NULL)
  {
//2. 构造三级跳板
    self->backend =
        _gum_interceptor_backend_create (&self->mutex, &self->allocator);
  }
//3. 初始化 ctx
  ctx = gum_function_context_new (self, function_address, type);
 
  if (gum_process_get_code_signing_policy () == GUM_CODE_SIGNING_REQUIRED)
  {
    if (!_gum_interceptor_backend_claim_grafted_trampoline (self->backend, ctx))
      goto policy_violation;
  }
  else
  {
//4. 构造二级跳板
    if (!_gum_interceptor_backend_create_trampoline (self->backend, ctx))
      goto wrong_signature;
  }
//5. 函数入表，表示已经完成基本操作
  g_hash_table_insert (self->function_by_address, function_address, ctx);
//6. 添加任务，设置回调函数 gum_interceptor_activate 用于激活跳板
  gum_interceptor_transaction_schedule_update (&self->current_transaction, ctx,
      gum_interceptor_activate);
 
  return ctx;
 
policy_violation:
  {
    *error = GUM_INSTRUMENTATION_ERROR_POLICY_VIOLATION;
    goto propagate_error;
  }
wrong_signature:
  {
    *error = GUM_INSTRUMENTATION_ERROR_WRONG_SIGNATURE;
    goto propagate_error;
  }
propagate_error:
  {
    gum_function_context_finalize (ctx);
 
    return NULL;
  }
}

```

在注释中，笔者提到了二级跳板和三级跳板，那么一级跳板是什么？

一级跳板其实就是函数开头的跳转指令，该指令将会让程序跳转到二级跳板中，而二级跳板会转入三级跳板，最后由三级跳板分发，选择用户提供的 `on_entry` 函数进行调用。

### _gum_interceptor_backend_create

首先创建的是三级跳板，因此我们跟到 `_gum_interceptor_backend_create` 里看看它是如何实现的。该函数是平台相关的具体函数，由于笔者打算分析 arm64 下的实现，因此这里的源代码应为 frida-gum/gum/backend-arm64/guminterceptor-arm64.c：

```
GumInterceptorBackend *
_gum_interceptor_backend_create (GRecMutex * mutex,
                                 GumCodeAllocator * allocator)
{
  GumInterceptorBackend * backend;
 
  backend = g_slice_new0 (GumInterceptorBackend);
  backend->mutex = mutex;
  backend->allocator = allocator;
 
  if (gum_process_get_code_signing_policy () == GUM_CODE_SIGNING_OPTIONAL)
  {
    gum_arm64_writer_init (&backend->writer, NULL);
    gum_arm64_relocator_init (&backend->relocator, NULL, &backend->writer);
 
    gum_interceptor_backend_create_thunks (backend);
  }
 
  return backend;
}

```

跟入 `gum_interceptor_backend_create_thunks` ：

```
static void
gum_interceptor_backend_create_thunks (GumInterceptorBackend * self)
{
  gsize page_size, code_size;
 
  page_size = gum_query_page_size ();
  code_size = page_size;
 
  self->thunks = gum_memory_allocate (NULL, code_size, page_size, GUM_PAGE_RW);
  gum_memory_patch_code (self->thunks, 1024,
      (GumMemoryPatchApplyFunc) gum_emit_thunks, self);
}

```

此处通过调用 `gum_memory_patch_code` 把 `gum_emit_thunks` 的实现写入到 `self->thunks` 中，因此我们这里跟入 `gum_emit_thunks` ：

```
static void
gum_emit_thunks (gpointer mem,
                 GumInterceptorBackend * self)
{
  GumArm64Writer * aw = &self->writer;
 
  self->enter_thunk = self->thunks;
  gum_arm64_writer_reset (aw, mem);
  aw->pc = GUM_ADDRESS (self->enter_thunk);
  gum_emit_enter_thunk (aw);//1. 此处创建三级跳板 enter_thunk
  gum_arm64_writer_flush (aw);
 
  self->leave_thunk =
      (guint8 *) self->enter_thunk + gum_arm64_writer_offset (aw);
  gum_emit_leave_thunk (aw);//2. 此处创建三级跳板 leave_thunk
  gum_arm64_writer_flush (aw);
}

```

此处涉及到了具体的三级跳板的创建，分别由 `gum_emit_enter_thunk` 和 `gum_emit_leave_thunk` 完成，这里笔者先从 `gum_emit_enter_thunk` 进行分析：

```
static void
gum_emit_enter_thunk (GumArm64Writer * aw)
{
  gum_emit_prolog (aw);//1. 保存上下文信息
 
//2. add x1,sp,0
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X1, ARM64_REG_SP,
      GUM_FRAME_OFFSET_CPU_CONTEXT);
//3. add x2,sp,G_STRUCT_OFFSET (GumCpuContext, lr)
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X2, ARM64_REG_SP,
      GUM_FRAME_OFFSET_CPU_CONTEXT + G_STRUCT_OFFSET (GumCpuContext, lr));
//4. add x3,sp,sizeof (GumCpuContext)
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X3, ARM64_REG_SP,
      GUM_FRAME_OFFSET_NEXT_HOP);
//5. call _gum_function_context_begin_invocation(x17,x1,x2,x3)
  gum_arm64_writer_put_call_address_with_arguments (aw,
      GUM_ADDRESS (_gum_function_context_begin_invocation), 4,
      GUM_ARG_REGISTER, ARM64_REG_X17,
      GUM_ARG_REGISTER, ARM64_REG_X1,
      GUM_ARG_REGISTER, ARM64_REG_X2,
      GUM_ARG_REGISTER, ARM64_REG_X3);
 
  gum_emit_epilog (aw);
}

```

此处主要负责调用四级跳板 `_gum_function_context_begin_invocation` 并进行传参，跟入该函数：

```
void
_gum_function_context_begin_invocation (GumFunctionContext * function_ctx,
                                        GumCpuContext * cpu_context,
                                        gpointer * caller_ret_addr,
                                        gpointer * next_hop)

```

注意，此处第三个参数 `caller_ret_addr` 代表的是**被 hook** 函数用于储存返回地址的内存地址，而第四个参数则是四级跳板返回时执行的下一个函数地址。

稍微向下看看函数的实现：（省略部分）

```
//1. 如果替换了函数实现，或注册了 on_leave 则设置 will_trap_on_leave
  will_trap_on_leave = function_ctx->replacement_function != NULL ||
      (invoke_listeners && function_ctx->has_on_leave_listener);
  if (will_trap_on_leave)
  {
//2. 如果设置了 will_trap_on_leave，就需要保存原本的返回地址，这样在 on_leave 时能给正确返回
    stack_entry = gum_invocation_stack_push (stack, function_ctx,
        *caller_ret_addr);
    invocation_ctx = &stack_entry->invocation_context;
  }
  else if (invoke_listeners)
  {
//3. 如果没设置 will_trap_on_leave，但有注册 linsters，那么在这里把原本的函数地址保存到栈里
    stack_entry = gum_invocation_stack_push (stack, function_ctx,
        function_ctx->function_address);
    invocation_ctx = &stack_entry->invocation_context;
  }

```

这里不难理解：

*   如果我们替换了函数实现，或者设置了 on_leave ，那么在返回以前到原本的执行流之前就需要先保存当前的返回内容，它会在后续被用于指向正确的返回地址。
*   如果我们只是想钩一些调用点，那么执行流应该从这里返回到原本的函数去恢复执行。

此处只是先在栈中保存数据。

然后接下来会调用注册的 on_enter：

```
if (listener_entry->listener_interface->on_enter != NULL)
{
  listener_entry->listener_interface->on_enter (
      listener_entry->listener_instance, invocation_ctx);
}

```

后续过程中：

```
if (will_trap_on_leave)
{
  *caller_ret_addr = function_ctx->on_leave_trampoline;
}
 
if (function_ctx->replacement_function != NULL)
{
  stack_entry->calling_replacement = TRUE;
  stack_entry->cpu_context = *cpu_context;
  stack_entry->original_system_error = system_error;
  invocation_ctx->cpu_context = &stack_entry->cpu_context;
  invocation_ctx->backend = &interceptor_ctx->replacement_backend;
  invocation_ctx->backend->data = function_ctx->replacement_data;
 
  *next_hop = function_ctx->replacement_function;
}
else
{
  *next_hop = function_ctx->on_invoke_trampoline;
}

```

可以看到此处会将 `on_leave_trampoline` 二级跳板写入到用于储存返回地址的内存中去。也就是说被 hook 函数在执行完毕以后会返回到 `on_leave_trampoline` 。

然后如果需要替换函数实现，那么就要把用于替换的实现代码地址写入当前函数的返回地址去，否则就将跳板注入进去。

因此在该函数结束后会根据这一步选择接下来是执行 `function_ctx->replacement_function` 还是 `function_ctx->on_invoke_trampoline` 。

前者就不难理解了，接下来就是调用我们自己实现的函数，并在返回的时候回到二级跳板。

我们看看后者的实现：

```
(gdb) x/17i function_ctx->on_invoke_trampoline
   0x7fb6c82a30:    stp x29, x30, [sp, #-16]!
   0x7fb6c82a34:    mov x29, sp
   0x7fb6c82a38:    sub sp, sp, #0x10
   0x7fb6c82a3c:    mov x8, #0x0                    // #0
   0x7fb6c82a40:    ldr x16, 0x7fb6c82a48
   0x7fb6c82a44:    br  x16
   0x7fb6c82a48:    cbnz    x12, 0x7fb6cb7ae8//执行原本的函数

```

此处用于调用原本的函数。

这里我们留个疑问，先不管二级跳板 `on_leave_trampoline` 的实现是什么，现在再跟一下 `leave_chunk`：

```
static void
gum_emit_leave_thunk (GumArm64Writer * aw)
{
  gum_emit_prolog (aw);
 
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X1, ARM64_REG_SP,
      GUM_FRAME_OFFSET_CPU_CONTEXT);
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X2, ARM64_REG_SP,
      GUM_FRAME_OFFSET_NEXT_HOP);
 
  gum_arm64_writer_put_call_address_with_arguments (aw,
      GUM_ADDRESS (_gum_function_context_end_invocation), 3,
      GUM_ARG_REGISTER, ARM64_REG_X17,
      GUM_ARG_REGISTER, ARM64_REG_X1,
      GUM_ARG_REGISTER, ARM64_REG_X2);
 
  gum_emit_epilog (aw);
}

```

和前一个 chunk 的结构差不多，我们跟入 `_gum_function_context_end_invocation` :

```
void
_gum_function_context_end_invocation (GumFunctionContext * function_ctx,
                                      GumCpuContext * cpu_context,
                                      gpointer * next_hop)
{
  gint system_error;
  InterceptorThreadContext * interceptor_ctx;
  GumInvocationStackEntry * stack_entry;
  GumInvocationContext * invocation_ctx;
  GPtrArray * listener_entries;
  guint i;
 
#ifdef HAVE_WINDOWS
  system_error = gum_thread_get_system_error ();
#endif
 
  gum_tls_key_set_value (gum_interceptor_guard_key, function_ctx->interceptor);
 
#ifndef HAVE_WINDOWS
  system_error = gum_thread_get_system_error ();
#endif
 
  interceptor_ctx = get_interceptor_thread_context ();
 
  stack_entry = gum_invocation_stack_peek_top (interceptor_ctx->stack);
//1. 此处将函数返回时的地址设置为真正的返回值。该值在 enter_chunk 中被保存
  *next_hop = gum_sign_code_pointer (stack_entry->caller_ret_addr);
//此处省略......
#ifndef GUM_DIET
    if (listener_entry->listener_interface->on_leave != NULL)
    {
//2. 此处调用注册的 on_leave
      listener_entry->listener_interface->on_leave (
          listener_entry->listener_instance, invocation_ctx);
    }
}

```

接下来回到最开始我们跳过的地方，按照代码的顺序，其实在完成上述跳板设置以后才开始准备二级跳板：

```
gboolean
_gum_interceptor_backend_create_trampoline (GumInterceptorBackend * self,
                                            GumFunctionContext * ctx)
{
 ctx->on_enter_trampoline = gum_sign_code_pointer (gum_arm64_writer_cur (aw));
//此处省略
 gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X17, GUM_ADDRESS (ctx));
  gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X16,
      GUM_ADDRESS (gum_sign_code_pointer (self->enter_thunk)));
  gum_arm64_writer_put_br_reg (aw, ARM64_REG_X16);
 
  ctx->on_leave_trampoline = gum_arm64_writer_cur (aw);
 
  gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X17, GUM_ADDRESS (ctx));
  gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X16,
      GUM_ADDRESS (gum_sign_code_pointer (self->leave_thunk)));
  gum_arm64_writer_put_br_reg (aw, ARM64_REG_X16);
 
  gum_arm64_writer_flush (aw);//(6)
  g_assert (gum_arm64_writer_offset (aw) <= ctx->trampoline_slice->size);
 
  ctx->on_invoke_trampoline = gum_sign_code_pointer (gum_arm64_writer_cur (aw));

```

该代码大致会实现如下结构：

```
on_enter_trampoline:
ldr x17 context_addr
ldr x16 enter_thunk
br x16
 
on_leave_trampoline:
ldr x17 context_addr
ldr x16 leave_thunk
br x16
 
//处写入代码，下面三个字存放地址
context_addr:
.dword address
enter_thunk:
.dword address
leave_thunk:
.dword address
 
on_invoke_trampoline:

```

到这一步其实流程就清晰很多了。

*   一级跳板进入到二级跳板 `on_enter_trampoline`
*   二级跳板再跳转到三级跳板 `enter_thunk`
*   三级跳板中再调用四级跳板 `_gum_function_context_begin_invocation`
*   四级跳板将会调用注册的 `on_enter` 函数，并设置真正的返回地址，同时决定接下来要执行谁
    *   如果执行 `on_invoke_trampoline` ，那么将调用原本的函数
    *   否则将会调用我们用于替换的函数
*   接下来执行流将返回到 `on_leave_trampoline`
*   然后从该处跳转入三级跳板 `leave_thunk`
*   再从三级跳板进入四级跳板 `_gum_function_context_end_invocation`
*   在四级跳板中，将恢复真正的返回地址，并调用注册好的 `on_leave` 函数，最后从中返回

而如果用户没有注册 on_leave 函数，那么钩子的步骤将会减少很多。在 `_gum_function_context_begin_invocation` 中将不会修改真正的返回地址，并直接让 `next_hop` 设置为 `on_invoke_trampoline` ，此时程序将直接离开钩子，因为后续不会再进入到 `leave_chunk` 了。

### 额外的问题

在上文中可以发现，几个跳板的实现其实是走了固定的寄存器的。因此如果程序本身本来就要使用这两个寄存器进行工作的话，去 hook 那些函数会导致非预期的结果。这个地方可能需要注意一下吧，毕竟大多数时候使用 frida 感觉 hook 基本上都是透明的，容易忽略到这种级别的问题。

Stalker
-------

其实笔者没怎么用过这个功能，它所实现的 “代码跟踪” 能力其实和调试器差不多，而如果能使用调试器进行调试的话，大部分问题其实都能解决，而就算不能使用调试器，靠 frida 的 hook 也能解决不少问题了，这导致笔者基本上没用过它。

不过没用过不影响看看原理。因为 Stalker 是靠代码插桩的方式实现跟踪的，这和事情的 Interceptor 有点相似。

Stalker 的测试样例就比较多了，但我们只想对源代码的实现有所了解，因此不需要每个都看，选几个比较有意思的就行。这里笔者选了 `TESTENTRY(call)` 作为分析样例：

```
TESTCASE (call)
{
  StalkerTestFunc func;
  GumCallEvent * ev;
 
  func = invoke_flat (fixture, GUM_CALL);
 
  g_assert_cmpuint (fixture->sink->events->len, ==, 2);
  g_assert_cmpint (g_array_index (fixture->sink->events, GumEvent,
      0).type, ==, GUM_CALL);
  ev = &g_array_index (fixture->sink->events, GumEvent, 0).call;
  GUM_ASSERT_CMPADDR (ev->location, ==, fixture->last_invoke_calladdr);
  GUM_ASSERT_CMPADDR (ev->target, ==, gum_strip_code_pointer (func));
}

```

关键的实现在 `invoke_flat` 中，这里我们跟入：`invoke_flat` - `invoke_flat_expecting_return_value`

```
static StalkerTestFunc
invoke_flat_expecting_return_value (TestArm64StalkerFixture * fixture,
                                    GumEventType mask,
                                    guint expected_return_value)
{
  StalkerTestFunc func;
  gint ret;
 
  func = (StalkerTestFunc) test_arm64_stalker_fixture_dup_code (fixture,
      flat_code, sizeof (flat_code));
 
  fixture->sink->mask = mask;
  ret = test_arm64_stalker_fixture_follow_and_invoke (fixture, func, -1);
  g_assert_cmpint (ret, ==, expected_return_value);
 
  return func;
}

```

再跟入 `test_arm64_stalker_fixture_follow_and_invoke`：

```
static gint
test_arm64_stalker_fixture_follow_and_invoke (TestArm64StalkerFixture * fixture,
                                              StalkerTestFunc func,
                                              gint arg)
{
  GumAddressSpec spec;
  guint8 * code;
  GumArm64Writer cw;
  gint ret;
  GCallback invoke_func;
 
  spec.near_address = gum_strip_code_pointer (gum_stalker_follow_me);
  spec.max_distance = G_MAXINT32 / 2;
//1. 创建新代码页用来储存接下来将要生成的代码
  code = gum_alloc_n_pages_near (1, GUM_PAGE_RW, &spec);
 
  gum_arm64_writer_init (&cw, code);
//2. 保存寄存器
  gum_arm64_writer_put_push_reg_reg (&cw, ARM64_REG_X29, ARM64_REG_X30);
  gum_arm64_writer_put_mov_reg_reg (&cw, ARM64_REG_X29, ARM64_REG_SP);
//3. 调用 gum_stalker_follow_me
  gum_arm64_writer_put_call_address_with_arguments (&cw,
      GUM_ADDRESS (gum_stalker_follow_me), 3,
      GUM_ARG_ADDRESS, GUM_ADDRESS (fixture->stalker),
      GUM_ARG_ADDRESS, GUM_ADDRESS (fixture->transformer),
      GUM_ARG_ADDRESS, GUM_ADDRESS (fixture->sink));
 
  /* call function -int func(int x)- and save address before and after call */
  gum_arm64_writer_put_ldr_reg_address (&cw, ARM64_REG_X0, GUM_ADDRESS (arg));
  fixture->last_invoke_calladdr = gum_arm64_writer_cur (&cw);
//4. 调用原本的代码
  gum_arm64_writer_put_call_address_with_arguments (&cw, GUM_ADDRESS (func), 0);
  fixture->last_invoke_retaddr = gum_arm64_writer_cur (&cw);
  gum_arm64_writer_put_ldr_reg_address (&cw, ARM64_REG_X1, GUM_ADDRESS (&ret));
  gum_arm64_writer_put_str_reg_reg_offset (&cw, ARM64_REG_W0, ARM64_REG_X1, 0);
//5. 取消跟踪
  gum_arm64_writer_put_call_address_with_arguments (&cw,
      GUM_ADDRESS (gum_stalker_unfollow_me), 1,
      GUM_ARG_ADDRESS, GUM_ADDRESS (fixture->stalker));
 
  gum_arm64_writer_put_pop_reg_reg (&cw, ARM64_REG_X29, ARM64_REG_X30);
  gum_arm64_writer_put_ret (&cw);
 
  gum_arm64_writer_flush (&cw);
  gum_memory_mark_code (cw.base, gum_arm64_writer_offset (&cw));
  gum_arm64_writer_clear (&cw);
 
  invoke_func =
      GUM_POINTER_TO_FUNCPTR (GCallback, gum_sign_code_pointer (code));
  invoke_func ();
 
  gum_free_pages (code);
 
  return ret;
}

```

逻辑比较清晰，相当于将原本的代码注入到另外一片内存，然后对其进行插桩执行，并在插桩代码中记录覆盖率相关的信息。这里我们先从 `gum_stalker_follow_me` 开始，它是一个由汇编实现的函数：

```
#ifdef __APPLE__
  .globl _gum_stalker_follow_me
_gum_stalker_follow_me:
#else
  .globl gum_stalker_follow_me
  .type gum_stalker_follow_me, %function
gum_stalker_follow_me:
#endif
  stp x29, x30, [sp, -16]!
  mov x29, sp
  mov x3, x30
#ifdef __APPLE__
  bl __gum_stalker_do_follow_me
#else
  bl _gum_stalker_do_follow_me
#endif
  ldp x29, x30, [sp], 16
  br x0


```

此处对于 Apple 架构和其他架构选用了两个不同的函数，不过我在源代码中并没有找到 `__gum_stalker_do_follow_me` 的声明或实现，这里我们将就这用 `_gum_stalker_do_follow_me` 进行理解吧：

```
gpointer
_gum_stalker_do_follow_me (GumStalker * self,
                           GumStalkerTransformer * transformer,
                           GumEventSink * sink,
                           gpointer ret_addr)
{
  GumExecCtx * ctx;
  gpointer code_address;
 
  ctx = gum_stalker_create_exec_ctx (self, gum_process_get_current_thread_id (),
      transformer, sink);
  g_private_set (&gum_stalker_exec_ctx_private, ctx);
 
  ctx->current_block = gum_exec_ctx_obtain_block_for (ctx, ret_addr,
      &code_address);
 
  if (gum_exec_ctx_maybe_unfollow (ctx, ret_addr))
  {
    gum_stalker_destroy_exec_ctx (self, ctx);
    return ret_addr;
  }
 
  gum_event_sink_start (ctx->sink);
  ctx->sink_started = TRUE;
 
  return code_address + GUM_RESTORATION_PROLOG_SIZE;
}

```

关键内容跟入 `gum_event_sink_start` ，里面是用于记录覆盖率信息的具体函数，分别有两套实现，一套是用 `quickjs` ，另外一套是 `v8` 的实现，细节这里笔者就不深究了，大致逻辑如图：

![](https://bbs.kanxue.com/upload/attach/202308/924548_CTXSR93K4FSA6MM.png)

内存监控
----

这部分内容也不是笔者关心的重点，但笔者找了一圈似乎没找到 arm64 下的实现，倒是有 x86 平台下的测试样例。因此本文也就不过多赘述了，大致原理就是设置内存页的读写权限，从而在读写监控页面的时候引发中断来监视内容。

实现的内容分了 Windows 平台和 posix 平台两种，如下代码为 posix 平台：

```
GumMemoryAccessMonitor *
gum_memory_access_monitor_new (const GumMemoryRange * ranges,
                               guint num_ranges,
                               GumPageProtection access_mask,
                               gboolean auto_reset,
                               GumMemoryAccessNotify func,
                               gpointer data,
                               GDestroyNotify data_destroy)
{
  GumMemoryAccessMonitor * monitor;
  guint i;
 
  monitor = g_object_new (GUM_TYPE_MEMORY_ACCESS_MONITOR, NULL);
  monitor->ranges = g_memdup (ranges, num_ranges * sizeof (GumMemoryRange));
  monitor->num_ranges = num_ranges;
  monitor->access_mask = access_mask;
  monitor->auto_reset = auto_reset;
  monitor->pages_total = 0;
 
  for (i = 0; i != num_ranges; i++)
  {
    GumMemoryRange * r = &monitor->ranges[i];
    gsize aligned_start, aligned_end;
    guint num_pages;
 
    aligned_start = r->base_address & ~((gsize) monitor->page_size - 1);
    aligned_end = (r->base_address + r->size + monitor->page_size - 1) &
        ~((gsize) monitor->page_size - 1);
    r->base_address = aligned_start;
    r->size = aligned_end - aligned_start;
 
    num_pages = r->size / monitor->page_size;
    g_atomic_int_add (&monitor->pages_remaining, num_pages);
    monitor->pages_total += num_pages;
  }
 
  monitor->notify_func = func;
  monitor->notify_data = data;
  monitor->notify_data_destroy = data_destroy;
 
  return monitor;
}

```

```
gboolean
gum_memory_access_monitor_enable (GumMemoryAccessMonitor * self,
                                  GError ** error)
{
  if (self->enabled)
    return TRUE;
  // ...
  self->exceptor = gum_exceptor_obtain ();
  gum_exceptor_add (self->exceptor, gum_memory_access_monitor_on_exception,
      self);
  // ...
}

```

### gum-js

这部分主要是做一个扫盲。细节可以参考 evilpan 大佬的文章，里面也大致介绍了 gum-js 的实现。

简单来说就说，V8 支持对 JavaScript 的动态解析，并能够将其抽象到 C 语言层面进行调用。

```
Local attach_name =
      String::NewFromUtf8Literal(GetIsolate(), "Attach");
// 判断对象是否存在，以及类型是否是函数
Local attach_val;
if (!context->Global()->Get(context, attach_name).ToLocal(&attach_val) || !attach_val->IsFunction()) {
    return false;
}
// 如果是，则转换为函数类型
Local attach_func = attach_val.As();
 
// 将调用参数封装为 JS 对象
Local obj =
      templ->NewInstance(GetIsolate()->GetCurrentContext()).ToLocalChecked();
obj->SetInternalField(0, 0xdeadbeef);
// 使用自定义的参数调用该 JS 函数，并获取返回结果
TryCatch try_catch(GetIsolate());
const int argc = 1;
Local argv[argc] = {obj};
Local result;
attach_func->Call(context, context->Global(), argc, argv).ToLocal(&result); 
```

通过这种交互面板，就能给允许用户动态传入脚本进行执行了。之所以选择的是 JavaScript 而不是其他语言，就笔者估计来看，大致上有两个原因（以下内容为笔者的猜测，各位读者可以当看个乐子）：

*   首先编译型的语言肯定是不行了，因为它不支持动态调整，每次都要编译一份再运行肯定不太灵活
*   而在解释型语言里就比较看重运行效率了。笔者曾写过一篇 V8 优化引擎 Turbofan 的原理分析，从其中可以大致理解到，V8 对 JavaScript 的优化效率几乎已经达到理论上限了，这意味着在各种解释型语言里，JS 的效率可能是最高的（这是区分场景的，但在 Frida 里，可能它是最合适的）

> Turbofan 可参考本文：[https://bbs.kanxue.com/thread-273791.htm](https://bbs.kanxue.com/thread-273791.htm)

参考
==

[https://evilpan.com/2022/04/05/frida-internal/#stalker](https://evilpan.com/2022/04/05/frida-internal/#stalker)  
[https://zhuanlan.zhihu.com/p/603717118](https://zhuanlan.zhihu.com/p/603717118)  
[https://o0xmuhe.github.io/2019/11/15/frida-gum%E4%BB%A3%E7%A0%81%E9%98%85%E8%AF%BB/#2-2-2-hook%E4%BB%8E0%E5%88%B01](https://o0xmuhe.github.io/2019/11/15/frida-gum%E4%BB%A3%E7%A0%81%E9%98%85%E8%AF%BB/#2-2-2-hook%E4%BB%8E0%E5%88%B01)

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

最后于 1 天前 被 Tokameine 编辑 ，原因： [#调试逆向](forum-4-1-1.htm) [#其他内容](forum-4-1-10.htm)