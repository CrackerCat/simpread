> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287383.htm)

> [原创] 安卓旧系统 OTA 包分析与漏洞提权适配

之前碰到一款安卓旧系统版本的设备，对其有提权调试需求，试了几款 Root 工具感觉不太好用，加上发现设备系统有检测措施会进行相应的保护，于是思路转向了通过 CVE 漏洞对设备进行临时提权。经过一番摸索分析，找到了还没有挂掉的系统 OTA 升级包的下载链接，恢复出来了内核符号表及地址，从而打开突破口，定位到 CVE 提权漏洞需要的符号信息，最后适配编译了 CVE-2015-1805（Pipe Read）、CVE-2015-5195（Ping Pong）和 CVE-2017-8890（Phoenix Talon）三个漏洞利用提权程序，可以成功提权，这里记录一下。

OTA 包提取内核符号表
============

如果有 root 权限的话，可以执行 `echo 0 > /proc/sys/kernel/kptr_restrict` 和 `cat /proc/kallsyms > kallsyms.txt` 来获取内核符号表及地址信息，但只有 OTA 更新包的话情况会有些不同。

可以通过最简单的抓包方式获取到 OTA 更新包下载地址，不同系统的 OTA 更新包内容各不相同，常见的是会包含一个 boot.img，由内核自解压引导程序、压缩的内核镜像、文件系统、设备树等一起打包，使用 `binwalk -Me` 命令能查看和提取打包的内容。

图中根据描述信息和区块大小，可以判断 0x800 位置开始包括后面的 0x47F7 区块，是一个 ARM 架构的 zImage 格式内核镜像文件，前者是内核引导头，后者是压缩的 Linux 文件 vmlinux。得到的 47F7 文件是解压后的 vmlinux 文件，如果是一个 ELF 文件的话，直接用 `nm -n` 命令即可查看符号表，但分析发现碰到的是一个没有 ELF 头的纯二进制内核代码，这个方法不能使用。

![](https://bbs.kanxue.com/upload/attach/202506/802108_WFQP4M23Q86AVGU.png)

经过一番分析搜索，使用 [vmlinux-to-elf](elink@e9dK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6E0j5i4u0A6L8W2)9J5k6r3#2Q4x3V1k6$3L8h3I4A6L8Y4g2^5i4K6u0V1N6r3!0Q4x3X3c8W2L8r3j5`.) 项目的 [kallsyms_find.py](elink@6b6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6E0j5i4u0A6L8W2)9J5k6r3#2Q4x3V1k6$3L8h3I4A6L8Y4g2^5i4K6u0V1N6r3!0Q4x3X3c8W2L8r3k6Q4x3V1k6T1L8r3!0T1i4K6u0r3L8h3q4K6N6r3g2J5i4K6u0r3N6X3#2D9K9h3&6#2P5q4)9#2k6Y4c8G2i4K6g2X3k6h3I4X3i4K6u0r3K9$3q4D9L8s2y4&6L8i4y4Q4y4h3k6X3K9h3&6V1k6i4u0Q4x3X3g2H3P5b7`.`.) 模块，可以从 47F7 文件中提取生成内核符号表。

![](https://bbs.kanxue.com/upload/attach/202506/802108_J6PCDZSTNN39N3R.png)

有了内核符号表后，可以直接把 47F7 拖进 IDA，选择 arm 小端序架构后将 ROM start address 和 Loading address 设置为 0xc0008000 加载，再撰写执行 IDAPython 脚本来给内核二进制文件恢复符号。

```
ksyms = open(r"D:\47F7.kallsyms")
for line in ksyms:
    addr = int(line[0:8],16)
    name = line[11:-1]
    print(f"addr:{hex(addr)},{name}")
    if not ida_funcs.add_func(addr):
        print(f"Warning: Failed to add function at {hex(addr)}")
    if not idc.set_name(addr, name, idc.SN_NOWARN):
        print(f"Warning: Failed to set name at {hex(addr)}")
```

执行完后便可看到已经有函数符号了，现在就能查找相关 CVE 提取漏洞函数来进行适配了。

![](https://bbs.kanxue.com/upload/attach/202506/802108_J3TKGNS6XJEE29B.png)

其实整个过程还可以更简单，[vmlinux-to-elf](elink@958K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6E0j5i4u0A6L8W2)9J5k6r3#2Q4x3V1k6$3L8h3I4A6L8Y4g2^5i4K6u0V1N6r3!0Q4x3X3c8W2L8r3j5`.) 项目包含了解压 zImage、提取内核符号表和封装为 ELF 文件的功能，直接执行 `vmlinux-to-elf 47F7 kernel.ELF`，便可得到一个拥有符号可拖进 IDA 直接进行分析的内核文件。

![](https://bbs.kanxue.com/upload/attach/202506/802108_9TKDHMAP4QBKFPH.png)

以及上面的 47F7 是我们手动用 binwalk 解包出来的，有些 OTA 包里面可能没有 boot.img 文件，而是有一个 kernel 文件，binwalk 查看是由内核引导头和压缩的 Linux 文件 vmlinux 组成，这时候直接用 vmlinux-to-elf 来生成内核 ELF 文件，就会用到其解压 zImage 模块功能，经测试是要比一些 [extract-vmlinux](elink@fa0K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4y4@1i4K6u0W2k6$3W2@1K9s2g2T1i4K6u0W2j5$3!0E0i4K6u0r3L8h3N6W2k6h3E0&6i4K6u0r3j5K6j5K6j5h3b7$3k6e0V1J5j5K6j5%4x3h3u0U0z5e0l9%4z5e0k6X3x3h3f1%4k6h3x3I4k6h3g2U0k6U0R3`.) 脚本支持适配得更广泛。

![](https://bbs.kanxue.com/upload/attach/202506/802108_96SN233RQXTZE4G.png)

翻一下 [vmlinux-to-elf](elink@adbK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6E0j5i4u0A6L8W2)9J5k6r3#2Q4x3V1k6$3L8h3I4A6L8Y4g2^5i4K6u0V1N6r3!0Q4x3X3c8W2L8r3j5`.) 项目代码会发现，这个工具根据 Linux 内核 kallsyms 系统演变的不同版本进行了适配，通过特征搜索定位，模拟内核解压算法过程，最后构建符号表，实现从内核二进制文件中直接解析出符号表。

CVE 漏洞适配
========

拥有内核符号表后，加上在前人大量的漏洞分析与分享的基础上，进行提权漏洞适配就轻松得多了，这里就不进行具体的漏洞复现、调试和分析过程了，简要说下漏洞适配过程，还有一些踩的坑。

`CVE-2015-1805` 参考 [dosomder/iovyroot](elink@48eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4y4G2L8h3c8W2M7W2)9J5c8X3W2G2N6Y4W2J5L8$3!0@1) 项目，在 `jni/inlcude/offsets.h` 中看到有结构体 offset，我们需要在内核符号表中找到 ptmx_fops、sidtab、policydb、selinux_enabled 和 selinux_enforcing 位置值，在 `jni/offset.c` 文件中添加即可。

```
struct offsets {
    char* devname; //ro.product.model
    char* kernelver; // /proc/version
    union {
        void* fsync; //ptmx_fops -> fsync
        void* check_flags; //ptmx_fops -> check_flags
    };
#if (__LP64__)
    void* joploc; //gadget location, see getroot.c
    void* jopret; //return to setfl after check_flags() (fcntl.c), usually inlined in sys_fcntl
#endif
    void* sidtab; //optional, for selinux contenxt
    void* policydb; //optional, for selinux context
    void* selinux_enabled;
    void* selinux_enforcing;
};
```

`CVE-2015-5195` 参考项目 [fi01/CVE-2015-3636](elink@de3K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6X3K9e0l9I4i4K6u0r3b7#2k6q4i4K6u0V1x3U0l9I4y4g2)9J5k6o6x3$3x3K6j5`.)，需要确定结构体 sock 的 sk_prot、sk_stamp 和结构体 inet_sock 的 mc_list 成员偏移，可分别在 inet_release、sock_get_timestampns 和 ip_mc_drop_socket 函数中对比源码找到，随后替换即可。

`CVE-2017-8890` 参考项目 [idhyt/androotzf](elink@323K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6A6k6r3S2&6N6q4)9J5c8X3q4F1k6s2u0G2L8%4c8*7k6W2)9J5c8X3u0D9L8$3u0Q4x3V1k6E0j5h3W2F1i4K6u0r3k6i4S2H3L8r3!0A6N6q4)9J5c8V1y4h3c8g2)9J5k6o6t1H3x3e0N6Q4x3X3b7^5z5o6V1H3i4K6u0r3x3e0l9H3x3g2)9J5c8Y4u0G2L8%4c8*7x3K6u0Q4x3X3g2U0)，需要的结构体代码中直接定义了，不用怎么修改，就 mmap 传入的 fake_iml_next_rcu 值根据自己系统情况进行了修改。

编译的话据我测试 NDK 版本是有影响的，高些版本的编译出来的可能会报错或者利用失败，测下来 NDK r11c 算是比较稳定，可在 [Android NDK Unsupported Downloads](elink@556K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0r3L8X3c8C8i4K6u0r3N6$3W2C8K9g2)9J5c8W2g2F1M7%4g2H3M7r3!0J5N6r3g2V1i4K6u0V1c8r3!0%4L8X3I4G2j5h3c8K6) 中下载。

不同系统上内存占用情况不一样，mmap 函数调用传入的 MMAP_SIZE 可能需要进行一些尝试修改。

总结
==

本文记录了笔者在想办法 Root 安卓旧系统设备时候，通过未过期的 OTA 更新包下载链接，提取分析出了内核文件和符号表，进一步适配了三个提权漏洞，实现了对旧设备的提权调试目的。

之前并没有怎么了解过 Linux 漏洞挖掘和利用分析，虽然事后看来整个过程相当简单，只是提取下内核符号信息找偏移适配下即可，但也是花了一番功夫摸索实践。相当多的时间是在翻阅不同的 CVE，去理解各自的漏洞成因和利用过程，来看该怎么适配。以前只是听说过 “堆喷占位” 等觉得高深莫测的词语，也有了具象可验证的实例，没那么抽象遥远了，准备日后空了动手复现调试分析一些经典漏洞，来加深理解。

[[培训] 科锐逆向工程师培训第 53 期 2025 年 7 月 8 日开班！](https://bbs.kanxue.com/thread-51839.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Linux](forum-150-1-161.htm) [#Andorid](forum-150-1-162.htm)