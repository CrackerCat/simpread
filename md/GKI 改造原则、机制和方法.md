> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/w1TvS9jjSQLvxxbnvzp0Ug)

Google 在 android11-5.4 分支上开始要求所有下游厂商使用 Generic Kernel Image（GKI），需要将 SoC 和 device 相关的代码从核心内核剥离到可加载模块中（下文称之为 GKI 改造），从而解决内核碎片化问题。GKI 为内核模块提供了稳定的内核模块接口（KMI），模块和内核可以独立更新。本文主要介绍了在 GKI 改造过程中需遵循的原则、遇到的问题和解决方法。

**一、不能破坏 KMI**

冻结 KMI 后，分支在其整个生命周期中都保持冻结状态，原则上不会接受破坏 KMI 的修改（除非发现严重的安全问题，且不能在不影响 KMI 稳定的情况下得到缓解）。在冻结的分支中，只有不破坏 KMI 的 bug 修复和 partner features 才能被接收。在不影响现有 KMI 接口的前提下，可以使用新导出的符号扩展 KMI，新接口被接收添加到 KMI 后，必须保持稳定，不能被将来的修改破坏。

**1. 问题现象：**

替换 google boot.img 后，开机串口有如下打印错误，

[1.669135] init: Loading module /lib/modules/foo.ko with args ""

[1.676281] foo: disagrees about version of symbol xxx

**2. 原因分析：**

出现这种错误可能的原因是 ko 中调用的符号与 vmlinux 中的符号 crc 值不匹配，如在 KMI 接口使用的结构体中增加了新字段，间接地修改了接口定义：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4f6w5GBTencQtBk3aYH0TicLw6TSudMM4eWmtD0N9xoYjfnDtMFPeCyA/640?wx_fmt=png)

**3. 推荐方法：**

(1) 扩展内核原生结构体和接口

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4xmROvOKPSibjnxicdQ66iarhEdBNWHUR10oicuBTYTel6DJrSXPGO3LbYA/640?wx_fmt=png)

(2) 向 google 申请，在内核原生结构体中加入一些 padding

Google 在 android11-5.4 分支中新增了两个宏

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4sZhDjJIgo1LpwOSo1vOqPdc9rGyRseaceJbXkamxfmjanDKTiabTQ3g/640?wx_fmt=png)

ANDROID_VENDOR_DATA，在结构体中保留一些 padding 以备将来可能的使用，这些 padding 正常的都是位于结构体尾部，padding 变量标识 n 从 1 开始递增。

ANDROID_VENDOR_DATA_ARRAY 同 ANDROID_VENDOR_DATA，分配一个数组，数组大小是 s，元素是 u64 类型。

下面是使用 ANDROID_VENDOR_DATA 和 ANDROID_VENDOR_DATA_ARRAY 在内核结构体中增加新字段的示例：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4YSX99KaeniaLza6OJicsp7txMnspcHSANI4SAmbGic0otZoHks2egStQg/640?wx_fmt=png)

**4. 措施：**

在编译脚本中增加检测，编译 GKI kernel 后，比较生成的 Module.symvers 和原生 android/abi_gki_aarch64.xml 文件中符号的 crc 值，如果 crc 值不匹配，编译报错，表示使用了非规则的方法修改原生接口或结构体。

**二、内核模块只能使用已 export 并添加到白名单中的接口**

android11-5.4 分支 build.config.gki.aarch64 文件中有如下配置

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj480ibfDZmibn4QcAubgBjqibRfPKL4VoAPkCtDBD5aOdRpicurYTmNRGv8g/640?wx_fmt=png)

表示模块只能使用 abi_gki_aarch64 文件中的符号。

**1. 问题现象：**

替换 google boot.img 后，串口有如下打印错误：

[1.735506] foo: Unknown symbol xxx(err -2)

**2. 原因分析：**

原生内核没有用 EXPORT_SYMBOL_GPL 把接口 xxx export，或已经 export 的接口没有添加到白名单（该接口 abi 可能不稳定）

**3. 推荐方法：**

(1)  Google 建议向上游 linux 社区申请将要使用的内核接口 export，并向 Google 申请，将接口添加到白名单 android/abi_gki_aarch64_xxx

（2）使用其他已在白名单中的接口替代。

**三、vendor hook 机制**

考虑到 SoC 和 OEM 厂商可能要对原生内核做一些客制化修改和优化，Google 提供了一套 vendor hook 机制，下游厂商在需要修改内核源码的地方添加 hook，并向 Google 申请，将 patch upstream 到 AOSP。

**1. vendor hook 实现步骤如下：**

(1) 在 include/trace/hooks / 目录下创建一个新的头文件 xxx.h，定义一组 hook 接口 register_trace_android_vh_xxx、trace_android_vh_xxx 和全局变量__tracepoint_android_vh_xxx

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4gianXFxtEBfksKZvMg1OiaGSUVgWjQAcb4DrhF437HUmicbCnN8b0BY7Q/640?wx_fmt=png)

(2) 在 drivers/android/vendor_hooks.c 文件中包含 hook 头文件 xxx.h，export hook 变量__tracepoint_android_vh_xxx 给模块使用

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4mtGepHeIHMN2W6VgYttBAHs1mklouxdkNLWP4QXoROxm0RVwatmgVg/640?wx_fmt=png)

(3) 在内核模块中增加 register 代码将回调函数绑定到 hook 变量__tracepoint_android_vh_xxx

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4gyBu9HmJ24BAqdnpsmGaKRm5KibE6ibf4QpOfCJebno4chDCpzphomjg/640?wx_fmt=png)

(4) 在内核 xxx.c 文件中包含 hook 头文件 xxx.h，调用 hook 接口 trace_android_vh_xxx(即 hook 变量__tracepoint_android_vh_xxx 绑定的 callback 函数)

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4JGY9YryjuicaFeuxXcRcQ5W0YxgpiceRB5zTDyS3jhldwar5FeXfH1icQ/640?wx_fmt=png)

**2. 问题现象：**

测试偶现 dump “BUG: scheduling while atomic:”

**3. 原因分析：**

vendor hook 变量有两种，都是基于 tracepoints 的：

正常的：使用 DECLARE_HOOK 宏创建 tracepoint 函数 trace_<name>，要求 name 在 trace 中是独一无二的，callback 函数的调用是在关抢占的场景中使用的

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4bjKq5OnWZcx9jcIHzyjnfAow50zXpelHKv8oNTVzBSBdr0am9yAibMw/640?wx_fmt=png)

受限制的：受限制的 hook 在 scheduler hook 类的场景中使用，绑定的 callback 函数可以在 cpu offline 或非原子上下文中调用（调用前没有关抢占），受限制的 vendor hook 不能被解绑定，所以绑定的模块不能卸载，只允许有一个绑定（任何其他绑定将会返回 - EBUSY 错误）。

**4. 推荐方法：**

根据使用场景选择适合的 vendor hook 变量，在可能会调度的场景需要使用受限制的 vendor hook

**四、vendor hook 延伸**

SoC 和 OEM feature 都要从内核剥离出来编译成内核模块，内核源码中相互调用 export 接口是没有问题的，那么模块之间相互调用 export 接口呢？

**1. 问题现象：**

编译报错 depmod: ERROR: Found 2 modules in dependency cycles!

**2. 原因分析：**

模块之间相互调用 export 接口，导致编译时报错。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4b9kb7OubMsYx3b4G30fYyuEtCIsGDzL60icezdhFxm5icGf9O5zJpfiag/640?wx_fmt=png)

**3. 推荐方法：**

借鉴 Google 的 vendor hook 机制，在 A 模块中定义并 export 全局变量，B 模块初始化函数中将 callback 函数注册绑定到该全局变量，这样只有 B 模块调用 A 模块中的变量和接口，A 模块通过 hook 变量回调 B 模块中接口，解决编译调用问题。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4NcJleRNUkkc3oPo48SiaU2FzpVLRg9JT74QfRfkGxVgfFPZpmflPwZw/640?wx_fmt=png)

**五、通过内核已有的事件注册接口替代 vendor hook**

是不是只能通过 vendor hook 实现修改内核呢？

android11-5.4 分支 log 中有这么一笔关于 vendor hook 的提交：

为按键组合增加 vendor hook

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4ibRvn2R4g6jFCYn0bzibWNZhM3MibThopVSPt1AhCuiba3lFHmX7ODrEww/640?wx_fmt=png)

我们注意到内核中有一个接口 input_register_handle，它的注释是注册一个新的 input handle，添加到设备和 handle 列表中，只要使用 input_open_device() 接口打开设备，输入事件触发后会轮询到该 handle，input_register_handle 接口应该在 handler 的 connect 方法中调用，因此我们可以使用 input_register_handler、input_register_handle 来实现上述的按键组合功能，不需要在内核中增加 vendor hook。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOXpO5Oow2ONIPkvibSpjjj4y2Ud3l0ibIowJH4uyrE4bF6qAjJaxKic85l8VGvTqeujBoxBWWqibYUTg/640?wx_fmt=png)

通过这个改造示例，启发我们思考是否只能使用 vendor hook 机制，是否可以使用其他内核已有的机制、事件注册接口将原先嵌入在内核中的 feature 剥离出来。

参考文章

[1]  https://source.android.com/devices/architecture/kernel/generic-kernel-image

[2]  https://blog.csdn.net/geshifei/article/details/94360470

[3]  https://blog.csdn.net/qq_39937242/article/details/82631165

![](https://mmbiz.qpic.cn/mmbiz_gif/d4hoYJlxOjM9WWBsVsUpiaGmAPiaTAJIsM8YMTErJrQy9vichOzuhB2BNSdKrKyQ0eOC0lYRVrPMLUJOvEOGto4Mg/640?wx_fmt=gif)

**长按关注**

**内核工匠微信**

Linux 内核黑科技 | 技术文章 | 精选教程