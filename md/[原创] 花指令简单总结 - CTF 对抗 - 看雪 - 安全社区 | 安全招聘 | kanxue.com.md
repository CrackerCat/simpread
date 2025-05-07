> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286716.htm)

> [原创] 花指令简单总结

背景

*   线性扫描算法：逐行反汇编（无法将数据和内容进行区分）
*   递归下降算法：根据一条指令是否被另一条指令引用来决定是否对其进行反汇编（难以准确定位）

正是因为这两种反汇编的规格和缺陷机制，所以才导致了会有花指令的诞生

**花指令简单的说就是在代码中混入一些垃圾数据阻碍静态分析**

![](https://bbs.kanxue.com/upload/attach/202505/973236_B25NFZQMY9Q6UNA.jpg)

### 常见指令

*   0xE8 call + 4 字节偏移地址
*   0xE9 jmp + 4 字节偏移地址
*   0xEB jmp + 2 字节偏移地址
*   0xFF15 call + 4 字节地址
*   0xFF25 jmp + 4 字节地址
*   0xCC int 3
*   0xe2 loop
*   0x0f84 jz
*   0x0f85 jnz

指令也不一定唯一，比如上面有两种表示 call 的方式，0x74 也能表示 jz

常规
==

### 1. 简单 jmp

OD 能被骗过去，但是 ida 主要采用的是递归扫描的办法（会用线性扫描补充），所以能够正常识别

![](https://bbs.kanxue.com/upload/attach/202505/973236_APGJSF4TB965ZEQ.jpg)

### 2.jx+jnx（x 可为 e,z,l）

![](https://bbs.kanxue.com/upload/attach/202505/973236_GYPCV3Q64YYUCQ9.jpg)

jnz 实际上是 fake 的，因为 jz 这个指令，让 ida 认为 jz 下面的是另外一个分支，所以这里将 jz 下面包括 jz 全转化为代码

call 指令按 u，下一行按 c，再 nop call，把 90 转为数据，再按 c 变为 nop

![](https://bbs.kanxue.com/upload/attach/202505/973236_E3XEYEA6EYS6DFB.jpg)

### 3.call +add esp，4 或 call + add [esp], n + retn

![](https://bbs.kanxue.com/upload/attach/202505/973236_7P6SUHG7DRWYSDN.jpg)

![](https://bbs.kanxue.com/upload/attach/202505/973236_TJVGJ6BNRSPHK4A.jpg)

#### 2023 凌武杯 flower_tea

![](https://bbs.kanxue.com/upload/tmp/973236_ER2V7BWY7XPWSD3.jpg)

这里 call 指令，其实本质就是 jmp&push 下一条指令的地址，但是这里只需要 jmp 指令，push 这条指令是多余的，后续的 add 指令又会修改下一条指令的地址，造成爆红

易语言自带的花指令，与上面的本质相同

```
004010BF   .  E8 00000000   call 1111.004010C4
004010C4  /$  830424 06     add dword ptr ss:[esp],0x6
004010C8  \.  C3            retn
004010C9      B9            db B9
```

只需要将下面的特征码 patch 掉就可以了

```
E80000000083042406C3??
```

### 4.jmp XXX（红色）

题目练习：[https://www.nssctf.cn/note/set/2970](elink@630K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2F1M7%4y4U0N6r3k6Q4x3X3g2U0L8W2)9J5c8X3&6G2N6r3g2Q4x3V1k6K6k6i4c8Q4x3V1j5J5z5e0M7H3)

![](https://bbs.kanxue.com/upload/attach/202505/973236_A2SQCDGYE5HWFY7.jpg)

这也是一种常见的花指令，虚拟地址不可能那么大，实际是 e9 在搞鬼，ida 会默认将 e9 后面的 4 个字节当成地址，只要 nop 掉 jmp(E9) 就好了

### 5.stx/jx

![](https://bbs.kanxue.com/upload/attach/202505/973236_T7E5G9CU8MEQHQY.jpg)

clc 是清除 EFlags 寄存器的 carry 位的标志，而 jnb 是根据 cf==0 时跳转的，然而 jnb 这个分支指令，ida 又将后面的部分认作成了另外的分支。

### 6. 汇编指令共用 opcode

加了一些无效指令导致 ida 等反编译工具识别错误

![](https://bbs.kanxue.com/upload/tmp/973236_N839AJDJWVDVXWZ.jpg)

创意
==

### 1. 替换 ret 指令

call 指令的本质：push 函数返回地址然后 jmp 函数地址

ret 指令的本质： pop eip

所以在 call 指令之后，函数返回地址存放于 esp，可以将值取出，用跳转指令跳转到该地址，即可代替 ret 指令

### 2. 控制标志寄存器跳转

这一部分需要精通标志寄存器，每一个操作码都会对相应的标志寄存器产生相应的影响，如果对标志寄存器足够熟练，就可以使用对应的跳转指令构造永恒跳转

### 3. 利用函数返回确定值

有些函数返回值是确定的，比如自己写的函数，返回值可以是任意非零整数；如果故意传入一个不存在的模块名称，那么就会返回一个确定的值 NULL；另一方面，某些 api 函数，一定要调用成功的，而这些 api 函数基本上只要调用成功就就会返回一个确定的零或者非零值，如 MessageBox。这些都可以构造永恒跳转

### 4. 针对反编译

0x1165 开始的花指令和前面的花指令原来相似，这条花指令会使 IDA 误以为 0x116B 处的指令可能会执行，导致 IDA 的栈分析出现错误

![](https://bbs.kanxue.com/upload/attach/202505/973236_ZFTWY96QPP2H2UZ.jpg)

修复方法除了 patch 外还有修改 ida 对栈的分析结果

在 Options - General 菜单中勾上 Stack pointer 选项可以查看每行指令执行之前的栈帧大小

![](https://bbs.kanxue.com/upload/attach/202505/973236_9MQNCG8UW5S7VJF.jpg)

Alt + K 可以修改某条指令对栈指针的影响，从而消除这条花指令对反编译的影响

![](https://bbs.kanxue.com/upload/attach/202505/973236_NGMVKRRJABCQV5M.jpg)

清除
==

### nop 单字节 (E8/E9)

练习题目：[https://www.nssctf.cn/problem/2313](elink@bc8K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2F1M7%4y4U0N6r3k6Q4x3X3g2U0L8W2)9J5c8Y4m8J5L8$3u0D9k6h3#2Q4x3V1j5J5x3K6p5K6)

![](https://bbs.kanxue.com/upload/attach/202505/973236_9HGJWD3TYHKDMR9.jpg)

在 0x401051 设置为数据类型（快捷键 D），将 call 转成硬编码 E8  再将光标放到 db 0E8 上 将 E8 改成 nop(90) 后按 C（转化为代码类型）点 yes 将硬编码修复成代码

然后向下逐⼀修复 将光标放置在黄色的行上 按 C 修复 直到没有黄色地址  

![](https://bbs.kanxue.com/upload/attach/202505/973236_7WGX6B24VKBC28R.jpg)

最后全选函数，按 P 生成函数  

### nop 跨越汇编指令

jz 指令指向下一条指令中间

![](https://bbs.kanxue.com/upload/attach/202505/973236_5T2X74Y2R7XU8CY.jpg)

这个时候让 jz 正常分析，也就是把中间的 nop

![](https://bbs.kanxue.com/upload/attach/202505/973236_TEJDU9NFH3RUUHA.jpg)

如果后面有数据没被分析为 code，需要重新分析一下

### nop 部分汇编

一般去菜单中的编辑选项修补单字节会比较好把控

像下面这种红色标志离原函数有一定距离又是 call+retn 组合，400f64 又没什么用，还有 ret 会干扰函数的分析，那就都 nop，这是一种暴力破解方法

![](https://bbs.kanxue.com/upload/attach/202505/973236_WGXFH5JYQNEKHJA.jpg)

nop 完后看到有 %lld, 删除函数，修补函数即可反编译

![](https://bbs.kanxue.com/upload/attach/202505/973236_DACZXTNXRTUHEZ2.jpg)

![](https://bbs.kanxue.com/upload/attach/202505/973236_U9TC928EKTCW3TU.jpg)

xchg 很少用到，后面还有 retn，主打不想要的直接全部 nop

![](https://bbs.kanxue.com/upload/attach/202505/973236_E6XYT3AMEXVTCQG.jpg)

一开始对于这一段汇编不知道怎么处理，后面看到一堆和 rax 有关的操作，从 push rax 开始异或，刚好下面有 pop rax，这是 ida 对 call retn 指令识别漏洞的花指令，ida 错误解析 call 后的指令边界，导致后续代码显示为无效指令造成的

![](https://bbs.kanxue.com/upload/attach/202505/973236_HW95BVZBE73CJEF.jpg)

先可以小范围的尝试，把 call 到红色的地方先 nop，发现不行，再从头把关键数据之前（D7,flag is 上面那一行) 也给 nop 掉，发现可以了

![](https://bbs.kanxue.com/upload/attach/202505/973236_EVUYXSDKRWYP89W.jpg)

### 代码自动去花

#### nop

```
#include <idc.idc>
static main(){
auto addr_start =0x00415990;
auto addr_end = 0x00416048;
auto i=0,j=0;
    for(i=addr_start;i<addr_end;i++){
          if(Dword(i) == 0x1E8){
              for(j=0 ; j<6; j++,i++ ){
                  PatchByte(i,0x90);
              }
              i=i+4;
              for(j=0 ; j<3; j++,i++ ){
                  PatchByte(i,0x90);
              }
              i=i+10;
              for(j=0 ; j<3; j++,i++ ){
                  PatchByte(i,0x90);
              }
              i=i+5;
              for(j=0 ; j<1; j++,i++ ){
                  PatchByte(i,0x90);
              }  
              i=i+3;
              for(j=0 ; j<2; j++,i++ ){
                  PatchByte(i,0x90);
              }    
              i--;    
          }
    }
}
```

1.  定义了一个名为 f 的函数，用于执行二进制搜索并对匹配的位置进行 nop，hexStr 是用于搜索的十六进制模式字符串（用空格分开，其中?? 表示通配符，注意不可以使用诸如 2? 这样的情况）
2.  接着，将 hexStr 转换为两个字节数组：bMask 和 bPattern。bMask 是用于表示可变字节的，其中 00 表示需要匹配的字节，01 表示不需要匹配的字节；bPattern 则是用于搜索的固定字节序列
3.  定义了一个 signs 变量，用于指定搜索的标志位，其中 BIN_SEARCH_FORWARD 表示向前搜索，BIN_SEARCH_NOBREAK 表示不允许搜索中途中断，BIN_SEARCH_NOSHOW 表示不显示搜索结果
4.  接着使用 ida_bytes.bin_search 函数进行二进制搜索，从 begin_addr 到 end_addr 之间搜索 bPattern，其中可变字节由 bMask 指定。搜索到匹配的位置后，将其打印出来，并调用 patch 函数对其进行补丁操作
5.  最后更新 begin_addr 为下一个搜索的起始地址，通常为当前匹配位置加上一个偏移量

```
import ida_bytes
import ida_ida
def patch(ea,num=1):
    for i in range(num):
        ida_bytes.patch_byte(ea+i,0x90)
    return
def f(begin_addr,end_addr,hexStr)
  xx=(len(hexStr)-1)//2
    bMask = bytes.fromhex(hexStr.replace('00', '01').replace('??', '00'))
    bPattern = bytes.fromhex(hexStr.replace('??', '00'))
    signs=ida_bytes.BIN_SEARCH_FORWARD| ida_bytes.BIN_SEARCH_NOBREAK| ida_bytes.BIN_SEARCH_NOSHOW
    while begin_addr<end_addr:
        ea=ida_bytes.bin_search(begin_addr,end_addr,bPattern,bMask,1,signs)
        if ea == ida_idaapi.BADADDR:
            break
        else: 
            print(hex(ea))
            patch(ea,xx)
            begin_addr=ea+xx 
f(0x0,0x1000,"?? ?? 00 00 00 ??")
```

#### jx+jnx

![](https://bbs.kanxue.com/upload/attach/202505/973236_6GRXEBJUFMTVFFZ.jpg)

```
from ida_bytes import get_bytes,patch_bytes
start= 0x401000
end = 0x422000
buf = get_bytes(start,end-start)

def patch_at(p,ln):
    global buf
    buf = buf[:p]+b"\x90"*ln+buf[p+ln:]

fake_jcc=[]
for opcode in range(0x70,0x7f,2):
    pattern = chr(opcode)+"\x03"+chr(opcode|1)+"\x01"
    fake_jcc.append(pattern.encode())
    pattern = chr(opcode|1)+"\x03"+chr(opcode)+"\x01"
    fake_jcc.append(pattern.encode())

print(fake_jcc)
for pattern in fake_jcc:
    p = buf.find(pattern)
    while p != -1:
        patch_at(p,5)
        p = buf.find(pattern,p+1)

patch_bytes(start,buf)
print("Done")
```

#### 清除常见花指令

```
import idc
import ida_bytes
import keystone
import capstone

def set_x86():
    global ks, cs
    ks = keystone.Ks(keystone.KS_ARCH_X86, keystone.KS_MODE_32)
    cs = capstone.Cs(capstone.CS_ARCH_X86, capstone.CS_MODE_32)

set_x86()

def asm(code, addr=0):
    return bytes(ks.asm(code, addr)[0])

def disasm(code, addr=0):
    for i in cs.disasm(code, addr):
        return ('%s %s' %(i.mnemonic, i.op_str))

def MakeCode(ea):
    return idc.create_insn(ea)

def disasm_at(addr):
    size = MakeCode(addr)
    code = idc.get_bytes(addr, size)
    return addr + size, disasm(code, addr)

def disasm_block(start, end):
    codes = []
    addr = start
    while addr < end:
        _addr, code = disasm_at(addr)
        codes.append((addr, code))
        addr = _addr
    return codes


addr = 0x4010b0
end = 0x402e41
while addr < end:
    ida_bytes.del_items(addr, 0, 10)
    size = MakeCode(addr)
    assert size, hex(addr)
    if size == 1:
        if ida_bytes.get_bytes(addr, 3) == b'\xf9\x72\x01': # stc; jb $+3
            ida_bytes.patch_bytes(addr, b'\x90' * 4)
        elif ida_bytes.get_bytes(addr, 3) == b'\xf8\x73\x01': # clc; jnb $+3
            ida_bytes.patch_bytes(addr, b'\x90' * 4)
        else:
            addr += size
    elif size == 2 and ida_bytes.get_bytes(addr, 2) == b'\xeb\x01': # jmp $+3
        ida_bytes.patch_bytes(addr, b'\x90' * 3)
    elif size == 5:
        if ida_bytes.get_bytes(addr, 10) == b'\xe8\x00\x00\x00\x00\x83\x04\x24\x06\xc3': # call $+5; add dword ptr [esp], 6; ret
            ida_bytes.patch_bytes(addr, b'\x90' * 11)
        else:
            addr += size
    else:
        addr += size

print('Done.')
```

#### jmp db1

![](https://bbs.kanxue.com/upload/attach/202505/973236_8QBXCZE6VQ94JXA.jpg)

分析 main() 函数可以看到是杂乱的字节，观察 0x1144 可以发现，存在着 jmp db1 这种类型的花指令，因此可以写 idapython 脚本来解决

```
import ida_bytes
import ida_ida
def patch(ea,num=1):
  for i in range(num):
    ida_bytes.patch_byte(ea+i,0x90)
  return
print("-----")
hexStr="EB FF C0 BF ?? 00 00 00 E8"
bMask = bytes.fromhex(hexStr.replace('00', '01').replace('??', '00'))
bPattern = bytes.fromhex(hexStr.replace('??', '00'))
signs=ida_bytes.BIN_SEARCH_FORWARD| ida_bytes.BIN_SEARCH_NOBREAK| ida_bytes.BIN_SEARCH_NOSHOW
print(bMask,bPattern)
begin_addr=0x1135
end_addr=0x3100
while begin_addr<end_addr:
  ea=ida_bytes.bin_search(begin_addr,end_addr,bPattern,bMask,1,signs)
  if ea == ida_idaapi.BADADDR:
    break
  else: 
    print(hex(ea))
    patch(ea,3)
    begin_addr=ea+8
```

### pyc 去花指令

`脚本读取机器码+010删掉花指令+修改co_code长度`

花指令通常开头是 JUMP_ABSOLUTE X，然后填充错误代码

```
import marshal, dis
f = open("D:\\new\\AD\\game\\vnctf2022\\re\\BabyMaze.pyc", "rb").read()
code = marshal.loads(f[16:]) #这边从16位开始取因为是python3 python2从8位开始取
dis.dis(code)
dis.dis(code)
print(len(code.co_code))
```

![](https://bbs.kanxue.com/upload/attach/202505/973236_36XKX4SY9HEGZ7M.jpg)

转为十六进制后去 010editor 修改，去掉即可，这里 JUMP_ABSOLUTE16 进制为 71

![](https://bbs.kanxue.com/upload/attach/202505/973236_FEJG9HFYFFYQUAS.jpg)

![](https://bbs.kanxue.com/upload/attach/202505/973236_2QXY9HVKMNJY4PV.jpg)

![](https://bbs.kanxue.com/upload/attach/202505/973236_65G2M456MX6J2XY.jpg)

![](https://bbs.kanxue.com/upload/attach/202505/973236_BUS7Q7YVJMFAHMR.jpg)

之后重新反编译即可

参考链接：

[https://blog.csdn.net/m0_51246873/article/details/127167749](elink@cf3K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6T1L8r3!0Y4i4K6u0W2j5%4y4V1L8W2)9J5k6h3&6W2N6q4)9J5c8X3@1H3i4K6g2X3y4e0p5J5y4o6j5^5y4K6y4Q4x3V1k6S2M7Y4c8A6j5$3I4W2i4K6u0r3k6r3g2@1j5h3W2D9M7#2)9J5c8U0p5J5y4K6p5$3y4K6M7@1z5b7`.`.)

[https://www.cnblogs.com/YenKoc/p/14136012.html](elink@294K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2U0L8X3u0D9L8$3N6K6i4K6u0W2j5$3!0E0i4K6u0r3h3h3g2F1d9$3!0U0i4K6u0r3M7q4)9J5c8U0p5@1x3e0x3$3x3o6p5J5i4K6u0W2K9s2c8E0L8l9`.`.)

[https://mp.weixin.qq.com/s/MUth1Qw-Fl2a5OrLw_2_0g](elink@036K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0M7q4)9J5k6i4N6W2K9i4S2A6L8W2)9J5k6i4q4I4i4K6u0W2j5$3!0E0i4K6u0r3M7#2)9J5c8V1#2g2N6r3R3I4f1i4N6Q4x3X3c8r3L8o6u0S2y4f1!0J5e0s2N6Q4y4h3j5J5i4K6g2X3x3r3M7`.)

[https://blog.csdn.net/abel_big_xu/article/details/117927674](elink@f08K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6T1L8r3!0Y4i4K6u0W2j5%4y4V1L8W2)9J5k6h3&6W2N6q4)9J5c8X3q4T1k6h3I4Q4y4h3k6T1K9h3N6Q4y4h3k6^5N6g2)9J5c8X3q4J5N6r3W2U0L8r3g2Q4x3V1k6V1k6i4c8S2K9h3I4K6i4K6u0r3x3e0p5%4z5e0t1%4y4U0M7@1)

出题参考：

[https://blog.csdn.net/m0_46296905/article/details/117336574](elink@364K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6T1L8r3!0Y4i4K6u0W2j5%4y4V1L8W2)9J5k6h3&6W2N6q4)9J5c8X3@1H3i4K6g2X3y4o6j5J5z5e0j5&6x3o6g2Q4x3V1k6S2M7Y4c8A6j5$3I4W2i4K6u0r3k6r3g2@1j5h3W2D9M7#2)9J5c8U0p5I4y4K6x3K6y4U0f1%4y4l9`.`.)

[https://bbs.kanxue.com/thread-279604.htm](https://bbs.kanxue.com/thread-279604.htm)

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#Reverse](forum-37-1-111.htm)