> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/hG7nlduMuY8mtTB3Q1BUAw)

**一、Android 反调试 **  

  反调试在代码保护中扮演着非常重要的角色，虽然不能完全阻止攻击者，但是能加大攻击者的分析时间成本。目前绝大多数 Android app 都加固了, 为了防止 App 被调试分析，加固功能中添加了各种反调试功能，比如: 自己 ptrace 自己、检查函数执行时间、读取 /proc/$pid/wchan 或者 /proc/$pid/task/$pid/wchan 文件信息判断调试状态、读取 /proc/$pid/stat 或者 /proc/$pid/task/$pid/stat 文件信息判断调试状态、读取 /proc/$pid/status 或者 /proc/$pid/task/$pid/status 文件检测 TracerPid、读取检测调试进程名、检测调试端口等等。网上关于 Android 反调试的文章比较多，想了解更多 Android 反调试检测手段可以看看 [https://bbs.pediy.com/thread-223324.htm] 中总结的方法。本篇文章我们只讨论读取 /proc/$pid/status 文件和 /proc/$pid/wchan 文件检测 app 是否处于被调试。

**二、读取 / proc/$pid 检测原理分析**

 **1. 读取 /proc/$pid 下的 status 文件检测被调试**

   在 Android 中调试状态下，linux 内核会向 / proc/$pid/status 或者 /proc/$pid/task/$pid/status 中写入进程状态信息。其中 TracerPid 字段写入调试该进程的进程的的 Pid。其中 State 字段中写入该进程当前处于的状态, 取值如下之一:

```
R (running)", "S (sleeping)", "D (disk sleep)", "T (stopped)", "t(tracing stop)", "Z (zombie)",  "X (dead)"

```

当 State 为 t (tracing stop) 或者 T (stopped) 的时候, 表示正被调试追踪。所以反调试的方法之一就是通过不断轮询读取 **/proc/$pid/status 或者 /proc/$pid/task/$pid/status** 文件检测检查 TracerPid 的值或者 State 的值，如果 TracerPid 非 0 说明该进程被调试; 如果 state 值为 t (tracing stop) 或者 T (stopped), 说明被调试追踪。如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433PxibWibqWrQsUe07s2oichr1OPINDbZEd08LN3cyT3z9KzpOPxNN1oibAgBglM3m7UbxPHy6tg4icXNQ/640?wx_fmt=png)

   **2. 读取 /proc/$pid/stat、/proc/$pid/task/$pid/stat 检测被调试**  

    进程被调试状态下内核会向 / proc/$pid/stat、/proc/$pid/task/$pid/stat 文件中的第三个字段写入 t 标识。

        **3. 读取 /proc/$pid/wchan、/proc/$pid/task/$pid/wchan**

          wchan 文件内容表示显示当进程 sleep 时，kernel 当前运行的函数。若进程被调试，内核会往 / proc/$pid/wchan、/proc/$pid/task/$pid/wchan 文件中写入 ptrace_stop 信息。

 **三、修改内核**  

         内核源码中和 "二" 中相关的信息代码位于内核 **/fs/proc** 目录中，如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432HmYJ0nYP8hjyCB7Zics1ql1ZudlbDmeYgh1zO9lKsVVibQXvLcjh0ypGWyEBZUIHOAHz5GicdGibNicg/640?wx_fmt=png)

      ** 1. 修改 State 标识值**  

        该标识写入相关信息位于内核文件 **/fs/proc/array.c** 中，如下所示。

原代码内容:  

```
static const char * const task_state_array[] = {
  "R (running)",    /*   0 */
  "S (sleeping)",    /*   1 */
  "D (disk sleep)",  /*   2 */
  "T (stopped)",    /*   4 */
  "t (tracing stop)",  /*   8 */
  "X (dead)",    /*  16 */
  "Z (zombie)",    /*  32 */
};

```

我们需要把以上内容中的 "t (tracing stop)" 和 "T (stopped)" 替换为 "S (sleeping)", 不管 App 如何被调试，内核写入的都是正常状态来对抗反调试检测。修改之后的代码如下:  

```
static const char * const task_state_array[] = {
  "R (running)",    /*   0 */
  "S (sleeping)",    /*   1 */
  "D (disk sleep)",  /*   2 */
  ///ADD START
  //"T (stopped)",    /*   4 */
  "S (sleeping)",       /*   4 */
  ///ADD END
  /*
  ///ADD START
  //"t (tracing stop)",/*   8 */  
  */
  "S (sleeping)",  /*   8 */
  /*
  ///ADD END
  */
  "X (dead)",    /*  16 */
  "Z (zombie)",    /*  32 */
};

```

2. 修改 TracerPid 相关

          该标识写入相关信息位于内核文件 /fs/proc/array.c 中的 task_state 函数中，如下所示。

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432HmYJ0nYP8hjyCB7Zics1qljBlgC4etS33OE6QQiayuhcYwRqVLDRDDnr4WQVswJXYJB28XTp6gJGQ/640?wx_fmt=png)

其中 tpid 就表示当前正在调试的进程 Pid，我们将 tpid 永远修改为 0。就可以绕过检测 TracerPid。修改之后如下:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432HmYJ0nYP8hjyCB7Zics1qlgJbezyFpdSiawTqAUtntj1UBKA1WPWmpyibvm7DBK6LOGF6fal23Oibag/640?wx_fmt=png)

3. 修改 **wchan 文件**相关的信息

    文件 **/proc/$pid/wchan、/proc/$pid/task/$pid/wchan** 内核写入相关代码路径位于:\fs\proc\base.c 中，如下所示:

```
#ifdef CONFIG_KALLSYMS
/*
 * Provides a wchan file via kallsyms in a proper one-value-per-file format.
 * Returns the resolved symbol.  If that fails, simply return the address.
 */
static int proc_pid_wchan(struct seq_file *m, struct pid_namespace *ns,
        struct pid *pid, struct task_struct *task)
{
  unsigned long wchan;
  char symname[KSYM_NAME_LEN];
  wchan = get_wchan(task);
  if (lookup_symbol_name(wchan, symname) < 0)
    if (!ptrace_may_access(task, PTRACE_MODE_READ_FSCREDS))
      return 0;
    else
      return seq_printf(m, "%lu", wchan);
  else
    return seq_printf(m, "%s", symname);
}
#endif /* CONFIG_KALLSYMS */

```

将以上写入符号名称的地方，修改成过滤符号名称包含 trace 关键字, 然后写入 sys_epoll_wait。修改后如下所示:  

```
#ifdef CONFIG_KALLSYMS
/*
 * Provides a wchan file via kallsyms in a proper one-value-per-file format.
 * Returns the resolved symbol.  If that fails, simply return the address.
 */
static int proc_pid_wchan(struct seq_file *m, struct pid_namespace *ns,
        struct pid *pid, struct task_struct *task)
{
  unsigned long wchan;
  char symname[KSYM_NAME_LEN];
  wchan = get_wchan(task);
  if (lookup_symbol_name(wchan, symname) < 0)
    if (!ptrace_may_access(task, PTRACE_MODE_READ_FSCREDS))
      return 0;
    else
      return seq_printf(m, "%lu", wchan);
  else{
    ///ADD START
    if(strstr(symname,"trace")){
          return seq_printf(m, "%s","sys_epoll_wait");
    }
    ///ADD END
    return seq_printf(m, "%s", symname);
  }
}
#endif /* CONFIG_KALLSYMS */

```

四、编译刷机  

      修改保存之后，在 Android 源码根目录执行如下命令编译刷机包然后刷机:

```
source build/envsetup.sh
brunch oneplus3-userdebug

```

关注公众号，及时获取文章更新:  

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF2ymxCQDEXFGMfmo5RCNCVLStiaTMH4InlrATFZibeOcglQ2dEWOv2VicKVicSYiajjC69qXON5tKBHxpg/640?wx_fmt=jpeg)