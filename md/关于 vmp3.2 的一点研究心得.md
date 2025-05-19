> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2032343-1-1.html)

> 最近在学代码虚拟化，看了章立春老师的代码虚拟与自动化分析这本书前几章，第三章自己实现了一个简单的虚拟机，明白了虚拟化的基本原理。遂开始研究了一下 vmp3.2，谈论一下 ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)yoyoRev 最近在学代码虚拟化，看了章立春老师的代码虚拟与自动化分析这本书前几章，第三章自己实现了一个简单的虚拟机，明白了虚拟化的基本原理。遂开始研究了一下 vmp3.2，谈论一下我的一点理解和心得，如有错误请指正。刚开始接触 vmp, 所以弄的简单一点，先写了一个简单的程序，编译的时候把随机基址，增量链接啥的全部关掉，然后用 vmp3.2 对选中的代码进行虚拟化，只模拟这几条指令，代码如下

```
#include<windows.h>
#include "VMProtectSDK.h"
#pragma warning(disable : 4996)
int main() {
    DWORD dwval;
    VMProtectBeginVirtualization("yoyo");
    __asm {
        mov eax,0x10000000
        add eax,0x1234
        mov dwval,eax

    }
    VMProtectEnd();
    char buff[16];
    itoa(dwval, buff, 16);
    MessageBoxA(0,buff,"result",MB_OK| MB_ICONEXCLAMATION);
    return 0;

}
```

然后使用 vmp3.2 进行虚拟化，只虚拟化三条指令即可。  
![](https://attach.52pojie.cn/forum/202505/18/184958hlhne1flgglxglu2.png)

**f8dcac95-8727-48bf-a74d-5de9f0081652.png** _(51.4 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkwN3wyZjdiYmQ3OHwxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

要虚拟化的指令

2025-5-18 18:49 上传

  
接下来看虚拟化前后的区别：  
![](https://attach.52pojie.cn/forum/202505/18/185615iggttyj1mq9b1y1v.png)

**c7adec27-66c6-4a66-a230-c30c15da1988.png** _(169.57 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkwOHxhNzhlZTNkN3wxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

虚拟化之前的结果

2025-5-18 18:56 上传

  
![](https://attach.52pojie.cn/forum/202505/18/185759vpu59rr58uvvu8u5.png)

**0eb45bd4-88ab-4fc8-82ce-9a720ee41216.png** _(162.23 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkwOXw3ZDliMWE2NnwxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

虚拟化之后的结果

2025-5-18 18:57 上传

  
可以看到已经被一个 push 和一个 jmp 指令所替换，在这里就得说一下虚拟化的基本原理了，如章老师这本书所述，总结一下就是保存现场，模拟执行，然后将模拟之后的结果恢复到真实的寄存器，退出虚拟机。  
![](https://attach.52pojie.cn/forum/202505/18/190042bbfawtpfutawbgwt.png)

**8676a87af638767b227bbc3b717c839c.png** _(143.7 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkxMHxkOWQ5YzY0MXwxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

章老师书中所述的虚拟化基本原理

2025-5-18 19:00 上传

  
接下来看 vmp 是如何做的，在 0x401010 处压栈，紧接着在 0x401015 处一个 jmp 指令跳转到虚拟机入口，  
![](https://attach.52pojie.cn/forum/202505/18/190916uoxj4hee7eve37sz.png)

**54283c97-7f72-4e46-8379-f6fc025b0cd9.png** _(140.92 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkxNXxkOWQxNGQ0M3wxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

2025-5-18 19:09 上传

  
可以看到有两条指令 push 0x92124021 和 call 0x0041542B，单步 f7 跟进去

```
push ebx
seto bh
not bl
push ecx
movzx ebx, sp
jmp 0x0047CDD3
...
```

入口部分代码，还记得上面所说的吗？要保存现场环境的，所以说真正有用的指令就两条（或者说三条吧）分别是 push ebx,push ecx,jmp 0x0047cdd3, 其余指令全是垃圾指令，可以当成 nop 反正不要受到这些指令的影响 就当看不到。  
接下来看 0x0047cdd3 的指令，由于 vmp 的指令非常多且杂，所以后续不再截图或者写出来  
![](https://attach.52pojie.cn/forum/202505/18/191828shmcyn17l781r7hj.png)

**db897092-5609-4cac-92a6-479c8acd5ada.png** _(144.03 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkxOHxhNzU0YmE1OHwxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

0x0047cdd3 的指令

2025-5-18 19:18 上传

  
可以看到还是很多的垃圾指令，真正有意义的还是几条 push 指令罢了。所以我直接将有用的指令重新写了一下  

```
push 0x402110
    push 0x92124021
    call 0x41542b

0x41542b:
    push ebx
    push ecx
    push ebp
    push eax
    push esi
    push edx
    push edi
    push eflags

    mov ecx,0
    push ecx
    mov esi,dword ptr ds:[esp+0x28]  //此时esi的值是0x92124021
    dec esi
    xor esi,0x7733592f
    inc esi
    xor esi,0x66bf7d8f
    rol esi,3
    sub esi,0x1cb30155   //上面操作对esi进行解密 此时esi=0x4023a7 
    lea esi,dword ptr ds:[esi+ecx]

    mov edi,esp         //  
    lea esp,dword ptr ss:[esp-0xc0]
    mov ebx,
    sub ebx,ecx
    lea ebp,dword ptr ds:[0x466596]
    sub esi,0x4
    mov eax,[esi]
    xor eax,ebx
    not eax
    dec eax
    ror eax,0x3
    not eax
    bswap eax
    lea eax, dword ptr ds:[eax-0x62320A2C]
    xor ebx,eax
    add ebp,eax
    jmp ebp   //此时ebp = 0x42ca8c  等价于jmp 0x42ca8c

0x42ca8c:
    mov eax,dword ptr ds:[edi]
    add edi,0x4
    lea esi,dword ptr ds:[esi-1]
    movzx ecx, byte ptr ds:[esi]
    xor cl,bl
    inc cl
    neg cl
    add cl,0xB
    ror cl,0x1
    sub cl,0x8e
    neg cl
    xor cl,0x54
    dec cl
    xor bl,cl

    mov dword ptr ss:[esp+ecx], eax     //上述对cl寄存器的操作最终得到的是一个偏移量 ,后续都是这种操作,不再赘述.此时ecx=8
    sub esi,0x4
    mov ecx, dword ptr ds:[esi]
    xor ecx, ebx
    sub ecx, 0x3E850F9
    rol ecx, 0x1
    sub ecx, 0x6BA107A2
    xor ecx, 0x53345A
    lea ecx, ds:[ecx+0x53205C24]
    neg ecx
    add ecx, 0x177B784E
    rol ecx, 0x1
    xor ebx, ecx
    add ebp,ecx
    push ebp   
    ret       //跳转到0x40d855处

0x40d855:
    mov eax, dword ptr ds:[edi] //[edi]的内容是初始时压栈的eflags寄存器的值 后续会保存到虚拟机栈中
    add edi,4                   //修正edi，指向下一个成员
    lea esi, ds:[esi-0x1]
    movzx ecx, byte ptr ds:[esi]
    xor cl, bl
    inc cl
    neg cl
    add cl, 0xB
    ror cl, 0x1
    sub cl, 0x8E
    neg cl
    xor cl,0x54
    dec cl
    xor bl,cl
    mov dword ptr ss:[esp+ecx], eax //ecx=0x2c 保存真实的elags寄存器到虚拟机的elags寄存器，初始化虚拟机环境 vmcontext.eflags
    sub esi,4
    mov ecx, dword ptr ds:[esi]
    xor ecx, ebx
    sub ecx, 0x3E850F9
    rol ecx, 0x1
    lea ecx, ds:[ecx-0x6BA107A2]
    xor ecx, 0x53345A
    lea ecx, ds:[ecx+0x53205C24]
    neg ecx
    add ecx, 0x177B784E
    rol ecx, 0x1
    xor ebx, ecx
    add ebp,ecx
    jmp ebp //  此时ebp=0x0044FA3F

0x0044FA3F:

    mov eax, dword ptr ds:[edi]  //[edi]的内容是初始时压栈的edi寄存器的值 后续会保存到虚拟机栈中
    add edi,4
    sub esi, 0x1
    movzx ecx, byte ptr ds:[esi]
    xor cl, bl
    inc cl
    neg cl
    add cl,0xB
    ror cl,1
    sub cl,0x8e
    neg cl
    xor cl,0x54
    dec cl
    xor bl, cl
    mov dword ptr ss:[esp+ecx], eax //ecx=0x24 保存原始edi vmcontext.edi
    sub esi,4
    mov ecx, dword ptr ds:[esi]
    xor ecx, ebx
    sub ecx, 0x3E850F9
    rol ecx, 0x1
    sub ecx, 0x6BA107A2
    xor ecx, 0x53345A
    lea ecx, ds:[ecx+0x53205C24]
    neg ecx
    add ecx, 0x177B784E
    rol ecx, 0x1
    xor ebx, ecx
    add ebp, ecx
    jmp ebp //ebp=0x004710C6
```

  
  
  
  
依旧是部分指令，在这里我们可以先划出堆栈图，此时有朋友应该能发现刚开始压入的 0x92124021 不是随便的，而是会通过这个值进行解密得到 opcode 尾地址，并且复制给 esi 寄存器，为啥是尾地址呢，因为后续操作都是 sub esi,4  
sub esi,1 lea esi,[esi-4],lea esi,[esi-1] 其中这两条 lea 指令是和 sub 指令等价的。还有两条指令也是我们需要注意的，lea esp,dword ptr ss:[esp-0xc0]        // 提升堆栈, 为虚拟机分配堆栈和环境，lea ebp,dword ptr ds:[0x466596] // 可以将 ebp 理解为基址，因为在后面都是加 ebp 实现跳转，在得到 opcode 尾地址之后，就开始对 opcode 进行访问，然后解密，解密完成加上 ebp 得到最终要跳转的地址。  
![](https://attach.52pojie.cn/forum/202505/18/192543z1c2eexyyy6jez16.png)

**dbb8bd34-79b4-40ea-9f78-f40b68f90a19.png** _(23.7 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkyM3wzZjY5MzU5NnwxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

堆栈图

2025-5-18 19:25 上传

  
再仔细观察一下，应该会想到这些指令看着很复杂，但是终究就是算跳转地址，所做的真正有用的指令是有限的，比如 mov dword ptr ss:[esp+ecx], eax //ecx=0x2c 保存真实的 elags 寄存器到虚拟机的 elags 寄存器，初始化虚拟机环境 vmcontext.eflags 这些指令，边做完有用的指令边计算跳转，那么在这条指令中 ecx 又是怎么来的呢，在上面所述的代码中 ecx 是一个偏移量，也是通过 opcoede 进行动态解密得到的，看下面的堆栈图  
![](https://attach.52pojie.cn/forum/202505/18/194156hszrq0sj7zhjah7k.png)

**78db96cc-c591-4b7f-b423-d78417e17061.png** _(43.21 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkyNHxiNmU1MWI3N3wxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

堆栈图 2

2025-5-18 19:41 上传

  
这些偏移都是通过 opcode 计算得出，理解成虚拟机的虚拟环境，一个特别的数据结构，这么一说是不是感觉有种拨开云雾的感觉，然后我跟着虚拟机的解密写了一部分的解密，太多了实现不想写了，因为原理就是这样了，解密 opcode, 得到跳转地址，偏移啥的  

```
#include <stdio.h>
#include<windows.h>
UINT32 init_esi;
UINT32 init_ebp = 0x466596;
UINT32 CalcEsiBase(UINT32 key, UINT32 ecx) {
    key--;                  // dec esi
    key ^= 0x7733592f;      // xor esi, 0x7733592f
    key++;                  // inc esi
    key ^= 0x66bf7d8f;      // xor esi, 0x66bf7d8f
    key = (key << 3) | (key >> (32 - 3));  // rol esi, 3
    key -= 0x1cb30155;      // sub esi, 0x1cb30115
    return key + ecx;       // lea esi, [esi+ecx]
}

UINT32 CalcJmpAddress(UINT32& ebx, UINT32& opcode_ptr, UINT32& initial_ebp, UINT32 ecx) {
    ebx -= ecx;
    opcode_ptr -= 4;                         // sub esi, 4
    UINT32 eax = *reinterpret_cast<UINT32*>(opcode_ptr); // mov eax, [esi] (读取 DWORD)
    eax ^= ebx;                       // xor eax, ebx
    eax = ~eax;                       // not eax
    eax--;                            // dec eax
    eax = (eax >> 3) | (eax << (32 - 3)); // ror eax, 3
    eax = ~eax;                       // not eax
    eax = _byteswap_ulong(eax);     // bswap eax (GCC/Clang)
    eax -= 0x62320A2C;                // lea eax, [eax - 0x62320A2C]
    ebx ^= eax;                       // xor ebx, eax
    initial_ebp += eax;                       // add ebp, eax

    return initial_ebp;                       // jmp ebp
}

UINT32 CalcRetAddress(UINT32& ebx, UINT32& opcode_ptr, UINT32& initial_ebp) {
    opcode_ptr -= 4;               // sub esi, 0x4
    UINT32 ecx = *reinterpret_cast<UINT32*>(opcode_ptr); // mov ecx, [esi]

    ecx ^= ebx;                    // xor ecx, ebx
    ecx -= 0x3E850F9;              // sub ecx, 0x3E850F9
    ecx = (ecx << 1) | (ecx >> 31); // rol ecx, 1
    ecx -= 0x6BA107A2;             // sub ecx, 0x6BA107A2
    ecx ^= 0x53345A;               // xor ecx, 0x53345A
    ecx += 0x53205C24;             // lea ecx, [ecx+0x53205C24]
    ecx = (~ecx) + 1; ;                    // neg ecx
    ecx += 0x177B784E;             // add ecx, 0x177B784E
    ecx = (ecx << 1) | (ecx >> 31); // rol ecx, 1

    ebx ^= ecx;                    // xor ebx, ecx
    initial_ebp += ecx;            // add ebp, ecx

    return initial_ebp;            // 返回更新后的 ebp
}
UINT32 CalcEbpAddress(UINT32& ebx, UINT32& opcode_ptr, UINT32& initial_ebp) {
    opcode_ptr -= 4;               // sub esi, 4
    UINT32 eax = *reinterpret_cast<UINT32*>(opcode_ptr); // mov eax, [esi]

    eax ^= ebx;                    // xor eax, ebx
    eax = (eax >> 3) | (eax << (32 - 3)); // ror eax, 3
    eax ^= 0x643A73B8;             // xor eax, 0x643A73B8
    eax--;                         // dec eax
    eax = ~eax;                    // not eax

    ebx ^= eax;                    // xor ebx, eax
    initial_ebp += eax;            // add ebp, eax

    return initial_ebp;            // 返回更新后的 ebp
}
UINT32 CalcEbpAddress2(UINT32& ebx, UINT32& opcode_ptr, UINT32& initial_ebp) {
    opcode_ptr -= 4;               // sub esi, 0x4
    UINT32 eax = *reinterpret_cast<UINT32*>(opcode_ptr); // mov eax, [esi]

    eax ^= ebx;                    // xor eax, ebx
    eax = (eax << 3) | (eax >> (32 - 3)); // rol eax, 3
    eax++;                         // inc eax
    eax = _byteswap_ulong(eax);    // bswap eax (字节序反转)
    eax = (eax << 2) | (eax >> (32 - 2)); // rol eax, 0x2

    ebx ^= eax;                    // xor ebx, eax
    initial_ebp += eax;            // add ebp, eax

    return initial_ebp;            // 返回更新后的 ebp
}
UINT32 CalcEbpAddress3(UINT32& ebx, UINT32& opcode_ptr, UINT32& initial_ebp) {
    opcode_ptr -= 4;               // sub esi, 4
    UINT32 ecx = *reinterpret_cast<UINT32*>(opcode_ptr); // mov ecx, [esi]

    ecx ^= ebx;                    // xor ecx, ebx
    ecx -= 0x7F4B67A2;             // sub ecx, 0x7F4B67A2
    ecx = ~ecx;                    // not ecx
    ecx--;                         // dec ecx
    ecx = ~ecx;                    // not ecx

    ebx ^= ecx;                    // xor ebx, ecx
    initial_ebp += ecx;            // add ebp, ecx

    return initial_ebp;            // 返回更新后的 ebp
}
UINT32 CalcEbpAddress4(UINT32& ebx, UINT32& opcode_ptr, UINT32& initial_ebp) {

    opcode_ptr -= 4;
    UINT32 ecx = *reinterpret_cast<UINT32*>(opcode_ptr);
    ecx ^= ebx;
    ecx = (ecx << 2) | (ecx >> 30);
    ecx += 0x3C7C310B;
    ecx = ~ecx + 1;
    ecx =  _byteswap_ulong(ecx);
    ecx--;
    ebx ^= ecx;
    initial_ebp += ecx;
    return initial_ebp;
}
UINT32 CalcImm32(UINT32& ebx, UINT32& opcode_ptr) {
    opcode_ptr -= 4;               // sub esi, 0x4
    UINT32 ecx = *reinterpret_cast<UINT32*>(opcode_ptr); // mov ecx, [esi]

    ecx ^= ebx;                    // xor ecx, ebx
    ecx--;                         // dec ecx
    ecx = _byteswap_ulong(ecx);    // bswap ecx (字节序反转)
    ecx ^= 0x408E4C0B;             // xor ecx, 0x408E4C0B
    ecx -= 0x7FF90503;             // sub ecx, 0x7FF90503

    ebx ^= ecx;                    // xor ebx, ecx

    return ecx;                    // 返回计算后的ecx值
}
UINT32 DecodeImm32(UINT32& ebx, UINT32& opcode_ptr) {
    opcode_ptr -= 4;
    UINT32 ecx = *reinterpret_cast<UINT32*>(opcode_ptr);
    ecx ^= ebx;
    ecx--;
    ecx = _byteswap_ulong(ecx);
    ecx ^= 0x408E4C0B;
    ecx -= 0x7FF90503;
    ebx ^= ecx;
    return ecx;
}
UINT8 opcode[] = {
0x00, 0x00, 0x00, 0x00, 0x00, 0x77, 0xE4, 0x8F, 0xA9, 0x3A, 0x59, 0x37, 0xD8, 0x75, 0x8D, 0x2A,
 0x98, 0xAB, 0xCB, 0x10, 0x40, 0xE8, 0xBE, 0x81, 0x5A, 0x99, 0xEB, 0x74, 0x4B, 0xA0, 0x7F, 0xFA,
 0x37, 0xA8, 0x4B, 0x2C, 0x1F, 0xB6, 0x0D, 0x61, 0x4C, 0x01, 0xF1, 0xA5, 0xF1, 0x04, 0x57, 0x67,
 0x0C, 0xD1, 0xD7, 0x72, 0x8D, 0xA8, 0x82, 0x1F, 0xBA, 0xC8, 0x6B, 0xC8, 0xD2, 0x85, 0x67, 0x7F,
 0xF0, 0x9B, 0x00, 0x00, 0x00, 0x3B, 0xB3, 0x67, 0x08, 0xF7, 0xCF, 0xC4, 0x36, 0x08, 0x3B, 0x25,
 0xB9, 0x89, 0xF7, 0x35, 0x40, 0xC7, 0x40, 0x08, 0x40, 0xA8, 0x4D, 0x3F, 0x08, 0x54, 0x0D, 0x8B,
 0x92, 0xF7, 0xD5, 0x65, 0x5E, 0xCB, 0xF7, 0xC1, 0x5E, 0x5D, 0xE2, 0xF7, 0x66, 0x23, 0x5A, 0xDC,
 0x24, 0x03, 0xE2, 0xEF, 0xD5, 0x2A, 0xF0, 0x4C, 0x10, 0x05, 0x3E, 0xF9, 0x8B, 0x33, 0x42, 0xB5,
 0x12, 0x6F, 0xBB, 0x94, 0xEB, 0x35, 0xFC, 0x23, 0x0B, 0xF0, 0x7E, 0x14, 0x3E, 0x08, 0x86, 0xAA,
 0x66, 0x47, 0x08, 0x6E, 0xA8, 0xE2, 0xFC, 0x4B, 0x79, 0x16, 0x80, 0xFE, 0xCB, 0xA2, 0x7D, 0x7F,
 0x0E, 0x6F, 0x46, 0xD7, 0x38, 0x08, 0x8A, 0xA1, 0x46, 0xF1, 0x05, 0xE2, 0x23, 0xE1, 0xD3, 0xD1,
 0xDA, 0xF8, 0x4B, 0x9A, 0x3C, 0x73, 0xEA, 0x30, 0x03, 0x9B, 0xF2, 0xF7, 0x0E, 0x1A, 0x30, 0x98,
 0xD1, 0xDF, 0xF9, 0x1B, 0xCF, 0xEE, 0xD1, 0xF1, 0x80, 0x2D, 0x31, 0xD3, 0xDB, 0x86, 0xD9, 0x0C,
 0x0C, 0x5D, 0x6E, 0x7A, 0xDE, 0x82, 0x29, 0x7C, 0x64, 0x3A, 0xB3, 0x7C, 0x00, 0x24, 0x4D, 0x07,
 0xFC, 0xFE, 0x5B, 0xA5, 0x8B, 0xD3, 0xFF, 0x9B, 0xEF, 0xB3, 0xFC, 0xFD, 0x9B, 0x49, 0x75, 0xBE,
 0x03, 0x24, 0xE7, 0x39, 0xD5, 0x85, 0xDB, 0x21, 0x9A, 0xE9, 0xF9, 0x1B, 0x7D, 0x5A, 0x0C, 0xFB,
 0xDB, 0x86, 0x3D, 0xBE, 0xFF, 0x9B, 0x70, 0x01, 0x22, 0x7C, 0xE4, 0xF9, 0xB7, 0x50, 0x39, 0x13
};

UINT32 CalcOffset(UINT32& opcode_ptr, UINT32& ebx) {

    opcode_ptr--;
    UINT8 cl = *reinterpret_cast<UINT8*>(opcode_ptr);      // movzx ecx, byte ptr [esi]
    UINT8 bl = ebx & 0xFF;       // 取 key 的低 8 位 (bl)

    cl ^= bl;                      // xor cl, bl
    cl++;                         // inc cl
    cl = -cl;                      // neg cl (cl = ~cl + 1)
    cl += 0xB;                     // add cl, 0xB
    cl = (cl >> 1) | (cl << 7);    // ror cl, 1 (循环右移 1 位)
    cl -= 0x8E;                    // sub cl, 0x8E
    cl = -cl;                      // neg cl
    cl ^= 0x54;                    // xor cl, 0x54
    cl--;                          // dec cl
    bl ^= cl;                      // xor bl, cl (更新 key 的低 8 位)
    ebx = (ebx & 0xFFFFFF00) | bl; // 更新 key 的低 8 位
    return cl;                     // 返回计算后的 cl
}
int main() {
    init_esi = CalcEsiBase(0x92124021, 0);
    UINT32 init_ebx = init_esi;
    UINT32 opcode_ptr = reinterpret_cast<UINT32>(opcode) + sizeof(opcode);    
    UINT32 jmpebp = CalcJmpAddress(init_ebx, opcode_ptr, init_ebp, 0);
    printf("jmpebp:%x\n", jmpebp);
    UINT32 offset = CalcOffset(opcode_ptr, init_ebx);
    printf("ecxoffset:%x\n", offset);
    for (int i = 0; i < 11;i++) {
        jmpebp = CalcRetAddress(init_ebx, opcode_ptr, init_ebp);
        printf("retebp:%x\n", jmpebp);
        if (i == 10) {
            break;
        }
        offset = CalcOffset(opcode_ptr, init_ebx);
        printf("ecxoffset:%x\n", offset);
    }
    jmpebp = CalcEbpAddress(init_ebx, opcode_ptr, init_ebp);
    printf("ebpaddress:%x\n", jmpebp);
    UINT32 imm32 = CalcImm32(init_ebx, opcode_ptr);
    printf("imm32:%x\n", imm32);
    jmpebp = CalcEbpAddress2(init_ebx, opcode_ptr, init_ebp);
    printf("ebpaddress:%x\n", jmpebp);
    jmpebp = CalcEbpAddress3(init_ebx, opcode_ptr, init_ebp);
    printf("ebpaddress:%x\n", jmpebp);
    offset = CalcOffset(opcode_ptr, init_ebx);
    printf("ecxoffset:%x\n", offset);
    jmpebp = CalcRetAddress(init_ebx, opcode_ptr, init_ebp);
    printf("retebp:%x\n", jmpebp);
    jmpebp = CalcEbpAddress4(init_ebx, opcode_ptr, init_ebp);
    printf("ebpaddress:%x\n", jmpebp);
    imm32 = DecodeImm32(init_ebx, opcode_ptr);
    printf("imm32:%x\n", imm32);
    jmpebp = CalcEbpAddress2(init_ebx, opcode_ptr, init_ebp);
    printf("ebpaddress:%x\n", jmpebp);
    offset = CalcOffset(opcode_ptr, init_ebx);
    printf("ecxoffset:%x\n", offset);
    jmpebp = CalcRetAddress(init_ebx, opcode_ptr, init_ebp);
    printf("retebp:%x\n", jmpebp);
    imm32 = DecodeImm32(init_ebx, opcode_ptr);
    printf("imm32:%x\n", imm32);
    return 0;
}
```

  
跳转算法有很多，直接扣汇编给 ai, 让 ai 实现就好了，从初始 esi 之前复制，扣出 opcode 进行解密，已经可以看到了立即数 0x10000000 和 0x1234  
![](https://attach.52pojie.cn/forum/202505/18/194953huq14vj7f24eu477.png)

**bbd2ee0b-5630-4e4f-bd30-afa7bb0c0857.png** _(89.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NzkyN3xlMjI3ZTcxZnwxNzQ3NjEzMDg4fDIxMzQzMXwyMDMyMzQz&nothumb=yes)

解密效果

2025-5-18 19:49 上传

  
又可以对上述汇编指令进行进一步优化 将计算跳转的指令，计算偏移的指令，计算立即数的指令全部优化掉可以得到更精简的指令  

```
push 0x402110
    push 0x92124021
    push 0x440841

    push ebx
    push ecx
    push ebp
    push eax
    push esi
    push edx
    push edi
    pushfd  
    mov ecx,0
    push ecx
    //mov esi, 0x4023a7
    mov edi,esp
    lea esp, ss:[esp-0xC0]  //  提升堆栈 为虚拟机分配所需内存

    lea ebp, ds:[0x00466596]    //  跳转基址

    mov eax, dword ptr ds:[edi]     //0 压入的ecx 
    add edi,4
    mov dword ptr ss:[esp+8], eax

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x2c], eax //初始context.eflags

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x24], eax //context.edi

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x18], eax //context.edx

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0], eax //context.esi

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x30], eax //context.eax

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x10], eax //context.ebp

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x14], eax //context.ecx

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x38], eax //context.ebx

    mov eax, dword ptr ds:[edi]     
    add edi,4
    mov dword ptr ss:[esp+0x20], eax //0x440841 最开始的call的返回,其实没啥用

    mov eax, dword ptr ds:[edi] 
    add edi,4
    mov dword ptr ss:[esp+0x1c], eax  ////0x92124021 用于解密opcode的数据,现在也没啥用

    //上面是保存现场环境
    mov edx,edi
    sub edi,4
    mov dword ptr ds:[edi], edx

    sub edi,4
    mov dword ptr ds:[edi],4
    mov ecx, dword ptr ds:[edi] //mov ecx,4
    mov edx, dword ptr ds:[edi+0x4]
    add ecx,edx
    mov dword ptr ds:[edi+0x4], ecx //ecx = old_esp 被vm之前的esp (第一条push指令(push 0x402110)前的esp) 感觉这几行代码完全没有意义

    pushfd
    pop dword ptr ss:[edi]
    mov eax, dword ptr ds:[edi]     //一个eflags的值 依旧没意义 

    add edi,4
    mov dword ptr ss:[esp+0x3c], eax //一个毫无意义的eflags，可能是为了占位esp+0x3c
    mov edi, dword ptr ds:[edi]     //edi = old_esp

    //解密得到imm32,解密步骤直接省略
    mov ecx,0x10000000
    sub edi,4
    mov dword ptr ds:[edi], ecx //ecx=0x10000000

    mov eax, dword ptr ds:[edi]
    mov dword ptr ss:[esp+0xc], eax     //0x10000000

    //解密得到imm32
    mov ecx,0x1234
    sub edi,4
    mov dword ptr ds:[edi], ecx

    //将解密得到的数据写入栈，覆盖之前的数据

    mov eax, dword ptr ss:[esp+0xc]     //  取出0x10000000
    sub edi,4
    mov dword ptr ds:[edi], eax

    mov ecx, dword ptr ds:[edi]     //0x10000000
    mov edx, dword ptr ds:[edi+0x4]     //0x1234
    add ecx, edx    //  求和
    mov dword ptr ds:[edi+0x4], ecx //保存求和后的结果到栈

    //由于add指令会影响eflgas寄存器 得到求和之后的eflags
    pushfd
    pop dword ptr ss:[edi]

    mov eax, dword ptr ds:[edi]     //求和之后的eflags
    add edi,4
    mov dword ptr ss:[esp+0x1c], eax //求和之后的eflags覆盖之前的垃圾数据(0x92124021) vmcontext.eflags

    mov eax, dword ptr ds:[edi] //求和后的结果
    add edi,4
    mov dword ptr ss:[esp+0x3c], eax    //将得到的0x10001234覆盖之前的无意义eflags 后续会发现esp+0x3c是vmcontext.eax

    mov eax, dword ptr ss:[esp+0x3c]
    sub edi,4
    mov dword ptr ds:[edi], eax //无意义的指令 因为[edi]此时的值就是0x10001234 可能单纯为了sub edi,4？猜测

    mov eax, dword ptr ss:[esp+0x10]   //vmcontext.ebp
    sub edi,4
    mov dword ptr ds:[edi], eax        //vmcontext.ebp
    sub edi,4

    //就是是解密imm32 可能突然觉得这个立即数毫无头绪，但是模拟的指令有一条是mov dword ptr ss:[ebp-0x18], eax
    //这样看是不是就有什么发现了
    mov ecx,0xFFFFFFE8

    mov dword ptr ds:[edi], ecx     //0xFFFFFFE8或者-0x18
    mov ecx, dword ptr ds:[edi]     
    mov edx, dword ptr ds:[edi+0x4] //  ebp
    add ecx,edx                     //得到ebp-0x18的地址
    mov dword ptr ds:[edi+0x4], ecx //保存ebp-0x18的地址

    pushfd
    pop dword ptr ss:[edi]  //上面有add指令，依旧是保存eflags,但是没啥用其实
    mov eax, dword ptr ds:[edi]     //eflags
    add edi,4
    mov dword ptr ss:[esp+0x4], eax //eflags

    mov edx, dword ptr ds:[edi]     //此时edx=ebp-0x18
    mov eax, dword ptr ds:[edi+0x4] //取出0x10001234
    mov dword ptr ss:[edx], eax //完成mov dword ptr ss:[ebp-0x18], eax的虚拟化

    //解密得到返回地址
    mov ecx,0040102E

    sub edi,4
    mov dword ptr ds:[edi], ecx    //此时ecx的值恰好是虚拟机返回地址

    //后面将虚拟寄存器转化成特殊的数据结构,从低地址到高地址分别是 
    /*
    eflags
    edi
    edx
    esi
    eax=0x10001234
    ebp
    ecx
    ebx
    0x40102e //解密得到的返回地址
    */
    //与后面的pop reg刚好是对应的

    mov eax, dword ptr ss:[esp+0x38] 
    sub edi,4
    mov dword ptr ds:[edi], eax //vmcontext.ebx

    mov eax, dword ptr ss:[esp+0x14] 
    sub edi,4
    mov dword ptr ds:[edi], eax //vmcontext.ecx

    mov eax, dword ptr ss:[esp+0x10] 
    sub edi,4
    mov dword ptr ds:[edi], eax //vmcontext.ebp

    mov eax, dword ptr ss:[esp+0x3c]    
    sub edi,4
    mov dword ptr ds:[edi], eax //应该是vmcontext.eax

    mov eax, dword ptr ss:[esp+0]       
    sub edi,4
    mov dword ptr ds:[edi], eax //vmcontext.esi

    mov eax, dword ptr ss:[esp+0x18]        
    sub edi, 0x4
    mov dword ptr ds:[edi], eax //vmcontext.edx

    mov eax, dword ptr ss:[esp+0x24]        
    sub edi,4
    mov dword ptr ds:[edi], eax //vmcontext.edi

    mov eax, dword ptr ss:[esp+0x1c]
    sub edi,4
    mov dword ptr ds:[edi], eax         //vmcontext.eflags

    mov esp,edi         //设置esp寄存器 准备恢复寄存器的值到真实寄存器

    //恢复现场
    popfd
    pop edi
    pop edx
    pop esi
    pop eax
    pop ebp
    pop ecx
    pop ebx
    ret 

//至此，整个虚拟化的分析大概完成
```

  
至此，整个虚拟化的流程大概分析完毕了，vmp 就是边解密边执行再加上各种的垃圾指令很容易懵逼，刚开始接触 vm，依旧有很多不足之处，仍需继续努力 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) likemian 牛 B 啊猫猫 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif)dengbin 学习了，感谢大神 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) wsck63304521 现在很多大佬都玩 vm 中直接爆破 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) schm168 有个虚拟问题很想请教大师指点 ![](https://avatar.52pojie.cn/data/avatar/001/91/24/95_avatar_middle.jpg) GoogleHacking 楼主用的这啥软件，olldbg 吗？