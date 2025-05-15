> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286837.htm)

> [原创]stalker 源码浅入浅出

### 目的  

记录一下之前的学习研究, 最初的想法是熟悉一下 stalker 的大体轮廓, 以便将来使用的时候万一需要修改或者定制, 能够比较快速的进入状态.  

### 原理  

stalker 是以基本块为单位, 翻译执行的, 它会把原始的二进制代码以基本块为单位, 拷贝到新内存, 然后对这个拷贝过来的基本块执行翻译, 也就是 api 中 transform 函数干的事情, 这个图片很形象了, 另外, stalker 是线程级别的, 只对某个线程 trace  

![](https://bbs.kanxue.com/upload/attach/202505/844633_8B6CV42F8DNTRWH.jpg)

### 核心数据结构  

_GumExecCtx, 这个是 stalker 的执行上下文, 这个指针本身是存储在 tls 中的, 线程独有, 指针指向的内容是放在正常堆内存中的, 以此来实现各个线程的执行上下文独立开来

```
1.struct _GumExecCtx
 
{
  volatile gint state;
  gint64 destroy_pending_since;
  GumStalker * stalker;
  GumThreadId thread_id;
  GumArm64Writer code_writer;
  GumArm64Writer slow_writer;
  GumArm64Relocator relocator;
  GumStalkerTransformer * transformer;
  void (* transform_block_impl) (GumStalkerTransformer * self,
      GumStalkerIterator * iterator, GumStalkerOutput * output);
  GumEventSink * sink;
  gboolean sink_started;
  GumEventType sink_mask;
  void (* sink_process_impl) (GumEventSink * self, const GumEvent * event,
      GumCpuContext * cpu_context);
  GumStalkerObserver * observer;
  gboolean unfollow_called_while_still_following;
  GumExecBlock * current_block;
  gpointer pending_return_location;
  gsize pending_calls;
  gpointer resume_at;
  gpointer return_at;
  gconstpointer activation_target;
  gpointer thunks;
  gpointer infect_thunk;
  GumAddress infect_body;
  GumSpinlock code_lock;
  GumCodeSlab * code_slab;
  GumSlowSlab * slow_slab;
  GumDataSlab * data_slab;
  GumCodeSlab * scratch_slab;
  GumMetalHashTable * mappings;
  gpointer last_prolog_minimal;
  gpointer last_epilog_minimal;
  gpointer last_prolog_full;
  gpointer last_epilog_full;
  gpointer last_invalidator;
 
  /*
   * GumExecBlocks are attached to a singly linked list when they are generated,
   * this allows us to store other data in the data slab (rather than relying on
   * them being found in there in sequential order).
   */
  GumExecBlock * block_list;
  /*
   * Stalker for AArch64 no longer makes use of a shadow stack for handling
   * CALL/RET instructions, so we instead keep a count of the depth of the stack
   * here when GUM_CALL or GUM_RET events are enabled.
   */
  gsize depth;
#ifdef HAVE_LINUX
  GumMetalHashTable * excluded_calls;
#endif
 
};
```

之后是 GumExecBlock, 这玩意儿就是 stalker 复制并且插装好的基本块, 它的内存布局如下  

```
+----------------------+ <-- block->code_start
| 插装后的代码          |
| (已转换/已重写的代码)  |
| (大小: block->code_size)|
+----------------------+ <-- block->code_start + block->code_size
| 原始代码快照          |     (这就是 gum_exec_block_get_snapshot_start 返回的位置)
| (大小: block->real_size)|
+----------------------+
```

可以看到, 除了保存了插装后的基本块, 他还保存了原始代码的快照, 为什么要保存这个快照呢, 是因为后续的优化, stalker 对基本块执行一次 transform, 之后再次执行这个基本块的时候, 它会用原始代码和这个快照对比, 如果没有发生变化, 就不会重新翻译, 变化了, 就会重新翻译; 下面是_GumExecBlock 和 stalker 本身的定义

```
struct _GumExecBlock
{
 
  /*
   * GumExecBlock instances are held in a singly linked list to allow them to be
   * disposed. This is necessary since other data may also be stored in the data
   * slab (e.g. inline caches) and hence we cannot simply rely on them being
   * contiguous.
   */
  GumExecBlock * next;
  GumExecCtx * ctx;
  GumCodeSlab * code_slab;
  GumSlowSlab * slow_slab;
  GumExecBlock * storage_block;
  guint8 * real_start;
  guint8 * code_start;
  guint8 * slow_start;
  guint real_size;
  guint code_size;
  guint slow_size;
  guint capacity;
  guint last_callout_offset;
  GumExecBlockFlags flags;
  gint recycle_count;
  GumIcEntry * ic_entries;
 
};
 
struct _GumStalker
{
  GObject parent;
 
  guint ic_entries;
 
  gsize ctx_size;
  gsize ctx_header_size;
 
  goffset thunks_offset;
  gsize thunks_size;
 
  goffset code_slab_offset;
  gsize code_slab_size_initial;
  gsize code_slab_size_dynamic;
 
  /*
   * The instrumented code which Stalker generates is split into two parts.
   * There is the part which is always run (the fast path) and the part which
   * is run only when attempting to find the next block and call the backpatcher
   * (the slow path). Backpatching is applied to the fast path so that
   * subsequent executions no longer need to transit the slow path.
   *
   * By separating the code in this way, we can improve the locality of the code
   * executing in the fast path. This has a performance benefit as well as
   * making the backpatched code much easier to read when working in the
   * debugger.
   *
   * The slow path makes use of its own slab and its own code writer.
   */
  goffset slow_slab_offset;
  gsize slow_slab_size_initial;
  gsize slow_slab_size_dynamic;
 
  goffset data_slab_offset;
  gsize data_slab_size_initial;
  gsize data_slab_size_dynamic;
 
  goffset scratch_slab_offset;
  gsize scratch_slab_size;
 
  gsize page_size;
  GumCpuFeatures cpu_features;
  gboolean is_rwx_supported;
 
  GMutex mutex;
  GSList * contexts;
 
  GArray * exclusions;
  gint trust_threshold;
  volatile gboolean any_probes_attached;
  volatile gint last_probe_id;
  GumSpinlock probe_lock;
  GHashTable * probe_target_by_id;
  GHashTable * probe_array_by_address;
 
  GumExceptor * exceptor;
};
```

stalker 本身是全局性的对象, 虽然执行的时候以线程独立的方式各自运行的, 但是各个线程都会用到一些通用的东西, 所以 stalker 本身设计成全局性的, 其中有一个变量叫做

```
gint trust_threshold;
```

这个变量配合着_GumExecBlock 中的

```
gint recycle_count;
```

每次执行到某个基本块, stalker 就会先去对比原始代码和快照, 上文也有提到, 如果没变化给 recycle_count 加一, 一直加到 trust_threshold 为止, 到了 trust_threshold, 就说明这个基本块已经稳定了, 不会发生变化, 此后如果再次执行到这个基本块, 就啥也不做了直接跳转到插好桩后的基本块, 这个信任阈值是用户指定的

之后是_GumSlab, 这个是 stalker 内存分配用的, 为了避免每次都使用 malloc 或者 mmap, 比较影响性能, 所以自己实现了内存分配机制, 先预先申请一大块内存, 之后需要用内存, 就从这个块里获取, 看一眼就知道大概意思和作用了, stalker 里保存着很多

```
_GumSlab
```

他们是以链表的形式保存的, 里面保存着 stalker 插装好的基本块, slob 还有快慢两种不同用处的内存块, 把执行频率高的代码集中放入快路径, 以提供 cache 命中率, 不过这是琐碎的细节优化问题, 借用著名外交官耿爽大使的一句话, 不必理会

```
struct _GumSlab
{
 
  guint8 * data;      // 数据区起始位置
 
  guint offset;       // 当前已使用的偏移量
 
  guint size;         // 可用数据区大小
 
  guint memory_size;  // 整个内存块大小（包括头部）
 
  GumSlab * next;     // 链表指向下一个slab
```

接下来是_GumStalkerIterator, 主要负责配合 transform 函数进行具体的翻译的, 后文会提到  

```
struct _GumStalkerIterator
{
  GumExecCtx * exec_context;
  GumExecBlock * exec_block;
  GumGeneratorContext * generator_context;
  GumInstruction instruction;
  GumVirtualizationRequirements requirements;
};
```

最后是_GumGeneratorContext, 他是翻译过程中用于生成指令的主要数据结构, 而更加具体的功能是利用 frida gum 提供的能力, 比如 GumArm64Relocator 用于将指令复制到新内存, 会自动处理 pc 相关指令, GumArm64Writer 用于生成指令

```
struct _GumGeneratorContext
{
  GumInstruction * instruction;
  GumArm64Relocator * relocator;
  GumArm64Writer * code_writer;
  GumArm64Writer * slow_writer;
  gpointer continuation_real_address;
  GumPrologType opened_prolog;
};
```

### 流程  

stalker 的入口有两个 api,gum_stalker_follow_me 和 gum_stalker_follow, 以 gum_stalker_follow 为例说明, 后文给出的代码只给出重要的部分, 省略一部分平凡的

```
1.gum_stalker_follow (GumStalker * self,
                    GumThreadId thread_id,
                    GumStalkerTransformer * transformer,
                    GumEventSink * sink)
    gumInfectContext ctx;
    ctx.stalker = self;
    ctx.transformer = transformer;
    ctx.sink = sink;
 
    gum_process_modify_thread (thread_id, gum_stalker_infect, &ctx,
 
        GUM_MODIFY_THREAD_FLAGS_NONE);
 
2.gum_process_modify_thread (GumThreadId thread_id,
                           GumModifyThreadFunc func,
                           gpointer user_data,
                           GumModifyThreadFlags flags)
                            
  GumModifyThreadContext ctx;
   
  ctx.func = func; //gum_stalker_infect
 
  ctx.user_data = user_data; //gum_stalker_follow的ctx,gumInfectContext
  return gum_linux_modify_thread (thread_id, GUM_REGS_GENERAL_PURPOSE,
      gum_do_modify_thread, &ctx, NULL);
 
3. gum_linux_modify_thread (GumThreadId thread_id,
                         GumLinuxRegsType regs_type,
                         GumLinuxModifyThreadFunc func,
                         gpointer user_data,
                         GError ** error)
 
  ctx.thread_id = thread_id;
  ctx.regs_type = regs_type;
  ctx.func = func; //gum_do_modify_thread
  ctx.user_data = user_data;  //gum_process_modify_thread的ctx,GumModifyThreadContext
 
     child = gum_libc_clone (
      gum_linux_do_modify_thread,
      stack + gum_query_page_size (),
      CLONE_VM | CLONE_SETTLS,
      &ctx,
      NULL,
      desc,
      NULL);
    prctl (PR_SET_PTRACER, child);
      if (thread_id == gum_process_get_current_thread_id ())
  {
    success = GPOINTER_TO_UINT (g_thread_join (g_thread_new (
            "gum-modify-thread-worker",
            gum_linux_handle_modify_thread_comms,
            &ctx)));
  }
  else
  {
    success = GPOINTER_TO_UINT (gum_linux_handle_modify_thread_comms (&ctx));
  }
 
4.gum_linux_handle_modify_thread_comms (gpointer data)
    GumLinuxModifyThreadContext * ctx = data;
    //func是 gum_do_modify_thread
    ctx->func (ctx->thread_id, &ctx->regs_data, ctx->user_data);
 
5.gum_do_modify_thread (GumThreadId thread_id,
                      GumRegs * regs,
                      gpointer user_data)
    GumModifyThreadContext * ctx = user_data;
    // func 是 gum_stalker_infect
    // user_data是这三个的ctx
    //    gumInfectContext ctx;
    //    ctx.stalker = self;
    //    ctx.transformer = transformer;
    //    ctx.sink = sink;
    ctx->func (thread_id, &cpu_context, ctx->user_data);
 
6.gum_stalker_infect (GumThreadId thread_id,
                    GumCpuContext * cpu_context,
                    gpointer user_data)
```

其中最后一个参数是事件相关的, stalker 支持一些事件以及回调, 比如执行基本块事件, 执行指令事件, 函数调用事件等等, 通过事件机制让用户在这些重要的时间点执行一些操作  

gum_stalker_follow 做了很多事情, 可分为两个阶段, 第一阶段是为第二阶段调用 gum_stalker_infect 做好各种准备, 也就是上面贴出来的代码; 太细的就不说了, 只说一些最重要的事情和大概的轮廓, stalker 是通过不断地拉取基本块, 翻译基本块, 执行基本块, 这样子工作的, 那么如何拉取基本块呢, 特别首次如何拉取基本块, 就是 gum_stalker_follow 做的事情了, 如果是 gum_stalker_follow_me 进入的, 那么拉取基本块就比较简单, 因为此时正在调用 gum_stalker_follow_me 函数, 只要获取的 lr 寄存器, 就可可以作为首个基本块的入口了;gum_stalker_follow 跟踪的并不是本线程, 而是跨线程, 所以有一些技术在里面, stalker 主要是通过 clone 系统调用, 生成一个新线程, 然后这个新线程通过 ptrace 系统调用, 去 attach 要跟踪的目标线程, 由此就可以拿到目标线程的寄存器上下文, 通过 pc 寄存器即可拿到首个基本块的入口地址了, 之所以采用 clone 而不是 fork, 是有两个原因的, 一是因为 stalker 期望与主进程共享内存, fork 出来的线程, 是一个新进程, 其内存是独立的, 第二个原因是 ptrace 系统调用不允许 attach 同一个线程组的线程, 使用 clone 可以比较精细的配置新线程, 让它属于不同线程组, 以满足这个要求;

注意, 此时出现了三个线程, 执行 gum_stalker_follow 的线程, 成为主线程, clone 出来的线程, 称为 clone 线程, 还有将要跟踪的目标线程, 主线程和 clone 线程通过 socket 通信, 主线程通过 clone 线程去 ptrace 目标线程, 拿到目标线程的寄存器上下文, 然后执行一定的逻辑, 这些逻辑会修改寄存器上下文, 最后让 clone 线程把被修改后的寄存器上下文写回目标线程, 主要的修改就是获取了目标线程的 pc, 通过 pc 拉取基本块, 翻译基本块, 之后把 pc 改为指向翻译好的基本块, 这样就完成了跟踪

上面说到 gum_stalker_follow 分为两阶段, 一阶段主要是配置和 clone 新线程, 二阶段就是在 clone 的线程中执行 gum_stalker_infect

```
1.gum_stalker_infect (GumThreadId thread_id,
                    GumCpuContext * cpu_context,
                    gpointer user_data)
 
  GumInfectContext * infect_context = user_data;
  GumStalker * self = infect_context->stalker;
  GumExecCtx * ctx;
  guint8 * pc;
  gpointer code_address;
  GumArm64Writer * cw;
  const guint potential_svc_size = 4;
 
  ctx = gum_stalker_create_exec_ctx (self, thread_id,
      infect_context->transformer, infect_context->sink);
  pc = GSIZE_TO_POINTER (gum_strip_code_address (cpu_context->pc));
  ctx->current_block = gum_exec_ctx_obtain_block_for (ctx, pc, &code_address);
 
2.gum_exec_ctx_obtain_block_for (GumExecCtx * ctx,
                               gpointer real_address,
                               gpointer * code_address)
  GumExecBlock * block;
  gum_spinlock_acquire (&ctx->code_lock);
  block = gum_metal_hash_table_lookup (ctx->mappings, real_address);
  if (block != NULL)
  {
    const gint trust_threshold = ctx->stalker->trust_threshold;
    gboolean still_up_to_date;
    still_up_to_date =
        (trust_threshold >= 0 && block->recycle_count >= trust_threshold) ||
        memcmp (block->real_start, gum_exec_block_get_snapshot_start (block),
            block->real_size) == 0;
    gum_spinlock_release (&ctx->code_lock);
    if (still_up_to_date)
    {
      if (trust_threshold > 0)
        block->recycle_count++;
    }
    else
    {
      gum_exec_ctx_recompile_block (ctx, block);
    }
  }
  else
  {
    block = gum_exec_block_new (ctx);
    block->real_start = real_address;
    gum_exec_block_maybe_inherit_exclusive_access_state (block, block->next);
    gum_exec_ctx_compile_block (ctx, block, real_address, block->code_start,
        GUM_ADDRESS (block->code_start), &block->real_size, &block->code_size,
        &block->slow_size);
    gum_exec_block_commit (block);
    gum_exec_block_propagate_exclusive_access_state (block);
    gum_metal_hash_table_insert (ctx->mappings, real_address, block);
    gum_spinlock_release (&ctx->code_lock);
    gum_exec_ctx_maybe_emit_compile_event (ctx, block);
  }
  *code_address = block->code_start;
  return block;
 
3.gum_exec_ctx_compile_block (GumExecCtx * ctx,
                            GumExecBlock * block,
                            gconstpointer input_code,
                            gpointer output_code,
                            GumAddress output_pc,
                            guint * input_size,
                            guint * output_size,
                            guint * slow_size)
 
 
  GumArm64Writer * cw = &ctx->code_writer;
  GumArm64Writer * cws = &ctx->slow_writer;
  GumArm64Relocator * rl = &ctx->relocator;
  GumGeneratorContext gc;
  GumStalkerIterator iterator;
  GumStalkerOutput output;
  gboolean all_labels_resolved;
  gboolean all_slow_labels_resolved;
  gum_arm64_writer_reset (cw, output_code);
  cw->pc = output_pc;
  gum_arm64_writer_reset (cws, block->slow_start);
  cws->pc = GUM_ADDRESS (block->slow_start);
  gum_arm64_relocator_reset (rl, input_code, cw);
  gum_ensure_code_readable (input_code, ctx->stalker->page_size);
  gc.instruction = NULL;
  gc.relocator = rl;
  gc.code_writer = cw;
  gc.slow_writer = cws;
  gc.continuation_real_address = NULL;
  gc.opened_prolog = GUM_PROLOG_NONE;
  iterator.exec_context = ctx;
  iterator.exec_block = block;
  iterator.generator_context = &gc;
  iterator.instruction.ci = NULL;
  iterator.instruction.start = NULL;
  iterator.instruction.end = NULL;
  iterator.requirements = GUM_REQUIRE_NOTHING;
  output.writer.arm64 = cw;
  output.encoding = GUM_INSTRUCTION_DEFAULT;
//我的猜测是保存这两个寄存器,是因为frida频繁用到,并且有自己的协议,而其他的寄存器,可能用的时候就已经保存了
gum_arm64_writer_put_ldp_reg_reg_reg_offset (cw, ARM64_REG_X16, ARM64_REG_X17,
      ARM64_REG_SP, 16 + GUM_RED_ZONE_SIZE, GUM_INDEX_POST_ADJUST);
 
  gum_exec_block_maybe_write_call_probe_code (block, &gc);
  ctx->pending_calls++;
  ctx->transform_block_impl (ctx->transformer, &iterator, &output);
  ctx->pending_calls--;
 
  if (gc.continuation_real_address != NULL)
  {
    GumBranchTarget continue_target = { 0, };
    continue_target.absolute_address = gc.continuation_real_address;
    continue_target.reg = ARM64_REG_INVALID;
    gum_exec_block_write_jmp_transfer_code (block, &continue_target,
      GUM_ENTRYGATE (jmp_continuation), &gc);
 
  }
```

首先通过 gum_stalker_create_exec_ctx 创建线程独立的执行上下文 GumExecCtx, 之后通过 gum_exec_ctx_obtain_block_for 和拿到基本块入口 (目标线程的 pc 寄存器值) 拉拉取基本块 GumExecBlock, 拉取基本块的逻辑是, 首先用入口地址查 hash 表, 看看是否翻译过, 如果翻译过就对比一下这个基本块 recycle_count 了多少次, 是否达到信任阈值, 如果达到了, 则直接跳转到翻译好的基本块, 如果没达到, 就对比是否有变化, 没变化的话就 recycle_count 加一, 然后跳转到翻译好的基本块, 有变化的话就触发 gum_exec_ctx_recompile_block, 重新翻译基本块, 最后跳转到翻译好的基本块; 如果是第一次执行这个基本块, 就会触发 transform 去翻译, 主要是通过 gum_exec_ctx_compile_block 函数完成, 可以看到, 在这个函数中调用了 ctx->transform_block_impl (ctx->transformer, &iterator, &output); 使用用户提高的 transform 回调去执行具体的翻译; 这个函数另一个值得注意的点是, 每次函数一进来会执行 gum_arm64_writer_put_ldp_reg_reg_reg_offset, 去恢复 x16 和 x17 寄存器, 这两个寄存器是 arm64 的约定中, 调用方和被调用方都不需要保存的, frida 选择了这两个寄存器来完成全内存间接跳转, 也就是 br/blr 寄存器, 之所以要在每次执行 gum_exec_ctx_compile_block 的时候要恢复这两个寄存器, 原因是, stalker 执行的时候是一个基本块一个基本块执行的, 假如此时正在执行的是翻译后的基本块 a, 执行完 a 以后会去执行下一个翻译好的基本块, 假若下一个基本块尚未翻译好, 就要执行 gum_exec_ctx_compile_block 来翻译, 而 stalker 跳转是通过 x16,x17 跳转的, 污染了这两个寄存器, 所以跳转前会先保存, 跳转后再恢复, 以保证寄存器上下文一致, 特别的, 如果不是通过 br 跳转到 gum_exec_ctx_compile_block 函数的, 比如通过 bl, 那么 stalker 并不会跳到 gum_exec_ctx_compile_block 首地址, 而是跳过了第一条指令, 也就是不用恢复 x16 和 x17 了

gum_exec_ctx_compile_block 函数中有一些涉及 gc.continuation_real_address 的逻辑, 这些逻辑主要是处理内存不够的情况, 比如翻译某个基本块的时候, 翻译到原始基本块某条指令, 发现存放翻译后基本块的内存不够了, 就会使用 gc.continuation_real_address 记录下原始基本块这条指令的下一条指令, 待 stalker 分配了新内存块后从这里继续翻译, 并且处理从内存不够的基本块跳转到新分配基本块的那些逻辑,

最后看看 gum_stalker_iterator_next 函数, 这个函数式 transform 函数会用到的, 用于迭代原始基本块的所有指令, 以完成用户自定义的插装逻辑

```
gboolean
gum_stalker_iterator_next (GumStalkerIterator * self,
                           const cs_insn ** insn)
                            
  GumGeneratorContext * gc = self->generator_context;
  GumArm64Relocator * rl = gc->relocator;
  GumInstruction * instruction;
  gboolean is_first_instruction;
  guint n_read;
  instruction = self->generator_context->instruction;
  is_first_instruction = instruction == NULL;
 
    //判断slob空间是否足够
   if (gum_stalker_iterator_is_out_of_space (self))
    {
    // 使用 continuation_real_address 空间不足设置继续点当前指令的下一条指令为继续点
      gc->continuation_real_address = instruction->end;
      return FALSE;
    }
    // 判断最后一条,relocator把最后的设为eob
     if (!skip_implicitly_requested && gum_arm64_relocator_eob (rl))
      return FALSE;
  instruction = &self->instruction;
  n_read = gum_arm64_relocator_read_one (rl, &instruction->ci);
  if (n_read == 0)
    return FALSE;
  instruction->start = GSIZE_TO_POINTER (instruction->ci->address);
  instruction->end = instruction->start + instruction->ci->size;
  把当前指令赋值到generator_context,这样下一轮的时候generator_context的instruction就是当前指令了,对应了前面的检测generator_context->instruction是否为空来检测而是不是第一条指令的逻辑了
  self->generator_context->instruction = instruction;
 
 
void
gum_stalker_iterator_keep (GumStalkerIterator * self)
{
  
  // 写执行指令事件
  if ((self->exec_context->sink_mask & GUM_EXEC) != 0 &&
      (block->flags & GUM_EXEC_BLOCK_USES_EXCLUSIVE_ACCESS) == 0)
  {
    gum_exec_block_write_exec_event_code (block, gc, GUM_CODE_INTERRUPTIBLE);
  }
 
  switch (insn->id)
  {
    //对各种分割基本块的指令都会去虚拟化处理,以夺回控制权
      requirements = gum_exec_block_virtualize_branch_insn (block, gc);
      break;
      requirements = gum_exec_block_virtualize_ret_insn (block, gc);
   // 处理svc
    case ARM64_INS_SVC:
      requirements = gum_exec_block_virtualize_sysenter_insn (block, gc);
      break;
      requirements = GUM_REQUIRE_RELOCATION;
  }
 
  gum_exec_block_close_prolog (block, gc, gc->code_writer);
 
  if ((requirements & GUM_REQUIRE_RELOCATION) != 0)
  // 写指令
    gum_arm64_relocator_write_one (rl);
  self->requirements = requirements;
 
}
```

首先判断 slob 内存, 如不足, 通过

```
continuation_real_address
```

记录好断点位置的下一条指令, 作为继续点

第二个事情是 GumGeneratorContext 的 instruction 存放着正在翻译的基本块正在迭代的那条指令, 初始的时候设为空, 这样根据是否为空就可以判断当前翻译的是不是第一条指令了; 往下看, 通过 gum_arm64_relocator_read_one 函数从原始基本块中读取一条指令, 赋给 self->generator_context->instruction.  gum_arm64_relocator_read_one 是 GumArm64Relocator 的成员函数, 因为我们翻译基本块的时候, 很多时候都是会保留原始指令的, 这就相当于把一条指令从原始基本块移到到新基本块, 就需要用到 GumArm64Relocator, 而翻译的过程, 其实也就是逐条迭代 GumArm64Relocator 中所有指令 (某个基本块的所有指令) 的过程, 迭代的过程中, 用户可以选择复制过来, 或者不复制, 同时也可以进行一些别的操作, 比如插入回调, 插入指令等等

对于 stalker, 了解原理之后我就有两个疑问, 一个是首次的时候他是怎么劫持到控制流, 然后开始他的翻译执行过程的, 这个问题上文已经回答了, 第二个问题是, 它执行完某个翻译好的基本块之后, 是怎么拿回控制权的, 现在开始回答这个问题, 首先是 gum_stalker_iterator_keep 函数

```
void
gum_stalker_iterator_keep (GumStalkerIterator * self)
{
  
  // 写执行指令事件
  if ((self->exec_context->sink_mask & GUM_EXEC) != 0 &&
      (block->flags & GUM_EXEC_BLOCK_USES_EXCLUSIVE_ACCESS) == 0)
  {
    gum_exec_block_write_exec_event_code (block, gc, GUM_CODE_INTERRUPTIBLE);
  }
 
  switch (insn->id)
  {
    //对各种分割基本块的指令都会去虚拟化处理,以夺回控制权
      requirements = gum_exec_block_virtualize_branch_insn (block, gc);
      break;
      requirements = gum_exec_block_virtualize_ret_insn (block, gc);
   // 处理svc
    case ARM64_INS_SVC:
      requirements = gum_exec_block_virtualize_sysenter_insn (block, gc);
      break;
      requirements = GUM_REQUIRE_RELOCATION;
  }
 
  gum_exec_block_close_prolog (block, gc, gc->code_writer);
 
  if ((requirements & GUM_REQUIRE_RELOCATION) != 0)
  // 写指令
    gum_arm64_relocator_write_one (rl);
  self->requirements = requirements;
 
}
```

这个函数主要是把原始基本块中的指令放到新基本块, 主要利用 GumArm64Relocator 完成; 核心需要关注的地方是, stalker 回去判断指令, 是否是会分割基本块, 比如各种跳转, 函数调用指令等等, 对于会分割基本块的指令, stalker 会通过 gum_exec_block_virtualize_* 系列函数来对指令进行特殊处理, 这里叫做虚拟化, 以 gum_exec_block_virtualize_branch_insn 为例

```
// 以无条件跳转为例
static GumVirtualizationRequirements
 
gum_exec_block_virtualize_branch_insn (GumExecBlock * block,
 
                                       GumGeneratorContext * gc)
 
{
 
        ...
   else
      {
        if (target.reg != ARM64_REG_INVALID)
          regular_entry_func = GUM_ENTRYGATE (jmp_reg);
        else
          regular_entry_func = GUM_ENTRYGATE (jmp_imm);
        cond_entry_func = NULL;
      }
 
      gum_exec_block_write_jmp_transfer_code (block, &target,
          is_conditional ? cond_entry_func : regular_entry_func, gc);
 
}
```

这个函数主要是检查是哪种跳转指令, 执行不同的逻辑, 但是大体是相同的, 所以随便挑了一个来看, 首先调用 GUM_ENTRYGATE 宏生成一个函数名, 不同的跳转指令会调用不同的函数, 之后调用 gum_exec_block_write_jmp_transfer_code 把这个函数写入新基本块中, 代替那条会分割基本块的指令,

gum_exec_block_write_jmp_transfer_code 函数如下

```
static void
gum_exec_block_write_jmp_transfer_code (GumExecBlock * block,
                                        const GumBranchTarget * target,
                                        GumExecCtxReplaceCurrentBlockFunc func,
                                        GumGeneratorContext * gc)
 
 
  这部分处理信任阈值有效但不能静态回填的情况（寄存器间接跳转）：
  1. 关闭任何已打开的序言代码
  2. 生成内联缓存查找代码，存储到快速路径
  3. 将结果放入寄存器并使用无身份验证的分支指令跳转
  内联缓存是一种重要的优化技术，它缓存之前跳转的目标及其对应的已插桩代码，加速后续相同目标的跳转。
  if (trust_threshold >= 0 && !can_backpatch_statically)
  {
  arm64_reg result_reg;
  gum_exec_block_close_prolog (block, gc, cw);
 
  /* 内联缓存处理逻辑 */
  result_reg = gum_exec_block_write_inline_cache_code (block, target->reg,
      cw, cws);
  gum_arm64_writer_put_br_reg_no_auth (cw, result_reg);
 
  }
  // 直接分支处理
  else
{
  guint i;
  /* 直接跳转到慢速路径 */
  gum_exec_block_write_slab_transfer_code (cw, cws);
  /* 添加NOP填充，为后续可能的回填留空间 */
  for (i = 0; i != 10; i++)
    gum_arm64_writer_put_nop (cw);
}
 
  // 慢路径实现
  1. 打开最小序言以保存寄存器状态
  2. 将跳转目标地址压栈后弹出到X0/X1寄存器
  3. 调用传入的替换函数，传递当前块、目标地址和指令地址
    - 这个函数会查找或编译目标块，然后切换到该块
  gum_exec_block_open_prolog (block, GUM_PROLOG_MINIMAL, gc, cws);
  gum_exec_ctx_write_push_branch_target_address (block->ctx, target, gc, cws);
  gum_arm64_writer_put_pop_reg_reg (cws, ARM64_REG_X0, ARM64_REG_X1);
  gum_arm64_writer_put_call_address_with_arguments (cws,
    GUM_ADDRESS (func), 3,
    GUM_ARG_ADDRESS, GUM_ADDRESS (block),
    GUM_ARG_REGISTER, ARM64_REG_X1,
   
    GUM_ARG_ADDRESS, GUM_ADDRESS (gc->instruction->start));
```

可以看到主要是调用 gum_arm64_writer_put_call_address_with_arguments, 来写入调用 GUM_ENTRYGATE 宏生成的函数, 看下这个宏

```
#define GUM_ENTRYGATE(name) \
 
    gum_exec_ctx_replace_current_block_from_##name
 
#define GUM_DEFINE_ENTRYGATE(name) \
 
    static gpointer \
 
    GUM_ENTRYGATE (name) ( \
 
        GumExecBlock * block, \
 
        gpointer start_address, \
 
        gpointer from_insn) \
 
    { \
 
      GumExecCtx * ctx = block->ctx; \
 
      \
 
      if (ctx->observer != NULL) \
 
        gum_stalker_observer_increment_##name (ctx->observer); \
 
      \
 
      return gum_exec_ctx_switch_block (ctx, block, start_address, from_insn); \
 
    }
 
   
 
GUM_DEFINE_ENTRYGATE (call_imm)
 
GUM_DEFINE_ENTRYGATE (call_reg)
 
GUM_DEFINE_ENTRYGATE (excluded_call_imm)
 
GUM_DEFINE_ENTRYGATE (excluded_call_reg)
 
GUM_DEFINE_ENTRYGATE (ret)
 
   
 
GUM_DEFINE_ENTRYGATE (jmp_imm)
 
GUM_DEFINE_ENTRYGATE (jmp_reg)
 
   
 
GUM_DEFINE_ENTRYGATE (jmp_cond_cc)
 
GUM_DEFINE_ENTRYGATE (jmp_cond_cbz)
 
GUM_DEFINE_ENTRYGATE (jmp_cond_cbnz)
 
GUM_DEFINE_ENTRYGATE (jmp_cond_tbz)
 
GUM_DEFINE_ENTRYGATE (jmp_cond_tbnz)
 
   
 
GUM_DEFINE_ENTRYGATE (jmp_continuation)
 
核心是通过gum_exec_ctx_switch_block切换基本块,同时stalker始终拥有控制权
这个函数是真正执行"查找或编译目标块，然后切换"的核心
    
 
核心是通过gum_exec_ctx_switch_block切换基本块,同时stalker始终拥有控制权
这个函数是真正执行"查找或编译目标块，然后切换"的核心
```

GUM_ENTRYGATE 宏生成函数名, GUM_DEFINE_ENTRYGATE 生成各种对应的函数, 各个函数的核心目的只有一个, 就是去调用

```
gum_exec_ctx_switch_block
```

由此, 我们就回答了第二个问题了; 在 stalker 执行完任何基本块后, 都会回到 gum_exec_ctx_switch_block, 夺回控制权之后, 继续分发下一个要执行的基本块, 以此形成闭环, stalker 就可以不断的拉取 - 翻译 - 执行基本块了

### 题外话 - stalker 之殇  

在实际使用 stalker 的时候, 发现偶尔会报一个错误  

Unable to allocate code slab near 0x6fd644d000 with max_distance=2138779647

找到对应源码  

```
static GumCodeSlab *
gum_code_slab_new (GumExecCtx * ctx)
{
  GumStalker * stalker = ctx->stalker;
  gsize total_size;
  GumCodeSlab * code_slab;
  GumSlowSlab * slow_slab;
  GumAddressSpec spec;
 
  total_size = stalker->code_slab_size_dynamic +
      stalker->slow_slab_size_dynamic;
 
  gum_exec_ctx_compute_code_address_spec (ctx, total_size, &spec);
 
  code_slab = gum_memory_allocate_near (&spec, total_size, stalker->page_size,
      stalker->is_rwx_supported ? GUM_PAGE_RWX : GUM_PAGE_RW);
  if (code_slab == NULL)
  {
    g_error ("Unable to allocate code slab near %p with max_distance=%zu",
        spec.near_address, spec.max_distance);
  }
 
  gum_code_slab_init (code_slab, stalker->code_slab_size_dynamic, total_size,
      stalker->page_size);
 
  slow_slab = gum_slab_end (&code_slab->slab);
  gum_slow_slab_init (slow_slab, stalker->slow_slab_size_dynamic, 0,
      stalker->page_size);
 
  return code_slab;
}
```

gum_code_slab_new 中, stalker 通过 gum_memory_allocate_near 分配一些邻近 GumExecCtx 的内存, 以方便它使用相对跳转指令来跳过去, 但是程序执行起来之后, 内存碎片众多, 哪有那么多相邻 GumExecCtx, 而且又比较大的内存给你使用呢, 这个问题有几个 issue 提了

[对应 issue](elink@ea3K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6X3M7X3W2V1j5g2)9J5c8X3k6J5K9h3c8S2i4K6u0r3K9i4y4K6N6h3g2K6i4K6u0r3x3U0R3I4z5b7`.`.)

大胡子好像没理, 只能说, 大胡子已经不是从前那个可爱的宝宝了

![](https://bbs.kanxue.com/upload/attach/202505/844633_6KAQ4HJ2K96HNAH.jpg)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#源码框架](forum-161-1-127.htm)