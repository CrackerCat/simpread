> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268598.htm)

> [原创]OLLVM Deflattener

文章首发于安全客——[链接](https://www.anquanke.com/post/id/248154)

0x00 前言
-------

笔者于五月份时遇到几个经控制流平坦化的样本，由于之前没有接触过这方面知识，未深入分析。

 

![](https://bbs.pediy.com/upload/attach/202107/817966_8FVQAXZUH93PD2T.jpg)

 

七月初看到一篇博客——[MODeflattener - Miasm's OLLVM Deflattener](https://mrt4ntr4.github.io/MODeflattener/)，作者同时在 Github 上公开其源码，故笔者决定对其一探究竟。该工具使用`miasm`框架，Quarklab 于 2014 年的博客——[Deobfuscation: recovering an OLLVM-protected program](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html) 中已提到此框架，但其采用符号执行以遍历代码，计算每个 Relevant Block 目标终点，恢复执行流程。腾讯 SRC 于 2017 年的博客——[利用符号执行去除控制流平坦化](https://security.tencent.com/index.php/blog/msg/112)中亦是采用符号执行 (基于 angr 框架)，而 MODeflattener 采用静态分析方式来恢复执行流程。笔者于下文首先结合具体样本分析其源码及各函数功能，之后指出该工具不足之处并提出改进方案。关于涉及到的`miasm`框架中类，函数，属性及方法可参阅 [miasm Documentation](https://miasm.re/miasm_doxygen/) 与 [miasm 源码](https://github.com/cea-sec/miasm)。LLVM，OLLVM，LLVM Pass 及控制流平坦化不再赘述，具体可参阅[基于 LLVM Pass 实现控制流平坦化](https://bbs.pediy.com/thread-266082.htm)与 [OLLVM Flattening 源码](https://github.com/obfuscator-llvm/obfuscator/blob/llvm-4.0/lib/Transforms/Obfuscation/Flattening.cpp)。

0x01 源码分析
---------

#### 0x01.1 检测平坦化

```
def calc_flattening_score(asm_graph):
    score = 0.0
    for head in asm_graph.heads_iter():
        dominator_tree = asm_graph.compute_dominator_tree(head)
        for block in asm_graph.blocks:
            block_key = asm_graph.loc_db.get_offset_location(block.lines[0].offset)
            dominated = set(
                [block_key] + [b for b in dominator_tree.walk_depth_first_forward(block_key)])
            if not any([b in dominated for b in asm_graph.predecessors(block_key)]):
                continue
            score = max(score, len(dominated)/len(asm_graph.nodes()))
    return score

```

计算每个块支配块数量，除以图中所有块，取最高分——若不低于 0.9，证明该函数经过控制流平坦化。如下图：

 

![](https://bbs.pediy.com/upload/attach/202107/817966_ANVVJ34N2MWWP3H.jpg)

 

块 b 支配除块 a 外其余所有块 (包含自身)，那么其得分为`9/10=0.9`即最高分。读者可自行查阅 Dominator Tree 及 Predecessors 相关数据结构知识，此外不作展开。

#### 0x01.2 获取平坦化相关信息

![](https://bbs.pediy.com/upload/attach/202107/817966_M9VGFWFKSC8HN5Q.jpg)

```
def get_cff_info(asmcfg):
    preds = {}
    for blk in asmcfg.blocks:
        offset = asmcfg.loc_db.get_location_offset(blk.loc_key)
        preds[offset] = asmcfg.predecessors(blk.loc_key)
    # pre-dispatcher is the one with max predecessors
    pre_dispatcher = sorted(preds, key=lambda key: len(preds[key]), reverse=True)[0]
    # dispatcher is the one which suceeds pre-dispatcher
    dispatcher = asmcfg.successors(asmcfg.loc_db.get_offset_location(pre_dispatcher))[0]
    dispatcher = asmcfg.loc_db.get_location_offset(dispatcher)
 
    # relevant blocks are those which preceed the pre-dispatcher
    relevant_blocks = []
    for loc in preds[pre_dispatcher]:
        offset = asmcfg.loc_db.get_location_offset(loc)
        relevant_blocks.append(get_block_father(asmcfg, offset))
 
    return relevant_blocks, dispatcher, pre_dispatcher

```

首先拥有最大数量 Predecessor 者即为`Pre-Dispatcher`，而`Pre-Dispatcher`后继为`Dispatcher`，`Pre-Dispatcher`所有前趋均为`Relevant Block`。

```
dispatcher_blk = main_asmcfg.getby_offset(dispatcher)
dispatcher_first_instr = dispatcher_blk.lines[0]
state_var = dispatcher_first_instr.get_args_expr()[1]

```

`Dispatcher`块第一条指令为状态变量：

 

![](https://z3.ax1x.com/2021/07/23/WyrPwn.png)

#### 0x01.3 去平坦化

```
for addr in relevant_blocks:
        _log.debug("Getting info for relevant block @ %#x"%addr)
        loc_db = LocationDB()
        mdis = machine.dis_engine(cont.bin_stream, loc_db=loc_db)
        mdis.dis_block_callback = stop_on_jmp
        asmcfg = mdis.dis_multiblock(addr)
 
        lifter = machine.lifter_model_call(loc_db)
        ircfg = lifter.new_ircfg_from_asmcfg(asmcfg)
        ircfg_simplifier = IRCFGSimplifierCommon(lifter)
        ircfg_simplifier.simplify(ircfg, addr)

```

遍历每个`Relevant Block`，为其创建 ASM CFG 及 IR CFG，作者实现了一`save_cfg`函数可将 CFG 输出 (需要安装 Graphviz)：

```
def save_cfg(cfg, name):
    import subprocess
    open(name, 'w').write(cfg.dot())
    subprocess.call(["dot", "-Tpng", name, "-o", name.split('.')[0]+'.png'])
    subprocess.call(["rm", name])

```

之后查找所有使用状态变量及定义状态变量语句：

```
nop_addrs = find_state_var_usedefs(ircfg, state_var)
...
def find_state_var_usedefs(ircfg, search_var):
    var_addrs = set()
    reachings = ReachingDefinitions(ircfg)
    digraph = DiGraphDefUse(reachings)
    # the state var always a leaf
    for leaf in digraph.leaves():
        if leaf.var == search_var:
            for x in (digraph.reachable_parents(leaf)):
                var_addrs.add(ircfg.get_block(x.label)[x.index].instr.offset)
    return var_addrs

```

![](https://z3.ax1x.com/2021/07/23/WyrCes.png)

 

可以看到 DefUse 图与语句对应关系：

 

![](https://bbs.pediy.com/upload/attach/202107/817966_TFJ6U4BDV75YXRN.jpg)

 

查找状态变量所有可能值：

```
def find_var_asg(ircfg, var):
    val_list = []
    res = {}
    for lbl, irblock in viewitems(ircfg.blocks):
        for assignblk in irblock:
            result = set(assignblk).intersection(var)
            if not result:
                continue
            else:
                dst, src = assignblk.items()[0]
                if isinstance(src, ExprInt):
                    res['next'] = int(src)
                    val_list += [int(src)]
                elif isinstance(src, ExprSlice):
                    phi_vals = get_phi_vars(ircfg)
                    res['true_next'] = phi_vals[0]
                    res['false_next'] = phi_vals[1]
                    val_list += phi_vals
    return res, val_list

```

为状态变量赋值有两种情况——一种是直接赋值，不涉及条件判断及分支跳转，其对应`ExprInt`：

 

![](https://z3.ax1x.com/2021/07/23/WyrEWT.png)

 

![](https://z3.ax1x.com/2021/07/23/WyrelF.png)

 

另一种是`CMOV`条件赋值，其对应`ExprSlice`：

 

![](https://bbs.pediy.com/upload/attach/202107/817966_VCTKHMRRV329FQB.jpg)

 

![](https://bbs.pediy.com/upload/attach/202107/817966_HW3B86VXQEEJ272.jpg)

 

之后计算每一状态变量可能值对应`Relevant Block`：

```
# get the value for reaching a particular relevant block
for lbl, irblock in viewitems(main_ircfg.blocks):
    for assignblk in irblock:
        asg_items = assignblk.items()
        if asg_items:    # do not enter if nop
            dst, src = asg_items[0]
            if isinstance(src, ExprOp):
                if src.op == 'FLAG_EQ_CMP':
                    arg = src.args[1]
                    if isinstance(arg, ExprInt):
                        if int(arg) in val_list:
                            cmp_val = int(arg)
                            var, locs = irblock[-1].items()[0]
                            true_dst = main_ircfg.loc_db.get_location_offset(locs.src1.loc_key)
                            backbone[hex(cmp_val)] = hex(true_dst)

```

如下图：

 

![](https://bbs.pediy.com/upload/attach/202107/817966_ZVY94ZWCA4QVBAD.jpg)

 

将`fixed_cfg`中存储的状态变量值替换为目标`Relevant Block`地址：

```
for offset, link in fixed_cfg.items():
        if 'cond' in link:
            tval = fixed_cfg[offset]['true_next']
            fval = fixed_cfg[offset]['false_next']
            fixed_cfg[offset]['true_next'] = backbone[tval]
            fixed_cfg[offset]['false_next'] = backbone[fval]
        elif 'next' in link:
            fixed_cfg[offset]['next'] = backbone[link['next']]
        else:
            # the tail doesn't has any condition
            tail = int(offset, 16)

```

将所有无关指令替换为 NOP：

```
for addr in rel_blk_info.keys():
        _log.info('=> cleaning relevant block @ %#x' % addr)
        asmcfg, nop_addrs = rel_blk_info[addr]
        link = fixed_cfg[hex(addr)]
        instrs = [instr for blk in asmcfg.blocks for instr in blk.lines]
        last_instr = instrs[-1]
        end_addr = last_instr.offset + last_instr.l
        # calculate original length of block before patching
        orig_len = end_addr - addr
        # nop the jmp to pre-dispatcher
        nop_addrs.add(last_instr.offset)
        _log.debug('nop_addrs: ' + ', '.join([hex(addr) for addr in nop_addrs]))
        patch = patch_gen(instrs, asmcfg.loc_db, nop_addrs, link)
        patch = patch.ljust(orig_len, b"\x90")
        patches[addr] = patch
        _log.debug('patch generated %s\n' % encode_hex(patch))
 
    _log.info(">>> NOPing Backbone (%#x - %#x) <<<" % (backbone_start, backbone_end))
    nop_len = backbone_end - backbone_start
    patches[backbone_start] = b"\x90" * nop_len

```

其中`patch_gen`函数定义如下：

```
def patch_gen(instrs, loc_db, nop_addrs, link):
    final_patch = b""
    start_addr = instrs[0].offset
 
    for instr in instrs:
        #omitting useless instructions
        if instr.offset not in nop_addrs:
            if instr.is_subcall():
                #generate asm for fixed calls with relative addrs
                patch_addr = start_addr + len(final_patch)
                tgt = loc_db.get_location_offset(instr.args[0].loc_key)
                _log.info("CALL %#x" % tgt)
                call_patch_str = "CALL %s" % rel(tgt, patch_addr)
                _log.debug("call patch : %s" % call_patch_str)
                call_patch = asmb(call_patch_str, loc_db)
                final_patch += call_patch
                _log.debug("call patch asmb : %s" % encode_hex(call_patch))
            else:
                #add the original bytes
                final_patch += instr.b
 
    patch_addr = start_addr + len(final_patch)
    _log.debug("jmps patch_addr : %#x", patch_addr)
    jmp_patches = b""
    # cleaning the control flow by patching with real jmps locs
    if 'cond' in link:
        t_addr = int(link['true_next'], 16)
        f_addr = int(link['false_next'], 16)
        jcc = link['cond'].replace('CMOV', 'J')
        _log.info("%s %#x" % (jcc, t_addr))
        _log.info("JMP %#x" % f_addr)
 
        patch1_str = "%s %s" % (jcc, rel(t_addr, patch_addr))
        jmp_patches += asmb(patch1_str, loc_db)
        patch_addr += len(jmp_patches)
 
        patch2_str = "JMP %s" % (rel(f_addr, patch_addr))
        jmp_patches += asmb(patch2_str, loc_db)
        _log.debug("jmp patches : %s; %s" % (patch1_str, patch2_str))
 
    else:
        n_addr = int(link['next'], 16)
        _log.info("JMP %#x" % n_addr)
 
        patch_str = "JMP %s" % rel(n_addr, patch_addr)
        jmp_patches = asmb(patch_str, loc_db)
 
        _log.debug("jmp patches : %s" % patch_str)
 
    _log.debug("jmp patches asmb : %s" % encode_hex(jmp_patches))
    final_patch += jmp_patches
 
    return final_patch

```

首先检查每个块中是否存在函数调用，如果存在，计算其修补后偏移；否则直接使用原指令：

```
   if instr.is_subcall():
    #generate asm for fixed calls with relative addrs
    patch_addr = start_addr + len(final_patch)
    tgt = loc_db.get_location_offset(instr.args[0].loc_key)
    _log.info("CALL %#x" % tgt)
    call_patch_str = "CALL %s" % rel(tgt, patch_addr)
    _log.debug("call patch : %s" % call_patch_str)
    call_patch = asmb(call_patch_str, loc_db)
    final_patch += call_patch
    _log.debug("call patch asmb : %s" % encode_hex(call_patch))
else:
    #add the original bytes
    final_patch += instr.b

```

之后修复执行流程：

```
# cleaning the control flow by patching with real jmps locs
if 'cond' in link:
    t_addr = int(link['true_next'], 16)
    f_addr = int(link['false_next'], 16)
    jcc = link['cond'].replace('CMOV', 'J')
    _log.info("%s %#x" % (jcc, t_addr))
    _log.info("JMP %#x" % f_addr)
 
    patch1_str = "%s %s" % (jcc, rel(t_addr, patch_addr))
    jmp_patches += asmb(patch1_str, loc_db)
    patch_addr += len(jmp_patches)
 
    patch2_str = "JMP %s" % (rel(f_addr, patch_addr))
    jmp_patches += asmb(patch2_str, loc_db)
    _log.debug("jmp patches : %s; %s" % (patch1_str, patch2_str))
 
else:
    n_addr = int(link['next'], 16)
    _log.info("JMP %#x" % n_addr)
 
    patch_str = "JMP %s" % rel(n_addr, patch_addr)
    jmp_patches = asmb(patch_str, loc_db)

```

如果是条件赋值，替换为条件跳转；否则直接替换为 JMP 指令。

 

至此，源码分析结束。如下图所示，可以看到其效果很不错：

 

![](https://bbs.pediy.com/upload/attach/202107/817966_43TA6HT5GKE6V9J.jpg)

0x02 不足及改进
----------

#### 0x02.1 不支持 PE 文件

其计算基址是通过访问`sh`属性：

```
section_ep = cont.bin_stream.bin.virt.parent.getsectionbyvad(cont.entry_point).sh
bin_base_addr = section_ep.addr - section_ep.offset
_log.info('bin_base_addr: %#x' % bin_base_addr)

```

![](https://z3.ax1x.com/2021/07/23/WyrQT1.png)

 

故笔者首先增加一参数`baseaddr`：

```
parser.add_argument('-b',"--baseaddr", help="file base address")

```

之后判断文件类型：

```
try:
    if args.baseaddr:
        _log.info('Base Address:'+args.baseaddr)
        baseaddr=int(args.baseaddr,16)
    elif cont.executable.isPE():
        baseaddr=0x400C00
        _log.info('Base Address:%x'%baseaddr)
except AttributeError:
    section_ep = cont.bin_stream.bin.virt.parent.getsectionbyvad(cont.entry_point).sh
    baseaddr = section_ep.addr - section_ep.offset
    _log.info('Base Address:%x'%baseaddr)

```

如果是 PE 文件且未提供`baseaddr`参数，则直接给其赋值为 0x400C00；若是 ELF 文件，则访问`sh`属性。

#### 0x02.2 获取 CMOV 指令

```
elif len(var_asg) > 1:
            #extracting the condition from the last 3rd line
            cond_mnem = last_blk.lines[-3].name
            _log.debug('cond used: %s' % cond_mnem)
            var_asg['cond'] = cond_mnem
            var_asg['true_next'] = hex(var_asg['true_next'])
            var_asg['false_next'] = hex(var_asg['false_next'])
            # map the conditions and possible values dictionary to the cfg info
            fixed_cfg[hex(addr)] = var_asg

```

通常 CMOV 指令位于倒数第三条语句，但笔者在测试时发现部分块 CMOV 指令位于倒数第四条语句：

 

![](https://bbs.pediy.com/upload/attach/202107/817966_EQ3RY4SE8MG8QPP.jpg)

 

故改进如下：

```
elif len(var_asg) > 1:
            #extracting the condition from the last 3rd line
            cond_mnem = last_blk.lines[-3].name
            _log.debug('cond used: %s' % cond_mnem)
            if cond_mnem=='MOV':
                cond_mnem = last_blk.lines[-4].name
            var_asg['cond'] = cond_mnem
            var_asg['true_next'] = hex(var_asg['true_next'])
            var_asg['false_next'] = hex(var_asg['false_next'])
            # map the conditions and possible values dictionary to the cfg info
            fixed_cfg[hex(addr)] = var_asg

```

#### 0x02.3 调用导入函数

```
if instr.is_subcall():
                #generate asm for fixed calls with relative addrs
                patch_addr = start_addr + len(final_patch)
                tgt = loc_db.get_location_offset(instr.args[0].loc_key)
                _log.info("CALL %#x" % tgt)
                call_patch_str = "CALL %s" % rel(tgt, patch_addr)
                _log.debug("call patch : %s" % call_patch_str)
                call_patch = asmb(call_patch_str, loc_db)
                final_patch += call_patch
                _log.debug("call patch asmb : %s" % encode_hex(call_patch))

```

仅判断是否为函数调用，但并未判断调用函数地址是否为导入函数地址，如下图两种情形所示：

 

![](https://bbs.pediy.com/upload/attach/202107/817966_NPSRRJG747AYHN5.jpg)

 

![](https://bbs.pediy.com/upload/attach/202107/817966_ZTWDJ8EEMEYBKRT.jpg)

 

改进如下：

```
if instr.is_subcall():
                if isinstance(instr.args[0],ExprMem) or isinstance(instr.args[0],ExprId):
                    final_patch += instr.b
                else:
                    #generate asm for fixed calls with relative addrs
                    patch_addr = start_addr + len(final_patch)
                    tgt = loc_db.get_location_offset(instr.args[0].loc_key)
                    _log.info("CALL %#x" % tgt)
                    call_patch_str = "CALL %s" % rel(tgt, patch_addr)
                    _log.debug("call patch : %s" % call_patch_str)
                    call_patch = asmb(call_patch_str, loc_db)
                    final_patch += call_patch
                    _log.debug("call patch asmb : %s" % encode_hex(call_patch))

```

#### 0x02.4 get_phi_vars

```
def get_phi_vars(ircfg):
    res = []
    blks = list(ircfg.blocks)
    irblock = (ircfg.blocks[blks[-1]])
 
    if irblock_has_phi(irblock):
        for dst, sources in viewitems(irblock[0]):
            phi_vars = sources.args
            parent_blks = get_phi_sources_parent_block(
                ircfg,
                irblock.loc_key,
                phi_vars
            )
 
    for var, loc in parent_blks.items():
        irblock = ircfg.get_block(list(loc)[0])
        for asg in irblock:
            dst, src = asg.items()[0]
            if dst == var:
                res += [int(src)]
 
    return res

```

未考虑到由局部变量存储状态变量值情形：

```
mov     eax, 0B3584423h
mov     ecx, 0CAE581F5h
...
test    cl, 1
mov     r9d, [rbp+18Ch]
mov     r14d, [rbp+17Ch]
cmovnz  r9d, r14d
mov     [rbp+230h], r9d
jmp     loc_4044AE

```

如果是上述情形，状态变量值往往位于头节点中，故改进如下：

```
def get_phi_vars(ircfg,loc_db,mdis):
    res = []
    blks = list(ircfg.blocks)
    irblock = (ircfg.blocks[blks[-1]])
 
    if irblock_has_phi(irblock):
        for dst, sources in viewitems(irblock[0]):
            phi_vars = sources.args
            parent_blks = get_phi_sources_parent_block(
                ircfg,
                irblock.loc_key,
                phi_vars
            )
 
    for var, loc in parent_blks.items():
        irblock = ircfg.get_block(list(loc)[0])
        for asg in irblock:
            dst, src = asg.items()[0]
            if dst == var and isinstance(src,ExprInt):
                res += [int(src)]
            elif dst==var and isinstance(src,ExprOp):
                reachings = ReachingDefinitions(ircfg)
                digraph = DiGraphDefUse(reachings)
                # the state var always a leaf
                for head in digraph.heads():
                    if head.var == src.args[0]:
                        head_offset=loc_db.get_location_offset(head.label)
                        if isinstance(mdis.dis_instr(head_offset).args[1],ExprInt):
                            res+=[int(mdis.dis_instr(head_offset).args[1])]
 
    return res

```

0x03 参阅链接
---------

*   [MODeflattener - Miasm's OLLVM Deflattener](https://mrt4ntr4.github.io/MODeflattener/)
*   [Deobfuscation: recovering an OLLVM-protected program—Quasklab](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html)
*   [利用符号执行去除控制流平坦化—腾讯 SRC](https://security.tencent.com/index.php/blog/msg/112)
*   [Automated Detection of Control-flow Flattening](https://synthesis.to/2021/03/03/flattening_detection.html)
*   [miasm—Github](https://github.com/cea-sec/miasm)
*   [OLLVM Flattening—Github](https://github.com/obfuscator-llvm/obfuscator/blob/llvm-4.0/lib/Transforms/Obfuscation/Flattening.cpp)
*   [基于 LLVM Pass 实现控制流平坦化—看雪](https://bbs.pediy.com/thread-266082.htm)

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#其他内容](forum-4-1-10.htm)