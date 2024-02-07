> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38384263/article/details/132290737?spm=1001.2014.3001.5502)

#### KGDB 原理分析及远程挂载调试 ARM64 内核

*   [为什么使用 KGDB](#KGDB_1)
*   [KGDB 原理分析](#KGDB_5)
*   *   [简介](#_7)
    *   [调试原理](#_32)
    *   [部分 RSP 命令的解析](#RSP_207)
    *   *   [`m/M` 命令](#mM_209)
        *   [`Z/z` 命令](#Zz_213)
        *   [`s` 命令](#s_241)
*   [如何使用 KGDB](#KGDB_366)
*   *   [内核启用 KGDB](#KGDB_368)
    *   [修改内核启动参数](#_390)
    *   [KGDB 与 Server 的远程连接](#KGDBServer_394)
    *   *   [示例](#_423)
*   [Linux 内核 GDB Script 命令集简介](#LinuxGDB_Script_441)

为什么使用 KGDB
----------

我们在开发或者是调试驱动过程中，往往会通过串口的打印日志来分析驱动运行逻辑，定位问题。但是这种方法需要我们增加大量的打印日志来实现，有时候日志加的地方不对，可能打印不出来，就要重新加，编译驱动烧录，这个过程是很痛苦的。那有没有一种方法，可以实现驱动的[断点调试](https://so.csdn.net/so/search?q=%E6%96%AD%E7%82%B9%E8%B0%83%E8%AF%95&spm=1001.2101.3001.7020)呢？答案是有的，可以使用 KGDB 在内核态进行源码级别的调试。

KGDB 原理分析
---------

### 简介

kgdb 是内核提供的一种内核调试程序，kgdb 将自身代码插入在内核中，编译后，与内核一起作为一个[整体的](https://so.csdn.net/so/search?q=%E6%95%B4%E4%BD%93%E7%9A%84&spm=1001.2101.3001.7020)程序运行。所以 kgdb 本身运行在内核空间，因此 kgdb 可以访问内核空间或者用户空间，获取相应的被调试程序的数据信息。在 2.6.25 以后的内核版本，kgdb 已经被整合到内核中，是内核源码的一部分，要开启 kgdb 调试，只需要添加对应的 kgdb 配置选项即可。

kgdb 调试的连接通信过程和 gdbserver 类似，需要远程服务器的 gdb 配合。target 和 host 之间通过串口或网络连接。target 运行配置了 kgdb 的内核，host 运行 gdb，中间使用 RSP 协议进行通信。gdb 向 kgdb 发送 RSP 命令包，kgdb 解析命令包，从而实现 target 的内核调试。调试模型见下图：

kgdb+gdb  
调试模型 HOST TARGET RSP 协议 串口或网络 GDB KGDB 硬件 硬件

### 调试原理

kgdb 是一种 stub 调试程序，它是 Linux 内核的一部分，会和内核一起编译运行。那么 Linux 内核运行的好好的，kgdb 如何打断 Linux 内核获取 CPU 控制权呢？在获取到 CPU 控制权，进入到 kgdb 源码后又会做一些什么事情？

kgdb 其实是在异常通知链 `(die_notifier)`中注册自己的回调函数 `kgdb_notify`，在 step_hook 以及 break_hook 里注册钩子函数，捕获 Linux 内核产生的异常，实现对 Linux 的控制。注册代码如下所示：

```
//arm64的注册oops异常代码
static struct notifier_block kgdb_notifier = {
	.notifier_call	= kgdb_notify,
	/*
	 * Want to be lowest priority
	 */
	.priority	= -INT_MAX,
};
register_die_notifier(&kgdb_notifier);

static struct break_hook kgdb_brkpt_hook = {
	.esr_mask	= 0xffffffff,
	.esr_val	= (u32)ESR_ELx_VAL_BRK64(KGDB_DYN_DBG_BRK_IMM),
	.fn		= kgdb_brk_fn
};

static struct break_hook kgdb_compiled_brkpt_hook = {
	.esr_mask	= 0xffffffff,
	.esr_val	= (u32)ESR_ELx_VAL_BRK64(KGDB_COMPILED_DBG_BRK_IMM),
	.fn		= kgdb_compiled_brk_fn
};

static struct step_hook kgdb_step_hook = {
	.fn		= kgdb_step_brk_fn
};

register_break_hook(&kgdb_brkpt_hook);
register_break_hook(&kgdb_compiled_brkpt_hook);
register_step_hook(&kgdb_step_hook);


```

可看出 arm64，是把 `kgdb_notify`注册到了最低优先级的位置，内核发生 oops 异常后，会先处理正常的异常处理函数，完成后才会执行 kgdb 异常处理函数，kgdb 响应完成后，会接着正常往下运行，等待下次异常触发。

由于 kgdb 捕获了所有的 Linux 内核异常，所以在启用 kgdb 的情况下，内核崩溃后，会进入 kgdb，等待开发者收集异常信息。其工作流程如下所示：

与 HOST GDB 通信 RSP 协议处理过程 g G m M z Z ... c s 向 Host 发送当前异常的线程信息 kgdb_serial_stub 读  
C  
P  
U  
寄  
存  
器 解析 RSP 写  
C  
P  
U  
寄  
存  
器 读  
内  
核  
态  
内  
存  
地  
址 写  
内  
核  
态  
内  
存  
地  
址 移  
除  
断  
点 设  
置  
断  
点 其  
他  
.  
.  
. 继  
续  
运  
行 单  
步  
调  
试 将信息回复 HOST GDB 接收 gdb 命令包 kgdb_arch_handle_exception c s 解析 RSP 包的 PC 地址 取消 DBG_MDSCR_SS 设置 DBG_MDSCR_SS 获取 CPU 控制权 KGDB 获取 CPU 控制权 KGDB 获取 CPU 控制权 KGDB 获取 CPU 控制权 oops 遍历通知回调函数 CPU oops 异常处理函数 notify_die 进入 KGDB 异常回调函数 kgdb_notify kgdb_handle_exception brk 遍历 brk_hook CPU brk brk_hook 进入 KGDB break hook step 遍历 step_hook CPU step step_hook 进入 KGDB step hook 触发方式 kdgbwait sysrq -g 内核挂死 断点 单步调试 退出 KGDB 等待下次异常触发

可看出，kgdb 共有 5 种触发方式，其中 3 种是首次触发 KGDB 的方式：

1.  内核挂死后自动触发
2.  内核正常运行时，使用 `echo g > /proc/sysrq-trigger`手动触发。这里其实就是使用 `sysrq`执行了一个 `bkpt`的汇编指令，触发了 CPU 异常，从而进入 kgdb。
3.  在启动参数中加入 `kgdbwait`字段，可以在内核启动时触发。

另外的断点和单步调试触发方式是进入到 kgdb 后，打断掉或者单步调试触发的。

其工作机制决定了使用 kgdb 调试的过程是被动调试：即 Host 的 GDB 不能主动暂停内核的运行，只有 target 自己主动触发异常，Host 的 GDB 才可以通过 kgdb 接管内核。

### 部分 RSP 命令的解析

#### `m/M`命令

m/M 命令读 / 写内存的命令，kgdb 收到该命令后，会解析协议包中的要读 / 写的地址：`addr`和数量：`count`，最终会调用到 `probe_kernel_read()/probe_kernel_write()`函数。这两个函数其实是在内核态访问内存地址的函数。所以 kgdb 当前只能返回内核态的内存数据给 HOST GDB，对于硬件寄存器的数据，就没法返回了，不过可以根据地址分布去拓展这个指令。

#### `Z/z`命令

`Z/z`命令是设置 / 移除断点的命令，kgdb 收到该命令后，会解析协议包中的要设置 / 移除断点的 PC 地址：`addr`。随后会调用 `kgdb_arch_set_breakpoint()/kgdb_arch_remove_breakpoint()`函数设置 / 移除断点。函数原型如下所示：

```
int kgdb_arch_set_breakpoint(struct kgdb_bkpt *bpt)
{
	int err;

	BUILD_BUG_ON(AARCH64_INSN_SIZE != BREAK_INSTR_SIZE);

	err = aarch64_insn_read((void *)bpt->bpt_addr, (u32 *)bpt->saved_instr);
	if (err)
		return err;

	return aarch64_insn_write((void *)bpt->bpt_addr,
			(u32)AARCH64_BREAK_KGDB_DYN_DBG);
}

int kgdb_arch_remove_breakpoint(struct kgdb_bkpt *bpt)
{
	return aarch64_insn_write((void *)bpt->bpt_addr,
			*(u32 *)bpt->saved_instr);
}

```

可看出，设置断点其实就是在 `addr`处添加 `AARCH64_BREAK_KGDB_DYN_DBG`，即 `bkpt`异常。当程序走到该地址时，就会触发 break 异常，进入到 break 的钩子函数,，从而 kgdb 获得控制权，继续和 HOST GDB 通信。在设置断点时，kgdb 会把 `addr`处的值取出存起来，移除断点就是将先前 `addr`的值恢复。

#### `s`命令

`s`命令用于响应 HOST GDB 的 s/n 指令，即 GDB 单步调试的步入和步出指令。最终会调用到平台相关的 `kgdb_arch_handle_exception()`函数中。当 HOST GDB 执行步入指令时，HOST GDB 发送的 RSP 协议包中的 `addr`部分会是 0。当 HOST GDB 执行步出指令时，HOST GDB 发送的 RSP 协议包中的 `addr`部分会是步出后的程序地址。kgdb 会根据 addr 的值来判断是步入还是步出，具体函数如下：

```
static void kgdb_arch_update_addr(struct pt_regs *regs,
				char *remcom_in_buffer)
{
	unsigned long addr;
	char *ptr;

	ptr = &remcom_in_buffer[1];
	if (kgdb_hex2long(&ptr, &addr))
		kgdb_arch_set_pc(regs, addr);
	else if (compiled_break == 1)
		kgdb_arch_set_pc(regs, regs->pc + 4);

	compiled_break = 0;
}
void kgdb_arch_set_pc(struct pt_regs *regs, unsigned long pc)
{
	regs->pc = pc;
}

```

可看出，当解出的 addr 为 0 时，kgdb 会将 `pc+4`的值存到 `regs->pc`中。当 addr 不为 0 时 kgdb 会将 `addr`的值存到 `regs->pc`中。以 arm64 平台为例，之后会执行 `kernel_enable_single_step()`函数对处的地址加上 `DBG_MDSCR_SS`(step 异常)。当走到对应的程序地址处便会触发 step 异常，kgdb 获得控制权，继续与 HOST KGDB 通信。

```
void kernel_enable_single_step(struct pt_regs *regs)
{
	WARN_ON(!irqs_disabled());
	set_regs_spsr_ss(regs);
	mdscr_write(mdscr_read() | DBG_MDSCR_SS);
	enable_debug_monitors(DBG_ACTIVE_EL1);
}

```

这里需要注意一下，触发 step 异常时，某些 ARM64 平台的 CPU 会一直挂死在 `el1_irq`里无法退出，无法继续调试，原因未知。具体代码如下：

```
el1_irq:
	kernel_entry 1
	enable_da_f
#ifdef CONFIG_TRACE_IRQFLAGS
	bl	trace_hardirqs_off
#endif

	irq_handler

#ifdef CONFIG_PREEMPT
	ldr	x24, [tsk, #TSK_TI_PREEMPT]	// get preempt count
	cbnz	x24, 1f				// preempt count != 0
	bl	el1_preempt
1:
#endif
#ifdef CONFIG_TRACE_IRQFLAGS
	bl	trace_hardirqs_on
#endif
	kernel_exit 1

```

为了解决这个问题，我在内核社区找到了一个补丁，不过该补丁未合进内核，补丁的作用是在 kgdb 单步调试时禁用中断，等单步调试完成后恢复中断。具体修改如下：

```
int kgdb_arch_handle_exception(int exception_vector, int signo,
			       int err_code, char *remcom_in_buffer,
			       char *remcom_out_buffer,
			       struct pt_regs *linux_regs)
{
	int err;

	switch (remcom_in_buffer[0]) {
	//前面的代码省略，只看修改内容...........
	case 's':
		/* 单步调试时禁用中断 */
		__this_cpu_write(kgdb_pstate, linux_regs->pstate);
		linux_regs->pstate |= PSR_I_BIT;

		/*
		 * Update step address value with address passed
		 * with step packet.
		 * On debug exception return PC is copied to ELR
		 * So just update PC.
		 * If no step address is passed, resume from the address
		 * pointed by PC. Do not update PC
		 */
		kgdb_arch_update_addr(linux_regs, remcom_in_buffer);
		atomic_set(&kgdb_cpu_doing_single_step, raw_smp_processor_id());
		kgdb_single_step =  1;

		/*
		 * Enable single step handling
		 */
		if (!kernel_active_single_step())
			kernel_enable_single_step(linux_regs);
		err = 0;
		break;
	default:
		err = -1;
	}
	return err;
}

static int kgdb_step_brk_fn(struct pt_regs *regs, unsigned int esr)
{
	unsigned int pstate;

	if (user_mode(regs) || !kgdb_single_step)
		return DBG_HOOK_ERROR;

	if (kernel_active_single_step())
		kernel_disable_single_step();

	/* 单步调试后恢复中断 */
	pstate = __this_cpu_read(kgdb_pstate);
	if (pstate & PSR_I_BIT)
		regs->pstate |= PSR_I_BIT;
	else
		regs->pstate &= ~PSR_I_BIT;

	kgdb_handle_exception(0, SIGTRAP, 0, regs);
	return DBG_HOOK_HANDLED;
}

```

如何使用 KGDB
---------

### 内核启用 KGDB

在 2.6.25 以后的内核版本，kgdb 已经被整合到内核中，是内核源码的一部分，开启 KGDB 只需要启用如下内核配置选项：

```
CONFIG_KGDB=y
CONFIG_KGDB_KDB=y
CONFIG_DEBUG_INFO=y
CONFIG_GDB_SCRIPTS=y
CONFIG_DEBUG_INFO_DWARF4=y

```

<table><thead><tr><th align="center">config</th><th align="center">desc</th></tr></thead><tbody><tr><td align="center"><code onclick="mdcp.copyCode(event)">CONFIG_KGDB=y</code></td><td align="center">使能 KGDB</td></tr><tr><td align="center"><code onclick="mdcp.copyCode(event)">CONFIG_KGDB_KDB=y</code></td><td align="center">使能 KGDB_KDB</td></tr><tr><td align="center"><code onclick="mdcp.copyCode(event)">CONFIG_DEBUG_INFO=y</code></td><td align="center">生成 debug 信息</td></tr><tr><td align="center"><code onclick="mdcp.copyCode(event)">CONFIG_DEBUG_INFO_DWARF4=y</code></td><td align="center">生成等级 4 的 debug 信息</td></tr><tr><td align="center"><code onclick="mdcp.copyCode(event)">CONFIG_GDB_SCRIPTS=y</code></td><td align="center">生成 linux GDB 调试脚本</td></tr></tbody></table>

重新编译内核，烧入设备。

### 修改内核启动参数

在内核启动参数后面加入 `nokaslr kgdboc=ttyAMA0,115200`。

### KGDB 与 Server 的远程连接

在上文中，我们了解到，KGDB 是使用串口和 HOST 连接的。但是当我们是 PC 连接 Server 开发的话，我们的设备只能通过串口跟我们的 PC 机连接，无法连接至开发的 Server。但是我们 PC 机又无法运行 GDB 连接 target。为了解决这个矛盾，可以在 PC 机上将与板子相连的串口虚拟成 TCP 端口，编译服务器端运行 GDB，远程连接 PC 机虚拟出的 TCP 端口进行调试。具体流程如下：

TARGET SERVER RSP 协议 ETH 串口 KGDB 硬件 GDB 硬件 PC  
串口  
转发  
TCP

#### 示例

设备启动后，进入根文件系统。设备端执行步骤：

1.  关闭看门狗
2.  使用 sysrq 触发异常 `echo g > /proc/sysrq-trigger`
3.  输入 kgdb 等待远程主机调试 `kgdb`

PC 端执行步骤：  
如果是使用服务器开发的话，使用虚拟串口软件将 PC 连接设备的 com 口转发为 tcp 端口，

1.  设置监听端口
2.  设置串口
3.  设置波特率
4.  开启 TCP 转发服务  
    服务器运行 gdb 直接`target remote pc_ip:port`，其中`pc_ip`是自己电脑的本地 ip，`port`是 com 口转 tcp 的端口号

就跟正常使用 gdb 是一样的，功能很强大，当调试驱动时，可以在 server 端 gdb 连接上设备 kgdb 后使用内核调试脚本自带的工具`lx-symbols`将要调试的 ko 符号表链接到 vmlinux 后，进行源码级调试。

Linux 内核 GDB Script 命令集简介
-------------------------

<table><thead><tr><th>命令</th><th>功能</th></tr></thead><tbody><tr><td>lx-cmdline</td><td>显示当前内核的命令行</td></tr><tr><td>lx-cpus</td><td>列出 CPUS 当前状态</td></tr><tr><td>lx-dmesg</td><td>打印内核日志，类似直接在 shell 敲 dmesg</td></tr><tr><td>lx-fdtdump</td><td>dump 设备树到文件</td></tr><tr><td>lx-iomem</td><td>打印当前 iomem 资源</td></tr><tr><td>lx-ioports</td><td>打印当前 ioport 资源</td></tr><tr><td>lx-list-check</td><td>验证列表一致性</td></tr><tr><td>lx-lsmod</td><td>列出当前插入的 ko</td></tr><tr><td>lx-mounts</td><td>显示当前 VFS 的挂载点</td></tr><tr><td>lx-ps</td><td>dump 当前 linux 的 tasks</td></tr><tr><td>lx-symbols</td><td>加载 ko 符号表到当前调试内核</td></tr><tr><td>lx-version</td><td>显示当前内核的版本信息</td></tr><tr><td>lx_current</td><td>显示当前正在运行的 task</td></tr><tr><td>lx_module</td><td>按名称查找模块</td></tr><tr><td>lx_per_cpu</td><td>查看 per cpu 变量</td></tr><tr><td>lx_task_by_pid</td><td>通过 pid 寻找 task 结构体指针</td></tr><tr><td>lx_thread_info</td><td>通过 task 指针查看 thread 信息</td></tr><tr><td>lx_thread_info_by_pid</td><td>通过 pid 来查看 thread 信息</td></tr></tbody></table>