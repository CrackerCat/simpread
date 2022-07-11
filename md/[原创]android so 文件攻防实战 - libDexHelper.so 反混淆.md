> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273614.htm)

> [原创]android so 文件攻防实战 - libDexHelper.so 反混淆

计划是写一个 android 中 so 文件反混淆的系列文章，目前这是第三篇。  
第一篇：[android so 文件攻防实战 - 百度加固免费版 libbaiduprotect.so 反混淆](https://bbs.pediy.com/thread-271388.htm)  
第二篇：[android so 文件攻防实战 - 某团 libmtguard.so 反混淆](https://bbs.pediy.com/thread-271853.htm)  
今天分析的是企业版 64 位，我用 LibChecker 查了一下手机上的 APP 找到的，时间也还比较新。根据其他人的分析可知，libDexHelper.so 是指令抽取的实现，libdexjni.so 是 VMP 的实现。

去除混淆
----

首先因为加密过，肯定是不能直接反编译的，可以在 libart.so 下断点，进入 JNI_onLoad 以后就可以 dump 下来。  
![](https://bbs.pediy.com/upload/tmp/734571_MUERXQMXWUZW8C9.jpg)  
不过此时也不能直接 F5，还存在以下混淆方式：  
1. 垃圾指令  
![](https://bbs.pediy.com/upload/tmp/734571_NTKBTYX3ZGZW44J.png)  
这些垃圾指令是在 switch 的一个永远不会被执行到的分支里面，可以直接将 IDA 不能 MakeCode 的地方 patch 成 NOP 再 MakeCode。  
2. 字符串加密  
有好几个解密字符串的函数，0x186C4，0x7783C，0x95B9C。在 [android so 文件攻防实战 - 百度加固免费版 libbaiduprotect.so 反混淆](https://bbs.pediy.com/thread-271388.htm)中我们是交叉引用拿到加密后的字符串和它对应的解密函数的表然后 frida 主动调用得到的解密后的字符串，但是在这里这个方法就不太好用了。因为这里加密后的字符串是在栈上一个 byte 一个 byte 拼起来的，和最后调用解密函数之间可能隔了很多条指令，甚至都不在一个 block。  
我最后用的是下面这种方案：以 0x40110 处调用 0x186C4 处的解密函数为例，这里面字符串解密的逻辑比较简单，需要三个参数。我们可以自己实现也可以用 unicorn，我就用 unicorn 了。  
![](https://bbs.pediy.com/upload/tmp/734571_7KHQTMV635URVY9.png)  
![](https://bbs.pediy.com/upload/tmp/734571_BKXHC8P5CGP3V4K.png)

```
import sys
import unicorn
import binascii
import threading
import subprocess
 
from capstone import *
from capstone.arm64 import *
 
with open("C:\\Users\\hjy\\Downloads\\out1.fix.so","rb") as f:
    sodata = f.read()
 
uc = unicorn.Uc(unicorn.UC_ARCH_ARM64, unicorn.UC_MODE_ARM)
code_addr = 0x0
code_size = 8*0x1000*0x1000
uc.mem_map(code_addr, code_size)
stack_addr = code_addr + code_size
stack_size = 0x1000000
stack_top = stack_addr + stack_size - 0x8
uc.mem_map(stack_addr, stack_size)
uc.mem_write(code_addr, sodata)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X29, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X28, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X27, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X26, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X25, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X24, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X23, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X22, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X21, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X20, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X19, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X18, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X17, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X16, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X15, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X14, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X13, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X12, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X11, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X10, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X9, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X8, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X7, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X6, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X5, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X4, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X3, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X2, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X1, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X0, stack_addr)
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_SP, stack_top)
X0 = uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X0)
 
uc.mem_write(X0, bytes.fromhex(sys.argv[1]))
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X1, int(sys.argv[2], 16))
uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X2, int(sys.argv[3], 16))
 
uc.emu_start(0x1777C, 0x17780)
 
X0 = uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X0)
decstr = uc.mem_read(X0, 80)
 
print("decstr:", decstr)
uc.mem_unmap(stack_addr, stack_size)
uc.mem_unmap(code_addr, code_size)
```

总共有几百处调用，不可能全部人工去这样解出来，我写了另外以一个脚本去调用 decstr.py。首先通过交叉引用找到所有调用解密函数的地方，然后把起始地址设为该 block 的起始地址，结束地址设为调用解密函数的地址，通过 unicorn 跑出 decstr.py 需要的三个参数之后调用 decstr.py。遇到 unicorn.unicorn.UcError 也有两个处理策略，一个是跳过该地址 (loop_call_prepare_arg1)，起始地址不变；一个是将起始地址设为下一条地址 (loop_call_prepare_arg2)。当然这套方案还有优化的空间，比如生成调用解密函数需要的参数的代码和最后调用解密函数的代码不在一个 block，就处理不了。

```
import unicorn
import binascii
import threading
import subprocess
 
from capstone import *
from capstone.arm64 import *
 
inscnt = 0
start_addr = 0
end_addr = 0
stop_addr = 0
stop_addr_list = []
 
def hook_code(uc, address, size, user_data):
    global inscnt
    global end_addr
    global stop_addr
    global stop_addr_list
 
    md = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
 
    for ins in md.disasm(sodata[address:address + size], address):
        #rint(">>> 0x%x:\t%s\t%s" % (ins.address, ins.mnemonic, ins.op_str))
        stop_addr = ins.address
 
        if ins.address in stop_addr_list:
            #print("will pass 0x%x:\t%s\t%s" %(ins.address, ins.mnemonic, ins.op_str))
            uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_PC, address + size)
            return
 
        inscnt = inscnt + 1
        if (inscnt > 500):
            uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_PC, 0xffffffff)
            return
 
        if ins.mnemonic.find("b.") != -1:
            print("will pass 0x%x:\t%s\t%s" %(ins.address, ins.mnemonic, ins.op_str))
            uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_PC, address + size)
            return
 
        if ins.mnemonic.find("bl") != -1:
            print("will pass 0x%x:\t%s\t%s" %(ins.address, ins.mnemonic, ins.op_str))
            uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_PC, address + size)
            return
 
            if ins.op_str in ["x0","x1","x2","x3"]:
                    X1 = uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X1)
                    if X1 > 0x105A88:
                        print("will pass 0x%x:\t%s\t%s" %(ins.address, ins.mnemonic, ins.op_str))
                        uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_PC, address + size)
                        return
            if ins.op_str.startswith("#0x"):
                addr = int(ins.op_str[3:],16)
                if (addr > 0x14E50 and addr < 0x15820) \
                or addr == 0x186C4 \
                or addr > 0x105A88:
                    print("will pass 0x%x:\t%s\t%s" %(ins.address, ins.mnemonic, ins.op_str))
                    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_PC, address + size)
                    return
 
def call_prepare_arg():
    global inscnt
    global start_addr
    global end_addr
    global stop_addr
    global stop_addr_list
 
    inscnt = 0
 
    uc = unicorn.Uc(unicorn.UC_ARCH_ARM64, unicorn.UC_MODE_ARM)
    code_addr = 0x0
    code_size = 8*0x1000*0x1000
    uc.mem_map(code_addr, code_size)
    stack_addr = code_addr + code_size
    stack_size = 0x1000000
    stack_top = stack_addr + stack_size - 0x8
    uc.mem_map(stack_addr, stack_size)
    uc.hook_add(unicorn.UC_HOOK_CODE, hook_code)
 
    uc.mem_write(code_addr, sodata)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X29, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X28, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X27, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X26, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X25, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X24, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X23, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X22, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X21, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X20, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X19, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X18, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X17, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X16, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X15, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X14, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X13, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X12, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X11, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X10, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X9, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X8, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X7, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X6, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X5, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X4, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X3, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X2, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X1, stack_addr)
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_X0, stack_addr)
 
    uc.reg_write(unicorn.arm64_const.UC_ARM64_REG_SP, stack_top)
    uc.emu_start(start_addr, end_addr)
 
    X0 = uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X0)
    decstr = uc.mem_read(X0, 80)
    end_index = decstr.find(bytearray(b'\x00'), 1)
    decstr = decstr[:end_index]
 
    decstr = binascii.b2a_hex(decstr)
    decstr = decstr.decode('utf-8')
 
    X1 = uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X1)
    X2 = uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X2)
 
    pi = subprocess.Popen(['C:\\Python38\\python.exe', 'decstr.py', decstr, hex(X1), hex(X2)], stdout=subprocess.PIPE)
    output = pi.stdout.read()
    print(output)
 
def loop_call_prepare_arg1():
    global inscnt
    global end_addr
    global stop_addr
    global stop_addr_list
 
    loopcnt = 0
    stop_addr_list = []
 
    while True:
        try:
            loopcnt = loopcnt + 1
            if(loopcnt > 200):
                break
            call_prepare_arg()
        except unicorn.unicorn.UcError:
            print("adding....")
            print(hex(stop_addr))
            stop_addr_list.append(stop_addr)
        else:
            break
 
def loop_call_prepare_arg2():
    global inscnt
    global end_addr
    global stop_addr
    global stop_addr_list
 
    global start_addr
 
    loopcnt = 0
    stop_addr_list = []
 
    while True:
        try:
            loopcnt = loopcnt + 1
            if(loopcnt > 200):
                break
            call_prepare_arg()
        except unicorn.unicorn.UcError:
            start_addr = stop_addr + 4
        else:
            break
 
with open("C:\\Users\\hjy\\Downloads\\out1.fix.so","rb") as f:
    sodata = f.read()
 
all_addr = []
with open('xref_decstr.txt', 'r', encoding='utf-8') as f:
    for line in f:
        addr = "0x" + line[2:]
        addr = int(addr, 16)
        all_addr.append(addr)
 
for i in all_addr:
 
    print("i:")
    print(hex(i))
 
    end_addr = i
    CODE = sodata[i - 4:i]
    md = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
    for x in md.disasm(CODE, i - 4):
        mnemonic = x.mnemonic
 
    while mnemonic != "ret" \
        and mnemonic != "b" \
        and mnemonic != "br" \
        and mnemonic != "cbz" \
        and mnemonic != "cbnz":
        i = i - 4
        CODE = sodata[i - 4:i]
        for x in md.disasm(CODE, i - 4):
            mnemonic = x.mnemonic
 
    start_addr = i
 
    print("start_addr:")
    print(hex(start_addr))
    print("end_addr:")
    print(hex(end_addr))
 
    loop_call_prepare_arg1()
    loop_call_prepare_arg2()
```

更恶心的是还有很多字符串是自己在函数内解密的，这种情况我也没想到有什么好的方法。  
3. 控制流混淆  
第一种是把正常顺序执行的指令打乱成 switch 的形式，这个影响倒不是太大：  
![](https://bbs.pediy.com/upload/tmp/734571_UCHPFBYC4HRFQB9.png)  
第二种是动态计算跳转地址，基本上类似于在 [android so 文件攻防实战 - 某团 libmtguard.so 反混淆](https://bbs.pediy.com/thread-271853.htm)见过的那种，但是要更复杂。  
![](https://bbs.pediy.com/upload/tmp/734571_X8Q5S7DVWVRD63N.png)  
比如这里的指令，在 0x1DA0C 处给 X2 赋值，X2 此时为. data 段中的一个地址，W0 为偏移，取出值后在 0x1DA18 处乘 4 加上 0x1DA20，最后的值就是 0x1DA1C 处 X0 的值。那么需要解决这么几个问题：  
如何确定 0x1DA0C 处给 X2 赋的值  
将 0x1DA00 处的指令改成跳转指令，0x1DA00 这个地址又该如何确定  
找到所有会跳转到 0x1DA1C 的指令，将跳转地址改成计算出来的 X0 的值  
第一个问题，其实和字符串解密面临的情况是类似的，比如这里需要找到 "LDR XX, [X29,#0x190+var_118]" 这条指令，然后再找给 XX 寄存器赋值的指令，然而这两条指令很可能和 BR X0 隔了好几个 block。我的解决方法是通过 IDA 提供的 idaapi.FlowChar 功能，递归前面的 block，找到需要的指令。不足之处在于前提条件是 IDA 正确识别了函数的起始地址，否则会出现我们需要的指令和 BR X0 不在同一个函数的情况，这样就处理不了。  
第二个问题，在递归前面的 block 的时候就先找到 0x1D9D4 处这条给 W0 赋值的指令，然后从 0x1D9D4 处开始直到 0x1DA1C，找到第一个存在交叉引用的地址，也就是 0x1DA04。它的前一条指令 0x1DA00 就是需要改成跳转指令的地方。  
第三个问题，确定了 0x1DA00 之后，那么从 0x1DA00 到 0x1DA1C 所有存在交叉引用的地址都要去交叉引用的地方修改跳转地址。不过这里有很多细节。  
(1) 如果 W0 是由 CSEL，CSET，CSINC 这些指令赋值的，像下面这种情况，那么需要把 0x1DE80 和 0x1DE84 修改成 B.GE 和 B.LT。  
patch 前：  
![](https://bbs.pediy.com/upload/tmp/734571_MTNQTMSB3JZ3QW6.png)  
patch 后：  
![](https://bbs.pediy.com/upload/tmp/734571_QPT644VFGYA4HMD.png)  
(2)0x1DE80 处的`CSEL W0, WZR, W8, LT`，这里 W8 的值是在`0x1D9DC MOV W8, #5`赋值的，所以我的代码中有一个 register_value_dict，在改掉 0x1DA00 处的指令之后会读取 0x1DA00 所在的 block 到 0x1DA1C 所在的 block 的所有指令，找到给寄存器赋值的指令然后把值存起来。  
![](https://bbs.pediy.com/upload/tmp/734571_8R6FV9QREF2HZYU.png)  
![](https://bbs.pediy.com/upload/tmp/734571_QQXGSB3ZTU4Z2P3.png)  
(3) 有些地方还会有一条 sub 指令，这个也要考虑进去，比如下面这种情况 0x33394 处跳转的地址就应该按照 W8 为 4 计算。  
![](https://bbs.pediy.com/upload/tmp/734571_6Q3VEKDNFJ2UFS5.png)  
![](https://bbs.pediy.com/upload/tmp/734571_RBX7NRZEKUVEKWR.png)  
最后的脚本放附件了。当然还有一些脚本处理不了的地方，不过问题已经不算太大了，需要的话可以动态调试确定。  
4. 函数地址动态计算  
![](https://bbs.pediy.com/upload/attach/202207/734571_VDK9X3WF87FAD5V.png)  
![](https://bbs.pediy.com/upload/tmp/734571_A264FTHP7ZZAQGJ.png)  
![](https://bbs.pediy.com/upload/attach/202207/734571_YUKEP8N5KFUR5D6.png)  
这个在 IDA 里面是能看清楚的，v35 其实就是 off_12EB80[0]，即调用 0x80FE0 处的 p329AAB59961F6410ABA963EF972FE303。  
接下来我们就来分析 libDexHelper.so，来看看它都干了些什么。精力有限，很多地方没能很详细去分析。有些地方分析的可能也不一定对，将就看吧。

功能分析
----

JNI_OnLoad(0x3EA68) 的分析在最后。

### 0x15960

读 / proc/self/maps，特征字符串：

```
libDexHelper.so
libDexHelper-x86.so
libDexHelper-x86_64.so
/system/lib64/libart.so
/system/lib64/libLLVM.so
/system/framework/arm64/boot-framework.oat
/system/lib64/libskia.so
/system/lib64/libhwui.so
.oat
ff c3 01 d1 f3 03 04 aa f4 03 02 aa f5 03 01 aa e8 03 00 aa
GumInvocationListener
GSocketListenerEvent
```

### 0x16A30

获取系统属性，读 / proc/%d/cmdline，特征字符串：

```
ro.yunos.version
ro.yunos.version.release
persist.sys.dalvik.vm.lib
persist.sys.dalvik.vm.lib.2
/system/bin/dex2oat
LD_OPT_PACKAGENAME
LD_OPT_ENFORCE_V1
```

off_12EF10: 为 2 表示 yunos，art 模式；为 1 表示 yunos，dalvik 模式；为 0 表示非 yunos。

### 0x17A70

md5。

### 0x186C4

字符串解密函数。

### 0x19674

返回字符串 rw。

### 0x19778

返回字符串 su。

### 0x1987C

返回字符串 mount。

### 0x19998

写 classes.dve 文件。

### 0x19b48

读取目录中的文件。

### 0x19E08

创建 String 类型的数组，第一个参数是 String 列表，第二个参数是数组长度。

### 0x1A058

调用 0x19E08 创建数组：

```
/etc
/sbin
/system
/system/bin
/vendor/bin
/system/sbin
/system/xbin
```

### 0x1A740

调用 0x19E08 创建数组：

```
com.yellowes.su
eu.chainfire.supersu
com.noshufou.android.su
com.thirdparty.superuser
com.koushikdutta.superuser
com.noshufou.android.su.elite
```

### 0x1AF1C

调用 0x19E08 创建数组：

```
com.chelpus.lackypatch
com.ramdroid.appquarantine
com.koushikdutta.rommanager
com.dimonvideo.luckypatcher
com.ramdroid.appquarantinepro
com.koushikdutta.rommanager.license
```

### 0x1B7D0

调用 0x19E08 创建数组：

```
com.saurik.substrate
com.formyhm.hideroot
com.amphoras.hidemyroot
com.devadvance.rootcloak
com.formyhm.hiderootPremium
com.devadvance.rootcloakplus
com.amphoras.hidemyrootadfree
com.zachspong.temprootremovejb
de.robv.android.xposed.installer
```

### 0x1C40C

system_property_get ro.product.cpu.abi 和读 / system/lib/libc.so 判断是不是 x86 架构。

### 0x1C61C

查看 classes.dve 是否存在。

### 0x1C8D8

调用 0x19E08 创建数组：

```
/sbin/
/su/bin/
/data/local/
/system/bin/
/system/xbin/
/data/local/bin/
/system/sd/xbin/
/data/local/xbin/
/system/bin/.ext/
/system/bin/failsafe/
/system/usr/we-need-root/
```

### 0x1D518

初始化一些路径，特征字符串：

```
.cache
oat
.payload
v1filter.jar
classes.odex
classes.vdex
classes.dex
assets/classes.jar
.cache/classes.jar
.cache/classes.dex
.cache/classes.odex
.cache/classes.vdex
```

### 0x1E520

将 libc 中的一些函数的地址放到. DATA。

```
0x137BB0 fopen
0x137BB8 fclose
0x137BC0 fgets
0x137BC8 fwrite
0x137BD0 fread
0x137BD8 sprintf
0x137BE0 pthread_create
```

### 0x1F250

读 / proc/self/cmdline，判断是否含有 com.miui.packageinstaller 从而判断是否由小米应用包管理组件启动。

### 0x1F710

先 system_property_get ro.product.manufacturer 和 system_property_get ro.product.model 判断是否是 samsung，然后 system_property_get ro.build.characteristics 是否为 emulator。

### 0x1FDC8

注册如下 native 函数：

```
RegisterNative(com/secneo/apkwrapper/H, attach(Landroid/app/Application;Landroid/content/Context;)V, RX@0x4002f6e4[libDexHelper.so]0x2f6e4)
RegisterNative(com/secneo/apkwrapper/H, b(Landroid/content/Context;Landroid/app/Application;)V, RX@0x400247c0[libDexHelper.so]0x247c0)
RegisterNative(com/secneo/apkwrapper/H, c()V, RX@0x40024c08[libDexHelper.so]0x24c08)
RegisterNative(com/secneo/apkwrapper/H, d(Ljava/lang/String;)Ljava/lang/String;, RX@0x40023d04[libDexHelper.so]0x23d04)
RegisterNative(com/secneo/apkwrapper/H, e(Ljava/lang/Object;Ljava/util/List;Ljava/lang/String;)[Ljava/lang/Object;, RX@0x40035ab0[libDexHelper.so]0x35ab0)
RegisterNative(com/secneo/apkwrapper/H, f()[Ljava/lang/String;, RX@0x4001a740[libDexHelper.so]0x1a740)
RegisterNative(com/secneo/apkwrapper/H, g()[Ljava/lang/String;, RX@0x4001af1c[libDexHelper.so]0x1af1c)
RegisterNative(com/secneo/apkwrapper/H, h()[Ljava/lang/String;, RX@0x4001b7d0[libDexHelper.so]0x1b7d0)
RegisterNative(com/secneo/apkwrapper/H, n()[Ljava/lang/String;, RX@0x4001c8d8[libDexHelper.so]0x1c8d8)
RegisterNative(com/secneo/apkwrapper/H, j()[Ljava/lang/String;, RX@0x4001a058[libDexHelper.so]0x1a058)
RegisterNative(com/secneo/apkwrapper/H, k()Ljava/lang/String;, RX@0x40019778[libDexHelper.so]0x19778)
RegisterNative(com/secneo/apkwrapper/H, l()Ljava/lang/String;, RX@0x4001987c[libDexHelper.so]0x1987c)
RegisterNative(com/secneo/apkwrapper/H, m()Ljava/lang/String;, RX@0x40019674[libDexHelper.so]0x19674)
RegisterNative(com/secneo/apkwrapper/H, bb(Landroid/content/Context;Landroid/app/Application;Landroid/app/Application;)V, RX@0x4002921c[libDexHelper.so]0x2921c)
RegisterNative(com/secneo/apkwrapper/H, o(Landroid/content/Context;)I, RX@0x4002f158[libDexHelper.so]0x2f158)
RegisterNative(com/secneo/apkwrapper/H, p()V, RX@0x4001875c[libDexHelper.so]0x1875c)
RegisterNative(com/secneo/apkwrapper/H, q()I, RX@0x40023568[libDexHelper.so]0x23568)
RegisterNative(com/secneo/apkwrapper/H, mu()I, RX@0x4001f250[libDexHelper.so]0x1f250)
```

### 0x218A8

system_property_get ro.build.version.release/ro.build.version.sdk/ro.build.version.codename，最终返回 sdkversion。

### 0x22068

创建一些目录：

```
/data/usr/0/包名/.cache/oat
/data/usr/0/包名/.cache/oat/arm64
/data/usr/0/包名/.payload
```

### 0x22a90

模拟器检测，特征字符串：

```
vboxsf
/mnt/shared/install_apk
nemusf
/mnt/shell/emulated/0/Music sharefolder
/sdcard/windows/BstSharedFolder
```

### 0x23568

读 proc/pid/cmdline 找字符串 ":bbs"，没搞懂这是什么意思。这个函数名是 is_magisk_check_process。

### 0x247C0

调用 setOuterContext。

### 0x24C08

system_property_get ro.product.brand，针对华为 / 荣耀机型，调用 startLoadFromDisk。

### 0x26278

getDeclaredFields 获取 field 对象数组之后调用 equals，返回查找的指定的 field 对象。

### 0x27290

修改 mInitialApplication 和 mClassLoader。

### 0x2921C

修改 mAllApplications(remove 和 add)。

### 0x29CE8

模拟器检测，特征字符串：

```
com.bignox.app.store.hd
com.bluestacks.appguidance
com.bluestacks.settings
com.bluestacks.home
com.bluestack.BstCommandProcessor
com.bluestacks.appmart
```

### 0x2B670

通过 FLAG_DEBUGGABLE 判断是 debug 还是 release。

### 0x2CAE0

通过 android.content.pm.Signature 获取签名的 md5。

### 0x2F158

通过 access 以下文件判断是否被 root：

```
/sbin/.magisk/
/sbin/.core/img
/sbin/.core/mirror
/sbin/.core/db-0/magisk.db
```

### 0x2F6E4

读 / proc/self/cmdline，调用 java 层的 com.secneo.apkwrapper.H.j，调用 bindService，获取 android_id，调用 android.app.Application.attach，如果包名是 com.huawei.irportalapp.uat 调用 setOuterContext。

### 0x31474

调用 java 层的 com.secneo.apkwrapper.H.f(ff) 加载 v1filter.jar。  
查看 / proc/self/maps:  
anon:dalvik-classes.dex extracted in memory from v1filter.jar

### 0x3371C

hook libcutils.so/liblog.so 中的 android_log_write 和 android_log_buf_write，使其返回 0。

### 0x339FC

currentActivityThread-mPackages-LoadedApk-mResources-getAssets。

### 0x34A00

调用 android.content.res.Resources.getAssets，失败再调用 0x339FC。

### 0x351DC

读取 assets 文件。

### 0x35AB0

对传入参数调用 makeInMemoryDexElements，修改 dalvik.system.DexFile.mFileName。

### 0x3766C

初始化下列字符串:

```
/data/user/0/cn.missfresh.application/.cache/classes.jar
/data/user/0/cn.missfresh.application/.cache/classes.dex
/data/user/0/cn.missfresh.application/.cache/v1filter.jar
```

调用 0x80458 计算包名 hash，调用 0x75AA8  
调用 AAssetManager_open 读取 assets/resthird.data 写入 v1filter.jar，调用 0x31474  
(看别人的分析应该读 assets 下面两个文件：classes0.jar 是被加密的 dex，classes.dgc 是被加密的抽取后的指令。不过我分析的这个样本中没有 classes0.jar 和 classes.dgc，v1filter.jar 看了一下也并不是原始 dex，有点没太搞懂这个样本怎么加固的)

### 0x398F8/0x3A08C

检测 dexhunter，dumpclass 好像是 dexhunter 里面的吧。特征字符串：

```
_Z16hprofDumpClassesP15hprof_context_t
_Z12dvmDumpClassPK11ClassObjecti
_Z9dumpClassP7DexFilei
dumpclass
dump_class
```

### 0x3BF10

参数是文件名，返回文件是否存在。

### 0x3BF7C

模拟器检测，特征字符串：

```
ueventd.ttVM_x86.rc
init.ttVM_x86.rc
fstab.ttVM_x86
bluestacks
BlueStacks
```

### 0x3CE14

system_property_get ro.debuggable，调用检测模拟器的函数。

### 0x3D814

通过 android.hardware.usb.action.USB_STATE 监听 USB 状态。

### 0x42378

md5。

### 0x44708

hook 下列函数 (反调试)：

```
vmDebug::notifyDebuggerActivityStart(hook后：0x446C0)
art::Dbg::GoActive(hook后：0x446E4)
art::Runtime::AttachAgent(hook后：0x45CF8)
```

### 0x46194

system_property_get ro.yunos.version。

### 0x4C2F0

hook 下列函数 (指令抽取还原)：

```
art::ClassLinker::DefineClass(hook后：0x46BB8)
art::ClassLinker::LoadMethod(hook后：0x46ED4/0x47BB8/0x488C0/0x491F8/0x49B0C)
art::OatFile::OatMethod::LinkMethod(hook后：0x46BD8/0x46DB0)
```

### 0x4DB80

md5。

### 0x50280

读 / proc/self/maps 找到含有包名的段。

### 0x5074C

调用 java 层的 com.secneo.apkwrapper.H1.find_dexfile。

### 0x50B60

调用 java.lang.StackTraceElement.getMethodName 和 java.lang.StackTraceElement.getClassName。

### 0x57424

加载 assets 中的 classes.dgg。

### 0x598FC

读 / proc/self/maps 找到 libDexHelper.so。

### 0x59CE8

设置 dex2oat 的参数，--zip-fd/--oat-fd/--zip-location/--oat-location/--oat-file/--instruction-set。

### 0x5C600

hook libdvm.so 中的函数 (类似于 0x67544)，具体没仔细看，0x5BAA8-0x5BEF8 都是被 hook 后的实现。

### 0x61E3C

hook libc 中的下列函数：

```
fstatat64(hook后：0x5E778)
stat(hook后：0x5E858)
close(hook后：0x5EA20)
openat(hook后：0x5ED20)
open(hook后：0x5ED9C)
pread(hook后：0x5FAB8)
read(hook后：0x5FC14)
mmap64(hook后：0x5FDDC)
__openat_2(hook后：0x5FEF4)
__open_2(hook后：0x5FF74)
```

### 0x64AE8

根据不同 SDK 版本返回 Name Mangling 之后的 art::DexFileLoader::open。

### 0x65FE4

根据不同 SDK 版本返回 Name Mangling 之后的 art::OatFileManager::OpenDexFilesFromOat。

### 0x67544

hook 下列函数：

```
art::DexFileLoader::open(hook后：0x6D39C/0x6D3E8)
art::OatFileManager::OpenDexFilesFromOat(hook后：0x6A2C0/0x6AF14/0x6B9B0/0x6C188/0x6CB5C)
```

### 0x6D4A0

patch 掉 art::Runtime::IsVerificationEnabled。

### 0x6DAD8

hook art::DexFileVerifier::Verify(hook 后：0x6D38C/0x6D394，直接返回 1)。

### 0x6E40C

hook art::DexFileLoader::open(hook 后：0x6D39C/0x6D3E8)。

### 0x70410

hook 下列函数：

```
art::DexFileVerifier::Verify(hook后：0x6EB04/0x6EB0C/0x6EB14，直接返回1)
art::DexFile::OpenMemory(hook后：0x74EE8/0x74E90/0x74F38)
Art::DexFile(hook后：0x74E30/0x74F88)
```

### 0x75054

hook libdvm.so 中的函数，具体没仔细看，0x6EB1C/0x74DEC/0x6FFBC 都是被 hook 后的实现。

### 0x75AA8

读 java.lang.DexCache.dexfile(这个 dexfile 就是解压 apk 之后根目录的那个 classes.dex)。

### 0x767F8

参数是 so 文件路径，打开该 so 文件。

### 0x76C8C

参数是 libart.so 中的一个函数，返回该函数地址。

### 0x76CCC

第一个参数是 so 中的函数名，第二个参数是 so 的相对路径，返回该函数在 so 中的地址。

### 0x76D90

参数是 libdexfile.so 中的一个函数，返回该函数地址。

### 0x76DD4

参数是 libjdwp.so 中的一个函数，返回该函数地址。

### 0x76E18

md5。

### 0x7783C

字符串解密函数。

### 0x79270

计算传入字符串的 hash(不完全是 md5)。

### 0x7A240

热补丁检测，特征字符串：

```
nuwa
andfix
hotfix
.RiskStu
tinker
```

### 0x80458

调用 0x79270。

### 0x804A8

hook libc 中的下列函数：

```
msync(hook后：0x78470)
close(hook后：0x7AF50)
munmap(hook后：0x7A568)
openat64(hook后：0x7DC48)
__open_2(hook后：0x7DC80)
_open64(hook后：0x7DCB8)
_openat_2(hook后：0x7DCF0)
ftruncate64(hook后：0x7DD30)
mmap64(hook后：0x7EF60)
pread64(hook后：0x7F5D0)
read(hook后：0x7F7DC)
write(hook后：0x8022C)
```

### 0x87F98

hook libdvm.so 中的函数 (类似于 0x44708)，具体没仔细看，0x856C0/0x87F00/0x87F4C 都是被 hook 后的实现。

### 0x889B0

patch 掉 art::Runtime::UseJitCompilation。

### 0x8A794/0x8B71C/0x8B890

hook 函数实现。

### 0x917E8

读 / proc/sys/fs/inotify/max_queued_watches。

### 0x91848

读 / proc/sys/fs/inotify/max_user_instances。

### 0x918A8

读 / proc/sys/fs/inotify/max_user_watches。

### 0x95778

看起来好像是通过判断时间实现的反调试。

### 0x95A28

字符串查找函数。

### 0x95B9C

字符串解密函数。

### 0x95D60

socket 连接。

### 0x96398

frida 检测，读 / proc/self/task，特征字符串：gum-js-loop；读 / proc/self/fd，特征字符串 linjector。

### 0x995D0

xposed 检测，特征字符串：

```
.xposed.
xposedbridge
xposed_art
```

### 0x99D28

hook 框架检测，特征字符串：

```
frida
ddi_hook
dexposed
substrate
adbi_hook
MSFindSymbol
hook_precall
hook_postcall
MSHookFunction
DexposedBridge
MSCloseFunction
dexstuff_loaddex
dexposedIsHooked
ALLINONEs_arthook
dexstuff_resolv_dvm
dexposedCallHandler
art_java_method_hook
artQuickToDispatcher
dexstuff_defineclass
dalvik_java_method_hook
art_quick_call_entrypoint
frida_agent_main
```

### 0x9C0BC

调用 0x96398 检测 frida，system_property_get ro.product.model，调用 0x9FD88 检测 xposed 和自动脱壳机，hook dlopen(hook 后：0x9B89C) 和 ptrace(hook 后：0x95BF8)。

### 0x9CFCC

通过读取 / proc/%d/status 判断 TracerPid 等实现反调试。

### 0x9D878

通过读取 / proc/%d/wchan 判断是不是 ptrace_stop 实现反调试。

### 0x9DCF4

通过读取 / proc/%ld/task/%ld/status 判断 TracerPid 等实现反调试。

### 0x9ED44

通过 java.lang.StackTraceElement.getClassName 打印函数调用栈进行 xposed 检测。

### 0x9F770

通过 java.lang.ClassLoader.getSystemClassLoader.loadClass 打印类加载器进行 xposed 检测。

### 0x9FD88

调用 0x9ED44 和 0x9F770，通过判断 ServiceManager 里是否有 user.xposed.system 进行 xposed 检测，然后检测自动脱壳机：  
fart(https://github.com/hanbinglengyue/FART)  
FUPK3(https://github.com/F8LEFT/FUPK3)  
Youpk(https://github.com/Youlor/Youpk)  
检测方法是判断下列类或者方法是否存在：

```
dumpMethodCode
fartthread
fart
android/app/fupk3/Fupk
android/app/fupk3/Global
android/app/fupk3/UpkConfig
android/app/fupk3/FRefInvoke
cn/youlor/Unpacker
```

### 0xA18D4

getInstalledApplications 获取系统中安装的 APP 信息。

### 0xA7D3C

解密出字符串 Java 和 JNI_OnLoad，hook 了几个函数，被 hook 的原地址未知，新地址：0xA43A0/0xA485C/0xA48F4/0xA54B0；hook dlsym(hook 后：0xA4554) 和 dlopen(hook 后：0xA4D30)。

### 0xB4B94

hook libc 中的下列函数：

```
write(hook后：0xAA2CC)
pwrite64(hook后：0xAA51C)
close(hook后：0xAA774)
read64(hook后：0xAAA9C)
openat64(hook后：0xAACB8)
__openat_2(hook后：0xAB6D4)
__open_2(hook后：0xAC0F4)
open64(hook后：0xACB10)
read(hook后：0xAFE18)
mmap64(hook后：0xB1C54)
```

system_property_get debug.atrace.tags.enableflags，hook bionic_trace_begin 和 bionic_trace_end(hook 后：0xA8EF4 和 0xA8EF8，直接返回)，没有找到则 hook g_trace_marker_fd(hook 后：0xA8EFC，返回 - 1)。

### 0xB9BEC

sha1。

### 0xBAE64

md5init。

### 0xC3378

base64。

### 0xC5DDC

base64。

### 0xC7B18

APK 签名相关。

### 0xD024C

sha1。

### 0xD1C04

sha1init。

### 0xD2E98

md5。

### 0xD6484

调用 0xD75A0。

### 0xD5CDC

读 / proc/self/cmdline。

### 0xD6578

hook libdvm.so 中的函数 (hook 后：0xD6988)。

### 0xD68A8

根据 off_12EF10 处的值判断调用 0xD6484 还是 0xD6578。

### 0xD75A0

hook libaoc.so 中的函数 (hook 后：0xD69BC)。

### 0x3EA68

JNI_OnLoad。分析环境 pixel4 android10，动态分析过程中一些没有被调用的函数不再分析。  
1. 初始化 cpuabi 字符串 (arm64) 于 0x12E7C8  
2. 初始化 so 名字符串 (libDexHelper) 于 0x12EC38  
3. 初始化字符串 com/secneo/apkwrapper/H 于 0x137B10  
4. 调用 0x1E520  
5.

```
JNIEnv->FindClass(com/secneo/apkwrapper/H)
JNIEnv->GetStaticFieldID(com/secneo/apkwrapper/H.PKGNAMELjava/lang/String;)
JNIEnv->GetStaticObjectField(class com/secneo/apkwrapper/H, PKGNAME Ljava/lang/String; => "cn.missfresh.application")
JNIEnv->GetStringUtfChars("cn.missfresh.application")
```

6. 将包名存于 0x138040  
7.

```
JNIEnv->FindClass(android/app/ActivityThread)
JNIEnv->GetStaticMethodID(android/app/ActivityThread.currentActivityThread()Landroid/app/ActivityThread;)
JNIEnv->CallStaticObjectMethodV(class android/app/ActivityThread, currentActivityThread())
JNIEnv->GetMethodID(android/app/ActivityThread.getSystemContext()Landroid/app/ContextImpl;)
JNIEnv->CallObjectMethodV(android.app.ActivityThread, getSystemContext())
JNIEnv->FindClass(android/app/ContextImpl)
JNIEnv->GetMethodID(android/app/ContextImpl.getPackageManager()Landroid/content/pm/PackageManager;)
JNIEnv->CallObjectMethodV(android.app.ContextImpl, getPackageManager())
JNIEnv->GetMethodID(android/content/pm/PackageManager.getPackageInfo(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;)
JNIEnv->NewStringUTF("cn.missfresh.application")
JNIEnv->CallObjectMethodV(android.content.pm.PackageManager, getPackageInfo("cn.missfresh.application", 0x0))
JNIEnv->GetFieldID(android/content/pm/PackageInfo.applicationInfo Landroid/content/pm/ApplicationInfo;)
JNIEnv->GetObjectField(android.content.pm.PackageInfo, applicationInfo Landroid/content/pm/ApplicationInfo;)
JNIEnv->GetFieldID(android/content/pm/ApplicationInfo.sourceDir Ljava/lang/String;)
JNIEnv->GetObjectField(android.content.pm.ApplicationInfo, sourceDir Ljava/lang/String; => "/data/app/cn.missfresh.application-1")
JNIEnv->GetStringUtfChars("/data/app/cn.missfresh.application-1")
JNIEnv->GetFieldID(android/content/pm/ApplicationInfo.dataDir Ljava/lang/String;)
JNIEnv->GetObjectField(android.content.pm.ApplicationInfo, dataDir Ljava/lang/String; => "/data/data/cn.missfresh.application")
JNIEnv->GetStringUtfChars("/data/data/cn.missfresh.application")
```

8. 调用 0x218A8  
9.

```
JNIEnv->GetFieldID(android/content/pm/ApplicationInfo.nativeLibraryDir Ljava/lang/String;)
JNIEnv->GetObjectField(android.content.pm.ApplicationInfo@36d64342, nativeLibraryDir Ljava/lang/String; => "/data/app/cn.missfresh.application-1/lib/arm64")
JNIEnv->GetStringUtfChars("/data/app/cn.missfresh.application-1/lib/arm64")
```

10. 读 / proc/pid/fd，匹配包名 + base.apk，0x12EA38 存放指向 base.apk 完整路径的指针的指针  
11.

```
JNIEnv->FindClass(com/secneo/apkwrapper/H)
JNIEnv->GetStaticFieldID(com/secneo/apkwrapper/H.ISMPAASLjava/lang/String;)
JNIEnv->GetStaticObjectField(class com/secneo/apkwrapper/H, ISMPAAS Ljava/lang/String; => "###MPAAS###")
JNIEnv->GetStringUtfChars("###MPAAS###")
```

12. 将得到的结果和 ###MPAAS### 比较，0x12E7F8 指向 0x137D9C，0x137D9C 存放比较结果  
13. 调用 0x22068  
14. 调用 0x23568  
15. 调用 0x1F250  
16. 将字符串 lib/libart.so 存放于 0x1378A8  
17. 读 / proc/self/maps，找权限为 "r-xp" 的 lib/libart.so  
18. 初始化下列字符串:

```
/data/user/0/cn.missfresh.application/.cache
/data/user/0/cn.missfresh.application/.cache/oat/arm64
/data/user/0/cn.missfresh.application/.cache/classes.dve
/data/app/cn.missfresh.application-xxx/oat/arm64/base.odex
```

19.fstat /data/app/cn.missfresh.application-xxx/oat/arm64/base.odex  
20. 计算 md5(不太清楚具体算的什么)，0x12EC98 指向 0x130080，0x130080 存放计算结果  
21.access /data/user/0/cn.missfresh.application/.cache/classes.dve，不存在则把之前算的 md5 写入该文件；存在则读取其中的值和之前算的比较，不相等则写入新计算的值  
22. 调用 0x3371C(根据标记位决定是否调用)  
23. 调用 0x1FDC8  
24. 初始化下列字符串:

```
/data/user/0/cn.missfresh.application/.cache/libDexHelper32
/lib/armeabi-v7a/libDexHelper.so
/lib/armeabi/libDexHelper.so
assets/libDexHelper32
```

25. 调用 0x3766C  
26. 调用 0xB4B94  
27. 调用 0x9C0BC  
28. 调用 0xD5CDC  
29. 调用 0x24C08  
30. 调用 0x1C40C  
31. 调用 0xA7D3C  
32. 结束

总结
--

样本混淆强度还是比较大的，比前两篇文章中的样本要复杂很多。不过分析过程中也是有一些技巧：比如位置相邻的函数之前其实是有联系的，和另外某些壳的代码有类似的地方 (估计也是抄来抄去)，可以网上搜一下旧版本的分析博客，有一些函数名和字符串没有被抹去，等等。

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

最后于 10 小时前 被 houjingyi 编辑 ，原因：

[#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [fix-cfg.py](javascript:void(0)) （24.13kb，8 次下载）
*   [libDexHelper.so](javascript:void(0)) （1.16MB，7 次下载）