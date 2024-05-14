> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281761.htm)

> [原创]Frida 过检测通用思路之一

**前言**
======

过检测一般可以从退出方式入手。检测到环境异常会想办法退出程序，大致分成 libc 函数方案和 svc 退出方案。

libc: exit, kill, _exit 等，可以直接调用 `FCAnd.anti.anti_debug();` 过掉（有漏的函数可以补充）。

svc: syscall 相关。

libc 层面 frida  可以直接 hook 相关函数然后打印堆栈就可以定位到检测点。

svc 用 frida 无法直接 hook ，可以采用下面的通用过检测思路。  
文中主要脚本引用自： [https://github.com/deathmemory/FridaContainer](https://github.com/deathmemory/FridaContainer)

**检索 svc**
==========

网上也有人分享过用 frida stalker 检测 svc 的方式，也是办法之一，但 stalker 稳定性一直是个问题，所以这里采用了： ida python + frida 的方式。

主要就是将分析 so  调用 svc 的位置用 ida python 部分来实现。直接上 idapython 代码

```
from idautils import *
from idc import *
 
import ida_idp
ida_idp.get_idp_name()
# 定义要搜索的 svc 字节序列
# arm
# search_bytes = b"\x00\x00\x00\xef"
# arm64
search_bytes = b"\x01\x00\x00\xd4"
result = []
# 遍历整个二进制文件的可执行段
seg = ida_segment.get_segm_by_name(".text")
 
# 在每个段中搜索字节序列
for address in range(seg.start_ea, seg.end_ea):
    if get_bytes(address, len(search_bytes)) == search_bytes:
        print("Match found at address: 0x%x" % address)
        result.append(address)
 
print([hex(n) for n in result])

```

可以在 `python/android/idaSearchSvc.py`加载到 ida 执行，此时会输出该 so 的 svc 所有的调用地址，如：

```
['0x4826c', '0x487bc', '0x48dc4', '0x496d4', '0x49880', '0x499d0', '0x4b200', '0x4bf40', '0x51578', '0x51598', '0x516fc', '0x51984', '0x519bc', '0x51a34', '0x51b24', '0x51b9c', '0x51e98']

```

**利用 frida 定位检测点**
==================

将上面输出的 `svc_address_list`全部 hook 掉，或直接调用 FCAnd 的封装代码：

```
let svc_address_list = ['0x4826c', '0x487bc', '0x48dc4', '0x496d4', '0x49880', '0x499d0', '0x4b200', '0x4bf40', '0x51578', '0x51598', '0x516fc', '0x51984', '0x519bc', '0x51a34', '0x51b24', '0x51b9c', '0x51e98'];
FCAnd.watch_svc_address_list(mod.base, svc_address_list);

```

然后用 frida `spawn`的方式启动目标进程就会打印出所有调用 svc 的堆栈信息。为了及时 hook 可以选择在目标 so 加载之后开始 hook 

```
FCAnd.whenSoLoad("libshell-super.com.wm.xxxxxx.so", (mod) => {
    let svc_address_list = ['0x4826c', '0x487bc', '0x48dc4', '0x496d4', '0x49880', '0x499d0', '0x4b200', '0x4bf40', '0x51578', '0x51598', '0x516fc', '0x51984', '0x519bc', '0x51a34', '0x51b24', '0x51b9c', '0x51e98'];
    FCAnd.watch_svc_address_list(mod.base, svc_address_list);
    // DMLog.i("dd", "so load base: " + mod.base);
});

```

得到堆栈信息如下：

```
......
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-134][9551][showNativeStacks]:  Backtrace:
  0x6e86545b10 libshell-super.com.xxxxxx.so!0x51b10
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-135][9551][showNativeStacks]:  Backtrace:
  0x6e86545b10 libshell-super.com.xxxxxx.so!0x51b10
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-136][9551][showNativeStacks]:  Backtrace:
  0x6e864ede0c
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-137][9551][showNativeStacks]:  Backtrace:
  0x6e86545b10 libshell-super.com.xxxxxx.so!0x51b10
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-138][9551][showNativeStacks]:  Backtrace:
  0x6e86545b10 libshell-super.com.xxxxxx.so!0x51b10
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-139][9551][showNativeStacks]:  Backtrace:
  0x6e864ede0c
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-140][9551][showNativeStacks]:  Backtrace:
  0x6e86545b10 libshell-super.com.xxxxxx.so!0x51b10
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
[INFO][05/10/2024, 07:14:18 PM][PID:9502][Thread-141][9551][showNativeStacks]:  Backtrace:
  0x6e86545b10 libshell-super.com.xxxxxx.so!0x51b10
  0x6e86545580 libshell-super.com.xxxxxx.so!0x51580
  0x7187d9075c libc.so!_ZL15__pthread_startPv+0x44
  0x7187d30154 libc.so!__start_thread+0x44
......

```

大概翻一下，出现频率比较多的基本就是。0x51b10 -- 0x51580

接下来就是 ida 定位到相关地址，看看检测逻辑就可以判断和绕过了。

![](https://bbs.kanxue.com/upload/attach/202405/357319_WPTX79CPA8Z3DQ3.webp)

比如这里检测函数是 `sub_515C4`，那进去看一下这个函数都检测了什么，详细就不展开了，只说一下结论，这个函数主要检测了：

1.  /proc/self/task/%s/status 的线程名
    
2.  /proc/self/fd/%s  的 readlink 判断特征名
    
3.  /proc/self/maps 的 frida 特征名
    

没什么新鲜的检测点，但奇怪的是明明我的 frida 做过去特征，按理说这些点应该都过掉了才对，结果还是被检测然后程序退出了，又回过头反复确认了就是这个检测点无疑，那到底是 `sub_515C4`的哪个检测逻辑触发的检测呢？

**精确定位**
========

既然确定了就是 `sub_515C4`检测的，而且检测点就这么几个文件，那到底是哪里出了问题，先用 frida hook snprint 函数过滤它的路径拼接，发现影响检测的就是 `/proc/self/task/%s/status`。针对它的检测也就以下三个特征逻辑：

1.  gum-js-loop
    
2.  gmain
    
3.  pool-frida
    

![](https://bbs.kanxue.com/upload/attach/202405/357319_DZX88V4WS6AFSXP.webp)

前两个确定在编译去特征 frida 的时候都是替换过的，可疑的只有 `pool-frida`，但是我在 frida 源码里和生成的文件里没有直接搜到这个关键字符串，不信邪，为了确定到底是不是它，我们进一步用 frida 的 stalker 来验证一下。

在 `sub_515C4`onEnter 的时候开始 follow ，退出时 unfollow 减少不必要的性能损耗和异常。

```
Interceptor.attach(base.add(0x515C4), {
    onEnter: function (args) {
        DMLog.i("hook_515C4", "onEnter");
 
        // FCAnd.showNativeStacks(this.context);
        function logX3(context: any) {
            console.log('x3:', context.x3.readInt());
        }
 
        Stalker.follow(this.tid, {
            events: {
                call: false,
                ret: false,
                exec: true,
                block: false,
                compile: false
            },
            transform: function (iterator: StalkerArm64Iterator) {
                var ins = iterator.next();
                if (ins == null) return;
                if (ins.address.sub(base) > ptr(0x00000000000515C4)
                    && ins.address.sub(base) < ptr(0x000000000005207C)) {
                    // if (ins.address.sub(base).compare(ptr(0x52074)) == 0) {
                    //     // console.log(`==== 51afc fd: `);
                    //     // iterator.putCallout(logX3);
                    //     DMLog.i("0x52074", "sleep");
                    //     Thread.sleep(9999);
                    // }
                    console.log(`${ins.address.sub(base)} ${ins}`);
                }
                iterator.keep();
                while (iterator.next() != null) iterator.keep();
            }
        });
    },
    onLeave: function (retval) {
        DMLog.i("hook_515C4", "onLeave1111======: " + retval);
        if (retval.toInt32() == 1) {
            DMLog.i("hook_515C4", "sleep x3: " + (this.context as Arm64CpuContext).x3);
            Thread.sleep(9999);
        }
        retval.replace(ptr(0));
        Stalker.unfollow(this.tid);
    }
})

```

打印的结果如下：

```
......
0x302666770 cbz x0, #0x71d69bb788
0x302666774 ldaxr x21, [x0]
0x302666780 stlxr w8, x22, [x0]
0x302666780 stlxr w8, x22, [x0]
0x302666774 ldaxr x21, [x0]
0x302666788 ldp x20, x19, [sp, #0x20]
0x30266484c add x0, x19, #0x18
0x3026c8d20 adrp x16, #0x71d6a22000
0x3026c51a8 movi v0.2d, #0000000000000000
0x3026c51b8 strh wzr, [x0]
0x302664860 b #0x71d69b9868
0x302664868 mov x0, x19
0x51a20 cbz x0, #0x6ed43a7014
0x51a24 mov x27, x0
0x51a38 mov x0, x27
0x4440 adrp x16, #0x6ed43c6000
0x302664878 stp x29, x30, [sp, #-0x30]!
0x30266488c add x19, x0, #0x18
0x3026c8740 adrp x16, #0x71d6a22000
0x3026c5238 stp x29, x30, [sp, #-0x10]!
0x3026c5250 and x8, x8, #0x2000
0x3026c5264 stxrh w10, w9, [x0]
0x3026c5264 stxrh w10, w9, [x0]
0x3026c5258 ldaxrh w10, [x0]
0x3026c526c mov w0, wzr
0x30266489c ldr x8, [x20, #8]
0x3026648cc add x21, x20, #0x40
0x3026af860 mov x8, #0x3d
0x3026af874 ret
0x3026648e4 cmn w0, #1
0x302664900 cmp w0, #1
0x302664908 mov w8, w0
0x3026648a8 ldrh w9, [x21, #0x10]
0x302664918 mov x0, x19
0x3026c8780 adrp x16, #0x71d6a22000
0x3026c5740 stp x29, x30, [sp, #-0x30]!
0x3026c5764 orr w9, w8, #2
0x3026c5764 orr w9, w8, #2
0x3026c5774 cmp w10, w9
0x3026c586c mov w0, wzr
0x302664920 mov x0, x21
0x51a40 cbz x0, #0x6ed43a700c
0x51a44 movi v0.2d, #0000000000000000
0x51ac0 add x9, x9, #1
0x51ab0 ldrb w10, [x8]
0x51acc b #0x6ed43a6a38
0x3026c5250 and x8, x8, #0x2000
0x3026648a4 ldr x21, [x20, #0x10]
0x51ad0 adrp x9, #0x6ed43af000
0x51aec add x9, x9, #1
0x51adc ldrb w10, [x8]
0x51af8 b #0x6ed43a6a38
0x51afc adrp x2, #0x6ed43af000
0x4020 adrp x16, #0x6ed43c6000
0x3026bee88 stp x29, x30, [sp, #-0x20]!
0x3026bef00 add x10, sp, #0x118
0x3026a03d0 stp x29, x30, [sp, #-0x60]!
0x3026c8550 adrp x16, #0x71d6a22000
0x302661ff0 mrs x8, tpidr_el0
0x3026a040c movi v2.2d, #0000000000000000
0x3026a047c ldr w9, [x8, #0x30]
0x3026a0498 mov w9, #-1
0x3026a048c ldr x8, [x19, #0x18]
0x3026a04cc mov w8, #0x1a
0x3026a05cc add x9, x22, #0x18
0x3026a0688 cmp w8, #0x25
0x3026a0690 add x9, x9, #1
0x3026a069c add x10, x28, x9
0x3026a06a4 subs x9, x10, x28
0x3026a06ac sub w11, w20, w26
0x3026a06b8 ldr x23, [sp, #0x90]
0x3026a06e4 mov w15, wzr
0x3026a0720 adrp x11, #0x71d698d000
0x3026a0e4c mov w21, w15
0x3026a0e58 ldr x8, [sp, #0x1a0]
0x3026a1340 ldr x8, [sp, #0x50]
0x3026a134c ldr x10, [sp, #0x50]
0x3026a1360 ldr x9, [sp, #0x20]
0x3026a14e4 ldr x8, [x8]
0x3026a1548 str w27, [sp, #0xb4]
0x3026a1568 mov x0, x25
0x3026c8580 adrp x16, #0x71d6a22000
0x30265e870 and x4, x0, #0xfff
0x30265e880 ldp x2, x3, [x0]
0x30265e8a4 csel x4, x4, x5, lo
0x3026a1570 mov x26, x0
0x3026a157c str x20, [sp, #0xc0]
0x3026a1cbc ldrb w9, [sp, #0x1fc]
0x3026a1cfc ldr w8, [sp, #0xcc]
0x3026a1e14 mov w21, w24
0x3026a1e94 ldrb w8, [sp, #0x1ad]
0x3026a1f08 cmp w20, #0x80
0x3026a2094 cmp w27, #1
0x3026a2008 ldr w22, [sp, #0xb4]
0x3026a2010 ldr x8, [sp, #0x1c0]
0x3026a2074 ldr w26, [sp, #0xac]
0x3026a29ec ldr w21, [sp, #0xcc]
0x3026a2af0 cmp w21, w24
0x3026a2b08 ldr x9, [sp, #0x1c0]
0x3026a2b14 add x1, sp, #0x1b0
0x3026ab9b8 stp x29, x30, [sp, #-0x60]!
0x3026ab9dc ldr w8, [x0, #0x10]
0x3026ab9ec ldr x9, [x19, #0x18]
0x3026aba1c ldr x9, [x20]
0x3026aba2c tbnz w8, #0, #0x71d6a00c04
0x3026aba30 mov w25, #0x4200
0x3026aba8c cbnz x21, #0x71d6a00aa0
0x3026abaa0 ldp w23, w8, [x19, #0xc]
0x3026abb14 tbnz w8, #9, #0x71d6a00a38
0x3026aba38 cmp x21, w23, sxtw
0x3026c8500 adrp x16, #0x71d6a22000
0x30265e000 sub x5, x0, x1
0x30265de90 prfm pldl1keep, [x1]
0x30265dee0 cmp x2, #8
0x30265dee8 ldr x6, [x1]
0x3026aba54 ldr w8, [x19, #0xc]
0x3026aba90 ldr x21, [x26, #8]
0x3026aba9c ldur x22, [x26, #-0x10]
0x30265df00 tbz w2, #2, #0x71d69b2f18
0x30265df04 ldr w6, [x1]
0x3026abbfc mov w0, wzr
0x3026abd50 ldp x20, x19, [sp, #0x50]
0x3026a2b20 mov w14, #0x10
0x3026a2b30 str wzr, [sp, #0x1b8]
0x3026a067c mov x9, xzr
0x3026a06d8 b #0x71d69f7d58
0x3026a2d58 cbz x9, #0x71d69f7d74
0x3026a2d5c add x1, sp, #0x1b0
0x3026a2d68 str xzr, [sp, #0x1c0]
0x3026a2d74 str wzr, [sp, #0x1b8]
0x3026c8850 adrp x16, #0x71d6a22000
0x30264fb08 stp x29, x30, [sp, #-0x10]!
0x30264fb44 cbnz x8, #0x71d69a4b50
0x30264fb48 ldp x29, x30, [sp], #0x10
0x302658580 mov x1, x0
0x3026585a0 sub sp, sp, #0x40
0x3026585d0 adrp x8, #0x71d6a22000
0x3026585dc cbnz x22, #0x71d69ad608
0x3026585e0 b #0x71d69ad6a4
0x3026586a4 ldp x20, x19, [sp, #0x30]
0x3026a2d90 ldr x0, [sp, #0xb8]
0x3026a2d9c ldr x0, [sp, #0x1a0]
0x3026a2dbc ldr x8, [x25, #0x28]
0x3026a2dcc mov w0, w22
0x3026bef9c ldr x8, [sp, #0x118]
0x3026befb4 add sp, sp, #0x230
0x51b10 mov w8, #0x38
0x51b2c add x8, sp, #0x370
0x51b60 strb wzr, [x8], #1
0x51b6c strb wzr, [x8], #1
0x51b78 strb wzr, [x8], #1
0x51b54 strb wzr, [x8], #1
0x51b84 mov x10, xzr
0x51ba8 ldrb w8, [sp, #0x270]
0x51bb4 strb w8, [x23, x10]
0x51b8c mov w8, #0x3f
0x51bc4 ldr x8, [sp, #0x20]
0x51bd0 ldurb w10, [x8, #-5]
0x51bdc cbnz w10, #0x6ed43a6bcc
0x51bcc add x8, x8, #1
0x51be0 b #0x6ed43a6c60
0x51c60 ldr x8, [sp, #0x18]
0x51c6c ldurb w10, [x8, #-2]
0x51c78 cbnz w10, #0x6ed43a6c68
0x51c68 add x8, x8, #1
0x51c7c b #0x6ed43a6cb4
0x51cb4 ldr x8, [sp, #0x10]
0x51cc0 ldurb w10, [x8, #-4]
0x51ccc cbnz w10, #0x6ed43a6cbc
0x51cbc add x8, x8, #1
0x51cd0 b #0x6ed43a6a2c
0x51a2c mov w8, #0x39
0x51be4 ldurb w10, [x8, #-4]
0x51c80 ldurb w10, [x8, #-1]
0x51cd4 ldurb w10, [x8, #-3]
0x51ce0 ldurb w10, [x8, #-2]
0x51cec ldurb w10, [x8, #-1]
0x51cf8 ldrb w10, [x8]
0x51d04 ldrb w10, [x8, #1]
0x51d10 ldrb w10, [x8, #2]
0x51d1c ldrb w10, [x8, #3]
0x51d28 ldrb w10, [x8, #4]
0x51d34 ldrb w10, [x8, #5]
0x51d40 b #0x6ed43a7070
0x52070 ldr x9, [sp, #0x38]
0x52040 ldr x8, [x9, #0x28]
0x52050 add sp, sp, #0x580
0x305eb050c ldr x17, #0x71da205518
0x30476a14c sub sp, sp, #0x10
[INFO][05/07/2024, 07:52:08 PM][PID:18959][Thread-17][19001][hook_515C4]: onLeave1111======: 0x1

```

至此得到了该函数的执行路径，定位到关键点：0x51afc

![](https://bbs.kanxue.com/upload/attach/202405/357319_4MMZCZPVDZ78K6S.webp)

确定它走的就是 status 相关的检测逻辑。

第二个关键点：0x51D34

![](https://bbs.kanxue.com/upload/attach/202405/357319_DTBR63PVFBSU2FW.webp)

说明它确实是检测的 `pool-frida`这下是确定就是这个检测逻辑了，但之前搜索源码和生成 bin 文件都没有这个关键词，剩下的可能就是这个字符串可能是拼接出来的。

最终定位到是 `pool-%s`也就是拼接后的`pool-frida`。

顺便说一下这个`pool-frida`线程，是在进程启动时才会有的临时线程，frida 环境 ready 后这个线程会退出。

知道问题就好办了，在我们给 frida 过检测的脚本里把这个字符串替换掉就可以了。

又可以愉快的 hook 了 : )

[[培训] 二进制漏洞攻防（第 3 期）；满 10 人开班；模糊测试与工具使用二次开发；网络协议漏洞挖掘；Linux 内核漏洞挖掘与利用；AOSP 漏洞挖掘与利用；代码审计。](https://www.kanxue.com/book-section_list-174.htm)

最后于 9 小时前 被 DMemory 编辑 ，原因： update

[#工具脚本](forum-161-1-128.htm) [#HOOK 注入](forum-161-1-125.htm) [#逆向分析](forum-161-1-118.htm)