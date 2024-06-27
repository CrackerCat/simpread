> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282300.htm)

> [原创] 某短视频虚拟机分析和还原

前言
==

在 windows 流行的时候，虚拟机保护是多人不敢碰的东西现在依然也是如此，pc 的性能比移动端性能要高出不少，虚拟化和变异的代码多到令人发指，因此在加密保护强度上要比移动端要强很多很多，为了移动端 App 更好的体验 (ANR 率) 移动端加密强度短时间内不会达到 pc 上的强度，随着移动 cpu 性能越来越好相信加密强度会逐年加强。

早年兴趣使然分析研究过 windows 端 VMProtect、Safengine Shielden、Themida、VProtect、Enigma Protector 等等虚拟机，最近发现国内流行的短视频也有虚拟机加密同时也比较感兴趣，便开始了我的分析之旅。

分析任何虚拟机必须要扣汇编指令级细节。

准备
==

安卓诞生这么多年了至今没有像 windows 端 olldbg、x64dbg 那样友好的调试器，IDA PRO 虽然自带了安卓调试器总是没有相像中的稳定。lldb 作为移动端 iOS 和 android 开发的御用调试器，带源码调试在开发环境中还算比较友好，而汇编级调试只能输入命令行了，这是很多用惯了 gui 调试器的人接受不了的，但是个人发现 lldb 调试稳定性出奇的好，功能上比 IDA Pro 的安卓调试器强大太多了。

工具 && 参考文档
----------

*   静态分析工具  
    IDA Pro  
    IDA Pro 中的变量和标签命名约定: 1. 无下线为精确的名字，已经非常了解代码的功能和含义，例如："xxx"。2. "_" 单下划线定义为代码的功能和含义比较模糊，例如："_xxx"。3. 双下划线为分析的代码模糊不清，由于地址不好记忆但需要命名以区分，例如："__xxx"。
*   动态分析工具  
    lldb
*   ARM 参考文档  
    《Arm Architecture Reference Manual》  
    《ARM Cortex-A Series Programmer's Guide for ARMv8-A》
*   其他工具  
    CyberChef

虚拟机分析
=====

libEncryptor.so 一共包含了三套虚拟机，三套虚拟机各自独立并且代码一模一样，本文重点只分析 **vm2 虚拟机**。  
![](https://bbs.kanxue.com/upload/attach/202406/271698_W47SWUXQ4TF7RRC.webp#size=800x336)  
虚拟机**指令编解码**参考借鉴了 arm64 的一部分规则，并实现了自己的一套规则，在后面的解码分析中会有很多和 arm64 解码相似的地方。  
另外虚拟机并没有像 VMProtect 那样将一条指令分割成多条 "微指令" 的方式，此虚拟机没有把当前真实的上下文放到虚拟机上下文去模拟执行，而是运行了一套自己单独的上下文。

1.  `vm1`: 0xDA0  
    在 JNI_OnLoad 中被调用，该虚拟机起始位置在 0xDA0，主要作用解密是注册 java native 所需要的字符串和调用 RegisterNatives 进行注册。
2.  `vm2`: 0x2AC4  
    虚拟机起始位置在 0x2AC4，是 java 注册的原生接口 com.bytedance.frameworks.encryptor.EncryptorUtil.ttEncrypt(byte[] buff, int size)，用来加密 buff 字节数组。
3.  `vm3`: 0x444  
    代码位置在 0x444，用于生成 aes-128 cbc 算法的 key 和 iv。

在函数中调用虚拟机时会传入一个指针数组类型参数变量，这是传入到虚拟机入口的唯一参数。

```
void vm_entry(*int args)

```

c 伪代码来表示函数调用虚拟机入口

```
void ttEncrypt(char* buff, int buff_size) {
    int dst_size = size ＋0x76;
    char* pDstBuff = malloc(size ＋0x76);
    vm_entry(buff, size, pDstBuff, &dst_size);
}

```

反汇编版本：  
![](https://bbs.kanxue.com/upload/attach/202406/271698_72W97PV6F3V6RKQ.webp#size=880x407)

入口
--

IDA Pro 查看入口的 cfg 图，复杂程序看似很难其实一点都不简单，话说回来 cfg 看起来和 ollvm 的混淆平坦化非常相似，其实和 ollvm 混淆关系不大，只不过一部分的 switch 被拉平了，在了解调度逻辑后分析也不算复杂。  
在代码中依然能看到两个 ollvm swtich var 变量，其作用没有详细分析，但整个 cfg 图确定与 ollvm 关系不是很大，猜测开启 ollvm 后性能会大幅下降影响 app 启动速度了。  
![](https://bbs.kanxue.com/upload/attach/202406/271698_36EHQ5S4G2Y4YKQ.webp#size=920x520)  
在进入虚拟机运行时前，在入口需要准备虚拟机所需的内存和参数。对虚拟机内存布局情况必须了解如指掌，这样在动态和静态分析时才不会迷失方向。  
![](https://bbs.kanxue.com/upload/attach/202406/271698_22JYYSFF3NQPGRR.webp#size=960x610)

*   分配空间  
    入口首先会分配堆栈空间，真实堆栈空间中包含了参数占用、虚拟机运行时的上下文 (context) 和虚拟机堆栈的所使用的空间，  
    从上图代码中看出虚拟机堆栈起始位置位于 sp+0x38，结束位置 sp+0x4D8，大小 4D8-0x38=4A0，虚拟机所有可用真实内存为 0x4A0，其中上下文 (context) 使用 0x150，剩余分配给虚拟机堆栈 4A0-150=2C0。  
    vm_entry 调用 vm2_run 时的堆栈内存布局情况:

<table><thead><tr><th>SP</th><th>类型</th><th>变量名</th><th>注释</th></tr></thead><tbody><tr><td>0x0</td><td></td><td></td><td>未使用</td></tr><tr><td>0x8</td><td>char *</td><td>pSrcBuffer</td><td></td></tr><tr><td>0x10</td><td>int</td><td>srcSize</td><td></td></tr><tr><td>0x18</td><td>char *</td><td>pDstBuffer</td><td></td></tr><tr><td>0x20</td><td>int *</td><td>pDstBufferSize</td><td></td></tr><tr><td>0x28</td><td>void *</td><td>pCall_register_trampoline</td><td></td></tr><tr><td>0x30</td><td>void *</td><td>pVMMemoryEnd</td><td>偏移 0x510</td></tr><tr><td>0x38</td><td></td><td><strong>vmMemoryStart</strong></td><td>虚拟机运行时专用内存起始位置</td></tr><tr><td>...</td><td></td><td></td><td></td></tr><tr><td>0x510</td><td></td><td><strong>vmMemoryEnd</strong></td><td>虚拟机运行时内存结束位置</td></tr><tr><td>0x518</td><td></td><td></td><td></td></tr><tr><td>0x520</td><td>x28</td><td></td><td></td></tr><tr><td>0x528</td><td>x19</td><td></td><td></td></tr><tr><td>0x530</td><td>x29</td><td></td><td>上一个栈桢地址</td></tr><tr><td>0x538</td><td>x30</td><td></td><td></td></tr></tbody></table>

*   参数  
    在进入虚拟机正式执行前，vm2_entry 在进入虚拟机运行时前会对 5 个参数进行初始化。  
    `pOpcode`：opcode 指针指向虚拟机要执行的代码  
    `pArgs`：传入虚拟机的参数  
    `pReserve`：未使用，用途暂时未知  
    `pExternalFunc`：调用虚拟机外的函数列表，此列表是加密的。  
    `pVmData`：一个结构体指针，结构体存储了两指针分别是函数跳板地址另一个是虚拟机内存结束地址，{0x0: pCallRegisterTrampoline, 0x8: pVMMemoryEnd}

```
vm2_run(void* pOpcode,  void* pArgs,  void* pReserve,  void* pExternalFunc， void* pVmData);

```

*   opcode  
    vm2 的 opcode 起始地址在 0xB090，共占用 0x2D8 字节，每个 opcode 占用四个字节。
*   跳板函数  
    跳板函数是衔接适配虚拟机和外部函数以及参数的桥梁，它包含来自虚拟机传递的两个参数, 参数 1：x0 是调用跳转的一个外部函数地址，参数 2：x1 是外部函数需要使用的参数指针。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_9E3MA7TYBXQGVKT.webp#size=800x270)
*   外部函数  
    外部地址函数目标地址是被简单加密过的，在虚拟机指令中使用加法解密这 8 个地址，例如：0xDDD2D0(加密地址数据) + FF225870(key) = 0x2B40(目标地址)。  
    截图中 IDA Pro 以默认 0 基址的地址，在调试时看到的地址数据是被代码手动重定位后的数据。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_6XB3MV6G5PDB5B2.webp#size=800x400)

初始化
---

vm2_run 仅仅只分配了保存被调用者寄存器的堆栈内存空间，并没有分配空闲的堆栈内存，在虚拟机真实开始之前会将传递进来的 5 个参数即 0x-x4 对虚拟机中的虚拟寄存器和真实专用寄存器进行初始化。  
![](https://bbs.kanxue.com/upload/attach/202406/271698_4CFRTMMR9KCMKBG.webp#size=880x392)  
vm_run 还初始化了解码 opcode 的 switch 表，在初始化时发现一共初始化了 6 张 switch 表，当然在 handler 中还存在其他 switch 表，这么多表是如何来的？猜测编写时只有 1-2 张表，在编译器优化后表就被分割成多块了。  
![](https://bbs.kanxue.com/upload/attach/202406/271698_KVYTC36JHR2B54P.webp#size=800x284)

*   虚拟寄存器初始化  
    vm2_run 初始化时会将传递的 5 个参数赋值给虚拟寄存器，其中包括 PC 和 SP 的值。
    
    <table><thead><tr><th>虚拟寄存器</th><th>参数</th><th>注释</th></tr></thead><tbody><tr><td>x0</td><td></td><td>x0 始终为 0，XZR 寄存器?</td></tr><tr><td>x4</td><td>arg2: pArgs</td><td></td></tr><tr><td>x5</td><td>arg3: pReserve</td><td></td></tr><tr><td>x6</td><td>arg4: pExternalFunc</td><td></td></tr><tr><td>x7</td><td>arg4: pVmData-&gt;pCallRegisterTrampolineFunction</td><td></td></tr><tr><td>x29(SP)</td><td>arg5: pVmData-&gt;pVmMemoryLimit - 0x150</td><td>SP 的初始值</td></tr><tr><td>x31(LR)</td><td></td><td>初始值为 0，寄存器名不确定，vm 退出时保存退出代码编号</td></tr><tr><td>PC</td><td>arg1: pOpcode</td><td></td></tr></tbody></table>
*   真实专用寄存器  
    虚拟机运行时使用了真实虚拟器，其中包括临时寄存器和专用寄存器，临时寄存器保存多种类型的值，而专用寄存器在虚拟机从开始到退出只保存一种指定类型数据或恒定不变。  
    opcode 位域伪代码：  
    w12 保存了 4 位 32 字节的 opcode，在 opcode **首次解码**时，位域中的变量会放到真实寄存器。  
    位域伪代码示例：  
    `w12[26]`: 取第 26 位到放到入目标的 26 位  
    `w12[26->0]`: 取第 26 位并放入到目标的指定位  
    `w12[27-26->1-0]`: 取 27 位和 26 位放入到目标第 1 位和第 0 位  
    `|`: 按位或
    
    <table><thead><tr><th>真实寄存器</th><th>注释</th></tr></thead><tbody><tr><td>x0</td><td>temp/pcode</td></tr><tr><td>x1</td><td>temp, 保存 [x19-0x20] 的值某种流程控制</td></tr><tr><td>x2</td><td>temp</td></tr><tr><td>x3</td><td>temp</td></tr><tr><td>x4</td><td>虚拟寄存 x4，初始化时保存 pBufferInfo(x1)，虚拟机中保存跳板函数地址</td></tr><tr><td>x5</td><td>虚拟寄存 x5，初始化时保存数值为 0，虚拟机运行时跳板函数的参数指针</td></tr><tr><td>x6</td><td>ollvm 混淆 switch_var，初始值: 0x400000b，这个应该是 llvm 混淆的 switch var</td></tr><tr><td>w7</td><td>ollvm 混淆 switch_var，初始值: 0x200fff，这个应该是 llvm 混淆的 switch var</td></tr><tr><td>w8</td><td>w12[20-16] operand1 Xt/Xm,det register index?</td></tr><tr><td>w9</td><td>w12[25-21] operand2 Xn,src register index?</td></tr><tr><td>w10</td><td>w12[15-&gt;4] | w12[14-&gt;3] | w12[13-&gt;2] | w12[12-&gt;1] | w12[31-&gt;0],5 个位合并到低 5 位</td></tr><tr><td>w11</td><td>w12[30-&gt;4] | w12[29-&gt;3] | w12[28-&gt;2] | w12[27-26-&gt;0-1], 组成的低 5 位</td></tr><tr><td><strong>w12</strong></td><td><strong>32 位的 opcode</strong></td></tr><tr><td>w13</td><td>w12[31]</td></tr><tr><td>w14</td><td>w12[30]</td></tr><tr><td>w15</td><td>w12[29]</td></tr><tr><td>w16</td><td>w12[28]</td></tr><tr><td>w17</td><td>w12[27]</td></tr><tr><td>w18</td><td>w12[26]</td></tr><tr><td>x19</td><td>虚拟机下文<strong>负偏移指针</strong>指向可使用内存最高上限的地址与 vmMemoryLimit 值相同，使用<strong>负偏移</strong>对上下文进行访问。</td></tr><tr><td>x20</td><td>call_register_trampoline 跳板地址</td></tr><tr><td>x21</td><td>Context，上下文指针</td></tr><tr><td>x22</td><td>switch_table7</td></tr><tr><td>x23</td><td>switch_table6</td></tr><tr><td>w24</td><td>默认为 1</td></tr><tr><td>x25</td><td>switch_table5</td></tr><tr><td>x26</td><td>switch_table3</td></tr><tr><td>x27</td><td>switch_table_main</td></tr><tr><td>x28</td><td>switch_table2</td></tr><tr><td>x29(fp)</td><td>未使用</td></tr><tr><td>x30(lr)</td><td>switch_table_4(3A)</td></tr></tbody></table>
*   虚拟机 context  
    虚拟机中也有专用寄存器，在调用外部函数时，其中`x4`虚拟机中保存外部函数地址，`x5`虚拟机中保存参数指针，`x25`虚拟机中调用外部函数时的跳板地址。**另外虚拟寄存器和 aarch64 中的寄存器并不是一一对应的，在这里只是对每个虚拟寄存器启了一个相应的名字方便理解和记忆**。
    
    <table><thead><tr><th>负偏移</th><th>虚拟寄存器</th><th>注释</th></tr></thead><tbody><tr><td>-0x150</td><td></td><td>虚拟机堆栈 SP 初始位置</td></tr><tr><td>-0x148</td><td></td><td></td></tr><tr><td>-0x140</td><td></td><td></td></tr><tr><td>-0x138</td><td>pc</td><td>pc 指针，真实寄存器 x21 中的值指向此地址，handler 使用</td></tr><tr><td>-0x130</td><td>x0</td><td>初始化值: 0, 虚拟机中开始到结束始终为 0，XZR 寄存器?</td></tr><tr><td>-0x128</td><td>x1</td><td></td></tr><tr><td>-0x120</td><td>x2</td><td></td></tr><tr><td>-0x118</td><td>x3</td><td></td></tr><tr><td>-0x110</td><td>x4</td><td>初始化值：pBufferInfo，专用寄存器：虚拟机中调用外部函数时保存外部函数地址</td></tr><tr><td>-0x108</td><td>x5</td><td>初始化值: 0，专用寄存器：虚拟机中调用外部函数时保存参数指针</td></tr><tr><td>-0x100</td><td>x6</td><td>初始化值: vm2_external_func_list</td></tr><tr><td>-0xF8</td><td>x7</td><td>初始化值: call_register_trampoline_function</td></tr><tr><td>-0xF0</td><td>x8</td><td></td></tr><tr><td>-0xE8</td><td>x9</td><td></td></tr><tr><td>-0xE0</td><td>x10</td><td></td></tr><tr><td>-0xD8</td><td>x11</td><td></td></tr><tr><td>-0xD0</td><td>x12</td><td></td></tr><tr><td>-0xC8</td><td>x13</td><td></td></tr><tr><td>-0xC0</td><td>x14</td><td></td></tr><tr><td>-0xB8</td><td>x15</td><td></td></tr><tr><td>-0xB0</td><td>x16</td><td></td></tr><tr><td>-0xA8</td><td>x17</td><td></td></tr><tr><td>-0xA0</td><td>x18</td><td></td></tr><tr><td>-0x98</td><td>x19</td><td></td></tr><tr><td>-0x90</td><td>x20</td><td></td></tr><tr><td>-0x88</td><td>x21</td><td></td></tr><tr><td>-0x80</td><td>x22</td><td></td></tr><tr><td>-0x78</td><td>x23</td><td></td></tr><tr><td>-0x70</td><td>x24</td><td></td></tr><tr><td>-0x68</td><td>x25</td><td>虚拟机中的专用寄存器：虚拟机中调用外部函数时的跳板地址</td></tr><tr><td>-0x60</td><td>x26</td><td></td></tr><tr><td>-0x58</td><td>x27</td><td></td></tr><tr><td>-0x50</td><td>x28</td><td></td></tr><tr><td>-0x48</td><td>x29(SP)</td><td>虚拟机堆栈 SP, 初始化时指向 - 0x150</td></tr><tr><td>-0x40</td><td>x30</td><td></td></tr><tr><td>-0x38</td><td>x31(LR)</td><td>当值为 0 时退出虚拟机</td></tr><tr><td>-0x30</td><td></td><td></td></tr><tr><td>-0x28</td><td></td><td></td></tr><tr><td>-0x20</td><td>0</td><td>流程控制状态标记，值范围 0-3，0: 正常执行状态, 2 和 3: 程序中包含分支跳转某种流程, 3 和 1:call 某个地址函数</td></tr><tr><td>-0x18</td><td></td><td>new pc/call 某个地址函数的地址 / 为 0 时退出虚拟机</td></tr><tr><td>-0x10</td><td></td><td>LR, 在调用调用函数结束后返回虚拟机地址</td></tr><tr><td>-0x8</td><td></td><td>起始 pc 指针，一部分分支跳转指令参考的起始基址，例如: 跳转目标地址 = pc 起始地址 + offset</td></tr></tbody></table>

取指
--

pOpcode 取出 4 个字节的 opcode 并取得低 5 位解码出 op1。

```
# 初始化
LDP             X20, X19, [X4]              ;x20:call_register_trampoline,0x19:pVmMemoryLimit
SUB             X21, X19, #0x138        ;x19-0x138=x21计算出pc指针在负偏移指针的内存位置
STR             X0, [X21]                      ;保存pc到x21，此时x0和x21中的值相同并被关联。
# 取opcode
LDR            W12, [X0]                    ;取出4字节opcode
AND           W10, W12, #0x3F       ;opcode的低5位为op1
CMP           W10, #0x3F                ;op1的值最大只能小于63，说明op1只有0-63一共64个
B.HI            next_pc                      ;下一条指令

```

![](https://bbs.kanxue.com/upload/attach/202406/271698_U38SV7SDN5SMYT6.webp#size=880x590)

解码和分发
-----

### 首次解码

在 opcode **首次解码**时会解码 4 个寄存器操作数，从中提取 4 个位域的字段，分别保存到**真实专用寄存器** w8、w9、w10、w11，在 arm64 指令集中当指令是 MADD、MSUB、UMADDL、UMSUBL、SMADDL、SMADDL 等会有 4 个寄存器操作数的情况，在这里同样也是如此。  
首次解码时位域布局：w12[31,30-26,25-21,20-16,15-12,11-6,5-0]  
w12[5-0:6] = op1  
w12[11-6:6] = op2  
w12[15-12:4]|[31:1] = Xm/Wm  
w12[20-16:5] = Xt/Wt  
w12[25-21:5] = Xn/Wn  
w12[30-26:5] =Xa/Wa  
`w8`: Rd(Xt/Wt) 目标寄存器操作数，w12[20-16->5-0]，取出 16-20 位保存最低位，5 位可以表示 0-31 个寄存器。  
`w9`: Rn(Xn/Wn)(第一个源寄存器操作数，w12[25-21->5-0]，取出 21-25 位保存最低位，5 位可以表示 0-31 个寄存器。  
`w10`: Rm(Xm/Wm) 第二个源寄存器操作数，w12[15->4] | w12[14->3] | w12[13->2] | w12[12->1] | w12[31->0]，5 位可以表示 0-31 个寄存器。  
`w11`: Ra(Xa/Wa) 第三个源寄存器操作数，w12[30->4] | w12[29->3] | w12[28->2] | w12[27-26->1-0]，5 位可以表示 0-31 个寄存器。  
寄存器操作数伪代码表示如下:  
`操作数大小`: X 为 64 位操作数、W 为 32 位操作数。  
`目标操作数`: d=destination register, t=target register。  
`源寄存器`：第一个源寄存器为 n，第二个寄存器为 m，第三个寄存器为 a。  
`R`: 表示寄存器 64 或 32 位操作数，X 是 64 位操作数、W 代表 32 位操作数。  
![](https://bbs.kanxue.com/upload/attach/202406/271698_BS5V7574FHPNR35.webp#size=880x323)

### 分发和二次解码

在高级语言中如果一个 switch 太多，在某些编译器编译优化后出现在多张 switch 子表和子表的子表的情况，对于一些相同代码的多个 case 会合并到一个 case 中再次 switch 分发的情况。  
根据编译器的优化编译的特性，在处理 switch 的 case 为了减少查找次数找到最终的 case，在代码中会经常看到大于 (GT)、小于(LE) 等分支跳转，看到这个不要迷惑，这是编译器优化 case 的结果，这样做的目的是减少查找次数使用了类似二分查找的算法，当看到 B.EQ 的跳转目标和 B.NE 的下一条指令就是匹配到了 case 常量了。  
在**首次解码**后分发过程中通常还会有**二次解码**，除了 MADD 和 MSUB 等等指令有 4 个寄存器操作数，指令只需一般指令只有 2-3 个寄存器操作数，一条完整的 arm64 指令至少需要二个寄存器操作数，这里也同样如此，当指令只有二个寄存器操作数时即一个目标操作数 Xt 和第一个源操作数 Xn，其他的位域字段就会空闲下来，例如：op2、Xm、Xa，在二次解码时这些空闲的位域字段原有的值就会覆盖被再次利用组成其他寻址方式例如：shift、extend、imm 等等。  
在虚拟机指令 op1 的值是 11(0xb) 时，分发处理会解码 op2，解码 op2 时 Xt、Xn、Xm 等操作数位置会发生变化，原有的 x10/w10 第三个源操作数在 op2 中变成目标操作数，第一个源操作数变成 x8/w8，第二个源操作数变成 x9/w9，各操作数的位域解码方式不变，变的只是操作数角色。  
`Xt`：Rd(Xt/Wt) 目标寄存器操作数 (x10/w10)，w12[15->4] | w12[14->3] | w12[13->2] | w12[12->1] | w12[31->0]，5 位可以表示 0-31 个寄存器。  
`Xn`: Rn(Xn/Wn)(第一个源寄存器操作数 (x8/w8)，w12[20-16->5-0]，取出 16-20 位保存最低位，5 位可以表示 0-31 个寄存器。  
`Xm`: Rm(Xm/Wm) 第二个源寄存器操作数 (x9/w9)，w12[25-21->5-0]，取出 21-25 位保存最低位，5 位可以表示 0-31 个寄存器。

执行
--

从 op1 的取值范围为 0-63 一共 64 条指令，当 op1 是 11 时还有存在 op2 和 op3 的情况，由于虚拟机的 handler 太过庞大分析所有的指令太过耗时，我们目的是还原代码逻辑，只需分析执行过的 handler，对于没有执行过的 handler 放弃分析。  
在了解了 op1 解码方式后，通过脚本得到执行次数最多的指令并优先分析：

```
import struct
pcode_start = 0xB090
pcode_size = 0x2D8
with open("libEncryptor.so", "rb") as f:
        content = f.read()
bytes_code = content[pcode_start : pcode_start + pcode_size]
insts_sets = {}
for pc in range(0, pcode_size, 4):
    word = struct.unpack(" 0:
        opcode = word[0]
        op1 = opcode & 0x3F
        if insts_sets.get(op1, None) == None:
             insts_sets[op1] = 1
        else:
             insts_sets[op1] += 1
sorted_dict = dict(sorted(insts_sets.items(), key=lambda item: item[1], reverse=True))
for inst, count in sorted_dict.items():
     print(f'op1: {inst}, count: {count}') 
```

得到执行次数最多的指令：

```
op1: 23, count: 56
op1: 11, count: 51
op1: 40, count: 38
op1: 21, count: 27
op1: 24, count: 2
op1: 12, count: 2
op1: 17, count: 2
op1: 48, count: 2
op1: 52, count: 1
op1: 7, count: 1

```

从 python 打印的结果来看 23、11、40 前面三个指令使用最为频繁，因篇幅关系只分析这三条指令，其中 op1 值是 11 的还存在第二个操作码 op2，这里选择 op1=11 & op2=12 的指令进行分析。

### op1: 23

*   取指  
    在循环的开始，首先判断一个虚拟机状态控制码，指示了当前指令执行完成是否进入某些分支跳转指令流程，接着取出 4 个字节并判断指令 op1 的合法性，如果不合法忽略当前指令直接跳转 B.HI next_pc 执行一条指令。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_FKZXJSA5HTZBP24.webp#size=880x285)
*   解码  
    同样首次解码出四个寄存器操作数，w8 = 目标操作数 Xt，w9 = 第一源操作数 Xn，w10 = 第二个源操作数 Xm，w11 = 第三个源操作数 Xa。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_GT8VYTMHTA9H4YW.webp#size=880x299)
*   分发 & 解码 2  
    既然 op1=23，那么 switch 会跳转到主分发表的 case 23 的位置，从图中看出 20，23，27，35，36，48，54，58 多个 case 合并的到一个的 case，可能是这些的代码是相同的被合并到一个地方了。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_54WJGU4GUNWVPCM.webp#size=920x384)  
    2E50`: 代码开头4行代码是判断op1的范围是否20-58之间来确定case的合法性，代码中使用了首次解码的寄存器w10并被直接重新赋值使用了，首次解码时是第二个源寄存器Xm，说明此条指令可能没有第二和第三个源寄存器。` 2E60-2E8C`: 首次解码的寄存器 w11 是第三个源寄存器，同样被重新赋值给予新的含义了，此时有效的寄存器有 w12(opcode)、w8(Xt)、w9(Xn)、x21(pContext 上下文指针)，这段代码主要是做了三件事：

1.  获取 Imm16:  
    获取一个 16 位的立即数：w11 = w12[12-15] | w12[31->11] | w12[30->10] | w12[29->9] | w12[28->8] | w12[27->7] | w12[26->6] | w12[6-11->0-5]，解码过程的高级语言版本方便理解 (opcode & 0xF000) | (opcode >> 20 & 0xFC0) | (opcode >> 6 & 0x3F)

```
.text:0000000000002E60 060                 UBFX            W11, W12, #6, #6 ; w12[6-11->0-5]
.text:0000000000002E64 060                 AND             W12, W12, #0xF000 ; w12[12-15]
.text:0000000000002E68 060                 BFXIL           W12, W18, #20, #12 ; w12[12-15] | w12[26->6]
.text:0000000000002E6C 060                 ORR             W11, W12, W11 ; w12[12-15] | w12[26->6] | w12[6-11->0-5]
.text:0000000000002E78 060                 ORR             W11, W11, W17,LSR#20 ; w12[27->7]
.text:0000000000002E80 060                 ORR             W11, W11, W16,LSR#20 ; w12[28->8]
.text:0000000000002E84 060                 ORR             W11, W11, W15,LSR#20 ; w12[29->9]
.text:0000000000002E88 060                 ORR             W11, W11, W14,LSR#20 ; w12[30->10]
.text:0000000000002E8C 060                 ORR             W11, W11, W13,LSR#20 ; w12[31->11]


```

1.  获取源操作数 Xn 的值：  
    `2E70`：x21 是 pContext 指针，w9 是第一个源寄存器是一个偏移索引值，x9 = x21 + w9 * 8 得到相对的偏移植。  
    `2E7C`: 从上下文中取出第一个寄存器的值，x9 = x21 + w9 * 8 + 8。

```
.text:0000000000002E70 060                 ADD             X9, X21, W9,UXTW#3
.text:0000000000002E7C 060                 LDR             X9, [X9,#8]


```

1.  计算出一个偏移量：  
    第一个寄存器的值加上 imm16 扩展到 32 位的立即数得到一个偏移量。

```
2E94 060                 ADD             X9, X9, W11,SXTH ; w9=address,取出半字16立即数再相加


```

*   执行  
    再次分发后来到 case 23 的最终目标地：  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_BNAU7TNZV85DP43.webp#size=920x122)  
    x9：此时是一个偏移量  
    w8：是一个目标寄存器，它的值是一个索引值  
    `3380`: x21 是 pContext 指针，w8 是目标源寄存器是一个偏移索引值，x9 = x21 + w9 * 8 得到相对的偏移植。  
    `3384`: 取出目标寄存器的值，x8 = x21 + w8 * 8 + 8。  
    `3388`: 将目标寄存器存储到偏移量地址中，[Xn + imm]=Xt。

```
.text:0000000000003380     case_23__STR_Xt_Mem_Xn_simm
.text:0000000000003380 060                 ADD             X8, X21, W8,UXTW#3 ; jumptable 0000000000002E98 case 23
.text:0000000000003384 060                 LDR             X8, [X8,#8]
.text:0000000000003388 060                 STR             X8, [X9]


```

准备下一条指令: pc 指针地址加 4

```
338C 060                 B               next_pc


```

*   指令还原  
    储存指令：STR Xt, [Xn + imm16]  
    当 imm 为 0 时: STR Xt, [Xn]

### op1: 11 op2: 8

当 op1 值等于 11 时会解码第二个或第三个操作码。

*   取指  
    这里不再重复参考 op: 23 取指部分
*   解码  
    这里不再重复参考 op: 23 解码部分
*   分发 & 解码 2  
    11 号 case 首先会提取出 op2 对 op2 进行再次 switch case 分发处理。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_QCXQBM577CE3EP4.webp#size=920x159)  
    经过分发跳转跳转来到这里，op2 的多个 case 1,6,8,12,17,26,36,58 也被合并到一个位置。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_GBUXQFT9K48T2MK.webp#size=920x242)
*   执行  
    经过多次跳转来吧 op2=8 的最终目的地 case 8，上文提到过 op2 与 op1 的操作数位置不一样，Xn(w10)，Xn(w8)，Xm(w9)  
    `3D60`: 获取上下文中的寄存器偏移  
    `3D64`：取第二次源操作数值，x9=[x11+ w9 * 8]  
    `3D68`：取第一次源操作数值，x8=[x11+ w8 * 8]  
    `3D6C`：减法运算  
    `3D70`：将结果写入到目标寄存器，[x11+ w10 * 8] = x8  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_GQ66QHRD5YGKYF9.webp#size=920x219)
*   指令还原  
    SUB Xt, Xn, Xm

### op1: 40

*   取指  
    这里不再重复参考 op: 23 取指部分
*   解码  
    这里不再重复参考 op: 23 解码部分
*   分发 & 解码 2  
    在指令的开头 3 条指令中第二个 (w10)、第三个源寄存器(w11) 的值被重新赋值并使用，说明指令中有效寄存器操作数只有 Xt 和 Xn。  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_XE7H583DSH7HE92.webp#size=920x483)  
    经过分析二次解码和 op: 23 一模一样，获取立即数、获取源操作数的值、获取一个偏移值

1.  获取 imm  
    opcode 提取 imm16 位立即数:  
    w11 = w12[12-15] | w12[31->11] | w12[30->10] | w12[29->9] | w12[28->8] | w12[27->7] | w12[26->6] | w12[6-11->0-5]

```
.text:0000000000002D50 060                 UBFX            W11, W12, #6, #6 ; w12[6-11->0-5]
.text:0000000000002D54 060                 AND             W12, W12, #0xF000 ; w12[12-15]
.text:0000000000002D58 060                 BFXIL           W12, W18, #20, #12 ; w12[12-15] | w12[26->6]
.text:0000000000002D5C 060                 ORR             W11, W12, W11 ; w12[12-15] | w12[28->6] | w12[6-11->0-5]
.text:0000000000002D68 060                 ORR             W11, W11, W17,LSR#20 ; w12[27->7]
.text:0000000000002D70 060                 ORR             W11, W11, W16,LSR#20 ; w12[28->8]
.text:0000000000002D74 060                 ORR             W11, W11, W15,LSR#20 ; w12[29->9]
.text:0000000000002D78 060                 ORR             W11, W11, W14,LSR#20 ; w12[30->10]
.text:0000000000002D7C 060                 ORR             W11, W11, W13,LSR#20 ;


```

1.  获取源操作数的值

```
.text:0000000000002D60 060                 ADD             X9, X21, W9,UXTW#3
.text:0000000000002D6C 060                 LDR             X9, [X9,#8] ; << Xn


```

1.  计算一个偏移量

```
.text:0000000000002D84 060                 ADD             X9, X9, W11,SXTH ; X9 = Xn + imm16


```

*   执行  
    经过二次解码分析得出目前有效的真实寄存器有：x9 是源寄存器加上立即的一个偏移量，w8 是目标操作数索引值。  
    `3318`：偏移量的内存中取值  
    `3A70`：计算目标寄存器在 pContext 的偏移地址，x8= x21 + w8 * 8  
    `3A74`：将内存值放入目标寄存器

```
.text:0000000000003318 060                 LDR             X9, [X9] 
.text:0000000000003A70 060                 ADD             X8, X21, W8,UXTW#3
.text:0000000000003A74 060                 STR             X9, [X8,#8]


```

*   指令还原  
    加载指令：LDR Xt, [Xn + imm16]  
    当 imm 为 0 时：LDR Xt, [Xn]

还原
==

虚拟机执行框架
-------

分析完取指、解码、执行后大体得到了一个模糊的虚拟机执行框架，为了方便记忆和理解使用 c 伪代码来描述。  
![](https://bbs.kanxue.com/upload/attach/202406/271698_VKCTZRBYZANEPUE.webp#size=1000x993)

解码 op1 & op2
------------

使用脚本解析 opcode 的 op1 和 op2 打印所有需要分析的 handler。

```
import struct
pcode_start = 0xB090
pcode_size = 0x2D8
with open("libEncryptor.so", "rb") as f:
                content = f.read()
bytes_code = content[pcode_start : pcode_start + pcode_size]
insts_op1_sets = {}
insts_op2_sets = {}
for pc in range(0, pcode_size, 4):
        word = struct.unpack(" 0:
                opcode = word[0]
                op1 = opcode & 0x3F
                if op1 == 11:
                        op2 = (opcode >> 6) & 0x3F
                        if insts_op2_sets.get(op2, None) == None:
                                insts_op2_sets[op2] = 1
                        else:
                                insts_op2_sets[op2] += 1
                else:
                        if insts_op1_sets.get(op1, None) == None:
                                insts_op1_sets[op1] = 1
                        else:
                                insts_op1_sets[op1] += 1
 
print("op1指令统计:")
op1_sorted_dict = dict(sorted(insts_op1_sets.items(), key=lambda item: item[1], reverse=True))
for inst, count in op1_sorted_dict.items():
         print(f'op1: {inst}, count: {count}')
print("op2指令统计:")
op2_sorted_dict = dict(sorted(insts_op2_sets.items(), key=lambda item: item[1], reverse=True))
for inst, count in op2_sorted_dict.items():
         print(f'op1: 11, op2: {inst}, count: {count}') 
```

一共打印出 15 个 handler，好在需要分析还原的指令不多：

```
op1指令统计:
op1: 23, count: 56
op1: 40, count: 38
op1: 21, count: 27
op1: 24, count: 2
op1: 12, count: 2
op1: 17, count: 2
op1: 48, count: 2
op1: 52, count: 1
op1: 7, count: 1
op2指令统计:
op1: 11, op2: 7, count: 25
op1: 11, op2: 43, count: 14
op1: 11, op2: 12, count: 8
op1: 11, op2: 25, count: 2
op1: 11, op2: 39, count: 1
op1: 11, op2: 62, count: 1


```

还原脚本
----

在 15 个 handler 并还原后，现在重新编写脚本`decode_opcode.py`解码 opcode，将打印还原的指令打印出伪汇编代码并保存文件`ttencryptor.asm`和`generate_aes_key_iv.asm`。

```
def decode(pcode_start, pcode_size, liner_disasm = False):
        with open("libEncryptor.so", "rb") as f:
                content = f.read()
 
        print(f"{'pc':^8} {'指令':^8}  {'op1':^8}  {'op2':^8}\t{'助记符':^30}")
        print(f"{'-'*8}  {'-'*8}  {'-'*8}  {'-'*8}  MOV\tx0, #0\t\t\t;x0始终为0，XZR寄存器?")
        print(f"{'-'*8}  {'-'*8}  {'-'*8}  {'-'*8}  MOV\tx4, pArgs\t\t;参数列表指针")
        print(f"{'-'*8}  {'-'*8}  {'-'*8}  {'-'*8}  MOV\tx5, #0")
        print(f"{'-'*8}  {'-'*8}  {'-'*8}  {'-'*8}  MOV\tx6, pfn_external_func_list\t\t;外部函数列表指针")
        print(f"{'-'*8}  {'-'*8}  {'-'*8}  {'-'*8}  MOV\tx7, pCallRegisterTrampolineFunction\t;保存跳转函数地址")
        print(f"{'-'*8}  {'-'*8}  {'-'*8}  {'-'*8}  MOV\tx29, pVirualStackBottom;\t\t;虚拟机堆栈栈底")
        print(f"{'-'*8}  {'-'*8}  {'-'*8}  {'-'*8}  MOV\tlr, #0\t\t\t;x31=0")
 
        # 已解析的指令
        known_insns_op1 = {}        # dcode_insns_status=1
        known_insns_op2 = {}        # dcode_insns_status=2
        # 未解析的指令
        unknown_insts_op1 = set()   # dcode_insns_status=3
        unknown_insts_op2 = set()   # dcode_insns_status=4
        bytes_code = content[pcode_start : pcode_start + pcode_size]
 
        for pc in range(0, pcode_size, 4):
                word = struct.unpack(" 0:
                        opcode = word[0]           
                        op1 = get_op1(opcode)
                        Xt = get_openand1(opcode)   # x8/w8
                        Xn = get_operand2(opcode)   # x9/w9   
                        Xm = get_operand3(opcode)   # x10/w10
                        X4 = get_operand4(opcode)   # x11/w11
                        match op1:
                                case 11:
                                        Xt = get_operand3(opcode)   # x10/w10
                                        Xn = get_openand1(opcode)   # x8/w8
                                        Xm = get_operand2(opcode)   # x9/w9                
                                        op2 = get_op2(opcode)                   
                                        match op2:
                                                case 7:
                                                        dcode_insns_status=2
                                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tORR\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, {get_regsiter_name(Xm)}")
                                                case 12:
                                                        dcode_insns_status=2
                                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tADD\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, {get_regsiter_name(Xm)}")
                                                case 25:
                                                        dcode_insns_status=2
                                                        if X4 == 0:
                                                                print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tNOP\t\t\t\t;LSL\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, #{X4}")
                                                        else:
                                                                print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tLSL)\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, #{X4}")
                                                case 39:
                                                        dcode_insns_status=2
                                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tCMP\t{get_regsiter_name(Xm)}, {get_regsiter_name(Xn)}")
                                                        print(f"{' '*8}  {' '*8}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tCSET\t{get_regsiter_name(Xt)}, CC")
                                                case 43:
                                                        dcode_insns_status=2
                                                        if liner_disasm:
                                                                print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tBR\t{get_regsiter_name(Xm)}\t\t\t;LR={get_regsiter_name(Xt)}")
                                                        else:
                                                                asm = f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tBR\t{get_regsiter_name(Xm)}\t\t\t;LR={get_regsiter_name(Xt)}"
                                                                set_branch_control(asm)
                                                case 62:
                                                        dcode_insns_status=2
                                                        if liner_disasm:
                                                                print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tExitVm\t0\t\t\t;{get_regsiter_name(Xm)}")
                                                        else:
                                                                asm = f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\tExitVm\t0\t\t\t;{get_regsiter_name(Xm)}"
                                                                set_branch_control(asm)
                                                case _:
                                                        dcode_insns_status=4
                                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {op2:02d}(0x{op2:02X})\t>> op2_xxx\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, {get_regsiter_name(Xm)}")
                                case 7:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode)
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tORR\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, #{hex(imm16)}")
                                case 12:
                                        dcode_insns_status = 1
                                        imm26 = get_imm26(opcode)
                                        offset = imm26 * 4
                                        if liner_disasm:
                                                print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tB\t{hex(offset)}")
                                        else:
                                                asm = f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tB\t{hex(offset)}"
                                                set_branch_control(asm)
                                case 17:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode)
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tADD\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, #{hex(imm16)}")
                                case 21:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode)
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tADD\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, #{hex(imm16)}")
                                case 23:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode)
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tSTR\t{get_regsiter_name(Xt)}, [{get_regsiter_name(Xn)}, #{hex(imm16)}]")
                                case 24:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode)
                                        offset = pc + imm16 * 4 + 4
                                        if liner_disasm:
                                                print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tB.HS\t#{hex(offset)}\t\t\t;{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, ${hex(imm16 * 4)}")
                                        else:
                                                asm = f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tB.HS\t#{hex(offset)}\t\t\t;{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, ${hex(imm16 * 4)}"
                                                set_branch_control(asm)
                                case 40:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode)
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tLDR\t{get_regsiter_name(Xt)}, [{get_regsiter_name(Xn)}, #{hex(imm16)}]")
                                case 48:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode)
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tSTR\t{get_regsiter_name(Xt, 32)}, [{get_regsiter_name(Xn)}, #{hex(imm16)}]")
                                case 52:
                                        dcode_insns_status = 1
                                        imm16 = get_imm16(opcode, False)
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tMOVZ\t{get_regsiter_name(Xt, 32)}, #{hex(imm16)}, LSL#16")
                                        print(f"{' '*8}  {' '*8}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tSXTW\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xt, 32)}")
                                case _:
                                        dcode_insns_status=3                  
                                        print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\t>> op1_xxx\t{get_regsiter_name(Xt)}, {get_regsiter_name(Xn)}, {get_regsiter_name(Xm)}")
                        if not liner_disasm:
                                branch_pipeline_process()
                        record_insns(dcode_insns_status, known_insns_op1, known_insns_op2, unknown_insts_op1, unknown_insts_op2, opcode)
                else:
                        print("error")   
        decode_statistics(known_insns_op1, known_insns_op2, unknown_insts_op1, unknown_insts_op2)
 
def main():
        # 解码vm2
        pcode_start = 0xB090
        pcode_size = 0x2D8
        decode(pcode_start, pcode_size)
 
        # # 解码vm3,获取aes的key和iv
        pcode_start = 0xBDE0
        pcode_size = 0x1C8
        decode(pcode_start, pcode_size)
 
        # 解码vm1,JNI_OnLoad
        pcode_start = 0x85C0
        pcode_size = 0xCC
        decode(pcode_start, pcode_size)
 
if __name__ == "__main__":
        main() 
```

vm2 伪汇编分析
---------

vm2 还原的伪汇编代码，经过分析主要做了这些事件:

1.  保存寄存器环境
2.  解密外部函数地址
3.  生成随机数
4.  以随机数为入参生成 aes 加密的 key 和 iv
5.  生成 pSrcBuffer 的 hash
6.  拼接数据 hash+scrBuffer
7.  aes 加密拼接的数据
8.  拼接加密结果: magic(0x6 字节) + randNumber(0x20 字节) + aesOut
9.  恢复寄存器环境  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_ES6PT7GTPBUT344.webp)  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_8W29SJP6W5NA5XN.webp)  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_63VBQ4J4EJQC9WQ.webp)  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_DRGKJYJ5BFYRBGR.webp)

vm3 伪汇编分析
---------

由于 vm2 会调用 vm3 生成 aes 的 key 和 iv，因此 vm3 的代码也需要解析还原 **generate_aes_key_iv.asm**：

1.  保存寄存器环境
2.  解密外部函数地址
3.  对随机数生成 hash
4.  解密种子数据
5.  对随机数生成 hash 和种子数据再次生成 hash
6.  再次生成的 hash 数据中截取 key 和 iv
7.  恢复寄存器环境  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_HA3T79DE3XM4NWW.webp)  
    ![](https://bbs.kanxue.com/upload/attach/202406/271698_SD42R4BWZDBKGXU.webp)

最终算法
----

这就是 libEncryptor.so 中 ttEncrypt 函数的加密算法了。

```
import secrets
import hashlib
from Crypto.Cipher import AES   # pip install pycryptodome
from Crypto.Util.Padding import pad, unpad
 
def generate_rand_number():
        return secrets.token_bytes(32)
 
def decrypt_seeds():
        key1 =  [0x52, 0x09, 0x6A, 0xD5, 0x30, 0x36, 0xA5, 0x38, 0xBF, 0x40, 0xA3, 0x9E, 0x81, 0xF3, 0xD7, 0xFB]
        key1 += [0x7C, 0xE3, 0x39, 0x82, 0x9B, 0x2F, 0xFF, 0x87, 0x34, 0x8E, 0x43, 0x44, 0xC4, 0xDE, 0xE9, 0xCB]
        key1 += [0x54, 0x7B, 0x94, 0x32, 0xA6, 0xC2, 0x23, 0x3D, 0xEE, 0x4C, 0x95, 0x0B, 0x42, 0xFA, 0xC3, 0x4E]
        key1 += [0x08, 0x2E, 0xA1, 0x66, 0x28, 0xD9, 0x24, 0xB2, 0x76, 0x5B, 0xA2, 0x49, 0x6D, 0x8B, 0xD1, 0x25]
 
        key2 =  [0x1F, 0xDD, 0xA8, 0x33, 0x88, 0x07, 0xC7, 0x31, 0xB1, 0x12, 0x10, 0x59, 0x27, 0x80, 0xEC, 0x5F]
        key2 += [0x60, 0x51, 0x7F, 0xA9, 0x19, 0xB5, 0x4A, 0x0D, 0x2D, 0xE5, 0x7A, 0x9F, 0x93, 0xC9, 0x9C, 0xEF]
        key2 += [0xA0, 0xE0, 0x3B, 0x4D, 0xAE, 0x2A, 0xF5, 0xB0, 0xC8, 0xEB, 0xBB, 0x3C, 0x83, 0x53, 0x99, 0x61]
        key2 += [0x17, 0x2B, 0x04, 0x7E, 0xBA, 0x77, 0xD6, 0x26, 0xE1, 0x69, 0x14, 0x63, 0x55, 0x21, 0x0C, 0x7D]
 
        results = bytearray()
        for i in range(len(key1)):
                results.append(key1[i] ^ key2[i])
        return results
 
def sha512(buff):
        sha512_hash = hashlib.sha512()
        sha512_hash.update(buff)
        return sha512_hash.digest()
 
def get_aes_key_iv(rand_num):   
        rand_num_hash = sha512(rand_num)   
        seeds = decrypt_seeds()
        data = rand_num_hash + seeds   
        key_iv_hash = sha512(data)
 
        print(f"rand_num:              {rand_num.hex()}")
        print(f"rand_num_hash:         {rand_num_hash.hex()}")
        print(f"seeds:                 {seeds.hex()}")   
        print(f"rand_num_hash + seeds: {data.hex()}")
        print(f"key_iv_hash:           {key_iv_hash.hex()}")
        print(f"aes key:               {key_iv_hash[0:0x10].hex()}")
        print(f"aes iv:                {key_iv_hash[0x10:0x20].hex()}")
        return key_iv_hash[0:0x10], key_iv_hash[0x10:0x20]
 
def aes_128_cbc_encrypt(plaintext, key, iv):
        # 创建一个 AES 加密器对象
        cipher = AES.new(key, AES.MODE_CBC, iv)
 
        # 添加填充并加密数据
        padded_data = pad(plaintext, AES.block_size)
        ciphertext = cipher.encrypt(padded_data)
        return ciphertext
 
def get_magic():
        return b"\x74\x63\x05\x10\x00\x00"
 
def ttEncrypt(buff):
        magic = get_magic()
        rand_number = generate_rand_number()
        aes_key, aes_iv = get_aes_key_iv(rand_number)
        buff_hash = sha512(buff)
        aes_plaintext = buff_hash + buff
        aes_ciphertext = aes_128_cbc_encrypt(aes_plaintext, aes_key, aes_iv)
 
        print(f"plaintext:     {buff.hex()}")
        print(f"rand_number:   {rand_number.hex()}")  
        print(f"buff hash:     {buff_hash.hex()}")
        print(f"aes_plaintext: {aes_plaintext.hex()}")
        print(f"ciphertext:    {aes_ciphertext.hex()}")
 
        return magic + rand_number + aes_ciphertext
 
def main():
        buff = b'aabbccddeeffgg'   
        result = ttEncrypt(buff)
        print(f"ttEncrypt: {result.hex()}")
 
def test_aes():
        buff =  b"\x61\x02\xbe\x54\xa6\x2a\x73\xe7\x65\xba\x38\xc9\x87\x34\x09\xbd" +\
                        b"\xeb\xb6\xb0\xd3\x7e\xa0\x60\x40\x3d\x0c\x26\xfe\xa5\xeb\xb6\xba" +\
                        b"\x5a\x0c\x7f\x36\xec\xb7\x58\xc7\x7e\x19\x37\x50\x5f\xa8\x5b\x4e" +\
                        b"\x77\xce\x82\x7a\x70\x09\xd2\x2b\x2f\xaf\xc4\x68\x00\xd7\xa9\xff" +\
                        b"\x62\x69\x61\x6e\x66\x65\x6e\x67"
        aes_key = b"\xe8\xaf\x6e\x91\xde\x99\x7e\xf0\xfa\xfb\xcd\xbe\x97\x73\xb2\xc5"
        aes_iv = b"\x03\x7e\xed\x97\x4e\x1e\xc5\x19\xdc\xc2\xb4\x35\x5b\x26\xf0\x1b"  
        ciphertext = aes_128_cbc_encrypt(buff, aes_key, aes_iv)
        print(f"ciphertext: {ciphertext.hex()}")
 
if __name__ == "__main__":
        main()
        # test_aes()

```

验证
==

为了验证算法是否有效，这里使用了 dy 模块中的 ttEncrypt 算和 tt 的设备注册代码：

```
import secrets
import uuid
import time
import json
import hashlib
import gzip
import requests # pip install requests
import ttEncryptorUtil
 
 
# 注: 协议来自Tiktok
 
 
def UUID():
    return str(uuid.uuid4())
 
 
def md5(message):
    md5_hash = hashlib.md5()
    md5_hash.update(message)
    return md5_hash.hexdigest()
 
 
def get_timestamp_in_millisecond():
    return int(time.time() * 1000)
 
 
def get_timestamp_in_second():
    return int(time.time())
 
 
def generate_android_id():
    return secrets.token_bytes(8).hex()
 
 
def http_post(url, headers, payload):   
    response = requests.request("POST", url, headers=headers, data=payload)   
    return response.text
 
 
def gzip_compress(buff):
    return gzip.compress(buff)
 
 
def get_post_data(android_id, cdid, google_aid, clientudid):   
    openudid = android_id
    postDataObj = {
        "magic_tag": "ss_app_log",
        "header": {
            "display_name": "TikTok",
            "update_version_code": 2023205030,
            "manifest_version_code": 2023205030,
            "app_version_minor": "",
            "aid": 1233,
            "channel": "googleplay",
            "package": "com.zhiliaoapp.musically",
            "app_version": "32.5.3",
            "version_code": 320503,
            "sdk_version": "3.9.17-bugfix.9",
            "sdk_target_version": 29,
            "git_hash": "3e93151",
            "os": "Android",
            "os_version": "11",
            "os_api": 30,
            "device_model": "Pixel 2",
            "device_brand": "google",
            "device_manufacturer": "Google",
            "cpu_abi": "arm64-v8a",
            "release_build": "e7cd5de_20231207",
            "density_dpi": 420,
            "display_density": "mdpi",
            "resolution": "1794x1080",
            "language": "en",
            "timezone": -5,
            "access": "wifi",
            "not_request_sender": 1,
            "rom": "6934943",
            "rom_version": "RP1A.201005.004.A1",
            "cdid": cdid,
            "sig_hash": "194326e82c84a639a52e5c023116f12a", # md5(packageInfo.signatures[0])
            "gaid_limited": 0,
            "google_aid": google_aid,
            "openudid": openudid,
            "clientudid": clientudid,
            "tz_name": "America\\/New_York",
            "tz_offset": -18000,
            "req_id": UUID(),
            "device_platform": "android",
            "custom": {
                "is_kids_mode": 0,
                "filter_warn": 0,
                "web_ua": "Mozilla\\/5.0 (Linux; Android 11; Pixel 2 Build\\/RP1A.201005.004.A1; wv) AppleWebKit\\/537.36 (KHTML, like Gecko) Version\\/4.0 Chrome\\/116.0.0.0 Mobile Safari\\/537.36",
                "user_period": 0,
                "screen_height_dp": 683,
                "user_mode": -1,
                "apk_last_update_time": 1702363135217,
                "screen_width_dp": 411
            },
            "apk_first_install_time": 1697783355395,
            "is_system_app": 0,
            "sdk_flavor": "global",
            "guest_mode": 0
        },
        "_gen_time": get_timestamp_in_millisecond()
    }
     
    return gzip_compress(json.dumps(postDataObj).encode(encoding='utf-8'))
 
 
def get_headers(md5Hash):
    headers = {
            'log-encode-type': 'gzip',
            'x-tt-request-tag': 't=0;n=1',
            'sdk-version': '2',
            'X-SS-REQ-TICKET': f'{get_timestamp_in_millisecond()}',
            'passport-sdk-version': '19',
            'x-tt-dm-status': 'login=0;ct=1;rt=4',
            'x-vc-bdturing-sdk-version': '2.3.4.i18n',
            'Content-Type': 'application/octet-stream;tt-data=a',
            'X-SS-STUB': md5Hash,
            'Host': 'log-va.tiktokv.com'
        }
    return headers
 
 
def get_device_register_url(openudid, cdid):
    url = 'https://log-va.tiktokv.com/service/2/device_register/?' + \
            "tt_data=a" + \
            "ac=wifi" + \
            "channel=googleplay" + \
            "aid=1233" + \
            "app_ + \
            "version_code=320503" + \
            "version_ + \
            "device_platform=android" + \
            "os=android" + \
            "ab_version=32.5.3" + \
            "ssmix=a" + \
            "device_type=Pixel+2" + \
            "device_brand=google" + \
            "language=en" + \
            "os_api=30" + \
            "os_version=11" + \
            f"openudid={openudid}" + \
            "manifest_version_code=2023205030" + \
            "resolution=1080*1794" + \
            "dpi=420" + \
            "update_version_code=2023205030" + \
            f"_rticket={get_timestamp_in_millisecond()}" + \
            "is_pad=0" + \
            "current_region=TW" + \
            "app_type=normal" + \
            "timezone_ + \
            "residence=TW" + \
            "app_language=en" + \
            "ac2=wifi5g" + \
            "uoo=0" + \
            "op_region=TW" + \
            "timezone_offset=-18000" + \
            "build_number=32.5.3" + \
            "host_abi=arm64-v8a" + \
            "locale=en" + \
            f"ts={get_timestamp_in_second()}" + \
            f"cdid={cdid}"
    return url
 
 
def device_register():   
    android_id = generate_android_id()
    cdid = UUID()
    google_aid = UUID()
    clientudid = UUID()
    gzip_post_data = get_post_data(android_id, cdid, google_aid, clientudid)
    ttencrypt_post_data = ttEncryptorUtil.ttEncrypt(gzip_post_data)
    headers = get_headers(md5(ttencrypt_post_data))
    url = get_device_register_url(android_id, cdid)
    response = http_post(url, headers, ttencrypt_post_data)
    print(f"设备注册结果:\n{response}")
 
if __name__ == "__main__":
    device_register()

```

结果：

```
设备注册结果:
{"server_time":1719390270,"device_id":7384724147647088134,"install_id":7384724761467832069,"new_user":1,"device_id_str":"7384724147647088134","install_id_str":"7384724761467832069"}

```

总结
==

在分析完之后虚拟机保护的强度没有想像中的那么好，在分析过的虚拟化加密强度非要分 10 个等级的话，VMProtect 为 10 级、Safengine Shielden 强度为 9 级、Themida 强度 9 级、VProtect 强度 7 级、Enigma Protector 强度 3 级，而它的强度等级仅为 1 级。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 2 分钟前 被金罡编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)

上传的附件：

*   [ttEncrypt.zip](javascript:void(0)) （37.79kb，36 次下载）