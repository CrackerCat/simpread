> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270799.htm)

> [原创] 某 DEX_VMP 安全分析与还原

目录:
===

> 一. 思路整理  
> 二. 某 VMP 入口特征  
> 三. 定位 VMP 字节码  
> 四. 分割 VMP 字节码  
> 五. 还原为 SMALI  
> 六. 攻击面总结  
> 七. 深入 VMP 还原的一些问题  
> 八. 调试与工具总结

一. 思路整理
=======

#### 还原 VMP 需要哪些铺垫?

> (1) 定位 VMP 字节码  
> (2) 分割 VMP 字节码  
> (3) 还原成 SMALI

#### (1) 为什么要找 VMP 字节码的位置?

> 因为如果目标方法的字节码地址, 都找不到, 还原也就没法展开了.

 

![](https://p2.ssl.qhimg.com/t01d54c17de98df4082.png)

#### (2) 为什么要分割 VMP 字节码?

> 如果要反汇编成 smali,  
> 起码要知道这条 smali 对应的字节码一共几个字节.
> 
> 在确定一条指令占几个字节后,  
> 还要知道这几个字节中,  
> 谁是操作码, 谁是操作数.

 

![](https://p4.ssl.qhimg.com/t018da0057fa3a227df.png)

#### (3) 还原为 SMALI

> 有了前两步铺垫, 最终我们可以解读一条完整的 smali 的含义.

 

![](https://p4.ssl.qhimg.com/t018676e985055bb954.png)

二. 某安卓 VMP 入口特征 (2021.8 月样本)
============================

> 跳板方法

 

![](https://p5.ssl.qhimg.com/t0197f8574a18a7a1db.png)

> 进入 native 后的参数处理逻辑

 

![](https://p2.ssl.qhimg.com/t0158d91c15664cceb6.png)

> 为了处理不同类型的返回值, 定义了多个 jni 方法

 

![](https://p1.ssl.qhimg.com/t01afa39bf34f44bd5e.png)

> 对应 jni 函数入口指令情况

 

![](https://p5.ssl.qhimg.com/t013527a90498db7307.png)

三. 定位 VMP 字节码
=============

#### 逻辑

![](https://p5.ssl.qhimg.com/t01c49a9ce57118745f.png)

> 根据上述逻辑, 则一定存在函数 F, 向 F 输入 index 可得到对应 codeitem_addr  
> F(index) == codeitem_addr
> 
> 我们看一下这个函数, 从 index 到 codeitem_addr 的过程  
> (0x2dce->0xcac85880)

 

![](https://p5.ssl.qhimg.com/t01324c6eb0cdbbba29.png)

 

![](https://p5.ssl.qhimg.com/t01e047183152ed9eaf.png)

#### 如何在十几万数量级的汇编中定位到这段代码的?

> 通过 Trace 记录 REG 信息,  
> 用到了两个关键数值, 0x2dce(index) 与 0xcac85880(codeitems),  
> 标记两个数值出现的中间区间即可.

 

![](https://p4.ssl.qhimg.com/t01957070c6763d8afe.png)

 

![](https://p2.ssl.qhimg.com/t011ea03bc08954efa1.png)

#### 展开上面的定位方式的两个前提条件:

> 我们已经有了关键数据 0x2dce, 但还需要知道另一个提前条件,  
> 即 codeitem 是 0xcac85880, 所以这个信息是从哪得知的?  
> 这里是本章的关键.

#### 如何分析出 codeitem 的地址是 0xcac85880?

> (1) 已知明文  
> (2) 沙箱日志获取切入点  
> (3) JNI 参数回溯  
> (4) 内存访问统计

#### (1) 已知明文

> 目标 APP 内很多的 onCreate() 方法, 其内部普遍调用了,  
> NBSTraceEngine.startTracing(); 以及 super.onCreate()

 

![](https://p5.ssl.qhimg.com/t01c04dc08b2adede1b.png)

> 我们选一个被 vmp 保护了的 onCreate() 作为分析目标, ZxWebViewActivity.onCreate()  
> ![](https://p5.ssl.qhimg.com/t01348cdf137ba8b5c0.png)

#### (2) 沙箱日志获取切入点

> ① ZxWebViewActivity.onCreate 内必定存在 NBSTraceEngine.startTracing(); 以及 super.onCreate()  
> ② startTracing 为静态方法, 会被编译器编译为 invoke-static  
> ③ super.onCreate() 为超类调用, 会被编译器编译为 invoke-super  
> ④我们猜测 vmp 对 invoke-static 模拟实现借助了 JNI 函数,  
> 所以我们触发 ZxWebViewActivity.onCreate() 执行, 截取其调用序列, 效果如下:

 

![](https://p3.ssl.qhimg.com/t01237c28dc561357a3.png)

> 大致逻辑为

 

![](https://p4.ssl.qhimg.com/t01e59852558eba525f.png)

 

![](https://p4.ssl.qhimg.com/t01a8ad1ce81367906a.png)

#### (3) JNI 参数 startTracing 来源回溯

> 我们在 trace 中找到这条 GetStaticMethodID() 的出现位置,  
> 然后作为起点向上展开回溯, 希望找到其参数”startTracing” 的最早出处,  
> 如果有自动化的脚本和条件可进行污点分析, 由于逻辑不是很复杂, 这里人工回溯完成.

 

![](https://p4.ssl.qhimg.com/t011b91cfc4429511e8.png)

> 具体过程省略……  
> 在 trace 中对参数”startTracing” 来源进行一番回溯,  
> 最终发现了一个起到决定性作用的偏移值 0x000081de.  
> 可以简单理解成, 它以 base+0x000081de 的形式确立的参数”startTracing”.
> 
> 结论:  
> 如果 0x000081de 是那个起到决定性意义的数值,  
> 那么毫无疑问 0x000081de 来自 codeitem.
> 
> 在 trace 中找到 0x81de 的出现位置,  
> 发现它来自于内存位置 0xcac858a8.

 

![](https://p3.ssl.qhimg.com/t01b994657e0660febb.png)

#### (4) 内存访问统计

> 0x81de 来自 0xcac858a8,  
> 由于这个地址可能是 codeitem,  
> 因此我们检索一下, trace 中对这片内存区域的访问情况  
> 0xcac858a8 取前 5 个高位, 忽略后 3 个地位, 即检索对 0xcac85??? 的访问

 

![](https://p5.ssl.qhimg.com/t01a3a3ca0f8b6dff9e.png)

> 找到 19 条指令, 而对 0xcac85??? 的访问, 最早的第一条指令, 出现在编号 5691 的位置,  
> 对应的内存地址为 0xcac85890, 说明这里是 ZxWebViewActivity.onCreate() 第一条字节码.
> 
> 由于 codeitem 第一条字节码之前 0x10 个字节还存在一些固定内容,  
> 所以 0xcac85890-0x10 取得 codeitem 地址 0xcac85880,  
> 即 codeitem 的地址是 0xcac85880

 

![](https://p2.ssl.qhimg.com/t01e01095fb78966ca5.png)

 

![](https://p5.ssl.qhimg.com/t014578881e6eabd13f.png)

四. 分割 VMP 字节码
=============

![](https://p2.ssl.qhimg.com/t0135ea25099af11987.png)

> 现在已经有了某厂 vmp codeitems 全部内容,  
> 但是还没法反汇编成 smali,
> 
> 因为还不知道,  
> 第一条指令一共占几个字节,  
> 第二条指令一共占几个字节,  
> 依次......
> 
> dalvik 指令是不等长,  
> 反汇编成 smali 的话,  
> 起码要知道这条 smali 对应的字节码一共几个字节  
> 在知道了每条指令占几个字节后,  
> 还要知道这几个字节中,  
> 谁是操作码, 谁是操作数.

#### 通过观察 codeitem 的内存段的读取情况, 可以达到这个目的

![](https://p5.ssl.qhimg.com/t010a76420d8a33b569.png)

 

![](https://p3.ssl.qhimg.com/t011eab621265748e0f.png)

#### 如何快速区分出操作码和操作数?

> 一般 opcode 后面会有一个 EOR 解密指令,  
> 以及一串类似定位 handle 的 CMP 指令操作,  
> 而 operand 没有, 这就为区分 opcode 和 operand 提供了特征依据.  
> ![](https://p0.ssl.qhimg.com/t01d72a7c9bc18a0d46.png)

 

![](https://p3.ssl.qhimg.com/t018ca3d60d7a82deab.png)

#### opcode 解密逻辑?

> 由 eor 指令向上回 key 出现的位置,  
> 即可确定 key 的来源,  
> 以及解密逻辑.
> 
> 大致逻辑:  
> off1 = sub(codeitem 当前指令地址, codeitem 基址)  
> off2 = lsl(off1, 1)  
> key = load(base + off2)  
> de_opcode = xor(en_opcode, key)

五. VMP 字节码还原为 SMALI
===================

> 1 标准 dalvik 指令反汇编过程  
> 2 VMP 指令反汇编过程  
> 3 还原 VMP 所有指令需要什么?  
> 4 没有 opcode 对照表时, 如何展开还原?

#### 1 标准 dalvik 指令反汇编过程

![](https://p4.ssl.qhimg.com/t01275f0ec42fbfc6f6.png)

 

![](https://p3.ssl.qhimg.com/t01907ec8f14061fc4e.png)

#### 2 VMP 指令反汇编过程

> 由于使用了已知明文条件作为切入点,  
> 已知分析目标 ZxWebViewActivity.onCreate() 中,  
> 必定会调用 startTracing() 方法,  
> 即必定存在 invoke-static {v0}, method@00da6f // ...startTracing
> 
> 又通过上面的分析得知关键值 81de 出现在这条 invoke-static 中,  
> 且充当操作数的角色, 那么按照我们按照标准 invoke-static 反汇编规则进行解析,  
> 就可以得到结论.

 

![](https://p4.ssl.qhimg.com/t0116a53404b9b9bb24.png)

 

![](https://p2.ssl.qhimg.com/t01e2ecf04333b6d9e2.png)

 

![](https://p0.ssl.qhimg.com/t01a6b224830fee198b.png)

 

.

> VMP 指令由标准指令基础上修改而来, 有哪些异同?

 

![](https://p0.ssl.qhimg.com/t015a0e146bfc48afc3.png)

#### 3 还原 VMP 所有指令需要什么?

![](https://p3.ssl.qhimg.com/t012001681e5321c48e.png)

#### 4 没有 opcode 对照表时如何展开还原?

> (1) 接口猜测法  
> (2) 参数推导法  
> (3) 标准 dalvik 指令格式的信息利用  
> (4) 人肉逆向法 (略)

#### (1) 接口猜测法

> method 相关的 invoke 系列指令, 可以通过 JNI 执行情况猜测.  
> Field 相关的 get set 系列指令, 也可以通过 JNI 执行情况猜测.

 

![](https://p5.ssl.qhimg.com/t010a9108825a7b9ab7.png)

#### (2) 参数推导法

> 方法调用前, 会先准备参数,  
> 通常是声明类型的指令,  
> 可以很大程度缩小猜测的候选指令范围.

 

![](https://p3.ssl.qhimg.com/t0175d34eb0896a6b83.png)

#### (3) 标准 dalvik 指令格式的信息利用

> 由于 vmp 指令是由 dalvik 标准指令略微修改 / 变异而来,  
> 只做了较小的改动, 仍然保留了 BIT 位分布特征这样信息.  
> 在做还原时, 可以利用这些信息, 一定程度缩小候选范围.  
> https://source.android.com/devices/tech/dalvik/instruction-formats  
> https://source.android.com/devices/tech/dalvik/dalvik-bytecode#instructions  
> ![](https://p2.ssl.qhimg.com/t01eb6677758f1f6985.png)

六. 攻击面总结
========

> 1 分析路径  
> 2 攻击面总结 && 启示

#### 1 分析路径

![](https://p1.ssl.qhimg.com/t01beb1b7edc5511057.png)

#### 2 攻击面总结 && 启示

> (1) 被 VMP 的方法内部存在已知明文指令.  
> (2) VMP 的实现高度依赖 JNI 函数, 通过 HOOK 拿到其调用信息, 是非常有效的切入点与突破口.  
> (3) codeitems 的连续性, 集中存储的特性, 通过内存访问统计最终被发现.  
> (4) vmp 指令由标准 dalvik 指令基础上略改而来, 整体仍然保留了很多可用信息,  
> 对于一些内部逻辑比较简单的方法, 可以以较小成本还原.

#### (1) 被 VMP 的方法内部存在已知明文指令.

![](https://p3.ssl.qhimg.com/t0126fb03e0071e2d9c.png)

#### (2) VMP 的实现高度依赖 JNI 函数, 通过 HOOK 拿到其调用信息, 是非常有效的切入点与突破口.

![](https://p3.ssl.qhimg.com/t0190824bdd1706c246.png)

#### (3) codeitems 的连续性, 集中存储的特性, 通过内存访问统计最终被发现.

![](https://p0.ssl.qhimg.com/t011c574051cf97351e.png)

#### (4) 某 vmp 指令由标准 dalvik 指令基础上略改而来, 整体仍然保留了很多可用信息

![](https://p4.ssl.qhimg.com/t017f0158396c633443.png)

七. 深入 VMP 还原的一些问题
=================

> 略

八. 调试与工具总结
==========

#### 核心问题:

> 获取程序完整的执行 && 数据信息 (trace).

#### 目前公开的主流的获取 trace 的方案:

> 1 GDB 调试  
> 2 FridaStalker 编译执行  
> 3 脱机 unicorn 模拟执行

#### 主流的获取 trace 的方案的弊端和缺陷:

> 1 IDA / GDB:  
> 速度极慢, 且会遭遇反调试
> 
> 2 FridaStalker  
> 不支持 arm 指令的 thumb 模式, 且 BUG 多,  
> 遭遇 vmp.so 中的花指令时, 基本无法正常使用.
> 
> 3 PC 上脱机 unicorn 模拟执行  
> vmp.so 中存在大量 jni call 和 system call, 需要手动实现它们, unicorn 才能完成运行.

#### 基于以上问题的尝试:

> 实现原始 APP 进程环境 && 原始 context 中,  
> 通过 unicorn 构造虚拟化 CPU,  
> 执行目标 function, 获得 trace,  
> 无已知检测和对抗手段, 简单过 anti.  
> ![](https://p4.ssl.qhimg.com/t01a4d0f5a0aa36a6a7.png)

#### 基于 trace 进行离线分析:

> 1 trace 形态可视化  
> 文本 / json / 数据库 / EXCEL 可视化表格 / 动态 CFG 图
> 
> 2 基本的分析  
> 地址含义解析 调用符号识别
> 
> 3 程序分析  
> 污点分析 相似性分析等..

 

![](https://p2.ssl.qhimg.com/t01ffa9dbf94a4552fe.png)

 

![](https://p5.ssl.qhimg.com/t012701f8a6ed809cfb.png)

 

![](https://p0.ssl.qhimg.com/t012fe36d1074c236fe.png)

 

![](https://p2.ssl.qhimg.com/t0161b2615808cb40bf.png)

2021 KCTF 秋季赛 第十题 《生命的馈赠》解题分析

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 3 小时前 被爱吃菠菜编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)