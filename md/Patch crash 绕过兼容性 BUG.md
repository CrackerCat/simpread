> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/WcUtp-zEVdICgjNzJ9ERUQ)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPMpQPD6aKlxibpo80NniaO328oRBCp9hbOMiaSYFAXPeys5VLThJMYp79dQB4TF1bUOEsEL32OV5hTrg/640?wx_fmt=png)

因故要对付古董内核 2.6.18-308.24.1.el5，本文具体内容已过时，但思路是通用的。

crash 有点 Windows 上 livekd 的味道，我猜这东西是受 Solaris 的 scat 启发，就跟 SystemTap 绝对受 Solaris 的 DTrace 启发一样。如果折腾的事涉及内核，SystemTap 和 crash 或许都可以试试，gdb 就不用强调了。

```
$ yum install crash

$ crash /usr/lib/debug/lib/modules/2.6.18-308.24.1.el5/vmlinux /dev/mem
...
WARNING: /usr/lib/debug/lib/modules/2.6.18-308.24.1.el5/vmlinux
         and /proc/version do not match!

WARNING: /proc/version indicates kernel version: 2.6.18-308.24.1.el5

crash: please use the vmlinux file for that kernel version, or try using
       the System.map for that kernel version as an additional argument.


```

这就扯淡了，一面说内核版本不匹配，一面显示出来的内核版本是匹配的。都是用官方 RPM 装的，没有自编译内核。在非常确认内核版本匹配的前提下，只能说 crash 的兼容性有 BUG。

若你的目标系统 crash 可以用 / dev/mem，就不要往下看了，本文与你无关。

```
$ crash /usr/lib/debug/lib/modules/2.6.18-308.24.1.el5/vmlinux /proc/kcore
...
/proc/kcore: read: No such file or directory

crash: /proc/kcore: initialization failed


```

同样扯淡，/proc/kcore 明明存在、可访问，crash 说找不到，BUG 真多。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPMpQPD6aKlxibpo80NniaO328icgKEflTAuNxe5zia7srP7mqnicmHNia6ZSX0Gpe07KZLC0B1gqzIJjM7w/640?wx_fmt=png)

最后只能这样启动 crash:

```
$ crash /usr/lib/debug/lib/modules/2.6.18-308.24.1.el5/vmlinux /boot/System.map-2.6.18-308.24.1.el5
...
  SYSTEM MAP: /boot/System.map-2.6.18-308.24.1.el5
DEBUG KERNEL: /usr/lib/debug/lib/modules/2.6.18-308.24.1.el5/vmlinux (2.6.18-308.24.1.el5)
    DUMPFILE: /dev/crash
...


```

此种方式进入 crash 提示符与是否 Patch 内核函数 devmem_is_allowed() 使之恒返回 1 无关。

```
crash> dis devmem_is_allowed
0xffffffff80083bb8 <devmem_is_allowed>: jmpq   0xffffffff886d4000
0xffffffff80083bbd <devmem_is_allowed+5>:       add    %al,(%rax)
0xffffffff80083bbf <devmem_is_allowed+7>:       jbe    0xffffffff80083c0d <devmem_is_allowed+85>
0xffffffff80083bc1 <devmem_is_allowed+9>:       mov    0x437938(%rip),%r8d        # 0xffffffff804bb500
0xffffffff80083bc8 <devmem_is_allowed+16>:      xor    %esi,%esi
0xffffffff80083bca <devmem_is_allowed+18>:      jmp    0xffffffff80083c08 <devmem_is_allowed+80>
0xffffffff80083bcc <devmem_is_allowed+20>:      movslq %esi,%rax
0xffffffff80083bcf <devmem_is_allowed+23>:      imul   $0x14,%rax,%rcx
0xffffffff80083bd3 <devmem_is_allowed+27>:      cmpl   $0x1,-0x7fb44aec(%rcx)
0xffffffff80083bda <devmem_is_allowed+34>:      jne    0xffffffff80083c06 <devmem_is_allowed+78>
0xffffffff80083bdc <devmem_is_allowed+36>:      mov    -0x7fb44afc(%rcx),%rdx
0xffffffff80083be3 <devmem_is_allowed+43>:      lea    0xfff(%rdx),%rax
0xffffffff80083bea <devmem_is_allowed+50>:      shr    $0xc,%rax
0xffffffff80083bee <devmem_is_allowed+54>:      cmp    %rax,%rdi
0xffffffff80083bf1 <devmem_is_allowed+57>:      jb     0xffffffff80083c06 <devmem_is_allowed+78>
0xffffffff80083bf3 <devmem_is_allowed+59>:      add    -0x7fb44af4(%rcx),%rdx
0xffffffff80083bfa <devmem_is_allowed+66>:      shr    $0xc,%rdx
0xffffffff80083bfe <devmem_is_allowed+70>:      cmp    %rdx,%rdi
0xffffffff80083c01 <devmem_is_allowed+73>:      jae    0xffffffff80083c06 <devmem_is_allowed+78>
0xffffffff80083c03 <devmem_is_allowed+75>:      xor    %eax,%eax
0xffffffff80083c05 <devmem_is_allowed+77>:      retq
0xffffffff80083c06 <devmem_is_allowed+78>:      inc    %esi
0xffffffff80083c08 <devmem_is_allowed+80>:      cmp    %r8d,%esi
0xffffffff80083c0b <devmem_is_allowed+83>:      jl     0xffffffff80083bcc <devmem_is_allowed+20>
0xffffffff80083c0d <devmem_is_allowed+85>:      mov    $0x1,%eax
0xffffffff80083c12 <devmem_is_allowed+90>:      retq

crash> rd -8 devmem_is_allowed 91
ffffffff80083bb8:  e9 43 04 65 08 00 00 76 4c 44 8b 05 38 79 43 00   .C.e...vLD..8yC.
ffffffff80083bc8:  31 f6 eb 3c 48 63 c6 48 6b c8 14 83 b9 14 b5 4b   1..<Hc.Hk......K
ffffffff80083bd8:  80 01 75 2a 48 8b 91 04 b5 4b 80 48 8d 82 ff 0f   ..u*H....K.H....
ffffffff80083be8:  00 00 48 c1 e8 0c 48 39 c7 72 13 48 03 91 0c b5   ..H...H9.r.H....
ffffffff80083bf8:  4b 80 48 c1 ea 0c 48 39 d7 73 03 31 c0 c3 ff c6   K.H...H9.s.1....
ffffffff80083c08:  44 39 c6 7c bf b8 01 00 00 00 c3                  D9.|.......

crash> rd -16 0xffffffff80083c03
ffffffff80083c03:  c031                                      1.


```

上面这些命令都正常执行。但这种方式启动 crash，只读不可写:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPMpQPD6aKlxibpo80NniaO328TRY82mNLLZTfHLqsG4nUm5JWxhTgRvAseIxkPyr5QgUwmmSD8noaiag/640?wx_fmt=png)

就读操作而言，crash 与 / dev/mem 同步，真实反应内核态内存变化。

为了可读可写，还得用 / dev/mem。

考虑到 crash 这种工具也不是给普通人用的，还是自己解决兼容性 BUG 吧。或许高版本 crash 已无此兼容性 BUG，但基于各种原因，没有自编译安装高版本 crash。

解决思路很简单，肯定有个函数在检查内核版本是否匹配，冒险让它返回真试试。可以直接 IDA 反汇编 crash，不过这是 Linux 开源世界，没必要跟自己过不去，先看源码。

```
$ rpm -qif $(which crash) | grep src.rpm
Group       : Development/Debuggers         Source RPM: crash-5.1.8-1.el5.centos.src.rpm

http://ftp.iij.ad.jp/pub/linux/centos-vault/5.9/os/SRPMS/crash-5.1.8-1.el5.centos.src.rpm


```

Source Insight 中搜 "do not match!"，在 filesys.c 中找到 match_proc_version():

```
/*
 *  If only a namelist argument is entered for a live system, and the
 *  version string doesn't match /proc/version, try to avert a failure
 *  by assigning it to a matching System.map.
 */
static void
match_proc_version(void)
{
    char buffer[BUFSIZE], *p1, *p2;

    if (pc->flags & KERNEL_DEBUG_QUERY)
        return;

    if (!strlen(kt->proc_version))
        return;

    /*
     * 目标环境中match_file_string()返回假，设法让其返回真
     */
    if (match_file_string(pc->namelist, kt->proc_version, buffer)) {
                if (CRASHDEBUG(1)) {
            fprintf(fp, "/proc/version:\n%s\n", kt->proc_version);
            fprintf(fp, "%s:\n%s", pc->namelist, buffer);
        }
        return;
    }

    error(WARNING, "%s%sand /proc/version do not match!\n\n",
        pc->namelist,
        strlen(pc->namelist) > 39 ? "\n         " : " ");
...


```

修改源码并重新编译 crash 是一种方案，因故不选这条路。此外，若只是绕过一个检查，只改几个字节，直接 Patch 二进制更符合我们的人设。

用 IDA 反汇编 crash，match_proc_version() 被 inline 展开到主调函数 fd_init() 中，match_file_string() 的符号则被 strip 掉了。通过 "/proc/version do not match!" 的交叉引用定位:

```
/*
 * match_file_string()是我自己重命名回来的，缺省没有这个符号
 */
00000000004926D1 E8 BA E2 FF FF              call    match_file_string
/*
 * 流程至此时ZF未置位，将"test eax,eax"改成"inc eax"或"nop nop"即可
 */
00000000004926D6 85 C0                       test    eax, eax
00000000004926D8 0F 84 C6 05 00 00           jz      loc_492CA4


```

上面注释已经说了理论上的 Patch 方案，先用 gdb 动态 Patch 测试一下:

```
$ gdb -q -nx -x /tmp/gdbinit_x64.txt /usr/bin/crash

(gdb) display/5i $pc
(gdb) x/5i 0x4926D6
0x4926d6 <fd_init+694>: test   eax,eax
0x4926d8 <fd_init+696>: je     0x492ca4 <fd_init+2180>
0x4926de <fd_init+702>: mov    rax,QWORD PTR [rip+0x6b09db]        # 0xb430c0 <pc>
0x4926e5 <fd_init+709>: cmp    QWORD PTR [rax+0xcb0],0x0
0x4926ed <fd_init+717>: je     0x492556 <fd_init+310>

(gdb) tb *0x4926d6
(gdb) r /usr/lib/debug/lib/modules/2.6.18-308.24.1.el5/vmlinux /dev/mem
(gdb) i r rax eflags
rax            0x0      0
eflags         0x206    [ PF IF ]


```

热 Patch:

```
(gdb) set $rax=1
(gdb) c


```

另一种热 Patch:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPMpQPD6aKlxibpo80NniaO328vUtHRnq4GzFegBMx1uR6Zys15bWfcE0xp22I41bEkODtMV60bBVXPQ/640?wx_fmt=png)

热 Patch 后对 / proc/version 的检查就绕过了，但这样还是用不了 / dev/mem:

```
crash: read error: kernel virtual address: ffffffff804bd2e0  type: "possible"
WARNING: cannot read cpu_possible_map
crash: read error: kernel virtual address: ffffffff8045f5a0  type: "present"
WARNING: cannot read cpu_present_map
crash: read error: kernel virtual address: ffffffff8045a260  type: "online"
WARNING: cannot read cpu_online_map
crash: read error: kernel virtual address: ffffffff80461210  type: "xtime"

crash: this kernel may be configured with CONFIG_STRICT_DEVMEM, which
       renders /dev/mem unusable as a live memory source.

crash: trying /proc/kcore as an alternative to /dev/mem

/proc/kcore: read: Operation not permitted
crash: /proc/kcore: initialization failed

Program exited with code 01.


```

目标系统 crash 用不了 / dev/mem，除了 crash 本身的兼容性 BUG，还有一个来自内核态的安全限制。crash 会检查编译内核时是否指定过 CONFIG_STRICT_DEVMEM，是否能不受限地读取 / dev/mem，这个安全限制是内核态的，不是 crash 应用程序本身的。缺省情况下用户态进程读取 / dev/mem 受限，只能读写前 1MB 多的数据，于是 crash 转而使用 / proc/kcore，也失败了。crash 使用 / proc/kcore 失败应该是另一个兼容性 BUG，我懒得理它，盯着 / dev/mem 用吧。

不考虑自编译内核时禁用 CONFIG_STRICT_DEVMEM，至少有两种方式热消除这个安全限制。用 SystemTap 让内核函数 devmem_is_allowed() 恒返回 1:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPMpQPD6aKlxibpo80NniaO328jTiaHRuR7J7eGtZquBbia9E5d7gz18dSaZuGzmFr1gMpybNlR9uUhv7A/640?wx_fmt=png)

或者自己写 LLKM 去 Patch devmem_is_allowed():

```
$ insmod /tmp/kernelwrite.ko address=0xffffffff80083c03 value=ffc0
insmod: error inserting '/tmp/kernelwrite.ko': -1 Operation not permitted

$ dmesg | tail -2
ffffffff80083c03: 31 c0                                            1.
ffffffff80083c03: ff c0                                            ..

$ xxd -g 1 -s $[0xffffffff80083c03-0xffffffff7fe00000] -l 2 /dev/mem
0283c03: ff c0                                            ..


```

切勿照搬上面的命令，领会精神后自行反汇编并 Patch 相应地址。

假设已经消除 CONFIG_STRICT_DEVMEM 限制，如下 gdb 操作对 crash 进行动态 Patch:

```
$ gdb -q -nx -x /tmp/gdbinit_x64.txt /usr/bin/crash

tb *0x4926d6
commands $bpnum
    silent
    i r rax eflags
    set *(unsigned short int *)$rip=0x9090
    c
end
r /usr/lib/debug/lib/modules/2.6.18-308.24.1.el5/vmlinux /dev/mem


```

进入 crash 提示符，可读可写。

静态 Patch 如下:

```
$ fc /b crash.orig crash
000926D6: 85 90
000926D7: C0 90


```

后续使用的常规组合是:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPMpQPD6aKlxibpo80NniaO328ooN5aYjQHKPqMJGQWjAM9Y4PtjZKhb9Jaj2WECX8jibwOxibUmbvrUcQ/640?wx_fmt=png)