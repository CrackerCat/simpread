> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-262039.htm)

> [原创]gdb 的那些奇淫技巧

[原创]gdb 的那些奇淫技巧

发表于: 2020-9-13 19:35 10228

[举报](javascript:void(0);)

### [原创]gdb 的那些奇淫技巧

 [![](http://passport.kanxue.com/upload/avatar/554/844554.png?1584801497)](user-home-844554.htm) [evilpan](user-home-844554.htm) ![](https://bbs.kanxue.com/view/img/rank/12.png) 13  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif) [ 举报](javascript:void(0);) 2020-9-13 19:35  10228

gdb 也用了好几年了，虽然算不上骨灰级玩家，但也有一些自己的经验，因此和大家分享交流下，顺便也作为一个存档记录。

 

目录

*   [多进程调试](#多进程调试)
*            [示例程序](#示例程序)
*            [踩坑过程](#踩坑过程)
*            [最终方案](#最终方案)
*   [多线程调试](#多线程调试)
*   [程序运行](#程序运行)
*            [参数](#参数)
*            [标准输入](#标准输入)
*            [环境变量](#环境变量)
*   [后记](#后记)
*   附录: gdb 命令表
*            启动 GDB
*            [帮助信息](#帮助信息)
*            [断点](#断点)
*            [运行程序](#运行程序)
*            [栈帧](#栈帧)
*            [代码浏览](#代码浏览)
*            [浏览数据](#浏览数据)
*            [文件操作](#文件操作)
*            [信号控制](#信号控制)
*            [线程调试](#线程调试)
*            [进程调试](#进程调试)
*            [汇编调试](#汇编调试)
*            [其他命令](#其他命令)
*   [参考资料](#参考资料)

多进程调试
=====

最近在调试一个漏洞的 exploit 时遇到一个问题。目标漏洞程序是一个 CGI 程序，由主进程调起，而且运行只有一瞬的时间；我的需求是想要在在该程序中下断点，在内存布局之后可以调试我的 shellcode，该如何实现？当然目标程序是没有符号的，而且我希望下的断点是一个动态地址。在 lldb 中有`--wait-for`，gdb 里却没有对应的命令，经过多次摸索，终于总结出一个比较完美的解决方案。

示例程序
----

这里构建一个简单的示例来进行实际演示。首先是父进程:

```
#include #include #include int main(int argc, char **argv, char **env) {
    printf("parent started, pid=%d\n", getpid());
 
    char *line = NULL;
    size_t len = 0;
    ssize_t read;
    while ((read = getline(&line, &len, stdin)) != -1) {
        pid_t pid = fork();
        if (pid == -1) {
            perror("fork");
            break;
        }
 
        if (pid == 0) {
            printf("1 fork return in child, pid=%d\n", getpid());
            char *const av[] = {"child", line, NULL};
            if (-1 == execve("./child", av, env)) {
                perror("execve");
                break;
            }
        } else {
            printf("2 fork return in parent, child pid=%d\n", pid);
            int status = 0;
            wait(&status);
        }
    }
 
    return 0;
} 
```

子进程很简单:

```
#include #include void vuln(char *str) {
    char buf[4];
    strcpy(buf, str);
    printf("child buf: %s", buf);
}
 
int main(int argc, char **argv) {
    puts("child started");
    vuln(argv[1]);
    return 0;
} 
```

这里编译子进程时候指定`-no-pie`，并且`strip`掉符号。我们的调试目标是断点在子进程的`strcpy`中，拓展来说是希望能断点在子进程的任意地址上。

踩坑过程
----

通过搜索可以找到一个 stackoverflow 的回答: [gdb break when entering child process](https://stackoverflow.com/questions/43882101/gdb-break-when-entering-child-process)。根据其说法，使用 `set follow-fork-mode child`即可。这是一个 gdb 命令，其目的是告诉 gdb 在目标应用调用`fork`之后接着调试子进程而不是父进程，因为在 Linux 中`fork`系统调用成功会返回两次，一次在父进程，一次在子进程。我们来试一下，直接断点在 strcpy 符号中:

```
gdb child --pid $parent_pid
(gdb) set follow-fork-mode child
(gdb) b strcpy
Breakpoint 1 at 0x4004c0
(gdb) c
Continuing.
Warning:
Cannot insert breakpoint 1.
Cannot access memory at address 0x4004c0
 
Command aborted.

```

噢，断点都打不上，理由很简单，因为不同进程之间的虚拟地址空间都不一样。

 

另外一个回答中说了，虽然不能断在指定地址，但我们可以`break main`，告诉 gdb 把断点设置在 main 函数。不过我们的子进程是没有符号的，所以`break main`并没有卵用。

 

现在已经有了让 gdb 跟着子进程的方法，只不过问题是无法把断点打到子进程上，因为子进程还没有启动，那么用硬件断点可不可以？

```
gdb child --pid $parent_pid
(gdb) set follow-fork-mode child
(gdb) hb *0x4004c0
Hardware assisted breakpoint 1 at 0x4004c0
(gdb) c
Continuing.
[New process 309]
process 309 is executing new program: /pwn/child
 
Thread 2.1 "child" received signal SIGABRT, Aborted.
[Switching to process 309]

```

可以是可以，但是断点压根没有触发，子进程直接拷贝溢出崩溃了都没有停下来！所以硬件断点在这里并没有用。

 

那么把断点设置在一些起始函数的上呢？根据之前对 ELF 以及动态链接的学习，我们可以断在比如`_start`或者`__libc_start_main`上面:

```
gdb child --pid $parent_pid
(gdb) set follow-fork-mode child
(gdb) b _start
Breakpoint 1 at 0x7fbfb4c30090

```

实际上该断点也不会触发，因为这个地址是是父进程的地址空间。

 

不过到现在答案已经呼之欲出了，总结一下，gdb 支持:

*   fork 之后跟踪到子进程
*   可以设置软断点
*   子进程有 `_start` 符号

所以，就有了一个最终方案。

最终方案
----

我的最终方案如下:

```
set detach-on-fork on
set follow-fork-mode child
set breakpoint pending on
b _start
attach $parent_pid
file child
continue

```

首先告诉 gdb 跟踪子进程；然后设置`set breakpoint pending on`是为了在设置断点时让 gdb 不强制在对符号下断点时就需要固定地址，这样在`b _start`时就会 pending 而不是报错；最后再连接到父进程以及加载子进程的符号。

> `detach-on-fork on`是为了在 fork 之后断开父进程，避免 gdb 退出时把父进程杀死，并不是这节的重点。

 

其中的时序非常重要。如果先 attach 父进程再下断点，那么断点会直接下到父进程空间从而不会触发；如果先读取了子进程的符号再下断点，可能会下在一个错误的虚拟地址上。

 

这也是我用了很久的一个方法，不过后来我知道了有更官方的解决方式:

```
set follow-fork-mode child
catch exec

```

囧，……

 

[Catch Point](https://www.zeuthen.desy.de/dv/documentation/unixguide/infohtml/gdb/Set-Catchpoints.html) 真是个好东西，支持很多有用的事件:

*   常规的 C++ 异常事件
*   系统调用事件 (可直接指定系统调用号)
*   动态库的加载 / 卸载事件
*   exec/fork/vfork
*   ...

看来文档搜索能力还有待提高啊。……

多线程调试
=====

在调试大型程序的时候，经常会遇到这么一个问题，即涉及到的线程很多，少则十几个多则上百个线程。在这些线程之间穿梭也是一个常见的困难。

 

首先最基本的是线程的切换命令:

*   `info threads`: 查看当前所有的线程
*   `thread n`: 切换到 id 为`n`的线程中

> 对于进程也有类似的命令`info inferiors`/`inferior n`，在调试多进程交互的程序时会经常用到。

 

其次，在对某个线程进行单步调试时，会遇到 CPU 的迷之调度，突然一个`next`或者`nexti`就跑到其他线程去了，这个时候有个特殊的参数`scheduler-locking`可以解决这个问题:

```
(gdb) help set scheduler-locking
Set mode for locking scheduler during execution.
off    == no locking (threads may preempt at any time)
on     == full locking (no thread except the current thread may run)
          This applies to both normal execution and replay mode.
step   == scheduler locked during stepping commands (step, next, stepi, nexti).
          In this mode, other threads may run during other commands.
          This applies to both normal execution and replay mode.
replay == scheduler locked in replay mode and unlocked during normal execution.

```

通常设置为`step`模式可解决单步调试的问题。

程序运行
====

我经常用到的一个功能是需要使用 gdb 执行某个程序，并且能精确控制程序的参数，包括命令行、标准输入和环境变量等。gdb 的 run 命令就是用来执行程序的。

 

这里还是先写个示例测试程序:

```
// demo.c
#include #include int main(int argc, char **argv) {
    int n;
    char buf[10];
    for (int i = 0; i < argc; i++)
        printf("argv[%d] = %s\n", i, argv[i]);
 
    int nread;
    nread = read(STDIN_FILENO, buf, 10);
    printf("first read: %d\n", nread);
 
    nread = read(STDIN_FILENO, buf, 10);
    printf("second read: %d\n", nread);
    return 0;
} 
```

参数
--

最基本的，通过 run 命令控制命令行参数:

```
$ gdb demo
(gdb) run hello world
Starting program: /pwn/demo hello world
argv[0] = /pwn/demo
argv[1] = hello
argv[2] = world

```

或者在运行前设置`args`参数:

```
(gdb) set args hello world
(gdb) run
Starting program: /pwn/demo hello world
argv[0] = /pwn/demo
argv[1] = hello
argv[2] = world

```

标准输入
----

在漏洞挖掘或者 CTF 比赛中经常遇到的情况是某些输入触发了进程崩溃，因此要挂 gdb 进行分析，这时候就需要 gdb 挂载的程序能够以指定的标准输入运行。如果标准输入是文件，那很简单:

```
$ gdb demo
(gdb) run 
```

但更多时候为了方便调试，希望能以其他程序的输出来运行，比如:

```
$ python -c 'print "A"*100' | ./demo

```

可惜 gdb 不支持这种管道，不过可以通过下面的方法实现:

```
$ gdb demo
(gdb) run < <(python -c 'print "A"*100')
Starting program: /pwn/demo < <(python -c 'print "A"*100')
argv[0] = /pwn/demo
first read: 10
second read: 10

```

或者:

```
$ gdb demo
(gdb) run <<<$(python -c 'print "A"*100')
Starting program: /pwn/demo < <(python -c 'print "A"*100')
argv[0] = /pwn/demo
first read: 10
second read: 10

```

后者实际上是 shell 命令 `here string`的一种形式。这两种方式是有区别的，注意示例程序中 read 调用会提前返回，所以如果我们想要第一次读取 3 个字符，第二次读取 4 个字符的话，就不能一次性全部输入。比如下面这样就不符合预期了:

```
$ gdb demo
(gdb) run < <(echo -n 1112222)
Starting program: /pwn/demo < <(echo -n 1112222)
argv[0] = /pwn/demo
first read: 7
second read: 0

```

正确的方式应该是这样:

```
$ gdb demo
(gdb) run < <(echo -n 111; sleep 1; echo -n 2222)
Starting program: /pwn/demo < <(echo -n 111; sleep 1; echo -n 2222)
argv[0] = /pwn/demo
first read: 3
second read: 4

```

值得注意的是，这种情况下，使用`here string`是没用的，因为该字符串是计算完再一次性传给命令:

```
(gdb) run <<<$(echo -n 111; sleep 1; echo -n 2222)
Starting program: /pwn/demo <<<$(echo -n 111; sleep 1; echo -n 2222)
argv[0] = /pwn/demo
first read: 8
second read: 0

```

而且这里是 8 字节，因为末尾还带了个回车。 所以我更偏向于使用第一种方式。

环境变量
----

对于运行程序而言，还有个重要的参数来源是环境变量，比如在调试 CGI 程序的时候。这在 gdb 中可以使用`environment`参数，不过需要注意的是该参数的设置是以空格为切分而不是传统的以`=`对环境变量赋值。

```
(gdb) help set environment
Set environment variable value to give the program.
Arguments are VAR VALUE where VAR is variable name and VALUE is value.
VALUES of environment variables are uninterpreted strings.
This does not affect the program until the next "run" command.

```

还有要注意的是这个参数要求变量是`uninterpreted strings`，也就是说只能指定可打印字符。如果我们要传输一个的 payload 或者 shellcode 还要用 gdb 调试怎么办呢？我一般使用的方式是在调用 gdb 时指定，比如:

```
$ env CONTENT_TYPE="$(python -c "print 'A'*10 + '\x04\x03\x02\x01'")" gdb demo
(gdb) run

```

后记
==

对于二进制研究人员来说，gdb 是一个锋利的好工具，支持 X86、ARM、MIPS、RISCV、Xtensa 等各种常用和不常用的系统架构，对其熟练使用有时候可以达到事半功倍的效果，在文末的附录中我也列举了一些比较常用的命令。由于 gdb 本身支持 python 接口，因此现实中使用通常结合一些拓展使用，比如:

*   gef: https://github.com/hugsy/gef
*   pwndbg: https://github.com/pwndbg/pwndbg
*   peda: https://github.com/longld/peda

这几个我都用过，各有千秋。现在工作中使用更多的是`gef`，因为安装太方便了，一个文件搞定。

 

上面这几个拓展可能大家可能都不陌生，但还有另外一个我比较常用的是 [gdb-dashboard](https://github.com/cyrus-and/gdb-dashboard)，其功能更为简单，而且使用的是 gdb 原本的信息，所以支持的指令集更多。比如下面的截图就是我曾经用 **gdb + OpenOCD** 来调试 `ESP32`固件的示例:

 

![](https://bbs.kanxue.com/upload/attach/202009/844554_4ZUW8PW2P7AY26V.png)

 

ESP32 是比较少见的`Xtensa`指令集架构，上面的拓展都不支持，不过 gdb 本身支持，因此配合使用的效果绝佳。

附录: gdb 命令表
===========

gdb 还有其他一些小技巧，可以参考 [awesome-cheatsheets/tools/gdb.txt](https://github.com/pannzh/awesome-cheatsheets/blob/master/tools/gdb.txt) 中的列表。该列表最初由韦神创建，我时不时也会添加一些上去。当然为了方便大家的查阅，这里直接给出汇总表格附录:

启动 GDB
------

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>gdb object</code></td><td>正常启动，加载可执行</td><td></td></tr><tr><td><code>gdb object core</code></td><td>对可执行 + core 文件进行调试</td><td></td></tr><tr><td><code>gdb object pid</code></td><td>对正在执行的进程进行调试</td><td></td></tr><tr><td><code>gdb</code></td><td>正常启动，启动后需要 file 命令手动加载</td><td></td></tr><tr><td><code>gdb -tui</code></td><td>启用 gdb 的文本界面（或 ctrl-x ctrl-a 更换 CLI/TUI）</td></tr></tbody></table>

帮助信息
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>help</code></td><td>列出命令分类</td><td></td></tr><tr><td><code>help running</code></td><td>查看某个类别的帮助信息</td><td></td></tr><tr><td><code>help run</code></td><td>查看命令 run 的帮助</td><td></td></tr><tr><td><code>help info</code></td><td>列出查看程序运行状态相关的命令</td><td></td></tr><tr><td><code>help info line</code></td><td>列出具体的一个运行状态命令的帮助</td><td></td></tr><tr><td><code>help show</code></td><td>列出 GDB 状态相关的命令</td><td></td></tr><tr><td><code>help show commands</code></td><td>列出 show 命令的帮助</td></tr></tbody></table>

断点
--

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>break main</code></td><td>对函数 main 设置一个断点，可简写为 b main</td><td></td></tr><tr><td><code>break 101</code></td><td>对源代码的行号设置断点，可简写为 b 101</td><td></td></tr><tr><td><code>break basic.c:101</code></td><td>对源代码和行号设置断点</td><td></td></tr><tr><td><code>break basic.c:foo</code></td><td>对源代码和函数名设置断点</td><td></td></tr><tr><td><code>break *0x00400448</code></td><td>对内存地址 0x00400448 设置断点</td><td></td></tr><tr><td><code>info breakpoints</code></td><td>列出当前的所有断点信息，可简写为 info break</td><td></td></tr><tr><td><code>delete 1</code></td><td>按编号删除一个断点</td><td></td></tr><tr><td><code>delete</code></td><td>删除所有断点</td><td></td></tr><tr><td><code>clear</code></td><td>删除在当前行的断点</td><td></td></tr><tr><td><code>clear function</code></td><td>删除函数断点</td><td></td></tr><tr><td><code>clear line</code></td><td>删除行号断点</td><td></td></tr><tr><td><code>clear basic.c:101</code></td><td>删除文件名和行号的断点</td><td></td></tr><tr><td><code>clear basic.c:main</code></td><td>删除文件名和函数名的断点</td><td></td></tr><tr><td><code>clear *0x00400448</code></td><td>删除内存地址的断点</td><td></td></tr><tr><td><code>disable 2</code></td><td>禁用某断点，但是部删除</td><td></td></tr><tr><td><code>enable 2</code></td><td>允许某个之前被禁用的断点，让它生效</td><td></td></tr><tr><td><code>rbreak {regexpr}</code></td><td>匹配正则的函数前断点，如 <code>ex_*</code> 将断点 ex_ 开头的函数</td><td></td></tr><tr><td><code>tbreak function/line</code></td><td>临时断点</td><td></td></tr><tr><td><code>hbreak function/line</code></td><td>硬件断点</td><td></td></tr><tr><td><code>ignore {id} {count}</code></td><td>忽略某断点 N-1 次</td><td></td></tr><tr><td><code>condition {id} {expr}</code></td><td>条件断点，只有在条件生效时才发生</td><td></td></tr><tr><td><code>condition 2 i == 20</code></td><td>2 号断点只有在 i == 20 条件为真时才生效</td><td></td></tr><tr><td><code>watch {expr}</code></td><td>对变量设置监视点</td><td></td></tr><tr><td><code>info watchpoints</code></td><td>显示所有观察点</td><td></td></tr><tr><td><code>catch exec</code></td><td>断点在 exec 事件，即子进程的入口地址</td></tr></tbody></table>

运行程序
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>run</code></td><td>运行程序</td><td></td></tr><tr><td><code>run {args}</code></td><td>以某参数运行程序</td><td></td></tr><tr><td><code>run &lt; file</code></td><td>以某文件为标准输入运行程序</td><td></td></tr><tr><td><code>run &lt; &lt;(cmd)</code></td><td>以某命令的输出作为标准输入运行程序</td><td></td></tr><tr><td><code>run &lt;&lt;&lt; $(cmd)</code></td><td>以某命令的输出作为标准输入运行程序</td><td>Here-String</td></tr><tr><td><code>set args {args} ...</code></td><td>设置运行的参数</td><td></td></tr><tr><td><code>show args</code></td><td>显示当前的运行参数</td><td></td></tr><tr><td><code>cont</code></td><td>继续运行，可简写为 c</td><td></td></tr><tr><td><code>step</code></td><td>单步进入，碰到函数会进去</td><td></td></tr><tr><td><code>step {count}</code></td><td>单步多少次</td><td></td></tr><tr><td><code>next</code></td><td>单步跳过，碰到函数不会进入</td><td></td></tr><tr><td><code>next {count}</code></td><td>单步多少次</td><td></td></tr><tr><td><code>CTRL+C</code></td><td>发送 SIGINT 信号，中止当前运行的程序</td><td></td></tr><tr><td><code>attach {process-id}</code></td><td>链接上当前正在运行的进程，开始调试</td><td></td></tr><tr><td><code>detach</code></td><td>断开进程链接</td><td></td></tr><tr><td><code>finish</code></td><td>结束当前函数的运行</td><td></td></tr><tr><td><code>until</code></td><td>持续执行直到代码行号大于当前行号（跳出循环）</td><td></td></tr><tr><td><code>until {line}</code></td><td>持续执行直到执行到某行</td><td></td></tr><tr><td><code>kill</code></td><td>杀死当前运行的函数</td></tr></tbody></table>

栈帧
--

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>bt</code></td><td>打印 backtrace</td><td></td></tr><tr><td><code>frame</code></td><td>显示当前运行的栈帧</td><td></td></tr><tr><td><code>up</code></td><td>向上移动栈帧（向着 main 函数）</td><td></td></tr><tr><td><code>down</code></td><td>向下移动栈帧（远离 main 函数）</td><td></td></tr><tr><td><code>info locals</code></td><td>打印帧内的相关变量</td><td></td></tr><tr><td><code>info args</code></td><td>打印函数的参数</td></tr></tbody></table>

代码浏览
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>list 101</code></td><td>显示第 101 行周围 10 行代码</td><td></td></tr><tr><td><code>list 1,10</code></td><td>显示 1 到 10 行代码</td><td></td></tr><tr><td><code>list main</code></td><td>显示函数周围代码</td><td></td></tr><tr><td><code>list basic.c:main</code></td><td>显示另外一个源代码文件的函数周围代码</td><td></td></tr><tr><td><code>list -</code></td><td>重复之前 10 行代码</td><td></td></tr><tr><td><code>list *0x22e4</code></td><td>显示特定地址的代码</td><td></td></tr><tr><td><code>cd dir</code></td><td>切换当前目录</td><td></td></tr><tr><td><code>pwd</code></td><td>显示当前目录</td><td></td></tr><tr><td><code>search {regexpr}</code></td><td>向前进行正则搜索</td><td></td></tr><tr><td><code>reverse-search {regexp}</code></td><td>向后进行正则搜索</td><td></td></tr><tr><td><code>dir {dirname}</code></td><td>增加源代码搜索路径</td><td></td></tr><tr><td><code>dir</code></td><td>复位源代码搜索路径（清空）</td><td></td></tr><tr><td><code>show directories</code></td><td>显示源代码路径</td></tr></tbody></table>

浏览数据
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>print {expression}</code></td><td>打印表达式，并且增加到打印历史</td><td></td></tr><tr><td><code>print /x {expression}</code></td><td>十六进制输出，print 可以简写为 p</td><td></td></tr><tr><td><code>print array[i]@count</code></td><td>打印数组范围</td><td></td></tr><tr><td><code>print $</code></td><td>打印之前的变量</td><td></td></tr><tr><td><code>print *$-&gt;next</code></td><td>打印 list</td><td></td></tr><tr><td><code>print $1</code></td><td>输出打印历史里第一条</td><td></td></tr><tr><td><code>print ::gx</code></td><td>将变量可视范围（scope）设置为全局</td><td></td></tr><tr><td><code>print 'basic.c'::gx</code></td><td>打印某源代码里的全局变量，(gdb 4.6)</td><td></td></tr><tr><td><code>print /x &amp;main</code></td><td>打印函数地址</td><td></td></tr><tr><td><code>x *0x11223344</code></td><td>显示给定地址的内存数据</td><td></td></tr><tr><td><code>x /nfu {address}</code></td><td>打印内存数据，n 是多少个，f 是格式，u 是单位大小</td><td></td></tr><tr><td><code>x /10xb *0x11223344</code></td><td>按十六进制打印内存地址 0x11223344 处的十个字节</td><td></td></tr><tr><td><code>x/x &amp;gx</code></td><td>按十六进制打印变量 gx，x 和斜杆后参数可以连写</td><td></td></tr><tr><td><code>x/4wx &amp;main</code></td><td>按十六进制打印位于 main 函数开头的四个 long</td><td></td></tr><tr><td><code>x/gf &amp;gd1</code></td><td>打印 double 类型</td><td></td></tr><tr><td><code>help x</code></td><td>查看关于 x 命令的帮助</td><td></td></tr><tr><td><code>info locals</code></td><td>打印本地局部变量</td><td></td></tr><tr><td><code>info functions {regexp}</code></td><td>打印函数名称</td><td></td></tr><tr><td><code>info variables {regexp}</code></td><td>打印全局变量名称</td><td></td></tr><tr><td><code>ptype name</code></td><td>查看类型定义，比如 ptype FILE，查看 FILE 结构体定义</td><td></td></tr><tr><td><code>whatis {expression}</code></td><td>查看表达式的类型</td><td></td></tr><tr><td><code>set var = {expression}</code></td><td>变量赋值</td><td></td></tr><tr><td><code>display {expression}</code></td><td>在单步指令后查看某表达式的值</td><td></td></tr><tr><td><code>undisplay</code></td><td>删除单步后对某些值的监控</td><td></td></tr><tr><td><code>info display</code></td><td>显示监视的表达式</td><td></td></tr><tr><td><code>show values</code></td><td>查看记录到打印历史中的变量的值 (gdb 4.0)</td><td></td></tr><tr><td><code>info history</code></td><td>查看打印历史的帮助 (gdb 3.5)</td></tr></tbody></table>

文件操作
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>file {object}</code></td><td>加载新的可执行文件供调试</td><td></td></tr><tr><td><code>file</code></td><td>放弃可执行和符号表信息</td><td></td></tr><tr><td><code>symbol-file {object}</code></td><td>仅加载符号表</td><td></td></tr><tr><td><code>exec-file {object}</code></td><td>指定用于调试的可执行文件（非符号表）</td><td></td></tr><tr><td><code>core-file {core}</code></td><td>加载 core 用于分析</td></tr></tbody></table>

信号控制
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>info signals</code></td><td>打印信号设置</td><td></td></tr><tr><td><code>handle {signo} {actions}</code></td><td>设置信号的调试行为</td><td></td></tr><tr><td><code>handle INT print</code></td><td>信号发生时打印信息</td><td></td></tr><tr><td><code>handle INT noprint</code></td><td>信号发生时不打印信息</td><td></td></tr><tr><td><code>handle INT stop</code></td><td>信号发生时中止被调试程序</td><td></td></tr><tr><td><code>handle INT nostop</code></td><td>信号发生时不中止被调试程序</td><td></td></tr><tr><td><code>handle INT pass</code></td><td>调试器接获信号，不让程序知道</td><td></td></tr><tr><td><code>handle INT nopass</code></td><td>调试起不接获信号</td><td></td></tr><tr><td><code>signal signo</code></td><td>继续并将信号转移给程序</td><td></td></tr><tr><td><code>signal 0</code></td><td>继续但不把信号给程序</td></tr></tbody></table>

线程调试
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>info threads</code></td><td>查看当前线程和 id</td><td></td></tr><tr><td><code>thread {id}</code></td><td>切换当前调试线程为指定 id 的线程</td><td></td></tr><tr><td><code>break {line} thread all</code></td><td>所有线程在指定行号处设置断点</td><td></td></tr><tr><td><code>thread apply {id..} cmd</code></td><td>指定多个线程共同执行 gdb 命令</td><td></td></tr><tr><td><code>thread apply all cmd</code></td><td>所有线程共同执行 gdb 命令</td><td></td></tr><tr><td><code>set schedule-locking ?</code></td><td>调试一个线程时，其他线程是否执行</td><td></td></tr><tr><td><code>set non-stop on/off</code></td><td>调试一个线程时，其他线程是否运行</td><td></td></tr><tr><td><code>set pagination on/off</code></td><td>调试一个线程时，分页是否停止</td><td></td></tr><tr><td><code>set target-async on/off</code></td><td>同步或者异步调试，是否等待线程中止的信息</td></tr></tbody></table>

进程调试
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>info inferiors</code></td><td>查看当前进程和 id</td><td></td></tr><tr><td><code>inferior {id}</code></td><td>切换某个进程</td><td></td></tr><tr><td><code>kill inferior {id...}</code></td><td>杀死某个进程</td><td></td></tr><tr><td><code>set detach-on-fork on/off</code></td><td>设置当进程调用 fork 时 gdb 是否同时调试父子进程</td><td></td></tr><tr><td><code>set follow-fork-mode parent/child</code></td><td>设置当进程调用 fork 时是否进入子进程</td></tr></tbody></table>

汇编调试
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>info registers</code></td><td>打印普通寄存器</td><td></td></tr><tr><td><code>info all-registers</code></td><td>打印所有寄存器</td><td></td></tr><tr><td><code>print/x $pc</code></td><td>打印单个寄存器</td><td></td></tr><tr><td><code>stepi</code></td><td>指令级别单步进入</td><td>si</td></tr><tr><td><code>nexti</code></td><td>指令级别单步跳过</td><td>ni</td></tr><tr><td><code>display/i $pc</code></td><td>监控寄存器（每条单步完以后会自动打印值）</td><td></td></tr><tr><td><code>x/x &amp;gx</code></td><td>十六进制打印变量</td><td></td></tr><tr><td><code>info line 22</code></td><td>打印行号为 22 的内存地址信息</td><td></td></tr><tr><td><code>info line *0x2c4e</code></td><td>打印给定内存地址对应的源代码和行号信息</td><td></td></tr><tr><td><code>disassemble {addr}</code></td><td>对地址进行反汇编，比如 disassemble 0x2c4e</td></tr></tbody></table>

其他命令
----

<table><thead><tr><th>命令</th><th>含义</th><th>备注</th></tr></thead><tbody><tr><td><code>show commands</code></td><td>显示历史命令 (gdb 4.0)</td><td></td></tr><tr><td><code>info editing</code></td><td>显示历史命令 (gdb 3.5)</td><td></td></tr><tr><td><code>ESC-CTRL-J</code></td><td>切换到 Vi 命令行编辑模式</td><td></td></tr><tr><td><code>set history expansion on</code></td><td>允许类 c-shell 的历史</td><td></td></tr><tr><td><code>break class::member</code></td><td>在类成员处设置断点</td><td></td></tr><tr><td><code>list class:member</code></td><td>显示类成员代码</td><td></td></tr><tr><td><code>ptype class</code></td><td>查看类包含的成员</td><td>/o 可以看成员偏移，类似 pahole</td></tr><tr><td><code>print *this</code></td><td>查看 this 指针</td><td></td></tr><tr><td><code>define command ... end</code></td><td>定义用户命令</td><td></td></tr><tr><td><code>&lt;return&gt;</code></td><td>直接按回车执行上一条指令</td><td></td></tr><tr><td><code>shell {command} [args]</code></td><td>执行 shell 命令</td><td></td></tr><tr><td><code>source {file}</code></td><td>从文件加载 gdb 命令</td><td></td></tr><tr><td><code>quit</code></td><td>退出 gdb</td></tr></tbody></table>

参考资料
====

*   [Debugging with GDB](https://www.zeuthen.desy.de/dv/documentation/unixguide/infohtml/gdb/)
*   [awesome-cheatsheet](https://github.com/pannzh/awesome-cheatsheet)

* * *

 

**欢迎交流**:  
blog: [https://evilpan.com](https://evilpan.com)  
公众号: 有价值炮灰  
![](https://bbs.kanxue.com/upload/attach/202009/844554_8X3ZRHR669XE2AF.jpg)

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#系统底层](forum-4-1-2.htm) [#调试逆向](forum-4-1-1.htm)