> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271960.htm)

> luajit 逆向之注入文件

自己制作的一个小工具, 工作场景为某些 cocos2dx 手游, 虽然 lua 可以直接拿到 lua_state 指针注入, 但在某些场景下直接注入文件是更加方便的, 所以自己研究了一下字节码, 笔记是以前写的. 思路很清晰, 自己动手做的话容易犯小错误

准备
--

Luajit 版本 2.1.0 beta2  
https://luajit.org/download/LuaJIT-2.1.0-beta2.zip  
mingw64

luajit 二进制文件结构
--------------

010editor 中有 bt 模板, 但是由于版本变化, 指令解析存在问题, 结构解析无问题

 

示例 lua 文件代码:

```
--target.lua
function Test(x)
    print("Test " .. x)
end
 
Test("enter main==>test()")
require("addLib")
 
function fun1()
    print("Test in target.lua fun1 function...")
end
function fun2()
    print("Test in target.lua fun2 function...")
end
 
print("Test in target.lua main function...")
print(add(1,2))

```

![](https://bbs.pediy.com/upload/tmp/831526_4JPPWVVJNXNQ9RA.png)

 

可以看到, 在 luajit 的二进制文件中, proto 的顺序是依次往下的嵌套关系, 主函数体在最外层, 也就是倒数第二个

 

在 proto 结构中, 存有头信息, 二进制字节码, 常量信息.

 

![](https://bbs.pediy.com/upload/tmp/831526_QRUNY8DYEU6UKN5.png)

 

在头信息中我们要关心的主要有 size, complex_constants_count, instructions_count 三个字段, 他们的大小由变长数字 uleb128 描述, 互相转化的方法如下

```
int ReadUleb128(unsigned char*& buf)
{
    unsigned int result;
    unsigned char cur;
 
    result = *buf++;
    if (result > 0x7f)
    {
        cur = *buf++;
        result = (result & 0x7f) | (unsigned int)((cur & 0x7f) << 7);
        if (cur > 0x7f)
        {
            cur = *buf++;
            result |= (unsigned int)(cur & 0x7f) << 14;
            if (cur > 0x7f)
            {
                cur = *buf++;
                result |= (unsigned int)(cur & 0x7f) << 21;
                if (cur > 0x7f)
                {
                    cur = *buf++;
                    result |= (unsigned int)cur << 28;
                }
            }
        }
    }
    return result;
}
void WriteUleb128(unsigned char*& buf, int x)
{
    unsigned int denominator = 0x80;
    unsigned char flag = 0x80;
 
    for (int i = 0; i < 5; i++)
    {
        if (x < denominator)
        {
            buf[i] = (unsigned char)x;
            i++;
            buf += i;
            return;
        }
        buf[i] = (flag) | (unsigned char)(x % denominator);
        x = x >> 7;
    }
}

```

在我们修改文件的时候需要修正这几条数据.

总体思路
----

修改二进制数据, 插入 require("inject") 语句. 在 luajit 中, require 语句由三条指令构成, 主要是加载 require 字符串, 待注入文件名, call 三步

<table><thead><tr><th><strong><em>\</em>OP**</strong></th><th><strong><em>\</em>A**</strong></th><th><strong><em>\</em>B**</strong></th><th><strong><em>\</em>C/D**</strong></th><th><strong><em>\</em>Description**</strong></th></tr></thead><tbody><tr><td>GGET</td><td>dst</td><td></td><td>str</td><td>A = _G[D]</td></tr></tbody></table>

 

字节码 36 00 00 00

 

第一个字节为 opcode, 第二个字节为寄存器编号, 第三个字节为常量池下表 (从下往上数)

<table><thead><tr><th><strong><em>\</em>OP**</strong></th><th><strong><em>\</em>A**</strong></th><th><strong><em>\</em>D**</strong></th><th><strong><em>\</em>Description**</strong></th></tr></thead><tbody><tr><td>KSTR</td><td>dst</td><td>str</td><td>Set A to string constant D</td></tr></tbody></table>

 

字节码为 27 01 01 00

 

第一个字节为 opcode, 第二个字节为寄存器编号, 第三个字节为常量池下表 (从下往上数)

<table><thead><tr><th><strong><em>\</em>OP**</strong></th><th><strong><em>\</em>A**</strong></th><th><strong><em>\</em>B**</strong></th><th><strong><em>\</em>C/D**</strong></th><th><strong><em>\</em>Description**</strong></th></tr></thead><tbody><tr><td>CALL</td><td>base</td><td>lit</td><td>lit</td><td>Call: A, ..., A+B-2 = A(A+1, ..., A+C-1)</td></tr></tbody></table>

 

42 00 02 01

 

这里我有点没太明白, 按照官网表格的注释 A+C-1=0, 参数为 0 个. 不过可以确定的是第二个字节为 require 的寄存器编号

1.  将三条指令插入指令数组最前面, 将字符串插入常量池最前面
2.  更新 heade 中的 complex_constants_count 和 instructions_count
3.  更新 size, 因为 uleb128 不定长, 所以最后更新这个数据

过程
--

首先跳过前置段, 找到 main, 然后解析 main proto 的信息

```
struct ProtoHeader
{
    int protoSize, complexCnt, numericCnt, instructionCnt;
    unsigned char flags, argCnt, frameSize, upValueCnt;
};
struct Proto
{
    ProtoHeader ph;
    unsigned char* instructions, * constants;
    int instructionsSize, constantsSize;
};
 
void IngoreSeg(unsigned char*& buf, int n)
{
    while (n--)
    {
        int size = ReadUleb128(buf);
        buf += size;
    }
}
Proto* ReadProto(unsigned char* buf)
{
    unsigned char* ed = buf;
    int size = ReadUleb128(ed);
    ed += size;
 
    Proto* proto = new Proto;
    proto->ph.protoSize = ReadUleb128(buf);
    proto->ph.flags = ReadU8(buf);
    proto->ph.argCnt = ReadU8(buf);
    proto->ph.frameSize = ReadU8(buf);
    proto->ph.upValueCnt = ReadU8(buf);
    proto->ph.complexCnt = ReadUleb128(buf);
    proto->ph.numericCnt = ReadUleb128(buf);
    proto->ph.instructionCnt = ReadUleb128(buf);
 
    proto->instructionsSize = 4 * proto->ph.instructionCnt;//每条指令4个字节
    proto->instructions = new unsigned char[proto->instructionsSize];
    memcpy(proto->instructions, buf, proto->instructionsSize);
    buf += 4 * proto->ph.instructionCnt;
 
    proto->constantsSize = ed - buf;
    proto->constants = new unsigned char[proto->constantsSize];
    memcpy(proto->constants, buf, proto->constantsSize);
    buf += proto->constantsSize;
    return proto;
}
 
IngoreSeg(filePtr, targetSeg);
Proto* proto = ReadProto(filePtr);

```

将原本的信息解析完毕后, 需要插入常量和指令

```
void InsertInstruction(Proto* proto, string& name)
{
    unsigned char* tmp;
 
    proto->ph.complexCnt += 2;
    tmp = new unsigned char[proto->constantsSize + 64];
    unsigned char* st = tmp;
    WriteString(tmp, string("require"));
    WriteString(tmp, name);
    memcpy(tmp, proto->constants, proto->constantsSize);
    proto->constantsSize += tmp - st;
    proto->constants = st;
 
    //_G[complexCnt-1]="require" _G[complexCnt-2]="inject"
    tmp = new unsigned char[proto->instructionsSize + 12];
    memcpy(tmp + 12, proto->instructions, proto->instructionsSize);
    unsigned char injectCode[] = { 0x36, 0x00, 0x00, 0x00,
                                    0x27, 0x01, 0x01, 0x00,
                                    0x42, 0x00, 0x02, 0x01 };
    injectCode[2] = proto->ph.complexCnt - 1;//第二个字节为require所处位置
    injectCode[6] = proto->ph.complexCnt - 2;//第二个字节为require所处位置
    memcpy(tmp, injectCode, 12);
    proto->instructions = tmp;
    proto->ph.instructionCnt += 3;
    proto->instructionsSize += 3 * 4;
}
 
InsertInstruction(proto, name);

```

最后更新字段数据

```
void CalcProtoSize(Proto* proto)//uleb128不定长, 须计算所有长度
{
    unsigned char* buf = new unsigned char[4096];
    unsigned char* st = buf;
 
    WriteU8(buf, proto->ph.flags);
    WriteU8(buf, proto->ph.argCnt);
    WriteU8(buf, proto->ph.frameSize);
    WriteU8(buf, proto->ph.upValueCnt);
    WriteUleb128(buf, proto->ph.complexCnt);
    WriteUleb128(buf, proto->ph.numericCnt);
    WriteUleb128(buf, proto->ph.instructionCnt);
    proto->ph.protoSize = (buf - st) + proto->constantsSize + proto->instructionsSize;
}
void WriteProtoIntoBuffer(unsigned char*& buf, Proto* proto)
{
    WriteUleb128(buf, proto->ph.protoSize);
    WriteU8(buf, proto->ph.flags);
    WriteU8(buf, proto->ph.argCnt);
    WriteU8(buf, proto->ph.frameSize);
    WriteU8(buf, proto->ph.upValueCnt);
    WriteUleb128(buf, proto->ph.complexCnt);
    WriteUleb128(buf, proto->ph.numericCnt);
    WriteUleb128(buf, proto->ph.instructionCnt);
 
    memcpy(buf, proto->instructions, proto->instructionsSize);
    buf += proto->instructionsSize;
    memcpy(buf, proto->constants, proto->constantsSize);
    buf += proto->constantsSize;
}
 
CalcProtoSize(proto);
WriteProtoIntoBuffer(filePtr, proto);

```

结束
--

最终效果成功注入

 

![](https://bbs.pediy.com/upload/attach/202203/831526_MP57HE43JG6574Q.png)

 

完整代码见 https://github.com/stickycookie/luajitInject

 

参考资料: https://www.anquanke.com/post/id/87281

 

​ https://en.wikipedia.org/wiki/LEB128

 

​ https://www.mickaelwalter.fr/reverse-engineering-luajit/

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

[#逆向分析](forum-161-1-118.htm) [#工具脚本](forum-161-1-128.htm)