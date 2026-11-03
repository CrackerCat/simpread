> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293406.htm#msg_header_h2_11)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> 一台小米 9，一款加固过的 App，六个星期。  
> 这篇讲的是：当所有常规工具都失效之后，我是怎么把代码塞进目标进程的。
> 
> （系列第一篇 · 用户态注入）  
> 第二篇：为了不让加固发现，我给自己编了个内核  
> 第三篇：加固不让你抓包？那就到内核层动手

先看抓包效果

事情的开始很普通：我想看看一个加固 App 的网络请求。

按惯例上 frida。结果进程秒死。

一开始以为是姿势问题，于是把能想到的全试了一遍：

*   纯 attach，什么都不做
*   hook 一个 libc 函数
*   hook 五个 libc 函数
*   空跑一次 `Java.perform`
*   Stalker 单线程
*   Stalker 全线程

六种，全部死。**存活时间从 0.4 秒到 1.2 秒，没有一个活过 2.5 秒。**

对照组很重要：不注入的时候，这个 App 稳定存活 63 秒以上。

所以这不是 "自然退出"，是被杀了。

死法也一样：`SIGKILL`，干净利落，连个 signal handler 都不给你留。

真正让我开窍的，是把六次实验的数据摆在一起看。

六种手法干的事情完全不同，但它们有一个唯一的共同产物：

**每次 attach，目标进程的线程数都精确 +5。**

```
纯 attach:    82 → 88   (+5)
hook 1 个:    78 → 83   (+5)
Java.perform: 75 → 80   (+5)
Stalker:      …… 全部 +5
```

这不是巧合。

frida-agent 是一个完整的运行时，进场必然带五条线程：JS 引擎、消息循环、心跳、GC、反调试。

而加固方在做什么？

**它在轮询 `/proc/self/task/` 数线程数，周期不到一秒。**

真相一下子清楚了：

1.  它根本不在乎你 hook 了什么、用了什么 API
2.  它只数线程
3.  所以换名字、改字符串、魔改 server —— **全部无效**

最后这点也被验证过：我找了两个 "魔改版" 的 frida-server，改包名改符号，结果死在同一个 2.5 秒。

> 这个结论不是猜出来的，是**对照实验**逼出来的。  
> 每次只改一个变量，其余全部固定。  
> 遇到 "不知道对方怎么检测的"，第一件事就是设计对照组。

知道它在数线程之后，有两条路：

**A. 用 frida，想办法把线程藏起来。**

**B. 不用 frida，写一个零线程的东西。**

A 走不通 —— 你不能既要求一个运行时常驻，又要求它没有线程。

只能走 B。

CPU 执行代码需要一条执行流：取指令 → 执行 → 下一条。

**线程是执行流的载体，但载体不是非得自己造。**

只要让 "别人的执行流" 路过你的代码，你的代码就跑起来了。

整个方案一共两次借流：

*   **第一次**：注入时，constructor 在 App 自己的线程里跑
*   **第二次**：hook 后，App 调用被 hook 的方法时，它自己的流流进我们的函数

不是它不懂这个技巧，是它的产品形态要求**常驻**：

"电脑上敲一行 JS、手机立刻生效" = 进程里必须随时有东西在监听消息 = 需要一条永远醒着的执行流 = 必须有自己的线程。

我的取舍很明确：**放弃常驻监听，换零线程。**

constructor 是一次性的，跑完就把线程还回去。想再来一次？从外面再 ptrace 敲一下。

ptrace 是 Linux 的调试接口，gdb 就是靠它做的。对我们最有用的三个能力：

*   `PTRACE_ATTACH`：让目标站住
*   `PTRACE_POKEDATA / PEEKDATA`：读写它的任意内存
*   `PTRACE_GETREGS / SETREGS`：读写它的寄存器，包括 PC

**关键：attach、改内存、改寄存器，全是 "外部动作"，目标进程里不增加任何东西。**

具体怎么 "调用 dlopen"：

1.  attach 后目标停住，读出当前寄存器组（保存备用）
2.  找到 dlopen 的真实地址 = 所在 so 的加载基址 + 函数偏移
    *   基址从 `/proc/<pid>/maps` 查
    *   偏移从 so 文件的符号表查
3.  把寄存器摆成调用姿势（ARM64 约定）：
    *   `x0` = so 路径字符串的地址（提前写进去）
    *   `x1` = `RTLD_NOW`
    *   `pc` = dlopen 地址
4.  让目标恢复执行 —— **它自己 "心甘情愿" 地调用了 dlopen**
5.  等它跑完，读 `x0` 拿到 so 的 handle
6.  恢复原寄存器，detach

整个过程目标进程只知道 "卡了一下"，没有任何新线程、新进程、新 fd 留下。

这一步有个反直觉的事实。

同一个 so，换路径加载，结果天差地别：

```
/data/app/<pkg>-xxx==/lib/arm64/    → 成功（App 自己的 lib 目录）
/data/local/tmp/                    → 失败，返回 NULL
/data/data/<pkg>/                   → 失败
memfd + /proc/self/fd/N             → 失败
```

一开始我以为是加固做了路径黑名单。

后来做了个跨进程对照：把同一个 so 注入一个**完全没有加固**的系统进程，一样失败。

那就不是加固干的。

真相是 Android 10 之后的 **linker namespace**：每个进程有个 "允许从哪里加载 so" 的白名单，App 进程的白名单里只有系统 so 目录 + 它自己的 `/data/app/.../lib/arm64/`。

**这是 AOSP 的系统级设计，跟加固无关。**

判定方法很简单：**跨进程对照**。找个没加固的进程试同样的事，如果它也失败，就不是加固干的。

所以操作流程是：把 so `cp` 到 App 自己的 lib 目录（root 下可写），再 dlopen。

能加载了，但还有个坑。

第一版 so 是用 glibc 交叉编译器编的，依赖清单长这样：

```
DT_NEEDED: ['libc.so.6', 'ld-linux-aarch64.so.1']
```

Android 的 libc 叫 `libc.so`，链接器叫 `linker64`。

这两个名字在 Android 上**根本不存在**。

所以它在任何路径、任何进程下都必然加载失败。

ELF 的 `.dynamic` 段里有张依赖清单，linker 加载时会逐项去找。

**清单为空 = 没有任何依赖要解析 = 绕开整个依赖查找。**

编译命令（本机 LLVM clang，不用 NDK）：

```
clang --target=aarch64-linux-android21 -fPIC -shared -nostdlib \
  -ffreestanding -fno-builtin -fno-stack-protector \
  -mbranch-protection=none -Wl,--build-id=none \
  -O1 -o libzthook.so zthook.c
```

其中 `-ffreestanding -fno-builtin` 这两个不能省，原因见下面。

没有这两个参数时，clang 会做一件很聪明的事：

**把 "长得像 memcpy 的循环" 优化成对 `memcpy` 的调用。**

问题是我用了 `-nostdlib`，没人提供 memcpy，依赖清单又是空的。

结果：linker 解析失败 → dlopen 返回 NULL → **日志里一个字都看不到**。

排查了很久，最后靠反汇编才看到那个 `bl memcpy`。

从此养成一个习惯：**每次编译完必须复查动态符号表。**

```
llvm-readobj --dyn-syms libzthook.so
# 只应该有 index 0 那个空表项
# 出现任何 Undefined 符号 = 这个 so 必然加载失败
```

不能调 libc，就自己发系统调用。ARM64 上是 `svc #0`：

```
static long sys3(long n, long a, long b, long c)
{
    register long x8 __asm__("x8") = n;
    register long x0 __asm__("x0") = a;
    register long x1 __asm__("x1") = b;
    register long x2 __asm__("x2") = c;
    __asm__ __volatile__("svc #0"
        : "+r"(x0) : "r"(x8), "r"(x1), "r"(x2) : "memory", "cc");
    return x0;
}
```

常用的号记一下：openat=56、close=57、read=63、write=64、mmap=222、getpid=172。

这一条差点让我误判 "constructor 没执行"。

**App 进程的 stdout/stderr 被重定向到 `/dev/null`。**

你往 fd 1/2 写东西，logcat 里什么都看不到。

验证代码有没有跑，唯一可靠的办法：

**用 openat 在 App 自己的 data 目录建文件，写进去，从外部 cat 出来。**

```
long fd = sys4(56 /*openat*/, -100 /*AT_FDCWD*/,
    "/data/data/<pkg>/hit.log", O_WRONLY|O_CREAT|O_APPEND, 0644);
```

constructor 会在这个 so 被 dlopen 时，**在发起调用的那个线程里同步执行**。

也就是说：只要让 App 的线程调了 dlopen，我们的代码就自动在 App 的线程里跑起来了 —— 不需要、也没有机会开新线程。

第一次跑通的时候，日志是这样的：

```
注入前线程 = 63
RESULT = dlopen_ok
注入后线程 = 63          ← 零增长
HIT pid=10057 tid=10057  ← constructor 确实在目标进程跑过
```

对照 frida 的 "+5 线程 → 1 秒被杀"，这一步就是**能活和必死的分水岭**。

linker 对已加载的 so 有缓存。

**同一个路径第二次 dlopen，直接返回旧 handle，constructor 不再执行。**

而我一路用 `cp` 覆盖同名文件 —— inode 不变 —— 从第二次起就静默不执行了。

于是伪造出两种假象：

*   "我改的代码怎么没生效"（其实压根没跑）
*   "以前能用的 so 突然不灵了"（其实它早就被注入过了）

**铁律：每次测试换新文件名。** libv001、libv002、libv003……

到这一步，我们已经在目标进程里跑起了一段 C 代码。

但它还只是个 "哑巴"—— 能读写文件、发系统调用，摸不到 Java 世界。

想摸 Java，需要 JNIEnv。

`dlsym` 是 libdl 的导出函数，我们的 freestanding so 没法直接调。

所以自己实现一遍符号查找。分三步：

**① 从 `/proc/self/maps` 找模块基址**

流式读取（目标进程 maps 有 2737 行、超过 256KB，一次读进固定 buffer 会被截断），逐行解析，路径以 `libart.so` 结尾的所有行里取最小起始地址。

**② 解析 ELF 头，定位动态段**

遍历程序头找 `PT_DYNAMIC`，读出一串 `{tag, value}`。

这里有个致命的坑：`DT_SYMTAB` / `DT_STRTAB` 的 value 是**链接期虚拟地址**，运行时必须加模块基址才是真地址。

漏加 = 解引用 `0x2a8` 这种野指针 = 秒崩。

**③ GNU hash 查符号**

现代 Android so 只导出 GNU hash 表。结构是：

```
[桶数] [布隆大小] [布隆位移]
[布隆过滤器 words]
[桶 × N]  → 存符号索引
[链 ...]
```

查找流程：算 hash → 布隆过滤（两个 bit 都是 1 才可能命中）→ 查桶拿候选索引 → 走链逐个比对。

**这里有个只花了半小时但很隐蔽的坑：hash 值必须是 32 位。**

我一开始用 `u64` 承载，结果 `h/64`、`h%64`、`h>>bloom_shift` 全部算错，符号永远查不到。

真机上自己算出来的地址，我用 Python 独立解析同一份 `libart.so` 做对照：

```
本地解析:     JNI_GetCreatedJavaVMs = 0x3726e8
运行时基址:   0x78e8cd1000
两者相加:     0x78e90436e8
so 内自算:    0x78e90436e8   ← 完全一致
```

**养成习惯：任何 "自己算的地址"，都要找第二种独立方法交叉验证。**

```
JNI_GetCreatedJavaVMs(&vm, 1, &n)      → 拿到 JavaVM*
JavaVM->GetEnv(&env, JNI_VERSION_1_6)  → 拿到 JNIEnv*
JNIEnv->FindClass / CallStaticMethod   → 能调任何 Java 方法
```

有意思的是，这三个 "函数" 的调用方式全是 **C 函数指针表**：

```
GetVersion           → index 4
FindClass            → index 6
CallStaticIntMethod  → index 129
```

所以 "调 JNI" 就是：拿函数表基地址 + 偏移，直接 call。

不需要头文件，不需要链接任何库。

验证的时候我耍了个小心眼：用 **Java 的 `Process.myPid()`** 和 **native 的 `getpid()`** 各查一次。

```
Java_Process.myPid() = 0x2749
native_getpid()      = 0x2749   ← 两个独立来源一致
```

一致，才算真的同时站在两个世界里。

诚实地说，这套方案不是 "全隐身"，是 "避开最强检测点"。

<table><thead><tr><th>观察维度</th><th>留下的痕迹</th><th>会不会被发现</th></tr></thead><tbody><tr><td>线程数</td><td>+0</td><td>看不见 ← 活命的根本</td></tr><tr><td>maps 里多了 so 映射</td><td>+1 个内存段</td><td>能看见（但实测不杀）</td></tr><tr><td>TracerPid（attach 毫秒窗口内）</td><td>短暂非零</td><td>能撞见</td></tr><tr><td>lib 目录多一个 so 文件</td><td>一个文件</td><td>能看见</td></tr></tbody></table>

结论：**它能看见我一小块，但它当前的策略只看线程，所以没杀我。**

要对抗更严的检测，就得往下走到内核层 —— 那是下一篇文章的事。

**整个注入链路**：

```
ptrace attach        → 目标停住
摆寄存器调 dlopen    → so 被加载
constructor 同步执行 → 代码在 App 线程里跑
自己解析 ELF         → 拿到 libart 里的函数
JNI 三级跳           → 摸到 Java 世界
```

**七个坑**：

<table><thead><tr><th>#</th><th>坑</th><th>现象</th><th>解法</th></tr></thead><tbody><tr><td>1</td><td>glibc 版 so</td><td>DT_NEEDED 里有 libc.so.6</td><td>换 <code>-nostdlib</code></td></tr><tr><td>2</td><td>clang 优化成 memcpy</td><td>dlopen 返回 0，无日志</td><td><code>-ffreestanding -fno-builtin</code></td></tr><tr><td>3</td><td>maps 太大被截断</td><td>明明有的模块找不到</td><td>流式扫描</td></tr><tr><td>4</td><td>DT_SYMTAB 没加基址</td><td>解引用野指针秒崩</td><td>加模块基址</td></tr><tr><td>5</td><td>GNU hash 用了 64 位</td><td>符号永远查不到</td><td>全部用 u32</td></tr><tr><td>6</td><td>同路径 dlopen 不重跑</td><td>"代码没生效" 的假象</td><td>每次换文件名</td></tr><tr><td>7</td><td>信 logcat</td><td>以为没执行</td><td>写文件到 data 目录</td></tr></tbody></table>

这一篇讲的是**怎么进去**。

但进去之后你会发现另一个问题：**环境本身是脏的。**

刷过机的设备，`/proc/cmdline` 里写着 `orange`，`uname` 里带着自编内核后缀，`/proc` 里躺着提权工具。

加固只要读一眼就知道你不是善茬 —— 它甚至不需要检测 frida。

所以第二篇讲的是：**我干脆自己编了个内核，把所有 "报告" 都改成它想看到的样子。**

> **免责声明**
> 
> 本文为安全技术研究与学习交流，涉及的设备和应用均为本人合法持有的测试环境。  
> 文中已对具体应用名称、域名、密钥等信息做脱敏处理。  
> 请勿将相关技术用于任何未经授权的系统。  
> 未经授权对他人系统进行渗透、抓包、逆向，可能违反《网络安全法》《数据安全法》  
> 《个人信息保护法》等法律法规。技术本身中立，使用者要为自己的行为负责。

[冰与火的战歌：Windows 内核攻防实战高级班！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)