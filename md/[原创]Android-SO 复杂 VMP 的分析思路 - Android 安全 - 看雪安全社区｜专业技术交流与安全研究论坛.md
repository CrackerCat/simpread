> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-290412.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

之前分享过：[[原创]Android-ARM64 的 VMP 分析和还原](https://bbs.kanxue.com/thread-288731.htm)，里面的样本比较简单，IDA 按 F5 可以直接看到伪代码。

**金罡大佬**分享了复杂的样本：[[原创] 用魔法打败魔法：互联网大厂虚拟机分析还原](https://bbs.kanxue.com/thread-286441.htm)，F5 没办法直接看到伪代码，文章里面对 vm_entry 进行 patch，让 IDA 顺利进行反编译。

**简单的简单**大佬分享了 [[原创] VMP 攻略笔记](https://bbs.kanxue.com/thread-290148.htm)，里面更是进一步对最复杂 vm_interpreter 进行 patch，大大提升了分析的效率。

可惜的是，两位大佬分享的更多是 VMP 的**分析结论**，VMP 的**分析思路**较少。而这篇文章主要补充两点：1、结合两位大佬的文章，猜测一下大佬对 VMP 的分析思路；2、进一步优化 patch，在 IDA 上能直观看到：只有一个主分发器 switch，并且 case 直接对应 handle。

先找到 VMP 的入口：vm_entry

正常写代码：caller（调用者）和 callee（被调用者）

```
void caller(args) {
    callee(args);
}

void callee(args) {
    insts;
}


```

因为同一个函数可能被多个函数调用，当 callee（被调用者）被虚拟化，一般不会修改多个 caller（调用者），而只是修改一个 callee（被调用者）本身。

```
void caller(args) {
    callee(args);
}

void callee(args) {
    // 原本的insts变成：
    vm_entry(vinsts, args); 
}


```

换句话说，被虚拟化的函数，内部所有逻辑变成只跳转到虚拟机入口 vm_entry。

利用这个特点，可以定位虚拟机入口：vm_entry

1、只有一个函数调用

2、调用完恢复栈帧

3、紧接着直接 RET

此外，对于大型一些的 apk，同一个 vm_entry 很可能被多个地方调用，因此加上规则：

4、调用 vm_entry 次数多

按照上面的规则，可以写对应的脚本：

```
import idc
import idaapi
import idautils

def get_operands_string(ea):
    """
    获取指定地址指令的操作数字符串（去除助记符和前导/尾随空格）。
    """
    disasm = idc.GetDisasm(ea)
    mnem = idc.print_insn_mnem(ea)
    # 从反汇编行中移除助记符部分
    if disasm.startswith(mnem):
        ops = disasm[len(mnem):].strip()
        return ops
    return ""   # 理论上不会发生

def check_func_pattern(func_ea):
    """
    检查函数是否符合以下精确模式：
       第一条指令：STP             X29, X30, [SP,#-0x10]!
       倒数第三条：BL              sub_xxx
       倒数第二条：LDP             X29, X30, [SP],#0x10
       倒数第一条：RET
    且整个函数中仅此一条 BL 指令。
    """
    # 确保是有效函数
    if not idaapi.get_func(func_ea):
        return False

    # 获取函数内所有指令地址
    insn_addrs = list(idautils.FuncItems(func_ea))
    if len(insn_addrs) < 3:
        return False

    # 1. 检查最后三条指令
    last_three = insn_addrs[-3:]

    # 倒数第三条：必须是 BL（操作数任意）
    bl_ea = last_three[0]
    bl_ops = get_operands_string(bl_ea)
    if idc.print_insn_mnem(bl_ea).upper() != "BL":
        return False
    if not bl_ops.startswith("sub_"):
        return False

    # 倒数第二条：必须是 LDP X29, X30, [SP],#0x10
    ldp_ea = last_three[1]
    if idc.print_insn_mnem(ldp_ea).upper() != "LDP":
        return False
    
    ldp_ops = get_operands_string(ldp_ea)
    if not ldp_ops.startswith("X29, X30, [SP"):
        return False

    # 最后一条：必须是 RET（无操作数）
    ret_ea = last_three[2]
    if idc.print_insn_mnem(ret_ea).upper() != "RET":
        return False

    # 2. 统计整个函数中的 BL 指令，确保只有一条（即倒数第三条）
    bl_count = 0
    for ea in insn_addrs:
        if idc.print_insn_mnem(ea).upper() == "BL":
            bl_count += 1
            if bl_count > 1:   # 超过一条则失败
                return False
    return bl_count == 1

def find_vm_entry():
    func_eas = []
    for func_ea in idautils.Functions():
        if check_func_pattern(func_ea) :
            func_eas.append(func_ea)
            # print(f"0x{func_ea:X}")

    m_vm_entry = {}
    for func_ea in func_eas:
        insns = list(idautils.FuncItems(func_ea))
        target = idc.get_operand_value(insns[-3], 0)
        m_vm_entry[target] = m_vm_entry.get(target, 0) + 1
        # print(f"0x{func_ea:X} 0x{target:X}")

    m_vm_entry = sorted(m_vm_entry.items(), key=lambda item: item[1], reverse=True)
    for vm_entry, count in m_vm_entry:
        # 4、调用vm_entry次数多
        if count > 10 : print(f"vm_entry 0x{vm_entry:X}")

# 调用：
find_vm_entry()

# 输出：
# vm_entry 0x1313F0


```

看下调用的地方，确实类似：vm_entry(vinsts, args); 

```
__int64 __fastcall sub_67B88(__int64 a1, __int64 a2, __int64 a3, __int64 a4)
{
    return sub_1313F0(&unk_168B60, 3672, &off_1E9AA0, a1, a2, a3, a4);
}

67B88  STP  X29, X30, [SP,#-0x10]!		// 保存栈帧
67B8C  MOV  X29, SP 					// 更新栈帧
67B90  MOV  X8, X3 						// 保存参数
67B94  MOV  X9, X2
67B98  MOV  X10, X1
67B9C  MOV  X11, X0
67BA0  ADRP X0, #unk_168B60@PAGE 		// pVC
67BA4  ADRP X2, #off_1E9AA0@PAGE 		// pApi
67BA8  ADD  X0, X0, #unk_168B60@PAGEOFF
67BAC  ADD  X2, X2, #off_1E9AA0@PAGEOFF
67BB0  MOV  W1, #0xE58 					// vc_count
67BB4  MOV  X3, X11 					// arg0
67BB8  MOV  X4, X10 					// arg1
67BBC  MOV  X5, X9 						// arg2
67BC0  MOV  X6, X8						// arg3
67BC4  BL   sub_1313F0					// vm_entry
67BC8  LDP  X29, X30, [SP],#0x10		// 恢复栈帧
68888  RET                              // 直接返回


```

进入 sub_1313F0，会发现不是一个正常的函数。

明明调用 sub_1313F0 的地方有传递多个参数，但 sub_1313F0 内部没看到参数。

这就意味着 ida 的反编译识别有问题，在这里被加了对抗手段。

```
void sub_1313F0() // vm_entry
{
  sub_25D00(4);
  JUMPOUT(0x131404);
}


```

看下 sub_1313F0 具体的汇编

```
1313F0  STP  X0, X30, [SP,#-0x50] 		// 将X0和X30压栈 X0=unk_168B60
1313F4  MOV  X0, #4 					// 将立即数4存入X0
1313F8  B    loc_131400                 // 跳到131400

131400  BL   sub_25D00                  // 跳到025D00

025D00  STP  X0, X1, [SP,#-0x10]! 		// 将X0和X1压入栈 X0=4 X1=3672
025D04  LDR  W0, [X30,W0,UXTW#2] 		// offset = [0x131404 + 4*4] = [0x131414] = 0xF4
025D08  ADD  X30, X30, W0,UXTW 			// X30 = 0x131404 + offset = 0x131404 + 0xF4 = 0x1314F8
025D0C  LDP  X0, X1, [SP],#0x10 		// 从栈中恢复X0和X1
025D10  RET 							// ret之后跳到X30对应的地址

// 跳转表
131404  DCD 0x90
131408  DCD 0x224
13140C  DCD 0xB0
131410  DCD 0x234
131414  DCD 0xF4
131418  DCD 0x1D0
13141C  DCD 0x1F8
131420  DCD 0x218
131424  DCD 0x25C


```

sub_25D00 的核心作用正是根据传入的索引值，动态修改返回地址，实现跳转。

用公式表达就是：目标地址 = 返回地址 + *(返回地址 + 索引 × 4)

简单说，这里通过间接跳转的方式，让 ida 反编译失效。

对应的，我们可以写一个脚本，让间接跳转变成直接跳转：

```
import idc
import idaapi
import idautils
import ida_bytes

def get_index_from_prev_insn(prev_ea):
    """
    从给定地址的前一条指令中提取索引值。
    支持 MOV X0, #imm 和 LDR X0, [PC, #offset] / LDR X0, =imm
    返回索引值，如果无法提取返回 None
    """
    mnem = idc.print_insn_mnem(prev_ea)
    if mnem == "MOV":
        # 检查目标寄存器为 X0 或 W0
        op0 = idc.print_operand(prev_ea, 0)
        if op0 not in ["X0", "W0"]:
            return None
        # 检查源操作数为立即数
        if idc.get_operand_type(prev_ea, 1) != idc.o_imm:
            return None
        return idc.get_operand_value(prev_ea, 1)

    elif mnem == "LDR":
        op0 = idc.print_operand(prev_ea, 0)
        if op0 not in ["X0", "W0"]:
            return None

        op1 = idc.print_operand(prev_ea, 1).upper()
        # 情况1：PC 相对寻址，如 [PC, #0x1234]
        if "[PC," in op1:
            offset = idc.get_operand_value(prev_ea, 1)  # 获取偏移量
            eff_addr = prev_ea + 4 + offset              # 计算有效地址
            return idaapi.get_dword(eff_addr)            # 读取索引值

        # 情况2：伪指令形式 LDR W0, =0x12345678
        elif op1.startswith('='):
            try:
                # 移除 '=' 并解析数值
                imm_str = op1[1:]
                if imm_str.startswith('0X'):
                    return int(imm_str, 16)
                else:
                    return int(imm_str, 0)
            except:
                return None
        else:
            # 其他 LDR 形式无法静态确定索引
            return None
    else:
        return None

def patch_b(ea, target):
    """
    把 ea 处的 B 指令改为跳转到 target
    """
    pc = ea
    imm26 = (target - pc) >> 2
    if imm26 < -(1 << 25) or imm26 >= (1 << 25):
        raise ValueError("目标超出 B 指令范围")
    opcode = 0b000101 << 26
    insn = opcode | (imm26 & 0x03FFFFFF)
    print("Patched 0x{:X} -> B 0x{:X}".format(ea, target))
    ida_bytes.patch_dword(ea, insn)

def bl_2_b(br_func_ea):
    for bl_ref in idautils.XrefsTo(br_func_ea):
        # 1、找到调用sub_25D00的第一层地址
        bl_ea = bl_ref.frm

        # 2、第一层地址+4，得到跳转表：table
        table = bl_ea + 4

        for b_bl_ref in idautils.XrefsTo(bl_ea):
            # 3、根据第一层地址，找到调用的第二层地址
            b_ea = b_bl_ref.frm

            # 4、第二次地址往上找X0，得到索引:index
            mov_ea = idc.prev_head(b_ea)
            index = get_index_from_prev_insn(mov_ea)
            if index is None:
                print(f"Warning: Cannot extract index at {mov_ea:#x} for b_ea {b_ea:#x}, skipping")
                continue
            
            # 5、算是偏移offset: *(table + index*4)
            offset = idaapi.get_dword(table + index * 4)

            # 6、得到最终跳转地址：table + offset
            target = table + offset

             # 7、修改成最终跳转目标
            patch_b(b_ea, target)

bl_2_b(0x25D00)


```

效果如下：

```
1313F8  B    loc_131400
直接变成
1313F8  B    loc_1314F8


```

但是这个时候按 F5 还是反编译出错

```
void sub_1313F0()
{
  JUMPOUT(0x1314F8);
}


```

需要把 patch 的结果保存到 so，然后重新加载 so 到 IDA 里面进行完整分析：

具体步骤：工具栏：Edit -> Patch program -> Apply patches to input file ...

![](https://bbs.kanxue.com/upload/attach/202603/1000535_9KQ24SS989KXR89.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1000535_GMP4R3KS9KSSNVU.webp)

退出当前 ida，重新加载 so，就可看到正常的 vm_entry：sub_1313F0

![](https://bbs.kanxue.com/upload/attach/202603/1000535_ST2HKXUSN6T8SSX.webp)

而这个函数的逻辑比较简单

封装可变参数：pVA，得到返回结果 pResult，然后传给 sub_131680

```
_int64 __fastcall sub_131680(__int64 pResult, unsigned __int64 *a2, unsigned __int64 a3, __int64 a4, double **pVA)


```

![](https://bbs.kanxue.com/upload/attach/202603/1000535_87JG2G8G9Z9GS5S.webp)

进入 sub_131680，IDA 能正常反编译，但这个函数比较大，以结果为导向，直接看 pResult 是怎么处理的。

从截图可以看出，pResult 没有做额外处理，而是直接传给 sub_138518，并且直接返回。所以 sub_131680 这个函数是 vm_ready：只是做一些准备，还没真正进行虚拟代码的解释运行。

![](https://bbs.kanxue.com/upload/attach/202603/1000535_TTPSGYZVK5WAQ44.webp)

进入 sub_138518，提示 13CEF4 不是有效的基本块

![](https://bbs.kanxue.com/upload/attach/202603/1000535_YM9F5JSBH63GVPY.webp)

跳到 13CEF4，可以看到 13CEF4 是一个数据，而不是基本块，并且被引用 XREF

![](https://bbs.kanxue.com/upload/attach/202603/1000535_JVHWPC7CZ2QXAMG.webp)

查看 13CEF4 引用，发现是在. data.rel.ro，这里一般存放的是函数指针或者虚函数表而 ida 根据这个规则，会对里面的地址识别成指向可以执行代码的基本块但 13CEF4 实际上是数据块，导致 ida 进行 F5 反编译的时候出错

![](https://bbs.kanxue.com/upload/attach/202603/1000535_PYFZNXA3BGTWZC3.webp)

而修改的办法就是把指向 13CEF4 改为指向一个新地址，让 13CEF4 不会被识别成代码，这里直接上**简单的简单**大佬的脚本：

```
def val(value, size):
    return value.to_bytes(size, byteorder='little', signed=False)
 
def raw_patch(ea, data):
    import idaapi
    for i, b in enumerate(data):
        idaapi.put_byte(ea + i, b)
    written = ida_bytes.get_bytes(ea, len(data))
    print(f"验证写入: {written.hex()}")
     
raw_patch(0x1F15F0, val(0x144218, 4))
raw_patch(0x144218, val(0xe, 4))


```

修改之后，效果如下：

![](https://bbs.kanxue.com/upload/attach/202603/1000535_G7PJRJXE2C4U8ZH.webp)

而 sub_138518 就可以按 F5 进行反编译，结果如下：

![](https://bbs.kanxue.com/upload/attach/202603/1000535_D8Y45GK8GCJTX2Q.webp)

伪代码可以看出这个函数比较简单，只有两个 case。但是从 CFG 图上看这个函数，会发现实际上很复杂。

![](https://bbs.kanxue.com/upload/attach/202603/1000535_24H6Q3ZH9A8PGJM.webp)

进一步跟踪 switch 和 case 的逻辑，从汇编可以看出这里是特殊的二级跳转表 switch，不是普通的一级跳转表 switch。普通的一级跳转表 switch，对应的 case 是从 0、1、2... 连续的。而特殊的两级跳转表 switch 一般是为了解决稀疏的 case 值，也就是 case 间隔比较大的情况。

```
switch ( *(&off_1F15E0 + (*a7 + 0x2E)) )

13D228  MOV  X24, X6                
13A5A4  LDR  W8, [X24]            // W8 = 0x6
13A5A8  ADRL X23, off_1F15E0
13A5B4  ADD  W8, W8, #0x2E        // W8 = 0x6 + 0x2E = 0x34
13A5B8  LDR  X8, [X23,W8,UXTW#3]  // X8 = *(0x1F15E0 + 0x34 << 3) = *(0x1F1780) =  0x28
13A5C8  LDR  X8, [X23,X8,LSL#3]   // X8 = *(0x1F15E0 + 0x28 << 3) = *(0x1F1720) =  0x13A5D4
13A5D0  BR   X8                   // 跳到 0x13A5D4

其中：
off_1F15E0
1F15E8  DCQ loc_139500
...
1F1720  DCQ loc_13A5D4
...
1F1780  DCB 0x28


```

并且这个二级跳转表还不是固定同一个位置，而是分散开被多处引用

![](https://bbs.kanxue.com/upload/attach/202603/1000535_KA87MQUFHMKUH66.webp)

因此，这里遇到两个麻烦的问题：

```
问题1：switch不是简单的一级跳转，而是复杂的二级跳转，导致IDA反编译有问题
问题2、switch不是统一同一个地方，而是分散到多个地方，导致后续分析会很麻烦


```

对于这个问题，**简单的简单**大佬的解决思路如下：

```
1、用ida python找到所有进行二级跳转的地址
2、用Firda拦截这些地址，得到case对应的最终地址addr
3、再用ida python把对应的二级跳转变成ifelse的判断跳转


```

第 3 步类似 inline hook，构建一段新的 ifelse 基本块，然后原地址 JMP 过去因为第 2 步拦截多个地址，并且对应的 case 有多个，导致 ifelse 基本块也要多个

这个思路确实解决了上面的问题 1，让 IDA 反编译出完整的函数。但问题 2 还是存在，毕竟是多个地方加上各种跳转。

为了方便后续的分析，可以进一步优化，更解决问题 2。对应的思路如下：

```
1、解析二级跳转表：得到case->addr，case不连续
2、变成一级跳转表：填充case->0，得到连续的case
3、新建一个分发器：方便后面统一同一个分发器
4、找到所有二级BR：筛选出二级跳转的地方
5、跳转到分发器：方便IDA反汇编成switch


```

简单来说就是：

```
解决问题1：把二级跳转表变成一级跳转表
解决问题2：把多处分发器变成一处分发器


```

```
13D228  MOV  X24, X6                
13A5A4  LDR  W8, [X24]            // W8 = 0x6
13A5A8  ADRL X23, off_1F15E0
13A5B4  ADD  W8, W8, #0x2E        // W8 = 0x6 + 0x2E = 0x34
13A5B8  LDR  X8, [X23,W8,UXTW#3]  // X8 = *(0x1F15E0 + 0x34 << 3) = *(0x1F1780) =  0x28
13A5C8  LDR  X8, [X23,X8,LSL#3]   // X8 = *(0x1F15E0 + 0x28 << 3) = *(0x1F1720) =  0x13A5D4
13A5D0  BR   X8                   // 跳到 0x13A5D4


```

```
def parse_jumptable(jt):
    mCase2Addr = {}
    base = idc.get_name_ea_simple(jt)
    if base == idc.BADADDR:
        print("Could not find off_1F15E0")
        return
    
    print(f"[+] 1、parse_jumptable at {jt}")

    seen = set()
    for case in range(0, 100):
        # 1、加上0x24得到第1层索引：index1
        index1 = case + 0x2E

        # 2、根据第1层索引，读取第2层索引：index2
        index2 = idc.get_qword(base + index1 * 8)
        if index2 > 50 : continue

        # 3、根据第2层索引，读取最终跳转地址：addr
        addr = idc.get_qword(base + index2 * 8)
        if addr in seen: continue
        seen.add(addr)

        # 4、检查跳转地址是不是有效的代码块：loc_
        if idc.get_name(addr).startswith("loc_") :
            mCase2Addr[case] = addr
            print(f"    case={case:2X} -> addr=0x{addr:X}")

    return mCase2Addr

# 调用
mCase2Addr = parse_jumptable("off_1F15E0")


```

可以看出，case 不是连续的

```
[+] 1、parse_jumptable at off_1F15E0
    case= 1 -> addr=0x138B80
    case= 2 -> addr=0x139400
    case= 6 -> addr=0x13A5D4
    case= A -> addr=0x13BEE8
    case= C -> addr=0x139628
    case= E -> addr=0x13AFDC
    case= F -> addr=0x13A934
    case=13 -> addr=0x13CD74
    case=16 -> addr=0x13BCF4
    case=1D -> addr=0x13B020
    case=1E -> addr=0x13A55C
    case=28 -> addr=0x13D038
    case=2A -> addr=0x13A1AC
    case=2E -> addr=0x1396A8


```

缺失的 case 默认指向 0

```
def find_empty_space(start_ea: int, size: int) -> int:
    """
    在当前二进制的 .text 段中查找一块大小为 size 且全部为 0x00 的空闲空间，
    返回 0x10 对齐的起始地址，找不到则返回 BADADDR。
    """
    import ida_bytes
    import ida_idaapi
    import ida_segment
 
    # 只在 .text 段中查找，和注释含义保持一致
    seg = ida_segment.get_segm_by_name(".text")
    if not seg:
        return ida_idaapi.BADADDR
 
    start = seg.start_ea
    if start_ea != 0 : start = start_ea
    end = seg.end_ea
 
    # 从第一个 0x10 对齐的地址开始，每次步进 0x10，直接在该对齐处读 size 字节检查是否全 0
    ea = (start + 0xF) & ~0xF
    while ea + size <= end:
        data = ida_bytes.get_bytes(ea, size)
        if data is not None and all(b == 0 for b in data):
            return ea
        ea += 0x10
 
    return ida_idaapi.BADADDR

def create_jumptable(mCase2Addr):
    """
    mCase2Addr : 字典，键为 case 值（整数），值为目标地址（整数）。
    """
    if not mCase2Addr:
        print("[-] Empty case map.")
        return

    max_case = max(mCase2Addr.keys())
    num_entries = max_case + 1
    table_size = num_entries * 8
    
    # 找到空闲的代码块
    njt_addr = find_empty_space(0, table_size)
    print(f"[+] 2、Creating jump table at 0x{njt_addr:X} size=0x{table_size:X}")

    # 写入数据，case -> addr
    for i in range(num_entries):
        ea = njt_addr + i * 8
        addr = mCase2Addr.get(i, 0)
        idaapi.patch_bytes(ea, addr.to_bytes(8, 'little'))
        if i in mCase2Addr:
            print(f"    case 0x{i:02X} -> 0x{addr:X} written at 0x{ea:X}")
        else:
            print(f"    case 0x{i:02X} -> 0x000000 written at 0x{ea:X}")

    return njt_addr, table_size

# 调用
jt_addr, jt_size = create_jumptable(mCase2Addr)


```

```
[+] 2、Creating jump table at 0x144220 size=0x178
    case 0x00 -> 0x000000 written at 0x144220
    case 0x01 -> 0x138B80 written at 0x144228
    case 0x02 -> 0x139400 written at 0x144230
    case 0x03 -> 0x000000 written at 0x144238
    case 0x04 -> 0x000000 written at 0x144240
    case 0x05 -> 0x000000 written at 0x144248
    case 0x06 -> 0x13A5D4 written at 0x144250
    case 0x07 -> 0x000000 written at 0x144258
    case 0x08 -> 0x000000 written at 0x144260
    case 0x09 -> 0x000000 written at 0x144268
    case 0x0A -> 0x13BEE8 written at 0x144270
    case 0x0B -> 0x000000 written at 0x144278
    case 0x0C -> 0x139628 written at 0x144280
    case 0x0D -> 0x000000 written at 0x144288
    case 0x0E -> 0x13AFDC written at 0x144290
    case 0x0F -> 0x13A934 written at 0x144298
    case 0x10 -> 0x000000 written at 0x1442A0
    case 0x11 -> 0x000000 written at 0x1442A8
    case 0x12 -> 0x000000 written at 0x1442B0
    case 0x13 -> 0x13CD74 written at 0x1442B8
    case 0x14 -> 0x000000 written at 0x1442C0
    case 0x15 -> 0x000000 written at 0x1442C8
    case 0x16 -> 0x13BCF4 written at 0x1442D0
    case 0x17 -> 0x000000 written at 0x1442D8
    case 0x18 -> 0x000000 written at 0x1442E0
    case 0x19 -> 0x000000 written at 0x1442E8
    case 0x1A -> 0x000000 written at 0x1442F0
    case 0x1B -> 0x000000 written at 0x1442F8
    case 0x1C -> 0x000000 written at 0x144300
    case 0x1D -> 0x13B020 written at 0x144308
    case 0x1E -> 0x13A55C written at 0x144310
    case 0x1F -> 0x000000 written at 0x144318
    case 0x20 -> 0x000000 written at 0x144320
    case 0x21 -> 0x000000 written at 0x144328
    case 0x22 -> 0x000000 written at 0x144330
    case 0x23 -> 0x000000 written at 0x144338
    case 0x24 -> 0x000000 written at 0x144340
    case 0x25 -> 0x000000 written at 0x144348
    case 0x26 -> 0x000000 written at 0x144350
    case 0x27 -> 0x000000 written at 0x144358
    case 0x28 -> 0x13D038 written at 0x144360
    case 0x29 -> 0x000000 written at 0x144368
    case 0x2A -> 0x13A1AC written at 0x144370
    case 0x2B -> 0x000000 written at 0x144378
    case 0x2C -> 0x000000 written at 0x144380
    case 0x2D -> 0x000000 written at 0x144388
    case 0x2E -> 0x1396A8 written at 0x144390


```

模拟一级跳转表 switch 的汇编，创建一个新的分发器：

```
ADRL X23, once_jump_table
LDR X16, [X23,X16,LSL#3]
BR  X16


```

```
def create_dispather(jt_addr, jt_size):
    dispatcher_addr = find_empty_space(jt_addr + jt_size, 4*4)
    print(f"[+] 3、Creating dispather at 0x{dispatcher_addr:X}")
    
    # 1、ADRL X23, jt_addr
    patch_adrl(dispatcher_addr, dispatcher_addr + 4, jt_addr, "X23")

    # 2、LDR X16, [X23, X16, LSL #3]
    patch_ldr_indexed(dispatcher_addr + 8, 16, 23, 16)

    # 3、BR X16
    patch_br(dispatcher_addr + 12, "X16")

    return dispatcher_addr

def patch_adrl(adrp_ea, add_ea, target_addr, reg="X23"):
    """
    在 adrp_ea 写入 ADRP reg, #page
    在 add_ea  写入 ADD reg, reg, #page_off
    并打印类似汇编的格式：
        0xADDR -> ADRP X23, #0x123000
        0xADDR -> ADD  X23, X23, #0x123
    """
    reg_num = int(reg[1:])
    pc = adrp_ea  # ADRP 指令的 PC
    page = target_addr & ~0xFFF
    page_off = target_addr & 0xFFF
    pc_page = pc & ~0xFFF
    imm = (page - pc_page) >> 12
    if imm < -2**20 or imm >= 2**20:
        raise ValueError(f"ADRP offset out of range: {imm:#x}")

    # 构造 ADRP
    immhi = imm >> 2
    immlo = imm & 3
    adrp_insn = (0x90 << 24) | (immlo << 29) | (immhi << 5) | reg_num
    adrp_bytes = adrp_insn.to_bytes(4, 'little')
    idaapi.patch_bytes(adrp_ea, adrp_bytes)

    # 打印 ADRP 汇编
    print(f"    0x{adrp_ea:X} -> ADRP {reg}, #{page:#x}")

    # 构造 ADD
    add_insn = (0x91 << 24) | (page_off << 10) | (reg_num << 5) | reg_num
    add_bytes = add_insn.to_bytes(4, 'little')
    idaapi.patch_bytes(add_ea, add_bytes)

    # 打印 ADD 汇编
    print(f"    0x{add_ea:X} -> ADD  {reg}, {reg}, #{page_off:#x}")

def patch_ldr_indexed(ea, rt, rn, rm, shift=3):
    """
    在地址 ea 写入 LDR Xrt, [Xrn, Xrm, LSL #3] 指令。
    基于已知模板 LDR X9, [X23, X9, LSL#3] = F8 69 7A E9 (小端)
    要求 rn == 23（固定不变），shift == 3。
    """
    if rn != 23:
        raise ValueError("此函数仅支持 rn=23")
    if shift != 3:
        raise ValueError("仅支持 shift=3")

    # 模板指令（小端序值）
    template = 0xF8697AE9   # 对应 LDR X9, [X23, X9, LSL#3]

    # 清除 rt 和 rm 位
    mask_rt = 0x1F          # 低5位
    mask_rm = 0x1F << 16    # 16-20位
    new_val = template & ~mask_rt & ~mask_rm

    # 设置新值
    new_val |= (rt & 0x1F) | ((rm & 0x1F) << 16)

    # 以小端序写入
    ida_bytes.patch_bytes(ea, new_val.to_bytes(4, 'little'))
    print(f"    0x{ea:X} -> LDR X{rt}, [X{rn}, X{rm}, LSL#3]")

def patch_br(ea, reg):
    """
    在地址 ea 写入 BR Xreg
    """
    reg_num = int(reg[1:])
    insn = 0xD61F0000 | (reg_num << 5)
    ida_bytes.patch_bytes(ea, insn.to_bytes(4, 'little'))
    print(f"    0x{ea:X} -> BR {reg}")

# 调用
dispatcher_addr = create_dispather(jt_addr, jt_size)


```

```
[+] 3、Creating dispather at 0x1443A0
    0x1443A0 -> ADRP X23, #0x144000
    0x1443A4 -> ADD  X23, X23, #0x220
    0x1443A8 -> LDR X16, [X23, X16, LSL#3]
    0x1443AC -> BR X16


```

其中

```
0x1443A0 -> ADRP X23, #0x144000
0x1443A4 -> ADD  X23, X23, #0x220


```

等价

```
0x1443A0 -> ADRL X23, #0x144220 // 指向第2步的一级跳转表


```

从下往上找到：BR->LDR->LDR->ADD

```
139DC4 LDR W9, [X9]
139DD0 ADD W9, W9, #0x2E
139DD4 LDR X9, [X23,W9,UXTW#3]
139DD8 LDR X9, [X23,X9,LSL#3]
139DE0 BR  X9


```

```
def find_double_br(ea):
    double_br_infos = []

    print(f"[+] 4、find_double_br")

    # 13A5A4 LDR  W8, [X24]
    # 13A5A8 ADRL X23, off_1F15E0
    # 13A5B4 ADD  W8, W8, #0x2E
    # 13A5B8 LDR  X8, [X23,W8,UXTW#3]
    # 13A5C8 LDR  X8, [X23,X8,LSL#3]
    # 13A5D0 BR   X8
    br_addrs = find_br_in_function(ea)
    for br_addr in br_addrs :
        # 13A5D0 BR   X8 -> X8
        reg_br = idc.print_operand(br_addr, 0)

        # 13A5C8 LDR  X8, [X23,X8,LSL#3] -> 13A5C8, X23
        ldr2_ea, base2 = find_prev_ldr(br_addr, reg_br)
        if ldr2_ea == None : continue

        # 13A5B8 LDR  X8, [X23,W8,UXTW#3] -> 13A5B8, X23
        ldr1_ea, base1 = find_prev_ldr(ldr2_ea, reg_br, base2)
        if ldr1_ea == None : continue
        if base1 != base2 : continue

        add_ea = find_prev_add(ldr1_ea, reg_br)
        if add_ea == None : 
            print(f"no add: 0x{ldr1_ea:x}")
            continue
        
        double_br_info = {
            "add_ea": add_ea,
            "ldr1_ea": ldr1_ea,
            "ldr2_ea": ldr2_ea,
            "br_addr": br_addr,
            "reg_br": reg_br
        }
        double_br_infos.append(double_br_info)
        print(f"    add_ea  0x{double_br_info['add_ea']:x}")
        print(f"    ldr_ea1 0x{double_br_info['ldr1_ea']:x}")
        print(f"    ldr_ea2 0x{double_br_info['ldr2_ea']:x}")
        print(f"    br_addr 0x{double_br_info['br_addr']:x}")
        print(f"    br_addr   {double_br_info['reg_br']}")
        print(f"")
        
    return double_br_infos

def find_br_in_function(ea):
    """
    在给定地址 ea 所在的函数中，查找所有 BR 指令的地址。

    Args:
        ea: 函数内的任意地址 (整数或整数形式的字符串)。

    Returns:
        list: 包含所有 BR 指令地址的列表。如果函数无效或未找到，返回空列表。
    """
    # 将地址转换为整数形式（如果传入的是字符串）
    ea = int(ea) if isinstance(ea, str) else ea

    # 1. 获取给定地址所在的函数
    func = idaapi.get_func(ea)
    if not func:
        print(f"错误：地址 0x{ea:x} 不在任何已知函数内。")
        return []

    func_start = func.start_ea

    br_addrs = []

    # 2. 遍历函数内的所有指令地址
    #    idautils.FuncItems() 返回函数内所有指令的起始地址的迭代器
    for instr_ea in idautils.FuncItems(func_start):
        # 3. 获取当前指令的助记符
        mnemonic = idc.print_insn_mnem(instr_ea)

        # 4. 判断是否为 BR 指令
        if mnemonic == "BR":
            br_addrs.append(instr_ea)
            # print(f"    BR: 0x{instr_ea:x}")

    return br_addrs

def get_reg_number(reg_str):
    """从寄存器字符串（如 'X8' 或 'W8'）中提取编号和宽度"""
    m = re.match(r'([XW])(\d+)', reg_str.upper())
    if m:
        return int(m.group(2)), m.group(1) == 'X'
    return None, None

def parse_mem_operand(ea):
    """
    解析指令的内存操作数，返回 (基址寄存器, 索引寄存器)
    适用于 LDR, LDRSW, LDRSH 等格式，例如 [X9, X8, LSL#2] 或 [X23, W8, UXTW#3]
    """
    mnem = idc.print_insn_mnem(ea)
    if mnem not in ["LDR", "LDRSW", "LDRSH", "LDRB", "LDRH"]:
        return None, None
    op2 = idc.print_operand(ea, 1)          # 第二个操作数，如 "[X9, X8, LSL#2]"
    m = re.search(r'\[([XW]\d+),\s*([XW]\d+)', op2)
    if m:
        base = m.group(1)
        index = m.group(2)
        return base, index
    return None, None

def find_prev_ldr(start_ea, target_reg, base_reg=None, max_instructions=20):
    """
    通用向上查找 LDR 类指令
    """
    ea = start_ea
    for _ in range(max_instructions):
        ea = idc.prev_head(ea)
        if ea == idc.BADADDR:
            break
        if idc.print_insn_mnem(ea) != "LDR":
            continue
        if idc.print_operand(ea, 0) != target_reg:
            continue
        base, index = parse_mem_operand(ea)
        if base is None:
            continue

        target_num, _ = get_reg_number(target_reg)
        idx_num, _ = get_reg_number(index)
        if target_num != idx_num:
            print(f"0x{ea:X}")
            print(target_num, idx_num)
            print(base, index)
            print(target_reg, base_reg)
            continue
        if base_reg is not None and base != base_reg:
            continue
        return ea, base
    return None, None

def find_prev_add(start_ea, target_reg, max_instructions=10):
    """向上查找 ADD 指令，要求 ADD target_reg, imm"""
    ea = start_ea
    for _ in range(max_instructions):
        ea = idc.prev_head(ea)
        if ea == idc.BADADDR:
            break
        if idc.print_insn_mnem(ea) != "ADD":
            continue
        
        add_reg = idc.print_operand(ea, 0)
        add_reg_num, _  = get_reg_number(add_reg)
        target_reg_nun, _  = get_reg_number(target_reg)
        if add_reg_num != target_reg_nun:
            continue
        return ea
    return None

# 调用
double_br_infos = find_double_br(0x138518)


```

```
[+] 4、find_double_br
    add_ea  0x139dd0
    ldr_ea1 0x139dd4
    ldr_ea2 0x139dd8
    br_addr 0x139de0
    br_addr   X9

    add_ea  0x13a184
    ldr_ea1 0x13a188
    ldr_ea2 0x13a198
    br_addr 0x13a1a8
    br_addr   X8

    add_ea  0x13a548
    ldr_ea1 0x13a54c
    ldr_ea2 0x13a550
    br_addr 0x13a558
    br_addr   X10

    add_ea  0x13a5b4
    ldr_ea1 0x13a5b8
    ldr_ea2 0x13a5c8
    br_addr 0x13a5d0
    br_addr   X8

    add_ea  0x13a914
    ldr_ea1 0x13a918
    ldr_ea2 0x13a928
    br_addr 0x13a930
    br_addr   X8

    add_ea  0x13afc8
    ldr_ea1 0x13afcc
    ldr_ea2 0x13afd0
    br_addr 0x13afd8
    br_addr   X8

    add_ea  0x13b00c
    ldr_ea1 0x13b010
    ldr_ea2 0x13b014
    br_addr 0x13b01c
    br_addr   X8


```

把一开始的 case 存入 X16，然后两条 LDR 都 NOP 掉，之后直接跳到分发器

```
139DC4 LDR W9, [X9]
139DD0 ADD W9, W9, #0x2E        -> MOV X16, X9   add_ea=0x139DD0 reg_br=X9
139DD4 LDR X9, [X23,W9,UXTW#3]  -> NOP           ldr1_ea=0x139DD4
139DD8 LDR X9, [X23,X9,LSL#3]   -> NOP           ldr2_ea=0x139DD8
139DE0 BR  X9                   -> B dispatcher  br_addr=0x139DE0


```

```
def double_br_2_dispatcher(double_br_infos, dispatcher_addr):
    print(f"[+] 5、double_br_2_dispatcher")

    for info in double_br_infos:
        add_ea = info["add_ea"]       # 0x139DD0
        ldr1_ea = info["ldr1_ea"]     # 0x139DD4
        ldr2_ea = info["ldr2_ea"]     # 0x139DD8
        br_addr = info["br_addr"]     # 0x139DE0
        reg_br = info["reg_br"]       # 例如 "X9"

        src_reg = int(reg_br[1:])     # 9
        patch_mov(add_ea, 16, src_reg)# MOV X16, X9
        patch_nop(ldr1_ea)            # NOP
        patch_nop(ldr2_ea)            # NOP
        patch_b(br_addr, dispatcher_addr) # B dispatcher
        print("")

def patch_mov(ea, dst, src):
    """
    在地址 ea 写入 MOV Xdst, Xsrc
    参数 dst, src 为寄存器编号（例如 dst=8, src=9 表示 X8, X9）
    机器码格式：ORR Xdst, XZR, Xsrc = 0xAA000000 | (src << 16) | (31 << 5) | dst
    示例：MOV X1, X2  => 0xAA0203E1 (小端序 E1 03 02 AA)
    """
    insn = 0xAA000000 | (src << 16) | (31 << 5) | dst
    ida_bytes.patch_bytes(ea, insn.to_bytes(4, 'little'))
    print(f"    0x{ea:X} -> MOV X{dst}, X{src}")

def patch_nop(ea):
    """在指定地址写入一条 ARM64 NOP (硬编码)"""
    
    # ARM64 NOP 机器码 (4字节)
    ARM64_NOP = b'\x1F\x20\x03\xD5'
    idaapi.patch_bytes(ea, ARM64_NOP)
    print(f"    0x{ea:X} -> NOP")

def patch_b(ea, target):
    """
    在地址 ea 写入无条件跳转 B target
    机器码格式：0x14000000 | (offset>>2 & 0x03FFFFFF)
    其中 offset = target - ea  （因为 PC 就是当前指令地址）
    """
    pc = ea
    offset = target - pc
    if offset & 3:
        raise ValueError("目标地址未4字节对齐")
    imm26 = offset >> 2
    if imm26 < -0x2000000 or imm26 >= 0x2000000:
        raise ValueError("跳转偏移超出范围 (±128MB)")
    insn = 0x14000000 | (imm26 & 0x03FFFFFF)
    ida_bytes.patch_bytes(ea, insn.to_bytes(4, 'little'))
    print(f"    0x{ea:X} -> B {hex(target)}")

# 调用
double_br_2_dispatcher(double_br_infos, dispatcher_addr)


```

```
[+] 5、double_br_2_dispatcher
    0x139DD0 -> MOV X16, X9
    0x139DD4 -> NOP
    0x139DD8 -> NOP
    0x139DE0 -> B 0x1443a0

    0x13A184 -> MOV X16, X8
    0x13A188 -> NOP
    0x13A198 -> NOP
    0x13A1A8 -> B 0x1443a0

    0x13A548 -> MOV X16, X10
    0x13A54C -> NOP
    0x13A550 -> NOP
    0x13A558 -> B 0x1443a0

    0x13A5B4 -> MOV X16, X8
    0x13A5B8 -> NOP
    0x13A5C8 -> NOP
    0x13A5D0 -> B 0x1443a0

    0x13A914 -> MOV X16, X8
    0x13A918 -> NOP
    0x13A928 -> NOP
    0x13A930 -> B 0x1443a0

    0x13AFC8 -> MOV X16, X8
    0x13AFCC -> NOP
    0x13AFD0 -> NOP
    0x13AFD8 -> B 0x1443a0

    0x13B00C -> MOV X16, X8
    0x13B010 -> NOP
    0x13B014 -> NOP
    0x13B01C -> B 0x1443a0

    0x13B6CC -> MOV X16, X8
    0x13B6D0 -> NOP
    0x13B6D4 -> NOP
    0x13B6DC -> B 0x1443a0

    0x13BCE0 -> MOV X16, X8
    0x13BCE4 -> NOP
    0x13BCE8 -> NOP
    0x13BCF0 -> B 0x1443a0

    0x13BED4 -> MOV X16, X8
    0x13BED8 -> NOP
    0x13BEDC -> NOP
    0x13BEE4 -> B 0x1443a0

    0x13C214 -> MOV X16, X8
    0x13C218 -> NOP
    0x13C21C -> NOP
    0x13C224 -> B 0x1443a0

    0x13D024 -> MOV X16, X8
    0x13D028 -> NOP
    0x13D02C -> NOP
    0x13D034 -> B 0x1443a0

    0x13D07C -> MOV X16, X8
    0x13D080 -> NOP
    0x13D084 -> NOP
    0x13D08C -> B 0x1443a0


```

参考前面说的，保存 patch 到 so，退出重新加载 so。会发现这个时候 IDA 依旧反编译有问题，需要手动调整一下。

```
[+] 2、Creating jump table at 0x144220 size=0x178


```

进入 0x144220，改成成数组：

![](https://bbs.kanxue.com/upload/attach/202603/1000535_TS2V9VYNRRNED4W.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1000535_GEJN4YNXX5PJMXX.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1000535_4MYZPMDJ9VWGK7E.webp)

```
[+] 3、Creating dispather at 0x1443A0


```

进入 0x1443A0，手动设置成 switch：

![](https://bbs.kanxue.com/upload/attach/202603/1000535_NYXE4DUXQZ7QU9G.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1000535_E7WATQTRJ8FV35H.webp)

按一下 Tab 键，就可以顺利进行反编译，可以直观看到有一个核心的 switch

![](https://bbs.kanxue.com/upload/attach/202603/1000535_9MT8P3YYR4H6CQ5.webp)

VM 的解释器是按 v_opcode 进行分发

```
switch ( v_opcode )


```

v_opcode 又是从 v_insts 读取

```
v_opcode = *v_insts;
switch ( v_opcode )


```

主操作码 v_opcode 的 case 1 里面，又有一个 swich，对应的是子操作码 v_sub_opcode 

```
switch ( v_opcode )
  case 1:
    switch ( v_sub_opcode )


```

子操作码是从前面的 v_insts 读取，而偏移就是虚拟程序计数器 v_pc

```
case 1:
  v_sub_opcode = v_insts[v_pc];
  switch ( v_sub_opcode )


```

v_pc 更新之后，和指令的数量 v_inst_count 比较，判断是否退出

```
LABEL_765:
  if ( v_pc + 5 >= v_inst_count )
    return 0;


```

从 case 1 里面的 case 0 可以看到异或的运算符：^

ARM64 里面异或指令的格式如下：

**EOR <Xd>, <Xn>, <Xm>**

![](https://bbs.kanxue.com/upload/attach/202603/1000535_DSF2FJQMEHQSPE8.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1000535_EDSDT7YBMXRJHSY.webp)

从 v_Xd 的赋值可以看出是基址加偏移，基址就是寄存器列表：v_Xregs

```
v_Xd = (v_Xregs + 24 * v_d); 
// 对应 v_Xd = v_Xregs[v_d]


```

寄存器的编号 v_d 正是从 v_insts 得到的

```
v_d = v_insts[v_pc + 4];


```

在子 case 0 下面，还进一步分类

```
case 1 :
  v_TypeObj = *(v_Types + 8 * v_t);
  case 0 :
    v_type = (*(*v_TypeObj + 32LL))(*(v_Types + 8 * v_t));
    if ( v_type == 4 )
      v_Xd = v_Xn ^ v_Xm;


```

类型的索引 v_t 也是来自 v_insts

```
v_t = v_insts[v_pc + 1];


```

综合上面的信息，可以得知 case 1 里面的 case 0，模拟的是：

****EOR <Xd>, <Xn>, <Xm>****

从 v_insts 读取的硬编码格式如下：

<table><thead><tr><th>主操作码</th><th>子操作码</th><th>类型</th><th>第二个操作数</th><th>第一个操作数</th><th>目标寄存器</th></tr></thead><tbody><tr><td>v_opcode</td><td>v_sub_opcode</td><td>v_t</td><td>v_m</td><td>v_n</td><td>v_d</td></tr></tbody></table>

![](https://bbs.kanxue.com/upload/attach/202603/1000535_UEPZXP4ZMH29BJN.webp)

按照这个思路就可以得到 case 1 是逻辑与运算的大类，里面的子类分别是：

<table><thead><tr><th>Case</th><th>符号</th><th>ARM64 指令</th><th>操作意义</th></tr></thead><tbody><tr><td>0</td><td>^</td><td>EOR Xd, Xn, Xm</td><td>按位异或操作</td></tr><tr><td>1</td><td>-</td><td>SUB Xd, Xn, Xm</td><td>减法操作</td></tr><tr><td>2</td><td>&gt;&gt;</td><td>ASR Xd, Xn, Xm</td><td>算术右移，高位补符号位</td></tr><tr><td>3</td><td>/</td><td>UDIV Xd, Xn, Xm</td><td>无符号整数除法</td></tr><tr><td>4</td><td>+</td><td>ADD Xd, Xn, Xm</td><td>加法操作</td></tr><tr><td>5</td><td>|</td><td>ORR Xd, Xn, Xm</td><td>按位或操作</td></tr><tr><td>6</td><td>%</td><td>SMOD / UMOD</td><td>整数取模</td></tr><tr><td>7</td><td>/</td><td>SDIV Xd, Xn, Xm</td><td>有符号整数除法</td></tr><tr><td>8</td><td>%</td><td>FMOD / FMODF</td><td>浮点取模</td></tr><tr><td>9</td><td>*</td><td>MUL Xd, Xn, Xm</td><td>乘法操作</td></tr><tr><td>10</td><td>&gt;&gt;</td><td>LSR Xd, Xn, Xm</td><td>逻辑右移，高位补 0</td></tr><tr><td>11</td><td>&lt;&lt;</td><td>LSL Xd, Xn, Xm</td><td>逻辑左移</td></tr><tr><td>12</td><td>&amp;</td><td>AND Xd, Xn, Xm</td><td>按位与操作</td></tr></tbody></table>

分析具体的 handle 的整个过程还是比较舒服的。这就是为什么一开始要对 so 进行 patch 的原因。

说到底，有 VMP 和没 VMP，对于机器来说只是算力上多与少的差异；但对人来说理解的难度是天差地别。因此，如果能让 IDA 顺利反编译，分析者就可以从汇编层面上升到代码层面，减少很多理解成本。

对 IDA 的高级用法不熟悉，即便是新建一个 switch 也失败了好几次；加上写脚本也要关注好多细节，导致时常想放弃。好在最后还是耐下心折腾了出来。与其说 VMP 是一堵高墙要去攀爬，不如说 VMP 是一面镜子，能照出是否静下心来。

最后，感谢两位大佬分析的文章，领进了门，受益匪浅！

[[培训]Windows 内核深度攻防：从 Hook 技术到 Rootkit 实战！](https://www.kanxue.com/book-section_list-220.htm)

最后于 2 小时前 被 GhHei 编辑 ，原因： 微调