> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-290148.htm)

> [原创] VMP 攻略笔记

前言
--

本篇笔记记录的是一次围绕 VMP 样本的完整逆向实战：从入口定位、混淆对抗、执行路径还原，到关键机制理解与自研虚拟化加固实现。项目地址：https://github.com/lxz-jiandan/VmpProject

逆向分析环境
------

Python 版本：3.11

IDA 版本：9.2

Frida 版本：hluda-server-16.0.10

样本：金罡大佬同款

虚拟机定位
-----

VMP 的入口函数特征确实不太好找，但根据 VMP 开发的常见思路，虚拟机入口函数一般都会被 Wrapper 函数包装调用，这些 Wrapper 函数一般拥有如下的特征，我们可以据此推测一些比较可疑的 VMP 入口地址，再配合 Frida 具体定位

1.  规模：指令数量极少，一般少于 20 条指令
2.  调调用函数数量限制：仅存在一条跳转指令
3.  调用函数方式限制：存在一条固定目标跳转指令（BL 跳转指令）
4.  返回方式：以 ret 结束
5.  尾指令数量：CALL 和 RET 之间的指令数量一般不会太多，一般应小于 3
6.  返回结果：CALL 之后不修改 X0/W0

```
# Wrapper 函数特征
# 1. 规模：指令数量极少，一般少于 20 条指令
# 2. 调调用函数数量限制：仅存在一条跳转指令
# 3. 调用函数方式限制：存在一条固定目标跳转指令（BL跳转指令）
# 4. 返回方式：以 ret 结束
# 5. 尾指令数量：CALL和RET之间的指令数量一般不会太多，一般应小于3
# 6. 返回结果：CALL 之后不修改 X0/W0
 
import ida_funcs
import ida_ua
import idautils
import idaapi
import ida_idp
 
# =============================================================================
# 配置
# =============================================================================
MAX_INSNS = 30             # wrapper 最大指令数（对应特征 1：规模）
MIN_WRAPPER_COUNT = 15     # 至少几个 wrapper 指向同一 VM Entry
 
# =============================================================================
# 工具函数（规模 / 调用关系 / 输出 检查用）
# =============================================================================
 
def iter_func_insns(func):
    """遍历函数内指令（IDA 9.2：func_item_iterator_t）"""
    fii = ida_funcs.func_item_iterator_t()
    if not fii.set(func):
        return
    for ea in fii.code_items():
        yield ea
 
def is_call_insn(ea):
    """是否为 call 指令（用于检查“仅一次 call”）"""
    insn = ida_ua.insn_t()
    if ida_ua.decode_insn(insn, ea):
        return ida_idp.is_call_insn(insn)
    return False
 
def writes_return_reg(ea):
    """是否存在写入X0/W0寄存器 """
    insn = ida_ua.insn_t()
    if ida_ua.decode_insn(insn, ea) == 0:
        return False
    if not ida_idp.has_insn_feature(insn.itype, ida_idp.CF_CHG1):
        return False
    op0 = insn.ops[0]
    if op0.type != ida_ua.o_reg:
        return False
    width = ida_ua.get_dtype_size(op0.dtype)
    reg_name = ida_idp.get_reg_name(op0.reg, width)
    return reg_name in ("X0", "W0")
 
# 获取函数中所有CALL指令
def get_call_list(func):
    call_list = []
    for ea in iter_func_insns(func):
        if is_call_insn(ea):
            call_list.append(ea)
    return call_list
 
# 获取 CALL 指令后面的所有指令（不含 call 本身）
def get_call_tail(func):
    call_list = get_call_list(func)
    if not call_list:
        return []
    tail_insns = []
    for ea in iter_func_insns(func):
        if ea > call_list[0]:
            tail_insns.append(ea)
    return tail_insns
 
def analyze_wrapper(func):
    """若 func 符合 Wrapper 特征，返回其唯一调用的 VM Entry 地址；否则返回 None。"""
    insns = list(iter_func_insns(func))
 
    # 1. 规模：指令数量极少
    if len(insns) == 0 or len(insns) > MAX_INSNS:
        return None
 
    # 2. 调用函数数量限制：仅存在一条跳转指令
    call_list = get_call_list(func)
    if len(call_list) != 1:
        return None
 
    # 3. 调用函数方式限制：存在一条固定目标跳转指令（BL跳转指令）
    targets = list(idautils.CodeRefsFrom(call_list[0], False))
    if len(targets) != 1:
        return None
 
    # 4. 返回方式：以 ret 结束
    if not idaapi.is_ret_insn(insns[-1]):
        return None
 
    # 5. 尾指令数量：CALL和RET之间的指令数量应小于5
    tail_insns = get_call_tail(func)
    if len(tail_insns) == 0 or len(tail_insns) > 3:
        return None
 
    # 6. 返回结果：CALL 之后不修改 X0/W0
    for ea in tail_insns:
        if writes_return_reg(ea):
            return None
     
    callee_func = ida_funcs.get_func(targets[0])
    if not callee_func:   # 目标不在任何函数内（如 thunk）则排除
        return None
    return callee_func.start_ea
 
def find_vm_entry_candidates():
    wrapper_map = {}
    for func_ea in idautils.Functions():
        func = ida_funcs.get_func(func_ea)
        if not func:
            continue
        vm_entry = analyze_wrapper(func)
        if not vm_entry:
            continue
        wrapper_map.setdefault(vm_entry, []).append(func.start_ea)
 
    print("\n====== VM Entry Candidates ======")
    for vm_entry, wrappers in wrapper_map.items():
        if len(wrappers) >= MIN_WRAPPER_COUNT:
            print(f"\n[+] VM Entry Candidate: {hex(vm_entry)} ({len(wrappers)} wrappers)")
            for w in wrappers:
                print(f"    wrapper: {hex(w)}")
 
def main():
    find_vm_entry_candidates()
 
main()

```

混淆对抗
----

### BL 混淆还原

防守方设计了大量的 BL 混淆代码，这里没啥说的，直接上罡佬的脚本，执行后可以去除 BL 混淆，注意要重新加载分析文件，Apply patches to input file --> 退出 IDA ---- 重新加载 so

```
import idautils
import idc
import idaapi
from keystone import *  # pip3 install keystone-engine
def get_insn_const(addr):
    op_val = None
    if idc.print_insn_mnem(addr) in ['MOV', 'LDR']:
        op_val = idc.get_operand_value(addr, 1)
        if op_val > 0x1000:  # 可能是间接引用
            op_val = idc.get_wide_dword(op_val)
    else:
        raise Exception(f"error ops const: {addr}")
    return op_val
def get_patch_data(addr):
    addr_list = []
    for bl_insn_addr in idautils.XrefsTo(addr):
        bl_insn_addr = bl_insn_addr.frm
        # print(f'L1 {hex(bl_insn_addr)}:')
        for xref_addr_l2 in idautils.XrefsTo(bl_insn_addr):               
            # print(f'\tL2 {hex(xref_addr_l2.frm)}:')
            index = get_insn_const(xref_addr_l2.frm - 4)
            const_table_start = bl_insn_addr + 4
            offset = idaapi.get_dword(const_table_start + index * 4)
            link_target = const_table_start + offset
            addr_list.append({"bl_insn_addr": bl_insn_addr, "patch_addr": xref_addr_l2.frm, "index": index,
                        "offset": offset, "link_target": link_target})
    return addr_list
def print_patch_data(patch_data):
    for item in patch_data:
        print(
            f"bl_insn_addr: {item["bl_insn_addr"]:#x}, patch_addr: {item["patch_addr"]:#x}, index: {item["index"]}, offset: {item["offset"]:#x}, link_target: {item["link_target"]:#x}")
def patch_insns(patch_data):
    index = 0
    for item in patch_data:
        ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
        asm = f'B {item["link_target"]:#x}'
        print(f'patch addr {item["patch_addr"]:#x}: {asm}')
        encoding, count = ks.asm(asm, as_bytes=True, addr=item["patch_addr"])
        print(encoding)
        for i in range(4):
            idc.patch_byte(item["patch_addr"] + i, encoding[i])
        index += 1
        # if index == 1:
        #     break       
def start():
    modify_x30_func_address = 0x25D00
    patch_data = get_patch_data(modify_x30_func_address)
    print_patch_data(patch_data)
    patch_insns(patch_data)
start()

```

### F5 干扰处理

防守方设计了干扰 F5 的代码，但不过是迷惑 IDA 的障眼法，我们调整一下重定位表，执行后就可以 F5 了

1.  直接对 sub_138518 函数直接 F5 会提示 `13CEF4: invalid basic block`
    
2.  此时我们查看 13CEF4 会发现，IDA 将 13CEF4 识别成代码块了
    
    ```
    .text:000000000013CEF4 0E 00 00 00 dword_13CEF4 DCD 0xE ; DATA XREF: .data.rel.ro:off_1F15E0↓o
    
    ```
    
    ```
    .data.rel.ro:1F15E0 A8 96 13 00 00 00 00 00 off_1F15E0 DCQ loc_1396A8       ; DATA XREF: sub_138518+1498↑o
    .data.rel.ro:1F15E0                                                         ; sub_138518+14C0↑o ...
    .data.rel.ro:1F15E8 00 95 13 00 00 00 00 00            DCQ loc_139500
    .data.rel.ro:1F15F0 F4 CE 13 00 00 00 00 00            DCQ dword_13CEF4
    .data.rel.ro:1F15F8 00 94 13 00 00 00 00 00            DCQ loc_139400
    .data.rel.ro:1F1600 28 96 13 00 00 00 00 00            DCQ loc_139628
    .data.rel.ro:1F1608 20 B0 13 00 00 00 00 00            DCQ loc_13B020
    .data.rel.ro:1F1610 D4 A7 13 00 00 00 00 00            DCQ loc_13A7D4
    .data.rel.ro:1F1618 F4 CF 13 00 00 00 00 00            DCQ loc_13CFF4
    .data.rel.ro:1F1620 00 97 13 00 00 00 00 00            DCQ dword_139700
    
    ```
    
3.  我们可以将 1F15F0 中的地址指向一个空白的代码段然后将 13CEF4 的值直接写入，随后即可成功执行 F5 反汇编
    
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
    

### BR 跳转还原

防守方在开发时大量的使用跳转表的结构，编译之后就形成了类似 BR 的混淆代码，虽然我们确实可以找到跳转表，但 IDA 经常无法识别跳转表或者识别跳转表错误或者是以函数指针的形式调用，导致 F5 看不全代码，我们可以分析一下跳转表的逻辑，然后构造 IDA 易于识别的跳转逻辑

1.  在 IDA 中模糊匹配无法正确建立 switch case 表的 BR X8/9/10 跳转块
    
    ```
    def search_wildcard_ex(hex_str1, *args):
        """
        search_wildcard_ex(
            hex_str1,
            gap1, hex_str2,
            gap2, hex_str3,
            ...
        )
     
        gap 语义：
            前一个 pattern 结束地址 与 下一个 pattern 起始地址 之间
            允许的最大字节数（不包含 pattern 自身）
        """
     
        import ida_bytes
        import ida_ida
        import ida_idaapi
     
        # -------------------------
        # 1. 参数校验
        # -------------------------
        if len(args) % 2 != 0:
            raise ValueError("arguments must be (gap, hex_str) pairs")
     
        patterns = [(hex_str1, None)]
        for i in range(0, len(args), 2):
            patterns.append((args[i + 1], args[i]))
     
        # -------------------------
        # 2. 工具函数
        # -------------------------
        def compile_pat(hex_str):
            cpv = ida_bytes.compiled_binpat_vec_t()
            base = ida_ida.inf_get_min_ea()
            err = ida_bytes.parse_binpat_str(cpv, base, hex_str, 16)
            if err:
                raise RuntimeError("binpat compile failed: %s" % hex_str)
            return cpv
     
        def pat_len_from_hex(hex_str):
            # "?? 7A ?? F8" -> 4
            return len(hex_str.strip().split())
     
        # -------------------------
        # 3. 编译 pattern
        # -------------------------
        pats = []
        for hex_str, gap in patterns:
            cpv = compile_pat(hex_str)
            pat_len = pat_len_from_hex(hex_str)
            pats.append((hex_str, cpv, pat_len, gap))
     
        start = ida_ida.inf_get_min_ea()
        end   = ida_ida.inf_get_max_ea()
     
        # -------------------------
        # 4. anchor 搜索
        # -------------------------
        results = []
        addr = start
     
        while True:
            addr, _ = ida_bytes.bin_search(
                addr, end,
                pats[0][1],
                ida_bytes.BIN_SEARCH_FORWARD
            )
            if addr == ida_idaapi.BADADDR:
                break
     
            cur_ea  = addr
            cur_len = pats[0][2]
            matched = True
            chain = [addr]  # 本条匹配链：各 pattern 的起始地址
     
            # -------------------------
            # 5. 逐段验证
            # -------------------------
            for i in range(1, len(pats)):
                _, cpv, pat_len, gap = pats[i]
     
                # gap = 前一个结束 与 下一个起始 之间的最大字节数（含下一段起始）
                # 下一段起始可在 [prev_end, prev_end+gap]，故搜索范围需覆盖到 prev_end+gap+pat_len（右开）
                search_start = cur_ea + cur_len
                search_end   = cur_ea + cur_len + gap + pat_len
     
                ea2, _ = ida_bytes.bin_search(
                    search_start,
                    search_end,
                    cpv,
                    ida_bytes.BIN_SEARCH_FORWARD
                )
     
                if ea2 == ida_idaapi.BADADDR:
                    matched = False
                    break
     
                chain.append(ea2)
                cur_ea  = ea2
                cur_len = pat_len
     
            if matched:
                results.append(chain)
     
            addr += 1  # 允许重叠
     
        if not results:
            print("[-] no match")
        else:
            match_map = {chain[0]: chain[1:] for chain in results}
            lines = ["    {}: [{}]".format(hex(k), ", ".join(hex(x) for x in v)) for k, v in sorted(match_map.items())]
            print("match_map = {")
            print(",\n".join(lines))
            print("}")
     
        return results
     
    # .text:000000000013A5B4 08 B9 00 11                 ADD             W8, W8, #0x2E ; '.'
    # .text:000000000013A5B8 E8 5A 68 F8                 LDR             X8, [X23,W8,UXTW#3]
    # .text:000000000013A5BC F9 03 1F 2A                 MOV             W25, WZR
    # .text:000000000013A5C0 F3 03 00 32                 MOV             W19, #1
    # .text:000000000013A5C4 F5 63 08 A9                 STP             X21, X24, [SP,#0x120+var_A0]
    # .text:000000000013A5C8 E8 7A 68 F8                 LDR             X8, [X23,X8,LSL#3]
    # .text:000000000013A5CC E0 7B 3B A9                 STP             X0, X30, [SP,#0x120+var_170]
    # .text:000000000013A5D0 00 01 1F D6                 BR              X8 ; apply_bit_mask compare_float_double
    #
    # .text:0000000000139DD0 29 B9 00 11                 ADD             W9, W9, #0x2E ; '.'
    # .text:0000000000139DD4 E9 5A 69 F8                 LDR             X9, [X23,W9,UXTW#3]
    # .text:0000000000139DD8 E9 7A 69 F8                 LDR             X9, [X23,X9,LSL#3]
    # .text:0000000000139DDC E0 7B 3B A9                 STP             X0, X30, [SP,#0x120+var_170]
    # .text:0000000000139DE0 20 01 1F D6                 BR              X9
    #
    # .text:000000000013A548 4A B9 00 11                 ADD             W10, W10, #0x2E ; '.'
    # .text:000000000013A54C EA 5A 6A F8                 LDR             X10, [X23,W10,UXTW#3]
    # .text:000000000013A550 EA 7A 6A F8                 LDR             X10, [X23,X10,LSL#3]
    # .text:000000000013A554 E0 7B 3B A9                 STP             X0, X30, [SP,#0x120+var_170]
    # .text:000000000013A558 40 01 1F D6                 BR              X10
    #
    # .text:000000000013A184 08 B9 00 11                 ADD             W8, W8, #0x2E ; '.'
    # .text:000000000013A188 E8 5A 68 F8                 LDR             X8, [X23,W8,UXTW#3]
    # .text:000000000013A18C 5A 17 00 11                 ADD             W26, W26, #5
    # .text:000000000013A190 F5 03 13 AA                 MOV             X21, X19
    # .text:000000000013A194 FC 03 18 AA                 MOV             X28, X24
    # .text:000000000013A198 E8 7A 68 F8                 LDR             X8, [X23,X8,LSL#3]
    # .text:000000000013A19C F8 03 09 AA                 MOV             X24, X9
    # .text:000000000013A1A0 F3 03 1A 2A                 MOV             W19, W26
    # .text:000000000013A1A4 E0 7B 3B A9                 STP             X0, X30, [SP,#0x120+var_170]
    # .text:000000000013A1A8 00 01 1F D6                 BR              X8
     
    search_wildcard_ex(
        "?? B9 00 11 ?? 5A ?? F8",
        12, "?? 7A ?? F8",
        12, "?? 01 1F D6"
    )
    -----------------------------------------------------------------
    match_map = {
        0x139dd0: [0x139dd8, 0x139de0],
        0x13a184: [0x13a198, 0x13a1a8],
        0x13a548: [0x13a550, 0x13a558],
        0x13a5b4: [0x13a5c8, 0x13a5d0],
        0x13a914: [0x13a928, 0x13a930],
        0x13afc8: [0x13afd0, 0x13afd8],
        0x13b00c: [0x13b014, 0x13b01c],
        0x13b6cc: [0x13b6d4, 0x13b6dc],
        0x13bce0: [0x13bce8, 0x13bcf0],
        0x13bed4: [0x13bedc, 0x13bee4],
        0x13c214: [0x13c21c, 0x13c224],
        0x13d024: [0x13d02c, 0x13d034],
        0x13d07c: [0x13d084, 0x13d08c]
    }
    
    ```
    
2.  利用 frida 批量 hook 建立 x23 和 x8/9/10 的映射表
    
    ```
    let trace_map = new Map();
     
    function _py_str(s) {
        return '"' + String(s).replace(/\\/g, "\\\\").replace(/"/g, '\\"') + '"';
    }
     
    function dump_as_python_dict(trace_map) {
        if (!trace_map || typeof trace_map.forEach !== "function") {
            console.log("{}");
            return;
        }
        let lines = [];
        trace_map.forEach(function (node, pc_hex) {
            let br_reg = (node && node.br_reg) ? node.br_reg : "";
            let values = (node && node.values) ? node.values : new Map();
            let innerParts = [];
            values.forEach(function (val, xn_1) {
                if (Array.isArray(val)) {
                    let listStr = "[" + val.map(function (s) { return _py_str(s); }).join(", ") + "]";
                    innerParts.push(_py_str(xn_1) + ": " + listStr);
                } else {
                    innerParts.push(_py_str(xn_1) + ": " + _py_str(val));
                }
            });
            let valuesStr = "{" + innerParts.join(", ") + "}";
            lines.push("  " + _py_str(pc_hex) + ": {\"br_reg\": \"" + br_reg + "\", \"values\": " + valuesStr + "}");
        });
        console.log("{\n" + lines.join(",\n") + "\n}");
    }
     
    function hook_libmtguard_by_offset(base) {
     
        // [[0x139dd0, 0x139dd8, 0x139de0], [0x13a184, 0x13a198, 0x13a1a8], [0x13a548, 0x13a550, 0x13a558], [0x13a5b4, 0x13a5c8, 0x13a5d0], [0x13a914, 0x13a928, 0x13a930], [0x13afc8, 0x13afd0, 0x13afd8], [0x13b00c, 0x13b014, 0x13b01c], [0x13b6cc, 0x13b6d4, 0x13b6dc], [0x13bce0, 0x13bce8, 0x13bcf0], [0x13bed4, 0x13bedc, 0x13bee4], [0x13c214, 0x13c21c, 0x13c224], [0x13d024, 0x13d02c, 0x13d034], [0x13d07c, 0x13d084, 0x13d08c]]
         
        let hook_addr_map = new Map();
        hook_addr_map.set(0x139dd0, "x9");
        hook_addr_map.set(0x13a184, "x8");
        hook_addr_map.set(0x13a548, "x10");
        hook_addr_map.set(0x13a5b4, "x8");
        hook_addr_map.set(0x13a914, "x8");
        hook_addr_map.set(0x13afc8, "x8");
        hook_addr_map.set(0x13b00c, "x8");
        hook_addr_map.set(0x13b6cc, "x8");
        hook_addr_map.set(0x13bce0, "x8");
        hook_addr_map.set(0x13bed4, "x8");
        hook_addr_map.set(0x13c214, "x8");
        hook_addr_map.set(0x13d024, "x8");
        hook_addr_map.set(0x13d07c, "x8");
     
        hook_addr_map.forEach(function (br_reg_name, off) {
     
            let addr = base.add(off)
     
            Interceptor.attach(addr, {
                onEnter() {
     
                    let changed = false;
     
                    let pc_offset = this.context.pc.sub(base);
                    let pc_offset_hex = "0x" + pc_offset.toString(16);
                    console.log("pc_offset_hex =", pc_offset_hex);
     
                    let reg = this.context[br_reg_name];
                    let xn_value_1 = reg.toInt32();
                    let xn_value_1_hex = "0x" + xn_value_1.toString(16);
                    console.log("xn_value_1_hex =", xn_value_1_hex);
     
                    let xn_value_2 = xn_value_1 + 0x2e;
                    let xn_value_2_hex = "0x" + xn_value_2.toString(16);
                    console.log("xn_value_2_hex =", xn_value_2_hex);
     
                    let xn_value_3 = this.context.x23.add(xn_value_2 * 8).readU64();
                    let xn_value_3_hex = "0x" + xn_value_3.toString(16);
                    console.log("xn_value_3_hex =", xn_value_3_hex);
     
                    let xn_value_4 = this.context.x23.add(xn_value_3 * 8).readU64();
                    let xn_value_4_hex = "0x" + xn_value_4.toString(16);
                    console.log("xn_value_4_hex =", xn_value_4_hex);
     
                    let xn_value_5 = ptr(xn_value_4).sub(base);  // 有符号差值
                    let xn_value_5_hex = "0x" + (xn_value_5 >>> 0).toString(16);  // 按无符号显示偏移
                    console.log("xn_value_5_hex =", xn_value_5_hex);
                    
                    if(!trace_map.has(pc_offset_hex)) {
                        trace_map.set(pc_offset_hex, {
                            br_reg: br_reg_name,
                            values: new Map()
                        });
                        changed = true;
                    }
     
                    let pc_node = trace_map.get(pc_offset_hex);
                    if(!pc_node.values.has(xn_value_1_hex)) {
                        pc_node.values.set(xn_value_1_hex, [xn_value_3_hex, xn_value_5_hex]);
                        changed = true;
                    }
     
                    if (!changed)
                        return;
     
                    dump_as_python_dict(trace_map);
                    console.log("======================");
                }
            });
        });
    }
     
    function hook_linker_load() {
        var dlopen = Module.findExportByName(null, "dlopen");
        var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
     
        if (dlopen) {
            console.log("[+] dlopen @", dlopen);
            Interceptor.attach(dlopen, {
                onEnter: function (args) {
                    var name = args[0].readCString();
                    if (name && name.indexOf(".so") > -1) {
                        // console.log("dlopen --> " + name);
                    }
                }
            });
        } else {
            console.log("[-] dlopen not found");
        }
     
        if (android_dlopen_ext) {
            console.log("[+] android_dlopen_ext @", android_dlopen_ext);
            Interceptor.attach(android_dlopen_ext, {
                onEnter: function (args) {
                    this.soname = args[0].readCString();
                    // console.log("android_dlopen_ext --> " + this.soname);
                },
                onLeave: function () {
                    if (!this.soname) return;
     
                    if (this.soname.indexOf("libxxguard.so") === -1) {
                        return;
                    }
     
                    var base = Module.findBaseAddress("libxxguard.so");
                    if (!base) {
                        base = Module.findBaseAddress(this.soname);
                    }
     
                    if (!base) {
                        console.log("[-] libxxguard.so base not found yet");
                        return;
                    }
     
                    console.log("[+] libxxguard.so base =", base);
                    console.log("[+] libxxguard.so pid =", Process.id);
     
                    hook_libmtguard_by_offset(base);
                }
            });
        } else {
            console.log("[-] android_dlopen_ext not found");
        }
    }
     
    function main() {
        hook_linker_load();
    }
     
    setImmediate(main);
    
    ```
    
3.  批量 patch BR 跳转，执行后此时 IDA 已经可以较好的反汇编生成伪 C 代码
    
    ```
    def set_asm(addr: int, asm: str) -> int:
        """
        在给定地址 addr 处写入一条 ARM64 汇编指令 asm（单行字符串）。
        返回实际汇编出的指令条数（通常为 1）。
        """
        # 懒加载依赖，便于拷贝到其他工程中使用
        import ida_bytes
        import ida_ua
        import ida_kernwin
        import ida_idaapi
        from keystone import Ks, KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN
     
        # 处理可能出现的 loc_xxxxx：将其替换为对应的绝对地址 0xXXXXXXXX
        if "loc_" in asm:
            parts = asm.split()
            for i, token in enumerate(parts):
                if token.startswith("loc_"):
                    ea = ida_kernwin.str2ea(token)
                    if ea == ida_idaapi.BADADDR:
                        raise RuntimeError(f"Cannot resolve {token}")
                    parts[i] = f"0x{ea:X}"
            asm = " ".join(parts)
     
        # Keystone 汇编
        ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
        encoding, count = ks.asm(asm, addr)
        code = bytes(encoding)
        # 以十六进制显示编码，例如 "70 BD FA 17"
        hex_encoding = " ".join(f"{b:02X}" for b in encoding)
        print(f"[+] Encoding: {hex_encoding}")
        # Patch 到 IDA
        ida_bytes.patch_bytes(addr, code)
     
        # 强制 IDA 重新反汇编
        ea = addr
        end = addr + len(code)
        while ea < end:
            ida_ua.create_insn(ea)
            ea += 4
     
        print(f"[+] Patched {count} ARM64 instructions at {hex(addr)}")
        return count
     
    def find_empty_space(size: int) -> int:
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
        end = seg.end_ea
     
        # 从第一个 0x10 对齐的地址开始，每次步进 0x10，直接在该对齐处读 size 字节检查是否全 0
        ea = (start + 0xF) & ~0xF
        while ea + size <= end:
            data = ida_bytes.get_bytes(ea, size)
            if data is not None and all(b == 0 for b in data):
                return ea
            ea += 0x10
     
        return ida_idaapi.BADADDR
     
    match_map = {
        0x139dd0: [0x139dd8, 0x139de0],
        ......
    }
     
    trace_map = {
        "0x139dd0": {"x9": {"0xc": "0x139628", "0xf": "0x13a934", "0x13": "0x13cd74"}},
        ......
    }
     
    for pc_offset, node in trace_map.items():
        for reg, info in node.items():
            print(f"reg:{reg}")
            br_reg_id = int(reg.replace("x", ""))
            print(f"br_reg_id:{br_reg_id}")
            empty_space = find_empty_space((len(info)*2+1)*4)
            set_asm(int(pc_offset, 16), "NOP")
            set_asm(int(pc_offset, 16)+4, "NOP")
            set_asm(match_map[int(pc_offset, 16)][0], "NOP")
            set_asm(match_map[int(pc_offset, 16)][1], "B " + hex(empty_space))
            index = 0
            for key, value in info.items():
                print(f"reg:{reg} key:{key} addr:{value}")
                set_asm(empty_space + 4 * index, f"CMP W{br_reg_id}, #{key}")
                index += 1
                set_asm(empty_space + 4 * index, f"B.EQ {value}")
                index += 1
            set_asm(empty_space + 4 * index, "RET")
            print("================================================")
    
    ```
    

代码跟踪工具
------

### Trace 工具

没什么说的，直接上追佬的工具，配合葫芦佬的 frida 一键 trace

https://github.com/jiqiu2022/vm-trace-release

https://github.com/hzzheyang/strongR-frida-android

### Trace 查看工具

强推 010Editor16，秒开超大文本，支持智能高亮

### Trace 分析工具

我自己拿 AI 写了一个简单的 UI 分析工具，主要是用来快速追溯寄存器的赋值情况（010Editor 来回翻太痛苦了）

项目地址：https://github.com/lxz-jiandan/TraceAnalyzer

![](https://jiandanyun.myds.me:4567/2026/01/image-20260129202733155.png)

关键分析整理
------

### VmState 数据结构与管理机制

我们可以简单的考虑一个问题，VMP 的原理是将编码的数据转换为自己能理解的 vmstate 然后再进行执行，并且每个被 VMP 保护的函数都对应一个 vmstate，那么 vmp 会每次执行的时候都重新解码一次构造一个新的 vmstate 么？换句话说如果我们自己来设计一个虚拟机我们会怎么做？显然是利用红黑树来管理所有的 vmstate，并且本次样本也是这么干的，此处可以说英雄所见略同。

```
struct VmState
{
    FunctionType *function_list;
    __int64 register_count;
    __int64 type_count;
    void *register_list;
    void *type_list;
    void *inst_list;
    void *param_list;
};
struct VmStateTreeNode
{
      char _color_pad[8];
      struct VmStateTreeNode *parent;
      struct VmStateTreeNode *left;
      struct VmStateTreeNode *right;
      unsigned __int64 key;
      VmState *value;
};
struct VmStateTree
{
      VmStateTreeNode *root;
      union
      {
            struct
            {
                  VmStateTreeNode *sentinel_left;
                  VmStateTreeNode *sentinel_parent;
                  VmStateTreeNode *sentinel_right;
                  void *sentinel_key;
                  VmState *sentinel_value;
            };
            char sentinel_area[40];
      };
      pthread_mutex_t mutex;
};

```

### ByteCode 与 ReTable 的重定位机制

我们从设计一个虚拟机来考虑一个问题，如果将虚拟机所有的代码都放在一个 BYTECODE 数据块中是否可行，理论上来讲一定是可行的，但这会导致我们几乎需要实现完整的 linker，因为虚拟机中解释执行到 BL 这类指令时虚拟机需要自己计算重定位，这除了增加我们的开发难度以外几乎没有任何正向收益。所以简单的办法是除了 BYTECODE 数据块以外，我们还需要重构一份重定位表，让系统的 Linker 帮我们计算，而我们只需要在 ByteCode 里直接取 ReTable[id] 就行了。

### ByteCode 解析机制

解析每组 BYTECODE 时都采用了如下的结构体进行解析

```
struct ByteCodeReader
{
    void *buffer_ptr;
    __int64 buffer_size;
    __int64 cached_bits;
    unsigned int bit_count;
    __int64 read_pos;
};

```

解码按 6 bit 为一单元消费，其中最高位为继续标志位，当前 cached_bits 不足 6 bit 时触发补读，补读会使 byteCodeReader 从 buffer 再读 8 字节（不足 8 字节则读到末尾）做 bit 拼接，再继续；若仍不足则解码失败。

```
type_tag = bit_stream_read((__int64)byteCodeReader, 6u);    //1.读取6位
type_tag_1 = type_tag;                                   //2.初始化结果
if ( (type_tag & 0x20) != 0 )                             //3.检查bit5(继续标志位)
{
  type_tag_1 = type_tag & 0x1F;                           //4.提取低5位数据/5。位移量初始化为5
  v88 = 5;                                              //5.位移量初始化为5
  do   
  {
    v89 = bit_stream_read((__int64)byteCodeReader, 6u);     //6. 读下一个6位
    type_tag_1 |= (v89 & 0x1F) << v88;                       //7.拼接低5位数据
    v88 += 5;
  }
  while ( (v89 & 0x20) != 0 );                            //9.检查继续标志
}                                                            //10.type_tag_1 现在包含完整的VLE解码值

```

### 不定参数与 ABI 机制

反汇编中出现的大量类似如下的代码，本质应该还是标准 API va_arg、va_copy 在编译器优化后的展开形式，这里提供一份分析用的代码，可以自行编译调试一下，还是挺有意思的。

```
struct __va_list_tag {
    void *__stack;
    void *__gr_top;
    void *__vr_top;
    int   __gr_offs;
    int   __vr_offs;
};
 
void test_va(int a, int b, int c, int d,int e, int f, int g, int h,...) {
    va_list ap;
    va_start(ap, h);
 
    struct __va_list_tag *va = (struct __va_list_tag *)≈
    __va_list_tag* caller_context = new __va_list_tag();
    memcpy(caller_context, &ap, sizeof(__va_list_tag));
     
    for(int i = 0; i < 3; i++){
        double* stack = nullptr;
        int vr_offs = caller_context->__vr_offs;
        if ( (int)vr_offs < 0 && (caller_context->__vr_offs = vr_offs + 16, (int)vr_offs + 16 <= 0))
        {
            stack = (double *)((char *)caller_context->__vr_top + vr_offs);
        }
        else
        {
            stack = (double *)caller_context->__stack;
            caller_context->__stack = (char *)caller_context->__stack + 8;
        }
        double v213 = *stack;
        LOGD("v213:%f", v213);
 
        uint64_t stack_args_value = 0;
        int* stack_args_1 = nullptr;
        int __gr_offs = caller_context->__gr_offs;
        if ( (int)__gr_offs < 0 && (caller_context->__gr_offs = __gr_offs + 8, (int)__gr_offs + 8 <= 0) )
        {
            stack_args_value = *(int *)((char *)caller_context->__gr_top + __gr_offs);
        }
        else
        {
            stack_args_1 = (int *)caller_context->__stack;
            caller_context->__stack = (char *)caller_context->__stack + 8;
            stack_args_value = *stack_args_1;
        }
        LOGD("stack_args_value：%lu", stack_args_value);
    }
    return;
}
test_va(1,2,3,4,5,6,7,8,1.25, 2.5);
test_va(1,2,3,4,5,6,7,8,100, 1.3, 300);

```

### 符号与 RTTI 泄露信息

通过 `.data.rel.ro` 段中的 **vtable 与 RTTI 符号**，可以还原出大量关键信息

```
.data.rel.ro:00000000001F1340             ; `vtable for'jg_vmp::Type
.data.rel.ro:00000000001F1340 00 00 00 00 _ZTVN6jg_vmp4TypeE DCQ 0                ; DATA XREF: decode_type+DF0↑o
.data.rel.ro:00000000001F1340 00 00 00 00                                         ; decode_type+EDC↑o ...
.data.rel.ro:00000000001F1340                                                     ; offset to this
.data.rel.ro:00000000001F1348 78 13 1F 00…                DCQ _ZTIN6jg_vmp4TypeE  ; `typeinfo for'jg_vmp::Type
.data.rel.ro:00000000001F1350 2C ED 13 00…off_1F1350      DCQ nullsub_5
.data.rel.ro:00000000001F1358 64 DB 13 00…                DCQ j_j_.free_3
.data.rel.ro:00000000001F1360 68 DB 13 00…                DCQ sub_13DB68
.data.rel.ro:00000000001F1368 74 DB 13 00…                DCQ sub_13DB74
.data.rel.ro:00000000001F1370 38 E0 13 00…                DCQ sub_13E038

```

1.  类型系统完整暴露，可明确识别虚拟机内部的类型体系，包括：

*   `jg_vmp::Type`（基类）
*   `IntegerType / StructType / PointerType / FunctionType / ArrayType / VectorType`（派生类）

1.  继承关系可准确还原，RTTI 使用 `__si_class_type_info`，表明所有类型均统一继承自 `jg_vmp::Type` 可完整重建类型继承树
    
2.  虚函数接口与调用约定泄露，vtable 中保存了虚函数数量和顺序以及派生类对基类虚函数的覆盖关系
    
3.  VM 类型解析与执行逻辑被轻易推断，vtable / RTTI 被 `decode_type` 与 `vm_interpreter` 大量使用
    

```
enum TypeKind : unsigned __int32
{
    VoidType = 0x0,
    FloatType = 0x2,
    DoubleType = 0x3,
    IntegerType = 0xB,
    FunctionType = 0xC,
    StructType = 0xD,
    ArrayType = 0xE,
    PointerType = 0xF,
    VectorType = 0x10,
};
struct /*VFT*/ Type_vtbl
{
    void (*_destructor)();
    void (*_operator_delete)();
    void (*_clone)();
    void (*_get_type_name)();
    TypeKind (*_get_type_size)();
};
struct Type
{
    Type_vtbl *vtable;
    TypeKind  kind;
    char field1;
    char field2;
    char field3;
    char field4;
};

```

### 虚拟寄存器机制

1.  寄存器数量动态生成，解码时从 bytecode 里读出，再按这个数量分配 RegManager。同一套解释器可跑 “不同寄存器规模” 的 bytecode。
2.  堆指针标志，每个槽 24 字节，除存值 / 指针外，还有一个布尔类型的堆指针标志。若某条指令（如 ALLOC）在某个寄存器上标记了堆指针标志，则退出时由 VM 对该寄存器中的指针做释放。
3.  虚拟寄存器初始化，当前指令的参数个数大于 0 时，会从 vava（VaStruct / va_list） 里按 ABI 取参，依次写入参数个数的寄存器，还会从 bytecode 取参并设置返回值类型。
4.  缓存的不是 “执行后的状态”，当指令没有参数时，每次执行都从初始寄存器缓存中拷贝一份新的虚拟寄存器内存用于执行，当指令存在参数时则从 va（VaStruct / va_list） 里按 ABI 取参，即初始化的寄存器缓存永远不用于执行，只作为拷贝的源数据。

基于逆向的自研 VMP 项目
--------------

前面做逆向时，我们已经把样本里的关键机制摸得比较清楚了：状态缓存、字节码组织、调用桥接、参数传递、寄存器槽管理。所以我们自己实现一个虚拟化加固，但路线选择上，我没有走 LLVM IR，而是选择了汇编语义翻译。虽然实现起来更困难，但更易于理解 VMP 核心机制。

项目地址：https://github.com/lxz-jiandan/VmpProject

### 加固流程

离线阶段的核心目标是： **将原始函数逻辑转化为虚拟执行载荷，并重构目标库的调用路径，实现执行权转移。**

#### 1. 导出函数编码产物

对选定的目标函数进行指令抽取与翻译处理：

*   解析函数控制流结构（BasicBlock / CFG）
*   将原生 ARM64 指令翻译为虚拟指令序列
*   生成函数级 VM 编码载荷

每个函数最终形成：

```
[函数标识] + [虚拟指令流] + [元数据]

```

构成独立的 “函数执行单元”。

#### 2. 构建扩展载荷容器

将所有函数执行单元汇总，组织为统一的扩展容器结构：

*   函数索引表（symbolKey → payloadOffset）
*   共享分支地址映射表
*   外部调用桥接信息
*   运行时所需元数据

容器结构写入扩展载荷库尾部区域，形成独立逻辑区块。

#### 3. 嵌入运行时宿主库

将构建完成的扩展载荷整体嵌入运行时宿主库（VM Engine）尾部：

*   保持单一 so 交付形态
*   避免额外文件依赖
*   降低部署复杂度

最终形成：

```
[VM Engine 本体] + [扩展载荷容器]

```

实现单库集成模型。

#### 4. 接管目标库导出符号

对宿主库（VM Engine）进行符号层重构：

*   将目标库的符号添加至宿主库符号表中
*   更新符号表与哈希布局（这里主要参考 vmprotect-3.5.1 的实现）
*   将函数入口重定向至统一分发函数

调用路径被收口为：

```
原始导出函数 → 接管分发层 → 虚拟执行

```

### 虚拟执行流程

运行时阶段的核心目标是：**在保持外部调用行为不变的前提下，完成载荷装载、链接恢复与虚拟执行调度。**

#### 1. 读取嵌入载荷并交由自定义链接器处理

*   定位宿主库尾部嵌入的扩展载荷
*   直接在内存中解析（这里主要参考 soLoader）
*   将载荷恢复为可执行内存映像

#### 3. 预热函数缓存

对扩展容器进行解析：

*   构建函数索引缓存
*   恢复函数执行元数据
*   挂载共享分支地址表
*   建立 symbolKey → payload 映射关系

#### 5. 路由到目标虚拟函数

当导出函数被调用时：

*   通过 symbolKey + 模块编号定位目标函数
*   构造虚拟执行上下文
*   装配参数与寄存器模型

#### 6. 解释执行与桥接调用

虚拟机进入执行循环：

*   逐条解析虚拟指令
*   模拟寄存器与栈环境
*   控制流跳转通过内部映射表完成

结语
--

这份笔记既是样本分析记录，也是工程实践说明。希望它能为同样在做 VMP 逆向或虚拟化保护研究的同学提供些许帮助

参考资料
----

https://bbs.kanxue.com/thread-286441.htm

https://0xacab.org/bidasci/vmprotect-3.5.1

https://github.com/SoyBeanMilkx/soLoader

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 1 小时前 被简单的简单编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm)