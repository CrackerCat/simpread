> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293405.htm#msg_header_h2_9)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> 上一篇讲的是怎么把代码塞进目标进程。  
> 这一篇讲的是更底层的问题：**环境本身是脏的。**
> 
> （系列第二篇 · 内核态伪装）  
> 第一篇：frida 进场 2.5 秒就被杀，我写了套零线程注入  
> 第三篇：加固不让你抓包？那就到内核层动手

企业级加固真正开始干活的时候发现另一件事：**这个环境本身，一眼就能看出来被动过。**

*   `/proc/cmdline` 里写着 `androidboot.verifiedbootstate=orange`（已解锁）
*   `uname -r` 返回 `4.14.268-Marisa-KSU`（自编内核的后缀）
*   `/data/local/tmp/` 里躺着提权工具
*   `/proc` 里可能还有调试进程的名字

加固根本不需要检测 frida。它只要读一眼这些，就知道你不是善茬。

先看看市面上主流方案留下了什么：

**Magisk**

*   su 文件
*   magiskd 守护进程
*   一堆 overlay 挂载

**KernelSU**

*   ksud 守护进程
*   特征内核版本号

**两者共同**

*   boot 分区被改 → `verifiedbootstate` 变成 orange

加固查什么？查文件、查进程表、查挂载、读 `/proc/cmdline`。

它的检测本质是**对比「应然」和「实然」**：

*   系统应该在锁定状态
*   应该没有 root 身份的进程
*   应该没有奇怪的映射

常规方案是改用户态。改完之后，跟你 "应该有的样子" 对不上，就露馅。

加固查的每一样东西 —— `/proc/cmdline`、进程表、maps —— 数据全部由**内核**产出。

而内核嘛…… 我可以自己编一个。

那么：

```
加固:  "看看 cmdline 是不是 orange（已解锁）"
内核:  "green"（骗你的）

加固:  "数数进程表里有没有 root 守护进程"
内核:  "（过滤掉了）没有"

加固:  "那自杀吧"
内核:  "（信号拦了）杀不动"
```

**没有任何一个用户态文件需要藏，因为谎言是在数据出厂前就编好的。**

代价是每次系统更新、换机型都要重新编译。

纪律也很重要：

*   **只改现役内核的源码**（历史版本不碰）
*   **每次动内核前先对指纹**：`sha256sum /dev/block/by-name/boot` 跟本地产物对上才动手
*   **救砖镜像永远在手边**

Linux 里 "是不是 root" 不是天赋，是一张证件。

每个进程在内核里都挂着一份 `struct cred`，记着 uid、gid 和 capability 集。

**所谓提权，就是把这张证件换成全 0 的。**

常规方案（su）是在用户态放一个 setuid 0 的文件，谁执行谁换证。

我的做法：**在内核里开一个后门命令，进程一发这个命令，内核亲手给它换证。**

不能新增 syscall —— 那会在 syscall 表上留一行新记录，一扫就看见。

所以借用一个已有的 `prctl`，在里面加一个私有 option：

```
case 0x4d52:                    /* 私有命令号 */
    if (!uid_eq(current_uid(), KUIDT_INIT(2000)))
        return -EPERM;          /* 只许 adb shell 调用 */
    error = __sc_swap();        /* 内核亲手换证 */
    break;
```

`__sc_swap()` 里做的事很朴素：

```
c = prepare_creds();               /* 复制当前凭证 */
c->uid = c->euid = c->suid = c->fsuid = KUIDT_INIT(0);   /* 身份归零 */
c->gid = c->egid = c->sgid = c->fsgid = KGIDT_INIT(0);
c->cap_inheritable = CAP_FULL_SET; /* 能力集拉满 */
c->cap_permitted   = CAP_FULL_SET;
c->cap_effective   = CAP_FULL_SET;
c->security = init_cred.security;  /* SELinux 凭证借用 init 的 */
commit_creds(c);                   /* 提交 */
```

`uid 2000` 就是 adb shell。

只有从电脑插线进来的 shell 能提权，**App 哪怕反编译出来知道了 `0x4d52` 这个魔数，也调不动**（直接 EPERM）。

而提权工具本身就是 `/data/local/tmp/` 下一个普通二进制文件。

谁都能删，删了随时再推。

**它本身不含任何权限，权限完全在内核里。**

<table><thead><tr><th>检测手段</th><th>结果</th></tr></thead><tbody><tr><td>找 su 文件 / setuid 程序</td><td>没有（提权工具是普通文件）</td></tr><tr><td>查 root 守护进程</td><td>没有（没有常驻提权服务）</td></tr><tr><td>对比 syscall 表</td><td>没有新增项（prctl 是老函数）</td></tr><tr><td>查已加载模块 / kallsyms</td><td><code>__always_inline</code> 让函数根本不生成符号</td></tr></tbody></table>

最后这条值得说一下：`__always_inline` 让 `__sc_swap` 不产生 kallsyms 符号。

`/proc/kallsyms` 里查不到，逆向也对不上号。

换了 uid 0 的证件，只是过了 Unix 层。

Android 还有第二套保险：**SELinux**。

就算你是 root，策略说你这个 domain 不能读那个文件，一样拒绝。

常规做法是 `setenforce 0` —— 全局关掉。

**这一下就露馅了。**

加固读一下 `getenforce`，返回不是 Enforcing，直接判定环境异常。

SELinux 的所有权限检查最终都汇聚到一个入口函数 `avc_has_perm_noaudit`。

补丁只在这里加三行：

```
if (unlikely(current_cred()->securebits & 0x80000000U))
    return 0;              /* 带标记的进程：直接放行 */
```

`securebits` 是 cred 里一个平时没人用的字段，征用一个 bit 当暗号。

配合上面换证时打的标记。

*   `getenforce` 返回 **Enforcing**（引擎确实还开着，别的进程照常被管）
*   `/sys/fs/selinux/enforce` 是 1
*   只有被我换过证、带标记的进程，检查时静默通过
*   AVC 审计日志里不会出现 "root 进程被拒绝" 这种异常记录

**这大概是整个框架里我最喜欢的一处：不破坏系统的任何可观测状态，只给自己开小灶。**

对比 `setenforce 0` 那种 "把保安室炸了" 的做法，高下立判。

剩下的就是逐个补检测面。

每个补丁的格式都一样：**谁在问 → 问哪个接口 → 答真话还是假话。**

加固读 `/proc/cmdline` 看 `androidboot.verifiedbootstate`。刷过机的是 `orange`。

补丁在内核启动早期把这个字符串**等长替换**成 `green`。

等长很重要 —— 长度变了会破坏后面参数的解析。

```
p = strstr(saved_command_line, "androidboot.verifiedbootstate=");
/* "orange" -> "green"，memset 补空格保持长度不变 */
memset(v, ' ', 6);
memcpy(v, "green", 5);
```

加固遍历 `/proc/*/comm` 找名字。

补丁在 `proc_pid_readdir()` 里跳过命中的进程 —— 那个 pid 直接不出现在列表里。

```
if (sc_hide_task(iter.task))
    continue;
```

加固读 `/proc/self/maps` 找 `frida-agent-*.so`、`/memfd:`。

这个补丁我翻过车，而且是两次，值得单独讲 —— 见下一节。

加固轮询 `/proc/self/status` 的 `TracerPid`，非 0 说明正被调试。

补丁在 `task_state()` 里：

```
if (tracer && current_uid().val < 10000)
    tpid = task_pid_nr_ns(tracer, ns);
```

意思是：**uid ≥ 10000（App 世界）读时永远返回 0；uid < 10000（自己人）读时返回真值。**

这个双视图设计很关键 —— **藏东西的人必须自己看得见藏哪了**，不然没法调试。

这是最狠的一招：检测到异常直接 `kill` 自己，让你连排查机会都没有。

补丁拦在 `sys_kill` / `sys_tgkill` / `sys_tkill` 三个入口：

```
if (sc_sigguard(tgid, sig))
    return 0;      /* 假装发送成功，实际不投递 */
```

效果：加固照常检测、照常判定异常、照常发起自杀 —— **但进程永远死不掉**。

我们还能继续观察它。

加固遍历 `/data/local/tmp` 找提权工具、boot 镜像、frida 相关文件名。

补丁在 `filldir()` —— 所有目录列表输出的必经之路。

一个补丁管住 `ls`、`opendir`、`find` 所有姿势。

`uname -r` 会显示 `4.14.268-Marisa-KSU` 这种自编后缀，一眼看出刷过机。

补丁按调用者 comm 匹配，命中的进程看到的版本变成 `3.10.0-lowmem`（一个古老低调的版本号）。

这个补丁也翻过车：v1 只匹配了主进程名，结果注入实际发生在 helper 子进程里，helper 的 uname 没被伪装，照样露馅。

教训：**伪装面要覆盖 "所有会报信的进程"，漏一个就白干。**

回到 maps 那个补丁。

思路很直接：命中就整行不输出 —— 假装这段映射不存在。

```
if (sc_hit(vma))
    return;     /* 什么都不打印 */
```

**结果：agent 挂死，CPU 100%。**

想了一下才明白：**agent 自己也要读自己的 maps 来定位注入段。**

它读不到自己的映射，就陷入忙等。

这告诉我们一件事：

> 视图伪造不能 "一刀切地删"，因为观察者不止敌人一个。

地址、权限、偏移全部照常输出，**只把路径字符串换成无辜的系统库**：

```
if (sc_fake) {
    seq_puts(m, "/system/lib64/libsecmedia-64.so\n");
    return;
}
```

加固看到的是 "一段普通的媒体库映射"。

agent 看到自己的映射范围完整无损。

**所有观察者都满意。**

这套框架里有五条思维模型，我觉得比代码本身更有价值：

**① 改报告，不改事实**

进程确实存在、文件确实存在、信号确实发了 —— 但 `/proc` 说不存在、`readdir` 说没有、`kill` 返回 0。

内核是所有用户态信息的唯一上游。**在上游撒谎，下游全体被骗。**

**② 哈希代替明文**

隐藏清单不存 `".cch"` 这样的字符串，全部存 FNV-1a 哈希。

为什么？

如果表里写的是明文，它会出现在内核的 `.rodata` 段里。任何人 dump 内核镜像、`strings` 一下，**你的隐藏清单就成了自供状。**

存哈希后，镜像里只有十几个四字节的 "数字"，没有特征可搜。

**③ inline 掉符号**

`__always_inline` 让补丁函数不产生 kallsyms 符号。

`/proc/kallsyms` 查不到，逆向也对不上号。

**④ uid 门控双视图**

自己人（uid < 10000）看真值方便调试，App（uid ≥ 10000）看伪造值。

**藏东西的人必须自己看得见藏哪了。**

**⑤ 幂等补丁脚本**

补丁不是手改源码，而是 Python 脚本对源码做锚点替换：

```
查标记 → 字节模式锚点定位 → assert 命中次数 → 替换 → 校验输出
```

`assert anchor in s` 保证找不到预期代码时报错退出，**绝不 "大概齐" 地插错位置**。

重复执行会 SKIP，不命中会 FAIL 而不是静默写错。

两篇文章合起来才是完整拼图：

```
内核态负责：进入进程之前和之外
            root 通道、环境伪装、进程/maps 伪造、信号拦截

用户态负责：进入进程之后
            零线程注入、符号解析、拿 JNIEnv、hook
```

三个咬合点：

**① TracerPid 补丁兜住 ptrace attach 的毫秒窗口**

上一篇提过，ptrace attach 的瞬间 `TracerPid` 会短暂非零。

这个补丁把它兜住了。

**② maps 伪造思路可以迁移**

注入之后 App 的 lib 目录里会多一个 so 文件。

将来在这个伪造表里加一条哈希，就能让它显示成正常库。

**③ 信号拦截是最后一道保险**

让被检测的进程死不掉。

我们的新方案根本不触发检测，但留着它兜底。

到这一步，环境干净了：  
![](https://bbs.kanxue.com/upload/attach/202611/1067244_9VXBKKWNSJH87DA.webp)

*   `getenforce` 是 Enforcing
*   `uname` 是官方版本号
*   `/proc` 里没有奇怪进程
*   `maps` 里没有可疑映射
*   文件列表里什么都没有

**加固随便查，看到的都是它想看到的。**

但问题又来了：环境干净了、代码也进去了，**为什么包还是抓不到？**

这才是最费劲的部分。第三篇讲。

> **免责声明**
> 
> 本文为安全技术研究与学习交流，涉及的设备均为本人合法持有的测试环境。  
> 文中已对具体应用名称、域名等信息做脱敏处理。  
> 请勿将相关技术用于任何未经授权的系统。  
> 修改内核、绕过安全机制等操作存在设备损坏风险，请勿在主力设备上尝试。  
> 技术本身中立，使用者要为自己的行为负责。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)