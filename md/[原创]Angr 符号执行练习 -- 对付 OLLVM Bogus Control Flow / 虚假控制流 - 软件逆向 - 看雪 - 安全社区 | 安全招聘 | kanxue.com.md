> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287076.htm)

> [原创]Angr 符号执行练习 -- 对付 OLLVM Bogus Control Flow / 虚假控制流

```
创建: 2025-05-28 17:14
更新: 2025-06-03 14:31
 
目录:
 
    ☆ 背景介绍
    ☆ hello.c
    ☆ hello_bcf
    ☆ hello_bcf_patch.py
    ☆ get_cond_jmp_list的技术原理
```

☆ 背景介绍

OLLVM 支持 "Bogus Control Flow / 虚假控制流"。启用 bcf 编译源码，生成的二进制用 IDA 反汇编时，会看到许多实际执行时永不可达的基本块。冗余的不可达块对 IDA F5 带来干扰，肉眼看不出原始代码逻辑，同时不影响实际执行逻辑。bcf 目的就是对抗静态分析。

OLLVM 项目提供过一张示意图

```
原始流程
 
         entry
           |
     ______v______
    |   original  |
    |_____________|
           |
           v
         return
```

```
启用bcf之后的流程
 
         entry
           |
       ____v_____
      |condition*| (false)
      |__________|----+
     (true)|          |
           |          |
     ______v______    |
+-->|   original* |   |
|   |_____________| (true)
|   (false)|    !-----------> return
|    ______v______    |
|   |   altered   |<--!
|   |_____________|
|__________|
```

上图那些条件判断，实际恒为 true，执行时沿 true 前进，与原始逻辑无异。但静态分析时，受 false 干扰，并不知道 false 路径永不可达。

参看

```
利用angr符号执行去除虚假控制流 - 34r7hm4n [2021-02-10]
https://bbs.kanxue.com/thread-266005.htm
```

作者用「angr 符号执行」识别目标程序中不可达块，静态 Patch 目标程序，将不可达块 NOP 化。虽然 IDA 反汇编结构图仍然显乱，但 F5 比较智能，已能去干扰，显示出原始逻辑。

```
NOP化之后的流程
 
         entry
           |
       ____v_____
      |condition*| (false)
      |__________|----+
     (true)|          |
           |          |
     ______v______    |
+-->|   original* |   |
|   |_____________| (true)
|   (false)|    !-----------> return
|    ______v______    |
|   |     nop     |<--!
|   |_____________|
|__________|
```

假设 angr 符号执行能精确识别干扰性质的条件跳转指令，我选择将所有干扰性质的条件跳转指令静态 Patch 成无条件跳转指令，直接跳向相应可达块；此方案本质上等价于 NOP 化不可达块。

```
JMP化之后的流程
 
         entry
           |
       ____v_____
      |condition*|
      |__________|
      (jmp)|
           |
     ______v______
+-->|   original* |
|   |_____________| (jmp)
|               !-----------> return
|    ______ ______
|   |   altered   |
|   |_____________|
|__________|
```

本文以学习 angr 进阶用法为目的，借 bcf 反混淆为靶标，演示 JMP 化思路。

```
$ pip3 show angr | grep Version
Version: 9.2.125.dev0
```

☆ hello.c

```
#include #include static unsigned int foo ( unsigned int n )
{
    unsigned int    mod = n % 4;
    unsigned int    ret = 0;
 
    if ( mod == 0 )
    {
        ret = ( n | 0xbaaad0bf ) * ( 2 ^ n );
    }
    else if ( mod == 1 )
    {
        ret = ( n & 0xbaaad0bf ) * ( 3 + n );
    }
    else if ( mod == 2 )
    {
        ret = ( n ^ 0xbaaad0bf ) * ( 4 | n );
    }
    else
    {
        ret = ( n + 0xbaaad0bf ) * ( 5 & n );
    }
    return ret;
}
 
int main ( int argc, char * argv[] )
{
    unsigned    int n,
                    ret;
 
    if ( argc < 2 )
    {
        fprintf( stderr, "Usage: %s \n", argv[0] );
        return -1;
    }
    n   = (unsigned int)strtoul( argv[1], NULL, 0 );
    ret = foo( n );
    fprintf( stdout, "n=%#x ret=%#x\n", n, ret );
    return 0;
} 
```

```
clang -pipe -O0 -s -mllvm -passes=bcf -o hello_bcf hello.c
```

用某版 OLLVM 启用 bcf 编译，得到 hello_bcf。

完整测试用例打包

```
https://scz.617.cn/unix/202505281714.txt
https://scz.617.cn/unix/202505281714.7z
```

☆ hello_bcf

```
$ file -b hello_bcf
ELF 64-bit LSB executable, x86-64, ..., stripped
```

IDA64 反汇编 hello_bcf

```
__int64 __fastcall main(int a1, char **a2, char **a3)
{
    ...
    v24 = a1;
    v25 = a2;
    //
    // unk_4040??位于.bss，实际运行时初始化为0。下列布尔值恒false
    //
    if ( ((unk_404098 * (unk_404098 + 1)) & 1) != 0 && unk_4040F8 >= 10 )
        //
        // 此跳转永不发生
        //
        goto LABEL_11;
    while ( 1 )
    {
        v3 = v25;
        v19 = (unsigned int *)(&v17 - 2);
        v20 = (const char ***)(&v17 - 2);
        v21 = (unsigned int *)(&v17 - 2);
        v22 = (unsigned int *)(&v17 - 2);
        *((_DWORD *)&v17 - 4) = 0;
        v17 = v3;
        v23 = (int)v3 < 2;
        //
        // 下列布尔值恒true
        //
        if ( ((unk_404120 * (unk_404120 + 1)) & 1) == 0 || unk_4040C4 < 10 )
            //
            // 此break必发生
            //
            break;
LABEL_11:
        //
        // 永可不达的基本块
        //
        LODWORD(v17) = v24;
        *(&v17 - 2) = v25;
    }
    if ( v23 )
    {
        //
        // 下列布尔值恒false
        //
        if ( ((unk_40411C * (unk_40411C + 1)) & 1) != 0 && unk_4040C0 >= 10 )
            //
            // 此跳转永不发生
            //
            goto LABEL_12;
        while ( 1 )
        {
            fprintf(stderr, "Usage: %s \n", **v20);
            *v19 = -1;
            //
            // 下列布尔值恒true
            //
            if ( ((unk_404110 * (unk_404110 + 1)) & 1) == 0 || unk_4040B8 < 10 )
                //
                // 此break必发生
                //
                break;
LABEL_12:
            //
            // 永可不达的基本块
            //
            fprintf(stderr, "Usage: %s \n", **v20);
            *v19 = -1;
        }
    }
    else
    {
        ...
    }
    do
        v18 = *v19;
    //
    // 下列布尔值恒false
    //
    while ( ((unk_4040E8 * (unk_4040E8 + 1)) & 1) != 0 && unk_404138 >= 10 );
    return v18;
} 
```

F5 的伪代码没必要深究，看个大概即可。

☆ hello_bcf_patch.py

这是对付 hello_bcf 的完整代码，演示性质，非通用实现。

```
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
 
import sys, logging
import angr
 
def get_func_from_addr ( proj, addr ) :
    try :
        return proj.kb.functions.get_by_addr( addr )
    except KeyError :
        return proj.kb.functions.floor_func( addr )
 
class PrivateHelper ( angr.ExplorationTechnique ) :
 
    def __init__ ( self, cond_jmp_set, hooksub, hooksubset ) :
        super().__init__()
        self.cond_jmp_set   = cond_jmp_set
        self.hooksub        = hooksub
        self.hooksubset     = hooksubset
 
    def successors ( self, sm, state, **kwargs ) :
        if self.hooksub and 0 != state.callstack.ret_addr :
            proj    = state.project
            target  = state.addr
            if target not in self.hooksubset :
                proj.hook( target, angr.SIM_PROCEDURES['stubs']['ReturnUnconstrained'](), replace=True )
                self.hooksubset.add( target )
        ret = sm.successors( state, **kwargs )
        if not self.hooksub or 0 == state.callstack.ret_addr :
            i   = len( ret.all_successors )
            j   = len( ret.successors )
            k   = len( ret.unsat_successors )
            if i == 2 and j == 1 and k == 1 :
                j_from  = ret.successors[0].history.jump_source
                j_to    = ret.successors[0].addr
                self.cond_jmp_set.add( ( j_from, j_to ) )
        return ret
 
def get_cond_jmp_list ( proj, func, hooksub, hooksubset ) :
    cond_jmp_set    = set()
    init_state      = proj.factory.blank_state(
        addr        = func.addr,
        add_options = {
            angr.options.SYMBOL_FILL_UNCONSTRAINED_MEMORY,
            angr.options.SYMBOL_FILL_UNCONSTRAINED_REGISTERS,
            angr.options.BYPASS_UNSUPPORTED_SYSCALL,
        }
    )
    sm              = proj.factory.simulation_manager( init_state )
    sm.use_technique( angr.exploration_techniques.DFS() )
    sm.use_technique( PrivateHelper( cond_jmp_set, hooksub=hooksub, hooksubset=hooksubset ) )
    sm.run()
    return sorted( cond_jmp_set )
 
def patch_buf ( proj, buf, cond_jmp_list ) :
    #
    # 本例简单处理，假设都能用EB短跳转
    #
    for j_from, j_to in cond_jmp_list :
        j_off   = j_to - ( j_from + 2 )
        assert j_off <= 0x7f and j_off >= -0x80
        off     = proj.loader.main_object.addr_to_offset( j_from )
        buf[off:off+2] \
                = bytes( [0xeb, j_off] )
 
def dosth ( proj, buf, addr, hooksub, hooksubset ) :
    func            = get_func_from_addr( proj, addr )
    cond_jmp_list   = get_cond_jmp_list( proj, func, hooksub, hooksubset )
    print( f'cond_jmp_list[{len(cond_jmp_list)}]' )
    for j in cond_jmp_list :
        print( f'cond_jmp {j[0]:#x} -> {j[1]:#x}' )
    patch_buf( proj, buf, cond_jmp_list )
 
def main ( argv ) :
 
    logging.getLogger( 'angr.engines.successors' ).setLevel( logging.ERROR )
    logging.getLogger( 'angr.analyses.cfg.cfg_base' ).setLevel( logging.ERROR )
 
    base_addr   = 0x400000
    proj        = angr.Project(
        argv[1],
        load_options    = {
            'auto_load_libs'    : False,
            'main_opts'         : {
                'base_addr' : base_addr
            }
        }
    )
    cfg         = proj.analyses.CFG(
        force_smart_scan    = False,
        force_complete_scan = True,
        normalize           = True,
        resolve_indirect_jumps \
                            = True,
        fail_fast           = True
    )
 
    with open( argv[1], 'rb' ) as f :
        buf = bytearray( f.read() )
 
    hooksub     = int( argv[3], 0 )
    #
    # 对应若干需要Patch的函数
    #
    if hooksub :
        addrlist    = ( 0x401140, 0x4014e0, )
        hooksubset  = set()
    else :
        #
        # 不单独指定0x4014e0，它是0x401140的子函数
        #
        addrlist    = ( 0x401140, )
        hooksubset  = None
    for addr in addrlist :
        if hooksub and addr in hooksubset :
            proj.unhook( addr )
            hooksubset.discard( addr )
        dosth( proj, buf, addr, hooksub, hooksubset )
 
    with open( argv[2], 'wb' ) as f :
        f.write( buf )
 
if "__main__" == __name__ :
    main( sys.argv )
```

```
python3 hello_bcf_patch.py hello_bcf hello_bcf_new_0 0
python3 hello_bcf_patch.py hello_bcf hello_bcf_new_1 1
```

IDA64 反汇编 hello_bcf_new_*，F5 查看 main、sub_4014E0，已能看出 hello.c 所展示的代码逻辑。

☆ get_cond_jmp_list 的技术原理

hello_bcf_patch.py 的核心是获取那些永远只走同一条分支的条件跳转指令，与下列代码强相关

```
sm.use_technique( PrivateHelper( cond_jmp_set, hooksub=hooksub, hooksubset=hooksubset ) )
```

进一步说，核心代码是

```
class PrivateHelper ( angr.ExplorationTechnique ) :
 
    #
    # 重载successors()
    #
    def successors ( self, sm, state, **kwargs ) :
        #
        # 若命令行参数要求hook子函数，检查调用栈回溯，发现当前state位于子
        # 函数入口时，hook子函数。
        #
        if self.hooksub and 0 != state.callstack.ret_addr :
            proj    = state.project
            target  = state.addr
            if target not in self.hooksubset :
                proj.hook( target, angr.SIM_PROCEDURES['stubs']['ReturnUnconstrained'](), replace=True )
                self.hooksubset.add( target )
        #
        # 获取当前状态的后继状态
        #
        ret = sm.successors( state, **kwargs )
        #
        # 若命令行参数不要求hook子函数，记录以当前函数为根的树中的条件跳转
        # 指令，否则只记录当前函数中的条件跳转指令，不记录子函数中的。
        #
        if not self.hooksub or 0 == state.callstack.ret_addr :
            i   = len( ret.all_successors )
            j   = len( ret.successors )
            k   = len( ret.unsat_successors )
            #
            # 发现永远只走同一条分支的条件跳转指令
            #
            if i == 2 and j == 1 and k == 1 :
                j_from  = ret.successors[0].history.jump_source
                j_to    = ret.successors[0].addr
                self.cond_jmp_set.add( ( j_from, j_to ) )
        return ret
```

是否 hook 子函数，要看子函数的返回值对父函数流程产生何种影响。单就 hello_bcf 而言，是否 hook 子函数，不影响最终结果，hello_bcf_new_* 是一样的。

假设需要 hook 子函数，在 successors() 中完成，不必遍历 "active stash"，不必动用 block.capstone.insns。

发现永远只走同一条分支的条件跳转指令后，记录 (from,to)；将来对 from 处的条件跳转指令(比如 jnz) 进行 Patch，改成 jmp，演示时假设均可用 "eb xx" 短跳转。

重载 successors() 的方案，带有巨大的 Hacking 性，只用 hello_bcf 测试过，仅为 PoC，非健壮通用实现。不论目标样本如何变，本例展示的 angr 技术始终派得上用场，只是反混淆逻辑要 case by case。

建议动态调试，加强理解 angr 流程。

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#调试逆向](forum-4-1-1.htm)

上传的附件：

*   [202505281714.7z](javascript:void(0);) （4.20kb，4 次下载）