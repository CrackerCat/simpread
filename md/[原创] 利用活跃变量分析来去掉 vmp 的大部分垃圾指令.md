> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265950.htm)

环境和工具
=====

Windows7 7601  
Ida7.5  
Python3.8  
qiling 框架

简介
==

首先简单介绍一下数据流分析和活跃变量分析。活跃变量分析属于数据流分析的一种，编译器的许多优化都依赖于数据流分析。龙书的简介截图如下  
![](https://bbs.pediy.com/upload/attach/202102/742617_VCS2SQUH3A663ZC.png)  
![](https://bbs.pediy.com/upload/attach/202102/742617_FJF7N8RHT3EFW2X.png)  
活跃变量分析的用途有删除无用赋值和为基本块分配寄存器。vmp 中的垃圾指令大部分都是些无用赋值，我们可以利用活跃变量分析来删除这些垃圾指令。  
![](https://bbs.pediy.com/upload/attach/202102/742617_D5QDMUVA6W2FCKN.png)  
如上图所示。如果把 test 指令看作是对 eflags 寄存器的赋值，其中的 test 指令就属于无用赋值，因为 eflags 寄存器在当前指令被赋值之后没有被使用，在下一条指令又被重新赋值。0064D721 处对 ebp 的赋值在 0064D748 之前也没有被使用，也是一个无用赋值。

记录基本块
=====

此次活跃变量分析仅局限于合并直接跳转后的基本块内，不做全局的数据流分析，可以不用添加前驱和后继，相当于做局部优化。本次使用的样本是 vmp3.5 版本加的壳，只点了虚拟化，没有做其它的处理，这个样本的源码在 vmprotect3.5 安装目录下 Example\Code Markers\MSVC。以去除 vmp1 段中的垃圾代码为例，也就是加壳后程序入口点处的那部分虚拟机代码。由于 vmp3.5 展开了 dispatch 结构，并且利用间接跳转干扰静态分析工具的控制流重建。比如 vmp 使用大量的 jmp register 和 push register; ret; 等指令。为了构建控制流，我使用 qiling 框架来模拟执行来记录这类指令的跳转目标。qiling 实现了一个 pe 加载器且模拟了部分系统 api。经过验证，是可以运行到原程序的入口点。有两个 api 需要自己添加模拟，GetProcessAffinityMask 和 SetThreadAffinityMask。模拟的代码如下：

```
@winsdkapi(cc=STDCALL, dll, replace_params={
    "lpProcessAffinityMask":POINTER,
    "lpSystemAffinityMask":POINTER
    })
def hook_GetProcessAffinityMask(ql, address, params):
    lpProcessAffinityMask = params["lpProcessAffinityMask"]
    lpSystemAffinityMask = params["lpSystemAffinityMask"]
 
    if(ql.mem.is_mapped(lpProcessAffinityMask, 4)):
        ql.mem.write(lpProcessAffinityMask,ql.pack32(1))
    else:
        print("GetProcessAffinityMask->lpProcessAffinityMask(0x%08x) unmapped!" % lpProcessAffinityMask)
        addr = ql.os.heap.alloc(4)
        ql.mem.write(addr,ql.pack32(1))
 
    if(ql.mem.is_mapped(lpSystemAffinityMask, 4)):
        ql.mem.write(lpSystemAffinityMask,ql.pack32(1))
    else:
        print("GetProcessAffinityMask->lpSystemAffinityMask(0x%08x) unmapped!" % lpSystemAffinityMask)
        addr = ql.os.heap.alloc(4)
        ql.mem.write(addr,ql.pack32(1))
 
    return 1
 
@winsdkapi(cc=STDCALL, dll, replace_params={
    "dwThreadAffinityMask":POINTER,
    })
def hook_SetThreadAffinityMask(ql, address, params):
    hThread = params['hThread']
    pdwThreadAffinityMask = params["dwThreadAffinityMask"]
 
    if(ql.mem.is_mapped(pdwThreadAffinityMask, 4)):
        mask = ql.unpack32(ql.mem.read(pdwThreadAffinityMask, 4))
        print("SetThreadAffinityMask(0x%08x, 0x%08x)" % (hThread,mask))
 
    return 1

```

然后利用 qiling 的 hook_code 的回调函数来跟踪指令的走向，记录所需的跳转信息。回调函数的部分代码如下：

```
def traceCode(ql, address, size, user_data=None):
    #print("trace address:0x%08x" % address)
 
    #trace = [address,size]
    #if(trace not in g_traceAddrList):
        #g_traceAddrList.append(trace)
 
    if(0x004012F5 == address): #真正的入口点
        print("execute to original entrypoint! address:0x%08x" % 0x004012F5)
        ql.emu_stop()
 
    #push reg;ret
    if(1 == size and ql.mem.read(address,size)[0] == 0xc3):
        target = ql.unpack32(ql.mem.read(ql.reg.esp, 4))
 
        if(None != g_RetAddrDict.get(address)):
            g_RetAddrDict[address].add(target)
        else:
            g_RetAddrDict[address] = {target}
 
    md = ql.disassember
    md.detail = True
 
    bInsn = ql.mem.read(address,size)
    insn = list(md.disasm(bInsn, address))[0]       
 
    #trace jmp Reg
    if(capstone.x86_const.X86_INS_JMP == insn.id and capstone.x86_const.X86_OP_REG == insn.operands[0].type):
        target = ql.reg.read(insn.operands[0].reg)
 
        if(None != g_jmpRegAddrDict.get(address)):
            g_jmpRegAddrDict[address].add(target)          
        else:
            g_jmpRegAddrDict[address] = {target}

```

由于 qiling 框架模拟执行的有点慢，所以模拟到入口点结束后就把获取到的信息通过 json 序列化保存到了文件中。代码和文件我都会上传，这里就不用一一展开了。获取到信息后，就要记录所有的基本块。大体思路是以入口点的代码为一个作为一个新的基本块的开始，然后不断的把后续指令加进去，直到碰到一个无条件跳转、条件跳转指令或其目的地址的指令为止。具体实现是从入口点开始扫描每一条指令，把 push imm; call imm; 当作直接跳转，不分析直接跳转后面的指令，继续从直接跳转的目的地址开始分析。需要注意的是 push register;ret; 和 jmp register 需要看成是含有多个分支的跳转。然后利用一个队列来保存待分析的基本块首地址，代码实现如下：

```
def GetVmp1BasicBlock():  
    EntryPoint = 0x400000 + 0x0037E533   
 
    insn = ida_ua.insn_t()
 
    qInsnAddr = queue.Queue()   #保留待分析的跳转分支起始地址
    qInsnAddr.put(EntryPoint)
 
    while(not qInsnAddr.empty()):
 
        ea = start_ea = qInsnAddr.get()
        if(IsRedundant(ea)):    #已经加入基本块,不用在分析
            continue
 
        #print("trace start_ea:0x%08x" % start_ea)
        #分析start_ea开始的基本块
        while(ea != 0x006FDE0C):    #0x006FDE0B为虚拟机出口      
            #print("\tea:0x%08x" % ea)           
            InsnLen = ida_ua.decode_insn(insn, ea)
            if(0 == InsnLen):
                print("decode_insn(ea=0x%08x) failed!" % ea)
                return 0 
 
            if(insn.itype in g_callInsnList and insn.ops[0].type in g_immOprand):
                prevInsn = ida_ua.insn_t()
                prevAddr = ea - 5   #push immediate;占用5个字节
                prevLen = ida_ua.decode_insn(prevInsn, prevAddr)#调用decode_prev_insn可能会失败
 
                if(0 == prevLen):
                    print("decode_insn(0x%08x) failed!" % prevAddr)
                    return 0
 
                #push xxx;call xxx;把call看出直接跳转
                if(ida_allins.NN_push == prevInsn.itype and ida_ua.o_imm == prevInsn.ops[0].type):
                    end_ea = ea + insn.size
                    vbb = VMPBasicBlock(start_ea, end_ea, insn.ea)
                    g_vmp1BlockList.append(vbb)
 
                    AdjustBlockByJccTarget(insn.ops[0].addr)
                    qInsnAddr.put(insn.ops[0].addr)
                    #print("\tqInsnAddr.put(0x%08x)" % insn.ops[0].addr)
 
                else:#暂不分析其它类型的call
                    ea = ea + insn.size
                    continue
 
                break 
 
            elif(insn.itype in g_jccInsnList):
                end_ea = ea + insn.size
                vbb = VMPBasicBlock(start_ea, end_ea, insn.ea)
                g_vmp1BlockList.append(vbb)
 
                qInsnAddr.put(end_ea)   #jcc需要分析其后面的指令
                #print("\tqInsnAddr.put(0x%08x)" % end_ea)
                jccTarget = insn.ops[0].addr
                AdjustBlockByJccTarget(jccTarget)
                qInsnAddr.put(jccTarget)
                #print("\tqInsnAddr.put(0x%08x)" % jccTarget)   
                break
 
            elif(insn.itype in g_jmpInsnList):
                end_ea = ea + insn.size
                vbb = VMPBasicBlock(start_ea, end_ea, insn.ea)
                g_vmp1BlockList.append(vbb)
 
                if(insn.ops[0].type in g_immOprand):  #不分析jmp immediate后面的指令
 
                    JmpTarget = insn.ops[0].addr
                    AdjustBlockByJccTarget(JmpTarget)
                    qInsnAddr.put(JmpTarget)
                    #print("\tqInsnAddr.put(0x%08x)" % JmpTarget)   
 
 
                elif(ida_ua.o_reg == insn.ops[0].type): #jmp reg;
                    for JmpTarget in g_jmpRegDict[ea]:
                        AdjustBlockByJccTarget(JmpTarget)
                        qInsnAddr.put(JmpTarget)
                        #print("\tqInsnAddr.put(0x%08x)" % JmpTarget) 
 
                break
 
            elif(ida_allins.NN_retn == insn.itype):#主要是push reg;ret;指令，还有少部分跳入vmp1的ret           
                end_ea = ea + insn.size
                vbb = VMPBasicBlock(start_ea, end_ea, insn.ea)
                g_vmp1BlockList.append(vbb)
                '''
                prevInsn = ida_ua.insn_t()
                prevInsnLen = ida_ua.decode_insn(prevInsn, ea - 1)#push占用一个字节
                if(0 == prevInsnLen):
                    break
                if(ida_allins.NN_push != prevInsn.itype or ida_ua.o_reg != prevInsn.ops[0].type):
                    break
                '''
                if(None == g_RetAddrDict.get(insn.ea)): #会遍历到没有模拟执行过的ret指令,可能由条件分支指令造成的
                    print("warning:cannot find ret target! address:0x%08x" % insn.ea)
                    break
 
                for JmpTarget in g_RetAddrDict[insn.ea]:    #g_RetAddrDict的key为ret的地址
                    #if(0x004012F5 == JmpTarget):#0x004012F5为原入口点
                    #    continue
                    if(JmpTarget < 0x63b000 or JmpTarget > 0x820000):#vmp1 segment bound
                        print("ret from 0x%08x to 0x%08x" % (ea, JmpTarget))
                        continue
 
                    AdjustBlockByJccTarget(JmpTarget)
                    qInsnAddr.put(JmpTarget)
                    #print("\tqInsnAddr.put(0x%08x)" % JmpTarget) 
                break
 
            else:
                ea = ea + insn.size
                '''
                L1: xxx;
                    ...
                L2: xxx;
                    ...
                L3  jcc;
                当有一条先跳转到L2的指令时，后面又有一条跳转到L1的指令，
                会出现L1到L2的块且包含L2到L3的块
                '''
                if(IsRedundant(ea)):    #检测到另一个块的起始地址
                    vbb = VMPBasicBlock(start_ea, ea, insn.ea)
                    g_vmp1BlockList.append(vbb)
                    break
    return 1

```

执行完后发现有 7000 多个这样的块，所以需要合并一下那些直接跳转的块，也方便之后做活跃变量分析。合并和添加前驱和后继的代码就不展示了，具体参考源码中的 AddVMPBasicBlockPrevsAndSuccs 和 TryMergeBasicBlock 两个函数。

活跃变量分析
======

合并基本块后就可以做活跃变量分析了，在和合并后基本块内做活跃变量分析可以把每一条指令看作是一个结点，寄存器看作一个变量，然后利用使用和定值信息计算进入结点和结点后的活跃信息。这里给出《现代编译原理 - C 语言描述》第 10 章活跃分析的一个例子，方便大家理解使用、定值和活跃性。  
![](https://bbs.pediy.com/upload/attach/202102/742617_X2RV4425JFAUCG8.png)  
活跃性计算的方法如下：  
![](https://bbs.pediy.com/upload/attach/202102/742617_BGUH8RF729WF48S.png)  
按照上述方法计算后，会得到每一个结点的入口活跃信息和出口活跃信息。考虑到合并后的基本块有 1700 多个，如果按照上面的迭代方法计算的话会很慢，所以具体实现要优化一下，加快数据流分析。基本块内指令是线性执行的，不存在环和分支，活跃变量分析属于逆向数据流问题。如果能够安排每一个结点的计算都先于它的前驱，是可以通过对所有结点的一次遍历就能完成数据流分析，得到每一个结点的入口活跃和出口活跃信息。获取基本块内每一个指令的 use 和 def 信息的话，可以使用 capstone CsInsn 类的 regs_access 方法，这还可以获取到 eflags 寄存器。一开始是想使用 ida 的 microcode API 来实现的，但是我觉得 ida 提供的 python 接口不好用，没有提供针对一条指令的转换接口。  
对合并后的基本块的分析思路如下：  
1、首先利用 capstone 反汇编基本块内的指令，获取每一条指令的 use 和 def 信息。  
2、从基本块的出口处的指令向入口计算每一条指令的出口活跃信息和入口活跃信息。  
3、在获取到指令的活跃信息后，然后根据每一条指令的 def 信息，如果 def 中的所有变量都不属于出口活跃的，我们就可以删掉这条指令。  
具体代码实现如下：

```
def OptimizeMergedBasicBlock(mvbb, IsRadical = False):
    cs_insnList = []#capstone.CsInsn
    cs_use = {} #key为cs_insnList的下标,value为使用的寄存器集合,对变量的任何使用都会使该变量成为活跃的
    cs_def = {}#key为cs_insnList的下标,value为修改的寄存器集合,对变量的任何定值都会杀死该变量的活跃性
 
    vbbId = mvbb.FirstVbbId
    index = 0
    while(vbbId in mvbb.vbbIdList):#获取变量和定值信息
        vbb = g_vmp1BlockList[vbbId]
        #print("vbbid %d:0x%08x - 0x%08x"%(vbb.block_id,vbb.start_ea,vbb.end_ea))
        bCode = idc.get_bytes(vbb.start_ea, vbb.end_ea - vbb.start_ea)
        if(None == bCode):
            print("get_bytes(0x%08x,%d) failed!" % (vbb.start_ea, vbb.end_ea - vbb.start_ea))
            return 0
 
        for insn in g_md.disasm(bCode, vbb.start_ea):          
            cs_insnList.append(insn)
            useList,defineList = insn.regs_access()#(list-of-registers-read, list-of-registers-modified)包括eflags
            cs_use[index] = {cs_extendRegTo32bit(cs_reg) for cs_reg in useList}
            cs_def[index] = {cs_extendRegTo32bit(cs_reg) for cs_reg in defineList}
            index += 1
 
 
        if(len(vbb.succs) != 1):
            break
 
        vbbId = vbb.succs[0]
    '''   
    for i in range(len(cs_insnList)):
        use_strList=[]
        def_strList=[]
        for cs_reg in cs_use[i]:
            use_strList.append(cs_insnList[i].reg_name(cs_reg))
        for cs_reg in cs_def[i]:
            def_strList.append(cs_insnList[i].reg_name(cs_reg))           
        print("0x%08x %s %s;%r %r" % (cs_insnList[i].address, cs_insnList[i].mnemonic, cs_insnList[i].op_str,
            use_strList,def_strList))
    '''       
    bChanged = True
    cs_insnTempList=[]
    while(bChanged):
        bChanged = False
 
        Out= {} #In和Out的key为cs_insnList的下标
        In = {}
        for i in range(len(cs_insnList)):
            Out[i]=set()
            In[i]=set()
 
        Exit = len(cs_insnList)
 
        if(not IsRadical):
            In[Exit] = {capstone.x86_const.X86_REG_EBP, capstone.x86_const.X86_REG_EDI, \
            capstone.x86_const.X86_REG_ESI, capstone.x86_const.X86_REG_ESP, \
            capstone.x86_const.X86_REG_EAX, capstone.x86_const.X86_REG_EBX, \
            capstone.x86_const.X86_REG_ECX, capstone.x86_const.X86_REG_EDX}
 
        else:
            In[Exit] = {capstone.x86_const.X86_REG_EBP, capstone.x86_const.X86_REG_EDI, \
            capstone.x86_const.X86_REG_ESI, capstone.x86_const.X86_REG_ESP,}
 
        #如果安排每一个结点的后继都先于该结点计算，则有可能只通过对所有结点的一次遍历就能完成数据流分析。
        #由于基本块内是线性执行的,不存在分支和环，所以采用倒序
        for i in range(len(cs_insnList)-1,-1,-1):
            nSucc = i+1
            Out[i] |= In[nSucc]
            In[i] = cs_use[i] | (Out[i]-cs_def[i])
        '''
        for i in range(len(cs_insnList)):
            in1=[]
            out1=[]
            for cs_reg in In[i]:
                in1.append(cs_insnList[i].reg_name(cs_reg))
            for cs_reg in Out[i]:
                out1.append(cs_insnList[i].reg_name(cs_reg))           
            print("0x%08x in:%r out:%r" % (cs_insnList[i].address,in1,out1))
        '''  
        DelIdxList=[]
        for i in range(len(cs_insnList)):    
            count = len(cs_def[i])
            if(0 == count):     #不处理没有产生定值的语句
                continue
 
            for cs_reg in cs_def[i]:    #某些指令会产生多个定值，比如sub指令会修改通用寄存器和eflags寄存器           
                if(cs_reg not in Out[i]):   #所有产生的定值都不是出口活跃的，则删除
                    count -= 1                  
 
            if(0 == count):#对于产生的定值不在Out集合中，则可以被删除
                bChanged = True
                DelIdxList.append(i)
 
        i=0
        cs_insnTempList = []
        for index in range(len(cs_insnList)):
            if(index not in DelIdxList):    #保留已有的use和def信息，并调整index
                cs_insnTempList.append(cs_insnList[index])
                cs_use[i] = cs_use[index]
                cs_def[i] = cs_def[index]
                i += 1
            else:   #删除对应的条目 
                cs_use.pop(index)
                cs_def.pop(index)
                ida_bytes.patch_bytes(cs_insnList[index].address,b"\x90"*cs_insnList[index].size)
                #g_cs_insnMnemonicSet.add(cs_insnList[index].mnemonic)
 
        cs_insnList = cs_insnTempList
        #print("cs_insnList len:%d" % len(cs_insnList))
    '''
    for insn in cs_insnList:
        print("0x%08x %s %s" % (insn.address, insn.mnemonic, insn.op_str))
    '''   
    return 1

```

这里有几点需要说明一下，capstone 的 al，ah，ax 等 8 位或 16 位的寄存器是单独定义的，需要转换到 32 位，因为 vmp 有 8 位或 16 位寄存器参与到下一个 handle 地址的运算。或者也可以把这些寄存器添加到 In[Exit] 中，添加 In[Exit] 是为了方便计算，作为整个基本块的出口活跃信息，不属于任意一条指令。把一些通用寄存器添加到基本块的出口活跃信息，也是为了保证不会 nop 掉有用的指令。那个 IsRadical 的判断主要是为了处理 push imm；call target 中 target 处的虚拟机入口。只添加 ebp，esp，esi 和 edi 作为整个基本块出口活跃信息是为了减少 target 处基本块没有 nop 掉的垃圾指令。在 nop 掉指令前，需要注意的是，编译器在删除死代码的优化中会考虑到当前被删除的指令是否有副作用，比如是否为访存指令、call 指令等。本次样本中 vmp 的垃圾指令好像都没有副作用，不存在那些有副作用的指令，所以就没有考虑这些，nop 完之后样本是可以正常运行的。最后在处理以下垃圾指令，直接遍历每一个基本块遇到这类指令直接 nop 掉。  
![](https://bbs.pediy.com/upload/attach/202102/742617_GQZYUCZBAQ3YGXN.png)  
这里展示一下入口处进入虚拟机那部分 nop 掉垃圾指令后的代码和部分 x64dbg 的 trace 截图

```
.vmp1:0064D713 56                      push    esi
.vmp1:0064D714 90                      nop
.vmp1:0064D715 90                      nop
.vmp1:0064D716 90                      nop
.vmp1:0064D717 90                      nop
.vmp1:0064D718 90                      nop
.vmp1:0064D719 90                      nop
.vmp1:0064D71A 90                      nop
.vmp1:0064D71B 90                      nop
.vmp1:0064D71C 55                      push    ebp
.vmp1:0064D71D 52                      push    edx
.vmp1:0064D71E 51                      push    ecx
.vmp1:0064D71F 9C                      pushf
.vmp1:0064D720 90                      nop
.vmp1:0064D721 90                      nop
.vmp1:0064D722 90                      nop
.vmp1:0064D723 90                      nop
.vmp1:0064D724 90                      nop
.vmp1:0064D725 90                      nop
.vmp1:0064D726 50                      push    eax
.vmp1:0064D727 90                      nop
.vmp1:0064D728 57                      push    edi
.vmp1:0064D729 90                      nop
.vmp1:0064D72A 90                      nop
.vmp1:0064D72B 90                      nop
.vmp1:0064D72C 87 FF                   xchg    edi, edi
.vmp1:0064D72E 53                      push    ebx
.vmp1:0064D72F 90                      nop
.vmp1:0064D730 90                      nop
.vmp1:0064D731 90                      nop
.vmp1:0064D732 90                      nop
.vmp1:0064D733 90                      nop
.vmp1:0064D734 66 0F A4 E7 8F          shld    di, sp, 8Fh
.vmp1:0064D739 BA 00 00 00 00          mov     edx, 0
.vmp1:0064D73E 90                      nop
.vmp1:0064D73F 90                      nop
.vmp1:0064D740 90                      nop
.vmp1:0064D741 90                      nop
.vmp1:0064D742 90                      nop
.vmp1:0064D743 90                      nop
.vmp1:0064D744 90                      nop
.vmp1:0064D745 52                      push    edx
.vmp1:0064D746 90                      nop
.vmp1:0064D747 90                      nop
.vmp1:0064D748 8B 6C 24 28             mov     ebp, [esp+24h+arg_0]
.vmp1:0064D74C 81 ED 25 26 6F 1D       sub     ebp, 1D6F2625h
.vmp1:0064D752 0F CD                   bswap   ebp
.vmp1:0064D754 90                      nop
.vmp1:0064D755 90                      nop
.vmp1:0064D756 90                      nop
.vmp1:0064D757 90                      nop
.vmp1:0064D758 90                      nop
.vmp1:0064D759 90                      nop
.vmp1:0064D75A 90                      nop
.vmp1:0064D75B F7 DD                   neg     ebp
.vmp1:0064D75D 90                      nop
.vmp1:0064D75E 90                      nop
.vmp1:0064D75F 90                      nop
.vmp1:0064D760 90                      nop
.vmp1:0064D761 90                      nop
.vmp1:0064D762 90                      nop
.vmp1:0064D763 90                      nop
.vmp1:0064D764 90                      nop
.vmp1:0064D765 90                      nop
.vmp1:0064D766 90                      nop
.vmp1:0064D767 90                      nop
.vmp1:0064D768 90                      nop
.vmp1:0064D769 90                      nop
.vmp1:0064D76A 90                      nop
.vmp1:0064D76B F7 D5                   not     ebp
.vmp1:0064D76D 90                      nop
.vmp1:0064D76E 90                      nop
.vmp1:0064D76F 90                      nop
.vmp1:0064D770 90                      nop
.vmp1:0064D771 90                      nop
.vmp1:0064D772 81 ED 7B 53 BE 38       sub     ebp, 38BE537Bh
.vmp1:0064D778 90                      nop
.vmp1:0064D779 90                      nop
.vmp1:0064D77A 90                      nop
.vmp1:0064D77B 90                      nop
.vmp1:0064D77C 90                      nop
.vmp1:0064D77D 90                      nop
.vmp1:0064D77E 90                      nop
.vmp1:0064D77F C1 C5 03                rol     ebp, 3
.vmp1:0064D782 90                      nop
.vmp1:0064D783 90                      nop
.vmp1:0064D784 90                      nop
.vmp1:0064D785 90                      nop
.vmp1:0064D786 90                      nop
.vmp1:0064D787 90                      nop
.vmp1:0064D788 90                      nop
.vmp1:0064D789 8D 6C 15 00             lea     ebp, [ebp+edx+0]
.vmp1:0064D78D 8B F4                   mov     esi, esp
.vmp1:0064D78F 90                      nop
.vmp1:0064D790 90                      nop
.vmp1:0064D791 90                      nop
.vmp1:0064D792 90                      nop
.vmp1:0064D793 90                      nop
.vmp1:0064D794 90                      nop
.vmp1:0064D795 90                      nop
.vmp1:0064D796 90                      nop
.vmp1:0064D797 47                      inc     edi
.vmp1:0064D798 81 EC C0 00 00 00       sub     esp, 0C0h
.vmp1:0064D79E 90                      nop
.vmp1:0064D79F 90                      nop
.vmp1:0064D7A0 90                      nop
.vmp1:0064D7A1 90                      nop
.vmp1:0064D7A2 90                      nop
.vmp1:0064D7A3 90                      nop
.vmp1:0064D7A4 90                      nop

```

![](https://bbs.pediy.com/upload/attach/202102/742617_EDQCTSZGECEEVKW.png)  
![](https://bbs.pediy.com/upload/attach/202102/742617_TEU8U4SZWCGA8G3.png)

总结
==

根据实际运行效果，说明我的分析思路是大体正确的。这里没有根据基本块作为一个结点做全局的活跃变量分析是因为把通用寄存器作为活跃分析中的变量是不适合这么做的。因为通用寄存器的数量有限，是重复使用的资源，其活跃性很容易在下一个基本块被杀死。要做全局的活跃变量分析的话，应先把整个流图转换到 SSA 形式，这样应该可以干掉那些漏网之鱼了。考虑到工作量有点大，就没有这么做了。上传的源码中我也实现了一个获取合并后基本块的出口活跃和入口活跃信息的函数，不是 SSA 形式的。只是写来巩固一下自己所学知识点而已，对去除垃圾指令也没什么作用，大家有兴趣的话可以参考一下。也没有使用迭代的方法，而是做了一部分优化，通过工作表算法和对结点的深度优先搜索遍历序号进行计算的。代码实现在 GlobalLiveness 函数中。优化方法可以参考《现代编译原理 - C 语言描述》17 章的加快数据流分析部分

[[公告] 推荐好文功能上线，分享知识还可以得雪币！推荐一篇文章获得 20 雪币！](https://zhuanlan.kanxue.com/article-external_link.htm)

上传的附件：

*   [EliminateVmpJunkCode.zip](javascript:void(0)) （7.57MB，11 次下载）