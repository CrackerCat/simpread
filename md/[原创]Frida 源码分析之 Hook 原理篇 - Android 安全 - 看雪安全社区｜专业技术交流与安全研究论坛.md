> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-290756.htm)

> [原创]Frida 源码分析之 Hook 原理篇

frida-gum 简介
============

Frida Gum 是一个底层代码插桩库，可在多个平台和架构上提供动态二进制插桩功能。它支持通过函数钩子（fun hooking `GumInterceptor`）、指令级跟踪（include-level tracing `GumStalker`）、内存访问监控（memory access monitoring `GumMemoryAccessMonitor`）和代码生成来运行时操作本地代码。该库支持 Darwin（macOS/iOS）、Linux、Windows、FreeBSD 和 QNX 平台上的 x86、x86_64、ARM、ARM64 和 MIPS 架构。

Hook 函数的 JS Bindings 入口
=======================

以 Native 层的 Hook 为例

```
Interceptor.replace(addr, new NativeCallback(), retType, paramTypes)
Interceptor.attach(addr, {onEnter(args){}, onLeave(retval){}})


```

这些 JS API 在`bindings/gumjs/gumquickinterceptor.c`绑定了 Native 层函数。

`Interceptor.attach`绑定`gumjs_interceptor_attach`函数。

```
GUMJS_DEFINE_FUNCTION (gumjs_interceptor_attach)
{
  JSValue target_val = args->elements[0];
  JSValue cb_val = args->elements[1];
  JSValue data_val = args->elements[2];
  GumQuickInterceptor * self;
  gpointer target, cb_ptr;
  GumQuickInvocationListener * listener = NULL;
  gpointer listener_function_data;
  GumAttachReturn attach_ret;

  self = gumjs_get_parent_module (core);

  //...
  // 解析 onEnter onLeave 并生成监听器
  else
  {
    JSValue on_enter_js, on_leave_js;
    GumQuickCHook on_enter_c, on_leave_c;
    
    if (!_gum_quick_args_parse (args, "pF*{onEnter?,onLeave?}", &target,
        &on_enter_js, &on_enter_c,
        &on_leave_js, &on_leave_c))
      goto propagate_exception;

    if (!JS_IsNull (on_enter_js) || !JS_IsNull (on_leave_js))
    {
      GumQuickJSCallListener * l;

      l = g_object_new (GUM_QUICK_TYPE_JS_CALL_LISTENER, NULL);
      l->on_enter = JS_DupValue (ctx, on_enter_js);
      l->on_leave = JS_DupValue (ctx, on_leave_js);

      listener = GUM_QUICK_INVOCATION_LISTENER (l);
    }
    else if (on_enter_c != NULL || on_leave_c != NULL)
    {
      GumQuickCCallListener * l;

      l = g_object_new (GUM_QUICK_TYPE_C_CALL_LISTENER, NULL);
      l->on_enter = on_enter_c;
      l->on_leave = on_leave_c;

      listener = GUM_QUICK_INVOCATION_LISTENER (l);
    }
    //...
  }

  //可选参数data解析...

  listener->parent = self;
  // 调用	gum_interceptor_attach
  attach_ret = gum_interceptor_attach (self->interceptor, target,
      GUM_INVOCATION_LISTENER (listener), listener_function_data,
      GUM_ATTACH_FLAGS_NONE);

  if (attach_ret != GUM_ATTACH_OK)
    goto unable_to_attach;

  listener->wrapper = JS_NewObjectClass (ctx, self->invocation_listener_class);
  JS_SetOpaque (listener->wrapper, listener);
  JS_DefinePropertyValue (ctx, listener->wrapper,
      GUM_QUICK_CORE_ATOM (core, resource),
      JS_DupValue (ctx, cb_val),
      0);

  g_hash_table_add (self->invocation_listeners, listener);

  return JS_DupValue (ctx, listener->wrapper);
    
  //...
}


```

调用`gum_interceptor_attach`函数，传入的第四个参数（即 flags）为`GUM_ATTACH_FLAGS_NONE = 0`。

而对于`Interceptor.replace`，其绑定的是`gumjs_interceptor_replace`，额外有另一个函数是`gumjs_interceptor_replace_fast`。它们都调用相同的方法`gum_interceptor_replace_with_type`，不同之处在于 replace 模式下传入的第二个参数为`GUM_INTERCEPTOR_TYPE_DEFAULT= 0`。而 replace fast 模式下传入的第二个参数为`GUM_INTERCEPTOR_TYPE_FAST=1`。

```
// gum\guminterceptor.c
GumReplaceReturn
gum_interceptor_replace (GumInterceptor * self,
                         gpointer function_address,
                         gpointer replacement_function,
                         gpointer replacement_data,
                         gpointer * original_function)
{
  return gum_interceptor_replace_with_type (self, GUM_INTERCEPTOR_TYPE_DEFAULT,
      function_address, replacement_function, replacement_data,
      original_function);
}

GumReplaceReturn
gum_interceptor_replace_fast (GumInterceptor * self,
                              gpointer function_address,
                              gpointer replacement_function,
                              gpointer * original_function)
{
  return gum_interceptor_replace_with_type (self, GUM_INTERCEPTOR_TYPE_FAST,
      function_address, replacement_function, NULL,
      original_function);
}


```

尽管如此，`gum_interceptor_replace_with_type`跟`gum_interceptor_attach`的关键代码调用相同，因此以`gum_interceptor_attach`为例进行分析。

gum_interceptor_attach
======================

```
GumAttachReturn
gum_interceptor_attach (GumInterceptor * self,
                        gpointer function_address,
                        GumInvocationListener * listener,
                        gpointer listener_function_data,
                        GumAttachFlags flags)
{
  GumAttachReturn result = GUM_ATTACH_OK;
  GumFunctionContext * function_ctx;
  GumInstrumentationError error;

  gum_interceptor_ignore_current_thread (self);
  GUM_INTERCEPTOR_LOCK (self);
  // 开始hook事务
  gum_interceptor_transaction_begin (&self->current_transaction);
  self->current_transaction.is_dirty = TRUE;
  // 获取被hook函数的地址
  function_address = gum_interceptor_resolve (self, function_address);
  // 生成跳板代码，第四个参数为false
  function_ctx = gum_interceptor_instrument (self, GUM_INTERCEPTOR_TYPE_DEFAULT,
      function_address, (flags & GUM_ATTACH_FLAGS_FORCE) != 0, &error);

  if (function_ctx == NULL)
    goto instrumentation_error;
  // 重复hook
  if (gum_function_context_has_listener (function_ctx, listener))
    goto already_attached;
  // 添加监听器(例如onEnter onLeave事件)
  gum_function_context_add_listener (function_ctx, listener,
      listener_function_data, (flags & GUM_ATTACH_FLAGS_UNIGNORABLE) != 0);

  goto beach;

// labels ...
beach:
  {
    // 结束hook事务，提交并处理hook事务  
    gum_interceptor_transaction_end (&self->current_transaction);
    GUM_INTERCEPTOR_UNLOCK (self);
    gum_interceptor_unignore_current_thread (self);

    return result;
  }
}


```

`gum_interceptor_attach`整个代码逻辑是通过事务的方式进行处理的。它首先调用`gum_interceptor_resolve()`函数以获取真实的函数入口地址。然后调用`gum_interceptor_instrument()`函数生成底层的跳板代码（相较于一级跳板），最后调用`gum_interceptor_transaction_end`提交 Hook 事务，里面会生成一级跳板。

获取真实函数入口地址
----------

### gum_interceptor_resolve

```
static gpointer
gum_interceptor_resolve (GumInterceptor * self,
                         gpointer address)
{
  // 进行指针认证，在支持指令认证的平台上调用ptrauth_strip进行认证，对于不支持的平台直接返回值
  address = gum_strip_code_pointer (address);
  // 判断当前地址是否已经存入哈希表中
  if (!gum_interceptor_has (self, address))
  {	
    // inline hook所修改的字节大小
    const gsize max_redirect_size = 16;
    gpointer target;

    gum_ensure_code_readable (address, max_redirect_size);// 修改所在页为RWX权限

    /* Avoid following grafted branches. */
    //检查代码签名策略，如果需要代码签名，则直接返回地址
    if (gum_process_get_code_signing_policy () == GUM_CODE_SIGNING_REQUIRED)// GUM_CODE_SIGNING_OPTIONAL
      return address;
    // 获取inline hook所涉及到的16字节中的相对跳转地址
    target = _gum_interceptor_backend_resolve_redirect (self->backend,
        address);
    if (target != NULL)// 如果存在相对跳转地址，则进行递归，以获取最
      return gum_interceptor_resolve (self, target);
  }

  return address;
}


```

该函数功能主要是递归以穿透跳板代码，获取到真实的 hook 地址。具体来说，它通过`_gum_interceptor_backend_resolve_redirect`函数获取目标地址处的前 16 字节中的首个相对跳转指令，解析出跳转地址，如果存在相对跳转，则进一步递归调用`gum_interceptor_resolve`函数以获取无相对跳转指令的空间用于 inline hook。

### gum_ensure_code_readable

```
void
gum_ensure_code_readable (gconstpointer address,
                          gsize size)
{
  /*
   * We will make this more generic once it's needed on other OSes.
   */
#ifdef HAVE_ANDROID
  gsize page_size;
  gconstpointer start_page, end_page, cur_page;

  // 低于Android 10 直接返回
  if (gum_android_get_api_level () < 29)
    return;

  page_size = gum_query_page_size ();//获取系统页大小
  start_page = GSIZE_TO_POINTER (
      GPOINTER_TO_SIZE (address) & ~(page_size - 1));
  end_page = GSIZE_TO_POINTER (
      GPOINTER_TO_SIZE (address + size - 1) & ~(page_size - 1)) + page_size;

  G_LOCK (gum_softened_code_pages);

  if (gum_softened_code_pages == NULL)
    gum_softened_code_pages = g_hash_table_new (NULL, NULL);
  // 添加起始页~结束页的地址到哈希表中
  for (cur_page = start_page; cur_page != end_page; cur_page += page_size)
  {
    if (!g_hash_table_contains (gum_softened_code_pages, cur_page))//是否已经加入到哈希表中
    {
      if (gum_try_mprotect ((gpointer) cur_page, page_size, GUM_PAGE_RWX))//修改页权限为RWX
        g_hash_table_add (gum_softened_code_pages, (gpointer) cur_page);//添加
    }
  }

  G_UNLOCK (gum_softened_code_pages);
#endif
}


```

`gum_ensure_code_readable`函数会对目标平台是 Android 的进行额外处理。对于 Android 10 以下的版本则不做处理，对于 Android 10 及以上版本则调用`gum_try_mprotect` 函数修改页权限。

### _gum_interceptor_backend_resolve_redirect

```
// gum\backend-arm64\guminterceptor-arm64.c
gpointer _gum_interceptor_backend_resolve_redirect (GumInterceptorBackend * self, gpointer address){
  return gum_arm64_reader_try_get_relative_jump_target (address);
}

// gum\arch-arm64\gumarm64reader.c
gpointer gum_arm64_reader_try_get_relative_jump_target (gconstpointer address){
  gpointer result = NULL;
  csh capstone;
  cs_insn * insn;
  const uint8_t * code;
  size_t size;
  uint64_t pc;
  const cs_arm64_op * ops;
  // capstone初始化
  cs_arch_register_arm64 ();		// 用于注册 ARM64 (AArch64) 架构支持的初始化函数
  cs_open (CS_ARCH_ARM64, GUM_DEFAULT_CS_ENDIAN, &capstone);	// 创建并初始化反汇编引擎实例
  cs_option (capstone, CS_OPT_DETAIL, CS_OPT_ON);	//开启“详细信息”模式，能够进行语义分析

  insn = cs_malloc (capstone);	//开辟内存

  code = address;	
  size = 16;		
  pc = GPOINTER_TO_SIZE (address);	

// 定义GUM_DISASM_NEXT()函数: 尝试反汇编下一条指令以获取操作数。如果失败（比如遇到了非法指令），直接跳到 beach 标签
#define GUM_DISASM_NEXT() \
    if (!cs_disasm_iter (capstone, &code, &size, &pc, insn)) \
      goto beach; \
    ops = insn->detail->arm64.operands
// 定义GUM_DISASM_NEXT()函数: 如果当前指令类型不是指定类型，跳转到beach
#define GUM_CHECK_ID(i) \
    if (insn->id != G_PASTE (ARM64_INS_, i)) \
      goto beach
// 定义GUM_CHECK_OP_TYPE()函数: 如果当前指令的第n个操作数类型不是指定类型，跳转到beach
#define GUM_CHECK_OP_TYPE(n, t) \
    if (ops[n].type != G_PASTE (ARM64_OP_, t)) \
      goto beach
// 定义GUM_CHECK_OP_REG()函数: 如果当前指令的第n个操作数不是指定寄存器，跳转到beach
#define GUM_CHECK_OP_REG(n, r) \
    if (ops[n].reg != G_PASTE (ARM64_REG_, r)) \
      goto beach
// 定义GUM_CHECK_OP_MEM()函数: 如果当前指令的第n个操作数的基址寄存器、索引寄存器、偏移其中一个不是指定目标，跳转到beach
#define GUM_CHECK_OP_MEM(n, b, i, d) \
    if (ops[n].mem.base != G_PASTE (ARM64_REG_, b)) \
      goto beach; \
    if (ops[n].mem.index != G_PASTE (ARM64_REG_, i)) \
      goto beach; \
    if (ops[n].mem.disp != d) \
      goto beach

  GUM_DISASM_NEXT (); // 反汇编下一条指令获取语义
    
  switch (insn->id)
  {
    case ARM64_INS_B:// B指令跳转，提取目标地址（即立即数）
      result = GSIZE_TO_POINTER (ops[0].imm);
      break;
#ifdef HAVE_DARWIN	
    // DARWIN系统
    // ...
#endif
    default:
      break;
  }

beach://释放内存
  cs_free (insn, 1);

  cs_close (&capstone);

  return result;
}


```

这里寻找前 16 字节中的第一个相对跳转指令，解析出跳转的目标地址并返回。

二三级跳板代码生成
---------

### gum_interceptor_instrument

```
static GumFunctionContext *
gum_interceptor_instrument (GumInterceptor * self,
                            GumInterceptorType type,
                            gpointer function_address, // 被hook函数的地址
                            gboolean force,	// 值为 false
                            GumInstrumentationError * error)
{
  GumFunctionContext * ctx;

  *error = GUM_INSTRUMENTATION_ERROR_NONE;
    // 获取被hook函数的地址所对应的上下文信息
  ctx = (GumFunctionContext *) g_hash_table_lookup (self->function_by_address,
      function_address);
  // 被hook函数的地址已经存在对应的上下文信息，说明是重复hook操作了，直接返回上下文信息
  if (ctx != NULL)
  {
    if (ctx->type != type)
    {
      *error = GUM_INSTRUMENTATION_ERROR_WRONG_TYPE;
      return NULL;
    }
    return ctx;
  }
  // 初始化拦截器后端(interceptor_backend)
  if (self->backend == NULL)
  {
    self->backend = _gum_interceptor_backend_create (&self->mutex, &self->allocator);
  }
  // 创建函数上下文
  ctx = gum_function_context_new (self, function_address, type);

  //...
  else
  {
    // 构造跳板代码
    if (!_gum_interceptor_backend_create_trampoline (self->backend, ctx, force))
      goto wrong_signature;
  }

  g_hash_table_insert (self->function_by_address, function_address, ctx);
  // 将hook任务加入到任务队列中	
  gum_interceptor_transaction_schedule_update (&self->current_transaction, ctx,
      gum_interceptor_activate);

  return ctx;

// labels ...
}


```

### 初始化拦截器后端

#### _gum_interceptor_backend_create

```
GumInterceptorBackend *
_gum_interceptor_backend_create (GRecMutex * mutex,
                                 GumCodeAllocator * allocator)
{
  GumInterceptorBackend * backend;

  backend = g_slice_new0 (GumInterceptorBackend);//分配内存并初始化全0
  backend->mutex = mutex;
  backend->allocator = allocator;

  if (gum_process_get_code_signing_policy () == GUM_CODE_SIGNING_OPTIONAL)// 进入此分支
  {
    // 初始化写入器writer、重定向器relocator
    gum_arm64_writer_init (&backend->writer, NULL);
    gum_arm64_relocator_init (&backend->relocator, NULL, &backend->writer);
    // 创建代码块thunks
    gum_interceptor_backend_create_thunks (backend);
  }

  return backend;
}


```

调用`gum_interceptor_backend_create_thunks`函数预先生成跳板代码`thunks`，具体来说是`enter_thunk`和`leave_thunk`，这些小片段通常负责**保存所有寄存器状态**、**调用 C 层的拦截函数**、**恢复寄存器状态**。

#### gum_interceptor_backend_create_thunks

```
static void
gum_interceptor_backend_create_thunks (GumInterceptorBackend * self)
{
  gsize page_size, code_size;
  GumPageProtection protection;
  GumMemoryRange range;

  page_size = gum_query_page_size ();
  code_size = page_size;
  // gum_memory_can_remap_writable()返回false，选择GUM_PAGE_RW	
  protection = gum_memory_can_remap_writable () ? GUM_PAGE_RX : GUM_PAGE_RW;
  // 分配内存，设置权限
  self->thunks = gum_memory_allocate (NULL, code_size, page_size, protection);

  range.base_address = GUM_ADDRESS (self->thunks);//代码块thunks的起始地址
  range.size = code_size;//页大小(thunks的大小)
  gum_cloak_add_range (&range);

  gum_memory_patch_code (self->thunks, 1024,
      (GumMemoryPatchApplyFunc) gum_emit_thunks, self);
}


```

给`thunks`分配内存后，之后调用`gum_memory_patch_code`函数（第三个参数传入的是`gum_emit_thunks`函数指针），该函数主要是对`thunks`内存的权限进行`RWX`修复，然后回调`gum_emit_thunks`函数。

##### thunks 内存权限修复

这部分可以不用看，大概就是给 thunks 所占的页赋予 RWX 权限，最后回调`gum_emit_thunks`函数。

###### gum_memory_patch_code

```
gboolean
gum_memory_patch_code (gpointer address,
                       gsize size,
                       GumMemoryPatchApplyFunc apply,
                       gpointer apply_data)
{
  gboolean result;
  gsize page_size;
  guint8 * start_page, * end_page;
  gsize page_offset;
  GPtrArray * page_addresses;
  GumPatchCodeContext context;

  address = gum_strip_code_pointer (address);
  // 获取页大小，获取thunks块的起始页地址、结束页地址，以及thunks块相对于起始页的偏移
  page_size = gum_query_page_size ();
  start_page = GSIZE_TO_POINTER (GPOINTER_TO_SIZE (address) & ~(page_size - 1));
  end_page = GSIZE_TO_POINTER (
      (GPOINTER_TO_SIZE (address) + size - 1) & ~(page_size - 1));
  page_offset = ((guint8 *) address) - start_page;//起始页偏移
  
  //创建指针数组，并用于存储thunks所分配的页起始地址
  page_addresses =
      g_ptr_array_sized_new (((end_page - start_page) / page_size) + 1);

  g_ptr_array_add (page_addresses, start_page);

  if (end_page != start_page)
  {
    guint8 * cur;

    for (cur = start_page + page_size;
        cur != end_page + page_size;
        cur += page_size)
    {
      g_ptr_array_add (page_addresses, cur);
    }
  }
  
  context.page_offset = page_offset;
  context.func = apply;	//gum_emit_thunks函数指针
  context.user_data = apply_data;	//_GumInterceptorBackend结构体指针

  result = gum_memory_patch_code_pages (page_addresses, TRUE,
      gum_apply_patch_code, &context);

  g_ptr_array_unref (page_addresses);

  return result;
}


```

创建数组存储`thunks`所占用的页，从代码逻辑上来看，`thunks`所占用的页是连续的。之后调用`gum_memory_patch_code_pages`函数，第三个参数为`gum_apply_patch_code`函数指针。

###### gum_memory_patch_code_pages

```
gboolean
gum_memory_patch_code_pages (GPtrArray * sorted_addresses,// thunks所占的页
                             gboolean coalesce,		//值为true
                             GumMemoryPatchPagesApplyFunc apply, // gum_apply_patch_code
                             gpointer apply_data)
{
  gboolean result = TRUE;
  gsize page_size;
  guint i;
  guint8 * apply_start, * apply_target_start;
  guint apply_num_pages;
  gboolean rwx_supported;

  rwx_supported = gum_query_is_rwx_supported ();// ARM64返回True
  page_size = gum_query_page_size ();

  // ...
  
  else if (rwx_supported || !gum_code_segment_is_supported ())
  {
    GumPageProtection protection;
    GumSuspendOperation suspend_op = { 0, };
    // 值为GUM_PAGE_RWX
    protection = rwx_supported ? GUM_PAGE_RWX : GUM_PAGE_RW;

    ...
    // 修改页权限
    for (i = 0; i != sorted_addresses->len; i++)
    {
      gpointer target_page = g_ptr_array_index (sorted_addresses, i);//获取第i个页地址指针

      if (!gum_try_mprotect (target_page, page_size, protection))//修改页权限为RWX
      {
        result = FALSE;
        goto resume_threads;//修改失败，恢复线程
      }
    }
    // 合并连续页（本身就是连续的）
    apply_start = NULL;
    apply_num_pages = 0;
    for (i = 0; i != sorted_addresses->len; i++)
    {
      gpointer target_page = g_ptr_array_index (sorted_addresses, i);//获取第i个页地址指针

      if (coalesce)//TRUE
      {
        if (apply_start != 0)
        {
          if (target_page == apply_start + (page_size * apply_num_pages))//页连续
          {
            apply_num_pages++;
          }
          else
          {//非连续页面调用apply函数，实际为gum_apply_patch_code函数
            apply (apply_start, apply_target_start, apply_num_pages,
                apply_data);
            apply_start = 0;
          }
        }

        if (apply_start == 0)
        {
          apply_start = target_page;
          apply_target_start = target_page;
          apply_num_pages = 1;
        }
      }
      ...  
    }

    if (apply_num_pages != 0)
      apply (apply_start, apply_target_start, apply_num_pages, apply_data);

    ...
    //清理缓存
    for (i = 0; i != sorted_addresses->len; i++)
    {
      gpointer target_page = g_ptr_array_index (sorted_addresses, i);

      gum_clear_cache (target_page, page_size);
    }

resume_threads:
  ...

  return result;
}


```

修改 thunks 块所占用的页的权限为 RWX。由于`thunks`本身分配的页就是连续的，因此只回调一次`gum_apply_patch_code`函数。

###### gum_apply_patch_code

```
static void
gum_apply_patch_code (gpointer mem,		//页地址
                      gpointer target_page,//页地址，值同上
                      guint n_pages,    // 连续页个数
                      gpointer user_data)// gum_memory_patch_code函数中创建的GumPatchCodeContext结构体
{
  GumPatchCodeContext * context = user_data;

  context->func ((guint8 *) mem + context->page_offset, context->user_data);//第一个参数为thunks起始地址
}


```

往上追溯`func`来源可以知道是`gum_emit_thunks`函数，第一个参数`mem + page_offset`就是`thunks`的起始地址，第二个参数`user_data`就是`_gum_interceptor_backend_create`函数中创建的`GumInterceptorBackend`结构体。

### thunks 跳板生成（三级跳板）

#### gum_emit_thunks

```
static void
gum_emit_thunks (gpointer mem, // thunks起始地址
                 GumInterceptorBackend * self)
{
  GumArm64Writer * aw = &self->writer;
  // 构造enter_thunk
  self->enter_thunk = self->thunks;
  gum_arm64_writer_reset (aw, mem);//重置GumArm64Writer的label_defs
  aw->pc = GUM_ADDRESS (self->enter_thunk);//设置writer的pc，表示从哪里开始写入指令
  gum_emit_enter_thunk (aw);
  gum_arm64_writer_flush (aw);//缓存写入
  // 构造leave_thunk
  self->leave_thunk =
      (guint8 *) self->enter_thunk + gum_arm64_writer_offset (aw);//紧挨着enter_thunk
  gum_emit_leave_thunk (aw);
  gum_arm64_writer_flush (aw);
}


```

主要调用`gum_emit_enter_thunk`函数和`gum_emit_leave_thunk`函数分别构建`enter_thunk`跳板以及`leave_thunk`跳板。

#### enter_thunk 跳板生成

##### gum_emit_enter_thunk

```
static void
gum_emit_enter_thunk (GumArm64Writer * aw)
{
  //保存CPU上下文
  gum_emit_prolog (aw);
  // add x1, sp, #0
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X1, ARM64_REG_SP,
      GUM_FRAME_OFFSET_CPU_CONTEXT);
  // add x2, sp, #lr_offset_in_ctx
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X2, ARM64_REG_SP,
      GUM_FRAME_OFFSET_CPU_CONTEXT + G_STRUCT_OFFSET (GumCpuContext, lr));//G_STRUCT_OFFSET返回偏移量
  // add x3, sp, #ctx_size
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X3, ARM64_REG_SP,
      GUM_FRAME_OFFSET_NEXT_HOP);
  // call _gum_function_context_begin_invocation(x17, x1, x2, x3)
  gum_arm64_writer_put_call_address_with_arguments (aw,
      GUM_ADDRESS (_gum_function_context_begin_invocation), 4,
      GUM_ARG_REGISTER, ARM64_REG_X17,
      GUM_ARG_REGISTER, ARM64_REG_X1,
      GUM_ARG_REGISTER, ARM64_REG_X2,
      GUM_ARG_REGISTER, ARM64_REG_X3);
  // 恢复CPU上下文信息	
  gum_emit_epilog (aw);
}


```

生成的跳板代码如下：

![](https://bbs.kanxue.com/upload/attach/202604/985561_683N979FCESB37S.webp)

###### gum_emit_prolog

```
static void gum_emit_prolog (GumArm64Writer * aw){
  gint i;
  /*
   * Set up our stack frame:
   *
   * [in: frame pointer chain entry, out: next_hop]
   * [in/out: cpu_context]
   */
  /* Reserve space for next_hop */
  gum_arm64_writer_put_sub_reg_reg_imm (aw, ARM64_REG_SP, ARM64_REG_SP, 16);//生成机器码并写入

  /* Store vector registers */
  for (i = 30; i != -2; i -= 2)
    gum_arm64_writer_put_push_reg_reg (aw, ARM64_REG_Q0 + i, ARM64_REG_Q1 + i);

  /* Store X1-X28, FP, and LR */
  gum_arm64_writer_put_push_reg_reg (aw, ARM64_REG_FP, ARM64_REG_LR);
  for (i = 27; i != -1; i -= 2)
    gum_arm64_writer_put_push_reg_reg (aw, ARM64_REG_X0 + i, ARM64_REG_X1 + i);

  /* Store NZCV and X0 */
  gum_arm64_writer_put_mov_reg_nzcv (aw, ARM64_REG_X1);
  gum_arm64_writer_put_push_reg_reg (aw, ARM64_REG_X1, ARM64_REG_X0);

  /* PC placeholder and SP */
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X0,
      ARM64_REG_SP, sizeof (GumCpuContext) -
      G_STRUCT_OFFSET (GumCpuContext, nzcv) + 16);
  gum_arm64_writer_put_push_reg_reg (aw, ARM64_REG_XZR, ARM64_REG_X0);

  /* Frame pointer chain entry */
  gum_arm64_writer_put_str_reg_reg_offset (aw, ARM64_REG_LR, ARM64_REG_SP,
      sizeof (GumCpuContext) + 8);
  gum_arm64_writer_put_str_reg_reg_offset (aw, ARM64_REG_FP, ARM64_REG_SP,
      sizeof (GumCpuContext) + 0);
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_FP, ARM64_REG_SP,
      sizeof (GumCpuContext));
}


```

该部分主要生成具有如下功能的代码：在函数进入钩时，保存所有 CPU 寄存器和设置帧指针链，以便后续恢复。构建的指令以及栈布局如下：

![](https://bbs.kanxue.com/upload/attach/202604/985561_5JY23KUW7MVDE32.webp)

为什么会存储两次 LR、FP 寄存器？这是因为第一次是用于构建`GumCpuContext`，以便在 hook 执行后能够恢复原始 CPU 上下文。第二次是用于构造帧指针链，这是 ARM64 的标准调试 / 栈回溯机制。每个栈帧存储前一个栈帧的 FP 和 LR，允许调试器、性能分析器或异常处理器遍历调用栈。

回到`gum_emit_enter_thunk`中，接下来的指令如下：

```
add x1, sp, #0;	//指向GumCpuContext起始地址
add x2, sp, #offset_lr_in_ctx;// 指向lr（返回地址）
add x3, sp, #ctx_size; // 指向next_hop起始地址
call _gum_function_context_begin_invocation(x17, x1, x2, x3);


```

###### _gum_function_context_begin_invocation

```
gboolean
_gum_function_context_begin_invocation (GumFunctionContext * function_ctx,//被hook函数的上下文信息
                                        GumCpuContext * cpu_context, //指向栈上GumCpuContext起始地址
                                        gpointer * caller_ret_addr,// 指向栈上GumCpuContext中lr（返回地址）
                                        gpointer * next_hop)// 指向栈上next_hop起始地址
{
  GumInterceptor * interceptor;
  InterceptorThreadContext * interceptor_ctx;
  GumInvocationStack * stack;
  GumInvocationStackEntry * stack_entry;
  GumInvocationContext * invocation_ctx = NULL;
  gint system_error;
  gboolean invoke_listeners = TRUE;
  gboolean only_invoke_unignorable_listeners = FALSE;
  gboolean will_trap_on_leave = FALSE;

  g_atomic_int_inc (&function_ctx->trampoline_usage_counter);

  interceptor = function_ctx->interceptor;
  // 如果发现当前线程已经在处理这个拦截器，则直接设置 next_hop 为原函数（on_invoke_trampoline），跳过 Hook 逻辑。
  if (gum_tls_key_get_value (gum_interceptor_guard_key) == interceptor)
  {
    *next_hop = function_ctx->on_invoke_trampoline;
    goto bypass;
  }
  gum_tls_key_set_value (gum_interceptor_guard_key, interceptor);

  interceptor_ctx = get_interceptor_thread_context ();
  stack = interceptor_ctx->stack;
  // Frida 为每个线程维护了一个隐藏的栈，用于保存原始的返回地址（caller_ret_addr），
  // 为 onLeave 存储状态，以及记录嵌套调用的层级。
  stack_entry = gum_invocation_stack_peek_top (stack);
  if (stack_entry != NULL && // 存在调用
      stack_entry->calling_replacement && // 且为replace hook
      gum_strip_code_pointer (GUM_FUNCPTR_TO_POINTER (
          stack_entry->invocation_context.function)) ==
          function_ctx->function_address) // 且 调用的函数地址 == 原函数地址
  { // 设置next_hop前8字节为on_invoke_trampoline
    gum_tls_key_set_value (gum_interceptor_guard_key, NULL);
    *next_hop = function_ctx->on_invoke_trampoline;
    goto bypass;
  }
  // ...
  // replace hook 或者 注册了on Leave
  will_trap_on_leave = function_ctx->replacement_function != NULL ||
      (invoke_listeners && function_ctx->has_on_leave_listener);
  if (will_trap_on_leave)
  { // 状态入栈（被hook函数的上下文信息、原始返回地址）
    stack_entry = gum_invocation_stack_push (stack, function_ctx,
        *caller_ret_addr, only_invoke_unignorable_listeners);
    invocation_ctx = &stack_entry->invocation_context;
  }
  else if (invoke_listeners)
  {
    stack_entry = gum_invocation_stack_push (stack, function_ctx,
        function_ctx->function_address, only_invoke_unignorable_listeners);
    invocation_ctx = &stack_entry->invocation_context;
  }

  if (invocation_ctx != NULL)
    invocation_ctx->system_error = system_error;
  
  // cpu_context->pc = function_ctx->function_address
  //修改栈中cpu_context的pc，指向原函数（有啥用？后续恢复CPU上下文然后跳转就会覆盖pc的值啊！？）
  gum_function_context_fixup_cpu_context (function_ctx, cpu_context);

  if (invoke_listeners)
  {
    // ...
    for (i = 0; i != listener_entries->len; i++)
    {
     //...

      if (listener_entry->listener_interface->on_enter != NULL)
      {
          // 执行OnEnter
        listener_entry->listener_interface->on_enter (
            listener_entry->listener_instance, invocation_ctx);
      }
    }

    system_error = invocation_ctx->system_error;
  }

  //...

  if (will_trap_on_leave)
  {//注册了 onLeave hook，将栈上GumCpuContext中lr值修改成on_leave_trampoline所在地址
   // 这样执行完原函数并ret时，会跳转到on_leave_trampoline
    *caller_ret_addr = function_ctx->on_leave_trampoline;
  }

  if (function_ctx->replacement_function != NULL)
  {//注册了 replace hook, 进入替换函数
    stack_entry->calling_replacement = TRUE;
    stack_entry->cpu_context = *cpu_context;
    stack_entry->original_system_error = system_error;
    invocation_ctx->cpu_context = &stack_entry->cpu_context;
    invocation_ctx->backend = &interceptor_ctx->replacement_backend;
    invocation_ctx->backend->data = function_ctx->replacement_data;
    // 修改栈上next_hop前8字节为replacement_function地址
    // 这样返回到enter_thunk后，br x16就直接跳转到replacement_function
    *next_hop = function_ctx->replacement_function;
  }
  else
  {// 修改栈上next_hop前8字节为on_invoke_trampoline地址
   // 这样返回到enter_thunk后，br x16就直接跳转到on_invoke_trampoline
    *next_hop = function_ctx->on_invoke_trampoline;
  }
  //...
  return will_trap_on_leave;
}


```

如果存在 onLeave hook，则将`function_ctx`和`caller_ret_addr`压入`invocation_stack`，之后会在`_gum_function_context_end_invocation`函数中取出并使用。

然后通过执行监听器的`on_enter`方法进入到自定义的 onEnter 逻辑。

如果后续存在 onLeave hook，就修改栈上保存的 GumCpuContext 中的 lr 寄存器的值修改成`on_leave_trampoline`入口地址，这样后续原函数执行完后，通过 ret 指令就会回到`on_leave_trampoline`入口处。

之后根据不同 hook 类型对栈上的`next_hop`进行不同修改：如果是 replace hook，则修改`next_hop`为`replacement_function`地址，否则修改成`on_invoke_trampoline`地址，这样一来，后续执行完`enter_thunk`最后一条指令`br x16`后，就会跳转到目标地址处。

###### gum_emit_epilog

```
static void
gum_emit_epilog (GumArm64Writer * aw)
{
  guint i;

  /* Skip PC and SP */
  // add sp, sp, #0x10    跳过栈上的PC和SP
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_SP, ARM64_REG_SP, 16);
  // 恢复CPU上下文
  /* Restore NZCV and X0 */
  gum_arm64_writer_put_pop_reg_reg (aw, ARM64_REG_X1, ARM64_REG_X0);
  gum_arm64_writer_put_mov_nzcv_reg (aw, ARM64_REG_X1);

  /* Restore X1-X28, FP, and LR */
  for (i = 1; i != 29; i += 2)
    gum_arm64_writer_put_pop_reg_reg (aw, ARM64_REG_X0 + i, ARM64_REG_X1 + i);
  gum_arm64_writer_put_pop_reg_reg (aw, ARM64_REG_FP, ARM64_REG_LR);

  /* Restore vector registers */
  for (i = 0; i != 32; i += 2)
    gum_arm64_writer_put_pop_reg_reg (aw, ARM64_REG_Q0 + i, ARM64_REG_Q1 + i);
  
  // return to next_hop
  // ldp x16, x17, [sp, #0x10]!
  gum_arm64_writer_put_pop_reg_reg (aw, ARM64_REG_X16, ARM64_REG_X17);
  
#ifndef HAVE_PTRAUTH
  gum_arm64_writer_put_ret_reg (aw, ARM64_REG_X16);
#else
  // br x16
  gum_arm64_writer_put_br_reg (aw, ARM64_REG_X16);
#endif
}


```

恢复 CPU 上下文，最终 x16 寄存器指向`next_hop`前 8 字节（此值在`_gum_function_context_begin_invocation`根据不同情况赋予了不同的值），x17 寄存器存储了原来的 LR 寄存器的值。

当执行最后一条指令时，对于进行了 replace hook 的情况，则是跳转到`replacement_function`处，对于非 replace hook（即 onEnter onLeave hook），则跳转到`on_invoke_trampoline`处。

#### leave_thunk 跳板生成

##### gum_emit_leave_thunk

```
static void
gum_emit_leave_thunk (GumArm64Writer * aw)
{
  // 保存CPU上下文信息
  gum_emit_prolog (aw);
  // add x1, sp, #0
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X1, ARM64_REG_SP,
      GUM_FRAME_OFFSET_CPU_CONTEXT);
  // add x2, sp, #ctx_size
  gum_arm64_writer_put_add_reg_reg_imm (aw, ARM64_REG_X2, ARM64_REG_SP,
      GUM_FRAME_OFFSET_NEXT_HOP);
  // call _gum_function_context_end_invocation(x17, x1, x2)
  gum_arm64_writer_put_call_address_with_arguments (aw,
      GUM_ADDRESS (_gum_function_context_end_invocation), 3,
      GUM_ARG_REGISTER, ARM64_REG_X17,
      GUM_ARG_REGISTER, ARM64_REG_X1,
      GUM_ARG_REGISTER, ARM64_REG_X2);
  // 恢复CPU上下文信息
  gum_emit_epilog (aw);
}


```

这部分对应的伪汇编指令如下：

![](https://bbs.kanxue.com/upload/attach/202604/985561_KX7KGRVYNGFSRWK.webp)

`gum_emit_prolog`函数和`gum_emit_epilog`函数前面已经分析过了，这里直接看`_gum_function_context_end_invocation`函数。

###### _gum_function_context_end_invocation

```
void
_gum_function_context_end_invocation (GumFunctionContext * function_ctx,
                                      GumCpuContext * cpu_context, //指向栈上GumCpuContext起始地址
                                      gpointer * next_hop)// 指向栈上next_hop起始地址
{
  gint system_error;
  InterceptorThreadContext * interceptor_ctx;
  GumInvocationStackEntry * stack_entry;
  GumInvocationContext * invocation_ctx;
  GPtrArray * listener_entries;
  gboolean only_invoke_unignorable_listeners;
  guint i;

  gum_tls_key_set_value (gum_interceptor_guard_key, function_ctx->interceptor);

  interceptor_ctx = get_interceptor_thread_context ();

  stack_entry = gum_invocation_stack_peek_top (interceptor_ctx->stack);

  // next_hop前8字节设置为之前在invocation_stack保存的返回地址
  *next_hop = gum_sign_code_pointer (stack_entry->caller_ret_addr);

  //...
  
  // 修改栈上CPU的PC为原函数地址
  gum_function_context_fixup_cpu_context (function_ctx, cpu_context);

  listener_entries = (GPtrArray *) g_atomic_pointer_get (&function_ctx->listener_entries);
  only_invoke_unignorable_listeners = stack_entry->only_invoke_unignorable_listeners;
  for (i = 0; i != listener_entries->len; i++)
  {
   //...

    if (listener_entry->listener_interface->on_leave != NULL)
    { 
      // onLeave
      listener_entry->listener_interface->on_leave (
          listener_entry->listener_instance, invocation_ctx);
    }
  }
  //...
}


```

修改`next_hop`设置为之前在`invocation_stack`保存的返回地址，这样执行完`leav_thunk`最后一条指令`br x16`后，就会跳转到目标地址处（被 hook 函数的 caller ret address）。之后`on_leave`执行自定义的`onLeave`代码。

至此，`thunks`的生成代码已经分析完毕，Hook 生成的代码逻辑最终回到`gum_interceptor_instrument`中，接下来执行的是`_gum_interceptor_backend_create_trampoline`。

### 二级跳板生成

#### _gum_interceptor_backend_create_trampoline

由于该函数太大了，这里就根据功能划分成多个部分进行分析。

```
gboolean
_gum_interceptor_backend_create_trampoline (GumInterceptorBackend * self,
                                            GumFunctionContext * ctx,
                                            gboolean force)
{
  GumArm64Writer * aw = &self->writer;
  GumArm64Relocator * ar = &self->relocator;
  gpointer function_address = ctx->function_address;
  GumArm64FunctionContextData * data = GUM_FCDATA (ctx);
  gboolean need_deflector;
  gpointer deflector_target;
  GString * signature;
  gboolean is_eligible_for_lr_rewriting;
  guint reloc_bytes;

// 决定 redirect code 的大小（4、8、16 字节），以及是否需要中继器deflector
if (!gum_interceptor_backend_prepare_trampoline (self, ctx, force, &need_deflector))
    return FALSE;

gum_arm64_writer_reset (aw, ctx->trampoline_slice->data);

aw->pc = GUM_ADDRESS (ctx->trampoline_slice->pc);


```

首先调用`gum_interceptor_backend_prepare_trampoline`来获取最大可用于重定向的空间（可用于 inline hook 空间），给`ctx->trampoline_slice`分配内存空间用于编写跳板，同时判断是否需要中继器（deflector），以及获取可用的临时寄存器`x16`或`x17`。然后初始化 writer（其 base、code 初始化为相同的值）。

```
if (ctx->type == GUM_INTERCEPTOR_TYPE_FAST)// replace fast hook
{ // 设置中继器的跳转目标为 replacement function
    deflector_target = ctx->replacement_function;
}
else
{
    // on_enter_trampoline指向的trampoline_slice的起始地址，因为后者offset为0
    ctx->on_enter_trampoline = gum_sign_code_pointer (
        (guint8 *) ctx->trampoline_slice->pc + gum_arm64_writer_offset (aw));
    // 设置中继器的跳转目标为 on_enter_trampoline
    deflector_target = ctx->on_enter_trampoline;
}


```

之后根据 hook 类型给中继器跳转目标设置不同的地址：replace fast hook 需要设置中继器目标为`replacement_function`地址。非 replace fast hook（replace 与 attach）设置中继器目标为`on_enter_trampoline`的地址，并初始化`on_enter_trampoline`的起始地址。

```
// 如果需要中继器（deflector），则在跳板代码中先保存 x0 和 lr 寄存器的值
if (need_deflector)
{
    GumAddressSpec caller;
    gpointer return_address;
    gboolean dedicated;
    // caller.near_address指向重定向指令中的最后一条指令
    caller.near_address =
        (guint8 *) function_address + data->redirect_code_size - 4;
    // caller.max_distance的值为b指令跳转的最大值(+128MB)
    caller.max_distance = GUM_ARM64_B_MAX_DISTANCE;
    // return_address指向重定向指令之后的原代码地址
    return_address = (guint8 *) function_address + data->redirect_code_size;
    // 是否是4字节重定向(inline hook)
    dedicated = data->redirect_code_size == 4;

    // 为中继器分配内存空间
    ctx->trampoline_deflector = gum_code_allocator_alloc_deflector (
        self->allocator, &caller, return_address, deflector_target, dedicated);
    if (ctx->trampoline_deflector == NULL)
    {
        gum_code_slice_unref (ctx->trampoline_slice);
        ctx->trampoline_slice = NULL;
        return FALSE;
    }
    // ldp x0，lr, [sp, #-16]!
    gum_arm64_writer_put_pop_reg_reg (aw, ARM64_REG_X0, ARM64_REG_LR);
}


```

对于需要中继器的，则计算相关地址，并调用`gum_code_allocator_alloc_deflector`为中继器分配内存空间。然后向`trampoline_slice`写入`ldp x0，lr, [sp, #-16]!`指令。对于中继器这一块的分析，我将其放置文章末尾了，此处分析没有中继器的情况，这两种情况生成的`trampoline_slice`差不多，唯一的区别就是多了个刚刚生成的`ldp x0，lr, [sp, #-16]!`。

```
// 不是 replace fast hook
if (ctx->type != GUM_INTERCEPTOR_TYPE_FAST){ 
    // PC 相对偏移加载（PC-relative LDR） 指令
    gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X17, GUM_ADDRESS (ctx));
    // enter_thunk -> x16
    gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X16,
        GUM_ADDRESS (gum_sign_code_pointer (self->enter_thunk)));
    // BR x16
    gum_arm64_writer_put_br_reg (aw, ARM64_REG_X16);

    ctx->on_leave_trampoline =
        (guint8 *) ctx->trampoline_slice->pc + gum_arm64_writer_offset (aw);
    
    // ctx -> x17
    gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X17, GUM_ADDRESS (ctx));
    // leave_thunk -> x16
    gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X16,
        GUM_ADDRESS (gum_sign_code_pointer (self->leave_thunk)));
    // BR x16
    gum_arm64_writer_put_br_reg (aw, ARM64_REG_X16);

    gum_arm64_writer_flush (aw);
    g_assert (gum_arm64_writer_offset (aw) <= ctx->trampoline_slice->size);
}


```

对于非 replace fast hook，则编写跳板`on_enter_trampoline`和`on_leave_trampoline`分别用于跳转到`enter_thunk`和`leave_thunk`。

```
ctx->on_invoke_trampoline = gum_sign_code_pointer (
    (guint8 *) ctx->trampoline_slice->pc + gum_arm64_writer_offset (aw));


```

此处开始，不区分 hook 模式，都设置`on_invoke_trampoline`的写入地址，它的地址紧挨着`on_leave_trampoline`，用于跳转到`invoke_trampoline`。

```
// 设置 relocator 的起始地址为目标函数地址
gum_arm64_relocator_reset (ar, function_address, aw);

// 构造指令签名
signature = g_string_sized_new (16);

do{
    const cs_insn * insn;

    reloc_bytes = gum_arm64_relocator_read_one (ar, &insn);
    if (reloc_bytes == 0){
        reloc_bytes = data->redirect_code_size;
        break;
    }
    // 指令助记符作为签名，中间以";"分割
    if (signature->len != 0)
        g_string_append_c (signature, ';');
    g_string_append (signature, insn->mnemonic);
}while (reloc_bytes < data->redirect_code_size);
// 是否存在指定签名
is_eligible_for_lr_rewriting = strcmp (signature->str, "mov;b") == 0 ||
    g_str_has_prefix (signature->str, "stp;mov;mov;bl");

g_string_free (signature, TRUE);

// 如果存在目标指令序列，则进行lr修复并写入on_invoke_trampoline中，否则直接将原指令写入on_invoke_trampoline中
if (is_eligible_for_lr_rewriting){
    const cs_insn * insn;

    while ((insn = gum_arm64_relocator_peek_next_write_insn (ar)) != NULL){
        const cs_arm64_op * source_op = &insn->detail->arm64.operands[1];
        // 匹配 mov xN, LR
        if (insn->id == ARM64_INS_MOV && source_op->type == ARM64_OP_REG &&
            source_op->reg == ARM64_REG_LR){
            arm64_reg dst_reg = insn->detail->arm64.operands[0].reg;
            const guint reg_size = sizeof (gpointer);
            const guint reg_pair_size = 2 * reg_size;
            guint dst_reg_index, dst_reg_slot_index, dst_reg_offset_in_frame;
            // 保存所有通用寄存器
            gum_arm64_writer_put_push_all_x_registers (aw);
            // call _gum_interceptor_translate_top_return_address(LR)
            gum_arm64_writer_put_call_address_with_arguments (aw,
                                                              GUM_ADDRESS (_gum_interceptor_translate_top_return_address), 1,
                                                              GUM_ARG_REGISTER, ARM64_REG_LR);
            // 获取指令中目标寄存器索引值  
            if (dst_reg >= ARM64_REG_X0 && dst_reg <= ARM64_REG_X28){
                dst_reg_index = dst_reg - ARM64_REG_X0;
            }
            else{
                g_assert (dst_reg >= ARM64_REG_X29 && dst_reg <= ARM64_REG_X30);

                dst_reg_index = dst_reg - ARM64_REG_X29;
            }
            // 获取目标寄存器在栈帧中的槽位索引值
            dst_reg_slot_index = (dst_reg_index * reg_size) / reg_pair_size;

            dst_reg_offset_in_frame = (15 - dst_reg_slot_index) * reg_pair_size;
            if (dst_reg_index % 2 != 0)
                dst_reg_offset_in_frame += reg_size;
            // STR X0, [SP, #dst_reg_offset_in_frame]
            gum_arm64_writer_put_str_reg_reg_offset (aw, ARM64_REG_X0, ARM64_REG_SP,
                                                     dst_reg_offset_in_frame);
            // 恢复所有通用寄存器    
            gum_arm64_writer_put_pop_all_x_registers (aw);
            // 跳过原始指令
            gum_arm64_relocator_skip_one (ar);
        }
        else{
            gum_arm64_relocator_write_one (ar);//重定位区间的原指令写入trampoline中
        }
    }
}
else{
    gum_arm64_relocator_write_all (ar);//重定位区间的原指令写入trampoline中
}


```

这部分就是对重定向区域内的原指令进行 LR 修复并写入`on_invoke_trampoline`中。

```
if (!ar->eoi)
{
    GumAddress resume_at;

    resume_at = gum_sign_code_address (
        GUM_ADDRESS (function_address) + reloc_bytes);// 原函数地址（除去inlink占用的大小）

    gum_arm64_writer_put_ldr_reg_address (aw, data->scratch_reg, resume_at);

    gum_arm64_writer_put_br_reg (aw, data->scratch_reg);
}


```

这里继续构造`on_invoke_trampoline`，用于跳转到后续的原函数代码。以上代码最终生成的跳板指令如下图所示。

![](https://bbs.kanxue.com/upload/attach/202604/985561_JJRZ436QJ5SXAF6.webp)

```
ctx->overwritten_prologue_len = reloc_bytes;
gum_memcpy (ctx->overwritten_prologue, function_address, reloc_bytes);


```

备份原函数中 inline hook 所占用的指令。

##### gum_interceptor_backend_prepare_trampoline

```
// gum\backend-arm64\guminterceptor-arm64.c
static gboolean
gum_interceptor_backend_prepare_trampoline (GumInterceptorBackend * self,
                                            GumFunctionContext * ctx,
                                            gboolean force,
                                            gboolean * need_deflector)
{
  GumArm64FunctionContextData * data = GUM_FCDATA (ctx);
  gpointer function_address = ctx->function_address;
  guint redirect_limit;

  *need_deflector = FALSE;
  // 检查是否可以进行16字节的全量重定向，同时获取可安全重定向的字节数->redirect_limit,
  // 以及获取可用的临时寄存器（x16\x17）-> scratch_reg	
  if (gum_arm64_relocator_can_relocate (function_address,
        GUM_INTERCEPTOR_FULL_REDIRECT_SIZE, GUM_SCENARIO_ONLINE,
        &redirect_limit, &data->scratch_reg))
  {
    data->redirect_code_size = GUM_INTERCEPTOR_FULL_REDIRECT_SIZE;
    // 获取用于跳板指令的内存空间
    ctx->trampoline_slice = gum_code_allocator_alloc_slice (self->allocator);
  }
  else if (force)//强制16字节的全量重定向，并设置临时寄存器为x16
  {
    data->redirect_code_size = GUM_INTERCEPTOR_FULL_REDIRECT_SIZE;

    ctx->trampoline_slice = gum_code_allocator_alloc_slice (self->allocator);

    if (data->scratch_reg == ARM64_REG_INVALID)
      data->scratch_reg = ARM64_REG_X16;

    return TRUE;
  }
  else if (ctx->type == GUM_INTERCEPTOR_TYPE_FAST)// repleace fast hook
  {
    // 此模式下要求修改目标函数使其直接跳转到替换函数，而4/8字节的空间无法始终能够到达替换函数，
    // 这是因为ADRP B指令跳转地址受限
    return FALSE;
  }
  else
  {
    GumAddressSpec spec;
    gsize alignment;

    if (redirect_limit >= 8)// 8字节重定向
    {
      data->redirect_code_size = 8;
      // 获取目标函数的起始页地址
      spec.near_address = GSIZE_TO_POINTER (
          GPOINTER_TO_SIZE (function_address) &
          ~((gsize) (GUM_ARM64_LOGICAL_PAGE_SIZE - 1)));
      // ADRP最大寻址范围
      spec.max_distance = GUM_ARM64_ADRP_MAX_DISTANCE;
      // 页大小
      alignment = GUM_ARM64_LOGICAL_PAGE_SIZE;
    }
    else if (redirect_limit >= 4)// 4字节重定向
    {
      data->redirect_code_size = 4;
      
      spec.near_address = function_address;
      // B最大寻址范围
      spec.max_distance = GUM_ARM64_B_MAX_DISTANCE;
      alignment = 0;
    }
    else
    {
      return FALSE;
    }
    // 尝试给跳板代码分配内存空间
    ctx->trampoline_slice = gum_code_allocator_try_alloc_slice_near (
        self->allocator, &spec, alignment);
    if (ctx->trampoline_slice == NULL)
    {
      ctx->trampoline_slice = gum_code_allocator_alloc_slice (self->allocator);
      *need_deflector = TRUE;// 需要中继点
    }
  }
  // 无可用的临时寄存器，返回false	
  if (data->scratch_reg == ARM64_REG_INVALID)
    goto no_scratch_reg;

  return TRUE;

no_scratch_reg:
  {
    gum_code_slice_unref (ctx->trampoline_slice);
    ctx->trampoline_slice = NULL;
    return FALSE;
  }
}


```

这个函数主要是判断最大可用于重定向的空间（可用于 inline hook 空间），对于 16 字节的 inline hook，不需要中继器（deflector），而对于 4 字节或 8 字节的 inline hook，先尝试通过`gum_code_allocator_try_alloc_slice_near` 函数探寻附近页中是否存在足够可用的空余空间，如果没有找到，则需要中继器（deflector）。之后就是给`ctx->trampoline_slice`分配内存空间用于编写跳板，同时获取可用的临时寄存器 x16 或 x17。

对于可用空间为 4/8 字节的 inline hook，其跳转范围受限，因此引入中继器（deflector），先让一级跳板跳转到中继器，然后由于中继器可用空间足够，可以像 16 字节的 inline hook 一样，跳转的地址不再受到限制。

### 当前 hook 任务加入 hook 任务队列

接下来的代码逻辑回到`gum_interceptor_instrument`函数中， 下一步调用的关键函数是

```
gum_interceptor_transaction_schedule_update (&self->current_transaction, ctx, gum_interceptor_activate);


```

传入的第三个参数为`gum_interceptor_activate`函数指针。

#### gum_interceptor_transaction_schedule_update

```
static void
gum_interceptor_transaction_schedule_update (GumInterceptorTransaction * self,
                                             GumFunctionContext * ctx,
                                             GumUpdateTaskFunc func)
{
  guint8 * function_address;
  gpointer start_page, end_page;
  GArray * pending;
  GumUpdateTask update;

  function_address = _gum_interceptor_backend_get_function_address (ctx);

  start_page = gum_page_address_from_pointer (function_address);
  end_page = gum_page_address_from_pointer (function_address +
      ctx->overwritten_prologue_len - 1);

  pending = g_hash_table_lookup (self->pending_update_tasks, start_page);
  if (pending == NULL)
  {
    pending = g_array_new (FALSE, FALSE, sizeof (GumUpdateTask));
    g_hash_table_insert (self->pending_update_tasks, start_page, pending);
  }

  update.ctx = ctx;
  update.func = func; // 设置回调函数为gum_interceptor_activate
  g_array_append_val (pending, update);

  if (end_page != start_page)
  {
    pending = g_hash_table_lookup (self->pending_update_tasks, end_page);
    if (pending == NULL)
    {
      pending = g_array_new (FALSE, FALSE, sizeof (GumUpdateTask));
      g_hash_table_insert (self->pending_update_tasks, end_page, pending);
    }
  }
}


```

这段代码主要功能是将一个 Hook 更新任务（Update Task）按 “内存页” 进行归类并排期。在 Frida 中，为了提高性能并确保线程安全，修改内存（Patching）通常不是立即执行的，而是先收集所有任务，最后统一修改。由于修改内存涉及到修改页属性，按页归类可以确保每一页只被执行一次权限切换操作。

Hook 事务提交并执行
------------

接下来的代码逻辑回到`gum_interceptor_attach`函数中， 下一步调用的关键函数是

```
// 结束hook事务，提交并处理hook事务  
gum_interceptor_transaction_end (&self->current_transaction);


```

### gum_interceptor_transaction_end

```
static void
gum_interceptor_transaction_end (GumInterceptorTransaction * self)
{
  GumInterceptor * interceptor = self->interceptor;
  GumInterceptorTransaction transaction_copy;
  GPtrArray * addresses;
  GHashTableIter iter;
  gpointer address;

  self->level--;
  if (self->level > 0)
    return;

  if (!self->is_dirty)
    return;

  gum_interceptor_ignore_current_thread (interceptor);

  gum_code_allocator_commit (&interceptor->allocator);

  if (g_queue_is_empty (self->pending_destroy_tasks) &&
      g_hash_table_size (self->pending_update_tasks) == 0)
  {
    interceptor->current_transaction.is_dirty = FALSE;
    goto no_changes;
  }

  transaction_copy = interceptor->current_transaction;
  self = &transaction_copy;
  gum_interceptor_transaction_init (&interceptor->current_transaction,
      interceptor);

  addresses =
      g_ptr_array_sized_new (g_hash_table_size (self->pending_update_tasks));
  g_hash_table_iter_init (&iter, self->pending_update_tasks);
  while (g_hash_table_iter_next (&iter, &address, NULL))
    g_ptr_array_add (addresses, address);
  g_ptr_array_sort (addresses, (GCompareFunc) gum_page_address_compare);

  // ...
  // 关键函数gum_memory_patch_code_pages
  else if (!gum_memory_patch_code_pages (addresses, FALSE, gum_apply_updates,
        self))
  {
    g_abort ();
  }

  // ...
}


```

关键部分是调用`gum_memory_patch_code_pages`函数，这个函数在前面已经分析过了，但这里传入的第三个参数是`gum_apply_updates`函数指针。

### gum_apply_updates

```
static void
gum_apply_updates (gpointer source_page,
                   gpointer target_page,
                   guint n_pages,
                   gpointer user_data)
{
  GumInterceptorTransaction * self = user_data;
  GArray * pending;
  guint i;

  pending = g_hash_table_lookup (self->pending_update_tasks, target_page);
  g_assert (pending != NULL);

  for (i = 0; i != pending->len; i++)
  {
    GumUpdateTask * update;
    gsize offset;
    
    update = &g_array_index (pending, GumUpdateTask, i);
    // 原函数地址相对所在页的偏移
    offset = (guint8 *)
        _gum_interceptor_backend_get_function_address (update->ctx) -
        (guint8 *) target_page;
    // 回调 gum_interceptor_activate 函数
    update->func (self->interceptor, update->ctx,
        (guint8 *) source_page + offset);
  }
}


```

这里回调的是`gum_interceptor_transaction_schedule_update`函数中给`update->func`设置的`gum_interceptor_activate`函数。

### gum_interceptor_activate

```
static void
gum_interceptor_activate (GumInterceptor * self,
                          GumFunctionContext * ctx,
                          gpointer prologue) // 原函数地址
{
  if (ctx->destroyed)
    return;

  g_assert (!ctx->activated);
  ctx->activated = TRUE;

  _gum_interceptor_backend_activate_trampoline (self->backend, ctx,
      prologue);
}


```

### 一级跳板生成

#### _gum_interceptor_backend_activate_trampoline

```
void
_gum_interceptor_backend_activate_trampoline (GumInterceptorBackend * self,
                                              GumFunctionContext * ctx,
                                              gpointer prologue)
{
  GumArm64Writer * aw = &self->writer;
  GumArm64FunctionContextData * data = GUM_FCDATA (ctx);
  GumAddress on_enter;
  //设置一级跳板跳转的目标
  if (ctx->type == GUM_INTERCEPTOR_TYPE_FAST)// replace fast hook
    on_enter = GUM_ADDRESS (ctx->replacement_function);// replacement_function地址
  else
    on_enter = GUM_ADDRESS (ctx->on_enter_trampoline);// on_enter_trampoline地址

  //...
  
  // 初始化writer，写指针指向原函数起始地址
  gum_arm64_writer_reset (aw, prologue);
  aw->pc = GUM_ADDRESS (ctx->function_address);

  if (ctx->trampoline_deflector != NULL)// 需要中继器（deflector）
  {
    if (data->redirect_code_size == 8)//重定向空间大小为8字节
    {
      // STP X0, LR, [SP, #-16]!
      gum_arm64_writer_put_push_reg_reg (aw, ARM64_REG_X0, ARM64_REG_LR);
      // BL trampoline_deflector->trampoline
      gum_arm64_writer_put_bl_imm (aw,
          GUM_ADDRESS (ctx->trampoline_deflector->trampoline));
    }
    else// 重定向空间大小为4字节
    {
      g_assert (data->redirect_code_size == 4);
      // B trampoline_deflector->trampoline
      gum_arm64_writer_put_b_imm (aw,
          GUM_ADDRESS (ctx->trampoline_deflector->trampoline));
    }
  }
  else// 不需要中继器，直接跳转到 on_enter
  {
    switch (data->redirect_code_size)
    {
      case 4:
        // b on_enter
        gum_arm64_writer_put_b_imm (aw, on_enter);
        break;
      case 8:
        // ADRP X16/x17, on_enter
        gum_arm64_writer_put_adrp_reg_address (aw, data->scratch_reg, on_enter);
        // BR x16/x17
        gum_arm64_writer_put_br_reg_no_auth (aw, data->scratch_reg);
        break;
      case GUM_INTERCEPTOR_FULL_REDIRECT_SIZE:
        // #on_enter -> X16/X17
        gum_arm64_writer_put_ldr_reg_address (aw, data->scratch_reg, on_enter);
        // BR X16/X17
        gum_arm64_writer_put_br_reg (aw, data->scratch_reg);
        break;
      default:
        g_assert_not_reached ();
    }
  }

  gum_arm64_writer_flush (aw);
  g_assert (gum_arm64_writer_offset (aw) <= data->redirect_code_size);
}


```

这里就是一级跳板生成的地方了，该函数主要负责把被 hook 函数开头的几字节替换成跳转到下一跳板处（`on_enter_trampoline`或者`deflector`）的指令。根据可用空间和距离，它可能生成的指令如下图所示。

![](https://bbs.kanxue.com/upload/attach/202604/985561_H7SQ567PXYVEVKB.webp)

deflector trampoline 生成
-----------------------

这里续接二级跳板生成时需要中继器（deflector）的情况，对应代码为

```
// gum\backend-arm64\guminterceptor-arm64.c
// L722: _gum_interceptor_backend_create_trampoline

// 如果需要中继器（deflector），则在跳板代码中先保存 x0 和 lr 寄存器的值
if (need_deflector)
{
    GumAddressSpec caller;
    gpointer return_address;
    gboolean dedicated;
    // caller.near_address指向重定向指令中的最后一条指令
    caller.near_address =
        (guint8 *) function_address + data->redirect_code_size - 4;
    // caller.max_distance的值为b指令跳转的最大值(+128MB)
    caller.max_distance = GUM_ARM64_B_MAX_DISTANCE;
    // return_address指向重定向指令之后的原代码地址
    return_address = (guint8 *) function_address + data->redirect_code_size;
    // 是否是4字节重定向(inline hook)
    dedicated = data->redirect_code_size == 4;

    // 为中继器分配内存空间
    ctx->trampoline_deflector = gum_code_allocator_alloc_deflector (
        self->allocator, &caller, return_address, deflector_target, dedicated);
    if (ctx->trampoline_deflector == NULL)
    {
        gum_code_slice_unref (ctx->trampoline_slice);
        ctx->trampoline_slice = NULL;
        return FALSE;
    }
    // ldp x0，lr, [sp, #-16]!
    gum_arm64_writer_put_pop_reg_reg (aw, ARM64_REG_X0, ARM64_REG_LR);
}


```

这里通过`gum_code_allocator_alloc_deflector`函数为中继器分配内存空间。

### gum_code_allocator_alloc_deflector

```
// gum\gumcodeallocator.c
GumCodeDeflector *
gum_code_allocator_alloc_deflector (GumCodeAllocator * self,
                                    const GumAddressSpec * caller,
                                    gpointer return_address,//剩余原函数入口地址
                                    gpointer target,// replace fast模式为replacement_function
                                                    // ,否则为on_enter_trampoline入口地址
                                    gboolean dedicated)// 是否是4字节重定向
{
  GumCodeDeflectorDispatcher * dispatcher = NULL;
  GSList * cur;
  GumCodeDeflectorImpl * impl;
  GumCodeDeflector * deflector;

  if (!dedicated)// 8字节重定向空间
  { 
    // 遍历已有的中继器，找到一个距离被hook的目标函数足够近的DeflectorDispatcher
    for (cur = self->dispatchers; cur != NULL; cur = cur->next)
    {
      GumCodeDeflectorDispatcher * d = cur->data;
      gsize distance;
      // 计算DeflectorDispatcher与重定向代码中最后一条指令的距离
      distance = ABS ((gssize) GPOINTER_TO_SIZE (d->address) -
          (gssize) caller->near_address);
      // 如果距离在允许的范围内（B +-128MB），则使用该中继器
      if (distance <= caller->max_distance)
      {
        dispatcher = d;
        break;
      }
    }
  }
  //创建新的中继器，并添加到中继器链表中
  if (dispatcher == NULL)
  {
    dispatcher = gum_code_deflector_dispatcher_new (caller, return_address,
        dedicated ? target : NULL);
    if (dispatcher == NULL)
      return NULL;
    self->dispatchers = g_slist_prepend (self->dispatchers, dispatcher);
  }
  // 构建中继器实例
  impl = g_slice_new (GumCodeDeflectorImpl);

  deflector = &impl->parent;
  deflector->return_address = return_address;// 原函数被修改后继续执行的地址
  deflector->target = target;// 中继器的跳转目标（on_enter_trampoline 或 replacement_function）
  deflector->trampoline = dispatcher->trampoline;//分发器跳板的地址
  deflector->ref_count = 1;

  impl->allocator = self;
  // 将中继器添加到分发器的调用者列表中
  dispatcher->callers = g_slist_prepend (dispatcher->callers, deflector);

  return deflector;// 对应ctx->trampoline_deflector
}


```

这里分为两种情况，对于 8 字节 inline hook，先查找已有的 deflector dispatcher 是否可以复用（目标 dispatcher 在 ±128MB 内），如果没有找到，则同 4 字节的 inline hook 一样，调用`gum_code_deflector_dispatcher_new`函数创建新的 deflector dispatcher。

### gum_code_deflector_dispatcher_new

```
static GumCodeDeflectorDispatcher *
gum_code_deflector_dispatcher_new (const GumAddressSpec * caller,
                                   gpointer return_address, // 剩余原函数入口地址
                                   gpointer dedicated_target) // 4字节为跳转目标(replacement_function / on_enter_trampoline) 8字节为NULL
{
#if defined (HAVE_DARWIN) || (defined (HAVE_ELF) && GLIB_SIZEOF_VOID_P == 4)
  GumCodeDeflectorDispatcher * dispatcher;
  GumProbeRangeForCodeCaveContext probe_ctx;
  GumInsertDeflectorContext insert_ctx;
  gboolean remap_supported;

  remap_supported = gum_memory_can_remap_writable ();

  probe_ctx.caller = caller;

  probe_ctx.cave.base_address = 0;
  probe_ctx.cave.size = 0;
  // 枚举模块，寻找空白区域
  gum_process_enumerate_modules (gum_probe_module_for_code_cave, &probe_ctx);
  // 没找到用于deflector trampoline的空白区域，返回NULL
  if (probe_ctx.cave.base_address == 0)
    return NULL;
  // 根据找到的空白区域，初始化deflector dispatcher
  dispatcher = g_slice_new0 (GumCodeDeflectorDispatcher);

  dispatcher->address = GSIZE_TO_POINTER (probe_ctx.cave.base_address);

  dispatcher->original_data = g_memdup (dispatcher->address,
      probe_ctx.cave.size);
  dispatcher->original_size = probe_ctx.cave.size;
  // 8字节空间的inline hook，因为共用dispatcher，
  //所以需要额外开辟空间用于编写跳转到on_enter_trampoline或者replacement_function处的trampoline
  if (dedicated_target == NULL)
  {
    gsize thunk_size;
    GumMemoryRange range;
    GumPageProtection protection;

    thunk_size = gum_query_page_size ();
    protection = remap_supported ? GUM_PAGE_RX : GUM_PAGE_RW;
    // 给 dispatcher thunk分配新的页
    dispatcher->thunk =
        gum_memory_allocate (NULL, thunk_size, thunk_size, protection);
    dispatcher->thunk_size = thunk_size;
     // 赋予dispatcher thunk所在页权限，并回调gum_write_thunk函数写入trampoline
    gum_memory_patch_code (dispatcher->thunk, GUM_MAX_CODE_DEFLECTOR_THUNK_SIZE,
        (GumMemoryPatchApplyFunc) gum_write_thunk, dispatcher);

    range.base_address = GUM_ADDRESS (dispatcher->thunk);
    range.size = thunk_size;
    gum_cloak_add_range (&range);
  }
  // 初始化insert_ctx
  insert_ctx.pc = GUM_ADDRESS (dispatcher->address);// deflector dispatcher地址
  insert_ctx.max_size = dispatcher->original_size; // deflector dispatcher可用最大空间
  insert_ctx.return_address = return_address; // on_enter_trampoline 或者 replacement_function
  // 4字节为跳转目标(replacement_function / on_enter_trampoline) 8字节为NULL
  insert_ctx.dedicated_target = dedicated_target;

  insert_ctx.dispatcher = dispatcher;

  gum_memory_patch_code (dispatcher->address, dispatcher->original_size,
      (GumMemoryPatchApplyFunc) gum_insert_deflector, &insert_ctx);

  return dispatcher;
#else
  (void) gum_insert_deflector;
  (void) gum_write_thunk;
  (void) gum_probe_module_for_code_cave;

  return NULL;
#endif
}


```

对于 Android ARM32，则会通过`gum_process_enumerate_modules`枚举现有模块，对每个模块利用 `gum_probe_module_for_code_cave` 函数在其中找一段没用的、被对齐填充出来的空白区域（Code Cave）。然后这部分空白区域就被当作 delfector dispatcher，写入跳转指令（目标为 on_enter_trampoline 或者 replacement_function）。

对于 8 字节空间的 inline hook，额外开辟新的页给到`thunk`，然后回调`gum_write_thunk`函数写入跳板代码。

之后 4/8 字节的 inline hook，回调`gum_insert_deflector`函数往`dispatcher`（附近页找到的空白区域 cave）写入跳板代码。

对于 Android ARM64，不创建 deflector，而是直接返回 NULL。

接下来就以 Android ARM32 的情况进行分析。

#### 8 字节 inline hook 额外开辟 thunk 空间

##### gum_write_thunk

```
static void
gum_write_thunk (gpointer thunk, // dispatcher->thunk
                 GumCodeDeflectorDispatcher * dispatcher) // dispatcher
{
# if defined (HAVE_ARM)
  GumThumbWriter tw;

  gum_thumb_writer_init (&tw, thunk);
  tw.pc = GUM_ADDRESS (dispatcher->thunk);
 // push {r9-r12}
  gum_thumb_writer_put_push_regs (&tw, 2, ARM_REG_R9, ARM_REG_R12);
 // call gum_code_deflector_dispatcher_lookup(dispatcher, LR)
  gum_thumb_writer_put_call_address_with_arguments (&tw,
      GUM_ADDRESS (gum_code_deflector_dispatcher_lookup), 2,
      GUM_ARG_ADDRESS, GUM_ADDRESS (dispatcher),
      GUM_ARG_REGISTER, ARM_REG_LR);
 // pop {r9-r12}
  gum_thumb_writer_put_pop_regs (&tw, 2, ARM_REG_R9, ARM_REG_R12);
// BX R0
  gum_thumb_writer_put_bx_reg (&tw, ARM_REG_R0);
  gum_thumb_writer_clear (&tw);
# elif defined (HAVE_ARM64)// 不要误会，这里ARM64架构不一定是Android
  // ...
}


```

对于 Android ARM32，往 thunk 中填写的是 thumb 指令，具体如下：

![](https://bbs.kanxue.com/upload/attach/202604/985561_8KNXT9MKQ9Q3XSV.webp)

> `BX` (Branch and Exchange) 指令的主要作用是实现状态切换，即在 ARM 状态（32 位指令）和 Thumb 状态（16/32 位混合指令）之间来回切换。

##### gum_code_deflector_dispatcher_lookup

我们可以看看`gum_code_deflector_dispatcher_lookup`函数干了啥。

```
static gpointer
gum_code_deflector_dispatcher_lookup (GumCodeDeflectorDispatcher * self, // dispatcher
                                      gpointer return_address) // lr
{
  GSList * cur;

  for (cur = self->callers; cur != NULL; cur = cur->next)
  {
    GumCodeDeflector * caller = cur->data; // deflector

    if (caller->return_address == return_address)// 是目标原函数被修改后继续执行的地址
      return caller->target;// 跳转目标（on_enter_trampoline 或 replacement_function）
  }

  return NULL;
}


```

遍历`DeflectorDispatcher`中的`deflector`，如果传入的返回地址与`deflector`中的原函数返回地址一致，那么就返回这个`deflector`的跳转目标，里面存储的是`on_enter_trampoline`或者 `replacement_function`的地址。

该函数返回后，借助`bx r0`指令跳转到目标地址。

#### 4/8 字节 inline hook 的 dispatcher

##### gum_insert_deflector

```
static void
gum_insert_deflector (gpointer cave, // dispatcher->address
                      GumInsertDeflectorContext * ctx) // insert_ctx
{
# if defined (HAVE_ARM)
  GumCodeDeflectorDispatcher * dispatcher = ctx->dispatcher;
  GumThumbWriter tw;

  if (ctx->dedicated_target != NULL) // 4字节inline hook的deflector trampoline构造
  {
    gboolean owner_is_arm;
   
    owner_is_arm = (GPOINTER_TO_SIZE (ctx->return_address) & 1) == 0;
    if (owner_is_arm)// arm指令模式
    { // 往 cave 写入跳板代码
      GumArmWriter aw;

      gum_arm_writer_init (&aw, cave);
      aw.cpu_features = gum_query_cpu_features ();
      aw.pc = ctx->pc;
      // LDR PC, =target
      gum_arm_writer_put_ldr_reg_address (&aw, ARM_REG_PC,
          GUM_ADDRESS (ctx->dedicated_target));// dedicated_target为跳转目标(replacement_function / on_enter_trampoline)
      gum_arm_writer_flush (&aw);
      g_assert (gum_arm_writer_offset (&aw) <= ctx->max_size);
      gum_arm_writer_clear (&aw);

      dispatcher->trampoline = GSIZE_TO_POINTER (ctx->pc);

      return;
    }
    // thumb指令模式
    gum_thumb_writer_init (&tw, cave);
    tw.pc = ctx->pc;
    // LDR PC, =target（on_enter_trampoline or replacement function）
    gum_thumb_writer_put_ldr_reg_address (&tw, ARM_REG_PC,
        GUM_ADDRESS (ctx->dedicated_target));
  }
  else// 8字节inline hook的deflector trampoline构造
  {
    gum_thumb_writer_init (&tw, cave);
    tw.pc = ctx->pc;
    // LDR PC, =thunk (dispatcher thunk)
    gum_thumb_writer_put_ldr_reg_address (&tw, ARM_REG_PC,
        GUM_ADDRESS (dispatcher->thunk) + 1);
  }

  gum_thumb_writer_flush (&tw);
  g_assert (gum_thumb_writer_offset (&tw) <= ctx->max_size);
  gum_thumb_writer_clear (&tw);
  
  dispatcher->trampoline = GSIZE_TO_POINTER (ctx->pc + 1);
    
  //...

}


```

这部分代码 4/8 字节的 inline hook，都会执行。主要功能为往`dispatcher`（附近页找到的空白区域 cave）写入跳板代码。根据重定向可用空间为 4 字节或 8 字节，deflector trampoline 的构造情况具体如下。

![](https://bbs.kanxue.com/upload/attach/202604/985561_MZSCRYTD8H2JT69.webp)

总结
==

Frida 整体的 inline hook 实现原理图如下（可能需要挂梯子看，svg 图看雪上传不了，我挂 github 图床了）：

![](https://raw.githubusercontent.com/gal2xy/blog_img/main/img/202604102108475.svg)

![](https://raw.githubusercontent.com/gal2xy/blog_img/main/img/202604102109338.svg)

话说这需要 deflector 的流程中，4 字节 inline hook 的一级跳板并没有将 x0、lr 存入栈中，但它同样也跳转到`on_enter_trampoline`，根据二级跳板的生成，需要中继器 deflector 的会在`on_enter_trampoline`写入第一条指令`ldp x0，lr, [sp, #-0x10]!`，这样对 4 字节的 inline hook 不会有什么影响嘛？希望大佬帮忙解答一下我的疑惑。

参考：

https://deepwiki.com/frida/frida-gum

[Frida Interceptor Hook 实现原理图 | LLeaves Blog](https://blog.lleavesg.top/article/Frida Interceptor Hook 实现原理图)

[[培训]《冰与火的战歌：Windows 内核攻防实战》！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)

最后于 5 天前 被 gal2xy 编辑 ，原因： 图片注释

[#源码框架](forum-161-1-127.htm)