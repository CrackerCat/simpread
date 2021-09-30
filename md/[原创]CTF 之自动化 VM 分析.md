> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269591.htm)

> [原创]CTF 之自动化 VM 分析

CTF 之自动化 VM
===========

案例：2021 长安杯 - SimpleVM  
一道当时比赛零解的题  
![](https://bbs.pediy.com/upload/attach/202109/921830_5ZXHMX2B8YFYFPN.png)  
![](https://bbs.pediy.com/upload/attach/202109/921830_M7ZTYPQZ3FR3JZH.png)  
题目文件以及脚本文件见附件

[](#一：程序分析)一：程序分析
=================

拖入 ida 知道输入长度为 8

 

输入 flag{11223344} 进行调试  
发现将 flag 的内容 11223344 分开存放到两个四字节的变量中

 

然后将其移动到了内存中的某个位置（vm 结构体中的寄存器）

 

下图是加载 vm 的 bytecode

 

![](https://bbs.pediy.com/upload/attach/202109/921830_ZE8VYZGPF3UZP38.png)

 

将 flag 的内容移动到 vm 结构体中  
![](https://bbs.pediy.com/upload/attach/202109/921830_F8HJFX3ANNWBFBW.png)

 

然后下面一个函数就是 dispatcher，典型的用 while 循环读取的字节码进行分发执行不同的虚拟指令

 

再下方就是检测两个寄存器是否是 0，为 0 才 right  
![](https://bbs.pediy.com/upload/attach/202109/921830_WAZJCBHVKB7CEUZ.png)

[](#二：vm分析)二：VM 分析
==================

分析 VM，先找到它的 bytecode，我们可以在内存中找到

 

![](https://bbs.pediy.com/upload/attach/202109/921830_34MVJ765HPUTT2W.png)

 

然后找 dispatcher

 

![](https://bbs.pediy.com/upload/attach/202109/921830_5HBU7QDFZ94ZM9V.png)

 

然后就是分析如何模拟的指令，以及根据这些模拟的指令来分析 vm 结构体

 

我恢复出来的 vm 结构体：

 

根据内存进行分析得到的

```
struct vm
{
    qword rip;
    qword* opcode;
    qword opcode_length;
    dword regs0;
    dword regs1;
        dword regs2;
        dword regs3;
        .....
}

```

然后就是分析虚拟机的模拟指令

### 1. 每次循环获取 opcode

get_rip_value_ripad1 函数：获取 rip 地址对应的字节码，然后将 rip+1

 

![](https://bbs.pediy.com/upload/attach/202109/921830_7KNS3EC24CXPNHH.png)

### 2. mov regs[num1], regs[num2]

![](https://bbs.pediy.com/upload/attach/202109/921830_546TRD89YHC39FV.png)

 

space 首地址偏移 24 字节开始是存放的寄存器

 

所以这条指令解析为：

 

mov regs[操作数 1]， regs[操作数 2]

### 3. add regs[num1], regs[num2]

![](https://bbs.pediy.com/upload/attach/202109/921830_TC665YENTUKWZG5.png)  
同上，不过这里是 +=

 

所以这条指令解析为：

 

add regs[操作数 1]，regs[操作数 2]

### 4. shl regs[num1], regs[num2]

![](https://bbs.pediy.com/upload/attach/202109/921830_R8569EDRN3A4E35.png)

 

同上，解析为：

 

shl regs[操作数 1], regs[操作数 2]

### 5. shr regs[num1], regs[num2]

![](https://bbs.pediy.com/upload/attach/202109/921830_PEFNAAUM85ZYD7N.png)

 

同上，解析为：

 

shr regs[num1], regs[num2]

### 6. regs[num1] %= regs[num2]

![](https://bbs.pediy.com/upload/attach/202109/921830_PG5SSQ74SKQPJDZ.png)

 

同上，解析为：

 

regs[num1] %= regs[num2]

### 7. xor regs[num1]， regs[num2]

![](https://bbs.pediy.com/upload/attach/202109/921830_2BBCZ2XZ5Z38MYQ.png)

### 8. mov regs[num1], [regs[num2]+0x86]

将内存中的值赋值给寄存器

 

![](https://bbs.pediy.com/upload/attach/202109/921830_TV3FKCJF3EQS9JM.png)

### 9. mov [regs[num1]+0x86], regs[num2]

![](https://bbs.pediy.com/upload/attach/202109/921830_P9RP5Z8D9YBWRTP.png)

### 10. cmp regs[num1], regs[num2]

比较两个寄存器并设置标志位

 

![](https://bbs.pediy.com/upload/attach/202109/921830_TQ9TBWEYFP6C3J6.png)

### 11. nop

![](https://bbs.pediy.com/upload/attach/202109/921830_SBQVU8QJ5JGWWJ6.png)  
![](https://bbs.pediy.com/upload/attach/202109/921830_HN2ARHEGEWVT69W.png)

 

其实这里是设计的一个简单的花指令，获取操作数之后

 

rip += 操作数

 

但是后来发现，基本上都是 + 1，显然是插了一个字节

 

直接 nop

### 12. 跳转

![](https://bbs.pediy.com/upload/attach/202109/921830_4SDF7T7CWYSQGFY.png)

[](#三：编写反汇编器)三：编写反汇编器
=====================

```
class Decompile:
    def __init__(self, file_name, opcode_keys_, code):
        self.filename = file_name   # 要逆向的文件路径
        self.asm_code = []          # 反汇编的指令列表
        self.pc = 0                 # 虚拟机的rip
        self.code = code            # 存放文件的虚拟code
        self.opcode_keys = opcode_keys_  # 存放文件的指令集代表的opcode
        self.recompile_to_c = ""    # 二维数组来存放每个文件反编译成c的代码
        self.key_table = []         # 二维数组来存放文件的加密用的key_table
        self.enc_number = 0         # 根据key来判断加密的次数
        self.a0 = 0                 # 加密之后的数据(后面得到key_table之后会设置)
        self.a1 = 0
        self.i = 0                 # 解密的递归循环变量,用来判断是否退出递归
        self.string = ''           # 解密之后的字符串
    def disamone(self):          # 反编译为伪汇编指令
        regs = ['eax', 'ebx', 'ecx', 'edx', 'esi', 'edi', 'reg6', 'reg7', 'reg8', 'reg9', 'reg10', 'reg11', 'reg12', 'reg12', 'reg14', 'reg15', 'reg16', 'reg17', 'reg18', 'reg19', 'eax', 'ebx', 'reg22', 'reg23', 'reg24', 'reg25', 'reg26', 'reg27', 'reg28', 'reg29', 'ecx', 'edx', 'reg32']
        opcode = self.get_opcode()
        offset = len(self.code)
        if opcode == self.opcode_keys[11]:
            pc = self.pc
            num1,num2 = self.rip_ad_six()
            self.asm_code.append("_%s: mov %s, %s" % (hex(offset+pc), regs[num1], num2))
        elif opcode == self.opcode_keys[0]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov %s, %s" % (hex(offset+pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[1]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: add %s, %s" % (hex(offset+pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[2]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: shl ecx, al" % (hex(offset + pc)))        # regs[num2] % (hex(offset + pc), regs[num1])
        elif opcode == self.opcode_keys[3]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: shr edx, bl" % (hex(offset + pc)))       # , regs[num2] % (hex(offset + pc), regs[num1])
        elif opcode == self.opcode_keys[4]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov eax, %s\ncdq\nidiv %s\nmov %s, edx" % (hex(offset + pc), regs[num1], regs[num2], regs[num1]))
        elif opcode == self.opcode_keys[7]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: xor %s, %s" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[5]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov %s, [%s+0x86]" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[6]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov [%s+0x86], %s" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[8]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: cmp %s, %s" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[9]:
            pc = self.pc
            num1 = self.rip_ad_five()
            self.asm_code.append("_%s: nop" % (hex(offset + pc)))
        elif opcode == self.opcode_keys[10]:
            pc = self.pc
            num1, val = self.used_by_jcc()
            jcc = ['jmp', 'jl', 'jg']
            temp = ''
            if val == 0:
                temp = jcc[0]
            elif val == 1:
                temp = jcc[1]
            elif val == 2:
                temp = jcc[2]
            self.asm_code.append("_%s: %s %s" % (hex(offset + pc), temp, num1))
        else:
            print("undefined instruction")
        def get_asm_code(self):         # 获取伪汇编代码
        while self.pc < len(self.code):
            self.disamone()
        print("\n".join(self.asm_code))

```

得到伪汇编指令为：

```
_0x37f8: mov eax, 4
_0x37fe: mov ebx, 5
_0x3804: mov ecx, ebx
_0x3807: mov edx, ebx
_0x380a: shl ecx, al
_0x380d: nop
_0x3813: shr edx, bl
_0x3816: mov esi, ecx
_0x3819: nop
_0x381f: xor esi, edx
_0x3822: add esi, ebx
_0x3825: mov edi, 64
_0x382b: xor esi, edi
_0x382e: add eax, esi
_0x3831: mov ecx, eax
_0x3834: nop
_0x383a: mov edx, eax
_0x383d: nop
_0x3843: shl ecx, al
_0x3846: shr edx, bl
_0x3849: nop
_0x384f: mov esi, ecx
_0x3852: xor esi, edx
.........

```

[asm.txt](CTF%E4%B9%8B%E8%87%AA%E5%8A%A8%E5%8C%96VM%209abcda060c034c20b6d94cf7263d7a75/asm.txt)

[](#四：优化汇编利用pwntools的asm直接将指令转换成机器码)四：优化汇编利用 pwntools 的 asm 直接将指令转换成机器码
=======================================================================

缺点：对汇编的格式要求很高，必须要严格满足对应架构

 

有些指令在写反汇编器的时候很难实现，比如乘，除等

 

对于这道题，想了一下，并没有涉及到操作栈的指令，几乎全部是用寄存器来操作的

 

所以，直接重编译成 C

[](#五：重编译成c直接导入编译器编译)五：重编译成 C 直接导入编译器编译
=======================================

寄存器直接声明成变量即可

 

初始化工作：

 

用变量代替寄存器，用数组代替虚拟机内存，声明变量看作 eflag

 

![](https://bbs.pediy.com/upload/attach/202109/921830_9ZVNG4ETV2YB7WN.png)

 

然后编写重编译器

```
def disamtoc(self):                # 根据分析反编为C代码
        regs = ['a0', 'a1', 'a2', 'a3', 'a4', 'a5', 'a6', 'a7', 'a8', 'a9', 'a10', 'a11', 'a12', 'a13', 'a14', 'a15', 'a16', 'a17', 'a18', 'a19', 'a20', 'a21', 'a22', 'a23', 'a24', 'a25', 'a26', 'a27', 'a28', 'a29', 'a30', 'a31', 'a32']
        opcode = self.get_opcode()
        if opcode == self.opcode_keys[11]:
            pc = self.pc
            num1, num2 = self.rip_ad_six()
            # self.recompile_to_c += "_{}: {}={};\n".format(hex(pc), regs[num1], num2)
            self.recompile_to_c += "{}={};\n".format(regs[num1], num2)
        elif opcode == self.opcode_keys[0]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}={};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[1]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}+{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}+={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[2]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}<<{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}<<={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[3]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}>>{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}>>={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[4]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {} % {};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{} %= {};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[7]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}^{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{} ^= {};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[5]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}=arr[{}+0x86];\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}=arr[{}+0x86];\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[6]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: arr[{}+0x86]={};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "arr[{}+0x86]={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[8]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.recompile_to_c += "eflag={}-{};\n".format(regs[num1], regs[num2])
            # self.recompile_to_c += "_{}: eflag={}-{};\n".format(hex(pc), regs[num1], regs[num2])
        elif opcode == self.opcode_keys[9]:
            pc = self.pc
            num1 = self.rip_ad_five()
            # self.recompile_to_c += "_{}: goto _{}".format(hex(pc), hex(pc+1))
            self.recompile_to_c += "\n"
        elif opcode == self.opcode_keys[10]:
            pc = self.pc
            num1, val = self.used_by_jcc()
            jcc = ['jmp', 'jl', 'jg']
            temp = ''
            if val == 0:
                temp = jcc[0]
            elif val == 1:
                temp = jcc[1]
            elif val == 2:
                temp = jcc[2]
            self.recompile_to_c += "\n"
            # self.recompile_to_c += "_{}: val={};\nif(val == 0):goto _{};\nif(val==1 && eflag<0):goto _{}\nif(val==2 && eflag>0):goto _{}\n".format(hex(pc), val, hex(pc+num1), hex(pc+num1), hex(pc+num1))
        else:
            print("undefined instruction")

```

```
a20=0x00000004;
a21=0x00000005;
a2=a1;
a3=a1;
a2<<=a20;
 
a3>>=a21;
a4=a2;
 
a4^= a3;
a4+=a1;
a5=0x7b7cc140;
a4^= a5;
a0+=a4;
a2=a0;
 
a3=a0;
 
a2<<=a20;
a3>>=a21;
 
a4=a2;
a4^= a3;
a4+=a0;
a5=0x2105b8c9;
a4^= a5;
a1+=a4;
a20=0x00000004;
a21=0x00000005;
 
a2=a1;
a3=a1;
a2<<=a20;
a3>>=a21;
a4=a2;
 
a4^= a3;
 
a4+=a1;
 
a5=0xfea4fb66;
a4^= a5;
 
a0+=a4;
a2=a0;
a3=a0;
a2<<=a20;
 
a3>>=a21;
a4=a2;
a4^= a3;
 
a4+=a0;
a5=0x7d1ae6c0;
a4^= a5;
a1+=a4;
a20=0x00000004;
 
a21=0x00000005;
a2=a1;
a3=a1;
a2<<=a20;
a3>>=a21;
 
a4=a2;
a4^= a3;
 
a4+=a1;
a5=0x2775177e;
a4^= a5;
a0+=a4;
 
......

```

然后复制到编译器中直接进行编译

 

得到程序的逻辑

 

编译后将 exe 拖入 ida

 

![](https://bbs.pediy.com/upload/attach/202109/921830_W8SKKZQYEQQEWFY.png)

 

可以得到程序的逻辑

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  _main(argc, argv, envp);
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x7B7CC140;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x2105B8C9;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xFEA4FB66;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7D1AE6C0;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x2775177E;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x925FBD3;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD987DA09;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xFAD38F20;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xFD4AEBC6;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4C78602E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x9C43184A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xD65BC5BB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x3EF2C893;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1B4F0892;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x9CC2356;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7F0EF800;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x6D84E09A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x59CE1C06;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x106AD753;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x879B9FFA;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x6A7D6C68;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x6A458E99;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x6A39546D;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xD3EDBF64;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x78DD036B;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xDDAE7767;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xAAA50E5C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x70AEAEB2;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x3102AE18;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x3CECC69A;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xA1583D2E;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xCEF3DC6D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x3594C51;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xABDC6667;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x456C5FF2;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x6CB61C56;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x3A7F577A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x57368E77;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x9F922D9B;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x12ECF301;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x25869D78;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x5D870F1A;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xB88B07B2;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x8F8DBC4;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD255375;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7ACF76CF;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xBB94A688;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1E6BF741;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD5623A3A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x44EBD4B4;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xBBB704D2;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1A3FE1D8;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xE6D571EB;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xF798AB91;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xEE52C1DE;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x550DED4D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x4A7E0D53;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x6F4F1020;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x9DD3B13E;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4C29CAB4;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x16ADA2C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x581B7FEB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xAD7DBBDB;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x50F5A3BE;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xBEE0CF2C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xA69D9701;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x333F88BB;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x6A4C98FE;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x56949F54;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xE699052A;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x3B1CF719;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xE891D313;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xE0FB59;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xF4BA724D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x3145BA6C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1717D7E1;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x3D612168;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4FE65709;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xEA376BDB;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB7039F36;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xAB8C9607;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xE932132E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x6DF54EC1;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xCAC96DBD;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xA7FBC65A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7B230A96;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x823EA159;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x6D31027D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xE1D299A8;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xC8814791;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x78F200FC;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x8A561714;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x638F3CA3;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x3C6AFE1E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xFED9B68D;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x9CE3799E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD0B232C8;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4CD3EF18;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xA644B0FF;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x2957FEAD;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x48A3CF3B;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x5C61789F;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x5868362B;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x2135F32E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xDB14088C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xF2AA0A21;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x4FEB0573;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1B0AA45C;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x7A91AB42;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x78BCF4C8;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD3A6CC4E;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xEA44D817;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x767F67C3;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xC76E48E3;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x8CC053ED;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4C2315F4;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x281C994F;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x11077D71;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x18BBA510;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1B5530B5;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x93828D94;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x5F95EEB0;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x2D913F0F;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB76F9152;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xC274D224;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xDA74FED7;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xF448F931;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xEA75D9C9;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x57FFA5B4;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4F3B4B3B;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x871B3FC8;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7725889;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x29C80A0C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x78E70B0C;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x1B57E9B7;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB81BC454;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xE93935C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xC28170AD;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x74A1E673;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x3A6EA904;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x98814E3C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xCF675AFE;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xF4FE401;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xCFD2CE19;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x97A0617;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x70D577BE;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x30458D1C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x59607271;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xBD30141F;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x37DFBDD9;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x81C48D18;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xABB4F09E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xEBCB14EC;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x744CD637;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x42A10E5A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7D750737;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xE705255;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x884CB2AB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xCA8E63B7;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xBC827AC1;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xFC556F35;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x38137C4C;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x59D7E723;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB4AB0854;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x67A2E134;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xF0462B20;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xDFD77342;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x5DCDAED0;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xC25DD48F;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x76ABE9EE;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x5F28D944;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB9AEC4AB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x7EC18FB1;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xAAF4E86D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x12FE6BAA;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB0EF7F44;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x1174ACFD;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x3780050C;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xB68AF075;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x83A7DC63;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xB56FF91C;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4958B5FD;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x5A51B2CE;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x359AC868;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x5A671B0E;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4D2E377D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x7DA8EDFC;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7C46A290;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xBDF48D46;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xD8A6E9F7;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xA7EB8AEF;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x47E4EDFB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x86EFCCF8;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x61ECDAE8;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x682795B1;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x365BB07E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x62731532;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1A04803D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD5A5E583;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xE9B10F05;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xDA683933;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1BB5181;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x240D3EA4;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x492AB8FB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x4CAFA828;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xCFB38FD9;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x364D6FA2;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xA4367616;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x1244F9C7;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB502CACA;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xCD123E96;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x5CC4BB9A;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD3BFF3E3;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x5D0069CB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x1D84208A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xBEA838D2;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xD7D9FC3;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x88339DAA;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x18435C21;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x8A51F2C8;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xC0BB0D11;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x3E9E2B5D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xBD00703F;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB614A05C;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x9D376AA8;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x714906EB;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x622C818;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4EDD979E;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xA01C1DDE;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7B98B9AA;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x6E70EE28;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x5D982AF0;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x4D937C7A;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x17DE064C;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x8C843EFF;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x4F3A8634;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xB35FC006;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xE97C7ED9;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x997CB12B;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB0999D5D;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xA6BC686;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xF2AB81F5;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xBE0D1038;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xC7B6F558;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x572933A1;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x553EEA26;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x564AAE33;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x50D5CFFA;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xA2CBAF92;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xF702C7A4;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xB2153A9F;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x9D5EC87;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x570E2F95;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x1979F9E8;
  a20 = 4;
  a21 = 5;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xC9F97045;
  a2 = 16 * a0;
  a3 = a0 >> 5;
  a5 = 0xB369A136;
  temp = (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0xB369A136;
  a1 += temp;
  a30 = 4188039363;
  a31 = 3587894964;
  a0 ^= 4188039363u;
  a1 ^= 3587894964u;
  return 0;
}

```

[](#六：实现自动化)六：实现自动化
===================

首先我们分析程序的加密逻辑，知道是一个魔改的 TEA

 

编写一个解密函数：

```
def decrypt(self):                         # 解密
        self.a1 -= (self.a0 + ((self.a0 >> 5) ^ (16 * self.a0))) ^ self.key_table[self.i + 1]
        self.a1 &= 0xffffffff
        self.a0 -= (self.a1 + ((self.a1 >> 5) ^ (16 * self.a1))) ^ self.key_table[self.i]
        self.a0 &= 0xffffffff
        self.i -= 2
        if self.i == -2:
            a0_string = binascii.unhexlify(hex(self.a0 & 0xffffffff)[2:]).decode("utf-8")
            a0_string = a0_string[::-1]
            a1_string = binascii.unhexlify(hex(self.a1 & 0xffffffff)[2:]).decode("utf-8")
            a1_string = a1_string[::-1]
            self.string = a0_string + a1_string
            print(self.string, end='')
            return
        self.decrypt()

```

分析下程序的规律

 

我们需要提取的，就是 vm 指令对应的 opcode 以及整个字节码

 

![](https://bbs.pediy.com/upload/attach/202109/921830_TZ75MC9R6D48NF3.png)  
分析 elf 的内存，偏移是固定的

 

直接每次读取即可

 

最终脚本：

```
# _*_ coding: utf-8 _*_
# editor: SYJ
# function: Reversed By SYJ
"""
describe:
 
"""
from pwn import *
import hashlib
 
class Decompile:
    def __init__(self, file_name, opcode_keys_, code):
        self.filename = file_name   # 要逆向的文件路径
        self.asm_code = []          # 反汇编的指令列表
        self.pc = 0                 # 虚拟机的rip
        self.code = code            # 存放文件的虚拟code
        self.opcode_keys = opcode_keys_  # 存放文件的指令集代表的opcode
        self.recompile_to_c = ""    # 二维数组来存放每个文件反编译成c的代码
        self.key_table = []         # 二维数组来存放文件的加密用的key_table
        self.enc_number = 0         # 根据key来判断加密的次数
        self.a0 = 0                 # 加密之后的数据(后面得到key_table之后会设置)
        self.a1 = 0
        self.i = 0                 # 解密的递归循环变量,用来判断是否退出递归
        self.string = ''           # 解密之后的字符串
 
    def recompile(self):      # 定义一个方法来重编译(没必要,指令很简单,没有涉及栈操作)
        memory = bytes(self.code)
        data_bin = memory      # 前面划分一段区域来当内存,直接将我们的opcode存放进去
        asm_code = "\n".join(self.asm_code)
        code_bin = asm(asm_code, arch="i386")
        file = "{}_bin".format(string[len(self.filename)-10:len(self.filename)-4])
        open(file, "wb").write(data_bin + code_bin)
 
    def get_opcode(self):          # 获取opcode
        opcode = self.code[self.pc]
        return opcode
 
    def rip_ad_three(self):       # 获取rip跳转和操作数
        fuck_num1 = self.code[self.pc+1]
        fuck_num2 = self.code[self.pc+2]
        self.pc += 3
        return fuck_num1, fuck_num2
 
    def rip_ad_five(self):
        fuck_num1 = self.code[self.pc+1]
        self.pc += 6
        return fuck_num1
 
    def used_by_jcc(self):          # 根据虚拟机实现跳转指令的方式(分析出来获取比较值和跳转值)
        fuck_num1 = self.code[self.pc + 1]
        self.pc += 5
        val = self.code[self.pc]
        self.pc += 1
        return fuck_num1, val
 
    def rip_ad_six(self):            # 这个里面有点特殊,除开获取跳转和操作数之外,还要获取key以及key_len
        fuck_num1 = self.code[self.pc+1]
        fuck_num2 = bytes(self.code[self.pc+2:self.pc+6])[::-1]
        hex_val = '0x'               # 该值用来返回
        for x in fuck_num2:
            hex_val += "{:02x}".format(x)
        self.pc += 6
        if hex_val != '0x00000004' and hex_val != '0x00000005':
            self.key_table.append(eval(hex_val))         # eval自动识别
        return fuck_num1, hex_val
 
    def disamone(self):          # 反编译为伪汇编指令
        regs = ['eax', 'ebx', 'ecx', 'edx', 'esi', 'edi', 'reg6', 'reg7', 'reg8', 'reg9', 'reg10', 'reg11', 'reg12', 'reg12', 'reg14', 'reg15', 'reg16', 'reg17', 'reg18', 'reg19', 'eax', 'ebx', 'reg22', 'reg23', 'reg24', 'reg25', 'reg26', 'reg27', 'reg28', 'reg29', 'ecx', 'edx', 'reg32']
        opcode = self.get_opcode()
        offset = len(self.code)
        if opcode == self.opcode_keys[11]:
            pc = self.pc
            num1,num2 = self.rip_ad_six()
            self.asm_code.append("_%s: mov %s, %s" % (hex(offset+pc), regs[num1], num2))
        elif opcode == self.opcode_keys[0]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov %s, %s" % (hex(offset+pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[1]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: add %s, %s" % (hex(offset+pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[2]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: shl ecx, al" % (hex(offset + pc)))        # regs[num2] % (hex(offset + pc), regs[num1])
        elif opcode == self.opcode_keys[3]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: shr edx, bl" % (hex(offset + pc)))       # , regs[num2] % (hex(offset + pc), regs[num1])
        elif opcode == self.opcode_keys[4]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov eax, %s\ncdq\nidiv %s\nmov %s, edx" % (hex(offset + pc), regs[num1], regs[num2], regs[num1]))
        elif opcode == self.opcode_keys[7]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: xor %s, %s" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[5]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov %s, [%s+0x86]" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[6]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: mov [%s+0x86], %s" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[8]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.asm_code.append("_%s: cmp %s, %s" % (hex(offset + pc), regs[num1], regs[num2]))
        elif opcode == self.opcode_keys[9]:
            pc = self.pc
            num1 = self.rip_ad_five()
            self.asm_code.append("_%s: nop" % (hex(offset + pc)))
        elif opcode == self.opcode_keys[10]:
            pc = self.pc
            num1, val = self.used_by_jcc()
            jcc = ['jmp', 'jl', 'jg']
            temp = ''
            if val == 0:
                temp = jcc[0]
            elif val == 1:
                temp = jcc[1]
            elif val == 2:
                temp = jcc[2]
            self.asm_code.append("_%s: %s %s" % (hex(offset + pc), temp, num1))
        else:
            print("undefined instruction")
 
    def disamtoc(self):                # 根据分析反编为C代码
        regs = ['a0', 'a1', 'a2', 'a3', 'a4', 'a5', 'a6', 'a7', 'a8', 'a9', 'a10', 'a11', 'a12', 'a13', 'a14', 'a15', 'a16', 'a17', 'a18', 'a19', 'a20', 'a21', 'a22', 'a23', 'a24', 'a25', 'a26', 'a27', 'a28', 'a29', 'a30', 'a31', 'a32']
        opcode = self.get_opcode()
        if opcode == self.opcode_keys[11]:
            pc = self.pc
            num1, num2 = self.rip_ad_six()
            # self.recompile_to_c += "_{}: {}={};\n".format(hex(pc), regs[num1], num2)
            self.recompile_to_c += "{}={};\n".format(regs[num1], num2)
        elif opcode == self.opcode_keys[0]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}={};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[1]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}+{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}+={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[2]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}<<{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}<<={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[3]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}>>{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}>>={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[4]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {} % {};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{} %= {};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[7]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}^{};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{} ^= {};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[5]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: {}=arr[{}+0x86];\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "{}=arr[{}+0x86];\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[6]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            # self.recompile_to_c += "_{}: arr[{}+0x86]={};\n".format(hex(pc), regs[num1], regs[num2])
            self.recompile_to_c += "arr[{}+0x86]={};\n".format(regs[num1], regs[num2])
        elif opcode == self.opcode_keys[8]:
            pc = self.pc
            num1, num2 = self.rip_ad_three()
            self.recompile_to_c += "eflag={}-{};\n".format(regs[num1], regs[num2])
            # self.recompile_to_c += "_{}: eflag={}-{};\n".format(hex(pc), regs[num1], regs[num2])
        elif opcode == self.opcode_keys[9]:
            pc = self.pc
            num1 = self.rip_ad_five()
            # self.recompile_to_c += "_{}: goto _{}".format(hex(pc), hex(pc+1))
            self.recompile_to_c += "\n"
        elif opcode == self.opcode_keys[10]:
            pc = self.pc
            num1, val = self.used_by_jcc()
            jcc = ['jmp', 'jl', 'jg']
            temp = ''
            if val == 0:
                temp = jcc[0]
            elif val == 1:
                temp = jcc[1]
            elif val == 2:
                temp = jcc[2]
            self.recompile_to_c += "\n"
            # self.recompile_to_c += "_{}: val={};\nif(val == 0):goto _{};\nif(val==1 && eflag<0):goto _{}\nif(val==2 && eflag>0):goto _{}\n".format(hex(pc), val, hex(pc+num1), hex(pc+num1), hex(pc+num1))
        else:
            print("undefined instruction")
 
    def get_asm_code(self):         # 获取伪汇编代码
        while self.pc < len(self.code):
            self.disamone()
        print("\n".join(self.asm_code))
 
    def get_c_code_and_encinfo(self):           # 获取重编的C代码和加密的一些信息
        while self.pc < len(self.code):
            self.disamtoc()
        self.a0 = self.key_table[len(self.key_table)-2]
        self.a1 = self.key_table[len(self.key_table)-1]
        self.enc_number = (len(self.key_table) - 2)
        self.i = len(self.key_table) - 4
        # print(self.i)
        # print(self.enc_number)      # 打印key的数量
        # print(self.recompile_to_c)  # print c_code
        # print(self.key_table)       # 打印key_table
        # print(self.a0)
        # print(self.a1)
 
    def decrypt(self):                         # 解密
        self.a1 -= (self.a0 + ((self.a0 >> 5) ^ (16 * self.a0))) ^ self.key_table[self.i + 1]
        self.a1 &= 0xffffffff
        self.a0 -= (self.a1 + ((self.a1 >> 5) ^ (16 * self.a1))) ^ self.key_table[self.i]
        self.a0 &= 0xffffffff
        self.i -= 2
        if self.i == -2:
            a0_string = binascii.unhexlify(hex(self.a0 & 0xffffffff)[2:]).decode("utf-8")
            a0_string = a0_string[::-1]
            a1_string = binascii.unhexlify(hex(self.a1 & 0xffffffff)[2:]).decode("utf-8")
            a1_string = a1_string[::-1]
            self.string = a0_string + a1_string
            print(self.string, end='')
            return
        self.decrypt()
 
path = 'D:\\ReverseSource\\2021wmx-VMautomatic'       # 所在的文件夹路径
file_number = 16                                      # 文件数量
for i in range(file_number):
    path += '\\solve{}.exe'.format(i)                 # 文件绝对路径
    data_bin = open(path, 'rb').read()                # 文件的bin数据
 
    opcode_length = data_bin[0xF390: 0xF390+4]        # 根据文件偏移为0x0000000000010390 - 0x1000获取opcode长度
    opcode_length = opcode_length[::-1]
    opcode_length_hex = '0x'
    for x in range(len(opcode_length)):
        opcode_length_hex += "{:02x}".format(opcode_length[x])
    opcode_length_value = eval(opcode_length_hex)
    # print(hex(opcode_length_value))
 
    opcode_keys = []                                  # 根据文件偏移都为0x4020-0x1000获取opcode_keys
    for j in range(12):
        opcode_keys.append(data_bin[0x3020+j])
    # print(opcode_keys)
 
    opcode = data_bin[0x3040: 0x3040+opcode_length_value]      # 根据文件偏移为0x4040-0x1000得出虚拟机的code,读取长度为前面得到的opcode_length_value
    opcode_arr = []
    for z in range(opcode_length_value):
        opcode_arr.append(opcode[z])
    # print(opcode_arr)
 
    temp_object = Decompile(path, opcode_keys, opcode_arr)
    # temp_object.get_asm_code()
    temp_object.get_c_code_and_encinfo()
    temp_object.decrypt()
    path = "D:\\ReverseSource\\2021wmx-VMautomatic"
 
print("\n")
# 将控制台输出的字符串全部md5即可得到flag
all_solve = "ozeletrtlhqtbthxqtkclucqtrkmrpguwewbzocqwnvhyxrxkdgfeqdefoucorbsbidzeijvcncopedpkfbezltqvebskounguxsvojwpgocejgmjdaxfscjflhjpqts"
print('flag{'+ hashlib.md5(all_solve.encode('utf-8')).hexdigest().lower() + '}')

```

个人 notion 笔记地址：https://agate-colony-3f5.notion.site/CTF-VM-94d5b358d2b946b0a64f9910030f78af

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 5 小时前 被 SYJ-Re 编辑 ，原因：

[#CrackMe](forum-37-1-198.htm) [#Reverse](forum-37-1-111.htm)

上传的附件：

*   [2021VMautomatic.zip](javascript:void(0)) （829.17kb，1 次下载）