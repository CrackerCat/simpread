> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/_xiao/article/details/40619797)

最近安卓项目中碰到一个踩内存导致死机的问题，通过分析 Log 找到了被踩的内存地址，但无法找到是谁踩掉的。一般踩内存的问题，可以通过下硬件数据断点来找到肇事者。但在这个项目中，踩内存是在安卓开机过程中发生的，来不及上 JTAG，另外被踩的内存是动态分配出来的，每次开机都不同（但总是踩在这个被分配的地址的固定字节上），无法预先指定断点地址（如果是全局变量被踩的话，其地址一般是不变的，可以直接在被踩的地址上下数据断点），所以这种情况下只能通过程序来给自己下断点（self-host），程序分配内存后，马上给被踩的内存下个数据断点（已知踩内存肯定发生在这个地址上），当其它程序意外写入该内存时，就会发生异常中断，从而找到踩内存的肇事者。

不幸的是，百度后，几乎没有文章讲这个事情怎么做。翻了 GDB 的代码，也没看到 ARM 中如何下硬件断点（所用的 GDB 版本太旧了）。后来多方搜索，找到一些信息，记录在下：

1. 在 ARM 的架构文档中（ARM 官网可下载），Cortext A8, A9, A15 才支持硬件断点（通过协处理器 CP14 操作调试寄存器 DBGWCR 和 DBGWVR 来下数据断点 (watchpoint)，Processor 和 JTAG Debuger 均可以操作它们）（现在跑安卓的机器一般都是 A9 以上的架构，所以基本都支持硬件断点）。

2. Linux 内核要在 2.6.37 以后的版本才支持对 ARM 添加硬件断点。

3. GDB 要在 7.3 以后的版本才支持对 ARM 添加硬件断点（最新版本是 7.8.1，2014 年 10 月）。

如果是自己的 ARM 系统，可以写汇编，通过 MCR 协处理器指令操作 CP14，从而读写 DBGWCR 和 DBGWVR 来下硬件断点。

如果是 Linux 系统（2.6.37 版本以上），就简单了，可以通过 ptrace 函数来操作 DBGWCR 和 DBGWVR，下硬件断点。

以下是从 GDB 7.8.1 中截取的下硬件断点的部分代码：

```
  for (i = 0; i < arm_linux_get_hw_breakpoint_count (); i++)
    if (arm_lwp_info->bpts_changed[i])
      {
        errno = 0;
        if (arm_hwbp_control_is_enabled (bpts[i].control))
          if (<span style="color:#ff0000;"><strong>ptrace (PTRACE_SETHBPREGS, pid,
              (PTRACE_TYPE_ARG3) ((i << 1) + 1), &bpts[i].address)</strong></span> < 0)
            perror_with_name (_("Unexpected error setting breakpoint"));
 
        if (bpts[i].control != 0)
          if (<span style="color:#ff0000;"><strong>ptrace (PTRACE_SETHBPREGS, pid,
              (PTRACE_TYPE_ARG3) ((i << 1) + 2), &bpts[i].control)</strong></span> < 0)
            perror_with_name (_("Unexpected error setting breakpoint"));
 
        arm_lwp_info->bpts_changed[i] = 0;
      }
 
  for (i = 0; i < arm_linux_get_hw_watchpoint_count (); i++)
    if (arm_lwp_info->wpts_changed[i])
      {
        errno = 0;
        if (arm_hwbp_control_is_enabled (wpts[i].control))
          if (<span style="color:#ff0000;"><strong>ptrace (PTRACE_SETHBPREGS, pid,
              (PTRACE_TYPE_ARG3) -((i << 1) + 1), &wpts[i].address)</strong></span> < 0)
            perror_with_name (_("Unexpected error setting watchpoint"));
 
        if (wpts[i].control != 0)
          if (<span style="color:#ff0000;"><strong>ptrace (PTRACE_SETHBPREGS, pid,
              (PTRACE_TYPE_ARG3) -((i << 1) + 2), &wpts[i].control)</strong></span> < 0)
            perror_with_name (_("Unexpected error setting watchpoint"));
 
        arm_lwp_info->wpts_changed[i] = 0;
      }
```

可以看到，调用 ptrace，参数使用 PTRACE_GETHBPREGS 和 PTRACE_SETHBPREGS 即可读写硬件断点寄存器，其中第 3 个参数是寄存器的索引号，正数是读写 breakpoint 寄存器，负数是读写 watchpoint 寄存器（即数据断点）。+1，+2 是读写第 1 个断点的地址和控制寄存器，+3，+4 是读写第 2 个断点的地址和控制寄存器，以此类推；而 - 1，-2 是读写第 1 个数据断点的地址和控制寄存器，-3，-4 是读写第 2 个数据断点的地址和控制寄存器，以此类推。如果第 3 个参数为 0，则表示读取调试信息，例如支持多少个硬件断点，多少个数据断点等。

查看 linux kernel 3.1.10 的代码，看看 ptrace 的实现，在 3.1.10/kernel/ptrace.c 中是 ptrace 的代码：

SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr, unsigned long, data)  

它会调用架构相关的 arch_ptrace 函数实现调试功能，在 ARM 中其实现在 3.1.10/arch/arm/kernel/ptrace.c 中：

long arch_ptrace(struct task_struct *child, long request, unsigned long addr, unsigned long data)  

在此函数中，对 PTRACE_GETHBPREGS 和 PTRACE_SETHBPREGS 分别调用 ptrace_gethbpregs 和 ptrace_sethbpregs 函数：

```
#ifdef CONFIG_HAVE_HW_BREAKPOINT
		case PTRACE_GETHBPREGS:
			if (ptrace_get_breakpoints(child) < 0)
				return -ESRCH;
 
			ret = ptrace_gethbpregs(child, addr,
						(unsigned long __user *)data);
			ptrace_put_breakpoints(child);
			break;
		case PTRACE_SETHBPREGS:
			if (ptrace_get_breakpoints(child) < 0)
				return -ESRCH;
 
			ret = ptrace_sethbpregs(child, addr,
						(unsigned long __user *)data);
			ptrace_put_breakpoints(child);
			break;
#endif
```

在 ptrace_gethbpregs 中，如果 num 为 0，调用 ptrace_get_hbp_resource_info 来获取调试信息，如果 num 不为 0，先调用 ptrace_hbp_num_to_idx 将 num 转换为索引（按前文的 + 1,+2, +3,+4, ... 规则转换，然后返回线程中所缓存的调试配置信息。

```
static int ptrace_gethbpregs(struct task_struct *tsk, long num,
			     unsigned long  __user *data)
{
	u32 reg;
	int idx, ret = 0;
	struct perf_event *bp;
	struct arch_hw_breakpoint_ctrl arch_ctrl;
 
	if (num == 0) {
		reg = ptrace_get_hbp_resource_info();
	} else {
		idx = ptrace_hbp_num_to_idx(num);
		if (idx < 0 || idx >= ARM_MAX_HBP_SLOTS) {
			ret = -EINVAL;
			goto out;
		}
 
		bp = tsk->thread.debug.hbp[idx];
		if (!bp) {
			reg = 0;
			goto put;
		}
 
		arch_ctrl = counter_arch_bp(bp)->ctrl;
 
		/*
		 * Fix up the len because we may have adjusted it
		 * to compensate for an unaligned address.
		 */
		while (!(arch_ctrl.len & 0x1))
			arch_ctrl.len >>= 1;
 
		if (num & 0x1)
			reg = bp->attr.bp_addr;
		else
			reg = encode_ctrl_reg(arch_ctrl);
	}
 
put:
	if (put_user(reg, data))
		ret = -EFAULT;
 
out:
	return ret;
}
```

顺着 ptrace_get_hbp_resource_info 一路看下去，其调用 hw_breakpoint_slots，后者再调用 get_num_brps，再后者再调用 ARM_DBG_READ 读取调试寄存器的值。

在 ptrace_get_hbp_resurce_info 中，可以看到其返回值由 4 个字节组合而成，最高字节是系统支持的硬件断点数（最大不超过 16），次高字节是系统支持的数据断点数 (watchpoint)（最大不超过 16），次低字节是 watchpoint 支持的最大长度，最低字节是调试信息版本号：

```
static u32 ptrace_get_hbp_resource_info(void)
{
	u8 num_brps, num_wrps, debug_arch, wp_len;
	u32 reg = 0;
 
	num_brps	= hw_breakpoint_slots(TYPE_INST);
	num_wrps	= hw_breakpoint_slots(TYPE_DATA);
	debug_arch	= arch_get_debug_arch();
	wp_len		= arch_get_max_wp_len();
 
	reg		|= debug_arch;
	reg		<<= 8;
	reg		|= wp_len;
	reg		<<= 8;
	reg		|= num_wrps;
	reg		<<= 8;
	reg		|= num_brps;
 
	return reg;
}
 
int hw_breakpoint_slots(int type)
{
	if (!debug_arch_supported())
		return 0;
 
	/*
	 * We can be called early, so don't rely on
	 * our static variables being initialised.
	 */
	switch (type) {
	case TYPE_INST:
		return get_num_brps();
	case TYPE_DATA:
		return get_num_wrps();
	default:
		pr_warning("unknown slot type: %d\n", type);
		return 0;
	}
}
 
/* Determine number of usable BRPs available. */
static int get_num_brps(void)
{
	int brps = get_num_brp_resources();
	if (core_has_mismatch_brps())
		brps -= get_num_reserved_brps();
	return brps;
}
 
/* Determine number of BRP register available. */
static int get_num_brp_resources(void)
{
	u32 didr;
	ARM_DBG_READ(c0, 0, didr);
	return ((didr >> 24) & 0xf) + 1;
}
```

最关键的，来看 ARM_DBG_READ 的实现。ARM_DBG_READ 和 ARM_DBG_WRITE 其实是一组宏，用来读取和写入调试寄存器，它们在 ARM 上就是一条汇编，通过 mrc 和 mcr 操作协处理器 CP14 来完成调试功能，其定义如下：

```
/* Accessor macros for the debug registers. */
#define ARM_DBG_READ(M, OP2, VAL) do {\
	asm volatile("mrc p14, 0, %0, c0," #M ", " #OP2 : "=r" (VAL));\
} while (0)
 
#define ARM_DBG_WRITE(M, OP2, VAL) do {\
	asm volatile("mcr p14, 0, %0, c0," #M ", " #OP2 : : "r" (VAL));\
} while (0)
```

以上就看到了 linux 如何操作 CP14 来获取调试信息。

继续来看 ptrace_sethbpregs 的实现，它先校验输入信息的有效性，然后用 decode_ctrl_reg 将信息组合成要写入到 DBG 寄存器的值，最后调用 modify_user_hw_breakpoint 来提交设置动作。

```
static int ptrace_sethbpregs(struct task_struct *tsk, long num,
			     unsigned long __user *data)
{
	int idx, gen_len, gen_type, implied_type, ret = 0;
	u32 user_val;
	struct perf_event *bp;
	struct arch_hw_breakpoint_ctrl ctrl;
	struct perf_event_attr attr;
 
	if (num == 0)
		goto out;
	else if (num < 0)
		implied_type = HW_BREAKPOINT_RW;
	else
		implied_type = HW_BREAKPOINT_X;
 
	idx = ptrace_hbp_num_to_idx(num);
	if (idx < 0 || idx >= ARM_MAX_HBP_SLOTS) {
		ret = -EINVAL;
		goto out;
	}
 
	if (get_user(user_val, data)) {
		ret = -EFAULT;
		goto out;
	}
 
	bp = tsk->thread.debug.hbp[idx];
	if (!bp) {
		bp = ptrace_hbp_create(tsk, implied_type);
		if (IS_ERR(bp)) {
			ret = PTR_ERR(bp);
			goto out;
		}
		tsk->thread.debug.hbp[idx] = bp;
	}
 
	attr = bp->attr;
 
	if (num & 0x1) {
		/* Address */
		attr.bp_addr	= user_val;
	} else {
		/* Control */
		decode_ctrl_reg(user_val, &ctrl);
		ret = arch_bp_generic_fields(ctrl, &gen_len, &gen_type);
		if (ret)
			goto out;
 
		if ((gen_type & implied_type) != gen_type) {
			ret = -EINVAL;
			goto out;
		}
 
		attr.bp_len	= gen_len;
		attr.bp_type	= gen_type;
		attr.disabled	= !ctrl.enabled;
	}
 
	ret = modify_user_hw_breakpoint(bp, &attr);
out:
	return ret;
}
```

modify_user_hw_breakpoint 只是将动作通过 perf_event_enable 提交。

```
int modify_user_hw_breakpoint(struct perf_event *bp, struct perf_event_attr *attr)
{
	u64 old_addr = bp->attr.bp_addr;
	u64 old_len = bp->attr.bp_len;
	int old_type = bp->attr.bp_type;
	int err = 0;
 
	perf_event_disable(bp);
 
	bp->attr.bp_addr = attr->bp_addr;
	bp->attr.bp_type = attr->bp_type;
	bp->attr.bp_len = attr->bp_len;
 
	if (attr->disabled)
		goto end;
 
	err = validate_hw_breakpoint(bp);
	if (!err)
		perf_event_enable(bp);
 
	if (err) {
		bp->attr.bp_addr = old_addr;
		bp->attr.bp_type = old_type;
		bp->attr.bp_len = old_len;
		if (!bp->attr.disabled)
			perf_event_enable(bp);
 
		return err;
	}
 
end:
	bp->attr.disabled = attr->disabled;
 
	return 0;
}
```

为了看 perf_event 上面的断点怎么工作的，需要从 3.1.10/kernel/events/hw_breakpoint.c 中的 init_hw_breakpoint 函数看起：

```
int __init init_hw_breakpoint(void)
{
	unsigned int **task_bp_pinned;
	int cpu, err_cpu;
	int i;
 
	for (i = 0; i < TYPE_MAX; i++)
		nr_slots[i] = hw_breakpoint_slots(i);
 
	for_each_possible_cpu(cpu) {
 
	}
 
	constraints_initialized = 1;
 
	perf_pmu_register(&perf_breakpoint, "breakpoint", PERF_TYPE_BREAKPOINT);
 
	return register_die_notifier(&hw_breakpoint_exceptions_nb);
 
 err_alloc:
}
```

这里注册了 perf_breakpoint 的操作，来看全局变量 perf_breakpoint 的定义：

```
static struct pmu perf_breakpoint = {
	.task_ctx_nr	= perf_sw_context, /* could eventually get its own */
 
	.event_init	= hw_breakpoint_event_init,
	.add		= hw_breakpoint_add,
	.del		= hw_breakpoint_del,
	.start		= hw_breakpoint_start,
	.stop		= hw_breakpoint_stop,
	.read		= hw_breakpoint_pmu_read,
};
```

所以硬件断点的添加，会进入 hw_breakpoint_add 函数，而 hw_breakpoint_add 函数只是简单调用架构相关的函数 arch_install_hw_breakpoint 来实现，在 ARM，该函数在 3.1.10/arch/arm/kernel/hw_breakpoint.c 中：

```
static int hw_breakpoint_add(struct perf_event *bp, int flags)
{
	if (!(flags & PERF_EF_START))
		bp->hw.state = PERF_HES_STOPPED;
 
	return arch_install_hw_breakpoint(bp);
}
 
/*
 * Install a perf counter breakpoint.
 */
int arch_install_hw_breakpoint(struct perf_event *bp)
{
	struct arch_hw_breakpoint *info = counter_arch_bp(bp);
	struct perf_event **slot, **slots;
	int i, max_slots, ctrl_base, val_base, ret = 0;
	u32 addr, ctrl;
 
	/* Ensure that we are in monitor mode and halting mode is disabled. */
	ret = enable_monitor_mode();
	if (ret)
		goto out;
 
	addr = info->address;
	ctrl = encode_ctrl_reg(info->ctrl) | 0x1;
 
	if (info->ctrl.type == ARM_BREAKPOINT_EXECUTE) {
		/* Breakpoint */
		ctrl_base = ARM_BASE_BCR;
		val_base = ARM_BASE_BVR;
		slots = (struct perf_event **)__get_cpu_var(bp_on_reg);
		max_slots = core_num_brps;
		if (info->step_ctrl.enabled) {
			/* Override the breakpoint data with the step data. */
			addr = info->trigger & ~0x3;
			ctrl = encode_ctrl_reg(info->step_ctrl);
		}
	} else {
		/* Watchpoint */
		if (info->step_ctrl.enabled) {
			/* Install into the reserved breakpoint region. */
			ctrl_base = ARM_BASE_BCR + core_num_brps;
			val_base = ARM_BASE_BVR + core_num_brps;
			/* Override the watchpoint data with the step data. */
			addr = info->trigger & ~0x3;
			ctrl = encode_ctrl_reg(info->step_ctrl);
		} else {
			ctrl_base = ARM_BASE_WCR;
			val_base = ARM_BASE_WVR;
		}
		slots = (struct perf_event **)__get_cpu_var(wp_on_reg);
		max_slots = core_num_wrps;
	}
 
	for (i = 0; i < max_slots; ++i) {
		slot = &slots[i];
 
		if (!*slot) {
			*slot = bp;
			break;
		}
	}
 
	if (WARN_ONCE(i == max_slots, "Can't find any breakpoint slot\n")) {
		ret = -EBUSY;
		goto out;
	}
 
	/* Setup the address register. */
	write_wb_reg(val_base + i, addr);
 
	/* Setup the control register. */
	write_wb_reg(ctrl_base + i, ctrl);
 
out:
	return ret;
}
```

arch_install_hw_breakpoint 先对信息组合 (encode_ctrl_reg)，再调用 write_wb_reg 分别将断点地址信息和断点控制信息写入到 DEBUG 寄存器。

write_wb_reg 是对 CP14 操作的一系列宏组成的，各宏的引用如下，最终是调用 ARM_DBG_WRITE 宏来写入值，即通过汇编操作协处理器 CP14 写入到 DEBUG 寄存器中。

```
static void write_wb_reg(int n, u32 val)
{
	switch (n) {
	GEN_WRITE_WB_REG_CASES(ARM_OP2_BVR, val);
	GEN_WRITE_WB_REG_CASES(ARM_OP2_BCR, val);
	GEN_WRITE_WB_REG_CASES(ARM_OP2_WVR, val);
	GEN_WRITE_WB_REG_CASES(ARM_OP2_WCR, val);
	default:
		pr_warning("attempt to write to unknown breakpoint "
				"register %d\n", n);
	}
	isb();
}
 
#define GEN_WRITE_WB_REG_CASES(OP2, VAL)	\
	WRITE_WB_REG_CASE(OP2, 0, VAL);		\
	WRITE_WB_REG_CASE(OP2, 1, VAL);		\
	WRITE_WB_REG_CASE(OP2, 2, VAL);		\
	WRITE_WB_REG_CASE(OP2, 3, VAL);		\
	WRITE_WB_REG_CASE(OP2, 4, VAL);		\
	WRITE_WB_REG_CASE(OP2, 5, VAL);		\
	WRITE_WB_REG_CASE(OP2, 6, VAL);		\
	WRITE_WB_REG_CASE(OP2, 7, VAL);		\
	WRITE_WB_REG_CASE(OP2, 8, VAL);		\
	WRITE_WB_REG_CASE(OP2, 9, VAL);		\
	WRITE_WB_REG_CASE(OP2, 10, VAL);	\
	WRITE_WB_REG_CASE(OP2, 11, VAL);	\
	WRITE_WB_REG_CASE(OP2, 12, VAL);	\
	WRITE_WB_REG_CASE(OP2, 13, VAL);	\
	WRITE_WB_REG_CASE(OP2, 14, VAL);	\
	WRITE_WB_REG_CASE(OP2, 15, VAL)
 
#define WRITE_WB_REG_CASE(OP2, M, VAL)		\
	case ((OP2 << 4) + M):			\
		ARM_DBG_WRITE(c ## M, OP2, VAL);\
		break
 
#define ARM_DBG_WRITE(M, OP2, VAL) do {\
	asm volatile("mcr p14, 0, %0, c0," #M ", " #OP2 : : "r" (VAL));\
} while (0)
```

写入到 DEBUG 寄存器后，所下的断点信息就生效了。

在 x86 中，通过操作 DR 寄存器下断点。

在 mips 中，也有专用的 DEBUG 寄存器。

未完，后面再来补充。