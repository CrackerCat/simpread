> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285567.htm)

> [原创][原创]ARM64 目前主流的反混淆技术的初窥

声明
==

本文仅限于技术讨论，不得用于非法途径，后果自负。

前言
==

随着混淆技术和虚拟化的发展，我们不可能很方便的去得到我们想要的东西，既然如此，那只能比谁头更铁了，本文列举一下在逆向某一个签名字段中开发者所布下的铜墙铁壁，以及我的头铁方案，本文更多的是分析思路，而不是解决方案，所以样本自己找

BR reg
======

在 ARM64 中 寄存器跳转只剩下 BR 指令了，由于 ida 为了 br 的准确性 (cs:ip 跳转)，ida 会识别成函数跳转，但是这却给开发者带来了天大的便利，于是乎，一个贼好用的 anti 反编译器函数分析方案横空出世  
![](https://bbs.kanxue.com/upload/attach/202502/856431_8SZXEBC9D4Y778F.png)  
让我们回到汇编

```
.text:000000000008D1E8                 BL              sub_8D1F8
.text:000000000008D1EC                 MOV             X1, X0
.text:000000000008D1F0                 ADD             X1, X1, #0x38 ; '8'
.text:000000000008D1F4                 BR              X1
```

可以看到 X1 的值是由 sub_8D1F8 返回值加上 0x38 ，而 sub_8D1F8 一看地址 不对啊，为什么这么近。我们再看看 sub_8D1F8 的汇编  
![](https://bbs.kanxue.com/upload/attach/202502/856431_6D3F9JC7T5DKHWT.png)  
如果经常看汇编的小伙伴已经懂了

```
.text:000000000008D1FC                 STP             X29, X30, [SP]
.text:000000000008D200                 LDR             X0, [SP,#8]
```

sub_8D1F8 的返回值 被 x30 也就是 lr 寄存器赋值 所以 X1 = sub_8D1F8 返回地址 +0x38 = 0x8D1EC + 0x38 = 0x8d224  
![](https://bbs.kanxue.com/upload/attach/202502/856431_2ZZDK45EW8KAJSH.png)  
既然如此 直接改跳转吧，写个脚本模式匹配下

```
def antiBR1(start,end):
    addr = start
    while addr < end :
        insn = idc.print_insn_mnem(addr)
        op0 = idc.print_operand(addr,0)
        op1 = idc.print_operand(addr,1)
        # .text:000000000008B03C                 BL              loc_8B04C
        # .text:000000000008B040                 MOV             X1, X0
        # .text:000000000008B044                 ADD             X1, X1, #0x38 ; '8'
        # .text:000000000008B048                 BR              X1
        # match 4 instructions
        if insn == "BL" and (idc.print_insn_mnem(addr + 0xc).find("BR") != -1 or idc.print_insn_mnem(addr + 0x8).find("BR") != -1):
            # find add
            addr1 =addr
            while 1:
                str = get_instruction_at_address(addr1)
                if str != None:
                    if str.find("ADD") != -1:
                        break
                addr1 = addr1 + 4
            opeValue = idc.get_operand_value(addr1,2)
            print("find add: %x" % opeValue)
            # patch addr B (addr +4 + opeValue)
            code,count =ks.asm("B "+ hex(addr + 4 + opeValue),addr)
            print(hex(addr))
            ida_bytes.patch_bytes(addr, bytes(code))
        addr = addr + 4
```

修复后  
![](https://bbs.kanxue.com/upload/attach/202502/856431_PQ7PJRYVQWYJB7P.png)

花指令
===

可以看到一排奇怪的东西  
![](https://bbs.kanxue.com/upload/attach/202502/856431_MM7XWEH7H59MTT4.png)  
正常程序怎么会有呢，看看汇编

```
.text:000000000008D53C loc_8D53C                               ; CODE XREF: sub_8D170:loc_8D530↑j
.text:000000000008D53C                 MRS             X1, NZCV
.text:000000000008D540                 MOV             X0, XZR
.text:000000000008D544                 CMP             X0, XZR
.text:000000000008D548                 MSR             NZCV, X0
.text:000000000008D54C                 B.NE            loc_8D558
.text:000000000008D550                 CLREX
.text:000000000008D554                 BRK             #3
.text:000000000008D558
.text:000000000008D558 loc_8D558                               ; CODE XREF: sub_8D170+3DC↑j
.text:000000000008D558                 MSR             NZCV, X1
```

MSR NZCV, X0 x0 = 0 所以 zf 位为 0 所以 B.NE 恒成立 所以啥事没干 所以可以直接 nop  
简单的看下逻辑  
![](https://bbs.kanxue.com/upload/attach/202502/856431_8WD6YTMMREF9P44.png)  
0x8d240 在这个函数内 所以不是代码自解码 就是校验  
![](https://bbs.kanxue.com/upload/attach/202502/856431_KZ7BFHZANG6ZF4Y.png)  
查看 sub_13C704  
![](https://bbs.kanxue.com/upload/attach/202502/856431_TASBMDHPGMFJ2BM.png)  
那就是检验咯 不管 下一个函数 sub_13DC3C  
修复后发现 f5 没东西  
![](https://bbs.kanxue.com/upload/attach/202502/856431_TXM6A4NMUEHQK86.png)  
这怎么可能 切换到汇编，研究后

```
.text:000000000013DCE4 000              MOV             X30, X17
.text:000000000013DCE8 000               SUB             SP, SP, #0x50 ; 'P'
.text:000000000013DCEC 050              STP             X29, X17, [SP,#0x50+var_10]
.text:000000000013DCF0 050              ADD             X29, SP, #0x50+var_10
.text:000000000013DCF4 050              STUR            X0, [X29,#-0x10]
.text:000000000013DCF8 050              STUR            X1, [X29,#-0x18]
.text:000000000013DCFC 050            STR             X0, [SP,#0x50+var_30]
```

```
.text:000000000013DD94 050              LDR             X9, [SP,#0x50+var_30]
.text:000000000013DD98 050              LDR             X10, [SP,#0x50+var_38]
.text:000000000013DD9C 050              STR             X9, [X10,#8]
```

var_38 = x29 x29 指向存放 x29 x30 的栈地址 所以 STR X9, [X10,#8] 等价于 x30 = x9 x0 是参数 所以回上一个函数  
![](https://bbs.kanxue.com/upload/attach/202502/856431_G9GSNPR8B7Y6B5D.png)

去看看吧 sub_8d5c8

ollvm
=====

![](https://bbs.kanxue.com/upload/attach/202502/856431_XW4XFDDEWJ5H4YR.png)

#### 标准控制流平坦化

这里简单介绍什么是控制流平坦化  
源代码

```
if(temp){
    print("ok");
    return 1
}
else{
    print("no");
     return 2
}
```

经过平坦化后

```
var flowState = 0;
while(1)
{
    switch(flowState):
        case 0:
            if(temp){
                flowState = 1
            }
            else{
                flowState = 2
            }
            break
        case 1:
            print("ok");
            flowState = 3
        case 2:
            print("no");
            flowState = 4
        case 3:
            return 1
        case 4:
            return 0
}
```

可以看到一个 while switch 的结构 其中 flowState 负责控制整个流程 这个就是平坦化的基本原理 可以看到 8 行普通的代码被膨胀到了 23 行 假如我再平坦化 一次呢 我们会发现随着平坦化的越来越多，肉眼阅读的能力也越来越困难  
下面介绍一个我从国外看到的方案 https://hex-rays.com/blog/hex-rays-microcode-api-vs-obfuscating-compiler

###### 1. 计算支配节点

下面画一个简单的 cfg 图

```
       0
       |
       1
     /   \
   2       3
 /  \     /  \
4      5      6
```

如果走到某一个节点 2 必须经过另外一个节点 1 那么 2 被 1 支配 1 是 2 的支配节点  
通过 bitset 可以快速计算出来

0: 0  
1: 1  
2:0,1  
3: 0,1  
4:0,1,2  
5:0,1,2,3  
6:0,1,3  
将其反转  
可得 0 是所有节点的支配节点  
有什么用？  
仔细看平坦化的代码 可以发现 while 会被所有节点支配 所以 控制流分发快 必定被所有节点支配 由此定位到了分发块

###### 2. 路径计算

我们来看 如果从 0 要走到 5 有几种 路径  
0125  
0135  
将 cfg 填充内容

```
            0
        flowState = 0
            |
            1
          /   \
        2       3
flowState = 1  flowState=2
      /  \     /  \
     4      5      6
     |      |       |
     1      1       1
```

可以得到 0125 flowState = 1， 0124 flowState = 2 那么如果计算支配节点的所有路径 是不是可以得到所有 flowState 的状态

###### 状态指向计算

通过 switch 可以轻松得到 状态指向的地址

###### 修改控制流

将每一个状态所对应的 地址 连接

##### 死代码消除

检查每一个 没有前继节点的节点 删除 反编译器自动优化

###### 活跃变量分析

假设 0 节点 flowState = 0 1 节点 flowState 参与 2 节点中 flowState = 1 那么 flowState 1 节点 将被删除 反编译器自动优化

#### 反混淆

接下来回到这个 demo  
将他转换成 cfg  
![](https://bbs.kanxue.com/upload/attach/202502/856431_JJGB3K94ZBRBE56.png)

###### 计算支配节点

计算支配节点 为 0

###### 路径状态计算

0：0 flowState =0

1：01 flowState=0

2：02 flowState =3

3：04 flowState =4

4：04 flowState =0

5：05 flowState =0

6: 016 flowState =1

7: 017 flowState =2

###### 状态指向计算

0 => 1  
1 => 2  
2 => 3  
3 => 4  
4 -> 5

###### 修改控制流

![](https://bbs.kanxue.com/upload/attach/202502/856431_WMMHB6JQYEDAPXC.png)

###### 死代码消除

###### 活跃变量分析

![](https://bbs.kanxue.com/upload/attach/202502/856431_3XBPHQ48E3UAMNF.png)

这就是简单的 ollvm 还原方案  
下面进入正题  
![](https://bbs.kanxue.com/upload/attach/202502/856431_XG5VBTAT4SH4HF3.png)

###### 1. 计算支配节点

计算支配节点 为 0x8C1EC

###### 2. 路径计算

dfs 算法

###### 3. 路径状态计算

符号执行 angr

###### 状态指向计算

！！！ 熟悉的 switch 呢 这么变成 bge 了  
![](https://bbs.kanxue.com/upload/attach/202502/856431_GFJ8RRXFW4QU96A.png)  
这就是非标准的 ollvm 了 简单概括就是控制节点下发  
![](https://bbs.kanxue.com/upload/attach/202502/856431_P55V3S3772F9N6Z.png)  
通过 if else if else 取代了 switch 结构 干扰了状态指向计算 并且带来了更多的复杂性，使其无法定位真实代码块  
通过研究 发现 开发者定位真实代码块的方案为

![](https://bbs.kanxue.com/upload/attach/202502/856431_ZGDREUXVJTGH4U5.png)

bne 跳转 等于值跳转  
于是乎 模式匹配 找出所有 cmp w8,value ? reg bne loc_??? 的代码 拿到真实块

###### 修改控制流

这里又又又有幺蛾子

![](https://bbs.kanxue.com/upload/attach/202502/856431_M8KH5K9F4YZ7DUE.png)  
![](https://bbs.kanxue.com/upload/attach/202502/856431_8K9ARC3HBVQPMFM.png)  
发现编译器优化 flowState 更新状态的代码 被优化成 1 份，类似于  
有点类似于

```
flowState2 =876832131267
flowState1 =312321312
if(..)
    flowState = flowState1
else
    flowState = flowState2
...
flowState2 =434876867
flowState1 =366512321312
if(..)
    flowState = flowState1
else
    flowState = flowState2
```

优化后

```
    flowState2 =876832131267
    flowState1 =312321312
    goto lable_1
 
    ...
    flowState2 =434876867
    flowState1 =366512321312
lable_1:
    if(..)
        flowState = flowState1
    else
        flowState = flowState2
```

这种只能想办法给他还原回去  
![](https://bbs.kanxue.com/upload/attach/202502/856431_BANJ43FNQ2MYKF8.png)

###### 死代码消除

###### 活跃变量分析

ida 自动优化

结果
--

![](https://bbs.kanxue.com/upload/attach/202502/856431_EVSTSKYXNWT5X45.png)  
![](https://bbs.kanxue.com/upload/attach/202502/856431_Y5VMFG344SRAHCA.png)

###### 碎碎念

由于编译器优化 你永远想不到会有多少幺蛾子 ，所以混淆还原应该还有 2 个步骤

1. 对抗编译器优化 上述

2. 魔改 ollvm 还原成标准 ollvm if elseif else 还原为 switch  
对这 2 个感兴趣的可以看看这个 http://profs.sci.univr.it/~giaco/download/Watermarking-Obfuscation/unflatten.pdf

完结撒花表情❀❀❀ ヾ (≧▽≦*)o

[[招生] 科锐逆向工程师培训 (2025 年 3 月 11 日实地，远程教学同时开班, 第 52 期)！](https://bbs.kanxue.com/thread-51839.htm)

[#逆向分析](forum-161-1-118.htm) [#协议分析](forum-161-1-120.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)