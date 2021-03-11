> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266397.htm)

##### 前言

之前用 [LIEF](https://github.com/lief-project/LIEF) 最多也就是添加一个依赖 so，偶然细读了一下 [LIEF 的文档](https://lief.quarkslab.com/doc/latest/api/python/elf.html) ，意外发现他还有一些比较好玩的操作，比如合并段，新增导出函数，修改导出函数之类的，结合一下 inlinehook，于是就有了这篇文章

##### 简介

使用 ndk 编译提供代码 so，再使用 lief 合并代码进原始 so，最后在对汇编代码稍作修改即可完成最原滋原味的 inlinehook

##### 具体操作

###### lief 的简单使用

```
import lief
 
def swap(obj,sym1,sym2):
    s1 = obj.get_symbol(sym1)
    s2 = obj.get_symbol(sym2)
    temp = s2.name
    s2.name = s1.name
    s1.name = temp
 
if __name__ == '__main__':
 
    lib_src = lief.parse("libnative-lib.so")
    lib_code = lief.parse("libcodeProvider.so")
    print(lib_code)
 
    # 互换导出函数地址
    swap(lib_src,"Java_com_lzy_lieftest_MainActivity_Test1","Java_com_lzy_lieftest_MainActivity_Test2")
 
    # libcodeProvider.so 的第二个段（包含.text节）添加进 libnative-lib.so
    segment_added = lib_src.add(lib_code.segments[1])
 
    # 合并进来的两个函数依旧为其添加导出函数（方便我们查找）
    code_test3 = lib_code.get_symbol("test3")
    code_innerCallFunctionRep = lib_code.get_symbol("innerCallFunctionRep")
    lib_src.add_exported_function(segment_added.virtual_address + code_test3.value, "test3Rep")
    lib_src.add_exported_function(segment_added.virtual_address + code_innerCallFunctionRep.value, "innerCallFunctionRep")
 
    lib_src.write("libnative-lib1.so")

```

 

由上脚本合并 so 的代码段  
下面介绍一下三种 inlinehook 姿势

###### 使用 BL 替换 BL

 

![](https://bbs.pediy.com/upload/attach/202103/868525_CYXSQH97KVB5J5R.png)

 

![](https://bbs.pediy.com/upload/attach/202103/868525_QFKUT95949CZ9UH.png)

 

nop 之间的代码就是我们正式的 hook 代码（ndk 编译的那部分代码），这部分代码如果使用到 got 表的函数需要手动去修正一下，比如上截图的日志函数  
这么操作的话可拓展性就非常的大，多余的参数零时变量什么的，你都可以进入 hook 函数的时候去申请更大的栈空间用来存放，退出的时候也记得恢复

###### 使用 B 替换

 

![](https://bbs.pediy.com/upload/attach/202103/868525_J9QGM3BKPNC95SZ.png)

 

![](https://bbs.pediy.com/upload/attach/202103/868525_XS68ZBXRZXCCMXZ.png)

 

![](https://bbs.pediy.com/upload/attach/202103/868525_7RZS84UPKRXB8VE.png)

 

在跳走的 empFunction 中我们就有空位随意的写入汇编指令  
跳走之后被替换部分的汇编跳转记得手动修复一下跳转地址

 

上文中举例的 Demo so 很小，所以直接就用 bl/b 跳转，但是这里需要注意 Arm 的 B 系列指令跳转范围只有 ±32MB，Thumb 的 B 系列指令跳转范围只有 ±256 字节，遇到 so 比较大的情况，考虑使用 add pc 来进行跳转

 

另一种跳转（ldr pc）可以参考 inlinehook 的这篇文档 http://ele7enxxh.com/Android-Arm-Inline-Hook.html

###### 使用 add pc 替换任意位置

 

![](https://bbs.pediy.com/upload/attach/202103/868525_EGHHEGEYWHJWVRS.png)

 

![](https://bbs.pediy.com/upload/attach/202103/868525_H7B5SAAEW92BHJR.png)

 

![](https://bbs.pediy.com/upload/attach/202103/868525_UV4CVWRF72JX95G.png)

 

这里简单的讲解一下 0xdf34 的值是怎么算的:  
代码执行到 0xdf30 时，pc 的值应该是等 0xdf38，所以 0xdf34 位置的值应该是 0x40b3c - 0xdf38

##### 拓展

```
extern "C"
void injectLog(){
 
    void* v_r0;
    void* v_r1;
    void* v_r2;
    void* v_r3;
 
    void* v_r4;
    void* v_r5;
    void* v_r6;
    void* v_r7;
    void* v_r8;
    void* v_r9;
    void* v_r10;
    void* v_r11;
 
    void* v_ip;
    void* v_sp;
    void* v_lr;
    void* v_pc;
 
    asm(
    "mov %0,r0\n"
    "mov %1,r1\n"
    "mov %2,r2\n"
    "mov %3,r3\n"
    "mov %4,r4\n"
    "mov %5,r5\n"
    "mov %6,r6\n"
    "mov %7,r7\n"
    "mov %8,r8\n"
    "mov %9,r9\n"
    "mov %10,r10\n"
    "mov %11,r11\n"
    :"=r"(v_r0),"=r"(v_r1),"=r"(v_r2),"=r"(v_r3)
        ,"=r"(v_r4),"=r"(v_r5),"=r"(v_r6),"=r"(v_r7),"=r"(v_r8),"=r"(v_r9),"=r"(v_r10),"=r"(v_r11)
    );
 
    asm(
    "mov %0,ip\n"
    "mov %1,sp\n"
    "mov %2,lr\n"
    "mov %3,pc\n"
    :"=r"(v_ip),"=r"(v_sp),"=r"(v_lr),"=r"(v_pc)
    );
 
    LOGD("inject register log : \n"
            "%p\t%p\t%p\t%p\n"
            "%p\t%p\t%p\t%p\t%p\t%p\t%p\t%p\n"
//            "%p\t%p\t%p\t%p\n"
            ,v_r0,v_r1,v_r2,v_r3
            ,v_r4,v_r5,v_r6,v_r7,v_r8,v_r9,v_r10,v_r11
//            ,v_ip,v_sp,v_lr,v_pc
            );
}

```

##### 总结

用到 lief 做这样的 inlinehook，目前还没有发现有什么好的实际用处，但是可以帮我们用实践去更好的理解 inlinehook 的原理，很多东西看着简单但是一操作起来就踩坑，实践大于理论，有兴趣的伙伴可以试试

 

附件（APK DEMO）：  
https://pan.baidu.com/s/176tG65p8v7s9Bzc2eiO_nQ  
tcui

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)

最后于 14 小时前 被唱过阡陌编辑 ，原因：