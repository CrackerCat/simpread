> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/O5Omgs1jEca0K6nNDzqWfg)

**一、eBPF 是什么**
--------------

eBPF 是 extended BPF 的缩写，而 BPF 是 Berkeley Packet Filter 的缩写。对 linux 网络比较熟悉的伙伴对 BPF 应该比较了解，它通过特定的语法规则使用基于寄存器的虚拟机来描述包过滤的行为。比较常用的功能是通过过滤来统计流量，tcpdump 工具就是基于 BPF 实现的。而 eBPF 对它进行了扩展来实现更多的功能。

主要区别如下：

1）允许使用 C 语言编写代码片段，并通过 LLVM 编译成 eBPF 字节码；

2）cBPF 只实现了 SOCKET_FILTER，而 eBPF 还有 KPROBE 、PERF 等。

3）BPF 使用 socket 实现了用户态与内核交互，eBPF 则定义了一个专用于 eBPF 的新的系统调用，用于装载 BPF 代码段、创建和读取 BPF map，更加通用。

4）BPF map 机制，用于在内核中以 key-value 的方式临时存储 BPF 代码产生的数据。

对于 eBPF 可以简单的理解成 kernel 实现了一个虚拟机机制，将类 C 代码编译成字节码（后文有详细解释），挂在到内核的钩子上，当钩子被触发时，kernel 在虚拟机的 "沙盒" 中运行字节码，这样既能方便的实现很多功能，也能通过沙箱保证内核的安全性。

**二、eBPF 能干什么**
---------------

如果说 BPF 专注于流量监控，那么 eBPF 主要专注的是性能领域，通过各种钩子，能在用户空间得到系统各种性能指标。可以大到监控系统整体的统计指标，也可以小到一个系统函数的运行时间。

这里需要提一下开源项目 BPF Compiler Collection (BCC)，这是一个很方便的基于 eBPF 的系统监视工具，下面这张 BCC 的说明图就能很好的说明我们使用 eBPF 能够做到的事。BCC 在 android 系统上也可以运行，但是要对系统进行一定程度的修改，后续可能会写单独的文章进行讲解。对于内核开发者我还比较关注怎么自己来实现监控的功能，下文也将做简单的讲解。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKhNc11UPD0n2pNp3Z0aRTSXb2AyiarfWAbr2Un5tTg9rlqTBib8fboZ9Q/640?wx_fmt=png)

从上图，我么可以看到，eBPF 几乎能监控系统的所有方面：

1）应用及虚拟机的各种指标

2）系统库性能监控

3）kernel 系统调用性能

4）文件系统性能

5）网络调用性能

6）CPU 调度器性能

7）内存管理性能

8）中断性能

**三、eBPF 框架**
-------------

在开始说明之前先解释下 eBPF 上的名词，来帮忙更好的理解。

1）eBPF bytecode：将 C 语言写的钩子代码，通过 clang 编译成二进制字节码，通过程序加载到内核中，钩子触发后在 kernel " 虚拟机 " 中运行。

2）JIT: Just-in-time compilation，将字节码编译成本地机器码来提升运行速度，和 Java 中的概念类似。

3）Maps：钩子代码可以将一些统计类信息保存在键值对的 map 中，来与用户空间程序进行通信，传递数据。

关于 eBPF 机制详细的讲解网上有很多，这里就不展开了，这里先上一张图，这里包括了使用或者编写 ebpf 涉及到的所有东西，下面会对这个图进行详细的讲解。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKw5RBGMQLLGMKibwJXvjia8ibLOy5THhQiag8Vic8cYg7qAlvPYw7CWCMEIw/640?wx_fmt=png)

1）foo_kern.c 钩子实现代码，主要负责：

*   声明使用的 Map 节点
    
*   声明钩子挂载点及处理函数
    

2）通过 LLVM/clang 编译成字节码

 编译命令：clang --target=bpf

 android 平台有集成 eBPF 的编译，后文会提到

3）foo_user.c 用户空间处理函数，主要负责：

*   将 foo_kern.c 编译成的字节码加载到 kenel 中
    
*   读取 Map 中的信息并处理输出给用户
    

4）kernel 当收到 eBPF 的加载请求时，会先对字节码进行验证，并通过 JIT 编译为机器码，当钩子事件来临后，调用钩子函数

 kernel 会对加载的字节码进行验证，来保证系统的安全性，主要验证规则如下：

a. 检查是否声明了 GNU GPL，检查 kernel 的版本是否支持

b. 函数调用规则：

*   允许 bpf 函数之间的相互调用
    
*   只允许调用 kernel 允许的 BPF helper 函数，具体可以参考 linux/bpf.h 文件
    
*   上述以外的函数及动态链接都是不允许的。
    

c. 流程处理规则：

*   不允许使用 loop 循环以防止进入死循环卡死 kernel
    
*   不允许有不可到达的分支代码
    

d. 堆栈大小被限制在 MAX_BPF_STACK 范围内。

e. 编译的字节码大小被限制在 BPF_COMPLEXITY_LIMIT_INSNS 范围内。

5) 钩子挂载点，主要包括：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKfnbAYgv9AMnmiacVX8TjEF0icy6s46CL46QfX4fFoBlej2nU7GvEdQJA/640?wx_fmt=png)

另外在 kernel 的源代码中 samples/bpf 目录下有大量的示例，感兴趣的可以阅读下。

**四、eBPF 在 Android 平台的使用**
--------------------------

经过上面枯燥的讲解，大家应该对 eBPF 有了基础的认识，下面我们就来通过 android 平台上的一个监控性能的小例子来实操下。

这个小例子的需求是统计系统中每个应用在一段时间内系统调用的次数。

### **1.** **android 系统对 eBPF 的编译支持**

目前 android 编译系统已经对 eBPF 进行了集成，通过 android.bp 就能很方便的在 android 源代码中编译 eBPF 的字节码。

android.bp 示例：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKpSNTjD8Lvems8UJuwRuEs2QYMqGoJrT8TuWIqKLAPG7dLAenUp3ibWg/640?wx_fmt=png)

相关的编译代码在 soong 的 bpf.go，虽然 google 关于 soong 的文档很少，但是至少代码是比较清晰的。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKq93t2nO5riaibkUfcXrmmW9cBTfMiacWs2GpvVia9lXTB5jDNE2wvpYbCw/640?wx_fmt=png)

这里的 $ccCmd 一般是 clang, 所以它的编译命令主要是 clang --target=bpf。和普通的 bpf 编译没有区别。

### **2. eBPF 钩子代码实现**

解决了编译问题，下一步我们开始实现钩子代码，我们准备使用 tracepoint 钩子，首先要找到我们需要的 tracepoint 函数 sys_enter 和 sys_exit。

函数定义在 include/trace/events/syscalls.h 文件中

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKd1ao6iaOxpNcIgicXoaMDejSiaAB67F1hsibArFExd8MWc55lMjExxZL5w/640?wx_fmt=png)

1）sys_enter 的 trace 参数是 id 和长度为 6 的数组。

2）sys_exit 的 trace 参数是两个长整形数 id 和 ret。

找到了钩子后，下一步就可以编写钩子处理代码了：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKVk8sdQribEH7MWjRKQ8S3yKdYIgkw7K4yXhibmVADWM00d87cNFGkaFA/640?wx_fmt=png)

1）定义 map 保存系统调用统计信息，在 DEFINE_BPF_MAP 声明 map 的同时，也会生成删，改，查的宏函数，例如本例中会生成如下函数

bpf_pid_syscall_map_lookup_elem

bpf_pid_syscall_map_update_elem

bpf_pid_syscall_map_delete_elem

2）定义回调函数参数类型，需要参考前面的 tracepoint 的定义。

3）指定监听的 tracepoint 事件。

4）使用 bpf_trace_printk 函数打印 debug 信息，会直接打印信息到 ftrace 中。

5）在 map 中查找指定 key。

6）更新指定的 key 的值。

### **3. 加载钩子代码**

我们只需要把我们编译出来的 *.o 文件 push 到手机的 system/etc/bpf 目录下，重启手机，系统会自动加载我们的钩子文件，加载成功后会在 /sys/fs/bpf 目录下显示我们定义的 map 及 prog 文件。

系统加载代码在 system/bpf/bpfloader 中，代码很简单。

主要有如下操作：

1）在 early-init 阶段向下面两个节点写 1

–  /proc/sys/net/core/bpf_jit_enable

 使能 eBPF JIT，当内核设定 BPF_JIT_ALWAYS_ON 的时候，默认为 1

– /proc/sys/net/core/bpf_jit_kallsyms

 使特权用户可以通过 kallsyms 节点读取 kernel 的 symbols

2）启动 bpfloader service

– 读取 system/etc/bpf 目录下的 *.o 文件，调用 libbpf_android.so 中的 loadProg 函数加载进内核。

– 生成相应的 /sys/fs/bpf / 节点。

– 设置属性 bpf.progs_loaded 为 1

sys 节点分为 map 节点和 prog 节点两种， 分别为 map_<filename>_<mapname>, prog_<filename>_<mapname>

下面是 Android Q 版本上的节点信息。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKmMAJ8DSP8BQpRE5bFf7JLVBDLsSStrgH99wA9Kkg5q1dw5I4HIPmDg/640?wx_fmt=png)

可以使用下面的命令调试动态加载

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKRhEibcJfz4uOy8635T5k1AZwFbaqufWVnU9uiaAApyFfuq4Nh55Ricmfw/640?wx_fmt=png)

### **4. 用户空间程序实现**

下面我们需要编写用户空间的显示程序，本质上就是在用户态通过系统调用把 BPF map 给读出来。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKPEjdxhX6fnFORFlRPkr2cp2znQwKlG7aibqB0oCPfibp8CFd9HvGI23w/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnKkLaEiaf4kOzz210nrE1zoMQ0znafHR6palHvwnBBTumuB09aK7QY0dA/640?wx_fmt=png)

1）eBPF 统计只有在调用 bpf_attach_tracepoint 只有才会起作用。bpf_attach_tracepoint 是 bcc 里面的函数，android 将 bcc 的一部分内容打包成了 libbpf，放到了系统库里面。

2）取得 map 的 fd, bpf_obj_get 会直接调用 bpf 的系统调用。

3）将 fd 包装成 BpfMap，android 在 BpfMap.h 中定义了很多方便的函数。

4）遍历 map 回调函数。返回值必须是 android::netdutils::status::ok（在 android 的新版本中已经进行修改）。

### **5. 运行结果查看**

直接在目录下执行 mm，将编译出来的 bpf.o push 到 / system/etc/bpf 目录下，将统计程序 push 到 / system/bin 目录下，重启，看下结果。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjPmEFzMwJunJo5xKElWuxnK5wJHlQM0gvEhcb5tgR2aFj2dlu8ib3Ziao67LKd5trXWUpQS17aA1Pcg/640?wx_fmt=png)

前面的是 pid， 后面的是系统调用次数。

至此，如何在 android 平台使用 eBPF 实现统计系统中每个 pid 在一段时间内系统调用的次数的功能就介绍完了。

此外还有很多技术细节没有深入研究，不过毕竟只是初探，就先讲到这里了，后续有时间再进一步深入研究。研究的时间还是比较短，如果有任何错误的地方欢迎指正。

参考资料
----

eBPF 简史 （下篇）：

https://cloud.tencent.com/developer/article/1006318

goolge 原生使用 ebpf 的两篇文章：

https://source.android.com/devices/architecture/kernel/bpf

https://source.android.com/devices/tech/datausage/ebpf-traffic-monitor

BCC：

https://github.com/iovisor/bcc

![](https://mmbiz.qpic.cn/mmbiz_gif/d4hoYJlxOjM9WWBsVsUpiaGmAPiaTAJIsM8YMTErJrQy9vichOzuhB2BNSdKrKyQ0eOC0lYRVrPMLUJOvEOGto4Mg/640?wx_fmt=gif)

**长按关注**

**内核工匠微信**

Linux 内核黑科技 | 技术文章 | 精选教程