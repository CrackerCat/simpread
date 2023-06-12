> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277569.htm)

> 看开水团 与 ARM 花指令构造与静态 Patch

看开水团 与 ARM 花指令构造与静态 Patch
=========================

在工作中, 多多少少会遇到花指令的情况，本系列文章我会慢慢更新, 把我学习过程分享给大家, 从简单的构造到如何结合手头的工具 patch

什么是花指令
------

洋文叫: Junk Code 脏指令 主要作用就是用来干扰, 反汇编引擎 (线形扫描 & 递归下降) 和 F5 增加逆向(PJ) 难度的一种技术, 能屏蔽大部分`脚本小子`, 这些指令 CPU 也不认识, 执行到这种指令 CPU 直接会报错, 如何不影响 CPU 正常执行又能干扰反汇编引擎的花指令呢

花指令种类与如何简单构造
------------

花指令的种类有很多, 这里只讨论移动端或者说 ARM 平台, 哪些团队在用花指令呢我的知识范围内发现的的商业级样本中,`魔兽世界怀旧服macOS版` `某泔水团` 等等等等........ 大家可以后续补充。

 

这里为什么没有研究 x86 是第一因为我不会第二个原因是 x86 是非定长指令集, 这会导致 x86 有那种双字节的花指令不适合线形扫描, 要校验的太多了无法通过定长 4 字节的方式进行线性扫描。

 

ARM64 指令集天生就是定长 4 字节, 这里可能有人会杠`ADRL x0, unk_1023d5501`8 字节你怎么说，其实这是两条指令`adrp x0, #0xc` `add x0, x0, #0x501`

我是谁我在哪儿
-------

行不更名坐不改姓:`周樟寿` 首先我要给`梁总`道个歉我就是那个在群里口嗨，要举报你的沙雕虽然您删了，但是还是要道歉。我呢是一个失业在家找不到工作研究技术的老逼蹬。

第一章简单的花指令构造与修正
--------------

*   example1
    
    ```
                                        _main:
    0x0000000100003c84 FF8300D1               sub        sp, sp, #0x20
    0x0000000100003c88 FD7B01A9               stp        fp, lr, [sp, #0x10]
    0x0000000100003c8c FD430091               add        fp, sp, #0x10
    0x0000000100003c90 08008052               mov        w8, #0x0
    0x0000000100003c94 FF0B00B9               str        wzr, [sp, #0x8]
    0x0000000100003c98                        db  0xb1 ; '.'
    0x0000000100003c99                        db  0x7f ; '.'
    0x0000000100003c9a                        db  0x39 ; '9'
    0x0000000100003c9b                        db  0x05 ; '.'
    0x0000000100003c9c                        db  0x4e ; 'N'
    0x0000000100003c9d                        db  0x61 ; 'a'
    0x0000000100003c9e                        db  0xbc ; '.'
    0x0000000100003c9f                        db  0x00 ; '.'
    0x0000000100003ca0 29048052               mov        w9, #0x21
    
    ```
    
    上边的样本是我自己构造的简单花指令样本, 可以看到 0x100003c98-0x100003c9f 无法被反汇编引擎正常解析并标记为数据 db, 这个样本是在 Hopper 中解析的, 在`ida pro`呈现的是: DCB/DCW/DWD/DCQ 这里简单理解他们都是数据只是大小不同 B = byte,W= word (2bytes),D = dword(4bytes),Q = qword(8bytes) [ARM 文档](https://developer.arm.com/documentation/dui0473/m/directives-reference/dcb?lang=en)
    

QA：那问题来了数据不是有数据段么, 怎么出现在代码段了？

 

DCB/DCW/DWD/DCQ 在 ARM 中表示`定义数据`的指令，正常的代码经过编译器怎么能犯这种错误呢, 问题就出在二进制工程师通过某种手段构造出来故意干扰`反汇编引擎`和`脚本小子`下边公布代码：

*   main.cpp
    
    ```
    /****
     *@author 周樟寿
     */
      #include /****
       * __attribute__((always_inline))  内联函数
       * @return
       */
      static __attribute__((always_inline)) int case1() {
     
      #if __arm64__
           __asm__(
              "udf #0x5397FB1\n"
              ".long 87654321\n"
              ".long 12345678\n"
          );
      #endif
          int b = 1 * 3 + (5 * 6);
     
          return b;
      }
      int main() {
      int c = case1();
      std::cout << (c) << std::endl;
      return 0;
    } 
    ```
    

这个例子是无法正常运行执行的, 因为 cpu 执行到了我们通过内联汇编构造的非法指令, 这里使用了两种方式来构造非法指令

*   .long 表示声明是数据即我们看到 DCQ
*   udf 字面意思就是未定义 具体请看文档 [官方文档 - udf](https://developer.arm.com/documentation/dui0801/h/A32-and-T32-Instructions/UDF)

有了基础知识, 我们先想办法修正这类这里使用`ida python`进行修正

*   fix_example_junk_code.py
    
    ```
    ""
    @author 周樟寿
    ""
    import ida_bytes
    import idautils
    import idc
    arm64_nop = b"\x1f\x20\x30\xd5"
     
    if __name__ == '___main__':
        print("fix_example_junk_code.py begin.....")
        segments = idautils.Segments() #获取所有段
        for set in segments:
            #只处理 text 段  这个脚本支持 so 和 match-o
            if idc.get_segm_name(seg) in ['__text','.text']
                #开始
                seg_start_ea = idc.get_segm_start(seg)
                #结束
                seg_end_ea = idc.get_segm_end(seg)-4
     
                #临时变量
                current_ea = seg_start_ea
                while current_ea <= seg_end_ea:
                    #指令大小对于非指令无效
                    item_size = idc.get_item_size(current_ea)
                    #是否是asm 指令判断
                    is_asm_code = ida_bytes.is_code(ida_bytes.get_flags(current_ea))
                    if not is_asm_code:
                        # patch nop
                        ida_bytes.patch_bytes(current_ea,arm64_nop)
                        current_ea = current_ea + 4
                    else:
                        current_ea = current_ea + item_size
     
        print("fix script end success ........")
    
    ```
    
*   第一章遗留下了一个问题就是, 如何构造不影响 cpu 执行指令的花指令
    

第二章构造可用的花指令: B+junkCode
-----------------------

这个小节需要解决的是让 CPU 正常执行并且还能干扰`反汇编引擎`&`F5`, 先看例子讲解原理

*   case2
    
    ```
    /****
     *@author 周樟寿
     */
      #include /****
       * __attribute__((always_inline))  内联函数
       * @return
       */
      static __attribute__((always_inline)) int case2() {
     
      #if __arm64__
           __asm__(
              "b #0x10\n"
              "udf #0x5397FB1\n"
              ".long 87654321\n"
              ".long 12345678\n"
          );
      #endif
          int b = 1 * 3 + (5 * 6);
     
          return b;
      }
      int main() {
      int c = case2();
      std::cout << (c) << std::endl;
      return 0;
    } 
    ```
    

这个例子与第一个例子最大的不同就是, 增加了一个`B指令`来让 CPU 正常执行跳过我们精心构造的花指令, 下面聊聊为什么用 `B指令` 和 为什么后边的立即数是 `#0x10`

*   为什么用 `B` 指令 [文档](https://developer.arm.com/documentation/ddi0596/2021-12/Base-Instructions/B--Branch-?lang=en)

使用 B 指令是因为, B 指令的寻址方式是相对寻址，相对于 PC 寄存器进行的, 其次 B 是无条件跳转无需关心 LR 寄存器

*   B 指令后边的立即数

立即数为什么是 `0x10` 这里先把转成 10 进制等于 `16` 这里要回顾一下 PC 寄存器的知识，`PC`寄存器永远存储的是需要执行下一条指令的地址, 这里就会衍生出来一个公式来计算具体跳多远，公式： 立即数 =(填充指令数量 + 1) _4, 套用上边例子最终结果 16=(3+1)_4

*   结语

这种简单类型的修正思路就是，发现 B 指令, 先计算出立即数，然后判断判断一条是否为指令集，然后轮训判断下去保存总指令大小，如果发现立即数 = 我们扫描的区域大小直接全`nop`掉,  
下一章我直接根据实际样本进行学习以及修正。

*   修正脚本

fix_case2.py

```
""
@author 周樟寿
""
import ida_bytes
import idautils
import idc
 
arm64_nop = b"\x1f\x20\x30\xd5"
 
 
def get_range():
    segments = idautils.Segments()  # 获取所有段
    for seg in segments:
        # 只处理 text 段  这个脚本支持 so 和 match-o
        if idc.get_segm_name(seg) in ['__text', '.text']:
            # 开始
            seg_start_ea = idc.get_segm_start(seg)
            # 结束
            seg_end_ea = idc.get_segm_end(seg) - 4
            break
    return seg_start_ea, seg_end_ea
 
 
if __name__ == '___main__':
    print("fix_example_junk_code.py begin.....")
 
    # 开始
    seg_start_ea, seg_end_ea = get_range()
 
    # 临时变量
    current_ea = seg_start_ea
    while current_ea <= seg_end_ea:
        # 指令大小对于非指令无效
        item_size = idc.get_item_size(current_ea)
        # 是否是asm 指令判断
        is_asm_code = ida_bytes.is_code(ida_bytes.get_flags(current_ea))
        if not is_asm_code:
            current_ea = current_ea + 4
        else:
 
            # 判断是否是指令B
            if idc.print_insn_mnem(current_ea) in ['b', 'B']:
 
                # 获取B 后边的目标地址   立即数=目标地址-当前地址
                b_imm = idc.get_operand_value(current_ea, 0) - current_ea
 
                # 判断指令B下边是否是指令不是指令开始非指令区域大小
                if not ida_bytes.is_code(ida_bytes.get_flags(current_ea + 4)):
                    data_size = 0
                    begin_ea = current_ea + 4
                    while True:
                        if not ida_bytes.is_code(ida_bytes.get_flags(begin_ea)):
                            data_size = data_size + 4
                            begin_ea = begin_ea + 4
                        else:
                            break
                    if data_size == b_imm:
                        # todo not
                        print("nop fix")
 
            current_ea = current_ea + item_size
 
    print("fix script end success ........")

```

花指令的剩余套路简介与基本构造
---------------

看了这么多代码我相信老鸟都看烦了, 不拖拉直接把基础剩余的花指令套路大概过一遍直接上真实样本分享我的修正思路，大家有更好的思路也可以交流。

 

花指令的类型还有以下几种：

1.  虚假控制流 + DCQ
2.  虚假控制流 + B + 栈不平
3.  B + 栈不平
4.  利用 x30 寄存器进行 RET 跳转

还有很多精心构造的下边我给出具体构造的例子，不会再写具体修正脚本大家可以自己试着去修正一下

*   虚假控制流 + B 样本. cpp

```
/****
   *@author 周樟寿
   */
int bcf(int a) {
    return (a + a);
}
static __attribute__((always_inline)) int case3() {
    int a = 3;
    if (bcf(a) != 0) {
    #if __arm64__
        __asm__(
        "b 0xc\n"
        "udf #0x5397FB1\n"
        ".long 87654321\n"
        ".long 12345678\n"
        );
    #endif
    }
    int b = 1 * 3 + (5 * 6);
    return b;
}

```

这只是简单的例子虚假控制流按我这个写法会被编译器优化掉实际情况远比这复杂的多, 仅供参考

*   B + 栈不平

```
static __attribute__((always_inline)) int case4() {
 
    int a = 3;
    if (bcf(a) != 0) {
#if __arm64__
        __asm__(
        "b 0xc\n"
        "add sp,sp,#0x100\n"
        "add sp,sp,#0x100\n"
        );
#endif
    }
    int b = 1 * 3 + (5 * 6);
    return b;
}

```

例子和构造样本就先简单到这里我们直接结合，真实样本样本学习其中的套路

开水团花指令学习与修正 xxxx.so & xxxx
--------------------------

看开水团系列的应用我是纯抱着学习态度去的，因为平时也用但是觉得卡卡的偶尔还非常热抱着好奇心的态度去学习, 你别说有点东西下边开始分享我的学习过程，尽量分享的细致一点和我的思路，这个学习记录没有参考别人的修正方案按自己的理解进行修正，称不上完美但是可以 f5 了看起来清爽一点，如果哪里有脚本上的优化大家可以交流集思广益。

*   样本总结  
    包含的种类非常多我只修正了我关心的部分会讲解修正过程以及他们之间的关系，开水团的花指令是有关联的属于`嵌套关系`而且还夹杂了 DCQ 这种数据，其中`BR x8` 这种动态跳转我没有仔细去看不在我的修正范围
    
*   搜索方向  
    安卓从`ida pro` 的 export 窗口看 JNI_onLoad 开始看起就能发现花指令的存在了, 一直跟着向下即可, 同 Group 产品我也都看了双平台的花指令套路是一致的，大家不要去看 iOS 端修起来很费时间，如果学习尽量看安卓下边正式开始吧。
    
*   [x] 样本 1 之 利用 x30 (LR) 寄存器 + RET 强制 停止函数  
    在点进去 `JNI_onLoad` 就会看到如下
    

```
.text:14580 E0 7B 3E A9                   STP             X0, X30,[SP,#var_20]
.text:14584 01 00 00 94                   BL              sub_14588
.text:14584                               ; End of function JNI_OnLoad
                                          sub_14588
.text:14588 60 00 00 10                   ADR             X0, loc_14594
.text:1458C FE 03 00 AA                   MOV             X30, X0
.text:14590 C0 03 5F D6                   RET

```

这个样本很有意思, 第一利用了 `ADR`地址无关性把目标地址放进了 `X0`寄存器, 第二利用 X30 寄存器也就是 LR 寄存器直接跳了过去, RET 其实等同于 `mov pc,lr`那如何修正呢, 先人工修正一下看看

 

修正. asm

```
.text:14580 E0 7B 3E A9                   STP             X0, X30,[SP,#var_20]
.text:14584 01 00 00 94                   BL              sub_14588
.text:14584                               ; End of function JNI_OnLoad
                                          sub_14588
.text:14588 60 00 00 10                   nop            
.text:1458C FE 03 00 AA                   nop            
.text:14590 C0 03 5F D6                   B      loc_14594

```

这里手工修正, 修正了 3 条指令核心就在 `RET` 修正 `B 目标地址`, 这里其实有朋友会说为什么不在 `BL sub_14588`这里修正 `B loc_14594` 因为实际情况远比想象的复杂得多, 这个`ret`中间的位置可能会出现 sp 栈相关操作要不要保留？不清楚干嘛的都要保留，我直接放出修正脚本写的比较 low 大家别喷有些没找到函数，我就直接手工操作的。

*   修正. py

```
/****
   *@author 周樟寿
   */
import ida_bytes
import idautils
import idc
import re
 
from keystone import *
 
if __name__ == '__main__':
 
    """
        初始化汇编 引擎 keystone
        """
 
    ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
 
    arm64_nop = b"\x1f\x20\x03\xd5"
 
    segments = idautils.Segments()
 
    for seg in segments:
        """这里只处理 代码段"""
        x_30_ret_total = 0
 
        if idc.get_segm_name(seg) in ["__text", ".text"]:
            """获取段信息"""
            seg_name = idc.get_segm_name(seg)
            seg_start_addr = idc.get_segm_start(seg)
            seg_end_addr = idc.get_segm_end(seg) - 4
            seg_size = seg_end_addr - seg_start_addr
 
            current_ea = seg_start_addr
 
            while current_ea <= seg_end_addr:
                item_size = idc.get_item_size(current_ea)
                is_asm_code = ida_bytes.is_code(ida_bytes.get_flags(current_ea))
 
                if is_asm_code:
                    current_mnem = idc.print_insn_mnem(current_ea)
 
                    """
                    优化修复逻辑 里边有很多地方都是 通过地址-4 这种计算方式来判断的,遇到了特殊情况就无法搜索
                    需要向上逐4 字节搜索
 
                    """
                    new_fix = True
                    if new_fix:
 
                        """
                        这是原始模板,但是存在中间穿插 栈平衡相关得指令无法使用 精确 -4  判断 第一个操作数是 x30
                        和 -8 判断 mnem 是 adr
                        adr x0 ,#0xdc
                        mov x30 x0
                        ret
 
                        这里优化搜索规则  当前是RET 指令向上搜索, 如果向上搜索 4-8 字节 发现有操作 X30 (LR) 寄存器 就标
                        标记好了，x30 的 操作位置，继续向上搜索 查找ADR 指令 找到要跳转 得目标
                        """
 
                        if current_mnem == 'RET':
                            if ((idc.print_operand(current_ea - 4, 0) == 'X30') and (
                                    idc.print_insn_mnem(current_ea - 4) == 'MOV')) \
                                    or \
                                    (idc.print_operand(current_ea - 8, 0) == 'X30' and idc.print_insn_mnem(
                                        current_ea - 8) == 'MOV'
 
                                    ):
                                # (
                                #         (idc.print_operand(current_ea - 4, 0) == 'X30') and (
                                #         idc.print_insn_mnem(current_ea - 4) == 'MOV')
                                # ):
 
                                if idc.print_insn_mnem(current_ea - 4) in ['ADD', 'MOV']:
                                    print(
                                        f'current {hex(current_ea)} x30_ret  sub_4={idc.print_insn_mnem(current_ea - 4)}')
                                    """统计"""
 
                                    x_30_mov = 0
                                    x_30_adr = 0
 
                                    """
                                    nop ret
                                    """
                                    # ida_bytes.patch_bytes(current_ea, arm64_nop)
 
                                    location_addr = 0
 
                                    if idc.print_insn_mnem(current_ea - 4) == "MOV":
                                        """
                                        nop mov
                                        """
                                        # ida_bytes.patch_bytes(current_ea, arm64_nop)
                                        # ida_bytes.patch_bytes(current_ea - 4, arm64_nop)
 
                                        """
                                        用于确定具体跳转目标得指令位置
                                        """
                                        location_addr = current_ea - 8
 
                                    elif idc.print_insn_mnem(current_ea - 4) == "ADD":
 
                                        # ida_bytes.patch_bytes(current_ea - 8, arm64_nop)
 
                                        location_addr = current_ea - 12
                                    else:
                                        print(f'未知:{hex(current_ea)}')
 
                                    if location_addr != 0:
 
                                        try:
                                            b_number = int(
                                                re.search(r'[0-9A-F]+', idc.print_operand(location_addr, 1)).group(),
                                                16)
 
                                            path_asm = f'b #{hex(b_number - current_ea)}'
                                            print(
                                                f' address:{hex(location_addr)}   operand:{hex(b_number - current_ea)}  asm:{path_asm}')
 
                                            encoding, count = ks.asm(path_asm)
                                            patch_byte = bytes([int(i) for i in encoding])
                                            print(f"------------------->{path_asm} = [", end='')
                                            for i in encoding:
                                                print("%02x " % i, end='')
                                            print(']')
 
                                            """
                                            修正跳转
                                            """
                                            ida_bytes.patch_bytes(location_addr, arm64_nop)
 
                                            """
                                            修正ret
                                            """
                                            ida_bytes.patch_bytes(current_ea, patch_byte)
 
                                            if idc.print_insn_mnem(current_ea - 4) == "MOV":
                                                """
                                                nop mov
                                                """
                                                # ida_bytes.patch_bytes(current_ea, arm64_nop)
                                                ida_bytes.patch_bytes(current_ea - 4, arm64_nop)
                                            elif idc.print_insn_mnem(current_ea - 4) == "ADD":
                                                ida_bytes.patch_bytes(current_ea - 8, arm64_nop)
 
 
                                        except Exception as e:
                                            print(f'出错地址:{hex(location_addr)}')
                                        #
                                        # asm_number = b_number - location_addr
                                        #
                                        # path_asm = f'b #{b_number}'
 
                                        # encoding, count = ks.asm(path_asm)
                                        # patch_byte = bytes([int(i) for i in encoding])
                                        # print(f"------------------->{path_asm} = [", end='')
                                        # for i in encoding:
                                        #     print("%02x " % i, end='')
                                        # print(']')
                                        #
                                        # ida_bytes.patch_bytes(location_addr, patch_byte)
 
                                    x_30_ret_total = x_30_ret_total + 1
 
                    if not new_fix:
                        """如果之 这种花指令跳转规则 """
                        if (current_mnem == 'RET') and (idc.print_operand(current_ea - 4, 0) == 'X30') and (
                                idc.print_operand(current_ea - 4, 1) in ['X0', 'W0']):
                            """当前指令-8"""
                            sub_8_mnem = idc.print_insn_mnem(current_ea - 8)
 
                            sub_8_imm = 0
                            b_imm = 0
 
                            if sub_8_mnem == "ADR":
 
                                has_patch = False
 
                                sub_8_imm = idc.print_operand(current_ea - 8, 1)
                                sub_8_imm = int(re.search(r'[0-9A-F]+', sub_8_imm).group(), 16)
 
                                """
                                计算地址
                                立即数 = 目标地址-当前地址
 
                                """
 
                                b_imm = sub_8_imm - (current_ea - 8)
 
                                if has_patch:
                                    ida_bytes.patch_bytes(current_ea, arm64_nop)
                                    ida_bytes.patch_bytes(current_ea - 4, arm64_nop)
 
                                path_asm = f'bl #{b_imm}'
 
                                encoding, count = ks.asm(path_asm)
                                patch_byte = bytes([int(i) for i in encoding])
                                print(f"------------------->{path_asm} = [", end='')
                                for i in encoding:
                                    print("%02x " % i, end='')
                                print(']')
 
                                if has_patch:
                                    ida_bytes.patch_bytes(current_ea - 8, patch_byte)
 
                            """统计"""
                            x_30_ret_total = x_30_ret_total + 1
                            print(
                                f'address:{hex(current_ea)}  sub_8_mnem[{sub_8_mnem}]  sub_8_imm={hex(sub_8_imm)}  b_offset:{hex(b_imm)}')
                """skip"""
                current_ea = current_ea + item_size
 
            print(f"total all ret30 total:{x_30_ret_total}")

```

整体脚本实现相对来说比较啰嗦, 我讲究的是能用就行剩下的交给 GPT 帮我优化，脚本包含我刚开始的修正代码, 当时修正的是错误的大家看看就好了。

*   [x] 第二种花指令, 是隐藏真实跳转地址, 并交给中央转发器统一分发  
    以前没见过就是觉得非常厉害大概的跳转思路设计的很精妙，这也导致了我刚开始无脑`NOP`陷入了痛苦深渊, 直到写这篇文章的时候有开源项目实现过这种隐藏真实跳转的 [LLVM 项目](https://github.com/amimo/goron), 写这篇文章的时候说实话没有看过他的核心思想就是根据样本学习和动态调试, 一步一步扣出来的, 下边结合样本和具体修正思路来看比较好，样本种类很多大家慢慢看别急、别急、
    
*   样本 1
    

```
.text:14594 E0 7B 7E A9                   LDP             X0, X30, [SP,#-0x20]
.text:14598 E0 7B 3B A9                   STP             X0, X30, [SP,#-0x50]
.text:1459C 20 00 80 D2                   MOV             X0, #1
.text:145A0 02 00 00 14                   B               loc_145A8
.text:145A4 06 00 00 00                   DCD 6
.text:145A8 15 02 00 94                   BL              sub_14DFC
.text:145AC 14 00 00 00                   DCD 0x14
.text:145B0 30 00 00 00                   DCD 0x30
.text:145B4 8C 00 00 00                   DCD 0x8C
.text:145B8 B8 00 00 00                   DCD 0xB8
.text:145BC 1F 20 03 D5                   DCD 0xD503201F

```

*   中央转发器

```
.text:14DFC E0 07 BF A9                   STP             X0, X1, [SP,#var_10]!
.text:14E00 C0 5B 60 B8                   LDR             W0, [X30,W0,UXTW#2]
.text:14E04 DE 43 20 8B                   ADD             X30, X30, W0,UXTW
.text:14E08 E0 07 C1 A8                   LDP             X0, X1, [SP+0x10+var_10],#0x10
.text:14E0C C0 03 5F D6                   RET

```

这要结合中央转发器来看, 先说中央转发器都干了什么里边通篇就只做了一个运算操作，最终跳出到目标地址依然利用了 RET X30 的特性这里不过多介绍, 核心突出的就是计算和跳转不要关心`STP`和`LDP`

*   计算就干了一件事情 公式如下 x30=x30+load(x30+(x0<<2))

公式是这样, 读懂需要看上下文刚开始我也很懵, 最后调试几遍就发现了规律, 首先看`x30`与`x0`的值是这么来的计算一遍看看。

 

先看 X30(LR) 已知条件是 LR 寄存器永远指向下一条要执行的指令地址拿样本 1 中来看，当前  
`X30=0x145AC`, 为什么因为调用中央转发器用的是`BL`有返回的跳转

 

再看`x0`的值, 向上溯源调用中央转发器前就有一条`MOV X0, #1`这个 #1 是什么，这里可以理解为跳转编号, 大家可以找到中央转发器看一下`XREF`引用每个调用前都会有 X0 或者 W0 的立即数赋值操作。

*   tips

大家要注意具有迷惑性的 `0x145A4 DCD 0x6` 实际样本中还有一种 x0 赋值方式就是  
`LDR w0,0xc` 这并不是直接把 0xc 赋值给 w0,LDR 是基于 PC 寄存器的一种地址无关性的读取操作, 说人话的意思就是 `加载 pc寄存器地址+0xc 存放的内容放到 w0寄存器中`具体会在脚本中体现因为要考虑到

*   根据样本我们实际算一遍就明白了

```
X30 = 0x145AC +Load(0x145AC+(1<<2))
 
#这一步只是地址计算
0x145AC+0x4 = 0x145B0
#加载这个地址的值
Load(0x145B0)=0x30
 
#实际得出的地址
0x145AC+0x30 = 0x145DC

```

*   核心思想与总结
    
*   [x] 利用中央转发起进行 x30 条转, 配合花指令存储偏移传递不同编号进行跳转，干扰编译器
    
*   [x] 利用花指令保存真实跳转偏移植，屏蔽`脚本小子`无脑 NOP, 让花指令数据变得有意义
*   [x] 样本都嵌套的一层嵌套一层

具体修正与脚本
-------

实际修正脚本也是写了非常多, 还有很多情况需要考虑还好我都写了中文注释

 

find_中央转发器. py

```
/****
   *@author 周樟寿
   */
import ida_bytes
import idautils
import idc
 
if __name__ == '__main__':
    segments = idautils.Segments()
    for seg in segments:
        """这里只处理 代码段"""
        if idc.get_segm_name(seg) in ["__text", ".text"]:
            seg_name = idc.get_segm_name(seg)
            seg_start_addr = idc.get_segm_start(seg)
            seg_end_addr = idc.get_segm_end(seg) - 4
            seg_size = seg_end_addr - seg_start_addr
            print(
                f's_name:{seg_name} s_start:{hex(seg_start_addr)} s_end:{hex(seg_end_addr)}  s_size:{seg_size} has_align:{seg_size % 4 == 0}')
 
            current_ea = seg_start_addr
            while current_ea <= seg_end_addr:
                """判断是不是 ASM"""
                is_asm_code = ida_bytes.is_code(ida_bytes.get_flags(current_ea))
                """ 获取长度 """
                item_size = idc.get_item_size(current_ea)
                if not is_asm_code:
                    create_result = ida_bytes.create_dword(current_ea, 4, True)
                    current_ea = current_ea + 4
                else:
                    c_asm = idc.GetDisasm(current_ea)
                    if c_asm == "ADD             X30, X30, W0,UXTW":
                        print(f'全局跳转.....address:{hex(current_ea)}  asm:{c_asm}')
                    current_ea = current_ea + item_size
 
    print('format script success ........')

```

修正中央跳转. py  
这个脚本的修正思路，通过中央转发器的 xref 引用进行向上修复，前提是通过脚本找到，我贴心的为大家打印了日志和修正开关，还有对跳转位置写了备注

```
import struct
 
import ida_bytes
import idautils
import idc
 
import re
 
from keystone import *
 
 
def op_convert(x):
    if x.startswith("0x"):
        return int(x, 16)
    else:
        return int(x)
 
 
"""
此脚本应用于 特殊花指令跳转........
"""
 
if __name__ == '__main__':
 
    arm64_nop = b"\x1f\x20\x03\xd5"
 
    """
    初始化汇编 引擎 keystone
    """
 
    ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
 
    """
    这里是经过分析 找到得全局分发器入口地址
    """
    # redirects = [0x1065FEBD4, 0x106D959F0]
    redirects = [0x14DFC]
 
    for redirect in redirects:
 
        for xref in idautils.XrefsTo(redirect, 0):
            print('============================================================================')
            print(xref.type, idautils.XrefTypeName(xref.type), 'from', hex(xref.frm), 'to', hex(xref.to))
 
            redirect_base = xref.frm + 4
            print(f'redirect base:{hex(redirect_base)}')
 
            for a_xref in idautils.XrefsTo(xref.frm, 0):
                """获得 x0 后边得立即数"""
                op1 = idc.print_operand(a_xref.frm - 4, 1).replace("=", "").replace("#", "")
                """转换 数字"""
                xx_number = op_convert(op1)
 
                """计算偏移"""
                offset = xx_number << 2
 
                f_bytes = ida_bytes.get_bytes(redirect_base + offset, 4)
 
                n_tuple = struct.unpack('",
                    a_xref.type, idautils.XrefTypeName(a_xref.type), 'from', hex(a_xref.frm), 'to', hex(a_xref.to))
 
                """
                开始修正这些指令
 
                计算方式 = 实际目标地址-当前地址 = 立即数
                b 所得立即数
                """
                has_patch = True
                if has_patch:
 
                    patch_imm = hex(index_arr - a_xref.frm)
                    path_asm = f'b #{patch_imm}'
 
                    encoding, count = ks.asm(path_asm)
                    patch_byte = bytes([int(i) for i in encoding])
                    print(f"------------------->{path_asm} = [", end='')
                    for i in encoding:
                        print("%02x " % i, end='')
                    print(']')
                    ida_bytes.patch_bytes(a_xref.frm, patch_byte)
 
                """
                patch 完 这些指令 要把  操作  x0 和 w0 这种指令也 nop掉 向上 搜索
                """
                if has_patch:
 
                    sub_4 = a_xref.frm - 4
                    sub_8 = a_xref.frm - 8
                    if (idc.print_insn_mnem(sub_4) in ["LDR", "MOV"]) and idc.print_operand(sub_4, 0) in ["X0", "W0"]:
                        ida_bytes.patch_bytes(sub_4, arm64_nop)
                    if (idc.print_insn_mnem(sub_8) in ["LDR", "MOV"]) and idc.print_operand(sub_8, 0) in ["X0", "W0"]:
                        ida_bytes.patch_bytes(sub_8, arm64_nop)
 
            print('============================================================================\n')
 
    print('xref  fix script sucess.......') 
```

后续的畅想 & 吐槽
----------

我只是分享出来我的学习过程，顺便吐槽一下 `开水团 Group`的 APP 很卡，还有就是我在不同意隐私条款的情况下依然在抽样我的设备信息，具体条款我也看了简直太流氓了。其次就是有那么点不单纯窥探，这些脚本并不能够完全修正但 F5 但是可以不致于抓瞎操作，当前这个脚本适用于 安卓和 iOS 端`开水团group` 的产品, 后边我会把修正字符串加密的脚本也放出，兄弟团大概率会把他们的编译器升级，把具体跳转地址也加密了，大家有一起学习 iOS&Android 逆向的朋友可以一起交流

```
base64_decode("aHR0cHM6Ly93ZWl4aW4ucXEuY29tL2cvQXdZQUFLWXhCZnlscG9RMExfdC05WjVRbFB3Q0hhalZmbUI0TF9mUWVRcDJKXzdVTEJmaldwelVBZVlnQWRMLQ==")
https://cli.im/

```

[反勒索软件开发实战篇](https://www.kanxue.com/book-section_list-46.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm) [#工具脚本](forum-161-1-128.htm)